# PRD — BMKG Weather Aggregator

| Field | Value |
|---|---|
| Versi dokumen | 0.1 |
| Status | Draft untuk direview |
| Tanggal | 2026-10-26 |
| Pemilik | Akbar |
| Repo | `akbartrii/portofolio-template` |
| Design spec | [`docs/product/design-spec.md`](design-spec.md) |

## 1. Ringkasan

BMKG Weather Aggregator adalah aplikasi web kecil dan REST API yang mengambil prakiraan cuaca publik dari API BMKG, menormalkan bentuk datanya, menyimpannya sementara di cache, lalu menyajikannya lewat API yang bersih dan satu halaman web untuk memilih kota.

Proyek ini adalah proyek portofolio. Nilai utamanya bukan sekadar "menampilkan cuaca", tetapi menunjukkan kemampuan merancang API, menangani integrasi pihak ketiga yang tidak stabil, caching, validasi input, error handling, testing, dan dokumentasi keputusan teknis.

## 2. Latar belakang dan masalah

API publik BMKG mengembalikan data yang kaya tetapi:

- berbentuk mentah dan bertingkat (`data[].cuaca[][]`), sehingga konsumen harus tahu detail internalnya;
- hanya bisa diakses lewat kode wilayah `adm4` yang tidak mudah ditebak pengguna;
- tidak punya cache atau kontrak versi yang ramah untuk aplikasi lain;
- respons dapat lambat atau gagal, sehingga aplikasi yang memanggilnya langsung akan ikut rapuh.

Pengguna awam juga tidak ingin menghafal `31.71.03.1001`; mereka ingin memilih "Jakarta" atau "Bandung" dan langsung melihat prakiraan.

## 3. Tujuan dan non-tujuan

### Tujuan v1

1. Menyediakan API `GET /api/weather` yang menerima nama kota (slug) atau kode `adm4` dan mengembalikan prakiraan ternormalisasi.
2. Menyediakan `GET /api/cities` untuk daftar kota yang didukung.
3. Menyediakan satu halaman web untuk memilih kota dan melihat prakiraan.
4. Menjaga beban ke API BMKG lewat cache TTL dan penggabungan request paralel (single-flight).
5. Menangani kegagalan upstream dengan timeout, retry terbatas, dan pesan error yang konsisten.
6. Menjaga kualitas lewat lint, typecheck, unit/integration test, dan CI.

### Non-tujuan v1

- Menyimpan riwayat prakiraan ke database.
- Menjadwalkan ingestion data secara berkala.
- Menyediakan autentikasi, akun pengguna, atau API key.
- Memberi peringatan cuaca ekstrem lewat push/email.
- Mencari wilayah dari teks bebas atau geocoding.
- Menjamin SLA produksi atau dukungan multi-region.
- Menyediakan UI berbahasa selain Indonesia (deskripsi English dari BMKG tetap diteruskan).

## 4. Persona

| Persona | Kebutuhan | Cara produk melayani |
|---|---|---|
| Pengunjung umum | Lihat cuaca kota tanpa tahu kode wilayah | Halaman web dengan dropdown kota |
| Recruiter / reviewer portofolio | Cepat menilai kualitas engineering | README, ADR, test, CI, API docs |
| Developer lain | Konsumsi data cuaca yang sudah rapi | REST API dengan kontrak dan contoh |
| Pemilik proyek | Merawat kode kecil yang jelas | Modul terpisah, konfigurasi tervalidasi, dokumentasi |

## 5. User stories dan acceptance criteria

### US-01 — Melihat prakiraan lewat web

Sebagai pengunjung, saya ingin memilih kota dan melihat prakiraan beberapa jam ke depan, supaya saya tahu kondisi cuaca tanpa membuka banyak situs.

Acceptance criteria:

- Halaman menampilkan daftar kota dari `GET /api/cities`.
- Setelah kota dipilih, halaman memanggil `GET /api/weather?city=<slug>`.
- Kartu prakiraan menampilkan waktu lokal, suhu, kelembapan, deskripsi cuaca, dan ikon.
- Saat memuat, ada indikator loading; saat gagal, ada pesan error yang bisa dibaca.
- Atribusi BMKG dan disclaimer terlihat di halaman.

### US-02 — Query lewat API berdasarkan kota

Sebagai developer, saya ingin memanggil `GET /api/weather?city=bandung` dan mendapat JSON yang stabil.

Acceptance criteria:

- Respons sukses `200` memakai envelope `{ data, meta }`.
- `data.location` memuat provinsi, kota/kabupaten, kecamatan, desa, koordinat, timezone.
- `data.forecasts` berupa array terurut naik berdasarkan waktu.
- `meta.cached` menandakan apakah respons berasal dari cache.
- `meta.fetchedAt` adalah waktu pengambilan dari BMKG dalam ISO 8601.

### US-03 — Query lewat API berdasarkan `adm4`

Sebagai developer, saya ingin memanggil `GET /api/weather?adm4=31.71.03.1001` untuk wilayah tertentu.

Acceptance criteria:

- Format `adm4` divalidasi sebelum memanggil BMKG.
- `adm4` yang valid tetapi tidak dikenal BMKG menghasilkan `404 ADM4_NOT_FOUND`.
- `adm4` yang formatnya salah menghasilkan `400 INVALID_QUERY`.

### US-04 — Hasil cepat saat data sudah di-cache

Sebagai pengguna, saya ingin respons berikutnya untuk kota yang sama terasa instan.

Acceptance criteria:

- Request kedua dalam TTL tidak memanggil BMKG.
- `meta.cached` bernilai `true`.
- `meta.fetchedAt` tetap menunjukkan waktu pengambilan asli.

### US-05 — Error yang jelas

Sebagai pengguna API, saya ingin tahu apa yang salah tanpa membaca log server.

Acceptance criteria:

- Semua error memakai `{ error: { code, message, requestId } }`.
- Kode error stabil dan terdokumentasi di `docs/architecture/api-spec.md`.
- Timeout BMKG menghasilkan `504`; kegagalan upstream lain menghasilkan `502`.
- Tidak ada stack trace atau detail internal yang bocor ke klien.

### US-06 — Katalog kota yang jujur

Sebagai pemilik produk, saya ingin hanya menampilkan kota yang kodenya sudah diverifikasi, supaya demo tidak menampilkan data salah.

Acceptance criteria:

- Setiap kota di katalog punya `adm4` yang sudah diuji ke API BMKG.
- Kota tanpa kode terverifikasi tidak dimasukkan.
- Katalog dapat ditambah lewat satu file data tanpa mengubah route.

## 6. Kebutuhan fungsional

| ID | Kebutuhan | Prioritas |
|---|---|---|
| FR-01 | `GET /health` mengembalikan status, uptime, dan versi | Must |
| FR-02 | `GET /api/cities` mengembalikan daftar kota terverifikasi | Must |
| FR-03 | `GET /api/weather?city=<slug>` mengembalikan prakiraan berdasarkan slug | Must |
| FR-04 | `GET /api/weather?adm4=<kode>` mengembalikan prakiraan berdasarkan kode wilayah | Must |
| FR-05 | Normalisasi respons BMKG ke model internal | Must |
| FR-06 | Cache TTL in-memory per `adm4` | Must |
| FR-07 | Single-flight untuk mencegah request paralel ke `adm4` yang sama | Must |
| FR-08 | Validasi query dengan pesan error terstruktur | Must |
| FR-09 | Retry terbatas dan timeout untuk panggilan BMKG | Must |
| FR-10 | Halaman web statis untuk memilih kota dan melihat prakiraan | Must |
| FR-11 | Atribusi dan disclaimer BMKG tampil di web dan README | Must |
| FR-12 | Konfigurasi lewat environment variable yang divalidasi saat boot | Must |
| FR-13 | Graceful shutdown | Should |
| FR-14 | `GET /` menyajikan UI | Should |
| FR-15 | Endpoint histori/agregasi harian | Won't (v1) |

## 7. Kebutuhan non-fungsional

| ID | Kebutuhan | Target |
|---|---|---|
| NFR-01 | Runtime | Node.js 24 LTS, TypeScript strict |
| NFR-02 | Latensi cache hit (lokal) | p95 < 50 ms |
| NFR-03 | Latensi cache miss | p95 < 3 dtk, di luar keterlambatan BMKG |
| NFR-04 | Ketersediaan saat BMKG gagal | Tetap mengembalikan error terstruktur, tidak crash |
| NFR-05 | Keamanan rahasia | Tidak ada secret di repo; secret scanning aktif |
| NFR-06 | Privasi | Tidak menyimpan data pribadi; tidak ada tracking |
| NFR-07 | Observability | Log terstruktur (pino) + request id |
| NFR-08 | Kualitas kode | Lint, format, typecheck, test lulus di CI |
| NFR-09 | Cakupan test | Minimal 80% baris pada `src/services`, `src/clients`, `src/cache` |
| NFR-10 | Kompatibilitas API | Prefix `/api`, perubahan breaking butuh versi baru |
| NFR-11 | Ukuran gambar | Image Docker < 200 MB (saat Docker ditambahkan) |
| NFR-12 | Etika upstream | Cache wajib aktif; tidak ada polling agresif |

## 8. Ruang lingkup

### Di dalam v1

- API, cache, UI, dokumentasi, test, CI.
- 15 kota terverifikasi (lihat `docs/product/business-rules.md`).
- Dockerfile dan Compose disiapkan pada tahap berikutnya, bukan bagian dari dokumen ini.

### Di luar v1 (lihat `docs/product/roadmap.md`)

- PostgreSQL untuk riwayat.
- Redis untuk cache bersama.
- Ingestion terjadwal.
- Rate limiting dan API key.
- Pencarian wilayah di luar allowlist.

## 9. Metrik keberhasilan

Metrik di bawah adalah **target yang harus diukur**, bukan klaim hasil. Angka aktual diisi setelah implementasi dan pengujian.

| Metrik | Target | Cara mengukur |
|---|---|---|
| Cache hit ratio pada beban demo | > 80% | Log `cache_hit` / `cache_miss` |
| p95 latensi cache hit | < 50 ms | k6 atau autocannon |
| p95 latensi cache miss | < 3 dtk | k6 dengan BMKG asli |
| Error rate request valid | < 1% | Log level error |
| Cakupan test modul inti | >= 80% | `vitest run --coverage` |
| Waktu boot | < 2 dtk | Ukur `npm start` sampai `/health` siap |

## 10. Risiko dan mitigasi

| Risiko | Dampak | Mitigasi |
|---|---|---|
| BMKG lambat atau down | Pengguna melihat error | Timeout 8 dtk, retry 1x, cache, error 502/504 |
| Bentuk respons BMKG berubah | Normalisasi rusak | Schema zod longgar + test fixture + log parsing error |
| `adm4` katalog salah | Data kota tertukar | Hanya kode terverifikasi; test katalog |
| Cache stampede | Lonjakan ke BMKG | Single-flight per key |
| Rate limit tidak ada | Penyalahgunaan | Batasi lewat cache + reverse proxy saat deploy; rate-limit di roadmap |
| Data dianggap resmi | Salah informasi | Disclaimer jelas: sumber BMKG, bukan produk resmi |
| Node 25 di mesin lokal | Test gagal karena engines | Pin `.nvmrc` ke 24 dan dokumentasikan `nvm use` |

## 11. Ketergantungan eksternal dan legal

- Sumber data: API publik BMKG (`https://api.bmkg.go.id/publik/prakiraan-cuaca`).
- Proyek ini **tidak berafiliasi** dengan BMKG.
- Hanya data publik; tidak ada data pribadi.
- Ikon cuaca berasal dari URL BMKG; jangan menyalin aset tanpa lisensi.
- Atribusi wajib ada di UI dan README.

## 12. Definisi istilah

| Istilah | Arti |
|---|---|
| `adm4` | Kode wilayah administratif level desa/kelurahan, format `XX.XX.XX.XXXX` |
| Prakiraan | Daftar kondisi cuaca per slot waktu dari BMKG |
| Normalisasi | Transformasi respons mentah BMKG ke model internal |
| Cache hit | Data ditemukan di cache dan belum kedaluwarsa |
| Single-flight | Hanya satu request upstream berjalan untuk key yang sama |
| Slug | Nama kota yang ramah URL, mis. `jakarta-kemayoran` |

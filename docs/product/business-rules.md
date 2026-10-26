# Business Rules — BMKG Weather Aggregator

Dokumen ini adalah aturan yang harus dipatuhi implementasi. Jika kode dan dokumen berbeda, kode dianggap bug sampai salah satunya diperbaiki.

## 1. Katalog kota

### BR-01 — Allowlist, bukan pencarian bebas

Hanya kota yang terdaftar di `src/data/cities.ts` yang bisa dipanggil lewat `?city=`. Setiap `adm4` harus sudah diverifikasi ke API BMKG sebelum masuk katalog.

### BR-02 — Kota terverifikasi v1

| Slug | Nama tampilan | Provinsi | Kab/Kota | Kecamatan | `adm4` |
|---|---|---|---|---|---|
| `jakarta-gambir` | Jakarta — Gambir | DKI Jakarta | Kota Adm. Jakarta Pusat | Gambir | `31.71.01.1001` |
| `jakarta-kemayoran` | Jakarta — Kemayoran | DKI Jakarta | Kota Adm. Jakarta Pusat | Kemayoran | `31.71.03.1001` |
| `bandung` | Bandung | Jawa Barat | Kota Bandung | Sukasari | `32.73.01.1001` |
| `surabaya` | Surabaya | Jawa Timur | Kota Surabaya | Karang Pilang | `35.78.01.1001` |
| `medan` | Medan | Sumatera Utara | Kota Medan | Medan Kota | `12.71.01.1001` |
| `makassar` | Makassar | Sulawesi Selatan | Kota Makassar | Mariso | `73.71.01.1001` |
| `denpasar` | Denpasar | Bali | Kota Denpasar | Denpasar Selatan | `51.71.01.1001` |
| `semarang` | Semarang | Jawa Tengah | Kota Semarang | Semarang Tengah | `33.74.01.1001` |
| `yogyakarta` | Yogyakarta | DI Yogyakarta | Kota Yogyakarta | Tegalrejo | `34.71.01.1001` |
| `cilegon` | Cilegon | Banten | Kota Cilegon | Cibeber | `36.72.01.1001` |
| `padang` | Padang | Sumatera Barat | Kota Padang | Padang Selatan | `13.71.01.1001` |
| `palembang` | Palembang | Sumatera Selatan | Kota Palembang | Ilir Barat Dua | `16.71.01.1001` |
| `banjarmasin` | Banjarmasin | Kalimantan Selatan | Kota Banjarmasin | Banjarmasin Selatan | `63.71.01.1001` |
| `manado` | Manado | Sulawesi Utara | Kota Manado | Bunaken | `71.71.01.1001` |
| `jayapura` | Jayapura | Papua | Kota Jayapura | Jayapura Utara | `91.71.01.1001` |

### BR-03 — Slug unik dan case-insensitive

- Slug disimpan huruf kecil, hanya `a-z`, `0-9`, dan `-`.
- Pencarian slug tidak membedakan huruf besar/kecil.
- Tidak boleh ada dua kota dengan slug sama.

### BR-04 — `adm4` unik

Tidak boleh ada dua entri katalog dengan `adm4` sama. Ini mencegah cache key bertabrakan.

## 2. Validasi query

### BR-05 — Format `adm4`

`adm4` harus cocok dengan regex:

```
^\d{2}\.\d{2}\.\d{2}\.\d{4}$
```

Contoh valid: `31.71.03.1001`. Contoh tidak valid: `3171031001`, `31.71.03`, `abc`.

### BR-06 — Tepat satu parameter

`GET /api/weather` menerima tepat satu dari `city` atau `adm4`.

| Kondisi | Hasil |
|---|---|
| `city` saja, valid | Lanjut |
| `adm4` saja, format valid | Lanjut |
| Keduanya diisi | `400 AMBIGUOUS_QUERY` |
| Tidak ada keduanya | `400 INVALID_QUERY` |
| `city` kosong (`?city=`) | `400 INVALID_QUERY` |
| `adm4` format salah | `400 INVALID_QUERY` |
| Parameter asing (mis. `?foo=bar`) | Diabaikan, tidak error |

### BR-07 — Panjang slug

Slug maksimal 64 karakter. Di luar itu `400 INVALID_QUERY`.

### BR-08 — Kota tidak dikenal

`?city=` yang formatnya benar tetapi tidak ada di katalog menghasilkan `404 CITY_NOT_FOUND`.

### BR-09 — `adm4` tidak dikenal BMKG

`adm4` berformat benar tetapi BMKG mengembalikan `404` menghasilkan `404 ADM4_NOT_FOUND`. Kode ini tidak harus ada di katalog, karena `?adm4=` mengizinkan wilayah mana pun yang valid di BMKG.

## 3. Cache

### BR-10 — Key cache

Key cache adalah `weather:<adm4>`, mis. `weather:31.71.03.1001`.

### BR-11 — TTL default dan batas

- TTL default: `600` detik (10 menit).
- TTL minimum: `60` detik.
- TTL maksimum: `3600` detik.
- Nilai di luar batas ditolak saat validasi konfigurasi (gagal boot), bukan dipotong diam-diam.

### BR-12 — Cache hit

Jika entri ada dan belum kedaluwarsa:

- Tidak memanggil BMKG.
- Respons memakai `meta.cached = true`.
- `meta.fetchedAt` tetap waktu pengambilan asli.

### BR-13 — Cache miss

Jika tidak ada atau kedaluwarsa:

- Ambil dari BMKG.
- Simpan hasil ternormalisasi dengan TTL.
- Respons memakai `meta.cached = false`.
- `meta.fetchedAt` diisi waktu pengambilan saat ini.

### BR-14 — Hanya hasil sukses yang di-cache

Error upstream, timeout, atau respons tidak valid tidak boleh disimpan di cache.

### BR-15 — Single-flight

Jika dua request untuk `adm4` yang sama datang saat cache kosong, hanya satu request ke BMKG yang dijalankan. Request kedua menunggu hasil yang sama. Setelah selesai, hasil dibagikan dan disimpan ke cache.

### BR-16 — Tidak ada cache negatif

404 dari BMKG tidak di-cache pada v1. Ini menghindari menyimpan kesalahan sementara.

## 4. Upstream BMKG

### BR-17 — Base URL dan endpoint

- Base URL default: `https://api.bmkg.go.id`
- Path: `/publik/prakiraan-cuaca?adm4=<kode>`
- Base URL dapat dioverride lewat `BMKG_BASE_URL` (untuk test/mock).

### BR-18 — Timeout

Timeout per percobaan: `8000` ms (dapat dikonfigurasi `BMKG_TIMEOUT_MS`, 1000–30000).

### BR-19 — Retry

- Maksimal `1` retry (total 2 percobaan).
- Retry hanya untuk: error jaringan, timeout, dan status `5xx`.
- Tidak retry untuk status `4xx`.
- Jeda sebelum retry: `250` ms.
- Setiap percobaan memakai timeout penuh.

### BR-20 — Pemetaan status upstream

| Kondisi upstream | Status ke klien | Kode error |
|---|---|---|
| `200` dengan body valid | `200` | — |
| `200` dengan body tidak bisa diparse | `502` | `UPSTREAM_ERROR` |
| `404` | `404` | `ADM4_NOT_FOUND` |
| `400`/`422` | `502` | `UPSTREAM_ERROR` |
| `429` | `502` | `UPSTREAM_ERROR` |
| `5xx` setelah retry | `502` | `UPSTREAM_ERROR` |
| Timeout setelah retry | `504` | `UPSTREAM_TIMEOUT` |
| DNS/connection error setelah retry | `502` | `UPSTREAM_ERROR` |

### BR-21 — Tidak membocorkan detail upstream

Pesan error ke klien tidak boleh memuat body mentah BMKG, URL internal, atau stack trace. Detail lengkap hanya di log server.

## 5. Normalisasi data

### BR-22 — Lokasi

Ambil dari `lokasi` BMKG dan petakan:

| Field internal | Field BMKG |
|---|---|
| `adm4` | `lokasi.adm4` |
| `province` | `lokasi.provinsi` |
| `city` | `lokasi.kotkab` |
| `district` | `lokasi.kecamatan` |
| `village` | `lokasi.desa` |
| `latitude` | `lokasi.lat` |
| `longitude` | `lokasi.lon` |
| `timezone` | `lokasi.timezone` |

### BR-23 — Flatten prakiraan

`data[].cuaca` adalah array of arrays. Semua elemen harus di-flatten menjadi satu array.

### BR-24 — Field prakiraan

| Field internal | Field BMKG | Aturan |
|---|---|---|
| `datetimeUtc` | `datetime` | ISO 8601 UTC |
| `datetimeLocal` | `local_datetime` | Tambahkan offset dari `timezone` |
| `temperatureC` | `t` | Angka; entri tanpa nilai dilewati |
| `humidityPct` | `hu` | Angka atau `null` |
| `precipitationMm` | `tp` | Angka atau `null` |
| `cloudCoverPct` | `tcc` | Angka atau `null` |
| `weatherId` | `weather` | Angka atau `null` |
| `descriptionId` | `weather_desc` | String atau `null` |
| `descriptionEn` | `weather_desc_en` | String atau `null` |
| `windSpeedKmh` | `ws` | Angka atau `null` |
| `windDirection` | `wd` | String atau `null` |
| `windDirectionTo` | `wd_to` | String atau `null` |
| `windDegree` | `wd_deg` | Angka atau `null` |
| `iconUrl` | `image` | URL atau `null` |

Field `vs`, `vs_text`, `time_index`, dan `analysis_date` tidak wajib diteruskan pada v1.

### BR-25 — Urutan dan duplikasi

- Buang entri yang tidak punya `datetimeUtc` valid atau `temperatureC` bukan angka.
- Urutkan naik berdasarkan `datetimeUtc`.
- Jika ada dua entri dengan `datetimeUtc` sama, pertahankan yang pertama.

### BR-26 — Hasil kosong

Jika setelah normalisasi tidak ada prakiraan, kembalikan `502 UPSTREAM_ERROR` karena data dianggap tidak dapat dipakai.

## 6. Respons API

### BR-27 — Envelope sukses

Semua respons sukses memakai:

```json
{
  "data": {},
  "meta": {}
}
```

### BR-28 — Envelope error

Semua respons error memakai:

```json
{
  "error": {
    "code": "STRING_CODE",
    "message": "Pesan yang bisa dibaca manusia",
    "requestId": "req-1"
  }
}
```

### BR-29 — `requestId`

Nilai diambil dari request id Fastify dan dikirim juga di header `x-request-id`. Klien dapat menyebutkannya saat melapor.

### BR-30 — Header cache HTTP

Respons `GET /api/weather` yang sukses mengirim `Cache-Control: public, max-age=300`. Ini batas cache di klien/proxy, terpisah dari TTL internal 600 detik.

### BR-31 — Kode error stabil

Kode yang dipakai v1: `INVALID_QUERY`, `AMBIGUOUS_QUERY`, `CITY_NOT_FOUND`, `ADM4_NOT_FOUND`, `UPSTREAM_ERROR`, `UPSTREAM_TIMEOUT`, `INTERNAL_ERROR`. Menambah kode baru tidak dianggap breaking; mengganti nama dianggap breaking.

## 7. Konfigurasi dan runtime

### BR-32 — Fail fast

Semua environment variable divalidasi saat boot. Nilai tidak valid membuat proses berhenti dengan pesan jelas dan exit code non-zero.

### BR-33 — Default yang aman

Variabel yang tidak wajib punya default yang aman untuk lokal. Tidak ada secret yang dibutuhkan pada v1.

### BR-34 — Graceful shutdown

SIGINT/SIGTERM menutup server, berhenti menerima koneksi baru, dan menyelesaikan request yang sedang berjalan dalam batas 10 detik sebelum keluar.

### BR-35 — Log

- Format JSON di produksi, pretty di development.
- Jangan pernah mencatat secret, header otorisasi, atau body respons penuh.
- Catat `adm4`, `cache_hit`, `duration_ms`, dan `request_id`.

## 8. Atribusi, legal, dan etika

### BR-36 — Atribusi

UI dan README wajib menyebut BMKG sebagai sumber data dan menyatakan proyek tidak berafiliasi dengan BMKG.

### BR-37 — Tidak menyimpan data pribadi

Tidak ada akun, cookie tracking, atau penyimpanan IP pengguna. Log tidak boleh memuat IP mentah pada v1.

### BR-38 — Etika penggunaan API

- Cache wajib aktif di semua lingkungan selain test.
- Tidak boleh melakukan polling lebih cepat dari TTL.
- Tidak boleh melakukan scraping halaman web BMKG; hanya API publik.

## 9. Definisi selesai (v1)

- Semua aturan di atas punya test atau disebut eksplisit sebagai tidak diuji.
- `npm run lint`, `npm run typecheck`, `npm test`, dan `npm run build` lulus.
- CI hijau di Node 24.
- README, API spec, dan ADR konsisten dengan perilaku kode.

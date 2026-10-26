# Design Spec — BMKG Weather Aggregator v1

| Field | Value |
|---|---|
| Tanggal | 2026-10-26 |
| Status | Disetujui untuk implementasi |
| Penulis | Akbar (dengan bantuan AI) |
| Repo | `akbartrii/portofolio-template` |

## 1. Ringkasan

Aplikasi satu proses Node.js 24 yang menyajikan REST API dan UI statis untuk prakiraan cuaca BMKG. Data diambil dari API publik BMKG, dinormalisasi, di-cache in-memory dengan TTL, dan dilindungi single-flight. Tidak ada database pada v1.

## 2. Tujuan

- API yang bersih untuk mengambil prakiraan berdasarkan slug kota atau `adm4`.
- UI satu halaman untuk memilih kota dan melihat prakiraan.
- Integrasi upstream yang tahan gagal: timeout, retry terbatas, error terstruktur.
- Kode yang mudah diuji: IO diinjeksi, normalizer murni.
- Dokumentasi lengkap sebelum implementasi.

## 3. Non-tujuan

Database, Redis, ingestion terjadwal, autentikasi, pencarian wilayah bebas, peringatan cuaca, SLA produksi.

## 4. Arsitektur

```
Browser / API Client
        |
        v
   Fastify (API + static)
        |
        v
   WeatherService
     |        \
     v         v
  Cache     BmkgClient --> BMKG API
              |
              v
          Normalizer
```

Detail: [`docs/architecture/architecture.md`](../architecture/architecture.md).

## 5. Komponen

| Komponen | Tanggung jawab |
|---|---|
| `config` | Validasi env saat boot |
| `routes` | HTTP parsing, validasi, status code |
| `services/weather` | Orkestrasi cache + single-flight + normalisasi |
| `clients/bmkg` | HTTP, timeout, retry, parsing schema |
| `cache` | TTL in-memory + single-flight |
| `schemas` | Model internal + normalizer |
| `data/cities` | Katalog 15 kota terverifikasi |
| `public` | UI statis |

Detail: [`docs/architecture/modules.md`](../architecture/modules.md).

## 6. Kontrak API

| Method | Path | Hasil |
|---|---|---|
| GET | `/health` | Status proses |
| GET | `/api/cities` | Daftar kota |
| GET | `/api/weather?city=<slug>` | Prakiraan berdasarkan slug |
| GET | `/api/weather?adm4=<kode>` | Prakiraan berdasarkan kode wilayah |
| GET | `/` | UI |

Envelope sukses `{ data, meta }`; error `{ error: { code, message, requestId } }`.

Detail dan contoh: [`docs/architecture/api-spec.md`](../architecture/api-spec.md).

## 7. Aturan inti

- Tepat satu parameter (`city` atau `adm4`).
- `adm4` cocok `^\d{2}\.\d{2}\.\d{2}\.\d{4}$`.
- Cache key `weather:<adm4>`, TTL default 600 dtk (60–3600).
- Hanya hasil sukses yang di-cache.
- Timeout upstream 8 dtk, retry 1x dengan jeda 250 ms, hanya untuk jaringan/5xx.
- Normalisasi: flatten `data[].cuaca[][]`, buang entri tanpa waktu/suhu, urut naik, buang duplikat.
- Hasil normalisasi kosong -> `502 UPSTREAM_ERROR`.

Detail: [`docs/product/business-rules.md`](business-rules.md).

## 8. Data

Model internal: `City`, `Location`, `Forecast`, `WeatherResult`, metadata cache. Pemetaan field BMKG dan algoritma normalisasi ada di [`docs/architecture/data-model.md`](../architecture/data-model.md).

## 9. Testing

- Unit: config, cache, single-flight, katalog, normalizer, client, service.
- Integrasi: seluruh route via `fastify.inject` dengan client mock.
- Tanpa jaringan di CI. Smoke live opsional dengan `LIVE_BMKG=1`.
- Target coverage >= 80% untuk modul inti.

Detail: [`docs/analysis/testing-strategy.md`](../analysis/testing-strategy.md).

## 10. Stack

Node 24 LTS, TypeScript 5.9.3, Fastify 5.12.7, zod 4.6.7, Vitest 5.0.5, ESLint 10, Prettier 3.9, pino.

Alasan dan ADR: [`docs/architecture/architecture.md`](../architecture/architecture.md#6-keputusan-arsitektur).

## 11. Kriteria penerimaan v1

- [ ] `nvm use 24` lalu `npm ci` berhasil.
- [ ] `npm run dev` menyajikan UI di `http://localhost:8080`.
- [ ] `GET /health` mengembalikan 200.
- [ ] `GET /api/cities` mengembalikan 15 kota.
- [ ] `GET /api/weather?city=bandung` mengembalikan 200 dengan prakiraan terurut.
- [ ] `GET /api/weather?adm4=31.71.03.1001` mengembalikan 200.
- [ ] Request kedua dalam TTL menghasilkan `meta.cached = true`.
- [ ] Semua kode error di `api-spec.md` punya test.
- [ ] `npm run lint`, `npm run typecheck`, `npm test`, `npm run build` lulus.
- [ ] CI hijau di Node 24.
- [ ] README, ADR, dan dokumen lain konsisten.
- [ ] Atribusi BMKG tampil di UI dan README.

## 12. Risiko

| Risiko | Mitigasi |
|---|---|
| BMKG berubah bentuk | Schema toleran + fixture + log parsing |
| Kode `adm4` salah | Hanya kode terverifikasi + test katalog |
| Cache stampede | Single-flight |
| Node 25 lokal | `.nvmrc` 24 + dokumentasi |
| Skema TypeScript 7 | Pakai 5.9.3 sampai ekosistem siap |

## 13. Dokumen terkait

- [`docs/product/prd.md`](prd.md)
- [`docs/product/business-rules.md`](business-rules.md)
- [`docs/product/roadmap.md`](roadmap.md)
- [`docs/architecture/architecture.md`](../architecture/architecture.md)
- [`docs/architecture/modules.md`](../architecture/modules.md)
- [`docs/architecture/workflows.md`](../architecture/workflows.md)
- [`docs/architecture/api-spec.md`](../architecture/api-spec.md)
- [`docs/architecture/data-model.md`](../architecture/data-model.md)
- [`docs/analysis/testing-strategy.md`](../analysis/testing-strategy.md)
- [`docs/analysis/quality-checklist.md`](../analysis/quality-checklist.md)
- [`docs/operations/operations.md`](../operations/operations.md)
- [`docs/changelog/CHANGELOG.md`](../changelog/CHANGELOG.md)

# Workflows — BMKG Weather Aggregator

Dokumen ini menjelaskan alur runtime dan alur kerja pengembangan. Setiap alur memetakan ke modul di `docs/architecture/modules.md` dan aturan di `docs/product/business-rules.md`.

## 1. Daftar alur

| ID | Alur | Aktor |
|---|---|---|
| WF-01 | Request cuaca berdasarkan slug kota | Klien API / web |
| WF-02 | Request cuaca berdasarkan `adm4` | Klien API |
| WF-03 | Cache hit | Sistem |
| WF-04 | Cache miss + upstream sukses | Sistem |
| WF-05 | Cache miss + upstream gagal/timeout | Sistem |
| WF-06 | Validasi query gagal | Sistem |
| WF-07 | Memuat halaman web | Pengunjung |
| WF-08 | Alur pengembangan & CI | Developer |

## 2. WF-01 — Request berdasarkan slug kota

```mermaid
sequenceDiagram
  autonumber
  participant C as Client
  participant R as Route /api/weather
  participant S as WeatherService
  participant D as CityCatalog
  participant K as Cache
  participant B as BmkgClient
  participant M as Normalizer

  C->>R: GET /api/weather?city=bandung
  R->>R: validasi query (zod)
  R->>S: getWeather({ city: "bandung" })
  S->>D: findCityBySlug("bandung")
  alt slug tidak ada
    D-->>S: undefined
    S-->>R: AppError CITY_NOT_FOUND (404)
    R-->>C: 404 { error }
  else slug ada
    D-->>S: City { adm4 }
    S->>K: get("weather:32.73.01.1001")
    alt cache hit
      K-->>S: WeatherResult
      S-->>R: { result, cached: true, fetchedAt }
      R-->>C: 200 { data, meta }
    else cache miss
      S->>S: singleFlight.run(adm4)
      S->>B: fetchForecast(adm4)
      B-->>S: BmkgRawResponse
      S->>M: normalizeForecast(raw)
      M-->>S: WeatherResult
      S->>K: set(key, result, ttl)
      S-->>R: { result, cached: false, fetchedAt }
      R-->>C: 200 { data, meta }
    end
  end
```

Langkah:

1. Route memvalidasi query. Jika gagal, langsung WF-06.
2. Service mencari slug di katalog. Tidak ditemukan -> `404 CITY_NOT_FOUND`.
3. Service memakai `adm4` dari katalog sebagai key cache.
4. Jika cache hit, lanjut WF-03. Jika miss, lanjut WF-04 atau WF-05.

## 3. WF-02 — Request berdasarkan `adm4`

```mermaid
sequenceDiagram
  autonumber
  participant C as Client
  participant R as Route /api/weather
  participant S as WeatherService
  participant K as Cache
  participant B as BmkgClient

  C->>R: GET /api/weather?adm4=31.71.03.1001
  R->>R: validasi regex adm4
  R->>S: getWeather({ adm4 })
  S->>K: get("weather:31.71.03.1001")
  alt cache hit
    K-->>S: WeatherResult
    S-->>R: cached true
    R-->>C: 200
  else cache miss
    S->>B: fetchForecast(adm4)
    alt BMKG 404
      B-->>S: AppError ADM4_NOT_FOUND
      S-->>R: AppError
      R-->>C: 404
    else sukses
      B-->>S: raw
      S-->>R: normalized, cached false
      R-->>C: 200
    end
  end
```

Catatan: `?adm4=` tidak harus ada di katalog. Katalog hanya dipakai untuk `?city=` dan daftar kota.

## 4. WF-03 — Cache hit

1. `cache.get(key)` mengembalikan entri yang belum kedaluwarsa.
2. Service tidak memanggil BMKG.
3. `cached = true`, `fetchedAt` dari entri cache.
4. Route mengirim `200` dengan `Cache-Control: public, max-age=300`.

## 5. WF-04 — Cache miss + upstream sukses

```mermaid
sequenceDiagram
  autonumber
  participant S as WeatherService
  participant F as SingleFlight
  participant B as BmkgClient
  participant M as Normalizer
  participant K as Cache

  S->>F: run(adm4, fetchAndNormalize)
  F->>B: fetchForecast(adm4)
  loop percobaan (maks 2)
    B->>B: fetch dengan timeout 8 dtk
    alt sukses 200
      B-->>F: raw
    else 5xx / jaringan / timeout
      B->>B: tunggu 250 ms
      B->>B: retry
    end
  end
  F->>M: normalizeForecast(raw)
  M-->>F: WeatherResult
  F-->>S: WeatherResult
  S->>K: set(key, result, ttl)
  S-->>S: cached = false
```

Aturan:

- Hanya hasil sukses yang disimpan.
- Normalisasi yang menghasilkan nol prakiraan dianggap gagal (`UPSTREAM_ERROR`).
- Request lain untuk key yang sama menunggu promise yang sama.

## 6. WF-05 — Upstream gagal atau timeout

```mermaid
sequenceDiagram
  autonumber
  participant S as WeatherService
  participant B as BmkgClient
  participant R as Route
  participant C as Client

  S->>B: fetchForecast(adm4)
  B->>B: percobaan 1 gagal
  B->>B: retry setelah 250 ms
  B->>B: percobaan 2 gagal
  alt timeout
    B-->>S: UpstreamTimeoutError
    S-->>R: AppError UPSTREAM_TIMEOUT
    R-->>C: 504 { error }
  else error lain
    B-->>S: UpstreamError
    S-->>R: AppError UPSTREAM_ERROR
    R-->>C: 502 { error }
  end
```

- Tidak ada cache negatif.
- Tidak ada data parsial yang dikirim.
- Log memuat `adm4`, `attempt`, `duration_ms`, dan `request_id`.
- Klien hanya menerima pesan umum, tanpa detail internal.

## 7. WF-06 — Validasi query gagal

| Input | Status | Kode |
|---|---|---|
| Tidak ada `city`/`adm4` | 400 | `INVALID_QUERY` |
| Keduanya ada | 400 | `AMBIGUOUS_QUERY` |
| `adm4` format salah | 400 | `INVALID_QUERY` |
| `city` kosong atau > 64 karakter | 400 | `INVALID_QUERY` |
| `city` tidak ada di katalog | 404 | `CITY_NOT_FOUND` |

Validasi dilakukan di route sebelum service dipanggil. Tidak ada panggilan BMKG untuk request tidak valid.

## 8. WF-07 — Memuat halaman web

```mermaid
sequenceDiagram
  autonumber
  participant U as Browser
  participant A as Fastify
  participant S as Static
  participant API as /api

  U->>A: GET /
  A->>S: index.html
  S-->>U: HTML + CSS + JS
  U->>API: GET /api/cities
  API-->>U: daftar kota
  U->>U: isi dropdown
  U->>API: GET /api/weather?city=<slug>
  API-->>U: data prakiraan
  U->>U: render kartu prakiraan
```

Perilaku UI:

- Saat memuat: tampilkan status loading.
- Saat error: tampilkan pesan dari `error.message` dan `requestId`.
- Menampilkan label "Sumber: BMKG" dan disclaimer.
- Tidak ada framework frontend dan tidak ada build step; file statis disajikan langsung.

## 9. WF-08 — Alur pengembangan dan CI

```mermaid
flowchart LR
  A[git checkout -b feat/...] --> B[nvm use 24]
  B --> C[npm ci]
  C --> D[npm run dev]
  D --> E[npm run lint]
  E --> F[npm run typecheck]
  F --> G[npm test]
  G --> H[npm run build]
  H --> I[commit Conventional Commits]
  I --> J[push + PR]
  J --> K[CI: lint, typecheck, test, build]
  K --> L{CI hijau?}
  L -- ya --> M[merge ke main]
  L -- tidak --> D
```

Langkah lokal:

```bash
nvm use 24
npm ci
cp .env.example .env
npm run dev
```

Definisi PR siap merge:

- Commits mengikuti Conventional Commits.
- Lint, typecheck, test, build lulus.
- Test baru untuk perilaku baru.
- Dokumentasi terkait diperbarui jika perilaku berubah.
- Tidak ada secret atau data pribadi.

## 10. Alur rilis (setelah v1 stabil)

1. Perbarui `docs/changelog/CHANGELOG.md` di bagian `[Unreleased]`.
2. Tentukan versi SemVer.
3. Pindahkan catatan ke heading versi dan tanggal.
4. Commit `chore(release): vX.Y.Z`.
5. Tag `vX.Y.Z` dan push tag.
6. Docker workflow (saat diaktifkan) membangun dan mempublikasikan image.

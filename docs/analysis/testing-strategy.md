# Testing Strategy — BMKG Weather Aggregator

## 1. Prinsip

- Test tidak boleh memanggil jaringan sungguhan. Semua HTTP di-mock.
- Waktu dikendalikan lewat `now` yang diinjeksi atau fake timer.
- Bagian yang paling rapuh (normalisasi dan error upstream) mendapat test terbanyak.
- Setiap aturan di `docs/product/business-rules.md` harus punya test atau ditandai tidak diuji.
- Test harus deterministik dan bisa jalan paralel.

## 2. Piramida test

| Lapisan | Porsi | Contoh |
|---|---|---|
| Unit | Mayoritas | normalizer, cache TTL, city lookup, config |
| Integrasi route | Sedang | `fastify.inject` dengan service/client mock |
| Smoke opsional | Sedikit | panggilan BMKG asli, hanya jika `LIVE_BMKG=1` |

Tidak ada E2E browser pada v1. UI diverifikasi manual dan lewat smoke test HTML.

## 3. Perkakas

| Kebutuhan | Alat |
|---|---|
| Test runner | Vitest 5 |
| Mock HTTP | `vi.stubGlobal('fetch', ...)` atau `fetchFn` injeksi |
| Fake time | `vi.useFakeTimers()` |
| Route test | `app.inject({ method, url })` |
| Coverage | `vitest run --coverage` (provider v8) |

## 4. Matriks test

### 4.1 `config`

| Kasus | Harapan |
|---|---|
| Env kosong | Memakai default aman |
| `PORT` bukan angka | Gagal dengan pesan jelas |
| `PORT` di luar 1-65535 | Gagal |
| `CACHE_TTL_SECONDS` < 60 | Gagal |
| `CACHE_TTL_SECONDS` > 3600 | Gagal |
| `BMKG_BASE_URL` bukan URL | Gagal |
| `LOG_LEVEL` tidak dikenal | Gagal |
| `NODE_ENV` tidak dikenal | Gagal |

### 4.2 `cache/ttl-cache`

| Kasus | Harapan |
|---|---|
| Set lalu get sebelum TTL | Mengembalikan nilai |
| Get setelah TTL | `undefined` |
| Get untuk key tidak ada | `undefined` |
| Delete | Key hilang |
| Max entries terlampaui | Entri tertua/kedaluwarsa dibuang |
| Nilai `null`/`0`/`false` | Tetap tersimpan dan dikembalikan |

### 4.3 `cache/single-flight`

| Kasus | Harapan |
|---|---|
| Dua panggilan paralel key sama | `fn` dipanggil sekali |
| Key berbeda | `fn` dipanggil per key |
| `fn` reject | Semua penunggu reject |
| Panggilan setelah selesai | `fn` dipanggil lagi |

### 4.4 `data/cities`

| Kasus | Harapan |
|---|---|
| Semua `adm4` cocok regex | Lulus |
| Semua `slug` unik | Lulus |
| Semua `adm4` unik | Lulus |
| `findCityBySlug` case-insensitive | Menemukan kota |
| Slug tidak ada | `undefined` |
| `findCityByAdm4` | Menemukan kota |

### 4.5 `schemas/bmkg` + normalizer

Fixture: `tests/fixtures/bmkg-kemayoran.json` (respons asli yang disimpan).

| Kasus | Harapan |
|---|---|
| Respons normal | Lokasi dan forecast terpetakan |
| `cuaca` array of arrays | Di-flatten semua |
| `datetime` duplikat | Satu dipertahankan |
| `t` bukan angka | Entri dibuang |
| Field opsional `null` | Menjadi `null`, bukan `undefined` |
| `forecasts` kosong setelah filter | Melempar `UPSTREAM_ERROR` |
| `local_datetime` tanpa offset | Menjadi ISO dengan `+07:00` |
| Urutan input acak | Output terurut naik |
| `lokasi` hilang | Melempar `UPSTREAM_ERROR` |
| `data` bukan array | Melempar `UPSTREAM_ERROR` |

### 4.6 `clients/bmkg.client`

| Kasus | Harapan |
|---|---|
| `200` valid | Mengembalikan parsed body |
| `404` | `ADM4_NOT_FOUND` |
| `400`/`422`/`429` | `UPSTREAM_ERROR`, tanpa retry |
| `500` lalu `200` | Retry sekali, sukses |
| `500` dua kali | `UPSTREAM_ERROR` setelah 2 percobaan |
| Timeout dua kali | `UPSTREAM_TIMEOUT` |
| Jaringan gagal lalu `200` | Retry, sukses |
| Body bukan JSON | `UPSTREAM_ERROR` |
| Body JSON tidak sesuai schema | `UPSTREAM_ERROR` |
| URL yang dipanggil | `{base}/publik/prakiraan-cuaca?adm4=...` |
| Jeda retry | `sleep` dipanggil 250 ms |

### 4.7 `services/weather.service`

| Kasus | Harapan |
|---|---|
| Cache hit | Client tidak dipanggil, `cached=true`, `fetchedAt` lama |
| Cache miss | Client dipanggil, hasil disimpan, `cached=false` |
| Dua request paralel miss | Client dipanggil sekali |
| Client gagal | Tidak menyimpan cache, error diteruskan |
| Slug tidak ada | `CITY_NOT_FOUND` |
| `adm4` diberikan | Tidak memakai katalog |
| TTL dipakai | `cache.set` menerima TTL dari config |

### 4.8 Route (integrasi)

| Request | Status | Kode |
|---|---|---|
| `GET /health` | 200 | — |
| `GET /api/cities` | 200 | — |
| `GET /api/weather?city=bandung` | 200 | — |
| `GET /api/weather?adm4=31.71.03.1001` | 200 | — |
| `GET /api/weather` | 400 | `INVALID_QUERY` |
| `GET /api/weather?city=&adm4=` | 400 | `INVALID_QUERY` |
| `GET /api/weather?city=bandung&adm4=32.73.01.1001` | 400 | `AMBIGUOUS_QUERY` |
| `GET /api/weather?city=atlantis` | 404 | `CITY_NOT_FOUND` |
| `GET /api/weather?adm4=3171031001` | 400 | `INVALID_QUERY` |
| `GET /api/weather?adm4=99.99.99.9999` | 404 | `ADM4_NOT_FOUND` |
| Upstream 500 | 502 | `UPSTREAM_ERROR` |
| Upstream timeout | 504 | `UPSTREAM_TIMEOUT` |
| Semua error | — | Punya `requestId` |
| `GET /` | 200 | HTML |

Test integrasi memakai `buildApp` dengan `weatherService` mock, bukan jaringan.

## 5. Yang tidak diuji otomatis (v1)

- Tampilan visual UI (diverifikasi manual).
- Perilaku browser sungguhan (tidak ada Playwright).
- Beban/kinerja (diukur manual dengan k6/autocannon, lihat PRD).
- Ketahanan BMKG jangka panjang.

## 6. Smoke test live (opsional)

Gated oleh `LIVE_BMKG=1`:

```bash
LIVE_BMKG=1 npm test -- tests/live
```

Memanggil satu `adm4` terverifikasi dan memastikan:

- status 200,
- `location.adm4` sesuai,
- `forecasts.length >= 1`,
- `datetimeUtc` valid.

Test ini tidak berjalan di CI default.

## 7. Coverage

Target minimal 80% baris untuk:

- `src/cache/**`
- `src/clients/**`
- `src/services/**`
- `src/schemas/**`
- `src/data/**`

`src/server.ts` dikecualikan karena hanya merakit dependency dan lifecycle.

## 8. Perintah

```bash
npm test                 # sekali jalan
npm run test:watch       # mode watch
npm run test:coverage    # dengan coverage
npm run lint
npm run typecheck
npm run build
```

## 9. CI

Workflow `.github/workflows/ci-node.yml` menjalankan, pada Node 24:

1. `npm ci`
2. `npm run lint`
3. `npm run typecheck`
4. `npm test`
5. `npm run build`

CI gagal jika salah satu langkah gagal. Tidak ada akses jaringan ke BMKG di CI.

## 10. Definisi "test cukup"

- Perilaku baru punya test yang gagal sebelum implementasi dan lulus sesudahnya.
- Bug fix disertai test regresi.
- Tidak ada test yang bergantung pada waktu nyata atau urutan eksekusi.
- Nama test menjelaskan perilaku, bukan nama fungsi internal.

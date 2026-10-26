# Operations — BMKG Weather Aggregator

## 1. Prasyarat

| Kebutuhan | Versi |
|---|---|
| Node.js | 24 LTS (lihat `.nvmrc`) |
| npm | 11.x (bawaan Node 24) |
| nvm-windows | Untuk berpindah versi Node |

Cek versi:

```powershell
nvm list
nvm use 24
node -v
npm -v
```

## 2. Environment variable

| Nama | Wajib | Default | Aturan |
|---|---|---|---|
| `NODE_ENV` | tidak | `development` | `development`, `test`, `production` |
| `HOST` | tidak | `0.0.0.0` | Alamat bind |
| `PORT` | tidak | `8080` | 1-65535 |
| `LOG_LEVEL` | tidak | `info` | `fatal`, `error`, `warn`, `info`, `debug`, `trace` |
| `BMKG_BASE_URL` | tidak | `https://api.bmkg.go.id` | URL valid, tanpa trailing slash |
| `BMKG_TIMEOUT_MS` | tidak | `8000` | 1000-30000 |
| `BMKG_RETRY` | tidak | `1` | 0-3 |
| `CACHE_TTL_SECONDS` | tidak | `600` | 60-3600 |

Tidak ada secret pada v1. `LLM_API_KEY` di template lama tidak dipakai dan dihapus dari `.env.example`.

Contoh `.env`:

```dotenv
NODE_ENV=development
HOST=0.0.0.0
PORT=8080
LOG_LEVEL=debug
BMKG_BASE_URL=https://api.bmkg.go.id
BMKG_TIMEOUT_MS=8000
BMKG_RETRY=1
CACHE_TTL_SECONDS=600
```

## 3. Menjalankan lokal

```powershell
nvm use 24
npm ci
Copy-Item .env.example .env
npm run dev
```

Buka `http://localhost:8080`.

Verifikasi:

```powershell
Invoke-RestMethod http://localhost:8080/health
Invoke-RestMethod 'http://localhost:8080/api/weather?city=bandung'
```

## 4. Script npm

| Script | Fungsi |
|---|---|
| `npm run dev` | Watch mode dengan `tsx` |
| `npm run build` | Kompilasi TypeScript ke `dist/` |
| `npm start` | Jalankan hasil build |
| `npm run typecheck` | `tsc --noEmit` |
| `npm run lint` | ESLint |
| `npm run lint:fix` | ESLint dengan `--fix` |
| `npm run format` | Prettier tulis |
| `npm run format:check` | Prettier cek |
| `npm test` | Vitest sekali jalan |
| `npm run test:watch` | Vitest watch |
| `npm run test:coverage` | Vitest + coverage |

## 5. Build dan produksi

```powershell
npm ci
npm run build
$env:NODE_ENV='production'
npm start
```

- `dist/` berisi JavaScript hasil kompilasi.
- `public/` disalin/dipertahankan dan disajikan oleh `@fastify/static`.
- Jalankan sebagai user non-root saat di container.
- Set `LOG_LEVEL=info` atau `warn` di produksi.

## 6. Health check

```
GET /health
```

Respons `200`:

```json
{ "status": "ok", "uptimeSeconds": 42, "version": "0.1.0" }
```

Tidak memanggil BMKG. Jika proses hidup, endpoint ini hidup.

## 7. Logging

- Development: pretty (`pino-pretty`).
- Produksi: JSON satu baris per event.
- Field yang dicatat: `level`, `time`, `request_id`, `method`, `url`, `status_code`, `duration_ms`.
- Event bisnis: `cache_hit`, `cache_miss`, `upstream_retry`, `upstream_error`, `upstream_timeout`.
- Dilarang mencatat: body respons penuh, header otorisasi, IP mentah, secret.

Contoh event:

```json
{
  "level": 30,
  "time": 1761462000000,
  "request_id": "req-7",
  "event": "cache_miss",
  "adm4": "32.73.01.1001",
  "cache_hit": false
}
```

## 8. Graceful shutdown

- Menangkap `SIGINT` dan `SIGTERM`.
- Berhenti menerima koneksi baru.
- Menunggu request aktif selesai maksimal 10 detik.
- Menutup server dan keluar dengan kode 0.
- Jika melewati batas, keluar dengan kode 1 dan log error.

## 9. Troubleshooting

| Gejala | Penyebab umum | Solusi |
|---|---|---|
| `EBADENGINE` saat `npm ci` | Node bukan 24 | `nvm use 24` |
| `EADDRINUSE` | Port 8080 dipakai | Ubah `PORT` atau hentikan proses lain |
| `502 UPSTREAM_ERROR` | BMKG 5xx/4xx atau body berubah | Cek log, coba `?adm4=` langsung |
| `504 UPSTREAM_TIMEOUT` | BMKG lambat / jaringan | Naikkan `BMKG_TIMEOUT_MS` sementara, cek koneksi |
| Semua kota 404 | Katalog/`adm4` salah | Jalankan test katalog, verifikasi ke API |
| Data tidak berubah setelah menit | Cache TTL | Tunggu TTL atau restart proses |
| `Cannot find module dist/...` | Belum build | `npm run build` |
| Lint gagal karena TS 7 | Versi TypeScript salah | Gunakan 5.9.3 sesuai `package.json` |

## 10. Keamanan

- `.env` tidak pernah di-commit; `.gitignore` sudah mengecualikannya.
- Secret scanning (`.github/workflows/secret-scan.yml`) berjalan di setiap push/PR.
- Tidak ada dependency dengan akses jaringan selain `fetch` ke BMKG.
- Validasi input mencegah URL injection; `adm4` dibatasi regex.
- Jangan menambahkan `eval`, `child_process`, atau deserialisasi tidak aman.

## 11. Biaya dan kuota

- API BMKG publik, tanpa API key.
- Tidak ada biaya layanan pihak ketiga pada v1.
- Beban keluar dibatasi cache (TTL 600 detik) dan single-flight.
- Saat deploy publik, tambahkan rate limit di reverse proxy untuk melindungi proses.

## 12. Rencana Docker (tahap berikutnya)

Belum diimplementasikan pada v1. Saat ditambahkan:

```dockerfile
# Garis besar, bukan file final
FROM node:24-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:24-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist
COPY --from=build /app/public ./public
COPY --from=build /app/package*.json ./
RUN npm ci --omit=dev
USER node
EXPOSE 8080
HEALTHCHECK CMD wget -qO- http://localhost:8080/health || exit 1
CMD ["node", "dist/server.js"]
```

Compose (setelah Dockerfile ada) cukup menjalankan satu service dan memetakan `8080:8080`, dengan `.env` sebagai sumber konfigurasi.

## 13. Checklist sebelum deploy

- [ ] `nvm use 24`
- [ ] `npm ci`
- [ ] `npm run lint`
- [ ] `npm run typecheck`
- [ ] `npm test`
- [ ] `npm run build`
- [ ] `/health` merespons 200
- [ ] Satu request `?city=` dan satu `?adm4=` berhasil
- [ ] Tidak ada `.env` atau secret di git
- [ ] README dan `docs/changelog/CHANGELOG.md` diperbarui

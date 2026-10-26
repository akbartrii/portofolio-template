# Roadmap — BMKG Weather Aggregator

## Prinsip

- Setiap tahap harus memberi nilai yang bisa didemokan.
- Jangan menambah infrastruktur sebelum ada kebutuhan yang terukur.
- Fitur baru harus punya aturan di `business-rules.md` dan test.

## Sekarang (v0.1)

- [x] Dokumen produk dan teknis
- [ ] Scaffold TypeScript + Fastify
- [ ] Konfigurasi tervalidasi
- [ ] Katalog 15 kota terverifikasi
- [ ] Client BMKG (timeout + retry)
- [ ] Normalizer + test
- [ ] Cache TTL + single-flight
- [ ] Endpoint `/health`, `/api/cities`, `/api/weather`
- [ ] UI statis
- [ ] Lint, typecheck, test, build
- [ ] CI Node 24
- [ ] README dan ADR

## Berikutnya (v0.2)

- [ ] Dockerfile multi-stage + `.dockerignore`
- [ ] `docker compose up` dari clone bersih
- [ ] Healthcheck container
- [ ] Rate limiting (`@fastify/rate-limit`)
- [ ] Header keamanan (`@fastify/helmet`)
- [ ] Endpoint `GET /api/weather/:adm4` sebagai alias
- [ ] Test coverage report di CI
- [ ] Deploy demo publik + link di README

## Nanti (v0.3+)

- [ ] Redis sebagai adapter `Cache` (tanpa mengubah service)
- [ ] PostgreSQL untuk riwayat prakiraan
- [ ] Ingestion terjadwal (cron/queue)
- [ ] Endpoint agregasi harian (`GET /api/weather/:adm4/history`)
- [ ] Pencarian wilayah di luar allowlist (butuh sumber kode wilayah)
- [ ] Peringatan cuaca ekstrem
- [ ] Multi-bahasa UI
- [ ] Metrik Prometheus + dashboard

## Tidak akan dikerjakan

- Autentikasi pengguna.
- Menyimpan data pribadi.
- Scraping halaman web BMKG.
- Menyalin aset berhak cipta tanpa lisensi.
- Klaim sebagai layanan resmi BMKG.

## Cara memilih prioritas

1. Apakah menambah nilai portofolio (menunjukkan skill baru)?
2. Apakah bisa diselesaikan dalam satu PR?
3. Apakah bisa diuji tanpa layanan eksternal?
4. Apakah menambah beban operasional yang tidak sepadan?

Jika jawaban 1 atau 2 "tidak", tunda.

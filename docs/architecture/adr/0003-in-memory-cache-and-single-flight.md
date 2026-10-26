# ADR-0003: Cache in-memory + single-flight untuk v1

- **Status:** Accepted
- **Date:** 2026-10-26

## Context

API BMKG publik tidak boleh dipanggil berlebihan, dan pengguna mengharapkan respons cepat. Pada v1 aplikasi berjalan sebagai satu proses, tanpa database dan tanpa Redis.

Kebutuhan:

- Mengurangi panggilan upstream untuk `adm4` yang sama.
- Tetap sederhana untuk dijalankan lokal.
- Tidak mengunci arsitektur jika nanti perlu scale-out.

## Options Considered

1. **Tanpa cache:** paling sederhana, tetapi setiap request memanggil BMKG. Lambat, boros, dan rapuh saat upstream lambat.
2. **Cache in-memory (Map + TTL):** tanpa infrastruktur tambahan, cepat, mudah diuji. Kekurangan: hilang saat restart dan tidak dibagi antar instance.
3. **Redis sejak awal:** cache bersama dan persisten. Kekurangan: menambah service, konfigurasi, dan beban operasional untuk v1 satu instance.
4. **HTTP cache saja (Cache-Control):** bergantung pada klien/proxy. Tidak melindungi server dari lonjakan.

## Decision

Gunakan **cache in-memory dengan TTL** di balik antarmuka `Cache`, ditambah **single-flight** per key. TTL default 600 detik.

Antarmuka `Cache` sengaja async agar implementasi Redis dapat ditambahkan tanpa mengubah `WeatherService`.

## Consequences

Lebih mudah:

- Tanpa dependensi infrastruktur.
- Test cepat dan deterministik dengan waktu yang diinjeksi.
- Single-flight mencegah cache stampede.
- Jalur migrasi ke Redis sudah disiapkan lewat antarmuka.

Lebih sulit:

- Cache tidak bertahan setelah restart.
- Tidak konsisten jika nanti ada lebih dari satu instance.
- Memori proses bertambah (dibatasi `maxEntries`).

Diterima karena v1 single instance dan data cuaca murah diambil ulang.

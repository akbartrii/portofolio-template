# ADR-0005: Katalog `adm4` terkurasi

- **Status:** Accepted
- **Date:** 2026-10-26

## Context

API BMKG membutuhkan kode wilayah `adm4`, bukan nama kota. Tidak tersedia pencarian wilayah resmi yang sederhana dan stabil untuk kebutuhan v1. Menebak kode berisiko menampilkan data wilayah yang salah.

## Options Considered

1. **Katalog terkurasi:** daftar kecil kota dengan `adm4` yang sudah diverifikasi manual ke API. Kekurangan: cakupan terbatas dan perawatan manual.
2. **Pencarian wilayah otomatis:** mengandalkan sumber kode wilayah pihak ketiga atau scraping. Kekurangan: tidak stabil, berisiko legal, dan menambah dependensi.
3. **Input bebas `adm4` saja:** paling murni, tetapi tidak ramah pengguna dan tidak bisa menampilkan dropdown.
4. **Geocoding koordinat:** butuh layanan tambahan dan pemetaan ke `adm4` yang belum tentu akurat.

## Decision

Gunakan **katalog terkurasi** untuk `?city=` dan daftar kota, sementara `?adm4=` tetap menerima kode valid mana pun yang dikenal BMKG. Setiap kode di katalog harus diverifikasi ke API sebelum masuk.

## Consequences

Lebih mudah:

- Tidak ada data kota yang salah.
- Demo dan test deterministik.
- Tidak ada dependensi pencarian wilayah.

Lebih sulit:

- Cakupan kota terbatas (15 pada v1).
- Menambah kota butuh verifikasi manual.
- Tidak melayani pencarian di luar allowlist lewat `city`.

Diterima karena kejujuran data lebih penting daripada cakupan luas pada proyek portofolio.

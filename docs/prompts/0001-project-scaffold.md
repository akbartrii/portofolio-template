# Prompt 0001 — Project scaffold

| Field | Value |
|---|---|
| Tanggal | 2026-10-26 |
| Alat | AI coding assistant di terminal |
| Tujuan | Mengubah repo template kosong menjadi proyek TypeScript/Node yang terdokumentasi sebelum coding |
| Hasil | PRD, aturan bisnis, arsitektur, API spec, data model, testing, operations, ADR, prompt log |

## Konteks

Repo awal hanya berisi README template, ADR template, workflow CI Java/Node, dan `.env.example`. Belum ada `package.json` atau kode. Stack yang dipilih: TypeScript/Node. Node 24 LTS tersedia lewat nvm. Domain yang dipilih: agregator cuaca BMKG.

## Prompt

Ringkasan permintaan:

- Baca `portfolio-template/` dan jelaskan isinya.
- Tentukan langkah memulai proyek.
- Gunakan TypeScript/Node saja.
- Gunakan versi pustaka terbaru yang stabil/LTS.
- Lengkapi dokumen (`PRD`, `business-rules`, `modules`, `workflows`, dan dokumen lain) sebelum menulis kode.
- Kelompokkan `docs/` ke kategori: analysis, architecture, changelog, product, prompts.

## Hasil

AI menghasilkan:

- Dokumen produk dan teknis di `docs/product`, `docs/architecture`, `docs/analysis`, `docs/operations`.
- Empat ADR baru (0002-0005).
- Katalog 15 `adm4` yang diverifikasi langsung ke API BMKG.
- README proyek dan `.env.example` baru.
- Restrukturisasi `docs/` menjadi kategori dan indeks `docs/README.md`.

Koreksi yang dilakukan selama proses:

- TypeScript 7.0.2 ditolak karena `typescript-eslint@8` mensyaratkan `<6.1.0`; dipilih 5.9.3.
- Node 25 di mesin tidak didukung Vitest 5; target diubah ke Node 24 LTS.
- Kode `adm4` yang gagal diverifikasi tidak dimasukkan ke katalog.

## Tindak lanjut manusia

- Menyetujui desain dan stack.
- Memverifikasi Node 24 terpasang.
- Akan meninjau dokumen sebelum implementasi kode.

## Tautan

- [Design spec](../product/design-spec.md)
- [PRD](../product/prd.md)
- [Business rules](../product/business-rules.md)

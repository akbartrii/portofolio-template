# Documentation Index

Dokumentasi proyek dikelompokkan per kategori. Setiap dokumen punya satu tujuan utama; jika sebuah dokumen tumbuh terlalu besar, pecah sesuai kategori.

## Kategori

| Kategori | Folder | Isi |
|---|---|---|
| Product | [`product/`](product/) | Apa yang dibangun, untuk siapa, aturan domain, rencana |
| Architecture | [`architecture/`](architecture/) | Bagaimana sistem dibangun, modul, alur, kontrak, data, ADR |
| Analysis | [`analysis/`](analysis/) | Strategi pengujian dan checklist kualitas |
| Operations | [`operations/`](operations/) | Menjalankan, mengonfigurasi, memantau, troubleshooting |
| Changelog | [`changelog/`](changelog/) | Riwayat perubahan |
| Prompts | [`prompts/`](prompts/) | Catatan prompt AI dan hasilnya |

## Product

| Dokumen | Isi |
|---|---|
| [PRD](product/prd.md) | Masalah, persona, user story, kebutuhan fungsional/non-fungsional, metrik |
| [Business rules](product/business-rules.md) | Validasi, cache, upstream, normalisasi, error, legal |
| [Roadmap](product/roadmap.md) | Sekarang, berikutnya, nanti, non-goals |
| [Design spec](product/design-spec.md) | Desain v1 yang disetujui dan kriteria penerimaan |

## Architecture

| Dokumen | Isi |
|---|---|
| [Overview](architecture/architecture.md) | Konteks sistem, komponen, alur data, stack, atribut kualitas |
| [Modules](architecture/modules.md) | Batas modul, antarmuka, aturan dependensi, error types |
| [Workflows](architecture/workflows.md) | Alur request, cache, upstream gagal, UI, pengembangan, rilis |
| [API spec](architecture/api-spec.md) | Kontrak HTTP, contoh, kode error |
| [Data model](architecture/data-model.md) | Tipe internal, pemetaan BMKG, algoritma normalisasi |
| [ADR](architecture/adr/) | Keputusan arsitektur 0001-0005 |

## Analysis

| Dokumen | Isi |
|---|---|
| [Testing strategy](analysis/testing-strategy.md) | Piramida test, matriks kasus, coverage, CI |
| [Quality checklist](analysis/quality-checklist.md) | Checklist sebelum repo dipajang |

## Operations

| Dokumen | Isi |
|---|---|
| [Operations](operations/operations.md) | Env vars, script, run, log, shutdown, troubleshooting, deploy |

## Changelog

| Dokumen | Isi |
|---|---|
| [CHANGELOG](changelog/CHANGELOG.md) | Riwayat perubahan format Keep a Changelog |

## Prompts

| Dokumen | Isi |
|---|---|
| [Prompt log](prompts/README.md) | Konvensi pencatatan prompt AI |
| [Template](prompts/0000-template.md) | Format entri prompt |
| [0001 — Project scaffold](prompts/0001-project-scaffold.md) | Permintaan awal dan hasilnya |

## Konvensi

- Nama file: `kebab-case.md`, kecuali `README.md`, `PRD` lama tidak dipakai lagi, dan `CHANGELOG.md` yang mengikuti konvensi.
- Setiap kategori punya satu topik. Jangan mencampur aturan bisnis ke dokumen arsitektur.
- Jika memindahkan dokumen, perbarui semua link di `README.md`, `docs/README.md`, dan dokumen yang mereferensikannya.
- ADR memakai nomor berurutan dan tidak dihapus; keputusan yang digantikan ditandai `Superseded by`.
- Prompt dicatat tanpa secret, kredensial, atau data pribadi.

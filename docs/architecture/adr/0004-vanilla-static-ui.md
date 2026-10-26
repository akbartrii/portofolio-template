# ADR-0004: UI statis tanpa framework frontend

- **Status:** Accepted
- **Date:** 2026-10-26

## Context

Proyek membutuhkan satu halaman untuk memilih kota dan menampilkan prakiraan. Fokus portofolio ada pada backend, API design, integrasi, dan testing. UI harus cukup baik untuk didemokan tanpa menambah kompleksitas build.

## Options Considered

1. **HTML/CSS/JS statis:** tanpa build step, disajikan `@fastify/static`. Kekurangan: state management manual, tidak ada komponen.
2. **React + Vite:** ekosistem besar, komponen rapi. Kekurangan: build step, dependensi, dan distraksi dari fokus backend.
3. **Next.js full-stack:** SSR/SSG dan React. Kekurangan: terlalu berat untuk satu halaman; menggandakan server.
4. **Server-side rendering dengan template engine:** sederhana, tetapi menambah dependensi dan mencampur tampilan dengan server.

## Decision

Gunakan **HTML, CSS, dan JavaScript statis** di `public/`, disajikan oleh `@fastify/static`. Tidak ada framework dan tidak ada build step frontend.

## Consequences

Lebih mudah:

- Satu proses, satu build (TypeScript server saja).
- Demo cepat dan dependensi minimal.
- Fokus tetap pada API dan kualitas backend.

Lebih sulit:

- Render dan state ditulis manual.
- Tidak ada komponen reusable.
- Jika UI tumbuh besar, perlu migrasi.

Diterima karena UI v1 hanya satu halaman dengan satu alur.

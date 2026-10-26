# ADR-0002: Gunakan Fastify dan TypeScript

- **Status:** Accepted
- **Date:** 2026-10-26

## Context

Proyek portofolio ini butuh REST API kecil yang cepat, mudah diuji, dan menunjukkan praktik engineering yang rapi. Kandidat utama: Express, Fastify, Hono, NestJS.

Batasan:

- Harus berjalan di Node.js 24 LTS.
- Harus punya validasi input dan error handling yang jelas.
- Harus ringan, tanpa banyak abstraksi.
- Harus mudah diuji tanpa server nyata.
- Harus mendukung static file untuk UI sederhana.

## Options Considered

1. **Express:** paling dikenal, ekosistem besar. Kekurangan: tidak punya validasi/serialisasi bawaan, error handling manual, performa lebih rendah, dan TypeScript terasa ditempel.
2. **Fastify:** cepat, schema-first, plugin resmi untuk static/CORS, `inject()` untuk test, logging pino bawaan. Kekurangan: kurva belajar plugin/encapsulation, ekosistem lebih kecil dari Express.
3. **Hono:** sangat ringan dan modern, bagus untuk edge. Kekurangan: ekosistem Node/plugin static dan pola test di Node kurang matang untuk kebutuhan ini.
4. **NestJS:** struktur lengkap dan opinionated. Kekurangan: terlalu berat untuk satu service kecil; banyak boilerplate dan konsep yang tidak sepadan.

## Decision

Gunakan **Fastify 5** dengan **TypeScript 5.9.3**. TypeScript 5.9 dipilih, bukan 7.x, karena `typescript-eslint@8` masih mensyaratkan `typescript <6.1.0`; memakai TS 7 akan mematikan lint.

## Consequences

Lebih mudah:

- Validasi dan error handling konsisten.
- Test route memakai `app.inject()` tanpa port nyata.
- Plugin resmi untuk static dan CORS.
- Performa dan logging baik secara default.

Lebih sulit:

- Perlu memahami encapsulation dan urutan registrasi plugin.
- Sebagian contoh Express tidak langsung berlaku.
- Versi TypeScript harus dijaga agar kompatibel dengan tooling lint.

Diterima sebagai trade-off karena ukuran proyek kecil dan kebutuhan utamanya adalah kualitas serta keterujian.

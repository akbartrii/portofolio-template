# API Spec — BMKG Weather Aggregator

Base URL lokal: `http://localhost:8080`

Versi API: `v1` secara implisit pada path `/api`. Perubahan breaking akan memakai `/api/v2`.

## 1. Konvensi

- Format: JSON, `Content-Type: application/json; charset=utf-8`.
- Semua respons sukses memakai `{ data, meta }`.
- Semua respons error memakai `{ error: { code, message, requestId } }`.
- Setiap respons memuat header `x-request-id`.
- Waktu memakai ISO 8601. Waktu lokal menyertakan offset (`+07:00`).
- Tidak ada autentikasi pada v1.
- Rate limiting belum diterapkan pada v1.

## 2. Endpoint

| Method | Path | Deskripsi |
|---|---|---|
| GET | `/health` | Status proses |
| GET | `/api/cities` | Daftar kota yang didukung |
| GET | `/api/weather` | Prakiraan berdasarkan `city` atau `adm4` |
| GET | `/` | Halaman web |

## 3. `GET /health`

Respons `200`:

```json
{
  "status": "ok",
  "uptimeSeconds": 42,
  "version": "0.1.0"
}
```

Tidak memanggil BMKG. Dipakai untuk liveness/readiness dan Docker healthcheck.

## 4. `GET /api/cities`

Respons `200`:

```json
{
  "data": [
    {
      "slug": "jakarta-kemayoran",
      "name": "Jakarta — Kemayoran",
      "province": "DKI Jakarta",
      "city": "Kota Adm. Jakarta Pusat",
      "district": "Kemayoran",
      "adm4": "31.71.03.1001"
    }
  ],
  "meta": {
    "count": 15
  }
}
```

Aturan:

- Urutan stabil sesuai katalog.
- `adm4` selalu terverifikasi.
- Tidak ada parameter query yang didukung pada v1.

## 5. `GET /api/weather`

### 5.1 Parameter

| Nama | Tipe | Wajib | Aturan |
|---|---|---|---|
| `city` | string | salah satu | Slug katalog, maks 64 karakter, case-insensitive |
| `adm4` | string | salah satu | Regex `^\d{2}\.\d{2}\.\d{2}\.\d{4}$` |

Tepat satu parameter harus diisi. Jika keduanya diisi -> `400 AMBIGUOUS_QUERY`. Jika tidak ada -> `400 INVALID_QUERY`.

### 5.2 Contoh request

```
GET /api/weather?city=bandung
GET /api/weather?adm4=31.71.03.1001
```

### 5.3 Respons sukses `200`

```json
{
  "data": {
    "location": {
      "adm4": "31.71.03.1001",
      "province": "DKI Jakarta",
      "city": "Kota Adm. Jakarta Pusat",
      "district": "Kemayoran",
      "village": "Kemayoran",
      "latitude": -6.1647214778,
      "longitude": 106.8453837867,
      "timezone": "Asia/Jakarta"
    },
    "forecasts": [
      {
        "datetimeUtc": "2026-10-26T08:00:00.000Z",
        "datetimeLocal": "2026-10-26T15:00:00+07:00",
        "temperatureC": 30,
        "humidityPct": 67,
        "precipitationMm": 1.2,
        "cloudCoverPct": 100,
        "weatherId": 61,
        "descriptionId": "Hujan Ringan",
        "descriptionEn": "Light Rain",
        "windSpeedKmh": 9,
        "windDirection": "W",
        "windDirectionTo": "E",
        "windDegree": 310,
        "iconUrl": "https://api-apps.bmkg.go.id/storage/icon/cuaca/hujan ringan-am.svg"
      }
    ]
  },
  "meta": {
    "cached": false,
    "fetchedAt": "2026-10-26T07:55:00.000Z",
    "source": "BMKG",
    "requestId": "req-7"
  }
}
```

Field prakiraan yang tidak tersedia dari BMKG bernilai `null`, bukan dihilangkan. `forecasts` selalu array (tidak pernah `null`).

### 5.4 Header respons

| Header | Nilai | Kapan |
|---|---|---|
| `x-request-id` | ID request | Semua respons |
| `Cache-Control` | `public, max-age=300` | Sukses `GET /api/weather` |

### 5.5 Kode status

| Status | Kondisi |
|---|---|
| 200 | Sukses |
| 400 | Query tidak valid atau ambigu |
| 404 | Slug tidak ada di katalog, atau `adm4` tidak dikenal BMKG |
| 502 | Upstream gagal, respons tidak valid, atau data kosong |
| 504 | Upstream timeout |
| 500 | Error tak terduga |

## 6. Error

### 6.1 Bentuk

```json
{
  "error": {
    "code": "INVALID_QUERY",
    "message": "Parameter city atau adm4 harus diisi tepat satu.",
    "requestId": "req-8"
  }
}
```

### 6.2 Daftar kode

| Kode | Status | Penyebab | Contoh pesan |
|---|---|---|---|
| `INVALID_QUERY` | 400 | Parameter hilang, kosong, terlalu panjang, atau format `adm4` salah | `Format adm4 tidak valid.` |
| `AMBIGUOUS_QUERY` | 400 | `city` dan `adm4` diisi bersamaan | `Gunakan salah satu: city atau adm4.` |
| `CITY_NOT_FOUND` | 404 | Slug tidak ada di katalog | `Kota tidak ditemukan.` |
| `ADM4_NOT_FOUND` | 404 | BMKG mengembalikan 404 | `Kode wilayah tidak ditemukan di BMKG.` |
| `UPSTREAM_ERROR` | 502 | BMKG 4xx/5xx, body invalid, atau data kosong | `Gagal mengambil data cuaca.` |
| `UPSTREAM_TIMEOUT` | 504 | Timeout setelah retry | `Layanan cuaca tidak merespons tepat waktu.` |
| `INTERNAL_ERROR` | 500 | Bug atau error tak terduga | `Terjadi kesalahan internal.` |

Pesan error tidak memuat stack trace, URL internal, atau body upstream.

## 7. Contoh error

### 7.1 Tidak ada parameter

```
GET /api/weather
```

```json
{
  "error": {
    "code": "INVALID_QUERY",
    "message": "Parameter city atau adm4 harus diisi tepat satu.",
    "requestId": "req-9"
  }
}
```

### 7.2 Dua parameter

```
GET /api/weather?city=bandung&adm4=32.73.01.1001
```

```json
{
  "error": {
    "code": "AMBIGUOUS_QUERY",
    "message": "Gunakan salah satu: city atau adm4.",
    "requestId": "req-10"
  }
}
```

### 7.3 Kota tidak dikenal

```
GET /api/weather?city=atlantis
```

```json
{
  "error": {
    "code": "CITY_NOT_FOUND",
    "message": "Kota tidak ditemukan.",
    "requestId": "req-11"
  }
}
```

### 7.4 `adm4` format salah

```
GET /api/weather?adm4=3171031001
```

```json
{
  "error": {
    "code": "INVALID_QUERY",
    "message": "Format adm4 tidak valid.",
    "requestId": "req-12"
  }
}
```

### 7.5 Upstream timeout

```json
{
  "error": {
    "code": "UPSTREAM_TIMEOUT",
    "message": "Layanan cuaca tidak merespons tepat waktu.",
    "requestId": "req-13"
  }
}
```

## 8. Semantik cache

| Field | Arti |
|---|---|
| `meta.cached` | `true` jika data diambil dari cache, `false` jika baru dari BMKG |
| `meta.fetchedAt` | Waktu data diambil dari BMKG (bukan waktu request) |
| `meta.source` | Selalu `BMKG` pada v1 |
| `Cache-Control` | Batas cache klien/proxy, default 300 detik |

TTL internal default 600 detik. Klien tidak dapat mengubah TTL lewat query.

## 9. Kompatibilitas

- Menambah field baru tidak dianggap breaking.
- Menghapus atau mengganti nama field dianggap breaking.
- Menambah kode error baru tidak dianggap breaking.
- Perubahan breaking memerlukan `/api/v2`.

## 10. Catatan implementasi

- Spesifikasi ini adalah sumber kebenaran kontrak HTTP.
- Test integrasi wajib mencakup setiap kode error di tabel 6.2.
- Contoh di dokumen ini memakai data nyata BMKG; waktu dan nilai akan berubah saat runtime.

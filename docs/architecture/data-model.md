# Data Model — BMKG Weather Aggregator

Dokumen ini menjelaskan model data internal, pemetaan dari respons mentah BMKG, dan invarian yang harus dijaga. Tidak ada database pada v1; "model" di sini adalah bentuk objek di memori dan JSON API.

## 1. Model internal

### 1.1 `City`

```ts
interface City {
  slug: string;       // unik, lowercase, a-z0-9-
  name: string;       // label tampilan
  province: string;
  city: string;       // kota/kabupaten
  district: string;   // kecamatan
  adm4: string;       // XX.XX.XX.XXXX, unik
}
```

Invarian:

- `slug` unik dan cocok `^[a-z0-9-]+$`.
- `adm4` unik dan cocok `^\d{2}\.\d{2}\.\d{2}\.\d{4}$`.
- Katalog dibekukan saat runtime.

### 1.2 `Location`

```ts
interface Location {
  adm4: string;
  province: string;
  city: string;
  district: string;
  village: string;
  latitude: number;   // derajat
  longitude: number;  // derajat
  timezone: string;   // IANA, mis. Asia/Jakarta
}
```

### 1.3 `Forecast`

```ts
interface Forecast {
  datetimeUtc: string;        // ISO 8601 UTC, mis. 2026-10-26T08:00:00.000Z
  datetimeLocal: string;      // ISO 8601 dengan offset, mis. 2026-10-26T15:00:00+07:00
  temperatureC: number;       // derajat Celsius
  humidityPct: number | null; // 0-100
  precipitationMm: number | null;
  cloudCoverPct: number | null; // 0-100
  weatherId: number | null;     // kode cuaca BMKG
  descriptionId: string | null;
  descriptionEn: string | null;
  windSpeedKmh: number | null;
  windDirection: string | null;   // arah asal, mis. W
  windDirectionTo: string | null; // arah tujuan, mis. E
  windDegree: number | null;      // 0-360
  iconUrl: string | null;
}
```

### 1.4 `WeatherResult`

```ts
interface WeatherResult {
  location: Location;
  forecasts: Forecast[];
}
```

Invarian:

- `forecasts.length >= 1` untuk hasil yang dianggap sukses.
- `forecasts` terurut naik berdasarkan `datetimeUtc`.
- Tidak ada `datetimeUtc` duplikat.
- `temperatureC` selalu angka; field lain boleh `null`.

### 1.5 Metadata respons

```ts
interface WeatherMeta {
  cached: boolean;
  fetchedAt: string; // ISO 8601
  source: 'BMKG';
  requestId: string;
}
```

## 2. Bentuk mentah BMKG

Endpoint: `GET {BMKG_BASE_URL}/publik/prakiraan-cuaca?adm4=<kode>`

Bentuk ringkas:

```json
{
  "lokasi": {
    "adm1": "31",
    "adm2": "31.71",
    "adm3": "31.71.03",
    "adm4": "31.71.03.1001",
    "provinsi": "DKI Jakarta",
    "kotkab": "Kota Adm. Jakarta Pusat",
    "kecamatan": "Kemayoran",
    "desa": "Kemayoran",
    "lon": 106.8453837867,
    "lat": -6.1647214778,
    "timezone": "Asia/Jakarta"
  },
  "data": [
    {
      "lokasi": { "...": "..." },
      "cuaca": [
        [
          {
            "datetime": "2026-10-26T08:00:00Z",
            "t": 30,
            "tcc": 100,
            "tp": 1.2,
            "weather": 61,
            "weather_desc": "Hujan Ringan",
            "weather_desc_en": "Light Rain",
            "wd_deg": 310,
            "wd": "W",
            "wd_to": "E",
            "ws": 9,
            "hu": 67,
            "vs": null,
            "vs_text": null,
            "time_index": "6-7",
            "analysis_date": "2026-10-26T00:00:00",
            "image": "https://api-apps.bmkg.go.id/storage/icon/cuaca/hujan ringan-am.svg",
            "utc_datetime": "2026-10-26 08:00:00",
            "local_datetime": "2026-10-26 15:00:00"
          }
        ]
      ]
    }
  ]
}
```

Catatan:

- `data` berisi satu elemen pada pengujian.
- `data[0].cuaca` adalah **array of arrays** (dikelompokkan per blok waktu), sehingga harus di-flatten.
- `lokasi` juga ada di dalam `data[0]`; gunakan yang di root untuk konsistensi.
- `vs` dan `vs_text` dapat `null`.

## 3. Pemetaan field

| Internal | BMKG | Transformasi |
|---|---|---|
| `location.adm4` | `lokasi.adm4` | Langsung |
| `location.province` | `lokasi.provinsi` | Langsung |
| `location.city` | `lokasi.kotkab` | Langsung |
| `location.district` | `lokasi.kecamatan` | Langsung |
| `location.village` | `lokasi.desa` | Langsung |
| `location.latitude` | `lokasi.lat` | Angka |
| `location.longitude` | `lokasi.lon` | Angka |
| `location.timezone` | `lokasi.timezone` | Langsung |
| `forecast.datetimeUtc` | `datetime` | Normalisasi ke ISO UTC |
| `forecast.datetimeLocal` | `local_datetime` | Tambahkan offset dari `timezone` |
| `forecast.temperatureC` | `t` | Angka; wajib |
| `forecast.humidityPct` | `hu` | Angka atau `null` |
| `forecast.precipitationMm` | `tp` | Angka atau `null` |
| `forecast.cloudCoverPct` | `tcc` | Angka atau `null` |
| `forecast.weatherId` | `weather` | Angka atau `null` |
| `forecast.descriptionId` | `weather_desc` | String atau `null` |
| `forecast.descriptionEn` | `weather_desc_en` | String atau `null` |
| `forecast.windSpeedKmh` | `ws` | Angka atau `null` |
| `forecast.windDirection` | `wd` | String atau `null` |
| `forecast.windDirectionTo` | `wd_to` | String atau `null` |
| `forecast.windDegree` | `wd_deg` | Angka atau `null` |
| `forecast.iconUrl` | `image` | String atau `null` |

Tidak dipakai v1: `vs`, `vs_text`, `time_index`, `analysis_date`, `utc_datetime`.

## 4. Algoritma normalisasi

```
function normalizeForecast(raw):
  location = mapLocation(raw.lokasi)
  rows = flatten(raw.data[*].cuaca[*][*])
  forecasts = []
  seen = set()

  for row in rows:
    if row.datetime is not valid datetime: continue
    if typeof row.t is not number: continue
    key = toUtcIso(row.datetime)
    if key in seen: continue
    seen.add(key)
    forecasts.append({
      datetimeUtc: key,
      datetimeLocal: localIso(row.local_datetime, location.timezone),
      temperatureC: row.t,
      humidityPct: numOrNull(row.hu),
      precipitationMm: numOrNull(row.tp),
      cloudCoverPct: numOrNull(row.tcc),
      weatherId: numOrNull(row.weather),
      descriptionId: strOrNull(row.weather_desc),
      descriptionEn: strOrNull(row.weather_desc_en),
      windSpeedKmh: numOrNull(row.ws),
      windDirection: strOrNull(row.wd),
      windDirectionTo: strOrNull(row.wd_to),
      windDegree: numOrNull(row.wd_deg),
      iconUrl: strOrNull(row.image)
    })

  sort forecasts by datetimeUtc ascending
  if forecasts is empty: throw UpstreamError
  return { location, forecasts }
```

Detail `datetimeLocal`:

- BMKG mengirim `local_datetime` tanpa offset, mis. `2026-10-26 15:00:00`.
- Ambil offset dari `timezone` (`Asia/Jakarta` -> `+07:00`) menggunakan `Intl.DateTimeFormat`.
- Jika offset tidak dapat ditentukan, gunakan `+07:00` sebagai fallback dan catat warning.

## 5. Contoh transformasi

Input (ringkas):

```json
{
  "datetime": "2026-10-26T08:00:00Z",
  "t": 30,
  "hu": 67,
  "tp": 1.2,
  "tcc": 100,
  "weather": 61,
  "weather_desc": "Hujan Ringan",
  "weather_desc_en": "Light Rain",
  "ws": 9,
  "wd": "W",
  "wd_to": "E",
  "wd_deg": 310,
  "image": "https://api-apps.bmkg.go.id/storage/icon/cuaca/hujan ringan-am.svg",
  "local_datetime": "2026-10-26 15:00:00"
}
```

Output:

```json
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
```

## 6. Bentuk cache

```ts
interface CacheEntry {
  result: WeatherResult;
  fetchedAt: string; // ISO 8601
}
```

Key: `weather:<adm4>`.

Menyimpan `fetchedAt` di dalam entri memastikan `meta.fetchedAt` tidak berubah saat cache hit.

## 7. Validasi schema zod

Prinsip:

- `lokasi` wajib, field yang dipakai wajib.
- `data` wajib array minimal 1.
- `cuaca` wajib array; elemen dalam boleh array atau objek (defensif).
- Field angka memakai `.nullable().optional()` lalu dinormalisasi ke `null`.
- Jika schema gagal, lempar `UpstreamError` dan jangan cache.

## 8. Invarian yang diuji

| Invarian | Test |
|---|---|
| `forecasts` terurut naik | Unit normalizer |
| Duplikat `datetimeUtc` dibuang | Unit normalizer |
| Entri tanpa suhu dibuang | Unit normalizer |
| Semua field opsional menjadi `null`, bukan `undefined` | Unit normalizer |
| Hasil kosong melempar error | Unit normalizer |
| `fetchedAt` stabil pada cache hit | Unit service |
| Key cache memakai format `weather:<adm4>` | Unit service |
| `adm4` format regex | Unit route |

# Pyro-sense — IntelliFire

End-to-end **wildfire risk monitoring**: a **Flask** web app, optional **ESP32 + DHT22** telemetry, **Open-Meteo** weather, and a **Random Forest** classifier trained on the **UCI Forest Fires** dataset with **synthetic sensor columns** correlated to fire labels. The live dashboard polls the server every **5 seconds** and shows risk, sensor cards, weather cards, a trend chart (**Chart.js**), and a high-risk banner.

---

## Table of contents

1. [Architecture](#architecture)
2. [Tech stack](#tech-stack)
3. [Repository layout (every file)](#repository-layout-every-file)
4. [Requirements & installation](#requirements--installation)
5. [Run the server](#run-the-server)
6. [Configuration (`config.py`)](#configuration-configpy)
7. [Authentication](#authentication)
8. [HTTP API (all routes)](#http-api-all-routes)
9. [JSON payloads](#json-payloads)
10. [Machine learning (`model.py`)](#machine-learning-modelpy)
11. [Threshold model (`models/risk_model.py`)](#threshold-model-modelsrisk_modelpy)
12. [Weather (`satellite.py`)](#weather-satellitepy)
13. [Frontend (templates & static)](#frontend-templates--static)
14. [ESP32 firmware](#esp32-firmware)
15. [Optional: Streamlit components](#optional-streamlit-components)
16. [Optional: data simulator](#optional-data-simulator)
17. [Dataset](#dataset)
18. [Troubleshooting](#troubleshooting)
19. [Security (production)](#security-production)
20. [License](#license)

---

## Architecture

```
┌─────────────┐     POST /data (JSON)      ┌──────────────────┐
│  ESP32      │ ─────────────────────────► │  Flask (app.py)  │
│  DHT22      │                            │  • merges sensor │
└─────────────┘                            │  • get_weather() │
                                           │  • predict_risk()│
┌─────────────┐     GET forecast           └────────┬─────────┘
│ Open-Meteo  │ ◄─────────────────────────────────┤
└─────────────┘                                     │
                                                    │ GET /latest (JSON)
                                                    ▼
                                           ┌──────────────────┐
                                           │  Browser         │
                                           │  dashboard.html  │
                                           │  + script.js     │
                                           └──────────────────┘

Flow: / (intro) → /login → /dashboard (session required)
```

On startup, `model.py` **trains the Random Forest automatically** (reads CSV, augments, balances classes, fits scaler + classifier).

---

## Tech stack

| Layer | Technology |
|--------|------------|
| Backend | Python, Flask, flask-cors |
| ML | scikit-learn `RandomForestClassifier`, `StandardScaler`, class balancing via `resample` |
| Data | pandas, numpy, CSV (`data/forestfires.csv`) |
| Weather | `requests` → Open-Meteo REST API |
| Dashboard UI | HTML templates, CSS, vanilla JS, Chart.js (CDN) |
| Intro | Three.js WebGL (`static/js/earth_intro.js`) |

---

## Repository layout (every file)

```
pyro-sense/
└── IntelliFire/
    ├── app.py                 # Flask app: routes, session auth, sensor state, calls model + weather
    ├── config.py              # Host/port, default lat/lon, risk band constants, NASA_TOKEN placeholder
    ├── model.py               # Load UCI CSV, augment synthetic sensors, train RF, predict_risk()
    ├── satellite.py           # Open-Meteo client; fallback cache on failure
    ├── requirements.txt       # Python dependencies (see below)
    ├── README.md              # Short pointer — full docs are in the repo root README
    │
    ├── data/
    │   └── forestfires.csv    # UCI Forest Fires dataset (517 rows; `area` → fire label)
    │
    ├── templates/
    │   ├── intro.html         # Full-screen Three.js cinematic intro → link to login
    │   ├── login.html         # Star-field themed login form (inline CSS)
    │   └── dashboard.html     # Main monitoring UI; loads /static/style.css + script.js
    │
    ├── static/
    │   ├── style.css          # Dark IoT dashboard theme
    │   ├── script.js          # Poll /latest every 5s; Chart.js risk trend; simulate buttons
    │   └── js/
    │       └── earth_intro.js # Earth + fire particle intro scene
    │
    ├── assets/
    │   └── style.css          # Extra stylesheet asset (not required for default Flask templates)
    │
    ├── components/            # Streamlit + Plotly widgets (optional alternate UI — not imported by app.py)
    │   ├── risk_meter.py      # Plotly gauge for risk score
    │   ├── sensor_cards.py    # Metric cards with threshold colors
    │   ├── charts.py          # Multi-panel Plotly time-series
    │   └── alert_box.py       # High-risk markdown banner
    │
    ├── models/
    │   └── risk_model.py      # Weighted threshold scoring + human-readable reasons (0–100)
    │
    ├── utils/
    │   └── data_simulator.py  # Random / time-series simulated readings (standalone utility)
    │
    └── esp32/
        └── intellifire.ino    # WiFi + DHT22 → HTTP POST JSON every 5s
```

---

## Requirements & installation

- **Python 3.10+** recommended (3.8+ may work if dependencies install cleanly).
- **ESP32 + DHT22** optional; default sensor values are used until `/data` updates them.
- **Arduino IDE** with ESP32 core + libraries (for firmware only).

```bash
cd IntelliFire
python -m venv .venv
```

**Windows:** `.venv\Scripts\activate`  
**macOS/Linux:** `source .venv/bin/activate`

```bash
pip install -r requirements.txt
```

**Current `requirements.txt`:** `flask`, `flask-cors`, `scikit-learn`, `numpy`, `pandas`, `requests`.

---

## Run the server

```bash
cd IntelliFire
python app.py
```

Console prints IntelliFire banner and model training logs. Open:

**http://localhost:5001**

(Default bind: `0.0.0.0:5001` from `config.py`.)

---

## Configuration (`config.py`)

| Symbol | Role |
|--------|------|
| `SERVER_HOST` | Bind address (default `0.0.0.0`) |
| `SERVER_PORT` | Port (default **5001**) — must match ESP32 `SERVER_PORT` |
| `UPDATE_INTERVAL` | Documented ESP32 cadence (seconds); sketch uses its own `SEND_INTERVAL` |
| `DEFAULT_LAT`, `DEFAULT_LON` | Open-Meteo location (default near Coimbatore, India) |
| `RISK_LOW_MAX`, `RISK_MEDIUM_MAX` | Documented bands; **live labels** in `model.predict_risk` use 40 / 70 |
| `NASA_TOKEN` | Placeholder for future Earthdata integration (not used by current `satellite.py`) |
| `SYNTHETIC_SAMPLES` | Reserved in config; training uses augmented UCI rows + resampling (see `model.py`) |
| `RANDOM_SEED` | Reproducible augmentation and resampling |

---

## Authentication

- **`VALID_USERS`** in `app.py`: default **`admin` / `admin123`**.
- **`@login_required`** protects `/dashboard`.
- **`/data`** has **no** session check (intended for devices).
- Flask **`secret_key`** is set in `app.py` — **change before production**.

---

## HTTP API (all routes)

| Method | Path | Login | Description |
|--------|------|-------|-------------|
| GET | `/` | No | Cinematic intro (`intro.html` + Three.js) |
| GET, POST | `/login` | No | Form login; sets session; redirects to dashboard if already logged in |
| GET | `/logout` | No | Clears session → redirect to login |
| GET | `/dashboard` | **Yes** | Main dashboard HTML |
| POST | `/data` | No | JSON body: any **numeric** keys update `_sensor` dict; returns `risk`, `level`, `updated` keys |
| GET | `/latest` | No | Full snapshot: `sensor`, `weather`, `risk`, `level`, `timestamp` |
| GET | `/simulate_high` | No | Random “dangerous” sensor values → JSON with new risk |
| GET | `/simulate_low` | No | Random “safe” sensor values → JSON with new risk |

**Server-side alert:** if computed `risk > 70`, `app.py` prints a **HIGH FIRE RISK** line to the console.

---

## JSON payloads

### `POST /data`

Send JSON with float-friendly values. Example from ESP32:

```json
{ "temperature": 28.4, "humidity": 71.2 }
```

Any top-level key whose value converts to `float` is merged into server state (e.g. `soil`, `co`, `xylem`). Non-numeric fields are skipped. Empty or invalid payload → `400`.

### `GET /latest` response shape

```json
{
  "sensor": {
    "temperature": 30.0,
    "humidity": 60.0,
    "soil": 50.0,
    "co": 100.0,
    "xylem": 50.0
  },
  "weather": {
    "temperature": 29.5,
    "humidity": 55.0,
    "rainfall": 0.0,
    "wind": 12.0
  },
  "risk": 42.3,
  "level": "Medium",
  "timestamp": "2026-03-30T12:00:00.000000"
}
```

`level` is **`Low`**, **`Medium`**, or **`High`** (from `model.py`, thresholds at 40 and 70).

---

## Machine learning (`model.py`)

1. **Loads** `data/forestfires.csv`. If missing, raises `FileNotFoundError` with a `curl` hint to the UCI URL.
2. **Label:** `fire = (area > 0)`.
3. **Maps** UCI columns: `temp` → `weather_temp`, `RH` → `weather_humidity`, `rain` → `rainfall`, `wind` → `wind`.
4. **Augments** each row with synthetic `sensor_temp`, `sensor_humidity`, `soil`, `co`, `xylem` drawn from different distributions for fire vs non-fire rows.
5. **Balances** classes by upsampling the minority class to match the majority, then shuffles.
6. **Trains** `StandardScaler` + `RandomForestClassifier` (200 trees, `max_depth=12`, etc.).
7. **`predict_risk(sensor_data, weather_data)`** builds a 9-feature row (defaults for missing keys), scales, returns **(score 0–100, level string)**.

**Feature order:**  
`sensor_temp`, `sensor_humidity`, `soil`, `co`, `xylem`, `weather_temp`, `weather_humidity`, `rainfall`, `wind`.

---

## Threshold model (`models/risk_model.py`)

Separate **interpretable** model: weighted contributions from **humidity**, **soil temperature**, **CO**, **xylem stress** (0–1 scale in formulas; dashboard stores xylem on a 0–100 style scale in other places — align inputs if you call this module directly). Returns `risk_score`, `risk_level` (`Low` / `Moderate` / `High`), and `risk_reason`. **Not** wired into `app.py` (the live API uses the Random Forest in `model.py`). Useful for Streamlit dashboards or explanations.

---

## Weather (`satellite.py`)

Despite the filename, the implementation fetches **Open-Meteo** `v1/forecast` **current** fields: `temperature_2m`, `relative_humidity_2m`, `precipitation`, `windspeed_10m`. On error, returns the **last successful** cached dict. No API key required.

**Upgrade path** (commented in file): NASA Earthdata, Google Earth Engine, etc.

---

## Frontend (templates & static)

| Asset | Purpose |
|--------|---------|
| `intro.html` | Loads Three.js from CDN + `static/js/earth_intro.js`; overlay logo; navigate to `/login` |
| `login.html` | POST username/password to `/login`; shows error on failure |
| `dashboard.html` | Risk gauge bar, badge, alert banner, ESP32 + weather cards, Chart.js canvas, demo buttons |
| `static/style.css` | Layout, dark theme, card states (`safe` / `warn` / `danger`) |
| `static/script.js` | `fetch("/latest")` every **5000 ms**; updates DOM; `buildReason()` heuristic text; `simulateHigh` / `simulateLow` |

**Dashboard risk factors (JS):** client-side explanations use thresholds on temp, humidity, soil, CO, xylem, and weather (not the Python `risk_model`).

---

## ESP32 firmware

**File:** `IntelliFire/esp32/intellifire.ino`

**Wiring (from sketch comments):**

| DHT22 | ESP32 |
|-------|--------|
| VCC | 3.3V |
| GND | GND |
| DATA | GPIO **4** (configurable via `DHT_PIN`) |
| DATA | 10kΩ pull-up to 3.3V |

**Libraries:** DHT (Adafruit), Adafruit Unified Sensor, ArduinoJson v6, `WiFi`, `HTTPClient` (ESP32 core).

**Configure:**

- `WIFI_SSID`, `WIFI_PASSWORD`
- `SERVER_IP` — your PC’s LAN IP (`ipconfig` on Windows)
- `SERVER_PORT` — **5001** by default to match Flask

**Behavior:** Serial **115200** baud; every **5000 ms** reads DHT22; POSTs `{"temperature":…,"humidity":…}` to `http://<SERVER_IP>:<PORT>/data`. Reconnects WiFi on loss.

---

## Optional: Streamlit components

Files under `components/` use **`streamlit`** and **`plotly`**. They are **not** imported by the Flask `app.py`. To experiment:

```bash
pip install streamlit plotly
```

Then build a separate `streamlit run` entry script that imports `risk_meter`, `sensor_cards`, `charts`, `alert_box`, and feeds them data from `risk_model` or your own pipeline.

---

## Optional: data simulator

`utils/data_simulator.py` provides:

- `generate_live_reading()` — one random dict (`humidity`, `soil_temp`, `co_level`, `xylem_stress`, `timestamp`)
- `generate_time_series(hours=24, interval_minutes=15)` — `pandas` DataFrame of synthetic history (sine + noise)
- `inject_high_risk_scenario(reading)` — overrides a dict with extreme values for demos/tests

Use in notebooks, tests, or a custom Streamlit app. **Not** used by the Flask server out of the box.

---

## Dataset

- **Source:** UCI Machine Learning Repository — Forest Fires (Montesinho park, Portugal).
- **Local path:** `IntelliFire/data/forestfires.csv`
- **Columns used in training pipeline:** includes `temp`, `RH`, `wind`, `rain`, `area` (see `model.py` for exact renaming).

If the file is missing, download from UCI (the error message in `model.py` includes the archive URL path).

---

## Troubleshooting

| Issue | What to check |
|--------|----------------|
| `ModuleNotFoundError: pandas` | `pip install -r requirements.txt` |
| `FileNotFoundError` for CSV | Ensure `data/forestfires.csv` exists |
| Dashboard empty / Offline | Flask running? Port **5001**? Logged in? |
| ESP32 cannot POST | Same WiFi as PC? Firewall allows inbound on port? Correct `SERVER_IP`? |
| DHT22 `nan` | Wiring, pull-up, power, GPIO pin |
| Weather always same | Open-Meteo failure — check network; server uses last good cache |

---

## Security (production)

- Replace hardcoded users and `secret_key`.
- Do not expose `/data` publicly without **device authentication** or **VPN**.
- Use **HTTPS** and a production WSGI server (e.g. gunicorn + reverse proxy).
- Rotate credentials and monitor logs.

---

## License

The bundled `IntelliFire/README.md` previously mentioned MIT; **add a `LICENSE` file** with your chosen terms if you redistribute the project.

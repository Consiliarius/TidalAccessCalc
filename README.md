# Tidal Access Calculator — Langstone Harbour

A browser-based tool for predicting when there is sufficient water to depart and arrive at a drying mooring in the Broom Channel of Langstone Harbour, Hampshire.

The tool computes access windows by comparing predicted tidal heights against a threshold derived from the mooring's drying height, the vessel's draught, and a configurable safety margin. It displays these windows visually, and can export them as calendar events (.ics) for import into Google Calendar, Outlook, or Apple Calendar.

## Getting Started

Open the tool at **[https://consiliarius.github.io/TidalAccessCalc/](https://consiliarius.github.io/TidalAccessCalc/)** or clone the repository and open `index.html` locally.

No installation is required for the dashboard itself — it runs entirely in the browser with no server dependencies. The optional Python script (`fetch_tides.py`) requires Python 3.6+ with no additional packages.

## Configuration

Before loading tidal data, set the parameters in the Configuration panel:

- **Draught (m):** The vessel's draught. Default 1.0m.
- **Safety Margin (m):** Additional clearance above the draught. Default 0.3m.
- **Drying Height Estimate (m above CD):** The height above Chart Datum at which the mooring dries. This is the key unknown — start with an estimate and refine it using observations (see below).
- **Min. Window (minutes):** Access windows shorter than this are discarded. Default 30 minutes. Prevents useless slivers on neap tides where HW barely crosses the threshold.

Settings are saved to browser local storage and persist between sessions.

The access threshold is calculated as: `drying height + draught + safety margin`. The boat can depart or arrive whenever the predicted tidal height exceeds this threshold.

## Tidal Data Sources

The tool accepts tidal data from three sources. All data is converted to UTC internally; the display timezone (UTC or BST) can be toggled independently.

### 1. Royal Navy KHM — 30 days (recommended for regular use)

**No API key required.** This is the simplest method for routine use.

1. Visit the [KHM Portsmouth Tide Tables](https://www.royalnavy.mod.uk/khm/portsmouth/port-information/tide-tables) page.
2. Select the table rows covering the period of interest (up to ~30 days ahead).
3. Copy the data (Ctrl+C / Cmd+C).
4. In the dashboard, select the **Royal Navy KHM (30d)** tab.
5. Paste the data into the text area.
6. Ensure the timezone selector is set correctly — KHM times are in local time (BST during summer).
7. Click **Load KHM Data**.

The KHM table has 13 tab-separated columns: Date, Sunrise, HW₁ time, HW₁ height, LW₁ time, LW₁ height, HW₂ time, HW₂ height, LW₂ time, LW₂ height, Sunset, Range, Spring%.

This source uses official UKHO data published by the King's Harbour Master for Portsmouth. Because Langstone Harbour is a secondary port, the dashboard automatically applies corrections derived from a comparison of Admiralty predictions for both stations: HW times are shifted +9 minutes and HW heights increased by +0.24m. LW times and heights require no significant correction. These adjustments are noted in the status message after loading.

### 2. UKHO Admiralty API — 7 days (highest precision)

Requires a free API key and Python.

#### Obtaining an API key

1. Visit the [ADMIRALTY Developer Portal](https://developer.admiralty.co.uk/).
2. Create an account (free).
3. Once logged in, go to **Products** and subscribe to the **UK Tidal API - Discovery** product.
4. After subscribing, find your subscription key on your **Profile** page — click **Show** next to the primary or secondary key and copy it.

The Discovery tier is free and provides 7 days of HW/LW predictions for 607 UK tidal stations.

#### Fetching tidal data

Download [`fetch_tides.py`](https://github.com/Consiliarius/TidalAccessCalc/blob/main/fetch_tides.py) and run:

```
python fetch_tides.py --source ukho YOUR_API_KEY
```

This fetches 7 days of tidal events for Langstone Harbour (station 0066) and writes `tidal_data.json` to the current directory.

Optional arguments:

```
python fetch_tides.py --source ukho YOUR_API_KEY 0065 7
                                     ^station    ^days (1-7)
```

Station 0065 is Portsmouth; 0066 is Langstone Harbour (default).

#### Loading into the dashboard

1. Select the **UKHO / JSON File** tab.
2. Either drag and drop `tidal_data.json` onto the upload area, or click to browse.
3. Alternatively, click "Paste JSON instead" and paste the file contents.

### 3. Harmonic Predictions — up to 365 days (approximate)

For longer-range planning, the fetch script can compute tidal predictions locally using harmonic constants for Portsmouth. No API key is needed.

```
python fetch_tides.py --source harmonic 90
python fetch_tides.py --source harmonic 365 2026-01-01
```

This generates `tidal_data.json` covering the specified number of days from the start date (defaults to today).

**Accuracy:** HW heights are typically within ±0.2m and HW times within ±30 minutes of official UKHO predictions. LW heights are less accurate (may overestimate by 0.5–1.0m). The spring/neap cycle and the double high water stand characteristic of Portsmouth/Langstone are reproduced. Use the UKHO API or KHM data for the current week; use harmonic predictions for planning further ahead.

Load the resulting JSON file using the same **UKHO / JSON File** tab.

### 4. Manual Entry

For data from printed sources (Reeds Nautical Almanac, Admiralty Tide Tables, etc.):

1. Select the **Manual Entry** tab.
2. Enter one tidal event per line in the format: `DD/MM HH:MM HW|LW height`
3. Set the timezone selector to match the source data.
4. Click **Load Events**.

Example:
```
14/04 10:10 HW 4.0
14/04 15:30 LW 1.3
14/04 22:38 HW 4.3
15/04 03:56 LW 1.2
```

## Access Windows and Calendar Export

Once tidal data is loaded, the **Access Windows** tab shows when the predicted tidal height exceeds the access threshold, displayed as green bars on a 24-hour timeline for each day.

Click **Export .ics Calendar** to download a calendar file. Each continuous access window becomes a single calendar event with:

- Start and end times spanning the full window (not split at midnight)
- Duration rounded down to the nearest quarter-hour in the event title
- HW times and heights in the event description
- A stable UID anchored to the nearest HW, so re-importing updated data updates existing events rather than creating duplicates

## Tidal Curve

The **Tidal Curve** tab shows the interpolated height curve for each day, with:

- The access threshold shown as a red dashed line
- Green shading where the height exceeds the threshold
- HW/LW event markers with heights
- Observation markers (if loaded)

The interpolation uses a power-cosine model calibrated against Admiralty EasyTide data for Langstone Harbour, which captures the characteristic double high water stand better than a standard cosine interpolation.

## Observation Log and Mooring Calibration

The drying height of the mooring is the most important parameter and is initially an estimate. It can be refined over time using observations recorded at the sailing club.

### Recording observations

Download the [observation log spreadsheet](https://github.com/Consiliarius/TidalAccessCalc/blob/main/observation_log.xlsx) and record:

- **Date** (DD/MM/YYYY)
- **Time (BST)** — local time as shown on your watch or phone. The mooring is only in use during BST, so all times are recorded in BST. The dashboard converts to UTC automatically on import.
- **Wind Direction** — from the dropdown (N, NE, E, SE, S, SW, W, NW, Calm, Variable). This matters because the boat swings on its mooring and may sit over different depths depending on whether wind or tide dominates the heading.
- **Afloat?** — Yes, No, or Uncertain. "Uncertain" observations are logged but excluded from model calibration.
- **Notes** — any relevant detail (e.g. "just touched", "well afloat", "heeled over on mud", "wind against tide", tidal state if known)

#### When to observe

Any time you are at the sailing club and can see the boat. The most useful observations are:

- **Around low water on spring tides** — constrains the maximum drying height
- **Around low water on neap tides** — constrains the minimum drying height
- **Transitional moments** — when the boat is just lifting off or just settling. These provide the tightest constraints on the drying height.

A handful of well-timed observations, especially near LW on springs, are worth more than many observations taken at high water when the boat is always afloat.

### Uploading observations

1. In the dashboard, go to the **Observations** tab.
2. Upload the `.xlsx` workbook directly, or a `.csv` export of it. The dashboard accepts both formats.

The calibration engine compares each observation's time against the interpolated tidal curve to estimate the drying height. The **Mooring Calibration** panel shows the current estimate, the bounding range from observations, and a confidence indicator.

### Historic tidal data

Calibration requires tidal data covering the dates of each observation. Since the UKHO API only provides 7 days ahead and the KHM page covers ~30 days, observations from earlier dates would normally have no tidal data to calibrate against.

To solve this, the dashboard automatically fetches `historic_data.json` from the repository when observations are loaded. This file contains an accumulating record of tidal predictions for Langstone Harbour, updated daily. If an observation's timestamp falls outside the actively loaded tidal data (e.g. KHM or UKHO), the dashboard falls back to this historic record for the tidal height lookup.

The status message after loading observations shows how many were matched against tidal data and how many fell outside the available range.

Wind direction is captured because the boat swings on its mooring and may sit over different depths depending on whether wind or tide dominates the heading. The current calibration model does not factor in wind direction — all observations contribute equally regardless of wind. If sufficient data accumulates showing a consistent difference in effective depth by wind direction, a future refinement could account for this.

## Files

| File | Purpose |
|---|---|
| `index.html` | The dashboard (self-contained, no dependencies) |
| `fetch_tides.py` | Python script for UKHO API and harmonic predictions |
| `observation_log.xlsx` | Excel workbook for recording mooring observations |
| `historic_data.json` | Accumulating record of Langstone tidal predictions (updated daily, used for observation calibration) |
| `README.md` | This file |

## Limitations

- **Not for navigation.** This tool provides estimates for planning purposes only. Always verify conditions visually before departing or arriving.
- **Meteorological effects** (storm surge, barometric pressure, prolonged wind) are not modelled and can significantly alter actual water levels.
- **Seabed changes** from silting may alter the effective drying height over time. Ongoing observations help detect this.
- **The double high water stand** is approximated by the interpolation model. The actual curve shape varies and the model is calibrated for Langstone Harbour specifically.
- **Tidal data:** © UK Hydrographic Office. Contains Crown copyright material.

## Licence

The tool is provided as-is for personal use. Tidal prediction data from the UKHO API is subject to Crown copyright and the terms of the Admiralty Developer Portal.

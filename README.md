# Interactive Time Series Annotator with Power Zones

A web app for visualizing and annotating cycling power data. Upload a CSV, classify efforts into power zones, and manually label interval start/end points.

Deployed via [Render](https://render.com).

---

## Input Format

The app accepts a **CSV file** with the following columns:

### Required columns

| Column | Type | Description |
|---|---|---|
| `timestamp` | datetime | Timestamp of each data point (1-second resolution recommended) |
| `power` | numeric | Raw power output in watts |

### Optional columns

| Column | Type | Description |
|---|---|---|
| `interval_zone_type` | string | If present, skips the zone classification pipeline and uses this column directly |
| `Manual_Timestamps` | boolean | Pre-existing manual annotations — points marked `True` will be loaded as selected timestamps |

No preprocessing required — the app handles common data quality issues automatically:
- **Missing seconds** are filled in by reindexing to a continuous 1-second time range — the raw `power` value for inserted rows is `NaN`, but the rolling average used for visualization and zone classification (`power_roll_avg`) smooths over these gaps using a 5-second centered window (requiring at least 1 valid value in the window)
- **Duplicate timestamps** are removed (first occurrence kept)
- **Unsorted rows** are sorted by timestamp before processing

### Example

```
timestamp,power
2024-06-01 10:00:00,210
2024-06-01 10:00:01,215
2024-06-01 10:00:02,230
...
```

---

## Parameters

| Parameter | Default | Description |
|---|---|---|
| **FTP** (Functional Threshold Power) | 230 W | Your threshold power in watts — used to calculate zone boundaries |
| **Watt Threshold** | 11 W/s | Minimum power derivative to detect an interval start or end |

---

## Power Zones

Zones are calculated as a percentage of FTP:

| Zone | % of FTP | Color |
|---|---|---|
| Zone 1 | < 55% | Dark blue |
| Zone 2 | 55–75% | Light blue |
| Zone 3 | 75–90% | Green |
| Zone 4 | 90–105% | Yellow |
| Zone 5 | 105–120% | Orange |
| Zone 6 | 120–150% | Red |
| Zone 7 | > 150% | Dark red |

---

## Features

- **Upload CSV** — drag and drop or click to upload
- **Auto-detect start points** — detects interval starts based on power derivative
- **Manual annotation** — click directly on the chart to add/remove timestamp markers
- **Delete rows** — remove individual annotations from the table
- **Download Selected Timestamps** — exports the original CSV with an added `Manual_Timestamps` boolean column
- **Labels Model** — exports a CSV with auto-detected interval start timestamps and their group IDs

---

## Output Files

| Button | Output file | Contents |
|---|---|---|
| Download Selected Timestamps | `<original_filename>_with_manual_labels.csv` | Full input CSV + `Manual_Timestamps` column |
| Labels Model | `model_labels.csv` | `timestamp` and `interval_group_id` for each detected interval start |
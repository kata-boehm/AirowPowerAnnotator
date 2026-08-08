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

---

### Processing Pipeline

The pipeline transforms a raw power time series into labelled interval segments through the following steps:

**1. `process_csv_df` — Preprocessing and zone classification**
Sorts records, removes duplicate timestamps and reindexes to a continuous 1-second grid. Computes a centred rolling mean of power (window = 5s) and its first derivative. Candidate interval starts and ends are flagged where the derivative exceeds ±`watt_drop`, and these candidates are extended backward/forward while the local gradient remains in the same direction. Power is then classified into seven zones as a percentage of FTP.

**2. `group_true_values` — Candidate grouping**
Collapses consecutive runs of start and end candidates into single boundary flags, keeping only the first occurrence of each group by checking the two neighbouring rows.

**3. `enforce_consecutive_intervals` — Boundary consistency**
Guarantees an alternating start/end structure: a row following an end is forced to be a start and vice versa. The first and last rows of the session are forced to be a start and an end respectively.

**4. `assign_dominant_zone_type_per_interval` — Interval labelling**
Assigns a group ID via cumulative sum over the start flags and labels each interval with its most frequent power zone (majority vote).

**5. `merge_consecutive_same_zone_intervals` — Merging identical zones**
Merges adjacent intervals sharing the same zone label, renumbers the group IDs and marks the definitive first and last row of each resulting interval.

**6. `detect_and_invalidate_stop_resume_events` — Artefact removal**
Detects short stop-and-resume events within a 20-second sliding window, defined as a power drop above the threshold followed by a return to the previous level. Boundaries inside such windows are invalidated and the interval metadata preceding the drop is carried forward.

**7. `merge_short_intervals` — Removal of spurious segments**
Reassigns intervals shorter than the minimum duration to whichever neighbour has the closer mean power over a 5-second comparison window.

**8. `reassign_first_n_seconds` — Warm-up handling**
Overwrites the zone label of the first 60 seconds with the first valid zone occurring after the cutoff, removing the artificial boundary caused by the session start.

Step 5 is applied a second time after step 8, since the preceding corrections can leave adjacent intervals with identical zone labels.

---

## Reasoning for using Dash

Dash was chosen because the annotation tool requires interactive callbacks on top of the existing Python processing pipeline, so the same pandas and numpy code used for interval detection can be reused directly without reimplementing it in JavaScript. Since Dash executes these callbacks server-side, the application depends on a running Python process and cannot be deployed on static hosting such as GitHub Pages. Render was used as the hosting platform, though any service that runs a Python web process (for example Heroku, Railway, or a self-hosted container) would serve the same purpose.

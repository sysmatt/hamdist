# hamdist

**Author:** Matt Hoskins, K2TTA — k2tta@arrl.net

Calculate the great-circle distance for every QSO in one or more ADIF log files, producing a per-QSO CSV and a per-band summary table.

---

## Requirements

- Python 3.10+
- No pip packages required — standard library only

### External data (not Python packages)

| Path | Purpose | How to get it |
|---|---|---|
| `~/.hamdat/hamdat.db` | FCC license database for US callsign ZIP lookup | `hamdat --pull` |
| `~/.hamdat/pgeocode/US.txt` | US ZIP code centroids | Seeded by `hamdat --pull` |
| `~/.hamdist_cty.dat` | AD1C Big CTY prefix database for DXCC lookup | Auto-downloaded on first run; also borrows `~/.jtcon_cty.dat` if present |

The hamdat database is **required**. The cty.dat file is auto-downloaded from `country-files.com` on first run if not already present.

---

## Installation

```
chmod +x hamdist
cp hamdist ~/bin/    # or anywhere on your PATH
```

---

## Usage

```
hamdist [options] ADIF [ADIF ...]
```

### Options

| Flag | Description |
|---|---|
| `--grid GRID` | Fallback `MY_GRIDSQUARE` when not present in the log |
| `--miles` | Report distances in miles (default: km) |
| `--include-dupes` | Count duplicate QSOs in distance totals (default: deduplicate) |
| `--csv FILE` | Per-QSO CSV output path (default: `hamdist.csv`) |
| `--hamdat-db DB` | Path to hamdat SQLite database (default: `~/.hamdat/hamdat.db`) |
| `--cty FILE` | Path to cty.dat (auto-detected or downloaded if absent) |
| `--version` | Print version and exit |

---

## Location resolution

### Their station (the contact)

Resolution is attempted in order; the first success wins:

1. **`GRIDSQUARE`** field in the ADIF record → 4-char Maidenhead center
2. **hamdat ZIP lookup** — callsign looked up in the FCC database → ZIP code → centroid (US callsigns only)
3. **cty.dat DXCC prefix** — callsign prefix matched against the AD1C Big CTY database → country centroid
4. **`unknown`** — if all methods fail (logged in CSV, counted in summary)

Special cases:
- `/MM` (maritime mobile) and `/AM` (aeronautical mobile): flagged as `mm/am`, no location derivable
- `/P`, `/M`, `/4`–`/9` and other portable/district suffixes: stripped before lookup

### My station (the operator)

1. **`MY_GRIDSQUARE`** field in each ADIF record (supports different QTHs per QSO across portable operations)
2. **`--grid GRID`** CLI argument — used as fallback when `MY_GRIDSQUARE` is absent
3. **Hard error** if neither is available for a QSO

### Grid square precision

All grid squares are truncated to 4 characters and the center of the resulting cell is used. This normalises accuracy across all records — a 6-char grid (`FN30at`) and a 4-char grid (`FN30`) produce identical distance results. The original grid precision is recorded in the CSV (`grid4` vs `grid6+`).

---

## Duplicate handling

A duplicate is defined as: same callsign + same band + same mode + same UTC date.

By default, duplicates are **excluded** from distance totals but their count is reported in the summary. Use `--include-dupes` to count everything.

---

## Band resolution

Band is derived from the ADIF `FREQ` field (MHz) first. The `BAND` field is used only as a fallback when `FREQ` is absent. QSOs where neither yields a recognised band are grouped as `unknown`.

Supported bands follow the full ADIF v3.1.4 band enumeration:
`2190m`, `630m`, `560m`, `160m`, `80m`, `60m`, `40m`, `30m`, `20m`, `17m`, `15m`, `12m`, `10m`, `6m`, `4m`, `2m`, `1.25m`, `70cm`, `33cm`, `23cm`, `13cm`, `9cm`, `6cm`, `3cm`, `1.25cm`, `6mm`, `4mm`, `2.5mm`, `2mm`, `1mm`

---

## Output

### Summary (always printed to stdout)

```
hamdist - QSO Distance Summary
Files : wsjtx_log.adi
My QTH: FN30 (--grid)

  Band    QSOs   Dupes       Max km       Total km  Unknown
------  ------  ------  ----------  -------------  -------
   20m     203       3    18,432.0    1,234,567.0        3
   40m     112       5    14,232.0      891,432.0        5
   80m      47       2     8,432.0      142,891.0        2
  160m       3       0     2,341.0        4,102.0        0
------  ------  ------  ----------  -------------  -------
 TOTAL     365      10    18,432.0    2,272,992.0       10
```

### CSV (written to `hamdist.csv` by default)

| Field | Description |
|---|---|
| `date` | QSO date (UTC, YYYYMMDD) |
| `time` | QSO start time (UTC, HHMMSS) |
| `call` | Contact callsign |
| `band` | Derived band |
| `mode` | Operating mode |
| `their_grid` | Raw `GRIDSQUARE` from log |
| `their_grid4` | Truncated to 4-char (used for distance) |
| `my_grid4` | My 4-char grid used for this QSO |
| `dist_km` | Distance in km |
| `dist_mi` | Distance in miles |
| `loc_method` | How their location was resolved: `grid4`, `grid6+`, `hamdat`, `dxcc`, `mm/am`, `unknown` |
| `dupe` | `Y` if this QSO was flagged as a duplicate |

---

## Related tools

- **[hamdat](https://github.com/sysmatt/hamdat)** — FCC amateur license database downloader and query tool; provides the `hamdat.db` and pgeocode data required by hamdist
- **[jtcon](https://github.com/sysmatt/jtcon)** — WSJT-X TUI monitor; shares the cty.dat prefix database used by hamdist

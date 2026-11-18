# Complaint Tracker — Technical Reference

## Part A — `app.py` (Flask)

### A1. Architecture

```
index.html (vanilla JS) ── POST /api/analyze ──▶ api_analyze()
                       ── GET /api/status, /api/pi-probe, /api/six-months-jumbos
                                     │
             ┌───────────────────────┼──────────────────────────┐
        SQL (pyodbc, one             PI (PIconnect)             Excel masters
        connection per thread)       server 10.35.28.240        PM-4/PM-5_Tags.xlsx
        invent_dhi_wound             AF PSPBCQDHIAF             NCP Defects Master.xlsx
        jumbo_ulma_remarks           ThreadPool (20 workers)
                                     │
                              save_analysis() → results/*.xlsx
```

Every error is returned as JSON by `handle_unexpected`, so the browser never receives an HTML error page ("Unexpected token '<'").

### A2. Configuration — top of `app.py`

| Constant | Value | Meaning |
|---|---|---|
| `MACHINES["04"/"05"]` | name, accepted ids, `jumbo_like` pattern (`____04____`), tags file, AF base path, `ulma_end_is` (`start` for PM4, `end` for PM5) | Everything that differs between the machines. The machine is **read from the reel's `PROD_MACH_ID`**, never guessed |
| `CONNECTION_FILE_CANDIDATES` | `F:\Env_new\pm5 Run chart\…`, `F:\Furnish_Simulator\Pm5 run chart\…`, `./connection.txt` | First one found wins |
| `PI_FETCH_MAX_WORKERS` | 20 | Parallel tag reads |
| `RUN_GAP_HOURS` / `RUN_SEARCH_DAYS` | 4 / 7 | A wound-time gap of more than 4 h starts a new grade-run block; search ±7 days |
| `JUMBO_MAX_MINUTES` | 90 | Longest believable jumbo window |
| `NEIGHBOUR_JUMBOS` | 4 | Previous/next comparison span |
| `BASELINE_DAYS` / `BASELINE_ULMA_ROWS` | 180 / 8000 | 6-month same-GSM baseline |
| `MIN_BASELINE_SAMPLES` / `BASELINE_REFINED_INTERVAL` | 12 / `30m` | Baseline sampling |
| `DEFAULT_DEVIATION_PCT` | 2.0 | Report threshold |

**Schema-tolerant SQL:** `FIELD_CANDIDATES` / `ULMA_FIELD_CANDIDATES` map logical fields to real column names. For example, the machine field may be `PROD_MACH_ID`, `MACHINE_ID` or `MACH_ID`. `resolve_schema()` resolves them once per table, and the reject columns (`REJECT_ID`, `REJECT_ID_2`, …) are found automatically.

### A3. The pipeline — `api_analyze()`

Request body: `{track_num, complaint_code, threshold?, tag_types?: ["Process","Quality"]}`

| Step | Function(s) | Output |
|---|---|---|
| 1 Reel | `fetch_track` | Reel details, machine key |
| 2 Jumbo | `fetch_jumbo_reels`, `reject_codes_of` | All sibling reels; which ones carry the complaint code |
| 3 Window | `fetch_ulma_neighbourhood` → `resolve_end_times` → `ulma_sequence` → `jumbo_window_from_ulma` | The jumbo's `(start, end)` |
| 4 Grade run | `find_grade_run`, `split_run_blocks` | Run boundaries (fallback ±12 h around the wound time) |
| 4b Spread | `defect_in_jumbos`, `count_reels_with_defect`, `remarks_defect_map`, `defects_in_text` | Where else the defect appears: neighbour jumbos, the run, remarks text |
| 5 Baselines | `windows_by_jumbo`, `same_gsm_jumbo_periods` | Period lists for the previous-4, next-4, run and 6-month scopes |
| 6 Tags | `analyse_tags` → `fetch_pi_average` + `deviation` + `exceeds` | Tags beyond the threshold, with evidence |
| Save | `save_analysis` | Excel files (see A6) |

**Time rules (same as GSM Tab 2).** `resolve_end_times` gives each jumbo an end time from, in order: **1** the time written in its own remarks; **2** `ULMA_END`; **3** a sibling with the same ULMA id; **4** a same-day sibling matched on shift code (`A1/B2/C3`) or ULMA suffix. C shift (22:00–06:00) moves early hours to the next day. An ULMA id carries its own date (`26041301` = 13 Apr 2026). `window_at()` then bounds each jumbo by its neighbour in ULMA order: PM5 ULMA = end, so start = the previous jumbo's end; PM4 is the mirror image. **The complained jumbo is measured by exactly the same rule as the jumbos it is compared against.** If its own `ULMA_END` is blank, the last jumbo with the same ULMA id stands in.

**PI averaging — `fetch_pi_average`**
* `interpolated_values` at a fixed interval (`pick_interval` scales it with the window length), so a plain mean is already time-weighted
* the **machine's stable-run filter** (e.g. `PM5 NoBreak & NoGradeChange`) is applied through an AF filter expression
* for multi-jumbo baselines, readings are kept only when they fall inside one of the jumbo periods (`in_periods`)
* returns its **evidence**: sample count, excluded count, interval, first and last raw readings, seconds taken, error. A quick answer with `samples=0` points to a filter or tag-name problem, and the UI shows it
* digital states (`AFEnumerationValue`) are converted with `pi_value_to_number`

**Deviation — `deviation(reel, base)`** returns `(abs, pct, reliable)`. `reliable` becomes False when |pct| > 100 %, which flags a baseline near zero or one that includes idle time. `exceeds(threshold, *pcts)` is true if any scope is beyond ±threshold.

**Tag sources — `load_tags(mkey)`** reads the `Process_Tags` and `Quality_Tags` sheets (label → tag). `parse_tag_path()` classifies each entry: a full PI point (`PSPD_BCM_PM05_DCS:AIC8701:MV`), an AF path, or a bare AF attribute name under the machine's `af_base`.

### A4. API

| Method | Route | Purpose |
|---|---|---|
| GET | `/` | UI |
| GET | `/api/status` | SQL/PI state, tag counts per machine |
| POST | `/api/analyze` | Full investigation (A3) |
| GET | `/api/six-months-jumbos?track_num=…` | Excel of every jumbo behind the 6-month baseline: same-GSM yes/no, usable window, time source. A quick check that doesn't run the slow tag analysis |
| GET | `/api/pi-probe?minutes=60` | Reads one tag over the last hour and returns the raw values. If this returns nothing, no average can be trusted |

### A5. UI sections (`index.html`)
Form: track · complaint code (code or defect name) · threshold %; buttons **Analyse**, **Test PI now** and **Download 6-month jumbo list**.
Results: **1** Reel details · **2** Jumbo (sibling reels, code hits) · **3** Grade run & process window · **4** Where else this defect appears · **5** Deviating parameters. Each row has a ⌄ button that shows the readings behind it.

### A6. Outputs (`results/`)

| Path | Content |
|---|---|
| `results/complaint_log.xlsx` | Master log: one row per search |
| `results/jumbos/<JUMBO>__<TRACK>__<YYYYMMDD-HHMMSS>.xlsx` | Full analysis workbook for one search |
| `results/raw_pi_readings/…xlsx` | Every raw PI reading used (capped at `RAW_SHEET_ROW_LIMIT = 900 000` rows per sheet) |
| `results/six_month_jumbo_lists/…xlsx` | Downloads from `/api/six-months-jumbos` |

`_free_path()` never overwrites an existing file. `save_analysis()` never raises, so a logging failure cannot break an analysis.

### A7. Self-test
`python app.py --selftest` runs the pure logic with no SQL or PI: column picking, tag-path parsing, interval choice, remarks time parsing, window rules, deviation. It prints `selftest OK`.

## Part B — `pm4_jumbo_tag_average_dashboard.py`

| Aspect | Detail |
|---|---|
| Input | Excel with Jumbo ID, Start and End. `detect_columns()` **scores** the candidate columns, because the source sheet has two "Jumbo End Time" columns (one time-only, one full datetime) |
| Date parsing | `parse_datetime_column`: collapses the double space and forces **day-first** (`17/01/2021  09:36`), so 03/04 is not read as 4 March |
| Tags | `PM-4_Tags.xlsx › Process_Tags › Tag` |
| PI | AF connection path copied from the PM4 jumbo batch extractor. `reset_pi_connection()` between batches clears COM handles and rate limits. Up to 5 reconnect attempts, 60 s apart |
| Average | `time_weighted_average`: trapezoidal TWA, because PI records on change, not on a clock. `boundary_type='interpolate'` pins the series to the window edges |
| Parallelism | 20 threads per jumbo (one per tag), 2 retries per tag |
| Batching | 500 jumbos per batch, run automatically one after another |
| Persistence | `tag_averages_parts/<sheet>/part_<year>_<batch>.csv`, one row appended and flushed per jumbo. A re-run skips saved jumbos, re-fetches any whose times changed in the sheet, and optionally retries errors |
| Output | `outputs/<sheet>_<year>_tag_averages.xlsx`: the averages plus a fetch log, rebuilt from the parts every `LIVE_EXPORT_EVERY = 25` jumbos and written via a temp file and swap. A live base64 download link does not stop the run |
| CLI | `--run "<file>.xlsx" --year 2024`; `--selftest` |

## Known issues and technical debt

| # | Issue | Suggested fix |
|---|---|---|
| 1 | PM1A has a tags file but no `MACHINES["1A"]` entry | Add the AF base, the `jumbo_like` pattern and the ULMA semantics for PM1A |
| 2 | Some SQL is built with f-strings (`col()` returns bracketed names, so the risk is limited to values) | Parameterise the remaining value interpolations |
| 3 | The 6-month baseline makes many PI calls and takes minutes | Cache per GSM/week, or precompute with the tag-averages tool |
| 4 | The Flask development server is used in production | Run under `waitress` on Windows |
| 5 | `TAGS_FILE_CANDIDATES` in the averages tool expects `complaint_tracker/PM-4_Tags.xlsx` *relative to the script*, i.e. a subfolder | Run it from `complaint_tracker/`; the second candidate `./PM-4_Tags.xlsx` covers that |

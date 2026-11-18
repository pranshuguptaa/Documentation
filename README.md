# PM4 / PM5 Complaint Tracker (+ PM4 Jumbo Tag Averages)

**Folder:** `complaint_tracker/`

| App | File | Type | Port / run |
|---|---|---|---|
| **Complaint Tracker** | `app.py` (2,359 lines) + `templates/index.html` + `static/style.css` | Flask + JS single page | `python app.py` → `:7120` (env `PORT`) |
| **PM4 Jumbo Tag Averages** | `pm4_jumbo_tag_average_dashboard.py` (968 lines) | Streamlit + headless CLI | `streamlit run …` / `python … --run file.xlsx --year 2024` |

## What the Complaint Tracker does

When a customer complains about a reel, QA enters the **track number** and the **complaint code** (an NCP code such as `M`, or a defect name such as `SPOT`). The tracker:

1. Finds the reel in `invent_dhi_wound`: its jumbo, machine (PM4/PM5), grade, GSM, reject codes and wound time.
2. Loads every reel from the same jumbo and marks which of them carry the same reject or complaint code.
3. Works out the jumbo's real **production window** from ULMA times and remarks, using the same rules as the GSM dashboard.
4. Finds where else the defect appears: in the 4 jumbos before and after, in the grade run, and in the jumbo remarks.
5. Averages every process and quality PI tag (`PM-4_Tags.xlsx` / `PM-5_Tags.xlsx`) over the jumbo window, using the stable-run filter. It compares each average against four baselines: **previous 4 jumbos · next 4 jumbos · the whole grade run · the same GSM over the last 6 months**.
6. Shows **only the tags that deviate by more than the threshold** (default 2 %), with the raw readings behind every number. Each search is saved to Excel.

## What PM4 Jumbo Tag Averages does

A standalone bulk tool. It takes an Excel of `Jumbo ID / Start / End` (e.g. *PCB TIME STAMPS.xlsx*) and writes one row per jumbo holding the **time-weighted average of every PM4 process tag** (about 402 of them) over that jumbo's window. It runs in batches of 500 that resume automatically and writes CSV part files, so a crash loses at most one jumbo. It uses no SQL at all.

## Versions

* **Current:** the Flask `app.py` (this folder; the copy in the inner `Backup/complaint_tracker/` is identical).
* **Previous / superseded:** `Backup/New folder/untitled folder/complaint_tracker_dashboard.py` (431 lines). This is the original **Streamlit, PM4-only** tracker. Its tag list was copied from the PM4 jumbo batch extractor and it read `INVENT_DHI`. The Flask app's docstring confirms it replaces this one ("same PI logic as the old Streamlit tracker"). It is kept for reference only.
* `PM-1A_Tags.xlsx` is in the folder, but `MACHINES` only defines `04` and `05`. **PM1A is not wired in yet.**

## Documents

| File | For |
|---|---|
| `TECHNICAL.md` | Pipeline, time-window rules, statistics, API, outputs, self-test, known issues |
| `OPERATIONS.md` | Install, run, how to investigate a complaint, the bulk averages tool, troubleshooting |

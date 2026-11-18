# Complaint Tracker — Operations & User Guide

## 1. Install
```bat
cd complaint_tracker
pip install flask pandas numpy openpyxl pyodbc PIconnect streamlit
```
These files must sit next to `app.py`: `PM-4_Tags.xlsx`, `PM-5_Tags.xlsx`, `NCP Defects Master.xlsx` and `connection.txt` (or the connection file in one of the `F:\…` paths).

## 2. Run
```bat
python app.py --selftest        :: logic check, no SQL/PI needed
python app.py                   :: http://<server>:7120
set PORT=5001 && python app.py  :: another port
```

## 3. Investigate a complaint (QA user guide)
1. Press **Test PI now** first. It should return samples. If it returns 0, PI or the tag names are wrong and the averages will be empty.
2. Enter the **Track number** and the **Complaint code**. Either the NCP code (e.g. `M`) or part of the defect name (e.g. `SPOT`) works; names need at least 4 characters to match.
3. Set the **Deviation threshold** (default 2 %) and press **Analyse**. A full run can take a few minutes because of the 6-month baseline.
4. Read the sections:
   * **1 Reel details:** machine, grade, GSM, reject codes
   * **2 Jumbo:** every sibling reel; reels carrying the same code are highlighted
   * **3 Grade run & process window:** the window used and where each time came from
   * **4 Where else this defect appears:** neighbour jumbos, grade run, remarks
   * **5 Deviating parameters:** only tags beyond the threshold against any scope. Press ⌄ to see the raw readings and sample counts. "Unreliable" means the deviation is above 100 %, so check the baseline before trusting it
5. **Download 6-month jumbo list** lets you check which jumbos made up the baseline, without running the full analysis.
6. Every search is saved under `results/`. See TECHNICAL A6.

## 4. Bulk tag averages (PM4)
```bat
streamlit run pm4_jumbo_tag_average_dashboard.py
:: or headless
python pm4_jumbo_tag_average_dashboard.py --run "PCB TIME STAMPS.xlsx" --year 2024
```
Upload the timestamp sheet, check the detected columns, optionally pick a year, and press Run. Leave it running; it resumes by itself if stopped. Results go to `outputs/…_tag_averages.xlsx`.

## 5. Troubleshooting
| Symptom | Action |
|---|---|
| "Track not found" | Check the track in `invent_dhi_wound`; it may not be wound yet |
| Machine not supported | The reel's `PROD_MACH_ID` is not 04 or 05 (PM1A is not wired in yet) |
| Every tag shows samples = 0 | The stable-run filter excluded everything (the machine was down for the whole window), or the tag names are wrong. Run `/api/pi-probe` |
| Window shows a "fallback" source | ULMA time or remarks missing; check `jumbo_ulma_remarks` |
| Run takes a long time | It's the 6-month baseline. Use the 6-month list to check the scope first. The API also accepts `tag_types: ["Process"]` to skip the Quality tags |
| Excel not saved | The file is open in Excel, or a permission problem; the analysis is still shown on screen |

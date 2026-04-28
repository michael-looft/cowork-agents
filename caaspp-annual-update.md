---
name: caaspp-annual-update
description: Check ETS website for new CAASPP research files, download and import when available (runs every 3 days Oct–Mar to catch late releases)
---

## CAASPP Annual Data Update — Automated Check and Import

**Objective:** Check whether the California CAASPP Smarter Balanced Assessment research files for the current year are available on the ETS website. If they are, download them, process them, and import into the Marin Promise Partnership public_datasets database. If they are not yet available, log the check and exit (the scheduler will retry in 3 days).

---

## Step 1 — Determine the target year

The target year is the current calendar year. CAASPP files for test year YYYY are released in the fall of YYYY (e.g., 2026 files appear in October 2026).

---

## Step 2 — Run the check-and-update script

The project folder is:
```
/Users/looftm/Documents/Claude/Projects/Update CAASPP Data/
```

Run the following in a bash shell:
```bash
cd "/Users/looftm/Documents/Claude/Projects/Update CAASPP Data/"
python3 check_and_update_caaspp.py
```

The script accepts an optional year argument; without one it defaults to the current year.

**What the script does:**
- Reads `.caaspp-file-check-state.json` to check if this year's update is already complete. If complete, it exits immediately — nothing to do.
- Checks whether all four source files are already present in the project folder (in case they were manually downloaded). If they are, it skips to processing.
- Otherwise, attempts to download all four zip files from the ETS CAASPP research file server:
  - `sb_ca<YEAR>_all_21_csv_v1.zip` — Marin county file (ELA + Math)
  - `sb_ca<YEAR>_all_csv_ela_v1.zip` — Statewide ELA file
  - `sb_ca<YEAR>_all_csv_math_v1.zip` — Statewide Math file
  - `sb_ca<YEAR>entities_csv.zip` — Entity reference file
- If any file returns 404, the files are not yet released. The script logs "not yet available" and exits (scheduler retries in 3 days).
- If all four files download successfully, it calls `update_caaspp.py` to process and import the data.

**Exit codes:**
- `0` — Files not yet available; retry later
- `1` — Error during processing or import; retry later
- `2` — Update completed successfully; no further action needed this year

---

## Step 3 — Report the outcome

After the script exits, report the result clearly:

- **Exit code 0:** "CAASPP files for [YEAR] are not yet available on the ETS website. Checked on [date]. The scheduler will check again in 3 days."
- **Exit code 1:** "An error occurred during the CAASPP update. Check `.caaspp-file-check-log.txt` in the project folder for details. The scheduler will retry in 3 days."
- **Exit code 2:** "✓ CAASPP [YEAR] data has been downloaded, processed, and imported successfully. The scheduler will resume checking on October 1, [YEAR+1]."

---

## Key file locations

| File | Purpose |
|------|---------|
| `/Users/looftm/Documents/Claude/Projects/Update CAASPP Data/check_and_update_caaspp.py` | Main scheduled script (download + process + import) |
| `/Users/looftm/Documents/Claude/Projects/Update CAASPP Data/process_caaspp.py` | Transforms CAASPP source files → import CSV |
| `/Users/looftm/Documents/Claude/Projects/Update CAASPP Data/update_caaspp.py` | Handles MySQL import (DELETE old + INSERT new) |
| `/Users/looftm/Documents/Claude/Projects/Update CAASPP Data/.caaspp-file-check-state.json` | Tracks completion status (prevents duplicate runs) |
| `/Users/looftm/Documents/Claude/Projects/Update CAASPP Data/.caaspp-file-check-log.txt` | Append-only log of all check runs |
| `/Users/looftm/Documents/Claude/Projects/Update CAASPP Data/.env` | Database credentials (DB_PASSWORD etc.) |

---

## Business context

This data feeds the Marin Promise Partnership public dashboard. The CAASPP Smarter Balanced Assessment data covers:
- **Third Grade Reading** — Grade 3 ELA proficiency (proxy for early literacy milestone)
- **Ninth Grade Math** — Grade 8 Math proficiency (proxy for algebra readiness)

The import covers Marin County districts/schools plus the State of California aggregate. Demographics include ethnicity breakdowns and a calculated "Students of Color" metric (Total − White).

---

## If the script fails or the files aren't downloadable

If the automated download fails repeatedly (e.g., the ETS URL structure changed), notify Michael (looftm@gmail.com) that the CAASPP files need to be downloaded manually from:
```
https://caaspp-elpac.ets.org/caaspp/ResearchFileListSB?ps=true&lstTestYear=<YEAR>&lstTestType=B&lstCounty=21&lstDistrict=00000&lstFocus=b
```
Place the four extracted `.txt` files in the project folder and re-run `check_and_update_caaspp.py` — it will detect the manually downloaded files and proceed to import.
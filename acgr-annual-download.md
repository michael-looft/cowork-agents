---
name: acgr-annual-download
description: Check CDE page every 3 days Nov–Jan for new ACGR file, download it, and run the college readiness processing script
---

## CDE ACGR Annual Data Update

Your job is to check the California Department of Education website for a newly released Adjusted Cohort Graduation Rate (ACGR) data file, download it if available, process it, save an import-ready CSV, and then manage your own schedule so you only run when needed.

---

### Step 0 — Restore daily schedule if this is a season-kickoff run

If this run was triggered as a one-time (fireAt) run — meaning you were dormant since last year's successful processing — switch yourself back to a daily recurring schedule before doing anything else:

Call `update_scheduled_task` with:
- taskId: `acgr-annual-download`
- cronExpression: `0 9 * 11,12,1 *`

Then continue to Step 1.

---

### Step 1 — Determine which file to look for

Use Python to calculate the expected filename based on today's date:

```python
import datetime
today = datetime.date.today()
if today.month in (11, 12):
    suffix = str(today.year)[2:]
else:  # January
    suffix = str(today.year - 1)[2:]
filename = f"acgr{suffix}.txt"
import_filename = f"acgr{suffix}_import.csv"
next_year = today.year + 1 if today.month in (11, 12) else today.year
```

---

### Step 2 — Find the workspace folder

Use the Glob tool to search for `process_acgr.py` under `/Users/looftm/`. The folder containing that script is the workspace folder. If you cannot find it, report an error and stop.

---

### Step 3 — Check if already processed this season

Look in the workspace folder for the `import_filename` computed in Step 1. If it already exists, report "Already processed for this season — skipping." and stop. Do NOT reschedule here; the task is already sleeping until next November 1.

---

### Step 4 — Check the CDE page for the new file

Use Claude in Chrome to navigate to: https://www.cde.ca.gov/ds/ad/filesacgr.asp

Look for a link containing the expected filename (e.g., `acgr26.txt`).
- If the link is **NOT** present: report "File not yet posted as of [today's date]. Will check again tomorrow." and stop.
- If the link **IS** present: proceed to Step 5.

---

### Step 5 — Download the file

Click the download link for the file. Wait for the download to complete (it will appear in `/Users/looftm/Downloads/`). Then move it into the workspace folder:

```bash
mv "/Users/looftm/Downloads/acgrNN.txt" "/path/to/workspace/acgrNN.txt"
```

---

### Step 6 — Run the processing script

```bash
python3 "/path/to/workspace/process_acgr.py" "/path/to/workspace/acgrNN.txt"
```

This produces `acgrNN_import.csv` in the same folder.

---

### Step 7 — Reschedule to next November 1, then report

After successful processing, use Python to compute next November 1:

```python
import datetime
today = datetime.date.today()
if today.month in (11, 12):
    next_nov1_year = today.year + 1
else:
    next_nov1_year = today.year  # January run: next Nov 1 is later this year
fire_at = f"{next_nov1_year}-11-01T09:00:00-08:00"
```

Call `update_scheduled_task` with:
- taskId: `acgr-annual-download`
- fireAt: the computed `fire_at` string

This clears the daily cron and puts the task to sleep until next November 1.

Then report:
- How many rows were generated
- Full path to the output CSV
- That the task is now sleeping until November 1 of next year
- Reminder: import into MySQL `public_datasets` table (`[your-database-name]` database). Leave `public_datasets_id` and `LastUpdated` blank — MySQL sets these automatically. `Active` = 1, `Dataset` = Milestone, `Indicator` = College Readiness.
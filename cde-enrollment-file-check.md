---
name: cde-enrollment-file-check
description: Sleeping until Apr 1 — CDE enrollment 2526 import complete; will resume every-2-day checks next April
---

Annual CDE Census Day Enrollment update. Check whether the new file for the most recent school year has been posted, and if so, download it and import it into the [your-database-name] MySQL database.

WORKING FOLDER
  The user's workspace is the folder named "Update Annual Enrollment Data" (selected in Cowork). Inside that folder: update_enrollment.py, process_cde_enrollment.py, csv_to_sql.py, .env (with DB_PASSWORD), and the .command launcher.

KEY REFERENCES (these replace a lot of searching — trust them)
  • phpMyAdmin URL (fallback if direct MySQL fails):
      https://phpmyadmin.[your-hosting-provider]
  • Remote MySQL on Tigertech is OFF by default. The one-time enable is:
      My Account → MySQL Databases → Change → check "Allow connections from any computer on the Internet" → Save.
  • See auto-memory `reference_[your-hosting-provider] for the full connection notes.

STEP 0 — Wake-up check (runs only when task fires on April 1 after a sleep cycle)
  If today is April 1 (day == 1 and month == 4), this task was just woken from a sleep cycle by a one-time fireAt. Immediately restore the recurring every-2-day schedule by calling update_scheduled_task with:
    - taskId: "cde-enrollment-file-check"
    - cronExpression: "0 8 1/2 4-7 *"
  Then proceed to Step 1 as normal.

STEPS

1. Determine the expected school-year code to check.
   - Jan–Jul: code ending in the current calendar year (Apr 2026 → "2526")
   - Aug–Dec: code ending in next calendar year (Oct 2026 → "2627")
   Use `date` in Bash to read today's date.

2. Skip if already done. In the workspace folder, look for `cdenroll{code}_import.csv`. If it exists AND is newer than the source `cdenroll{code}.txt`, we've already processed this cycle — report "already processed" and STOP.

3. Check if CDE has posted the file. HEAD the URL:
       https://www3.cde.ca.gov/demo-downloads/census/cdenroll{code}.txt
   If it 404s, the file isn't out yet — report "not yet posted" and STOP. Do NOT step back to a prior school year.

4. File is posted — run the full pipeline. From the workspace folder:
       python3 update_enrollment.py
   This (a) auto-downloads the latest file, (b) runs process_cde_enrollment.py to generate the import CSV, (c) runs csv_to_sql.py to generate a SQL fallback file, (d) connects to MySQL and (DELETE existing Year rows + INSERT new rows).

5. Sanity-check the spot-check values printed by the processing step:
     - Marin County Total Enrollment All: 26,000–32,000
     - State of California Total Enrollment All: 5.4M–6.1M
     - White, Students of Color, Hispanic/Latino for Marin County: all > 0
   If anything is out of range or NOT FOUND, STOP and alert the user — do not trust the import. The CDE file format likely changed.

6. If update_enrollment.py exits with code 2, the MySQL connection failed but the CSV + SQL fallback file are ready. In that case:
     - Identify the rejected IP from the (1045, "Access denied for user '[your-db-user]'@'<IP>'") error.
     - Tell the user clearly: direct MySQL is blocked for IP <IP>; the fallback is phpMyAdmin at the URL above; the SQL file is at `{workspace}/cdenroll{code}_import.sql`.
     - Offer (via computer-use) to open Chrome, navigate to the phpMyAdmin URL, and upload the SQL file. The user must log in themselves; after login, you can drive Import → Choose File → Go.
     - Alternatively, recommend the one-time fix of enabling "Allow connections from any computer on the Internet" in Tigertech My Account.
     - Do NOT proceed to the sleep step below — the import is not confirmed complete.

7. Report back: school-year code, CDE file size, verification spot-check values, and MySQL rows deleted + inserted — or, if fallback was used, "phpMyAdmin path taken, N rows affected."

8. SLEEP UNTIL NEXT APRIL — after a fully confirmed successful import (MySQL rows inserted in step 7, OR phpMyAdmin import confirmed complete):
     - Calculate next April 1 at 8:00 AM Pacific time. If today is in April–July of year Y, the target is Y+1-04-01T08:00:00-07:00.
     - Call update_scheduled_task with:
         taskId: "cde-enrollment-file-check"
         fireAt: "{next April 1 ISO timestamp}"
         description: "Sleeping until Apr 1 — CDE enrollment {code} import complete; will resume every-2-day checks next April"
     - Report to the user: "Import complete. Task is now sleeping until April 1, {next year}."

NOTES
- Credentials for direct MySQL come from .env next to the script.
- Year label: cdenroll2526.txt → "2526" → Year = 2026 (the later of the two calendar years).
- Students of Color = RE_I + RE_A + RE_F + RE_B + RE_H + RE_P + RE_T (not RE_W, not RE_D). Filipino (RE_F) counts as SoC even though it appears under "Ethnicity: Asian" in the output.
- District name mapping and the " County" suffix for county aggregates are handled inside process_cde_enrollment.py.
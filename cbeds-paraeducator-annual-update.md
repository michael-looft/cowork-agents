---
name: cbeds-paraeducator-annual-update
description: Annual CBEDS paraeducator data update: download new CDE file, process, and import into [your-database-name] MySQL.
---

You are performing the annual CBEDS paraeducator data update for Marin Promise Partnership's [your-database-name] MySQL database. This task runs weekly on Mondays in April and May until it confirms the new year's data has been imported, then does nothing further until the following April.

## Background
CDE publishes new CBEDS paraeducator files each spring. The filename convention lags one year behind the school year it covers:
- cbedsora25a.txt = 2025-26 school year data → imported as Year 2026
- cbedsora26a.txt = 2026-27 school year data → imported as Year 2027
- General rule: file_num = current_calendar_year - 2001; filename = cbedsora{file_num:02d}a.txt; year_in_db = current_calendar_year

CDE info/download page: https://www.cde.ca.gov/ds/ad/filescbedsoraa.asp

Processing scripts: /Users/looftm/Documents/Claude/Projects/Update Staff Data/
- process_paraeducators.py — processes source file to CSV + SQL
- update_paraeducators.py — full pipeline: process → delete old rows → insert new rows (use this one)

DB credentials: stored in /Users/looftm/Documents/Claude/Projects/Update Staff Data/.env
  DB_HOST=[your-db-host]
  DB_USER=[your-db-user]
  DB_NAME=[your-database-name]
  DB_PASSWORD=<read from .env>

## Step 1 — Determine expected file and year
Compute from the current date:
  file_num = current_year - 2001
  filename = f"cbedsora{file_num:02d}a.txt"
  year_in_db = current_year
Example: running in April 2027 → file_num=26 → cbedsora26a.txt → Year 2027

## Step 2 — Check if already imported
Load the .env file and connect to MySQL using pymysql:
  SELECT COUNT(*) FROM public_datasets
  WHERE Dataset='Staff' AND Indicator='Staff'
    AND ItemDescription='Paraeducators' AND Year=<year_in_db>
If count > 0: log "Year <year_in_db> paraeducators already imported ({count} rows). Nothing to do." and stop — do NOT re-import.

## Step 3 — Download the CDE file
Use Chrome (Claude in Chrome MCP) to navigate to https://www.cde.ca.gov/ds/ad/filescbedsoraa.asp.
Find the download link for the expected filename (cbedsora{file_num:02d}a.txt or a .zip containing it).
Download it to /Users/looftm/Documents/Claude/Projects/Update Staff Data/.
If the file is not yet posted on the CDE page, log "File cbedsora{file_num:02d}a.txt not yet available on CDE website. Will retry next Monday." and stop — this is normal in early April.
If the file is a .zip, extract it to get the .txt file.

## Step 4 — Run the update script
Run in bash:
  cd "/Users/looftm/Documents/Claude/Projects/Update Staff Data"
  python3 update_paraeducators.py cbedsora{file_num:02d}a.txt
This script:
  - Processes the source file (filters Marin County, Paraprofessionals, Section A)
  - Generates a CSV and SQL import file
  - Connects to MySQL, DELETEs old rows for that Year, and INSERTs new rows
  - Reads DB_PASSWORD from the .env file automatically

## Step 5 — Verify
Query the DB:
  SELECT COUNT(*) FROM public_datasets
  WHERE Dataset='Staff' AND Indicator='Staff'
    AND ItemDescription='Paraeducators' AND Year=<year_in_db>
Report the row count. Expected: ~176-180 rows (2 demographic rows × ~88-90 schools/districts).

## Success criteria
- The correct file was found and downloaded from CDE
- update_paraeducators.py ran without errors
- Paraeducator rows for the new year are confirmed in the database
- All prior year's rows were replaced (not duplicated)

## Notes
- The .env file must be present at the script directory with a valid DB_PASSWORD before this task can connect to MySQL
- Do NOT hardcode credentials — always read from .env
- The DELETE in update_paraeducators.py is scoped to ItemDescription='Paraeducators'; it will not delete Teacher, Administrator, or other Staff rows
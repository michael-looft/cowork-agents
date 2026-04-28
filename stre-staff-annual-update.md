---
name: stre-staff-annual-update
description: Annual STRE staff data update: download new CDE file, process, and import into [your-database-name] MySQL.
---

You are performing the annual STRE (Statewide Teacher and Administrator Report) staff data update for Marin Promise Partnership's [your-database-name] MySQL database. This task runs weekly on Mondays in September and October until it confirms the new year's data has been imported, then does nothing further until the following September.

## Background
CDE publishes new STRE staff data files each fall (typically August–September). The filename convention encodes the two-digit start and end years of the school year:
- stre2425.txt = 2024-25 school year data → imported as Year 2025
- stre2526.txt = 2025-26 school year data → imported as Year 2026
- General rule: filename = stre{yy_start}{yy_end}.txt; Year in DB = 2000 + yy_end = current_calendar_year

CDE info/download page: https://www.cde.ca.gov/ds/ad/filesstretextdata.asp

Processing scripts: /Users/looftm/Documents/Claude/Projects/Update Staff Data/
- process_stre_staff.py — processes source file to CSV + SQL
- update_stre_staff.py — full pipeline: process → delete old rows → insert new rows (use this one)

DB credentials: stored in /Users/looftm/Documents/Claude/Projects/Update Staff Data/.env
  DB_HOST=[your-db-host]
  DB_USER=[your-db-user]
  DB_NAME=[your-database-name]
  DB_PASSWORD=<read from .env>

## Staff types managed by this script
Administrators, All, Non-Instructional Support, Pupil Services, Teachers.
Paraeducators are sourced from CBEDS (not STRE) and are handled separately — never touch them here.

## Step 1 — Determine expected file and year
Compute from the current date:
  yy_end = current_year % 100
  yy_start = yy_end - 1
  filename = f"stre{yy_start:02d}{yy_end:02d}.txt"
  year_in_db = current_year
Example: running in September 2026 → yy_start=25, yy_end=26 → stre2526.txt → Year 2026

## Step 2 — Check if already imported
Load the .env file and connect to MySQL using pymysql:
  SELECT COUNT(*) FROM public_datasets
  WHERE Dataset='Staff' AND Indicator='Staff'
    AND ItemDescription IN ('Administrators','All','Non-Instructional Support','Pupil Services','Teachers')
    AND Year=<year_in_db>
If count > 0: log "Year <year_in_db> STRE staff already imported ({count} rows). Nothing to do." and stop — do NOT re-import.

## Step 3 — Download the CDE file
Use Chrome (Claude in Chrome MCP) to navigate to https://www.cde.ca.gov/ds/ad/filesstretextdata.asp.
Find the download link for the expected filename (stre{yy_start}{yy_end}.txt or a .zip containing it).
Download it to /Users/looftm/Documents/Claude/Projects/Update Staff Data/.
If the file is not yet posted on the CDE page, log "File stre{yy_start}{yy_end}.txt not yet available on CDE website. Will retry next Monday." and stop — this is normal in early September.
If the file is a .zip, extract it to get the .txt file.

## Step 4 — Run the update script
Run in bash:
  cd "/Users/looftm/Documents/Claude/Projects/Update Staff Data"
  python3 update_stre_staff.py stre{yy_start:02d}{yy_end:02d}.txt
This script:
  - Processes the source file (filters Marin County, Gender=ALL, correct aggregate levels)
  - Generates a CSV and SQL import file
  - Connects to MySQL, DELETEs old STRE rows for that Year, and INSERTs new rows
  - Reads DB_PASSWORD from the .env file automatically

## Step 5 — Verify
Query the DB:
  SELECT COUNT(*) FROM public_datasets
  WHERE Dataset='Staff' AND Indicator='Staff'
    AND ItemDescription IN ('Administrators','All','Non-Instructional Support','Pupil Services','Teachers')
    AND Year=<year_in_db>
Report the row count. Expected: ~840–860 rows (school + district + county records × 5 staff types × 2 demographics).

## Success criteria
- The correct file was found and downloaded from CDE
- update_stre_staff.py ran without errors
- STRE staff rows for the new year are confirmed in the database
- Prior year's STRE rows were replaced (not duplicated)
- Paraeducator rows are unchanged

## Notes on STRE data rules
- Filter: County Code = '21' (Marin), Staff Gender = 'ALL'
- School records: Aggregate Level = 'S', School Name != 'District Office'
- District "All Schools": Aggregate Level = 'D', Charter = 'ALL', DASS = 'ALL', School Grade Span = 'ALL'
- County "All Schools": Aggregate Level = 'C', Charter = 'ALL', DASS = 'ALL', School Grade Span = 'ALL'
- PoC = Total - White - Not Reported (Two or More Races counts as PoC)
- Group_Total for Total rows = "All" staff type total for that entity
- Group_Total for PoC rows = Total - Not Reported for that staff type
- Do NOT hardcode credentials — always read from .env
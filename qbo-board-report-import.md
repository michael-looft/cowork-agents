---
name: qbo-board-report-import
description: Monthly: fetch QBO report CSVs from Gmail, upload to Drive, import into PCC Board Reports Google Sheet, and trigger PDF export
---

Each month, QuickBooks Online sends three CSV reports to [your-email@org.org]. Your job is to retrieve those CSV attachments, save them to Google Drive, import each into the correct tab in the PCC Board Reports Google Sheet, and trigger the PDF export. The task runs at 10 AM Eastern on the 12th — but may also fire on the 13th or 14th as a catch-up if the app was closed on the 12th.

---

## CATCH-UP CHECK — Run this first, every time

Before doing anything else, determine whether this task has already run successfully this month:

1. Navigate to the Board Reports Automation Google Drive folder:
   [your-google-drive-folder-url]

2. Check the last-modified date of "RAW - SOFP.csv" in that folder.

3. If that file exists AND its last-modified date is in the current month and year — the task already ran successfully this month. Log: "Monthly QBO report already completed for [Month Year]. Skipping." and stop immediately.

4. If the file is missing or was last modified in a prior month, proceed to Step 1 below.

---

## STEP 1 — Find the QBO report emails in Gmail

Use the Gmail search tool to search for emails with subject lines beginning with "QBO REPORT:" that arrived this month. A good search query is:
  subject:"QBO REPORT:" newer_than:5d

You are looking for exactly 3 emails. Each subject line will look like: "QBO REPORT: RAW - SOFP" (the part after the colon is the report name).

Read each matching email using the Gmail read message tool. Note the exact report name from each subject line (the portion after "QBO REPORT: " — trim any leading/trailing spaces). Also note the CSV attachment for each email.

If fewer than 3 matching emails are found, wait and retry the search a few minutes later (up to 3 retries). If after retries you still cannot find all 3, log a clear error and stop.

---

## STEP 2 — Save the CSV files to Google Drive

Navigate to the Board Reports Automation Google Drive folder:
  [your-google-drive-folder-url]

For each of the 3 CSV attachments, save the file into this folder. Name each file using its report name plus ".csv" — for example:
- "QBO REPORT: RAW - SOFP" → RAW - SOFP.csv
- "QBO REPORT: RAW - CF"   → RAW - CF.csv
- "QBO REPORT: SOA"        → RAW - SOA.csv

Note: the SOA report is named "RAW - SOA.csv" (not "SOA.csv") to align with the sheet tab name.

If a file with the same name already exists in the folder, replace/overwrite it.

---

## STEP 3 — Import each CSV into the Google Sheet

Open the PCC Board Reports - FY26 Google Sheet:
  [your-google-sheet-url]

For EACH of the 3 CSV files:

1. Click the sheet tab whose name exactly matches the report name (e.g., the tab "RAW - SOFP" for the file from "QBO REPORT: RAW - SOFP"). Note: the SOA report goes into the "RAW - SOA" tab — do NOT use the locked "SOA" tab.
2. Click on cell A1 to make it the active cell.
3. Go to File → Import.
4. In the Import dialog, search Google Drive for the matching CSV file by name and select it.
5. Set import options:
   - Import location: "Replace data at selected cell" (A1)
   - Separator type: "Detect automatically"
   - "Convert text to numbers, dates, and formulas": CHECKED
6. Click "Import data" and wait for the import to complete before moving to the next file.

Repeat for all 3 CSV files.

---

## STEP 4 — Run the Apps Script PDF export

After all 3 CSVs have been successfully imported:

1. In the Google Sheet, go to Extensions → Apps Script.
2. In the Apps Script editor, open "Print docs.gs" and select the function: exportBoardReportPDF()
3. Click Run and wait for execution to complete. The function will generate and save the board report PDF to Drive automatically.

---

## Success criteria

- All 3 CSV files are saved in the Board Reports Automation Drive folder with the correct names (RAW - SOFP.csv, RAW - CF.csv, RAW - SOA.csv).
- All 3 sheet tabs (RAW - SOFP, RAW - CF, RAW - SOA) have been refreshed with new data starting at cell A1.
- The exportBoardReportPDF() Apps Script ran without errors.

If any step fails, log exactly which step failed and what the error was.
---
name: cde-chronic-absence-annual-file-check
description: Check CDE's Absenteeism Data files page every 5 days for the new annual chronic absenteeism file; downloads and marks processed once found.
---

This is an automated run of a scheduled task. The user is not present to answer questions. For implementation details, execute autonomously without asking clarifying questions — make reasonable choices and note them in your output. "write" actions (e.g. MCP tools that send, post, create, update, or delete), only take them if the task file asks for that specific action. When in doubt, producing a report of what you found is the correct output.

Objective: Check for the newest annual chronic absenteeism file from the California Department of Education by constructing a direct download URL based on today's date. When a new year's file is found, download it into the user's "Update Chronic Absence Data" workspace folder and mark it processed. Once this year has been processed, do nothing on subsequent runs until the next release cycle.

Workspace folder: the user has selected a folder named "Update Chronic Absence Data". It will be mounted in this session under /sessions/<current-session-id>/mnt/Update Chronic Absence Data. Locate it dynamically with:
  ls -d /sessions/*/mnt/"Update Chronic Absence Data" 2>/dev/null | head -1
If no matching mount exists, log that and exit — do not prompt the user (this runs unattended).

State file: <workspace>/.cde-file-check-state.json
Schema:
  {
    "last_processed_release_year": 2025,        // most recent calendar year already downloaded (integer)
    "last_processed_at": "2026-11-03T09:02:11-08:00",
    "last_downloaded_files": ["chronicabsenteeism25.txt"],
    "last_check_at": "2026-11-03T09:02:11-08:00",
    "history": [ { "release_year": 2025, "processed_at": "...", "files": [...] } ]
  }
If the file doesn't exist, treat last_processed_release_year as null.

Date gating (belt-and-suspenders; cron already narrows the window):
1. Determine today's date.
2. If month is before October OR (month is October AND day < 15), update last_check_at in the state file and exit without attempting a download.
3. Otherwise proceed.

Determine the target year:
4. Use the following logic to determine the expected data year (as a 4-digit integer, e.g. 2025):
   - If today's month is October or later: target_year = current calendar year (e.g. November 2025 → 2025)
   - If today's month is before October: target_year = current calendar year minus 1 (e.g. March 2026 → 2025)
5. Derive the two-digit suffix: YY = last two digits of target_year (e.g. 2025 → "25")

Early-exit if already done:
6. If state.last_processed_release_year >= target_year (and last_processed_release_year is not null), update last_check_at and exit — the current year's file has already been downloaded.

Attempt the download:
7. Construct the primary URL: https://www3.cde.ca.gov/demo-downloads/attendance/chronicabsenteeismYY.txt (substitute actual YY value)
8. Attempt to download using curl in the Bash tool:
     curl -L -w "%{http_code}" -o "<workspace>/chronicabsenteeismYY.txt" "<URL>"
   Capture the HTTP status code from the output.
9. If the status code is 404 (or the file is empty/missing):
   a. Delete the empty/partial file if it was created.
   b. Subtract 1 from target_year, recompute YY, and retry once with the new URL.
   c. If the retry also returns 404 or fails, log the failure to .cde-file-check-log.txt and exit with a clear message — do not guess further.
10. If the download succeeded (status 200, file is non-empty), verify the file landed correctly.

Update state and log:
11. Update the state file:
   - last_processed_release_year = target_year (the year that actually worked)
   - last_processed_at = now (ISO 8601 with local offset)
   - last_downloaded_files = [filename just saved]
   - append an entry to history
   - last_check_at = now
12. Write a short summary line to <workspace>/.cde-file-check-log.txt with timestamp and what happened (e.g. "Downloaded chronicabsenteeism25.txt for year 2025" or "404 on year 2025, retried 2024 — also 404, giving up").

Output: a concise message describing what happened on this run (e.g. "No action — year 2025 already processed" or "Downloaded chronicabsenteeism25.txt for year 2025 into the workspace folder"). Include a computer:// link to any newly downloaded file so the user can open it.

Constraints:
- Do NOT click links using computer-use tools; use curl/Bash to download files.
- Do NOT re-download a file that is already present with the same name and non-zero size (idempotent).
- Do NOT ask the user questions — this runs unattended. If something unexpected happens, log the failure to .cde-file-check-log.txt and exit with a clear message.
- Keep runs quiet on days when nothing has changed.
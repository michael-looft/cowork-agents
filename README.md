# Cowork Agents

A collection of scheduled agent definitions for [Claude Cowork](https://www.anthropic.com/cowork), Anthropic's desktop automation tool. These agents run on a schedule to maintain a live education data dashboard, automate financial reporting, and monitor public data sources for annual updates.

## What Are Cowork Agents?

Cowork agents are natural-language instruction sets that tell Claude how to perform a recurring task autonomously — browsing the web, running scripts, accessing email, importing data into databases, and more. Each agent definition (a `SKILL.md` file) describes the task, the steps, the decision logic, and the error handling in plain language that Claude executes as a computer-use agent.

This is an emerging approach to workflow automation that sits between traditional scripting (rigid, requires code changes for every variation) and manual processes (reliable but not scalable). The agents here handle the judgment calls that make automation hard: checking whether a file has already been processed, falling back gracefully when a server blocks direct access, sleeping until the right time of year, and logging results in human-readable form.

## Agents in This Repository

### Annual Data Pipeline Agents

These agents monitor California Department of Education (CDE) public data sources and trigger the ETL pipeline when new files are posted each year.

| Agent | File | Schedule | What It Does |
|---|---|---|---|
| CDE Enrollment Check | `cde-enrollment-file-check.md` | Every 2 days, April-July | Checks for new Census Day Enrollment file; runs full pipeline when found; sleeps until next April |
| Chronic Absence Check | `cde-chronic-absence-annual-file-check.md` | Every 2 days, Nov-Feb | Same pattern for chronic absenteeism data |
| ACGR Graduation Check | `acgr-annual-download.md` | Every 2 days, Nov-Feb | Same pattern for graduation rate data |
| CAASPP Assessment Check | `caaspp-annual-update.md` | Annual | Checks for new CAASPP standardized test results |
| STRE Staff Update | `stre-staff-annual-update.md` | Annual | Processes annual educator staffing data |
| CBEDS Paraeducator Update | `cbeds-paraeducator-annual-update.md` | Annual | Processes paraeducator staffing data |

### Financial Reporting Agent

| Agent | File | Schedule | What It Does |
|---|---|---|---|
| QBO Board Report Import | `qbo-board-report-import.md` | Monthly (12th) | Retrieves QuickBooks CSV exports from Gmail, imports into Google Sheets, triggers PDF generation |

## How These Work Together

The data pipeline agents pair with Python scripts in the companion repository [cde-data-pipeline](https://github.com/michael-looft/cde-data-pipeline). The agent handles the scheduling, file detection, and error recovery logic; the Python scripts handle the actual data transformation and database import.

```
Cowork Agent (scheduler + orchestrator)
    ↓ detects new file
    ↓ calls Python script
Python Script (transform + import)
    ↓ generates CSV + SQL
    ↓ imports to MySQL
    ↓ reports back to agent
Cowork Agent (logs result, schedules next run)
```

## Key Patterns

**Smart scheduling:** Agents run frequently during the window when new data is expected, then sleep for the rest of the year. This avoids unnecessary checks while ensuring prompt detection when files drop.

**Idempotency:** Each agent checks whether the current cycle has already been completed before doing any work. Re-running an agent that already succeeded is always safe.

**Graceful fallback:** When direct MySQL access is blocked (common on shared hosting), agents fall back to phpMyAdmin for manual import rather than failing silently.

**Structured logging:** Results are written to markdown log files in the working folder, creating a human-readable audit trail of every run.

## Adapting These for Your Own Use

These agents are designed for a specific data infrastructure (CDE public datasets, MySQL on shared hosting, Google Workspace for financial reporting). To adapt them:

1. Replace the database connection details with your own credentials pattern
2. Update the working folder paths to match your Cowork project structure
3. Adjust the scheduling windows to match when your data sources publish
4. Modify the sanity-check ranges to reflect your expected data volumes

The instruction style and decision logic are the reusable parts - the specific URLs, table names, and thresholds are all configurable.

## Context

These agents are part of a broader data infrastructure built at [Marin Promise Partnership](https://www.marinpromisepartnership.org), a collective impact backbone organization advancing educational equity in Marin County, California.

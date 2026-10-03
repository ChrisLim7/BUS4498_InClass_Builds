# Notify Organizer of Run Issue Task Specification

## Basic Information

- **Task ID:** H1
- **Task name:** Notify Organizer of Run Issue
- **Task type:** Verify
- **Automation level:** L0 — the CPVC event lead organizer checks and fixes the problem. The planned tool only supports the handoff.
- **Task owner:** CPVC event lead organizer

## 1. Task Description

When a run cannot produce a trustworthy forecast, the organizer receives an alert with the run ID, checkpoint, failing task, error, and what to check. This happens when T1 data is unusable, the T4 calculation fails, T6 cannot save or notify, or T7 review is blocked. The organizer checks and fixes the source of the problem, for example the registration export, the past-event table, or the prompt template, and records a short fix note.

No forecast is produced for the affected run. The fix is used by the next scheduled checkpoint. If the 1-day checkpoint fails, the last approved forecast from the 3-day checkpoint stays in effect. The workflow needs this task so failures are visible and never reported as a finished forecast.

**Human response deadline:** Before the next scheduled checkpoint; for a 1-day checkpoint failure, within 6 hours. If the deadline is missed, the issue stays open, the CPVC club president is notified, and nothing is treated as fixed or approved.

## 2. Inputs

### Input 1

- **Input name:** Run issue report
- **Contents and format:** Run ID, checkpoint, failing task (T1, T4, T6, or T7), error or Data load status, the affected source or record, and what has already been tried.
- **Source:** T1: Collect Registration Data (through D1), T4: Aggregate Predicted Headcount, T6: Present Forecast to Organizers, or T7: Review and Finalize Forecast

- **If a required input is missing or invalid:** If the report is missing details, still send the alert with the run ID and failing task, and mark the cause unknown.

## 3. Outputs

### Output 1

- **Output name:** Run issue record
- **Contents and format:** Run ID, failing task, alert status (sent, failed, or unknown), organizer acknowledgement time, fix note, and status (open, fixed, or overdue).
- **Next task or recipient:** Stop: Run incomplete; the CPVC club president if the deadline is missed. The fix note is visible to T1 at the next checkpoint.
- **Complete when:** The alert has been sent and the issue is recorded as open, or the organizer has recorded a fix note.

## 4. Planned Tools

### Tool 1

- **Tool name:** `send_run_issue_alert`
- **Input:** Run issue report
- **Output:** Run issue record
- **Implementation Route:** Web API calls (message to the CPVC organizer group channel), plus a database write keyed on run ID and failing task
- **Integration approach:** Direct integration
- **Role in this task:** Delivers the alert and saves the issue record; it cannot fix data or restart a run.
- **Task timeout:** Human response deadline: before the next scheduled checkpoint, or within 6 hours after a 1-day checkpoint failure. The alert send may take at most 10 seconds.
- **Maximum retries:** Not applicable — manual task. The supporting alert is sent once per run ID and failing task; it may be resent once only if the channel explicitly rejects it before acceptance and no accepted alert is logged for that key.
- **Retry only when:** Not applicable for the organizer's work. For the supporting alert, see Maximum retries; an uncertain send outcome is never resent.
- **On timeout, exhausted retries, or an error that cannot be retried:** If the organizer misses the deadline, mark the issue overdue and notify the CPVC club president. If the alert itself fails or its outcome is unknown, keep the issue open and visible on the HackTrack review page. The run stays incomplete and is never reported as a finished forecast.

# Analyze Attendance Signals Task Specification

## Basic Information

- **Task ID:** T2
- **Task name:** Analyze Attendance Signals
- **Task type:** Reason
- **Automation level:** L1 — fixed rules assign each registrant to exactly one signal group; no model is used.
- **Task owner:** HackTrack workflow controller

## 1. Task Description

For one registrant at a time, calculate the attendance signals from the Registration record: days since registration, whether the RSVP is confirmed, whether the RSVP was updated in the last 7 days, whether a confirmation prompt was sent, and whether the RSVP changed after that prompt. Then assign exactly one signal group using these fixed rules, applied in order:

1. **Cancelled:** current RSVP status is cancelled.
2. **Confirmed:** current RSVP status is confirmed, including after a prompt.
3. **Unsure:** current RSVP status is unsure.
4. **Prompted, no update:** a prompt was sent and the RSVP has not changed since.
5. **Registered, no update:** none of the above.

When two status records conflict, the one with the most recent timestamp wins. If timestamps are missing or tied, the conflict is noted for T3 rather than resolved here. A group that did not exist at the last event, such as Prompted, no update, is still assigned; T3 handles its small past sample under its fixed rule.

The workflow needs this task so that T3 compares each registrant only with past registrants who showed the same behavior. No personal attributes are used.

## 2. Inputs

### Input 1

- **Input name:** Registration record
- **Contents and format:** Pseudonymous registrant ID, registration date, RSVP status, last RSVP update date, prompt log entry if any, checkpoint, event ID, and run ID.
- **Source:** T1: Collect Registration Data

### Input 2

- **Input name:** Past-event base rates
- **Contents and format:** Signal group definitions and the version of the past-event table used for this run.
- **Source:** T1: Collect Registration Data

- **If a required input is missing or invalid:** If the RSVP status or registration date is missing, record the registrant as "Signal error: unscored" with the missing field and send the case to D4 so the loop continues. T4 counts the registrant separately, and T6 shows it to the CPVC event lead organizer. Do not guess a group.

## 3. Outputs

### Output 1

- **Output name:** Attendance signal summary
- **Contents and format:** Registrant ID, run ID, assigned signal group, the signal values used, and any conflicting or missing signals noted; or "Signal error: unscored" with the reason.
- **Next task or recipient:** T3: Score Attendance Probability; a signal error goes to D4: More registrants? and is counted by T4.
- **Complete when:** The registrant has exactly one signal group with its supporting signals listed, or a signal error is recorded.

## 4. Planned Tools

### Tool 1

- **Tool name:** `assign_signal_group`
- **Input:** Registration record; Past-event base rates
- **Output:** Attendance signal summary
- **Implementation Route:** Functions/scripts (deterministic rules listed in Section 1)
- **Integration approach:** Direct integration
- **Role in this task:** Calculates signal values and assigns one group per registrant.
- **Task timeout:** 5 seconds per registrant.
- **Maximum retries:** 0
- **Retry only when:** Not applicable. The rules are deterministic, so repeating the same input would give the same result.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record "Signal error: unscored" with the error and send the case to D4. Do not pass an unchecked group to T3.

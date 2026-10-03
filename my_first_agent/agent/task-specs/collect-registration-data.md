# Collect Registration Data Task Specification

## Basic Information

- **Task ID:** T1
- **Task name:** Collect Registration Data
- **Task type:** Retrieve
- **Automation level:** L1 — fixed, read-only data collection under scheduled controller rules; no model is used.
- **Task owner:** HackTrack workflow controller; the CPVC event lead organizer resolves data problems through H1.

## 1. Task Description

At each scheduled checkpoint (7, 3, and 1 day before the CPVC AI Hackathon), read the current registration list, each registrant's RSVP status and last RSVP update date, the confirmation-prompt log from earlier checkpoints, and CPVC's aggregated attendance rates from the last build event. Before passing records on, replace each registrant's name and email address with a pseudonymous registrant ID; contact details stay in the registration system and are read again only by T5 at send time. Then check that required fields are present and that the number of records loaded matches the number in the registration system.

This task does not change registrations or RSVP statuses and does not contact anyone. The workflow needs it because every later task depends on current, complete, privacy-safe records. Data is usable when all three sources are readable, the counts match, the past-event table is present, and no more than 10% of registrant records are missing a required field. A single record with a missing field is still passed on with the gap marked, and T2 records that registrant as unscored.

## 2. Inputs

### Input 1

- **Input name:** Checkpoint trigger
- **Contents and format:** Event ID, checkpoint (7, 3, or 1 day before the event), scheduled start time, run ID, and whether another HackTrack run is already active.
- **Source:** Scheduled checkpoint trigger

### Input 2

- **Input name:** CPVC registration records
- **Contents and format:** One row per registrant with name, email address, registration timestamp, RSVP status (registered, confirmed, unsure, or cancelled), and last RSVP update timestamp. Names and email addresses are used only to create the pseudonymous ID and are not passed on.
- **Source:** CPVC event registration system

### Input 3

- **Input name:** Confirmation-prompt log
- **Contents and format:** One row per prompted registrant with event ID, registrant ID, date sent, and send status (sent, failed, unknown, or not sent). An empty log at the 7-day checkpoint is valid.
- **Source:** T5: Send Single Confirmation Prompt (earlier checkpoints)

### Input 4

- **Input name:** Past-event attendance table
- **Contents and format:** For each signal group, the number of registrants, number who attended, and attendance rate at CPVC's last build event, plus the overall attendance-to-registration rate (about 40%). Aggregated counts only; no individual past attendee records.
- **Source:** CPVC organizers' post-event records, entered once before registration opens

- **If a required input is missing or invalid:** If another run is active, log the trigger as skipped and start no run. If the event ID is unknown, the registration system or past-event table cannot be read, the counts do not match, or more than 10% of records are missing required fields, set Data load status to unusable and route the run through D1 to H1: Notify Organizer of Run Issue. Never pass a partial list forward as if it were complete.

## 3. Outputs

### Output 1

- **Output name:** Registration record
- **Contents and format:** One record per registrant: pseudonymous registrant ID, registration date, current RSVP status, last RSVP update date, confirmation-prompt log entry if any, checkpoint, event ID, and run ID. Any missing field is marked as missing. No names or email addresses.
- **Next task or recipient:** T2: Analyze Attendance Signals and T3: Score Attendance Probability
- **Complete when:** Every registrant in the registration system has exactly one Registration record for this run.

### Output 2

- **Output name:** Past-event base rates
- **Contents and format:** Signal group definitions with attendance rate and sample size for each group, plus the overall past attendance rate.
- **Next task or recipient:** T2: Analyze Attendance Signals, T3: Score Attendance Probability, and T4: Aggregate Predicted Headcount
- **Complete when:** The table has been read and its version is recorded for this run.

### Output 3

- **Output name:** Data load status
- **Contents and format:** Usable or unusable, number of registrants loaded versus number in the registration system, list of missing fields, and any read error with the source that failed.
- **Next task or recipient:** D1: Registration data available and readable? (usable → T2; unusable → H1)
- **Complete when:** The status is recorded for this run.

## 4. Planned Tools

### Tool 1

- **Tool name:** `retrieve_registration_records`
- **Input:** Checkpoint trigger; CPVC registration records; Confirmation-prompt log
- **Output:** Registration record; Data load status
- **Implementation Route:** Database queries (read-only access to the CPVC registration table and prompt log)
- **Integration approach:** Direct integration
- **Role in this task:** Reads current registrations and the prompt log, and replaces names and emails with pseudonymous registrant IDs.
- **Task timeout:** 60 seconds for the whole T1 run, including all tools and retries; each query may take at most 15 seconds.
- **Maximum retries:** 1
- **Retry only when:** A temporary connection error or query timeout occurs. Wait 5 seconds and repeat the same read for the same run ID and checkpoint. The tool is read-only, so a retry cannot change or duplicate records.
- **On timeout, exhausted retries, or an error that cannot be retried:** Set Data load status to unusable, record the failed source and error, and route the run through D1 to H1. Do not pass partial records forward.

### Tool 2

- **Tool name:** `load_past_event_base_rates`
- **Input:** Past-event attendance table
- **Output:** Past-event base rates; Data load status
- **Implementation Route:** Database queries (read-only)
- **Integration approach:** Direct integration
- **Role in this task:** Loads the aggregated past-event rates once per run.
- **Task timeout:** Within the 60-second T1 limit; each query may take at most 10 seconds.
- **Maximum retries:** 1
- **Retry only when:** A temporary connection error or query timeout occurs. Wait 5 seconds and retry. Do not retry if the table is missing or empty; that needs an organizer fix. Read-only, so no duplicates.
- **On timeout, exhausted retries, or an error that cannot be retried:** Set Data load status to unusable with the reason "past-event table unavailable" and route to H1. Do not substitute guessed rates.

### Tool 3

- **Tool name:** `check_data_completeness`
- **Input:** Registration record; Past-event base rates
- **Output:** Data load status
- **Implementation Route:** Functions/scripts (deterministic count and required-field checks)
- **Integration approach:** Direct integration
- **Role in this task:** Compares record counts, marks missing fields, and applies the 10% usability rule.
- **Task timeout:** Within the 60-second T1 limit; at most 5 seconds.
- **Maximum retries:** 0
- **Retry only when:** Not applicable.
- **On timeout, exhausted retries, or an error that cannot be retried:** Set Data load status to unusable with the reason "completeness check failed" and route to H1. Do not mark the data usable without the check.

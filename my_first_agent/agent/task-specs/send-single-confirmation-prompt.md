# Send Single Confirmation Prompt Task Specification

## Basic Information

- **Task ID:** T5
- **Task name:** Send Single Confirmation Prompt
- **Task type:** Act
- **Automation level:** L1 — sends one fixed, organizer-approved message; no model drafts or personalizes it.
- **Task owner:** HackTrack workflow controller; CPVC organizers approve the message template before registration opens.

## 1. Task Description

For a registrant that D3 routes here (insufficient confidence, never prompted for this event, and not the 1-day checkpoint), look up the registrant's email address by registrant ID and send one fixed message. The message asks the registrant to confirm or update their RSVP through the registration link. It asks no new personal questions. The registrant's answer arrives as an RSVP update in the registration system, which T1 reads at the next checkpoint, so no reply parsing is needed.

A registrant is never prompted twice for the same event. Before sending, the task writes a "sending" entry keyed on event ID and registrant ID; a second entry with the same key is rejected. If the send outcome is unknown, the registrant is treated as prompted and is not contacted again.

The workflow needs this task because one short prompt is the least intrusive way to clarify the most uncertain registrants, which supports the system goal of avoiding excessive communication.

## 2. Inputs

### Input 1

- **Input name:** Prompt candidate
- **Contents and format:** Registrant ID, event ID, run ID, checkpoint, probability, and confidence from the T3 outbound deliverable, plus D3's eligibility result.
- **Source:** T3: Score Attendance Probability, routed through D2 and D3

### Input 2

- **Input name:** Confirmation-prompt log
- **Contents and format:** Existing prompt entries for this event, keyed on event ID and registrant ID.
- **Source:** T1: Collect Registration Data (Registration record) and this task's own earlier entries

### Input 3

- **Input name:** Approved prompt template
- **Contents and format:** Fixed subject line, message body, and RSVP update link approved by CPVC organizers.
- **Source:** CPVC organizers, before registration opens

- **If a required input is missing or invalid:** If the template is missing, the registrant already has a prompt log entry, or the checkpoint is the 1-day checkpoint, do not send. Record "not sent: ineligible" and send the registrant to D4 flagged as uncertain at their current probability.

## 3. Outputs

### Output 1

- **Output name:** Prompt send record
- **Contents and format:** Event ID, registrant ID, run ID, checkpoint, timestamp, and status (sent, failed, unknown, or not sent: ineligible). No email address is stored in this record.
- **Next task or recipient:** D4: More registrants? (the registrant is counted at their current probability); the Confirmation-prompt log used by T1 at the next checkpoint; failed and unknown counts appear in T6.
- **Complete when:** Exactly one prompt log entry exists for this event and registrant with a final status.

## 4. Planned Tools

### Tool 1

- **Tool name:** `lookup_registrant_contact`
- **Input:** Prompt candidate
- **Output:** Contact address used only inside this task (not stored in any output)
- **Implementation Route:** Database queries (read-only lookup by registrant ID in the registration system)
- **Integration approach:** Direct integration
- **Role in this task:** Finds the one email address needed to send the prompt.
- **Task timeout:** 20 seconds for the whole T5 run, including all tools and retries; this lookup may take at most 5 seconds.
- **Maximum retries:** 1
- **Retry only when:** A temporary connection error or timeout occurs. Wait 2 seconds and retry. Read-only, so no duplicates.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record "failed: contact lookup" in the Prompt send record and send the registrant to D4 as uncertain. Do not send without a verified address.

### Tool 2

- **Tool name:** `send_confirmation_prompt`
- **Input:** Approved prompt template; Prompt candidate; Confirmation-prompt log
- **Output:** Prompt send record
- **Implementation Route:** Web API calls (CPVC club email service send API)
- **Integration approach:** Direct integration
- **Role in this task:** Writes the "sending" log entry keyed on event ID and registrant ID, then sends the fixed message once.
- **Task timeout:** Within the 20-second T5 limit; the send call may take at most 10 seconds.
- **Maximum retries:** 1
- **Retry only when:** The email service explicitly rejects the request before accepting the message (for example, a temporary rate-limit error) and the prompt log shows no accepted send for this event and registrant. Wait 2 seconds before retrying. If the request times out or the response is unclear, the outcome is uncertain: do not retry.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record status as failed (explicit rejection) or unknown (uncertain outcome). Treat unknown as already prompted so the registrant is never contacted twice. Send the registrant to D4 as uncertain at their current probability. Do not record the prompt as sent unless the email service accepted it.

### Tool 3

- **Tool name:** `record_prompt_outcome`
- **Input:** Prompt send record
- **Output:** Prompt send record (saved and read back)
- **Implementation Route:** Database queries (upsert keyed on event ID and registrant ID)
- **Integration approach:** Direct integration
- **Role in this task:** Saves the final send status so T1 and later checkpoints can see it.
- **Task timeout:** Within the 20-second T5 limit; at most 5 seconds.
- **Maximum retries:** 1
- **Retry only when:** The write fails with a temporary error and a read-back on the same key shows the final status has not been saved. Because the write is an upsert on that key, repeating it cannot create a second entry.
- **On timeout, exhausted retries, or an error that cannot be retried:** Keep the "sending" entry in place so no second prompt can be sent, mark the outcome unknown in the run results, and show it in T6.

# Score Attendance Probability Task Specification

> **Level 3 task:** Intermediate findings decide what the agent does next for each registrant. It may verify the registrant's signal group, reconcile conflicting or missing RSVP signals, or compare the result with the previous checkpoint before it forms a probability and confidence level. These choices are bounded, and unresolved cases go to the CPVC event lead organizer.

```yaml
# BASIC INFORMATION
task_id: "T3"
task_name: "Score Attendance Probability"
task_owner: "HackTrack agent; the CPVC event lead organizer retains final authority over the forecast"

# Agent Inference Configuration
Provider: Groq
Model: "openai/gpt-oss-120b"
Role: Interpret one registrant's supplied signals, select the next permitted subtask, and explain the probability and confidence produced by the fixed scoring rule.
Maximum inference requests per task run: 4
On inference failure or exhausted limits: Record the unresolved status and hand the case to the CPVC event lead organizer.
```

## 1. Task Goal

- **Objective:** For one registrant at the current countdown checkpoint, produce an evidence-backed attendance probability between 0 and 1 and a confidence level of sufficient or insufficient, so that D2 can either add the registrant directly to the predicted headcount or route them to a single confirmation prompt. The probability must come from CPVC's past-event attendance rates for the registrant's signal group, not from the model's judgment. Use only RSVP and registration-timing signals; never use names, email addresses, majors, or other personal details. If signals conflict or are missing and cannot be resolved, record the registrant as unscored rather than guessing.

## 2. Inbound Inputs

### Input 1

- **Input name:** Registration record
- **What it contains:** Pseudonymous registrant ID, registration date, current RSVP status (registered, confirmed, unsure, or cancelled), date of the last RSVP update, confirmation-prompt log (date sent and response or no response), current checkpoint (7, 3, or 1 day before the event), and run ID. No names, email addresses, or other personal details are passed to this task.
- **Source:** T1: Collect registration data

### Input 2

- **Input name:** Attendance signal summary
- **What it contains:** The signal group T2 assigned to this registrant, the signals behind it (days since registration, whether the RSVP was confirmed or updated, and any prompt response), and any conflicting or missing signals T2 noted.
- **Source:** T2: Analyze attendance signals

### Input 3

- **Input name:** Past-event base rates
- **What it contains:** For each signal group, the attendance rate and the number of registrants in that group at CPVC's last build event, plus the overall attendance-to-registration rate (about 40%). These are aggregated counts only; no individual past attendee records are included.
- **Source:** T1: Collect registration data (loaded once per run)

### Input 4

- **Input name:** Prior checkpoint score
- **What it contains:** This registrant's probability, confidence, and signal group from the previous checkpoint of the same event. A valid empty value is expected at the 7-day checkpoint and is different from an unreadable record.
- **Source:** T3: Score attendance probability (previous checkpoint run)

## 3. Tool Permissions and Boundaries

### Task-Wide Limits

- **Total task timeout:** 30 seconds for one registrant's task run, including all tool calls, retries, waiting, and reasoning. A tool call or retry does not restart this clock.
- **Maximum tool calls:** 8 calls across all tools during one task run; retries count toward this total.

Tools may use only the supplied inputs for this registrant and this event. They may not read names, email addresses, or other personal details; search the web or social media; contact the registrant (only T5 may send a prompt); change a registrant's RSVP status; or edit the past-event base-rate table. The model classifies signals and explains the result. The calculation tool sets the probability and confidence using the fixed rule in Section 4; the model does not choose a number.

### Tool 1

- **Tool name:** `retrieve_registrant_signals`
- **Input:** Registration record; Attendance signal summary; Prior checkpoint score
- **Output:** Source-labeled signals for the Evidence summary, plus missing, stale, or conflicting signals for Unresolved issues
- **Implementation Route:** Database queries (read-only, restricted to this event's pseudonymous registration and score tables)
- **Integration approach:** Direct integration
- **Role in this task:** Supports Verify Signal Group, Reconcile Conflicting or Missing Signals, and Compare With Prior Checkpoint
- **Task timeout:** Subject to the 30-second task deadline. Each call may take at most 5 seconds or the remaining task time, whichever is shorter.
- **Maximum retries:** 1
- **Retry only when:** A temporary database read error or connection timeout prevents completion. Wait 2 seconds and retry only if time and call budget remain. Do not retry denied access, an invalid registrant ID, or a confirmed missing record. An empty prior checkpoint score at the 7-day checkpoint is not a read failure. This tool is read-only, so a retry cannot create duplicate records or messages.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record the input, attempted query, failure type, and number of attempts in Subtasks performed and Unresolved issues. Set Status to "Unscored: operational error" and Result to "unscored." Use the Handoff note to state which record the CPVC event lead organizer needs to check. Do not treat an unreadable record as evidence that the registrant will not attend.

### Tool 2

- **Tool name:** `lookup_past_event_base_rate`
- **Input:** Past-event base rates; Attendance signal summary
- **Output:** The attendance rate and sample size for the registrant's signal group, for the Evidence summary
- **Implementation Route:** Database queries (read-only, aggregated past-event table)
- **Integration approach:** Direct integration
- **Role in this task:** Supports Verify Signal Group and Form Probability and Confidence
- **Task timeout:** Subject to the 30-second task deadline. Each call may take at most 5 seconds or the remaining task time, whichever is shorter.
- **Maximum retries:** 1
- **Retry only when:** A temporary database read error or connection timeout prevents completion. Wait 2 seconds and retry only if time and call budget remain. Do not retry when the signal group does not exist in the table; that is a data issue, not a temporary error. This tool is read-only, so a retry cannot create duplicate records.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record the signal group and failure type in Unresolved issues. Set Status to "Unscored: operational error" and Result to "unscored." State in the Handoff note that the past-event table must be checked by the CPVC event lead organizer. Do not substitute a guessed rate.

### Tool 3

- **Tool name:** `calculate_attendance_probability`
- **Input:** Attendance signal summary; Past-event base rates
- **Output:** Result or recommendation (probability and confidence level)
- **Implementation Route:** Functions/scripts (deterministic calculation using the fixed rule in Section 4)
- **Integration approach:** Direct integration
- **Role in this task:** Supports Form Probability and Confidence
- **Task timeout:** Subject to the 30-second task deadline. Each call may take at most 2 seconds or the remaining task time, whichever is shorter.
- **Maximum retries:** 0
- **Retry only when:** Not applicable. A second calculation after Reconcile Conflicting or Missing Signals changes the signal group is a new invocation, not a retry, and must stay within the Section 4 limits and the task-wide budgets.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record the failed calculation and its inputs in Subtasks performed and Unresolved issues. Set Status to "Unscored: operational error" and Result to "unscored." Do not let the model estimate a probability in place of the calculation.

### Tool 4

- **Tool name:** `save_attendance_score`
- **Input:** Registration record (run ID, registrant ID, and checkpoint as the record key)
- **Output:** Result or recommendation (saved score record, confirmed by read-back)
- **Implementation Route:** Database queries (write, as an upsert keyed on run ID + registrant ID + checkpoint)
- **Integration approach:** Direct integration
- **Role in this task:** Supports Form Probability and Confidence by saving the score so T4 can aggregate it and the next checkpoint can compare against it
- **Task timeout:** Subject to the 30-second task deadline. Each call may take at most 5 seconds or the remaining task time, whichever is shorter.
- **Maximum retries:** 1
- **Retry only when:** The write returns a temporary error or times out, and a read-back using the same run ID + registrant ID + checkpoint key confirms that no score record exists. Wait 2 seconds before retrying. Because the write is an upsert on that key, repeating it cannot create a second score for the same registrant and checkpoint. If the read-back fails, or finds a record that differs from the attempted score, the outcome is uncertain: do not retry.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record the attempted score, key, failure type, and read-back result in Unresolved issues. Set Status to "Unscored: operational error" so T4 does not count an unsaved or uncertain score. Use the Handoff note to ask the CPVC event lead organizer to check the score table for that key. Do not continue as if the score was saved.

## 4. How the Agent Should Reason

### Permitted Subtask 1

- **Subtask name:** Verify Signal Group
- **Subtask description:** Check that the signal group T2 assigned matches the raw Registration record (RSVP status, last update date, and prompt response) under the group definitions in the past-event table.
- **Subtask boundary:** Use only the defined signal groups. Do not create a new group or use any personal attribute to choose one.
- **Retry limits:** Perform once. Repeat once only if Reconcile Conflicting or Missing Signals changes the signals.

### Permitted Subtask 2

- **Subtask name:** Reconcile Conflicting or Missing Signals
- **Subtask description:** When signals disagree (for example, the record shows "confirmed" but a later cancellation date) or a needed date is missing, use timestamps in the supplied inputs to decide which signal is most recent and valid.
- **Subtask boundary:** Use only supplied records. Do not guess a missing date, contact the registrant, or treat silence as a cancellation. If the conflict cannot be resolved from the records, leave it unresolved.
- **Retry limits:** Examine no more than two conflicts or missing signals before handing the case off.

### Permitted Subtask 3

- **Subtask name:** Compare With Prior Checkpoint
- **Subtask description:** If a prior checkpoint score exists, compare it with the new result. When the probability changes by more than 25 percentage points, identify the signal that changed.
- **Subtask boundary:** Explain the change; do not overwrite or edit the earlier score. If no signal explains a large change, record it as an unresolved issue rather than adjusting the number.
- **Retry limits:** Perform once per task run.

### Permitted Subtask 4

- **Subtask name:** Form Probability and Confidence
- **Subtask description:** Call `calculate_attendance_probability`, save the result with `save_attendance_score`, and write a short plain-language explanation of the signals and base rate behind the score.
- **Subtask boundary:** Do not invent a probability, weight, or signal group, and do not decide whether to send a prompt; D2 and D3 make that routing decision.
- **Retry limits:** Recalculate once only after another subtask changes the signal group.

**Fixed scoring rule (applied by Tool 3):** The probability equals the attendance rate of the registrant's signal group at the last CPVC build event. If that group had fewer than 15 registrants, use the overall past attendance rate (about 0.40) and set confidence to insufficient. Confidence is **sufficient** only when the group had at least 15 past registrants, no conflict remains unresolved, and the probability is 0.30 or below or 0.70 or above. Every other case is **insufficient**, because a registrant in the uncertain middle band is the one a single confirmation prompt is most likely to clarify.

- **Decision guidance:** After each subtask, choose the permitted subtask most likely to resolve the most important remaining uncertainty. Do not follow a fixed sequence. If no permitted subtask can make useful progress, stop and hand the case to the CPVC event lead organizer.

## 5. When to Stop or Hand Off to a Human

- **Stop successfully when:** The registrant has a saved, read-back-confirmed probability and confidence level from the fixed rule, and the Evidence summary shows the signals, signal group, and base rate behind it.
- **Hand off early when:** A conflict or missing signal is still unresolved after two attempts; the signal group is not in the past-event table; a large change since the prior checkpoint has no explanation; or a time, tool-call, or inference limit is reached. Record the registrant as unscored so T4 counts them separately instead of as zero or as a confirmed attendee.
- **Hand off to:** The CPVC event lead organizer, who sees all unscored registrants in the T6 forecast review.

Stop at the first applicable budget limit or handoff condition. While awaiting review, take no further action on this registrant.

## 6. Outbound Deliverable

- **Status:** Completed; Unscored: needs organizer review; or Unscored: operational error. Include the run ID, registrant ID, and checkpoint.
- **Result or recommendation:** Attendance probability (0 to 1) and confidence level (sufficient or insufficient), or "unscored" when no supported result was produced. This is a forecast input, not a decision about the registrant.
- **Evidence summary:** The signals used and the input each came from, the signal group, the past-event base rate and its sample size, and the change from the prior checkpoint when one exists.
- **Subtasks performed:** The permitted subtasks completed, including any repeated subtask.
- **Unresolved issues:** Remaining conflicting or missing signals or failed operations. Write `none` only for a completed task.
- **Handoff note:** The reason for stopping and exactly what the CPVC event lead organizer needs to check. Write `Not applicable` for a completed task.
- **Next task or recipient:** A completed score goes to D2: Confidence sufficient? (sufficient → added to the T4 headcount; insufficient → D3, then T5 or an uncertain flag). An unscored registrant is recorded as an exception that T4 counts separately and T6 shows to the CPVC event lead organizer.

# Aggregate Predicted Headcount Task Specification

## Basic Information

- **Task ID:** T4
- **Task name:** Aggregate Predicted Headcount
- **Task type:** Reason
- **Automation level:** L1 — fixed calculation; no model is used.
- **Task owner:** HackTrack workflow controller; the CPVC event lead organizer approves the result in T7.

## 1. Task Description

After every registrant has been processed, combine the per-registrant results into one forecast using a fixed calculation:

- **Expected headcount:** the sum of all scored registrants' probabilities, plus the overall past attendance rate (about 0.40) for each unscored registrant.
- **Low estimate:** the sum of sufficient-confidence probabilities, plus each insufficient-confidence probability minus 0.20 (not below 0), plus 0 for each unscored registrant.
- **High estimate:** the sum of sufficient-confidence probabilities, plus each insufficient-confidence probability plus 0.20 (not above 1), plus 1 for each unscored registrant.

Report counts of registrants by group and confidence, and the numbers of uncertain, unscored, and prompted registrants. The total of scored and unscored registrants must equal the number of registrants loaded in T1.

When T7 requests a revision, apply only the organizer's stated assumption, such as a walk-in buffer percentage. Label it as an organizer assumption and keep the original numbers visible. No more than 2 revisions are allowed per run.

The workflow needs this task because organizers plan food, drinks, and swag from one total with an honest range, not from per-person scores.

## 2. Inputs

### Input 1

- **Input name:** Attendance score results
- **Contents and format:** For each registrant: the T3 outbound deliverable (registrant ID, status, probability, and confidence), the D3 uncertain flag when applied, and T2 or T3 signal errors recorded as unscored.
- **Source:** T3: Score Attendance Probability (through D2 and D3) and T2: Analyze Attendance Signals for signal errors

### Input 2

- **Input name:** Past-event base rates
- **Contents and format:** Overall past attendance rate and table version.
- **Source:** T1: Collect Registration Data

### Input 3

- **Input name:** Organizer revision request
- **Contents and format:** Revision number (1 or 2), the assumption to apply, and the organizer's reason. Absent on the first calculation.
- **Source:** T7: Review and Finalize Forecast

- **If a required input is missing or invalid:** If the number of results does not equal the number of registrants loaded in T1, or a revision request is a third request or has no stated assumption, do not produce a forecast. Send the case to H1: Notify Organizer of Run Issue.

## 3. Outputs

### Output 1

- **Output name:** Headcount forecast
- **Contents and format:** Run ID, checkpoint, forecast version, expected headcount, low and high estimates, total registrants, counts by signal group and confidence, numbers of uncertain, unscored, and prompted registrants, the past-event table version, and any labeled organizer assumption with the original numbers.
- **Next task or recipient:** T6: Present Forecast to Organizers
- **Complete when:** All three numbers are calculated, every registrant is accounted for, and the forecast version is recorded.

## 4. Planned Tools

### Tool 1

- **Tool name:** `calculate_headcount_forecast`
- **Input:** Attendance score results; Past-event base rates; Organizer revision request
- **Output:** Headcount forecast
- **Implementation Route:** Functions/scripts (deterministic calculation described in Section 1)
- **Integration approach:** Direct integration
- **Role in this task:** Calculates the expected headcount, the range, and the counts; applies only an organizer-stated revision assumption.
- **Task timeout:** 30 seconds per calculation.
- **Maximum retries:** 0
- **Retry only when:** Not applicable. A recalculation after a T7 revision request is a new invocation, not a retry, and is limited to 2 per run.
- **On timeout, exhausted retries, or an error that cannot be retried:** Produce no forecast, record the error, and send the case to H1: Notify Organizer of Run Issue. The run cannot complete without a forecast.

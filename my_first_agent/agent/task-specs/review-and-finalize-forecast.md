# Review and Finalize Forecast Task Specification

## Basic Information

- **Task ID:** T7
- **Task name:** Review and Finalize Forecast
- **Task type:** Decide
- **Automation level:** L0 — the CPVC event lead organizer decides whether the forecast is used.
- **Task owner:** CPVC event lead organizer

## 1. Task Description

The event lead reviews the forecast and its evidence on the HackTrack review page and chooses one of four actions:

- **Approve:** use the expected headcount and range as presented.
- **Approve with override:** enter a different planning number and a reason.
- **Request revision:** state one assumption for T4 to apply, such as a walk-in buffer. No more than 2 revisions are allowed per run.
- **Reject:** enter a reason.

Only an approved forecast, or an approved override, guides food, drink, and swag orders. HackTrack does not place orders or contact vendors. The workflow needs this task because spending the club's budget is a human decision.

**Human response deadline:** 24 hours after the forecast appears on the review page. A missed deadline is not approval. The run ends as incomplete, the last approved forecast from an earlier checkpoint (if any) stays in effect, and the case goes to the CPVC club president.

## 2. Inputs

### Input 1

- **Input name:** Forecast review packet
- **Contents and format:** Saved forecast version, review page link, evidence and counts, notification status, and the time the forecast appeared on the review page.
- **Source:** T6: Present Forecast to Organizers

- **If a required input is missing or invalid:** If the review page cannot load the forecast or the version shown does not match the saved version, the organizer cannot approve it. Record "review blocked" and send the case to H1: Notify Organizer of Run Issue.

## 3. Outputs

### Output 1

- **Output name:** Forecast decision
- **Contents and format:** Run ID, forecast version, decision (approve, approve with override, request revision, or reject), override number and reason or revision assumption when applicable, organizer name, and timestamp.
- **Next task or recipient:** Approve or approve with override → C1: Run complete; request revision → T4: Aggregate Predicted Headcount (up to 2 times); reject, deadline missed, or third revision request → Stop: Run incomplete, with a missed deadline also sent to the CPVC club president.
- **Complete when:** The decision is saved and read back with the organizer's name, timestamp, and forecast version, or the deadline has passed and the run is recorded as incomplete.

## 4. Planned Tools

### Tool 1

- **Tool name:** `record_forecast_decision`
- **Input:** Forecast review packet; the organizer's selected action
- **Output:** Forecast decision
- **Implementation Route:** Database queries (the review page saves the decision keyed on run ID and forecast version, then reads it back)
- **Integration approach:** Direct integration
- **Role in this task:** Records the organizer's decision only. It cannot approve, override, or revise a forecast on its own.
- **Task timeout:** Human response deadline: 24 hours after the forecast appears on the review page. Each save may take at most 5 seconds.
- **Maximum retries:** Not applicable — manual task.
- **Retry only when:** Not applicable.
- **On timeout, exhausted retries, or an error that cannot be retried:** If the deadline passes, record "not approved: deadline missed," end the run as incomplete, and notify the CPVC club president. If a save fails, the review page shows the error and the decision does not count until the organizer saves it again and it is read back. The forecast is never treated as approved without a saved decision.

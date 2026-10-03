# Present Forecast to Organizers Task Specification

## Basic Information

- **Task ID:** T6
- **Task name:** Present Forecast to Organizers
- **Task type:** Verify
- **Automation level:** L1 — fixed save, read-back, and notification steps; no model is used.
- **Task owner:** HackTrack workflow controller; the CPVC event lead organizer receives the forecast.

## 1. Task Description

Save the Headcount forecast as a numbered version and read it back to confirm that the saved numbers match. Then show it on the HackTrack review page with its evidence: expected headcount, low and high estimates, counts by signal group and confidence, uncertain and unscored registrants (by registrant ID and reason only), prompt send results, the past-event table version, and any labeled organizer assumption. Finally, send the event lead one notification with a link to the review page. The notification includes totals only, never registrant identities.

The workflow needs this task because a human must see a verified, explained forecast before it guides any purchase.

## 2. Inputs

### Input 1

- **Input name:** Headcount forecast
- **Contents and format:** Run ID, checkpoint, forecast version, expected headcount, low and high estimates, counts, and any labeled organizer assumption.
- **Source:** T4: Aggregate Predicted Headcount

- **If a required input is missing or invalid:** If the forecast is missing, has no version, or its counts do not add up, do not present it. Send the case to H1: Notify Organizer of Run Issue.

## 3. Outputs

### Output 1

- **Output name:** Forecast review packet
- **Contents and format:** The saved forecast version (confirmed by read-back), review page link, notification status (sent, failed, or unknown), and the time the forecast appeared on the review page. That time starts T7's review deadline.
- **Next task or recipient:** T7: Review and Finalize Forecast
- **Complete when:** The saved version matches the forecast, it is visible on the review page, and the notification is sent.

## 4. Planned Tools

### Tool 1

- **Tool name:** `save_forecast_version`
- **Input:** Headcount forecast
- **Output:** Forecast review packet
- **Implementation Route:** Database queries (upsert keyed on run ID and forecast version, followed by read-back)
- **Integration approach:** Direct integration
- **Role in this task:** Saves the forecast and confirms that the stored numbers match.
- **Task timeout:** 30 seconds for the whole T6 run, including all tools and retries; each save or read-back may take at most 5 seconds.
- **Maximum retries:** 1
- **Retry only when:** The write fails with a temporary error and a read-back on the same key shows no saved version. Wait 2 seconds before retrying. The upsert key prevents a duplicate forecast version.
- **On timeout, exhausted retries, or an error that cannot be retried:** Do not present an unverified forecast. Record the failure and send the case to H1: Notify Organizer of Run Issue.

### Tool 2

- **Tool name:** `notify_event_lead`
- **Input:** Forecast review packet
- **Output:** Forecast review packet (notification status)
- **Implementation Route:** Web API calls (CPVC club email service send API)
- **Integration approach:** Direct integration
- **Role in this task:** Sends one notification per run ID and forecast version with the review page link.
- **Task timeout:** Within the 30-second T6 limit; the send call may take at most 10 seconds.
- **Maximum retries:** 1
- **Retry only when:** The email service explicitly rejects the request before accepting it and the notification log shows no accepted send for this run ID and forecast version. Wait 2 seconds. If the request times out or the response is unclear, do not retry.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record the notification status as failed or unknown. The forecast stays on the review page, and the case goes to H1, which alerts organizers through the organizer group channel so the event lead still learns about it. Do not resend an unknown notification.

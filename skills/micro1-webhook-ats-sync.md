---
name: Keep an ATS in sync with micro1 webhooks
description: >-
  Register micro1 webhook subscriptions, receive the platform's ten event types, and pull the report
  or recording each event points at so an applicant tracking system stays current without polling.
api: openapi/micro1-ai-recruiter-openapi.yml
base_url: https://public.api.micro1.ai
operations:
  - webhookSetup
  - webhookSetup-51ac71f1-e1da-425d-afa2-d527e716b7e6
  - webhookSetup-c050afb6-e424-4a49-81a7-839386c17724
  - webhookSetup-223f3d34-faeb-4c70-91df-a08a2baa2141
  - aiInterviewer-c5c30f81-a2b4-45ca-9943-f034683993b0
  - aiInterviewer-224f9971-cd82-4432-9f9f-8e4f1ba73345
method: generated
generated: '2026-07-31'
---

# Keep an ATS in sync with micro1 webhooks

## Model

**One subscription per event.** `CreateWebhookRequest` takes a single `event` string, not an array —
so subscribing to five event types means five `POST /webhook` calls. Every payload has the same
envelope: `{"event": "<type>", "data": { ... }}`.

The full event catalog with sample payloads is in `asyncapi/micro1-webhooks.yml`.

## Step 1 — Register a subscription

`POST /webhook` — operationId **`webhookSetup`**

Required: `url` (HTTPS, publicly reachable) and `event` (one of the enum below). Optional
`description`.

Valid `event` values, taken verbatim from the spec enum:

`interview_report.created`, `interview_recording.completed`,
`interview_webcam_recording.completed`, `proctoring_score.completed`, `applicant.completed`,
`applicant.created`, `applicant.updated`, `applicant.invited`, `interview.started`,
`interview.completed`.

For a typical ATS sync, register at minimum `interview_report.created` and `applicant.completed`.

## Step 2 — Verify what is registered

`GET /webhooks` — operationId **`webhookSetup-51ac71f1-e1da-425d-afa2-d527e716b7e6`**

Do this before every registration. There is no idempotency key on `POST /webhook`, so a retried
create leaves a duplicate subscription and you will process each event twice.

## Step 3 — Handle the delivery

Respond within 30 seconds. Treat delivery as **at-least-once**: micro1 documents no signature
header, no delivery id, and no documented retry/backoff contract, so dedupe on the payload's own
identifiers — `report_id` for report/recording/proctoring events,
`micro1_job_application_id` + `stage` for applicant events, `interview_id` + `candidate_id` for
`interview.started`.

Note `interview.completed` is documented as the same trigger as
`interview_recording.completed` — subscribe to one, not both.

## Step 4 — Pull the detail the event points at

- `interview_report.created` → the payload already carries the full evaluation. If you need to
  re-read it later, `GET /interview/reports` — operationId
  **`aiInterviewer-c5c30f81-a2b4-45ca-9943-f034683993b0`** — filtered by `report_id`.
- `interview_recording.completed` / `interview_webcam_recording.completed` → `GET /interview/recording`
  — operationId **`aiInterviewer-224f9971-cd82-4432-9f9f-8e4f1ba73345`** — with `report_id`.
  Recording URLs expire; fetch and re-host on receipt rather than storing the URL.
- `proctoring_score.completed` → carries `proctoring_score` (0–100) only. Treat proctoring
  violations as advisory signal, never as an automatic rejection.

## Step 5 — Maintain the subscriptions

`PUT /webhook/{webhookId}` — operationId **`webhookSetup-c050afb6-e424-4a49-81a7-839386c17724`** —
to repoint a URL. `DELETE /webhook/{webhookId}` — operationId
**`webhookSetup-223f3d34-faeb-4c70-91df-a08a2baa2141`** — to remove one. Delete before rotating an
endpoint host so you never have two live subscriptions for the same event.

## ATS identifier mapping

`ats_job_id` and `ats_job_application_id` are echoed straight back on every applicant and interview
event. Set them when you create the applicant and they become the join key between micro1 and your
ATS — a mismatch silently breaks downstream sync rather than erroring.

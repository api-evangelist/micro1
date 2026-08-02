---
name: Run a job applicant pipeline
description: >-
  List micro1 jobs, push applicants (with a resume URL) into a job, read back applicant detail and
  resume score, and move a qualified applicant into the job's AI interview.
api: openapi/micro1-ai-recruiter-openapi.yml
base_url: https://public.api.micro1.ai
operations:
  - jobsApplicant
  - jobsApplicant-c1ce8fbf-390b-412b-9895-d422b21e823e
  - jobsApplicant-8934b0b6-90a5-42a4-a5e6-c37d3833718e
  - jobsApplicant-e3619770-a47f-4349-838d-a612736b5621
  - aiInterviewer-08720534-ee71-4a6c-aff1-25c48ed78c6d
method: generated
generated: '2026-07-31'
---

# Run a micro1 job applicant pipeline

## Step 1 — Find the job

`GET /jobs` — operationId **`jobsApplicant`**

Returns `data[]` of Job objects: `job_id`, `job_title`, `job_description`, `job_code`, `ats_job_id`,
`interview_id`, `job_apply_url`, `job_status` (`open` | `closed`). Paginate with `page` / `limit`;
narrow with `keyword`.

Two fields matter downstream: `interview_id` is the AI interview already attached to this job (use
it in step 4 rather than creating a new one), and `ats_job_id` is your own ATS's job identifier
echoed back — use it to match jobs instead of matching on `job_title`.

Skip `closed` jobs; adding applicants to a closed job is not a documented flow.

## Step 2 — Create the applicant

`POST /job/{jobId}/applicant` — operationId
**`jobsApplicant-c1ce8fbf-390b-412b-9895-d422b21e823e`**

Path parameter `jobId` is the `job_id` from step 1. Required body: `first_name`, `last_name`,
`email_id`. Optional: `phone_number`, `resume_url` (a publicly fetchable PDF or DOCX).

Supplying `resume_url` is what triggers resume scoring. micro1 emits `applicant.created` on
creation and `applicant.completed` once `resume_score` has been generated — the scoring is
asynchronous, so do not expect a score on the create response. Subscribe to `applicant.completed`
(see the `micro1-webhook-ats-sync` skill).

There is **no idempotency key**. Read back with step 3 and match on `email_id` before creating, or
you will create duplicate applicants for the same person.

## Step 3 — Read back applicants

`GET /job/{jobId}/applicants` — operationId
**`jobsApplicant-8934b0b6-90a5-42a4-a5e6-c37d3833718e`** — lists every applicant on the job, with
`page` / `limit` / `keyword`.

`GET /job/applicant/{jobApplicantId}` — operationId
**`jobsApplicant-e3619770-a47f-4349-838d-a612736b5621`** — returns one applicant's full detail
including `resume_score`, `stage`, `candidate_id` and the ATS identifiers.

## Step 4 — Move a qualified applicant into the interview

`POST /interview/invite` — operationId **`aiInterviewer-08720534-ee71-4a6c-aff1-25c48ed78c6d`**

Use the job's `interview_id` from step 1 and pass `job_application_id` so the interview session is
bound back to the application rather than floating free. Set `candidate_id` when you are re-sending
an invite to somebody who already has a session — omitting it creates a **second** session for the
same person.

`disable_email_notification: true` lets you deliver the interview link through your own ATS email
templates instead of micro1's.

micro1 emits `applicant.invited` on success, then `interview.started` when the candidate begins.

## Gate on the right signal

`resume_score` is a screening signal, not a decision. The interview report's `ai_match_score`,
per-skill ratings and soft-skill CEFR band (retrieved in the `micro1-screen-candidates` skill) are
the evaluation output; `proctoring_score` is advisory integrity signal and should never be used as
an automatic reject on its own.

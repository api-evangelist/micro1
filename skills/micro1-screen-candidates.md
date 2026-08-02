---
name: Screen candidates with an AI interview
description: >-
  Create a skills-based AI interview on micro1, preview the questions it will ask, invite candidates
  by email, then poll for the completed report with per-skill ratings, soft-skill scores and a
  proctoring integrity score.
api: openapi/micro1-ai-recruiter-openapi.yml
base_url: https://public.api.micro1.ai
operations:
  - aiInterviewer
  - aiInterviewer-2bc90237-ae08-40ae-8d61-08907a167e52
  - aiInterviewer-08720534-ee71-4a6c-aff1-25c48ed78c6d
  - aiInterviewer-f690665f-6248-4c86-b317-0f18ebecf728
  - aiInterviewer-c5c30f81-a2b4-45ca-9943-f034683993b0
method: generated
generated: '2026-07-31'
---

# Screen candidates with a micro1 AI interview

## Before you start

- Every request carries `x-api-key: <key>` in the header. Keys come from the micro1 dashboard.
  There is no OAuth surface — see `authentication/micro1-authentication.yml`.
- There is **no idempotency key**. `POST /interview` and `POST /interview/invite` are not safe to
  retry blindly: a repeated invite for the same candidate creates a second interview session.
  Read-back before you retry (step 3).
- Errors come back as `{"status": false, "message": "..."}` with HTTP 400 / 500 (and 401/403 on
  auth). See `errors/micro1-problem-types.yml`. There is no error `code` field to branch on — you
  are matching on HTTP status plus the human-readable `message`.

## Step 1 — Preview the questions (optional, recommended)

`POST /mock/interview` — operationId **`aiInterviewer-2bc90237-ae08-40ae-8d61-08907a167e52`**

Send the skill list you intend to test and read back sample questions before you commit to an
interview. This is a read-shaped preview: it creates nothing.

## Step 2 — Create the interview

`POST /interview` — operationId **`aiInterviewer`**

Required: `interview_name`. Optional: `skills[]` (max 10, each `{name, description}`),
`custom_question_list[]` (max 20, each `{question, time (1–4 min), type: audio|text}`),
`interview_language`.

If you want an interview built **only** from your own questions, use
`POST /custom/interview` — operationId **`aiInterviewer-ce2743c0-2309-41eb-8ff2-44c55c138c25`**.

The response returns `data.interview_id` and `data.invite_url`. **Persist `interview_id`** — every
later call keys off it. You can hand `invite_url` straight to a candidate or post it on a job ad
instead of using step 3.

## Step 3 — Invite candidates

`POST /interview/invite` — operationId **`aiInterviewer-08720534-ee71-4a6c-aff1-25c48ed78c6d`**

Required: `interview_id`. Send `candidates[]` as `{name, email}`. Optional `job_application_id`,
`candidate_id` (to re-send an invite to an existing candidate rather than create a new session), and
`disable_email_notification` when you want to deliver the link yourself.

Before inviting, read back with `GET /interview/invites` — operationId
**`aiInterviewer-f690665f-6248-4c86-b317-0f18ebecf728`** — and search on the candidate's email. This
is the only guard against duplicate sessions, because there is no idempotency contract.

## Step 4 — Collect the report

`GET /interview/reports` — operationId **`aiInterviewer-c5c30f81-a2b4-45ca-9943-f034683993b0`**

Filter with `interview_id`, `candidate_id` or `report_id`. Paginate with `page` and `limit`; free-text
filter with `keyword` (see `conventions/micro1-conventions.yml`).

The report carries `ai_match_score`, `proctoring_score` and `proctoring_violations[]`,
`technical_skills_evaluation[]` (per skill: rating Junior/Mid/Senior + feedback),
`soft_skills_evaluation[]` (CEFR-style A1–C2 with vocabulary / grammar / critical-thinking
sub-scores), `coding_skills_evaluation`, `custom_question_evaluation[]`, the full
`interview_transcript[]`, and `report_url` / `interview_recording_url`.

**Do not poll immediately.** micro1 documents a 2–5 minute generation delay after the candidate
finishes. Prefer the `interview_report.created` webhook — see the
`micro1-webhook-ats-sync` skill and `asyncapi/micro1-webhooks.yml`.

## Step 5 — Retrieve the recording

`GET /interview/recording` — operationId **`aiInterviewer-224f9971-cd82-4432-9f9f-8e4f1ba73345`**

Requires `report_id`. Recording URLs are time-limited; download or re-host promptly.

## Housekeeping

`GET /interviews` (**`aiInterviewer-34586a03-d777-4757-8c54-f8d7e4c821a5`**) lists saved interviews,
`PUT /interview/{interviewId}` (**`aiInterviewer-6d5d608a-0b31-460d-80b8-a69aa43aac17`**) updates one,
`DELETE /interview/{interviewId}` (**`aiInterviewer-b7d10a6a-f334-4836-933f-c9a62d33d80b`**) removes
one. A deleted interview cannot be re-invited — create a new one and keep the old for the record.

---
name: Micro1
description: Use when building recruitment automation workflows, creating AI-powered interviews, managing candidate pipelines, inviting candidates to interviews, retrieving interview reports and recordings, or integrating recruitment processes with external ATS systems via webhooks.
metadata:
    mintlify-proj: micro1
    version: "1.0"
---

# micro1 AI Recruiter API Skill

## Product Summary

The micro1 AI Recruiter API enables you to generate custom conversational interviews based on job requirements, invite candidates, and receive detailed AI-evaluated reports on candidate performance. The API handles interview creation with skill-based questions, candidate invitations, interview execution with optional coding exercises, and comprehensive reporting including AI ratings per skill, soft skills evaluation, proctoring scores, and interview transcripts.

**Base URL:** `https://public.api.micro1.ai`

**Authentication:** All requests require the `x-api-key` header with your API key from the micro1 dashboard.

**Key Endpoints:**
- `POST /interview` — Create a new interview
- `POST /interview/invite` — Invite candidates to an interview
- `GET /interviews` — List all saved interviews
- `GET /interview/reports` — Retrieve completed interview reports
- `POST /webhook` — Register webhooks for real-time events
- `GET /webhooks` — List all registered webhooks

**Primary Documentation:** https://ai-recruiter.micro1.ai

## When to Use

Reach for this skill when:

- **Building recruitment automation:** Integrate interview creation and candidate management into your ATS or recruitment platform
- **Scaling candidate screening:** Create standardized interviews for multiple candidates without manual scheduling
- **Evaluating technical skills:** Generate interviews that test specific technical skills (React, FastAPI, etc.) with AI-powered scoring
- **Tracking candidate progress:** Monitor interview status, retrieve reports, and access recordings via API
- **Automating workflows:** Use webhooks to trigger downstream actions when interviews start, complete, or reports are generated
- **Integrating with external systems:** Map micro1 candidates and jobs to your ATS using applicant IDs and job IDs
- **Assessing soft skills:** Retrieve AI evaluations of communication, critical thinking, grammar, and vocabulary alongside technical assessments

## Quick Reference

### Authentication Header
```bash
x-api-key: YOUR_API_KEY
```

### Core API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/interview` | POST | Create interview with skills and job details |
| `/interview/{interviewId}` | PUT | Update interview configuration |
| `/interview/{interviewId}` | DELETE | Remove an interview |
| `/interviews` | GET | List all saved interviews |
| `/interview/invite` | POST | Send interview link to candidate(s) |
| `/interview/invites` | GET | List all candidate invitations |
| `/interview/reports` | GET | Retrieve completed interview reports |
| `/webhook` | POST | Register webhook for events |
| `/webhooks` | GET | List all webhooks |
| `/webhook/{webhookId}` | PUT | Update webhook configuration |
| `/webhook/{webhookId}` | DELETE | Remove webhook |

### Webhook Events to Subscribe To

| Event | Trigger | Use Case |
|-------|---------|----------|
| `interview.started` | Candidate begins interview | Track engagement, log start time |
| `interview_report.created` | Report generated after completion | Retrieve full evaluation results |
| `interview_recording.completed` | Recording processed and available | Access interview video |
| `interview_webcam_recording.completed` | Webcam recording ready | Review candidate video |
| `proctoring_score.completed` | Proctoring analysis finished | Check for cheating indicators |
| `applicant.created` | New applicant added | Trigger resume screening |
| `applicant.invited` | Candidate invited to interview | Log invitation event |
| `applicant.updated` | Applicant data changed | Sync with ATS |
| `applicant.completed` | Resume score generated | Filter candidates by resume match |

### Report Data Structure

Interview reports include:

- **Candidate info:** name, email, ID
- **Technical skills:** per-skill AI ratings (Junior/Mid/Senior), feedback, timestamp
- **Soft skills:** overall rating (A1-C2), vocabulary, grammar, critical thinking scores
- **Coding evaluation:** rating and feedback if coding exercise included
- **Proctoring:** integrity score (0-100), violation list (tab switches, etc.)
- **Custom questions:** answers with AI evaluation and rating
- **Recordings:** URLs to interview and webcam recordings
- **Transcript:** timestamped conversation between candidate and AI interviewer

## Decision Guidance

### When to Use Webhooks vs. Polling

| Scenario | Use Webhooks | Use Polling |
|----------|--------------|-------------|
| Real-time notifications needed | ✓ | |
| Immediate downstream actions required | ✓ | |
| Batch processing acceptable | | ✓ |
| High-frequency checks (every minute) | | ✓ |
| Low-latency critical path | ✓ | |
| Simple integration, no server setup | | ✓ |

**Recommendation:** Use webhooks for production workflows. Register for `interview_report.created` and `applicant.completed` to trigger downstream actions automatically.

### When to Create vs. Update Interviews

| Situation | Action |
|-----------|--------|
| New job opening, new skill set | Create new interview (POST /interview) |
| Adjust existing interview questions/skills | Update interview (PUT /interview/{id}) |
| Reuse interview for multiple candidates | Keep same interview, invite multiple candidates |
| Change job requirements mid-cycle | Create new interview, keep old for historical records |

## Workflow

### 1. Create an Interview

**Step 1:** Prepare interview parameters
- Job title and description
- Required skills to test (e.g., "React.js", "FastAPI")
- Interview duration
- Optional: custom questions, coding exercise details

**Step 2:** Call POST `/interview` with skill list and job details
```bash
curl -X POST https://public.api.micro1.ai/interview \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "job_title": "Backend Developer",
    "skills": ["FastAPI", "PostgreSQL", "Docker"],
    "duration_minutes": 45
  }'
```

**Step 3:** Store the returned `interview_id` for later use

### 2. Invite Candidates

**Step 1:** Gather candidate details (name, email, optional: resume URL, ATS IDs)

**Step 2:** Call POST `/interview/invite` with candidate list and interview ID
```bash
curl -X POST https://public.api.micro1.ai/interview/invite \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "interview_id": "123e4567-e89b-12d3-a456-426614174000",
    "candidates": [
      {
        "name": "John Doe",
        "email": "john@example.com",
        "ats_job_application_id": "APP123"
      }
    ]
  }'
```

**Step 3:** Candidate receives invitation link and completes interview

### 3. Retrieve Reports

**Step 1:** After interview completion, webhook fires `interview_report.created` (if registered)

**Step 2:** Call GET `/interview/reports` to list all completed reports
```bash
curl -X GET https://public.api.micro1.ai/interview/reports \
  -H "x-api-key: YOUR_API_KEY"
```

**Step 3:** Parse report data:
- Extract `ai_match_score` for overall fit
- Review `technical_skills_evaluation` for skill-specific ratings
- Check `proctoring_score` for test integrity
- Access `report_url` for PDF and `interview_recording_url` for video

### 4. Set Up Webhooks (Optional but Recommended)

**Step 1:** Prepare webhook endpoint (must be HTTPS, publicly accessible)

**Step 2:** Call POST `/webhook` to register
```bash
curl -X POST https://public.api.micro1.ai/webhook \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-domain.com/webhooks/micro1",
    "events": ["interview_report.created", "applicant.completed"]
  }'
```

**Step 3:** Receive webhook payloads at your endpoint with event data

**Step 4:** Parse event type and data, trigger downstream actions (update ATS, send notifications, etc.)

## Common Gotchas

- **Missing API key header:** Always include `x-api-key` in request headers. Requests without it return 401 Unauthorized.
- **Invalid API key:** If key is expired or incorrect, you'll get a 403 Forbidden response. Regenerate from dashboard if needed.
- **Interview not found:** Ensure interview_id is valid UUID format. Deleted interviews cannot be re-invited.
- **Candidate email required:** POST `/interview/invite` requires valid email addresses. Invitations fail silently if email is malformed.
- **Webhook timeout:** Ensure your webhook endpoint responds within 30 seconds. Slow endpoints may cause retries or failures.
- **Duplicate invitations:** Inviting the same candidate to the same interview twice creates separate sessions. Check existing invites first with GET `/interview/invites`.
- **Report not immediately available:** Interview reports take 2-5 minutes to generate after completion. Don't poll immediately; wait for webhook or retry after delay.
- **ATS ID mapping:** If using `ats_job_id` or `ats_job_application_id`, ensure these match your ATS exactly. Mismatches break downstream sync.
- **Recording URLs expire:** Interview recording URLs are time-limited. Cache or download recordings promptly after `interview_recording.completed` webhook fires.
- **Proctoring violations are advisory:** High proctoring violations (tab switches, etc.) don't automatically fail candidates. Use as one signal among many.

## Verification Checklist

Before submitting work with the micro1 API:

- [ ] API key is valid and included in `x-api-key` header on all requests
- [ ] Interview created successfully and interview_id is stored
- [ ] Candidate email addresses are valid and formatted correctly
- [ ] Invitations sent and candidates received email links
- [ ] Webhook endpoint is HTTPS and publicly accessible (if using webhooks)
- [ ] Webhook events registered match your use case (e.g., `interview_report.created`)
- [ ] Reports retrieved and parsed correctly (check for `ai_match_score`, skill ratings, proctoring score)
- [ ] ATS IDs (if used) match your system exactly
- [ ] Error responses handled gracefully (401, 403, 404 cases)
- [ ] Recordings downloaded or cached before URLs expire
- [ ] Soft skills and technical skills evaluations reviewed for accuracy

## Resources

**Comprehensive API Navigation:** https://ai-recruiter.micro1.ai/llms.txt

**Critical Documentation Pages:**
- [API Introduction & Base URL](https://ai-recruiter.micro1.ai/api-reference/getting-started/introduction)
- [Authentication & API Key Setup](https://ai-recruiter.micro1.ai/api-reference/getting-started/authentication)
- [Webhook Events & Payloads](https://ai-recruiter.micro1.ai/api-reference/getting-started/webhooks/introductions)

**Support:** Email support@micro1.ai for questions (6-hour response time guaranteed)

---

> For additional documentation and navigation, see: https://ai-recruiter.micro1.ai/llms.txt
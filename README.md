**Personal Study-Planner — Agentic Workflow**


This n8n workflow that turns a short study request into a fully scheduled study plan on Google Calendar and a Google Sheet, with AI reasoning for every session and a human-in-the-loop approval step that revises the plan if the user declines with a reason.


**Problem → Workflow Mapping**


 student says, in vague terms, what they want to learn and by when. To turn that into something useful, the system has to:

Figure out whether the request is even feasible by the deadline.
Expand a vague request ("React Hooks") into a concrete syllabus of subtopics.
Estimate how many study sessions each subtopic realistically needs at the student's level.
Spread those sessions across the available days in daytime hours, without overlaps.
Get the user to approve the plan before it touches their calendar.
If the user pushes back, understand why and redesign — not just retry.
Persist the final schedule to both Google Calendar and a tracking spreadsheet, with the reasoning attached.

The workflow maps each of those onto its own node (or small group of nodes), with deterministic code handling everything that doesn't need judgment and LLM calls only where judgment is required.

**Architecture**


Form Trigger
    ↓
Requirement Intake  ──── defaults applied if blank
    ↓
AI Prerequisite Generator  → Parse → Prerequisite Form → Parse Answers → Form Complete
    ↓
AI Syllabus Designer (thinking)  ← self-questions, decides scope + sessions/topic
    ↓
Parse & Schedule  ← deterministic: pad to level minimum, spread across days, daytime hours, Asia/Kolkata
    ↓
Check Plausible ──false──→ Exit (implausible)
    │ true
    ↓
Propose Plan & Wait Approval  (Gmail send-and-wait, custom form: Approve / Decline + reason)
    ↓
Check Approved ──false──→ AI Revise Plan ─→ back to Parse & Schedule (loop)
    │ true
    ↓
Expand Parts → Create Calendar Event → Write Task Tracker → Final Output
Two LLM models share the AI chains: Gemini (primary) with Groq wired as a fallback through n8n's needsFallback mechanism.

**AI vs Deterministic Steps**


LLMs handle ambiguity and content; code handles math, time, and structure.
StepTypeWhyRequirement IntakeDeterministicJust normalizes form input and fills defaults.AI Prerequisite GeneratorLLMGenerating relevant prerequisite questions requires judgment about the topic and level.Parse Prerequisites / AnswersDeterministicJSON parsing and shaping.AI Syllabus Designer (thinking)LLMThe core reasoning step: decides syllabus scope, per-topic session counts, feasibility, and records its self-reasoning in a thoughts field.Parse & ScheduleDeterministicPads to the level minimum, flattens the syllabus into sessions, spreads them across days, clamps to daytime, computes ISO times in Asia/Kolkata. Time math should not be left to an LLM.Check Plausible / Check ApprovedDeterministicBranch decisions on explicit fields.AI Revise PlanLLMReads the user's free-text decline reason and the previous plan, then redesigns the syllabus to address it.Expand PartsDeterministicFans the plan into one item per session.Create Calendar Event / Write Task TrackerDeterministic (tool calls)External side effects.

Branching Logic:


There are three different branch plates.

Plausibility branch — AI Syllabus Designer returns plausible: false for absurd targets (e.g., "master quantum mechanics in 5 minutes"). The workflow exits with a reason instead of proposing a doomed plan.
Approval branch — the approval email is a custom form with an Approve / Decline dropdown plus a free-text "Reason (if declining)" field. Approve commits to calendar + sheet. Decline routes to the revision loop.
Revision loop — AI Revise Plan receives the user's reason and the prior plan, returns a revised syllabus, which goes back through Parse & Schedule and a new approval email. The loop continues until the user approves (or stops responding).


Human-in-the-Loop


The Gmail sendAndWait node pauses the workflow until the user responds. The response form has structured fields, so the downstream logic can read them as data, not free text:

data["Approve this plan?"] — "Approve" or "Decline" (drives the branch).
data["Reason (if declining)"] — feeds the revise prompt verbatim, so the AI sees exactly what the user wrote and adapts to it.

This means a decline is not a dead end — it's a signal the planner uses to do a better job on the next iteration.

Structured Outputs


Every AI chain returns strict JSON (no markdown fences), parsed deterministically. The syllabus schema is:
json{
  "plausible": true,
  "reason": "",
  "goal": "...",
  "thoughts": "self-reasoning about scope and timing",
  "syllabus": [
    { "topic": "...", "detail": "...", "sessions": 2, "reason": "why this topic sits where it does" }
  ]
}
The final scheduled output (used by the calendar and sheet) is shaped per session:
requestId, eventId, session, topic, detail, reason, start, end, status
requestId ties all sessions of one plan together; eventId is the Google Calendar event ID; reason carries the AI's per-session rationale into both the calendar event description and the spreadsheet.

Scheduling Rules


The scheduler in Parse & Schedule enforces:

Timezone — all times are computed in Asia/Kolkata and emitted as ISO strings with a +05:30 offset.
Start day — sessions begin tomorrow at 09:00 local; today is left free.
Daytime only — every session sits between 09:00 and 20:00. No 3 a.m. study slots.
Day-spread — sessions are distributed across the entire window from tomorrow to the deadline. If there are more days than sessions, all days get used (one session each, spread out). The window only stacks multiple sessions on a single day when there aren't enough days.
No overlap — within a day, sessions are placed at non-overlapping hourly slots.
Level minimum — a deterministic floor regardless of what the AI returns:

School / High School: 4
Undergraduate: 5
Postgraduate: 6
Expert / Professional: 7


Padding rule — if the AI returns fewer than the minimum, sessions are added round-robin across the existing topics so every topic stays covered.


**Setup**


Credentials required
Servicen8n credential typeGoogle GeminigooglePalmApiGroqgroqApi (fallback model)GmailgmailOAuth2Google CalendargoogleCalendarOAuth2ApiGoogle SheetsgoogleSheetsOAuth2Api
The workflow file references specific credential IDs from the author's instance. Reselect each from the dropdown after importing.
Google Sheet
The Sheet ID and tab name are set in the file. The tab must have this exact header row (row 1) for the columns to populate:
requestId | eventId | session | topic | detail | reason | start | end | status
Form usage
All form fields are optional. Submit blank to run with the built-in sample (React Hooks, Undergraduate, deadline = now + 7 days). Type any field to override. For demoing on a short window, type a near deadline like 2026-06-05 18:30.

**Files**

workflow.json — the n8n workflow.
Demo video — https://www.loom.com/share/1c5b3f75689d4c57b473c446e7fc223b

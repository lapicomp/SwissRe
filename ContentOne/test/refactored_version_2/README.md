# [ES-CMP][V5] Create Tasks from Work Requests

## Overview

V5 is a 3-agent automation that creates Opal tasks from CMP work requests. It ships as **two workflow variants** for testing, sharing the same 3 agents. All mapping data (templates, campaigns, supporting activities) lives in `mappings.txt` — no prompt editing needed to add or update IDs.

---

## Two Workflow Variants

### Variant A — Linear (`workflow_linear.json`)
**Workflow ID:** `create-tsk-from-wrq-v5-linear`
**Trigger:** `cmp_work_request_modified_event_v5_linear`

```
Webhook → Process WR → [Accepted] → Create Task → [has_supporting_activity=true] → Supporting Activity
```

- Create Task always runs after acceptance, regardless of work request type.
- The branch to Supporting Activity happens AFTER Create Task, based on `has_supporting_activity`.

### Variant B — Branched (`workflow_branched.json`)
**Workflow ID:** `create-tsk-from-wrq-v5-branched`
**Trigger:** `cmp_work_request_modified_event_v5_branched`

```
                              ┌─ Event-With-SA ─→ Create Event Task ──→ Supporting Activity
Webhook → Process WR ────────┤
                              └─ Standard ──────→ Create Standard Task → End
```

- Process WR returns a `routing_key` field: `"Event-With-SA"` or `"Standard"`.
- The Event path and Standard path are visually distinct in the Opal canvas.
- Both Create Task steps use the same agent (`es-cmp-v5-create-task`).

---

## Agents (shared by both variants)

| Agent ID | File | Role |
|----------|------|------|
| `es-cmp-v5-process-wr` | `agent1_process_wr.json` | Hard gate on webhook, WR validation, routing |
| `es-cmp-v5-create-task` | `agent2_create_task.json` | Create main task, brief update, email |
| `es-cmp-v5-supporting-activity` | `agent3_supporting_activity.json` | Create supporting activity tasks |

---

## Files

| File | Purpose |
|------|---------|
| `mappings.txt` | **Reference file** — all template, campaign, activity mappings. Upload to all 3 agents in Opal. |
| `agent1_process_wr.json` | Agent 1 definition |
| `agent2_create_task.json` | Agent 2 definition |
| `agent3_supporting_activity.json` | Agent 3 definition |
| `workflow_linear.json` | Variant A workflow |
| `workflow_branched.json` | Variant B workflow |
| `prompt_templates.md` | Human-readable copy of all prompts |

---

## Key Changes from V4

| Area | V4 | V5 |
|------|----|----|
| **Modified fields gate** | "proceed if modified_fields is missing/null" — leaks assignee-only webhooks | Hard gate: ONLY proceeds if modified_fields is a non-empty array AND contains `"status"`. No fallback. |
| **has_supporting_activity** | Computed in Agent 2 by checking if `supporting_activity` string is non-empty (any template) | Computed in Agent 1. True ONLY for `"Event Request"` template with non-empty supporting_activity. |
| **Mappings** | Hardcoded in agent prompts (3 separate mapping objects across 3 agents) | Single `mappings.txt` reference file uploaded to all agents. Edit the file, not the prompts. |
| **Workflow variants** | Single workflow | Two variants (linear and branched) for testing |
| **routing_key** | Not present | New field in Agent 1 output: `"Event-With-SA"` or `"Standard"` — used by branched variant |
| **Campaign fallback** | Hardcoded default string in prompt | `_fallback` key in `mappings.txt` campaign_mapping |

---

## Setup Instructions

### 1. Upload mappings.txt as reference file
In Opal, on each of the three agent configuration pages, upload `mappings.txt` as a reference document (knowledge/context file). The agent will use it to look up mappings at runtime.

### 2. Create or import the agents
Import each `agent*.json` file as an agent in Opal.

### 3. Create or import the workflow
Import either `workflow_linear.json` or `workflow_branched.json` (or both) as a workflow.

### 4. Configure the CMP webhook
In CMP, set up the webhook to fire on `work_request_modified` events:
- **Endpoint:** Your Opal webhook trigger URL
- **Header:** `callback-secret: Csm2vDE6rc`

### 5. Update mappings (when needed)
When a workflow ID, campaign ID, or template ID changes:
1. Edit `mappings.txt`.
2. Re-upload it to the affected agent(s) in Opal as the reference document.
3. No prompt changes needed.

---

## Business Logic

### When does the automation run?
Only when a work request fires `work_request_modified` with `modified_fields` containing `"status"`. Specifically:

| CMP action | modified_fields | Result |
|------------|----------------|--------|
| Assign request (assign step) | `["assignee"]` | Stops at Step 1 — no task created |
| Accept request | `["status"]` | Continues — task creation proceeds |
| Any other change | anything without `"status"` | Stops at Step 1 |

### When is has_supporting_activity true?
Only when ALL of:
1. Template name (normalised) is exactly `"Event Request"` — not Event Info Capture, not any other template
2. The `Supporting Activity` form field is non-null and non-empty after trim

### Task title format
- No channel: `"{title_prefix} {work_request_title}"`
- With channel (Social Organic, Social Paid, Trade Media only): `"{title_prefix} {channel_value} {work_request_title}"`
  - `channel_value` is extracted from the `Social Media Account` field using the `before_colon` rule (e.g. `"LinkedIn: Social Post"` → `"LinkedIn"`)

### Supporting activity tasks
- Input: comma-separated list from `Supporting Activity` field (e.g. `"Emailing, Social Organic, Webpage"`)
- One task created per activity type
- Each task title: `"{activity_type} {event_task_title}"`
- Each task brief updated with work request summary
- Each task linked back to the work request

---

## Test Scenarios

| # | Scenario | Expected result |
|---|----------|----------------|
| 1 | Webhook with `modified_fields=["assignee"]` | Agent 1 stops at Step 1. No CMP API call. No task. |
| 2 | Webhook with `modified_fields=["status"]`, WR status = `"Draft"` | Agent 1 returns Not Accepted at Step 8. No task. |
| 3 | Standard WR (Email Marketing), status=Accepted | Task created. `has_supporting_activity=false`. Agent 3 does NOT run. |
| 4 | Event Request + supporting_activity non-empty, status=Accepted | Task created. `has_supporting_activity=true`. Agent 3 runs. Tasks per activity. |
| 5 | Event Request + supporting_activity empty, status=Accepted | Task created. `has_supporting_activity=false`. Agent 3 does NOT run. |
| 6 | Email Marketing + supporting_activity non-empty, status=Accepted | Task created. `has_supporting_activity=false`. Agent 3 does NOT run. *(Key fix)* |
| 7 | Unknown campaign name | Falls back to `_fallback` campaign in mappings.txt. Task still created. |
| 8 | Unknown template name | Agent 1 returns Not Accepted: `"Template not in mapping: ..."`. |
| 9 | Add new campaign to mappings.txt, reupload, fire webhook | New campaign maps correctly. No prompt edits needed. |

---

## Campaign Mapping

Defined in `mappings.txt` under `campaign_mapping`. The `_fallback` key provides the default campaign when no match is found.

| Campaign Name | Campaign ID |
|--------------|------------|
| [ES CMP] Test Campaign | 69228e01dc2727478b5f93ea |
| [ES-CMP] Test Opal Usecase - Campaign | 6903266dd19827a999a72490 |
| Event: Test Campaign | 68f243baac75a43a53dfb70a |
| Captives & Fronting (2025) | 6967c4318dbb9dc09d43b966 |
| Event: Baden Baden (2025) | 6967c3b18dbb9dc09d4342a6 |
| Event: RVS Monte Carlo (2025) | 692d9822465fe318491a02ef |
| Financial & Structured Solutions (2025) | 6967c3ce8dbb9dc09d438c29 |
| Forward in Life (2025) | 692d9845465fe318491a1a0c |
| IP/PULSE & Network (2025) | 692d985c465fe318491a612c |
| RDS Platform Overall (2025) | 6967c3ee8dbb9dc09d43a0fd |
| RDS Property Exposure (2025) | 6967c4068dbb9dc09d43a95f |
| Reinsurance Recalibrated (2025) | 692d9831465fe318491a062a |
| *(fallback)* | 6903266dd19827a999a72490 |

---

## Version History

| Version | Location | Description |
|---------|----------|------------|
| V3 | `original/` | 4-agent flow. Multiple failures in testing. |
| V4 | `refactored_version_1/` | 3-agent flow. Fixed parameter crash. Still has gate bug and hardcoded mappings. |
| V5 | `refactored_version_2/` | 3-agent flow. Hard gate fix. Reference file mappings. Corrected `has_supporting_activity`. Two workflow variants. |

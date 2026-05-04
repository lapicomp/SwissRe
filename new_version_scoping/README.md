# ES-CMP V6 — Create Tasks from Work Requests

## Overview

V6 replaces V5's three overloaded agents with six focused, single-responsibility agents. Email sending is separated into a dedicated Notify agent with exactly one tool, eliminating the email loop bug. A lightweight Gate agent blocks invalid webhooks before any API call is made.

---

## Flow Diagram

```
CMP Work Request Modified
        │
        ▼
  [Webhook Trigger]
        │  webhook_payload
        ▼
  ┌─────────────┐
  │    GATE     │  No tools. Checks modified_fields contains "status".
  │ es-cmp-v6   │  Extracts work_request_id.
  │   -gate     │
  └──────┬──────┘
         │ proceed == false → STOP (workflow ends)
         │ proceed == true
         ▼
  ┌──────────────────┐
  │  VALIDATE WR     │  get_cmp_resource ×1.
  │  es-cmp-v6       │  Validates status=Accepted, template, assignee.
  │  -validate-wr    │  Extracts all routing fields + routing_key.
  └────────┬─────────┘
           │ Not Accepted → STOP
           │ Accepted
           ▼
  ┌──────────────┐
  │   ROUTER     │  No tools. Reads routing_key, passes all fields through.
  │ es-cmp-v6    │  SA-capable template list editable in prompt.
  │  -router     │
  └──────┬───────┘
         │ ROUTE_BLOCKED → STOP
         │
    ┌────┴─────────────────────────────────┐
    │ ROUTE_STANDARD                        │ ROUTE_WITH_SA
    ▼                                       ▼
  ┌────────────────┐              ┌────────────────┐
  │  CREATE TASK   │              │  CREATE TASK   │  (same agent_id,
  │  es-cmp-v6     │              │  es-cmp-v6     │   different step)
  │  -create-task  │              │  -create-task  │
  └───────┬────────┘              └───────┬────────┘
          │ No email sent                 │ No email sent
          ▼                               ▼
  ┌──────────────┐              ┌────────────────────────┐
  │    NOTIFY    │              │  SUPPORTING ACTIVITY   │
  │  es-cmp-v6   │              │  es-cmp-v6             │
  │   -notify    │              │  -supporting-activity  │
  └──────────────┘              └───────────┬────────────┘
  send_email ×1                             │ No email sent
  return                                    ▼
                                    ┌──────────────┐
                                    │    NOTIFY    │
                                    │  es-cmp-v6   │  (same agent_id,
                                    │   -notify    │   different step)
                                    └──────────────┘
                                    send_email ×1
                                    return
```

---

## Agent Table

| Agent ID | File | Role | Tools | Inference Type |
|----------|------|------|-------|----------------|
| `es-cmp-v6-gate` | agent1_gate.json | Validates webhook payload, extracts work_request_id | *(none)* | simple |
| `es-cmp-v6-validate-wr` | agent2_validate_wr.json | Fetches WR, validates status/template/assignee, extracts all fields | `get_cmp_resource` | simple |
| `es-cmp-v6-router` | agent3_router.json | Reads routing_key from validate output, passes fields through | *(none)* | simple |
| `es-cmp-v6-create-task` | agent4_create_task.json | Creates main CMP task, assigns substep, returns task data | `create_task_from_work_request`, `get_cmp_resource`, `update_task_substep` | simple |
| `es-cmp-v6-supporting-activity` | agent5_supporting_activity.json | Creates one task per SA value, updates briefs | `get_cmp_resource`, `get_form_template_by_id`, `create_task_from_work_request`, `update_task_brief` | simple_with_thinking |
| `es-cmp-v6-notify` | agent6_notify.json | Sends exactly one email notification | `send_email` | simple |

---

## Key Changes from V5

| Change | V5 Behaviour | V6 Behaviour |
|--------|-------------|-------------|
| **Agent count** | 3 agents | 6 agents (single responsibility each) |
| **Email sending** | `send_email` in create-task and SA agents — caused 100+ duplicate emails | `send_email` exclusively in Notify agent, called once then returns |
| **Webhook gate** | Merged into Agent 1 (process-wr) with API call | Separate Gate agent with zero tools — blocks before any API call |
| **Routing** | `has_supporting_activity` computed in Agent 1, branched in Agent 2 | Dedicated Router agent with SA-capable template list in its prompt |
| **`has_supporting_activity`** | Recomputed in Agent 2 — caused non-Event templates to route to SA | Computed once in Validate, passed through Router and Create Task unchanged |
| **Condition matching** | `"contains"` used in some places — "Not Accepted" matched "Accepted" | `"equals"` everywhere |
| **Campaign fallback** | Hardcoded string in prompt | `_fallback` key in mappings.txt Section 2 |
| **"choose" in briefs** | Dropdown fields sometimes set to literal "choose" | SA agent instructed to use semantically matched choice or `"– Choose –"` placeholder |
| **Notify for failures** | Create-task and SA sent failure emails directly | All failure notifications go through Notify agent |

---

## Setup Instructions

### Step 1 — Import agents (do this first)

Import all 6 agent JSON files into Opal in any order:
1. `agent1_gate.json`
2. `agent2_validate_wr.json`
3. `agent3_router.json`
4. `agent4_create_task.json`
5. `agent5_supporting_activity.json`
6. `agent6_notify.json`

### Step 2 — Upload mappings.txt

Upload `mappings.txt` as a reference file to these agents:
- **Validate WR** (uses Sections 1 and 4)
- **Create Task** (uses Section 2)
- **Supporting Activity** (uses Section 3)

After upload, Opal will generate a `gs://` URL. Update the `file_urls[0].url` field in the corresponding agent JSON files if you re-export them.

### Step 3 — Import workflow

Import `workflow.json` into Opal. If you get a 500 error, check:
- Edge IDs in `agent_metadata.edges` follow the correct format
- `agent_metadata` appears before `internal_version` and `steps` in the JSON
- All `agent_id` values in steps match the imported agent IDs exactly

### Step 4 — Configure CMP webhook

In CMP settings (Work Request → Integrations or Admin → Webhooks):
- **URL:** Copy from the V6 workflow trigger in Opal
- **Auth header name:** `callback-secret`
- **Auth header value:** `Csm2vDE6rc`
- **Event:** `work_request_modified`

Point this webhook to V6 only. Disable or update the V5 webhook URL to avoid double-triggering.

### Step 5 — Updating mappings

To add a new campaign, activity type, or update IDs:
1. Edit `mappings.txt` locally
2. Re-upload to the affected agents in Opal (no prompt changes needed)

To add a new **SA-capable template** (like "Event Request"):
1. Edit the SA-capable template list in **Agent 2** prompt (Step 6) and **Agent 3** prompt (Step 2)
2. Add the template row to Section 1 of `mappings.txt` with a `task_template_id`
3. Re-upload `mappings.txt`

---

## Test Scenarios

| # | Scenario | Webhook trigger | Expected result |
|---|---------|-----------------|-----------------|
| 1 | Assignee-only webhook | WR assigned (not accepted) | Gate returns `proceed: false`. Workflow ends at gate. No email. |
| 2 | Status webhook — Draft status | WR modified but status is "Draft" | Gate passes. Validate returns `Not Accepted` ("status is not Accepted"). Workflow ends. No email. |
| 3 | Standard template accepted | "Web Requests (Marketing)" WR accepted | Router returns `ROUTE_STANDARD`. Create Task creates task. Notify sends 1 success email to owner + CC. |
| 4 | Event Request + supporting activities | "Event Request" WR accepted with SA values | Router returns `ROUTE_WITH_SA`. Create Task creates main task. SA agent creates SA tasks + updates briefs. Notify sends 1 email. |
| 5 | Event Request + no supporting activity | "Event Request" WR accepted, no SA field | Validate computes `routing_key = ROUTE_STANDARD` (supporting_activity empty). Router returns `ROUTE_STANDARD`. Standard path. |
| 6 | Unknown campaign | WR accepted, campaign not in mappings | Create Task uses `_fallback` campaign ID. Task created in fallback campaign. Notify sends success email. |
| 7 | Unknown template | WR accepted, template not in Section 1 | Validate returns `Not Accepted` ("Template not in mapping"). Workflow ends. No email. |
| 8 | No assignee | WR accepted with no assignee | Validate sets `accepted_by_id = null`. Create Task omits `owner_id`. Notify sends email to CC list only (no recipient). |

---

## File Reference

| File | Purpose |
|------|---------|
| `workflow.json` | Import into Opal to create the workflow |
| `agent1_gate.json` – `agent6_notify.json` | Import into Opal as individual agents |
| `mappings.txt` | Upload to Validate WR, Create Task, and SA agents in Opal |
| `mappings.json` | Machine-readable reference (not uploaded to Opal) |
| `prompt_templates.md` | Human-readable prompt reference; copy back to JSON after edits |

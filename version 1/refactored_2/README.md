# Refactored_2: Stable CMP Work Request → Task Workflow

Stable, consistent version of the Create Tasks from Specific Work Requests workflow. Same behaviour as the refactored flow, with consistent agent IDs, case-insensitive campaign lookup, and equals-only routing.

## Agent IDs

| Step | Agent ID |
|------|----------|
| 1. Extract Webhook Event Info | `es-cmp-r2-extract-webhook` |
| 2. Process Work Request Details | `es-cmp-r2-process-wr` |
| 3. Create and Update Task from Work Request | `es-cmp-r2-create-task` |
| 4. Create and Update Tasks for Supporting Activity | `es-cmp-r2-supporting-activity` |

## Workflow

- **Workflow agent_id:** `create-tsk-from-wrq-r2`
- **Trigger name:** `cmp_work_request_modified_event_r2`
- **Auth:** Header `callback-secret` with value `Csm2vDE6rc` (same as refactored).

## CMP webhook setup

- Subscribe to **`work_request_modified`** in CMP.
- Point the webhook URL to the refactored_2 workflow’s endpoint in Opal (after import).
- Task creation runs **only when** a work request has been **accepted** (status = Accepted) and has an assignee. Assigning someone without accepting does not create tasks; the workflow stops at Process WR (Agent 2).
- If your CMP supports filtering (e.g. only send when `modified_fields` includes `"status"`), use it to reduce unnecessary runs and token use.

## Files

- `workflow.json` — workflow definition (import into Opal).
- `agent1_extract_webhook.json` … `agent4_supporting_activity.json` — specialized agents (import and ensure IDs match).
- `prompt_templates.md` — copy-paste prompt text for all four agents.
- `README.md` — this file.

## Routing (Opal)

- **Extract → Process WR:** condition `event_type` **equals** `"status"` (first element of `modified_fields`).
- **Process WR → Create Task:** condition **proceed_status** (field) **equals** `"Accepted"`. Ensure the platform compares the `proceed_status` field value, not the full output string.
- **Create Task → Supporting Activity:** condition matches when Create Task returns `has_supporting_activity: true` (e.g. `matching_condition`: `"true"` if the platform evaluates against that field).

# Version 2: Simplified CMP Work Request → Task Workflow (V4)

V4 is a 3-agent flow. The separate Extract Webhook step has been eliminated. The `modified_fields` check and `request_id` extraction are now a workflow_gate inside Agent 1 (Process Work Request Details). This reduces the workflow from 5 pre-task steps to 3.

## Flow

```
Trigger → Process WR → Condition("Accepted") → Create Task → Condition("true") → Supporting Activity
```

## Agent IDs

| File | Agent ID | Role |
|------|----------|------|
| `agent1_process_wr.json` | `es-cmp-v4-process-wr` | Checks modified_fields, extracts request_id, fetches & validates WR |
| `agent2_create_task.json` | `es-cmp-v4-create-task` | Creates main task, sends email, returns has_supporting_activity flag |
| `agent3_supporting_activity.json` | `es-cmp-v4-supporting-activity` | Creates one task per supporting-activity value, links to WR, sends email |

## Workflow

- **Workflow agent_id:** `create-tsk-from-wrq-v4`
- **Trigger name:** `cmp_work_request_modified_event_v4`
- **Auth:** Header `callback-secret` with value `Csm2vDE6rc`

## Known issue: "substring not found" / parameter prediction failure

Opal uses AI-based **parameter prediction** to map trigger output fields to agent input parameters when `parameters_schema` is `null`. This is unreliable — it fails with `substring not found` on certain payloads (e.g. when `modified_fields = ["assignees"]`), crashing the workflow before any agent runs.

**Fix applied:** The Process WR step now has an explicit `parameters_schema` mapping `webhook_payload → {{webhook_payload}}`, so Opal maps it deterministically without prediction. Agent 1's Step 1 guard (`if "status" not in modified_fields → return Not Accepted`) then handles the early exit cleanly, completing the run instead of failing it.

## Routing (Opal)

- **Trigger → Process WR:** Direct connection — no condition step.
- **Process WR → Create Task:** condition `proceed_status` **equals** `"Accepted"`. Stops when `Not Accepted` (e.g. status not modified, missing id, wrong template, not yet Accepted, no assignee).
- **Create Task → Supporting Activity:** condition `has_supporting_activity` **equals** `"true"`. Only runs when supporting activity values are present.

## Files

- `workflow.json` — import into Opal
- `agent1_process_wr.json` — import and register (agent_id: `es-cmp-v4-process-wr`)
- `agent2_create_task.json` — import and register (agent_id: `es-cmp-v4-create-task`)
- `agent3_supporting_activity.json` — import and register (agent_id: `es-cmp-v4-supporting-activity`)
- `prompt_templates.md` — copy-paste prompts for all 3 agents

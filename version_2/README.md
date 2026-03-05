# Version 2 – Create Tasks from Specific Work Requests

V2 workflow and agents for SwissRe CMP task creation. Uses **equals** conditions for routing and applies guardrails from the plan (send_email once, case-insensitive campaign lookup, null handling, assignee required).

## Contents

- `workflow.json` – Opal workflow; trigger `cmp_work_request_modified_event_v2`; steps use match_type **equals**.
- `agent1_extract_webhook.json` – Extract event_type, request_id.
- `agent2_process_wr.json` – Process work request; proceed_status Accepted only when status Accepted and assignee set.
- `agent3_create_task.json` – Create task; case-insensitive campaign; has_supporting_activity for routing.
- `agent4_supporting_activity.json` – Supporting activity tasks; task_urls in schema.
- `prompt_templates.md` – Prompt summaries.

## Deployment

1. Import or create the four specialized agents in Opal (agent_id suffix `-v2`).
2. Import the workflow; ensure conditions use **equals** and, if the platform supports it, that the Create Task → Supporting Activity condition compares the **has_supporting_activity** field to true.
3. In CMP, point the webhook to this workflow’s URL and subscribe to `work_request_modified`. Use auth header `callback-secret` with the workflow’s auth_key value.

## References

- Plan: see project plan "SwissRe V2 Workflow and Rules".
- Analysis: `analysis/behavior_contract.md`, `analysis/guardrails_and_prompt_conventions.md`, `analysis/cmp_webhook_setup.md`.

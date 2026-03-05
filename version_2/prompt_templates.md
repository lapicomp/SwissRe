# V2 Prompt template summaries

Short reference for the V2 agent prompts. Full prompts are in the agent JSON files.

## 1. Extract Webhook Event Info (agent1)

- **Goal:** Output event_type and request_id for routing. event_type = first element of modified_fields (routing flag, not lifecycle status).
- **Guardrails:** Output only the JSON; request_id non-empty string.
- **Output:** `{ "event_type": "<trimmed first element>", "request_id": "<id>" }`

## 2. Process Work Request Details (agent2)

- **Goal:** Fetch work request, validate template/status, return structured object. Only "Accepted" or "Not Accepted".
- **Guardrails:** get_cmp_resource once; "Not Accepted" only (no "Declined"); proceed_status = "Accepted" only when work_request_status === "Accepted" and assignee present; normalise template name (trim, case); null-safe form_fields.
- **Early returns:** Template not in mapping → Not Accepted + reason. Status not Accepted / workflow null / no assignee → Not Accepted + reason.
- **Output:** proceed_status, work_request_id, template_name, workflow_id, workflow_name, status, assignee_name, assignee_id, campaign, supporting_activity, due_date, channel, task_title.

## 3. Create and Update Task from Work Request (agent3)

- **Goal:** Create task, set owner from assignee, send_email exactly once then return.
- **Guardrails:** Case-insensitive campaign lookup; no create if campaign not in mapping (failure email CC only); each tool at most once per path; send_email once then return; null checks for task_id and owner_email_address; pass supporting_activity (same key); set has_supporting_activity true only when supporting_activity non-null and non-empty.
- **Output:** execution_status, task_id, campaign_id, cmp_task_url, and when supporting: task_title, supporting_activity, work_request_id, owner_email_address, due_date, has_supporting_activity: true. Else has_supporting_activity: false.

## 4. Create and Update Tasks for Supporting Activity (agent4)

- **Goal:** One task per supporting activity value, link to work request, one summary email.
- **Guardrails:** get_cmp_resource once; work_request_fields_summary empty string if form_fields null; normalise values (split, trim, case); skip unmapped; empty workflow_id/template_id handled; send_email once; empty list → return task_urls [] without email.
- **Output:** execution_status "Tasks created successfully" or "Failed", task_urls array.

# Changelog: Refactored workflow and agents (Use Case 1)

Summary of changes per file. All refactored artifacts are under `refactored/`. `original/` is read-only.

---

## refactored/workflow.json

- **Conditions:** Step 1 (Extract → Process WR): `matching_condition` "status", `match_type` "equals" (was "contains") so we proceed only when event_type equals "status". Step 2 (Process WR → Create Task): "Accepted" with match_type "equals". Step 3 (Create Task → Supporting Activity): "true" with match_type "equals" so we proceed when has_supporting_activity is true.
- **Risks addressed:** R1 (deterministic first branch), R12 (no match = workflow stops by design; document in runbook).

---

## refactored/agent1_extract_webhook.json

- **Changelog in description:** Refactor note that workflow branches on event_type equals status.
- **Prompt:** Clear instruction that event_type = first element of webhook_payload.data.work_request.modified_fields (trim/normalise); when status changed this is "status". Request ID from webhook_payload.data.work_request.id.
- **Risks addressed:** R1.

---

## refactored/og_agent2_process_wr.json

- **Enum/prompt:** Use only "Not Accepted" (removed "Declined").
- **Explicit validation:** proceed_status = "Accepted" only when work_request_status === "Accepted", workflow_id non-null, template in mapping, and assignee_id or assignee_name non-empty. Otherwise "Not Accepted" with reason.
- **Assignee required:** When Accepted but no assignee, return Not Accepted with reason "Assignee required for task creation."
- **Template:** Normalise template name (trim, consistent case) before lookup; if not in mapping, return Not Accepted with reason "Template not in mapping".
- **form_fields:** Null/missing-key handling; safe access for all form_fields lookups.
- **Schema:** Added optional "reason" (string | null).
- **Call once:** Prompt states call get_cmp_resource once; derive all outputs from that response.
- **Risks addressed:** R2, R3, R4.

---

## refactored/agent3_create_task.json

- **Pass-through key:** Use [[work_request_object]["supporting_activity"] consistently (not "Supporting Activity") so Agent 4 receives the value.
- **Campaign not in mapping:** If campaign not in campaign_mapping_object, do not call create; send failure email once; return Failed with reason "Campaign not in campaign_mapping_object: <name>".
- **Null checks:** If task_id null after create, take failure path (send failure email once, return Failed); do not call get_cmp_resource, update_task_substep, or send_email with null. If owner_email_address null after get_cmp_resource, send failure email (e.g. to CC), return Failed.
- **Schema:** Added supporting_activity (string | null), has_supporting_activity (boolean). Output sets has_supporting_activity true only when supporting_activity is non-null and non-empty after trim.
- **Send_email:** Call send_email exactly once per path (success or failure); after calling, do not call again; proceed to return JSON. Strict step order (Step 5 → Step 6; do not return to Step 5).
- **Risks addressed:** R5, R6, R7; send_email loop prevention.

---

## refactored/agent4_supporting_activity.json

- **Normalise:** Trim and consistent case for comma-split values before lookup; skip unmapped values (do not fail whole step).
- **Empty workflow_id/template_id:** Skip workflow_id or update_task_brief when empty; document in prompt.
- **form_fields:** If form_fields null or missing, work_request_fields_summary = "".
- **Schema:** Added task_urls (array of strings).
- **Send_email:** Call send_email exactly once (success or failure); after calling, do not call again; strict step order (Step 4 → Step 5).
- **Risks addressed:** R8, R10, R11; send_email loop prevention.

---

## analysis/behavior_contract.md

- Optional: one sentence that "no match" on any conditional step means workflow ends by design (no error). Document trigger/event_type semantics and assignee-required behavior. (Update in behavior_contract if desired.)

---

## analysis/test_plan.md

- Created high-level test plan per refactor plan (see test_plan.md).

---

## Finalisation pass

- **Agent 1:** Added routing-flag note in description and prompt: "Sets routing flag 'status' only when the changed field includes 'status'. NOTE: 'status' here is a routing flag, not lifecycle status." Added Role/Goal and Guardrails (must follow) section.
- **Agent 2:** Step 8 return: when proceed_status is Accepted (only path that reaches Step 8), omit reason; early returns in Step 2 and 5 already include reason. Added Role/Goal and Guardrails.
- **Agent 3:** Step 6 return: explicit source for work_request_id ([[work_request_object]][\"work_request_id\"]), owner_email_address from Step 3, task_title and due_date from work_request_object. Step 1 and Step 7 failure path: when owner_email_address not available, send failure email to CC list only (four addresses). Added Role/Goal and Guardrails.
- **Agent 4:** Empty list: if supporting_activity_values_list is empty after Step 2, return success with task_urls: [] and do not call send_email. Null owner_email_address on failure: send to CC list only. Added Role/Goal and Guardrails.
- **analysis/guardrails_and_prompt_conventions.md:** New document: prompt template structure (Role/Goal, Guardrails, Input, Process, Output), cross-agent guardrails summary, key semantics (routing flag vs lifecycle status; failure email to CC only when owner_email_address unavailable), references to refactor plan and risk register.
- **analysis/behavior_contract.md:** Optional note added that when owner_email_address is unavailable (e.g. failure before task fetch), failure email is sent to CC list only.

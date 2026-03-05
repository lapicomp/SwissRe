# Use Case 1: Guidelines, Context & Agent Analysis

**Use case:** Automating Task Creation from Work Requests (Create Tasks from Specific Work Requests)  
**Status (from demos):** ~95% complete  
**Source:** Demo docs + `original/` Opal agents + `analysis/behavior_contract.md`

---

## 1. Use case guidelines and context (from demos)

### 1.1 Goal

- **Automate** creation of tasks, campaigns, and milestones when a work request is **accepted** in CMP (Optimizely Content Management Platform).
- **Reduce** manual clicks and human error.
- **Trigger:** User submits a work request via form (e.g. “social organic”) → on **accept** in CMP, the Opal workflow runs and creates the corresponding task(s).

### 1.2 Standard process (single task)

1. **Work request submission** – User submits a work request through a form (e.g. type: “social organic”).
2. **Task creation by Opal** – When the work request is **accepted**, an Opal agent creates a task in CMP.
3. **Field mapping & workflow** – The agent maps fields from the work request to the new task and applies the correct workflow (e.g. “Workflow: Social Organic (MARKETING)”).

### 1.3 Advanced functionality: supporting activities

- **One work request → multiple related tasks.**  
  A single accepted work request can generate **one main task** plus **N “supporting activity” tasks** (e.g. “Emailing”, “Social Organic”, “Paid Search”).
- **Unconventional usage:** This uses the platform in an “unintended way”; different APIs are used to **link** the supporting tasks back to the original work request (`update_work_request_resource_link`).
- **Output difference:** Supporting-activity tasks get a **summary field** (brief) populated with a readable summary of **all** work request form fields; the main task does not get this same brief treatment in the same way.
- **Emails:** Assignee gets an email for the **main task** and a **separate email** for the **supporting activity tasks** (links to all created supporting tasks).

### 1.4 Demo workflow (conditions and agents)

From the demo description, the flow when a work request is accepted:

| Order | Step / agent (demo name) | Role |
|-------|---------------------------|------|
| 1 | **Agent 1** | Extract event type and work request ID from the webhook event. |
| 2 | **Condition** | If extracted value **contains** `"status"`. |
| 3 | **Agent 2** | Process work request details to create task(s): work_request id, template name, workflow name, assignee, channel, supporting activity, due date. |
| 4 | **Condition** | If value (status) **contains** `"Accepted"`. |
| 5 | **Agent 3** | Create task from work request using extracted metadata. |
| 6 | **Condition** | If value **contains** `"supporting_activity"`. |
| 7 | **Agent 4 / Agent w3** | Supporting activity agent: create multiple tasks, link to work request, send email (2 emails total: main task + supporting activity tasks). |

**Mapping to `original/` implementation:**  
Agent 1 → Extract Webhook; Agent 2 → Process Work Request; Agent 3 → Create Task from WR; Agent 4/w3 → Create Tasks for Supporting Activity (single specialized step in the workflow).

---

## 2. Mapping: demo agents → Opal `original/` agents

| Demo / doc | Opal step name | Agent ID | File in `original/` |
|------------|----------------|----------|----------------------|
| Trigger | (webhook) | — | `workflow.json` |
| Agent 1 | [ES-CMP] Extract Webhook Event Info | `es-cmp-extract-webhook-event-info` | `agent1_extract_webhook.json` |
| Agent 2 | [ES-CMP][V3] Process Work Request Details | `es-cmp-process-work-request-details-v-3` | `og_agent2_process_wr.json` |
| Agent 3 | [ES-CMP][V3] Create and Update Task from Work Request | `create-and-update-task-from-work-request-v-3` | `agent3_create_task.json` |
| Agent 4 / w3 | [ES-CMP][V3.1] Create and Update Tasks for Supporting Activity | `create-and-update-tasks-for-supporting-activity-2` | `agent4_supporting_activity.json` |

The workflow is linear: **Trigger → Extract → Process WR → Create Task → [optional] Supporting Activity**. There is no separate “Agent w3” step in the JSON; the supporting-activity behaviour is implemented in the single Agent 4 step.

---

## 3. Workflow structure (`original/workflow.json`)

- **Workflow name:** `[ES-CMP][V3] Create Tasks from Specific Work Requests`
- **Trigger:** Webhook `cmp_work_request_modified_event_v3`  
  - Auth: header `callback-secret` = `{token}` (auth_key: `Csm2vDE6rc`), payload `application/json`.
- **Steps (order):**
  1. **Extract Webhook Event Info** (conditional)  
     - Condition: output **contains** `"status"` → go to Process Work Request Details.
  2. **Process Work Request Details** (conditional)  
     - Condition: output **contains** `"Accepted"` → go to Create and Update Task from Work Request.
  3. **Create and Update Task from Work Request** (conditional)  
     - Condition: output **contains** `"supporting_activity"` → go to Create and Update Tasks for Supporting Activity.
  4. **Create and Update Tasks for Supporting Activity** (specialized, no condition)  
     - End of workflow.

**Note:** The first branch uses `"status"` but the Extract agent’s **declared** output schema only has `event_type` and `request_id`. The real behaviour may rely on the full payload or evaluator behaviour; document as-is for refactor (see behavior_contract.md).

---

## 4. Agent-by-agent analysis

### 4.1 Agent 1 – Extract Webhook Event Info

| Attribute | Value |
|-----------|--------|
| **File** | `original/agent1_extract_webhook.json` |
| **Agent ID** | `es-cmp-extract-webhook-event-info` |
| **Type** | specialized |
| **Inference** | simple |
| **Tools** | None |

**Purpose:**  
Parse the CMP work-request webhook and expose `event_type` and `request_id` for downstream steps.

**Inputs (from workflow context):**  
- Webhook payload; prompt refers to:
  - `webhook_payload.data.work_request.modified_fields` → Event Type
  - `webhook_payload.data.work_request.id` → Request ID

**Output schema (declared):**  
- `event_type` (string), `request_id` (string). Required.

**Parameters (agent def):**  
- “Event Type”, “Request ID” (optional in schema; logically they are the extracted values).

**Guidelines / notes:**  
- No tools; purely prompt-driven extraction from payload.
- Branch to Process WR uses “output contains `status`”; schema does not include `status`. Any refactor should either align schema with what the evaluator sees or make the condition explicit (e.g. document that the evaluator may receive full payload or a different view).
- Normalisation: ensure `request_id` is a string and trimmed if needed for `get_cmp_resource`.

---

### 4.2 Agent 2 – Process Work Request Details

| Attribute | Value |
|-----------|--------|
| **File** | `original/og_agent2_process_wr.json` |
| **Agent ID** | `es-cmp-process-work-request-details-v-3` |
| **Type** | specialized |
| **Inference** | simple_with_thinking |
| **Tools** | `get_cmp_resource` |

**Purpose:**  
Fetch the work request from CMP, validate template, resolve workflow and channel, and decide whether to proceed (“Accepted” / “Not Accepted”). Produces a single, structured object for task creation and optional supporting-activity step.

**Inputs:**  
- **Event Type** – from Agent 1 output `event_type`.
- **Request ID** – from Agent 1 output `request_id`.

**Process (from prompt):**  
1. Call `get_cmp_resource`(resource_type = `"work_request"`, resource_id = Request ID).  
2. From work request: `template_name`, `title`, `work_request_status`, `assignee_name`, `assignee_id`, `form_fields`.  
3. If `template_name` not in `template_mapping` → return `{"proceed_status": "Declined"}`.  
4. From `template_mapping[template_name]`: `workflow_id`, `workflow_name`, `title_prefix_1`, `channel_field`, `channel_extraction_rule`.  
   - If `channel_field` is null → `channel_value = null`.  
   - Else get `raw_channel = form_fields[channel_field]`; if null → `channel_value = null`; else apply rule (e.g. “before_colon”: split on “:”, take text before first colon, trim).  
5. Build `task_title`: if `channel_value` is null then `"{title_prefix_1} {title}"`, else `"{title_prefix_1} {channel_value} {title}"`.  
6. If `work_request_status == "Accepted"` and `workflow_id` is not null and template in mapping → `proceed_status = "Accepted"`; else return `{"proceed_status": "Declined"}`.  
7. Campaign: `form_fields["Campaign"]` or fallback `"[ES-CMP] Test Opal Usecase - Campaign"`.  
8. Due date: `form_fields["Deliverable Due Date"]` or `form_fields["Event End Date"]` or `form_fields["End Date"]`.  
9. Return the full output object including `supporting_activity` from `form_fields["Supporting Activity"]`.

**Output schema (declared):**  
- Required: `proceed_status` (enum: `"Accepted"` | `"Not Accepted"`).  
- Optional (string | null): `status`, `channel`, `campaign`, `due_date`, `task_title`, `assignee_id`, `workflow_id`, `assignee_name`, `template_name`, `workflow_name`, `work_request_id`, `supporting_activity`.

**Template mapping (in prompt):**  
- Social Organic, Social Paid, Web requests (MARKETING), Event Request, Email Marketing, Event Info Capture, Graphic Design, Sales Enablement Asset, Trade Media, Video Production.  
- Each entry: `workflow_name`, `workflow_id`, `title_prefix_1`, `channel_field`, `channel_extraction_rule`.

**Guidelines / notes:**  
- Single source of “Accepted” vs “Declined”; downstream steps only run when “Accepted”.  
- `supporting_activity` drives the branch to Agent 4; empty/null/absent → no supporting-activity step.  
- Refactor: normalise template names (trim, case) before lookup; handle missing `form_fields` keys; explicit handling of `work_request_status` (e.g. only “Accepted” leads to proceed).  
- Schema uses “Not Accepted” but prompt sometimes says “Declined”; align enum and prompt for consistency (refactor scope allows enum/required-field alignment).

---

### 4.3 Agent 3 – Create and Update Task from Work Request

| Attribute | Value |
|-----------|--------|
| **File** | `original/agent3_create_task.json` |
| **Agent ID** | `create-and-update-task-from-work-request-v-3` |
| **Type** | specialized |
| **Inference** | simple |
| **Tools** | `update_task`, `update_task_substep`, `get_cmp_resource`, `create_task_from_work_request`, `send_email` |

**Purpose:**  
Create the **main** CMP task from the work request, set substep assignee, and send success/failure email. If `supporting_activity` is present, pass through the data needed for Agent 4 (workflow branches on “output contains supporting_activity”).

**Inputs:**  
- **work_request_object** – Output of Agent 2 when `proceed_status === "Accepted"`: `work_request_id`, `template_name`, `workflow_id`, `workflow_name`, `status`, `assignee_name`, `assignee_id`, `campaign`, `supporting_activity`, `due_date`, `channel`, `task_title`.

**Process (from prompt):**  
1. Resolve `campaign_id` from `campaign_mapping_object` using `work_request_object["campaign"]`.  
2. `create_task_from_work_request`(work_request_id, title, campaign_id, due_at, owner_id, workflow_id).  
3. `get_cmp_resource`(resource_type = `"task"`, resource_id = new task_id) → get `first_workflow_step_name`, `cmp_task_url`, `owner_email_address`.  
4. `update_task_substep`(task_id, first_workflow_step_name, assignee_type = `"user"`, assignee_name).  
5. `send_email`(recipient = owner_email_address, CC list, subject “New Task Created from Work Request in CMP”, body with `cmp_task_url`).  
6. If `work_request_object["supporting_activity"]` is not null, return object including `supporting_activity`, `work_request_id`, `owner_email_address`, `due_date`, `task_title`, etc., for Agent 4. Else return the smaller success object.  
7. On any failure: send “Task creation failed in CMP” email, return `execution_status: "Failed"`, `reason: "Task creation failed during execution."`

**Output schema (declared):**  
- Required: `execution_status` (enum: `"Failed"` | `"Task created successfully"`).  
- Optional: `reason`, `task_id`, `due_date`, `campaign_id`, `cmp_task_url`.  
- **Pass-through for Agent 4 (not in schema):** `task_title`, `supporting_activity`, `work_request_id`, `owner_email_address`, `due_date` (and in practice `campaign_id` for supporting tasks).

**Campaign mapping (in prompt):**  
- Multiple campaign names → CMP campaign IDs (e.g. “[ES-CMP] Test Opal Usecase - Campaign”, “Event: Baden Baden (2025)”, etc.).

**Guidelines / notes:**  
- Typo in prompt: return object uses `[[work_request_object]]["Supporting Activity"]` (capital S, space) vs elsewhere `supporting_activity`; should be consistent (`supporting_activity`).  
- If `campaign_name` not in `campaign_mapping_object`, behavior is undefined; refactor: define fallback or explicit fail.  
- Agent 4 expects `supporting_activity_object` with `campaign_id` (ID, not name); Agent 3 returns `campaign_id` in schema, so that’s correct when present.

---

### 4.4 Agent 4 – Create and Update Tasks for Supporting Activity

| Attribute | Value |
|-----------|--------|
| **File** | `original/agent4_supporting_activity.json` |
| **Agent ID** | `create-and-update-tasks-for-supporting-activity-2` |
| **Type** | specialized |
| **Inference** | simple_with_thinking |
| **Tools** | `send_email`, `update_work_request_resource_link`, `update_task`, `create_task`, `get_cmp_resource`, `update_task_brief` |

**Purpose:**  
Create **multiple** CMP tasks from the comma-separated “Supporting Activity” values, optionally set brief (summary of work request) per task, link each new task to the work request, and send one email with all supporting-task links.

**Inputs:**  
- **supporting_activity_object** – From Agent 3 when it ran and had non-null `supporting_activity`: `task_id`, `task_title`, `campaign_id`, `cmp_task_url`, `supporting_activity`, `work_request_id`, `due_date`, `owner_email_address`.

**Process (from prompt):**  
1. `get_cmp_resource`(resource_type = `"work_request"`, resource_id = work_request_id) → build `work_request_fields_summary` (readable summary of form fields, newline-separated, skip h-tag text).  
2. Split `supporting_activity` on “,”, trim each value → `supporting_activity_values_list`.  
3. For each value in list:  
   - `create_task`(title = “{value} {task_title}”, campaign_id, due_date; workflow_id if in mapping).  
   - From response get task URL, derive `newly_created_task_id` (e.g. last path segment).  
   - If mapping has `template_id` for this value: `update_task_brief`(task_id, brief_form_template_id, update_summary with `work_request_fields_summary`).  
   - `update_work_request_resource_link`(action = `"link"`, resource_type = `"task"`, resource_id = new task_id).  
   - Append task URL to `task_url_list`.  
4. `send_email`(recipient = owner_email_address, CC list, subject “Supporting Activity Tasks Created from Work Request in CMP”, body with `task_url_list`).  
5. Return `execution_status: "Tasks created successfully"`, `task_urls`: task_url_list.  
6. On failure: send “Task creation failed in CMP”, return `execution_status: "Failed"`, `reason: "Task creation failed during execution."`

**Output schema (declared):**  
- Required: `execution_status` (enum: `"Failed"` | `"Task created successfully"` | `"Tasks created successfully"`).  
- Optional: `reason`, `task_id`, `campaign_id`, `cmp_task_url`.  
- **Actual success return:** `task_urls` (list) not in schema.

**Supporting-activity mapping (in prompt):**  
- Keys: Emailing, Sales Enablement Asset, Social Organic, Social paid, Trade media, Webpage, Paid Search, Publication, Paid Display.  
- Each: `workflow_name`, `workflow_id`, `template_name`, `template_id` (optional; used for `update_task_brief`).  
- Some entries have empty `workflow_id` or `template_id`; refactor: document skip/fallback behaviour.

**Guidelines / notes:**  
- “Supporting Activity” is the use case’s “unintended” path: multiple tasks linked to one work request via `update_work_request_resource_link`.  
- Brief summary: only applied when `template_id` is non-empty; `update_summary` carries the full work request summary so supporting tasks get context.  
- Refactor: normalise supporting-activity values (trim, case) before mapping lookup; skip or handle unmapped values explicitly; idempotency (e.g. avoid creating duplicate tasks on retries) if in scope.  
- Schema alignment: add `task_urls` to output schema when multiple tasks, or document as implementation detail.

---

## 5. Cross-agent data flow summary

```
Webhook (CMP)
    → Agent 1: event_type, request_id  [branch: contains "status"]
    → Agent 2: get_cmp_resource(work_request) → proceed_status, work_request_id, template_name,
       workflow_id, workflow_name, assignee_*, campaign, supporting_activity, due_date, channel, task_title
       [branch: contains "Accepted"]
    → Agent 3: create_task_from_work_request + substep + email
       → execution_status, task_id, cmp_task_url, campaign_id (+ pass-through for Agent 4)
       [branch: contains "supporting_activity"]
    → Agent 4: get_cmp_resource(work_request) → summary; for each supporting value:
       create_task → update_task_brief (if template_id) → update_work_request_resource_link
       → send_email with task_url_list
```

---

## 6. Conventions and refactor guidelines (aligned with refactor_Scope.md)

- **Allowed:**  
  - Make conditions explicit and deterministic (e.g. “status” vs schema; “Declined” vs “Not Accepted”).  
  - Harden nulls, missing keys, and mapping lookups (template_mapping, campaign_mapping, supporting_activity_mapping).  
  - Normalise strings (trim, case) for mappings.  
  - Minimal idempotency to avoid duplicate tasks.  
  - Schema alignment (enums, required fields; optional `task_urls` / pass-through fields).  
  - Minor notification/error handling improvements.

- **Not allowed (unless necessary for robustness):**  
  - New triggers, new business rules, new UI behaviour.  
  - Major redesign, agent count reduction, new orchestration layer.  
  - Changing Swiss Re process or outputs except for bug fixes.

- **Use-case-specific:**  
  - Preserve “one main task + N supporting tasks” and the two-email behaviour (main task email + supporting activity email).  
  - Keep supporting-activity brief (work request summary) behaviour and the use of `update_work_request_resource_link` for linking.

---

## 7. Document references

- **Behavior (current):** `analysis/behavior_contract.md`  
- **Refactor scope:** `analysis/refactor_Scope.md`  
- **Agents:** `original/agent1_extract_webhook.json`, `original/og_agent2_process_wr.json`, `original/agent3_create_task.json`, `original/agent4_supporting_activity.json`  
- **Workflow:** `original/workflow.json`

# Behavior contract: Create Tasks from Specific Work Requests (V3)

Precise description of current behavior from `original/workflow.json` and agents under `original/`. No improvements; current behavior only.

**Refactored workflow (`refactored/`):** Branch conditions are explicit equality (event_type equals "status", proceed_status equals "Accepted", has_supporting_activity equals true). When a branch condition does not match, the workflow ends by design (no error). Webhook fires on any work request modification; we proceed only when event_type === "status". Assignee is required when Accepted; Agent 2 returns Not Accepted with reason "Assignee required for task creation" otherwise. When owner_email_address is unavailable (e.g. failure before task fetch in Create Task, or null in Supporting Activity), failure email is sent to CC list only.

---

## 1. Trigger conditions (when the workflow runs)

- **Trigger type:** Webhook.
- **Trigger name:** `cmp_work_request_modified_event_v3`.
- **When it runs:** On receipt of a webhook request to the workflow’s webhook endpoint.
- **Auth:** Header `callback-secret` with value `{token}` (auth_key: `Csm2vDE6rc`). Payload content type: `application/json`.
- **No other triggers** are defined. The workflow does not run on schedule or other event types.

---

## 2. Branch conditions (when it proceeds)

Flow: **Trigger → Extract Webhook Event Info → Process Work Request Details → Create and Update Task from Work Request → [optional] Create and Update Tasks for Supporting Activity.**

| Step | Step name | Branch condition | Target when match | When no match |
|------|-----------|------------------|-------------------|----------------|
| 1 | [ES-CMP] Extract Webhook Event Info | Output **contains** `"status"` (`matching_condition`: `"status"`, `match_type`: `"contains"`) | Process Work Request Details | No edge defined; workflow effectively stops. |
| 2 | [ES-CMP][V3] Process Work Request Details | Output **contains** `"Accepted"` (`matching_condition`: `"Accepted"`, `match_type`: `"contains"`) | Create and Update Task from Work Request | Workflow stops; no task creation. |
| 3 | [ES-CMP][V3] Create and Update Task from Work Request | Output **contains** `"supporting_activity"` (`matching_condition`: `"supporting_activity"`, `match_type`: `"contains"`) | Create and Update Tasks for Supporting Activity | Workflow ends after main task; supporting-activity step is not run. |

**Note:** Extract Webhook agent’s declared output schema has only `event_type` and `request_id`; it does not include `status`. The first branch therefore relies on either a different evaluator behavior or an undocumented output. Documented as-is.

**Process Work Request** sets `proceed_status` to `"Accepted"` or (in the prompt) `"Declined"`; the schema uses enum `["Accepted","Not Accepted"]`. Branch to Create Task is when the evaluator sees `"Accepted"` (contained in the step output).

---

## 3. What is created in each scenario

| Scenario | Condition | Tasks created |
|----------|-----------|----------------|
| **A) Status change but not accepted** | Webhook fires, Extract runs, Process WR runs and returns `proceed_status` ≠ Accepted (e.g. `"Not Accepted"` / `"Declined"`). Branch to Create Task does not match. | **No tasks.** Create Task and Supporting Activity steps are never run. |
| **B) Accepted with no supporting activity** | Process WR returns `proceed_status` = Accepted and `supporting_activity` null/empty. Create Task runs. Branch to Supporting Activity does not match (no `supporting_activity` in output or empty). | **1 main task** (from Create and Update Task from Work Request). No supporting-activity step. |
| **C) Accepted with supporting activity list** | Process WR returns Accepted and non-empty `supporting_activity`. Create Task runs and produces output that contains `supporting_activity`. Branch matches. | **1 main task** (Create Task) **+ N supporting tasks** (Create and Update Tasks for Supporting Activity), where N = number of comma-separated values in `supporting_activity` (after trim). |

---

## 4. Mandatory outputs per agent (fields and types)

### 4.1 [ES-CMP] Extract Webhook Event Info (`es-cmp-extract-webhook-event-info`)

- **Required:** `event_type` (string), `request_id` (string).
- **Schema:** `object` with required `["event_type","request_id"]`.

### 4.2 [ES-CMP][V3] Process Work Request Details (`es-cmp-process-work-request-details-v-3`)

- **Required:** `proceed_status` (string, enum: `"Accepted"` | `"Not Accepted"`).
- **Optional (all string | null):** `status`, `channel`, `campaign`, `due_date`, `task_title`, `assignee_id`, `workflow_id`, `assignee_name`, `template_name`, `workflow_name`, `work_request_id`, `supporting_activity`.
- **Schema:** `object` with required `["proceed_status"]`; other fields nullable strings. Prompt also returns these when accepted: `work_request_id`, `template_name`, `workflow_id`, `workflow_name`, `status`, `assignee_name`, `assignee_id`, `campaign`, `supporting_activity`, `due_date`, `channel`, `task_title`.

### 4.3 [ES-CMP][V3] Create and Update Task from Work Request (`create-and-update-task-from-work-request-v-3`)

- **Required:** `execution_status` (string, enum: `"Failed"` | `"Task created successfully"`).
- **Optional (schema):** `reason` (string | null), `task_id` (string | null), `due_date` (string | null), `campaign_id` (string | null), `cmp_task_url` (string | null).
- **Prompt (when success and supporting_activity non-null):** also returns `task_title`, `supporting_activity`, `work_request_id`, `owner_email_address`, `due_date` for the next step; these are not in the declared output schema.

### 4.4 [ES-CMP][V3.1] Create and Update Tasks for Supporting Activity (`create-and-update-tasks-for-supporting-activity-2`)

- **Required:** `execution_status` (string, enum: `"Failed"` | `"Task created successfully"` | `"Tasks created successfully"`).
- **Optional (schema):** `reason` (string | null), `task_id` (string | null), `campaign_id` (string | null), `cmp_task_url` (string | null).
- **Prompt (on success):** returns `execution_status: "Tasks created successfully"` and `task_urls` (list); `task_urls` is not in the declared schema.

---

## 5. Tool calls per path

### 5.1 [ES-CMP] Extract Webhook Event Info

- **Enabled tools in JSON:** `null` (no tools).
- **Tool calls:** None. Uses only webhook payload (e.g. `webhook_payload.data.work_request.modified_fields`, `webhook_payload.data.work_request.id`).

### 5.2 [ES-CMP][V3] Process Work Request Details

- **Enabled tools:** `get_cmp_resource`.
- **Order:**  
  1. `get_cmp_resource`(resource_type = `"work_request"`, resource_id = Request ID from Extract output).

### 5.3 [ES-CMP][V3] Create and Update Task from Work Request

- **Enabled tools:** `update_task`, `update_task_substep`, `get_cmp_resource`, `create_task_from_work_request`, `send_email`.
- **Order (success path):**  
  1. `create_task_from_work_request`(work_request_id, title, campaign_id, due_at, owner_id, workflow_id from work_request_object).  
  2. `get_cmp_resource`(resource_type = `"task"`, resource_id = new task_id) → use first_workflow_step_name, cmp_task_url, owner_email_address.  
  3. `update_task_substep`(task_id, step_name = first_workflow_step_name, assignee_type = `"user"`, assignee_name).  
  4. `send_email`(success: recipient = owner_email_address, cc list, subject "New Task Created from Work Request in CMP", body with cmp_task_url).
- **On failure (any tool fails or task_id null):**  
  1. `send_email`(recipient = owner_email_address, cc list, subject "Task creation failed in CMP", body with failure message).  
  2. Return `execution_status: "Failed"`, `reason: "Task creation failed during execution."`

### 5.4 [ES-CMP][V3.1] Create and Update Tasks for Supporting Activity

- **Enabled tools:** `send_email`, `update_work_request_resource_link`, `update_task`, `create_task`, `get_cmp_resource`, `update_task_brief`.
- **Order (success path):**  
  1. `get_cmp_resource`(resource_type = `"work_request"`, resource_id = work_request_id) → build `work_request_fields_summary`.  
  2. For each value in comma-split, trimmed `supporting_activity`:  
     - `create_task`(title, campaign_id, due_date, workflow_id if in mapping).  
     - From response, get task URL and extract new task_id.  
     - If mapping has template_id: `update_task_brief`(task_id, brief_form_template_id, update_summary with work_request_fields_summary).  
     - `update_work_request_resource_link`(action = `"link"`, resource_type = `"task"`, resource_id = new task_id).  
     - Append task URL to task_url_list.  
  3. `send_email`(recipient = owner_email_address, cc list, subject "Supporting Activity Tasks Created from Work Request in CMP", body with task_url_list).
- **On failure:**  
  1. `send_email`(recipient = owner_email_address, cc list, subject "Task creation failed in CMP", body with failure message).  
  2. Return `execution_status: "Failed"`, `reason: "Task creation failed during execution."`

---

## 6. Email behavior (recipients and when)

| When | Recipients | CC | Subject |
|------|------------|-----|--------|
| **Main task created successfully** (Create Task step) | `owner_email_address` (from task resource) | Neil.Mullarkey@optimizely.com, suriya.disha@optimizely.com, Nicola.Dack@optimizely.com, mehreen.rahman@optimizely.com | "New Task Created from Work Request in CMP" |
| **Main task creation failed** (Create Task step) | `owner_email_address` | Neil.Mullarkey@optimizely.com, mehreen.rahman@optimizely.com, Nicola.Dack@optimizely.com, suriya.disha@optimizely.com | "Task creation failed in CMP" |
| **Supporting activity tasks created successfully** (Supporting Activity step) | `owner_email_address` (from supporting_activity_object) | Same four as above | "Supporting Activity Tasks Created from Work Request in CMP" |
| **Supporting activity task creation failed** (Supporting Activity step) | `owner_email_address` | Same four as above | "Task creation failed in CMP" |

- **When sent:**  
  - Success emails: after the corresponding step’s tool sequence completes (main task created and substep updated, or all supporting tasks created and linked).  
  - Failure emails: when any tool call fails, task_id is null, or an unexpected value occurs in that step; then the failure email is sent and the step returns Failed.
- **Note:** If Process WR returns not Accepted, no step runs that sends email. If Create Task is not run (branch not matched), no email. Supporting Activity email is only sent when that step runs (branch matched and executed).

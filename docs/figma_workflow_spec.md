# Figma workflow diagram – build spec

Use this spec to build a workflow diagram in Figma for demos and onboarding. The diagram should explain the CMP task creation flow with clear building blocks.

---

## 1. User / system triggers

**Section label:** User & system triggers

- **User actions (swimlane or callouts):**
  - "Submit work request in CMP"
  - "Assign and accept work request"
- **System:**
  - "CMP fires webhook on work_request_modified"

---

## 2. Orchestration layer

**Section label:** Opal workflow (orchestration)

**Swimlane:** Single row or band for "Opal Workflow".

**Nodes (left to right):**

1. **Webhook (Trigger)** – Trigger node; label: "cmp_work_request_modified_event_v2"
2. **Extract Webhook Event Info** – Agent/step node
3. **Process Work Request Details** – Agent/step node
4. **Create and Update Task from Work Request** – Agent/step node
5. **Create and Update Tasks for Supporting Activity** – Agent/step node (optional path)

**Edges (arrows) with labels:**

- Trigger → Extract: (no condition)
- Extract → Process WR: **event_type = "status"**
- Process WR → Create Task: **proceed_status = "Accepted"**
- Create Task → Supporting Activity: **has_supporting_activity = true**

When a condition is not met, the workflow stops (no arrow to next step).

---

## 3. Data sources

**Section label:** Data sources

- **CMP:** Work request (id, modified_fields, form_fields, status, assignee, template, campaign, supporting activity, due date).
- **Mappings (in agents):** template_mapping, campaign_mapping, supporting_activity_mapping.

Show as data-store or database icons with short labels; optional: link to "Agent 2 / 3 / 4" where each mapping lives.

---

## 4. User responsiveness / notifications

**Section label:** User notifications

- **Success – main task:** Email to task owner (and CC) with task link.
- **Success – supporting activity:** Email to owner (and CC) with links to all created supporting tasks.
- **Failure:** Email to owner or CC-only with failure reason.

Use a single "Notifications" or "Email" block with Success / Failure branches if desired.

---

## 5. Inputs / outputs per step

**Section label:** Inputs & outputs (optional detail panel or table)

| Step | Input | Output |
|------|--------|--------|
| (Trigger) | Webhook payload | event_name, data.work_request.* |
| Extract | Webhook payload | event_type, request_id |
| Process WR | event_type, request_id | proceed_status, work_request_id, template_name, workflow_id, workflow_name, status, assignee_*, campaign, supporting_activity, due_date, channel, task_title |
| Create Task | work_request_object | execution_status, task_id, cmp_task_url, campaign_id; if supporting: pass-through for Agent 4, has_supporting_activity |
| Supporting Activity | supporting_activity_object | execution_status, task_urls |

---

## 6. Building blocks legend

Include a small legend so the diagram is reusable:

- **Trigger** – Webhook or event that starts the workflow
- **Condition** – Branch condition (e.g. event_type = "status"); when not met, workflow ends
- **Agent** – Opal specialized agent / workflow step
- **Data source** – CMP or mapping data
- **User notification** – Email (success or failure)

---

## File and naming

- **Figma file name:** e.g. `SwissRe-CMP-Task-Creation-Workflow-V2`
- **Frame/page:** e.g. "V2 – Create Tasks from Work Requests"
- **Export:** PDF or PNG for docs; link to Figma for interactive demos.

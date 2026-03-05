# Troubleshooting: Workflow routing and campaign mapping

Notes from testing the refactored workflow (Feb 2025).

---

## 1. Workflow runs Create Task (Agent 3) when work request is not accepted

**Symptom:** You only assigned yourself to the work request (did not accept it), but the workflow ran Agent 3 and failed with e.g. `"Campaign not in campaign_mapping_object: null"`.

**Cause:** The conditional step that routes from **Process WR (Agent 2)** to **Create Task (Agent 3)** must use **exact equality** on the **proceed_status** field. If the platform compares the **entire evaluator output as a string** and uses **"contains"** matching, then the string `"Not Accepted"` **contains** the substring `"Accepted"`, so the condition would incorrectly match and Agent 3 would run.

**Fix:**

1. **In Opal:** Ensure the condition for the step "Process Work Request Details" → "Create and Update Task from Work Request" uses **match_type: "equals"** and that the comparison is against the **proceed_status** field value only (not the full JSON output). In `refactored/workflow.json` the condition is already `matching_condition: "Accepted"`, `match_type: "equals"`. If your deployment still uses "contains", change it to "equals" and confirm the platform compares the `proceed_status` field.
2. **Deploy Agent 2 (v3.1):** Use the agent in `refactored/agent2_process_wr.json` (agent_id: `es-cmp-process-work-request-details-v3-1`). Its prompt states that **proceed_status = "Accepted"** only when **work_request_status** from CMP is exactly **"Accepted"**; for "Assigned", "In Progress", "Pending", etc. it must return **"Not Accepted"**.

After this, when you only assign (without accepting), Agent 2 should return `proceed_status: "Not Accepted"` and the workflow should stop at Agent 2.

---

## 2. Campaign not in campaign_mapping_object: [ES CMP] Test Campaign

**Symptom:** Work request is accepted and campaign name is e.g. `"[ES CMP] Test Campaign"`, but Agent 3 fails with "Campaign not in campaign_mapping_object: [ES CMP] Test Campaign" even though that key exists in the mapping.

**Cause:** The agent may have been doing a **case-sensitive** or **non-normalised** lookup. CMP might send the campaign name with different casing or trimming.

**Fix:** Agent 3 prompt was updated to use **case-insensitive** lookup: find a key K in `campaign_mapping_object` such that `K.trim().toLowerCase() === campaign_name.trim().toLowerCase()`, then use `campaign_id = campaign_mapping_object[K]`. Redeploy `refactored/agent3_create_task.json` (and/or update the prompt from `refactored/prompt_templates.md`). The mapping already includes `"[ES CMP] Test Campaign"`; case-insensitive match should resolve mismatches due to casing.

---

## 3. When does the workflow create tasks?

**Intended behaviour:** Tasks are created **only when** a work request has been **accepted** (status = Accepted) and has an assignee. The webhook fires on any `work_request_modified` (e.g. assignee change, status change). The workflow intentionally runs Extract and Process WR on every such event, but **Create Task runs only when** Process WR returns **proceed_status = "Accepted"**, which happens only when the work request’s current status is "Accepted" and assignee is set.

So: **assign only** → Process WR returns "Not Accepted" → workflow stops at Agent 2. **Accept** (with assignee) → Process WR returns "Accepted" → Agent 3 runs.

If you want to reduce unnecessary runs (Extract + Process WR) when the change is not a status change, configure CMP to send the webhook only when the modified field is status (if your CMP supports that filter). Otherwise the current design is correct: run on every modification, gate at Agent 2.

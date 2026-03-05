# CMP webhook setup – refactored workflow

Checklist for the refactored flow to be triggered from the CMP.

**CMP** = Optimizely Content Management Platform (from this project’s use-case docs).

---

## Where to find webhook integration in the CMP

This repo does **not** document the exact CMP (Optimizely) UI path. Typically it is in one of:

- **Settings / Integrations** – look for “Webhooks”, “Integrations”, “Callbacks”, or “External notifications”.
- **Work request or workflow configuration** – sometimes the callback URL is set per workflow or per work-request type (e.g. where you configure what happens when a work request is submitted or modified).
- **Admin / Developer** area – if your org has an Optimizely admin or developer docs, the webhook URL and secret are often configured there.

**If you can’t find it:** ask your CMP/Optimizely administrator or check Optimizely’s product documentation for “webhooks” or “work request callbacks”. The URL and `callback-secret` header value must match what Opal shows for this workflow.

---

## CMP webhook events (subscribe to)

CMP lets you choose which events trigger the webhook. **This refactored workflow is designed for one event only:**

| Subscribe in CMP | Use in this workflow |
|------------------|----------------------|
| **`work_request_modified`** | ✅ **Yes** – when a work request is modified (e.g. status → Accepted). The flow uses `modified_fields` and only continues when the change is to **status**. |
| All other events | ❌ Not used by this workflow. Subscribing to them would send extra webhooks that this flow ignores or doesn’t handle. |

### Full list of CMP webhook events (for reference)

**Library**  
`asset_added`, `asset_modified`, `asset_removed`, `asset_renditions_created`

**Task**  
`content_preview_requested`, `task_added`, `task_asset_added`, `task_asset_draft_added`, `task_asset_draft_removed`, `task_asset_modified`, `task_asset_removed`, `task_brief_added`, `task_brief_modified`, `task_brief_removed`, `task_custom_field_modified`, `task_metadata_modified`, `task_removed`, `workflow_sub_step_comment_added`, `workflow_sub_step_comment_modified`, `workflow_sub_step_comment_removed`, `workflow_sub_step_updated`

**External Work Management**  
`external_sub_step_comment_added`, `external_sub_step_comment_modified`, `external_sub_step_comment_removed`, `external_sub_step_completed`, `external_sub_step_modified`, `external_sub_step_started`

**Campaign**  
`campaign_added`, `campaign_brief_added`, `campaign_brief_modified`, `campaign_brief_removed`, `campaign_metadata_modified`

**Work Request**  
`work_request_added`, `work_request_comment_added`, `work_request_comment_modified`, `work_request_comment_removed`, **`work_request_modified`**, `work_request_removed`

**Event**  
`event_added`, `event_modified`

**For this workflow:** In CMP, subscribe the webhook to **`work_request_modified`** only (matching the original behaviour). If you also subscribe to `work_request_added`, the flow would receive “created” events but the current logic expects `modified_fields` and status changes; handling “created” would require flow changes.

---

## 1. CMP must call the **refactored** workflow’s webhook URL

- The **refactored** (V4) workflow has a different Opal agent/workflow identity than the original (V3).
- In Opal, after you deploy/import the refactored `workflow.json`, it gets its **own** webhook URL (e.g. under the workflow’s trigger or “Webhook” / “Endpoint” section).
- In the **CMP**, the webhook / integration configuration must point to **this new URL**, not the old V3 workflow URL.
- If CMP is still configured with the previous workflow’s URL, the refactored flow will never receive requests.

**Action:** In CMP, open the integration/webhook settings and set the callback URL to the webhook URL shown in Opal for the refactored workflow.

---

## 2. Auth header must match

- **Header name:** `callback-secret`
- **Header value:** `Csm2vDE6rc` (from workflow `auth_key`; use `auth_header_format` `{token}` if Opal documents it that way)
- CMP must send this header on every webhook request; otherwise Opal may reject the request.

---

## 3. Trigger is “work request **modified**”, not “created”

- In CMP you must subscribe to the **`work_request_modified`** event (see “CMP webhook events” above). The trigger name in the workflow is `cmp_work_request_modified_event_v3`.
- The workflow **creates tasks only when a work request has been accepted**. It runs on every `work_request_modified` (e.g. assignee change, status change), but: (1) **Agent 1 (Extract)** forwards only when `modified_fields` has **`"status"`** as the changed field; (2) **Agent 2 (Process WR)** returns "Accepted" only when the work request’s current status is exactly **Accepted** and an assignee is set — otherwise it returns "Not Accepted" and the workflow stops (no task creation). So assigning yourself without accepting will not create a task.
- **Creating** a new work request fires **`work_request_added`**, not `work_request_modified`. This workflow does not handle `work_request_added`; it expects modification payloads with `modified_fields`.

**Action:** In CMP, subscribe the webhook to **`work_request_modified`**. To test, **accept** an existing work request (change **status** to **Accepted** and ensure it has an assignee); that sends a webhook with `modified_fields` including `"status"` and the flow will create tasks.

---

## 4. Refactored workflow is deployed and active

- In Opal, confirm the refactored workflow is **deployed** (or imported) and **active** (`is_active: true` in `workflow.json`).
- Confirm the webhook trigger is enabled and the URL is the one you configured in CMP.

---

## Quick check

| Check | What to verify |
|-------|----------------|
| URL in CMP | Equals the webhook URL of the **refactored** workflow in Opal |
| Auth | CMP sends header `callback-secret: Csm2vDE6rc` |
| Event | You **accepted** a work request (status → Accepted, assignee set); assign-only does not create tasks |
| Opal | Refactored workflow is deployed and active; webhook trigger enabled |

# Test plan (high-level): Create Tasks from Specific Work Requests

High-level test cases for the refactored workflow and agents. Run against `refactored/` artifacts.

**V2:** For the Version 2 workflow and agents, see also `version_2/TEST_CHECKLIST.md`.

---

## 1. Trigger and auth

- Webhook fires with valid `callback-secret`; payload `application/json`. Verify workflow starts.

## 2. Branch 1 – event_type

- Send webhook with `modified_fields` such that first element is `"status"` → Process WR runs.
- Send with first element not `"status"` (e.g. `"assignees"`) → workflow stops after Extract (no Process WR). No other triggers.

## 3. Assignee-only changes (from testing)

- Webhook with modified_fields = assignees (e.g. assign or remove assignee) → Extract runs, event_type ≠ "status" → workflow stops at step 1; no Process WR, no task created.

## 4. Accept then add assignee later

- Work request accepted first, then assignee added → second webhook fires with event_type = assignees → workflow stops at step 1; no duplicate task.

## 5. Branch 2 – proceed_status

- Process WR returns `proceed_status: "Accepted"` → Create Task runs.
- Returns `"Not Accepted"` → workflow stops (no Create Task, no email).

## 6. Accept without assignee (from testing)

- Work request accepted but no assignee → Agent 2 returns `proceed_status: "Not Accepted"`, reason "Assignee required for task creation" (or equivalent); workflow stops; no Create Task, no email; no null owner_id passed to Agent 3.

## 7. Branch 3 – supporting_activity

- Create Task returns non-empty `supporting_activity` and `has_supporting_activity: true` → Supporting Activity step runs.
- Empty/null → workflow ends after main task; one email only.

## 8. Agent 2 – Template not in mapping

- Template (after normalisation) not in mapping → `proceed_status: "Not Accepted"`, no task created.

## 9. Agent 2 – work_request_status

- Only when `work_request_status === "Accepted"` → `proceed_status: "Accepted"`; otherwise "Not Accepted".

## 10. Agent 3 – Campaign not in mapping

- Campaign name not in campaign_mapping_object → failure email, `execution_status: "Failed"`, reason contains campaign.

## 11. Agent 3 – Null task_id / owner_email_address

- Simulate null from create or get_cmp_resource → failure path: failure email, Failed + reason; no subsequent tool calls with null.

## 12. Send_email no loop (from testing)

- Run Create Task to success; verify exactly one success email is sent; agent returns JSON and does not call send_email again (no duplicate emails, no manual termination needed). Same for Agent 4 success path.

## 13. Agent 3 – Pass-through

- When supporting_activity is set, Agent 4 receives `supporting_activity` (same key) and creates N tasks.

## 14. Agent 4 – Unmapped/empty

- Unmapped supporting-activity value → skip that value; empty workflow_id/template_id → skip workflow or brief; no crash.

## 15. Agent 4 – form_fields null

- work_request with null form_fields → work_request_fields_summary empty string; tasks still created.

## 16. Agent 4 – Idempotency (Phase 2)

- Run same work request twice (or retry); second run skips creation when marker/linked tasks present; return success without duplicate tasks.

## 17. Schema

- All agent outputs conform to declared schemas (including `task_urls` for Agent 4, `has_supporting_activity` / `supporting_activity` for Agent 3).

## 18. Emails

- Success/failure emails and recipients (owner_email_address, CC list) as in behavior contract; two-email behaviour (main + supporting activity) unchanged.

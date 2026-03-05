# V2 testing checklist

Run these scenarios against the V2 workflow and agents. See also `analysis/test_plan.md` and `cmp_testing/` for full context and prior results.

---

## Trigger and auth

- [ ] Webhook fires with valid `callback-secret`; payload `application/json`. Workflow starts.

---

## Branch 1 – event_type

- [ ] Webhook with `modified_fields` first element `"status"` → Process WR runs.
- [ ] Webhook with first element not `"status"` (e.g. `"assignees"`) → workflow stops after Extract; no Process WR, no task.

---

## Branch 2 – proceed_status

- [ ] Process WR returns `proceed_status: "Accepted"` → Create Task runs.
- [ ] Process WR returns `"Not Accepted"` → workflow stops; no Create Task, no email.

---

## Assignee and status edge cases

- [ ] Assign only (no accept) → Process WR returns Not Accepted; no task created.
- [ ] Accept without assignee → Process WR returns Not Accepted, reason "Assignee required for task creation"; no Create Task.
- [ ] Accept then add assignee later → second webhook has event_type = assignees → workflow stops at Extract; no duplicate task.

---

## Branch 3 – supporting_activity

- [ ] Create Task returns non-empty `supporting_activity` and `has_supporting_activity: true` → Supporting Activity step runs.
- [ ] Empty/null supporting_activity → workflow ends after main task; one email only.

---

## Agent 2 – Template and status

- [ ] Template (after normalisation) not in mapping → `proceed_status: "Not Accepted"`, no task.
- [ ] Only when `work_request_status === "Accepted"` → `proceed_status: "Accepted"`; otherwise "Not Accepted".

---

## Agent 3 – Create Task

- [ ] Campaign name not in mapping (or case mismatch) → failure email, `execution_status: "Failed"`, reason contains campaign; case-insensitive lookup used.
- [ ] Null task_id or owner_email_address → failure path: one failure email, Failed + reason; no further tool calls with null.
- [ ] Exactly one success email sent on success; no send_email loop.
- [ ] Task owner equals work request assignee (owner_id / update_task_substep assignee).

---

## Agent 4 – Supporting Activity

- [ ] Unmapped supporting-activity value → skip that value; no crash.
- [ ] Empty workflow_id/template_id in mapping → skip workflow or brief; no crash.
- [ ] form_fields null → work_request_fields_summary empty string; tasks still created.
- [ ] Exactly one send_email (success or failure); no loop.
- [ ] Output includes `task_urls`; schema matches.

---

## Schema and emails

- [ ] All agent outputs conform to declared schemas (including has_supporting_activity, task_urls).
- [ ] Success/failure emails and recipients (owner_email_address, CC list) as in behavior contract; two-email behaviour (main + supporting activity) unchanged when applicable.

---

## References

- **Full test plan:** `analysis/test_plan.md`
- **Prior test results:** `cmp_testing/task_creation_test.md`, `cmp_testing/task_creation_retest.md`
- **Behavior contract:** `analysis/behavior_contract.md`

# Risk Register: Create Tasks from Specific Work Requests (Use Case 1)

Risks identified from the current workflow and agents (`original/`), per `behavior_contract.md` and `use_case_1_guidelines_and_agent_analysis.md`. Mitigations are scoped to 1–3 days (deterministic checks, null handling, idempotency, schema alignment).

---

## Summary table

| # | Where (file / step) | Type | Likelihood | Impact | Mitigation (1–3 days) |
|---|----------------------|------|------------|--------|------------------------|
| R1 | `workflow.json` step 1; `original/agent1_extract_webhook.json` | Correctness | Medium | High | Align branch condition with schema or document evaluator; add deterministic check (e.g. output includes a status-like field). |
| R2 | `original/og_agent2_process_wr.json` | Correctness, Maintainability | Medium | Medium | Align enum and prompt: use single value (e.g. `"Not Accepted"`) everywhere; no "Declined". |
| R3 | `original/og_agent2_process_wr.json` | Correctness, Reliability | Medium | High | Null/missing-key handling for `form_fields`; normalise template name (trim, case) before template_mapping lookup; explicit "Not Accepted" when template missing. |
| R4 | `original/og_agent2_process_wr.json` | Correctness | Low | High | Explicit validation: only `work_request_status === "Accepted"` sets `proceed_status = "Accepted"`. |
| R5 | `original/agent3_create_task.json` | Correctness | Medium | High | Define fallback campaign ID or explicit fail with reason when campaign not in `campaign_mapping_object`. |
| R6 | `original/agent3_create_task.json` | Correctness | Low | High | Fix prompt key: use `supporting_activity` consistently (not `[[work_request_object]]["Supporting Activity"]`) so Agent 4 receives pass-through. |
| R7 | `original/agent3_create_task.json` | Reliability | Low | High | Null check for `task_id` and `owner_email_address` before tool calls; explicit failure path and reason. |
| R8 | `original/agent4_supporting_activity.json` | Correctness, Maintainability | Medium | Medium | Normalise supporting-activity values (trim, case) before mapping; document/skip unmapped values; handle empty `workflow_id`/`template_id`. |
| R9 | `original/agent4_supporting_activity.json` | Reliability, Operability | Medium | High | Minimal idempotency: e.g. check if work request already has linked tasks for this run (or correlation id) to avoid duplicate tasks on retry. |
| R10 | `original/agent4_supporting_activity.json` | Reliability | Low | Medium | Null handling for missing `form_fields` when building `work_request_fields_summary`; empty summary fallback. |
| R11 | `original/agent4_supporting_activity.json` | Maintainability | Low | Low | Add `task_urls` to declared output schema (or document as implementation detail). |
| R12 | `workflow.json` (all conditional edges) | Operability | Medium | Medium | Document "no match = workflow stops by design"; optional: add logging/no-op if platform allows. |

---

## Detailed risk list

### R1 – First branch condition relies on "status" not in Extract output schema

- **Where:** `original/workflow.json` (step 1 branch); `original/agent1_extract_webhook.json`.
- **Type:** Correctness.
- **Likelihood:** Medium (evaluator may see full payload or undocumented output).
- **Impact:** High (workflow may stop at step 1 even when it should proceed).
- **Mitigation:** In refactored workflow/agent: either add a status-like field to Extract output schema and branch on it deterministically, or document that the evaluator uses a different view and add a minimal deterministic check so the branch is predictable (1–2 days).

---

### R2 – Process WR: "Declined" vs "Not Accepted" inconsistency

- **Where:** `original/og_agent2_process_wr.json` (prompt vs schema).
- **Type:** Correctness, Maintainability.
- **Likelihood:** Medium (agent may return "Declined" while branch checks for "Accepted" only; "Not Accepted" is schema enum).
- **Impact:** Medium (branch still works if evaluator is contains-based; schema/API consumers may break).
- **Mitigation:** Use a single enum value (e.g. `"Not Accepted"`) in both prompt and schema; remove "Declined" from prompt (same day).

---

### R3 – Process WR: template not in mapping; missing form_fields keys

- **Where:** `original/og_agent2_process_wr.json`.
- **Type:** Correctness, Reliability.
- **Likelihood:** Medium (new template or malformed payload).
- **Impact:** High (undefined behavior, wrong task or failure).
- **Mitigation:** Normalise template name (trim, case) before lookup; explicit null/missing-key handling for `form_fields`; return `proceed_status: "Not Accepted"` with optional reason when template not in mapping (1–2 days).

---

### R4 – Process WR: work_request_status not explicitly validated

- **Where:** `original/og_agent2_process_wr.json`.
- **Type:** Correctness.
- **Likelihood:** Low.
- **Impact:** High (could set Accepted on wrong status).
- **Mitigation:** Explicit check: set `proceed_status = "Accepted"` only when `work_request_status === "Accepted"` (and other conditions); else "Not Accepted" (same day).

---

### R5 – Create Task: campaign not in campaign_mapping_object

- **Where:** `original/agent3_create_task.json`.
- **Type:** Correctness.
- **Likelihood:** Medium (new campaign name).
- **Impact:** High (undefined behavior or API failure).
- **Mitigation:** Define fallback campaign ID (e.g. test campaign) or explicit fail with `execution_status: "Failed"` and clear reason (1 day).

---

### R6 – Create Task: typo in pass-through key for supporting_activity

- **Where:** `original/agent3_create_task.json` (prompt: `[[work_request_object]]["Supporting Activity"]`).
- **Type:** Correctness.
- **Likelihood:** Low (agent may still pass correct key).
- **Impact:** High (Agent 4 may not get `supporting_activity` → no supporting tasks).
- **Mitigation:** Fix prompt to use `supporting_activity` consistently for the object passed to Agent 4 (same day).

---

### R7 – Create Task: task_id or owner_email_address null after create/get

- **Where:** `original/agent3_create_task.json` (after `create_task_from_work_request`, `get_cmp_resource`).
- **Type:** Reliability.
- **Likelihood:** Low.
- **Impact:** High (tool failures, wrong or missing email).
- **Mitigation:** Null checks before `get_cmp_resource`, `update_task_substep`, and `send_email`; on null, take failure path with clear reason (1 day).

---

### R8 – Supporting Activity: unmapped values; empty workflow_id / template_id

- **Where:** `original/agent4_supporting_activity.json`.
- **Type:** Correctness, Maintainability.
- **Likelihood:** Medium (new or mistyped supporting-activity value).
- **Impact:** Medium (task created with wrong workflow, or skip with no feedback).
- **Mitigation:** Normalise values (trim, case) before mapping lookup; document skip/fallback for unmapped or empty `workflow_id`/`template_id`; optional: skip task or use default workflow (1–2 days).

---

### R9 – Supporting Activity: duplicate tasks on webhook/workflow retry

- **Where:** `original/agent4_supporting_activity.json`; workflow retry or duplicate webhook.
- **Type:** Reliability, Operability.
- **Likelihood:** Medium (retries, double submit).
- **Impact:** High (duplicate tasks linked to same work request).
- **Mitigation:** Minimal idempotency: e.g. check if work request already has linked tasks for this run (correlation id or "supporting activity processed" marker); skip creation if already done (1–3 days depending on platform capabilities).

---

### R10 – Supporting Activity: missing form_fields when building work_request_fields_summary

- **Where:** `original/agent4_supporting_activity.json`.
- **Type:** Reliability.
- **Likelihood:** Low.
- **Impact:** Medium (empty brief or API error).
- **Mitigation:** Null handling for missing `form_fields`; build summary only from present keys; empty string fallback for summary (same day).

---

### R11 – Supporting Activity: task_urls not in output schema

- **Where:** `original/agent4_supporting_activity.json` (declared schema vs prompt return).
- **Type:** Maintainability.
- **Likelihood:** Low.
- **Impact:** Low (runtime may still work; schema drift).
- **Mitigation:** Add `task_urls` (array of strings) to output schema, or document as implementation detail (same day).

---

### R12 – Workflow: no explicit path when branch conditions do not match

- **Where:** `original/workflow.json` (all three conditional steps).
- **Type:** Operability.
- **Likelihood:** Medium.
- **Impact:** Medium (workflow "stops" with no visible outcome; operators may not know if it was by design or failure).
- **Mitigation:** Document in behavior contract and runbooks that "no match = workflow ends by design"; optional: add logging or no-op step if platform supports it (1 day doc; more if logging added).

---

## Risk types (definitions)

- **Correctness:** Wrong result, wrong branch, or wrong data passed downstream.
- **Reliability:** Failures under valid input, retries, or partial failures (e.g. nulls, duplicates).
- **Maintainability:** Schema drift, inconsistent naming, or unclear behavior that makes changes error-prone.
- **Operability:** Visibility, runbook clarity, and idempotency for production operations.

---

## References

- **Behavior (current):** `analysis/behavior_contract.md`
- **Guidelines & agent analysis:** `guidelines/use_case_1_guidelines_and_agent_analysis.md`
- **Refactor scope:** `analysis/refactor_Scope.md`
- **Cursor rules:** `.cursor/rules/opal-agent-refactoring.mdc`, `.cursor/rules/opal-docs-and-structure.mdc`

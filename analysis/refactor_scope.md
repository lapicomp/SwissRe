# Refactor scope: Create Tasks from Specific Work Requests (Use Case 1)

Agreed scope for refactoring the Opal workflow and agents. Refactored artifacts live in `refactored/`; `original/` is read-only. Scope items are tied to the risk register where applicable.

---

## Allowed (in scope)

### Deterministic conditions

- Make workflow branch conditions explicit and predictable (e.g. branch on a field that exists in the step output schema, not on undocumented payload).
- **Risk register:** [R1] (first branch uses `"status"` not in Extract schema), [R12] (document when "no match" = workflow stops by design).

### Null and edge-case handling

- Handle nulls, missing keys, and empty values in payloads and API responses so agents fail safely with a clear reason instead of undefined behaviour.
- **Risk register:** [R3] (template not in mapping, missing `form_fields`), [R4] (explicit `work_request_status` validation), [R5] (campaign not in mapping), [R7] (null `task_id` / `owner_email_address`), [R8] (unmapped/empty supporting-activity values), [R10] (missing `form_fields` for work request summary).

### Idempotency to avoid duplicates

- Minimal idempotency so retries or duplicate webhooks do not create duplicate tasks (e.g. check if work request already has linked tasks for this run before creating supporting-activity tasks).
- **Risk register:** [R9].

### Improvements to mappings

- Normalise lookup keys (trim, case) for template_mapping, campaign_mapping, and supporting-activity mapping.
- Define fallbacks or explicit "not in mapping" behaviour (e.g. return "Not Accepted" or "Failed" with reason) instead of undefined behaviour.
- **Risk register:** [R3], [R5], [R8].

### Better error reporting

- Clear, consistent failure reasons in output (e.g. `execution_status: "Failed"`, `reason: "..."`).
- Fix pass-through key typos so downstream steps receive expected data (e.g. `supporting_activity` not `"Supporting Activity"`).
- Schema alignment: enums and required fields match prompt returns; add missing output fields (e.g. `task_urls`) to schema or document as implementation detail.
- **Risk register:** [R2] (enum alignment), [R6] (pass-through key), [R11] (schema alignment).

---

## Not allowed (unless necessary for robustness)

- **New triggers** – No new webhooks, schedules, or event types.
- **New features** – No new business rules, new UI behaviour, or net-new capabilities beyond robustness fixes.
- **Major workflow redesigns** – No change to the linear flow (Extract → Process WR → Create Task → [optional] Supporting Activity), no new orchestration layer, no agent consolidation/rewrite unless required to implement an in-scope mitigation.
- **Changes to user-facing behaviour** – Swiss Re process and user-visible outcomes (e.g. emails, task creation, two-email behaviour for main + supporting activity) stay the same except for bug fixes and clearer failure handling.

If a change is needed to achieve an in-scope mitigation (e.g. a small workflow tweak for deterministic branching), it is allowed only to the extent required and must preserve functional parity.

---

## Effort and priorities

- Keep total effort within **1–3 working days**.
- Prioritise **Critical + Stability**: correctness (wrong branch/data), reliability (nulls, duplicates), then maintainability (schema, naming). See risk register summary table for likelihood/impact.

---

## Cursor rules: keep scope and rules lean

When updating or adding Cursor rules (e.g. under `.cursor/rules/`):

- **Keep rules under ~500 lines** – Prefer short, scannable rules; link to full docs instead of inlining them.
- **Be concrete** – Use concrete examples and file paths (e.g. `analysis/risk_register.md`, `refactored/`) rather than vague guidance.
- **Reference, don’t duplicate** – Point to `analysis/refactor_scope.md`, `analysis/behavior_contract.md`, and the risk register instead of copying their content into rules.
- **Avoid bloat** – Don’t copy entire style guides or document every command; only what’s needed for this refactor and parity.

This keeps rules concise and maintainable while staying aligned with Cursor rule best practices.

---

## References

- **Risk register (mitigations ↔ scope):** `analysis/risk_register.md`
- **Current behaviour:** `analysis/behavior_contract.md`
- **Guidelines & agent analysis:** `guidelines/use_case_1_guidelines_and_agent_analysis.md`
- **Cursor rules:** `.cursor/rules/opal-agent-refactoring.mdc`, `.cursor/rules/opal-docs-and-structure.mdc`

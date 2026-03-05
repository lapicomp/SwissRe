# Guardrails and prompt conventions (refactored agents)

Single reference for how refactored agents are structured and what guardrails are in place. Use this to verify behaviour and maintain prompts without opening each agent JSON.

---

## 1. Prompt template structure

Every refactored agent prompt follows the same structure (aligned with Optimizely Opal prompt template examples and `docs/opal-agent-examples/specialized-agents/bank-content-compliance.json`):

1. **Role / Goal** — One or two sentences: what this agent does and when it runs.
2. **Guardrails (must follow)** — Numbered list of non-negotiable rules (call-once, null handling, no re-entry, etc.).
3. **Input** — What the agent receives (variables, payload paths).
4. **Process** — Numbered steps (Step 1, Step 2, …). Early returns and final return object are unambiguous.
5. **Output** — Exact JSON shape and when to return it; tied to “after Step N, return only this; do not go back.”

---

## 2. Cross-agent guardrails summary

| Area | Rule |
|------|------|
| **Call-once** | Agents that use tools call each tool at most once per execution path (or once per loop iteration where applicable). Agent 2: get_cmp_resource once. Agent 3: create_task_from_work_request, get_cmp_resource, update_task_substep, send_email each at most once per path. Agent 4: get_cmp_resource once; send_email once after the loop. |
| **send_email** | Call send_email exactly once per path (success or failure). After calling send_email, do not call it again; proceed immediately to return the final JSON. Do not return to a previous step after send_email. |
| **Null handling** | On null task_id, null owner_email_address, or missing required data, take the failure path immediately. Do not call further tools with null. |
| **Failure email recipient** | When owner_email_address is not available (e.g. failure before get_cmp_resource in Agent 3, or null in Agent 4), send failure email to **CC list only**: Neil.Mullarkey@optimizely.com, suriya.disha@optimizely.com, Nicola.Dack@optimizely.com, mehreen.rahman@optimizely.com. |
| **No re-entry** | After completing a step (especially send_email or return), do not go back to an earlier step. Steps are strictly sequential. |
| **Empty list (Agent 4)** | If supporting_activity_values_list is empty after split/trim, return success with task_urls: [] and do not call send_email. |

---

## 3. Key semantics

- **Routing flag vs lifecycle status:** In Agent 1, `event_type` is set from the first element of `modified_fields` (what changed). When the changed field is the status field, `event_type` is `"status"`. This is a **routing flag** for the workflow (proceed only when event_type === "status"), not the work request’s **lifecycle status** (e.g. Accepted, Pending). Lifecycle status is read later by Agent 2 from the work request resource.
- **Failure email when owner_email_address unavailable:** If the agent fails before it has fetched the task (e.g. campaign not in mapping, or task_id null), owner_email_address is not available. In that case the failure email is sent to the **CC list only** (same four addresses as in the behavior contract).

---

## 4. References

- **Refactor plan:** See plan “Finalise refactored agents and workflow” (and the main refactor plan for Phase 1/2).
- **Risk register:** `analysis/risk_register.md`
- **Behavior contract:** `analysis/behavior_contract.md`
- **Changelog:** `analysis/changelog.md`

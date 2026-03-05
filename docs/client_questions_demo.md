# Client questions for demo

Use these questions in the demo to fill gaps and align on behaviour and environment. Document answers and update V2 behaviour and mappings as needed.

---

## Trigger and CMP

1. **Webhook filtering:** Does your CMP allow configuring the webhook to fire only when certain fields change (e.g. only when `modified_fields` includes "status")? If yes, we can reduce unnecessary runs and token use.

2. **Test environment:** In Test, why does the automation often not trigger (e.g. workflow disabled, wrong webhook URL, or different CMP config)? Can we align Test and Prod webhook configuration?

---

## Campaign and intake

3. **Campaign determination:** How should campaign be determined when the work request form uses a dropdown that is not linked to CMP campaigns? Prefer: static mapping table, external file (e.g. Excel), or fallback "undefined" campaign for later reassignment?

4. **Intake clarity:** Are you open to an "intake clarity" pass: mandatory fields (e.g. assignee, due date, campaign), validation rules, and a short schema so the agent receives consistent data?

---

## Owner and assignee

5. **Task owner:** Task owner should equal work request owner (assignee). In our tests it did not. Does CMP have a specific API or field we must use to set task owner, or is there a permission/configuration that overrides it?

6. **No assignee when accepted:** When a work request is accepted but has no assignee, should the workflow (a) stop and send no email, (b) stop and notify someone (e.g. CC list), or (c) create the task with a default owner?

---

## Supporting activities

7. **Supporting activity types:** Is the list of supporting activity types (Emailing, Social Organic, Paid Search, etc.) stable? Any new types we must add to the mapping?

8. **Linking behaviour:** For supporting-activity tasks, is linking them back to the work request via `update_work_request_resource_link` still the required behaviour?

---

## Operational and demo

9. **Email recipients:** Who should receive success/failure emails (recipients and CC)? Should we keep the current Optimizely addresses or switch to SwissRe-only?

10. **Fallback campaign:** Do you want a single "fallback" campaign ID for unmapped campaign names, or explicit failure with a clear reason (and no task created)?

11. **Demo scenarios:** For March/April demos, which scenarios must work reliably (e.g. Social Organic, Web Requests, one supporting activity)? We’ll prioritise those in V2 and test plans.

---

## Answers (to be filled during/after demo)

| # | Question topic | Answer | Action |
|---|----------------|--------|--------|
| 1 | Webhook filtering | | |
| 2 | Test env trigger | | |
| 3 | Campaign source | | |
| 4 | Intake clarity | | |
| 5 | Task owner API | | |
| 6 | No assignee | | |
| 7 | Supporting activity list | | |
| 8 | Linking behaviour | | |
| 9 | Email recipients | | |
| 10 | Fallback campaign | | |
| 11 | Demo scenarios | | |

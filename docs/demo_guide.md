# ES-CMP Automation — Demo Guide

---

## What triggers it

A webhook fires every time a Work Request is modified in CMP. The automation checks if the status changed to **Accepted** — that's the only trigger. Any other modification (field edits, reassignments, etc.) is ignored.

---

## The 3-Agent Flow

```
Work Request accepted
        ↓
  Agent 1 — Process WR
  Reads the WR, validates it, extracts fields
        ↓
  Agent 2 — Create Task
  Creates the main CMP task, assigns it, sends email
        ↓
  Agent 3 — Supporting Activity  (Event Requests only)
  Creates one task per supporting activity, fills each brief
```

---

## Agent 1 — Process Work Request

**What it does:** Reads and validates the incoming webhook, then extracts everything needed for task creation.

**Key decisions:**
- Checks `modified_fields` contains `"status"` — if not, stops immediately (no task created)
- Fetches the full WR from CMP to confirm status is `"Accepted"`
- Looks up the WR template name in mappings.txt to find the right workflow
- Extracts: title, campaign, due date, assignee, supporting activities, all form fields
- Sets `has_supporting_activity = true` only for **Event Request** templates with a non-empty Supporting Activity field
- If the template isn't in the mappings → rejected, no task created

---

## Agent 2 — Create Task

**What it does:** Creates the main CMP task and populates the task template.

**Key decisions:**

**Deduplication:** Before creating anything, checks if a task already exists in the same workflow for this WR. If yes → skips entirely, no duplicate is created.

**Campaign mapping:** Matches the WR campaign name to a CMP campaign ID. If no match is found, falls back to the default campaign (`[ES-CMP] Test Opal Usecase - Campaign`).

**Assignee handling:** If the WR has an assignee, the task is created with them as owner and they're assigned to the first workflow substep. If there's no assignee, the task is created without an owner and the substep assignment is skipped entirely. *(Routing logic is pending your feedback.)*

**Email:** Sends one notification email to the WR owner (CC to the Optimizely team) once the task is created successfully. If no owner email is available, sends to the CC list only.

---

## Agent 3 — Supporting Activity Tasks (Event Requests only)

**What it does:** For each activity listed in the "Supporting Activity" field of the Event WR, creates a separate CMP task and fills its brief template with data from the WR.

**Key decisions:**

**Per-activity loop:** Splits the Supporting Activity field on commas (e.g. `"Webpage, Social Organic, Social Paid"`) and processes each one independently. Unknown activity types are silently skipped.

**Activity mapping:** Each activity type is looked up in mappings.txt to find its workflow ID and brief template ID.

**Brief template filling — how fields are mapped:**

| Situation | What happens |
|---|---|
| Field label semantically matches a WR field (e.g. "Region" ↔ "Event Region") | WR value is used, closest choice selected |
| Description / Summary / Body text area | Filled with the full WR form fields as plain text |
| Date field | Uses the WR due date |
| Required selection field, no WR match | Set to `"choose"` placeholder |
| Required text/text_area field, no WR match | Set to `"N/A"` placeholder |
| Optional field, no WR match | Left blank entirely — never filled with a placeholder |

**Template-specific overrides (OVR rules):**

| Rule | What it handles |
|---|---|
| OVR-2 | Social Organic: "Legal sign-off" always set to "Yes - required before publishing" |
| OVR-3 | Social Organic: "Social Media Account" defaults to "LinkedIn: Swiss Re" when not in WR |
| OVR-4 | Social Paid: "Paid Media Budget" defaults to `0` when not in WR |
| OVR-5 | Graphic Task: "Content Format" defaults to "Image" unless WR specifies "Video" |
| OVR-6 | Trade Media: "Publication List" falls back to "Other" if WR value doesn't match any choice |
| OVR-7 | Business Unit: uses WR label name and selects closest match *(translation table pending)* |
| OVR-8 | Webpage: "Type of Web Request" defaults to "New webpage" when not specified in WR |

**Email:** Sends one email listing all created task URLs once the full loop is complete.

---

## Template Change: Why the Webpage On-page SEO Fields Were Made Optional

The Webpage brief template has three conditional sections — "New webpage", "Change of an existing webpage", and "On-page SEO" — controlled by the "Type of Web Request" radio button. In the CMP UI, selecting "New webpage" visually hides the On-page SEO section. However, the CMP API does not apply that conditional logic when validating a brief submission — it treats every field marked `is_required: true` as always required, regardless of which section is active in the UI.

This meant that every automated Webpage brief submission was being rejected because the On-page SEO fields (Link/URL, Topics, Country etc.) had no relevant data from the Event WR and couldn't be left blank. The fix was to mark those fields as `is_required: false` directly in the template. They remain visible and usable in the UI for genuine SEO requests, but the API no longer blocks submission when they're absent.

---

## Pending — Awaiting Client Feedback

These behaviours are functional but the defaults may need adjusting based on your input:

1. **Assignee routing** — currently assigns the task to the WR owner. Should it route differently for different task types or workflows?
2. **Due date handling** — currently uses the WR "Deliverable Due Date", falling back to "Event End Date" then "End Date". Is this the right priority order?
3. **Field default values** — `"choose"` and `"N/A"` are placeholders for required fields with no WR data. Should these default to something more specific per template?

---

## What you'll see in the demo

**Single task creation** (e.g. Social Organic WR accepted):
- Agent 1 validates and routes
- Agent 2 creates one task with the correct workflow, title prefix, and assignee
- Email sent to WR owner

**Event Request with supporting activities** (e.g. Webpage + Social Organic):
- Agent 1 + 2 create the main Event task
- Agent 3 creates one additional task per activity, each with their brief template pre-filled from the WR data
- One email listing all task URLs

---

## What's Changed from the Original (V3 → V5)

**1. Fewer agents: 4 → 3**
The original had a dedicated first agent just to extract the webhook event type and WR ID, passing those as separate string parameters to the next agent. V5 consolidates this — Agent 1 receives the full webhook payload directly, eliminating an unnecessary hop and the fragility of passing partial data between agents.

**2. Mappings moved out of the prompts**
The original had all template mappings, campaign IDs, and supporting activity mappings hardcoded inside the agent prompt strings themselves. Every time a new campaign, template, or activity type was added, a prompt had to be edited and redeployed. V5 externalises all of this to a single `mappings.txt` reference file uploaded to each agent — adding or updating a mapping is now a one-file change with no prompt edits needed.

**3. Stricter trigger gate**
The original took the first element of `modified_fields` and used it as an event type, meaning almost any WR modification could trigger the automation. V5 explicitly checks that `modified_fields` contains `"status"` — so it only fires on status changes, not field edits, reassignments, or other modifications.

**4. Deduplication check added**
The original had no guard against creating duplicate tasks. If the webhook fired twice for the same WR (which can happen), two tasks would be created. V5 checks whether a task already exists in the same workflow for that WR before creating anything.

**5. Null-safe assignee handling**
The original always passed `owner_id` to task creation even when `assignee_id` was null, causing failures on unassigned WRs. V5 only passes `owner_id` if it's non-null, skips the substep assignment if there's no assignee name, and falls back to CC-only email if `owner_email_address` is unavailable.

**6. Brief filling: "choose" for everything → semantic field mapping**
This is the biggest change. The original brief update instruction was literally: set all required fields to `"choose"` and put the WR summary in the Description field — every brief was filled with placeholders across the board. V5 actually maps WR data to brief fields intelligently:
- Semantic matching (e.g. "Event Region" → "Region", "Business Unit / Function" → "Business Unit")
- Date mapping from WR due date
- Template-specific OVR rules for fields that have no direct WR equivalent
- `"N/A"` for required text fields with no WR data, `"choose"` only as a last resort for selection fields
- Optional fields left blank entirely rather than filled with placeholders

**7. WR data passed through rather than re-fetched**
The original Agent 4 made an extra API call to fetch the WR again just to get its form fields. V5 passes `form_fields` through the entire chain from Agent 1 → 2 → 3, saving an API call per execution.

**8. Campaign fallback formalised**
The original hardcoded a fallback campaign name as a string inside the prompt. V5 uses a `_fallback` key in `mappings.txt`, making it explicit, visible, and easy to change without touching any prompt.
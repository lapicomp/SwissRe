# Stakeholder meeting prep – CMP task creation workflow

**Purpose:** Prepare for the upcoming stakeholder meeting by having a clear picture of the **current** workflow, its inputs/outputs, known issues and gaps, and **targeted questions** to gather comprehensive information. The goal is to leave with no outstanding uncertainties—not to propose technical solutions before business value and top-level requirements are understood.

**Approach:** Stakeholders often know what they want rather than exactly what they need. The team should collect information first and later propose how the system should function. This doc supports that by (1) documenting what we know, (2) flagging what we don’t know or assume, and (3) framing questions that uncover process, expectations, and context—without leading toward agent count, tool choice, or implementation details.

---

## 1. Original workflow at a glance (business view)

| What happens | When it runs | Outcome |
|-------------|--------------|---------|
| **Trigger** | CMP sends a webhook when a work request is **modified** (any field). | Workflow starts. |
| **Step 1 – Extract** | Reads the webhook: *which field changed*, *which work request*. | We get “event type” and “work request ID”. |
| **Decision** | We only continue if the change was to **status** (not e.g. only assignee). | If not status change → workflow stops (no task, no email). |
| **Step 2 – Process work request** | Fetches the full work request from CMP; checks template and status; builds task title, campaign, due date, supporting activity from form fields. | We get “proceed: Accepted / Not Accepted” and all data needed to create the main task (and optionally supporting tasks). |
| **Decision** | We only continue if the work request is **Accepted**. | If not Accepted → workflow stops. |
| **Step 3 – Create main task** | Creates one task in CMP from the work request (campaign, workflow, assignee, due date); assigns the first step; sends “task created” email. | One main task + email to owner and CC. |
| **Decision** | We only run the next step if the work request had **supporting activity** (e.g. “Emailing, Paid Search”). | If no supporting activity → workflow ends after main task. |
| **Step 4 – Supporting activity tasks** | For each supporting activity value: creates an extra task, links it to the work request, optionally fills the brief from work request summary; sends one email with all task links. | N extra tasks + one summary email. |

**Data the workflow relies on (sources):**

- **CMP:** Work request (id, modified_fields, form_fields, status, assignee, template name, campaign name, supporting activity, due date); task resource (URL, owner email, first step name).
- **Mapping tables (currently inside the automation):** Template name → workflow ID and rules; campaign name → campaign ID; supporting activity name → workflow ID and brief template ID.

---

## 2. Inputs and outputs per step (reference)

Use this to align with stakeholders on “what goes in and what comes out” of each step. Technical field names are in parentheses where useful for disambiguation.

| Step | Input (what the step receives) | Output (what the step produces / passes on) |
|------|--------------------------------|---------------------------------------------|
| **Trigger** | Webhook from CMP (event, work request id). | — |
| **1 – Extract** | Webhook payload: `modified_fields`, `work_request.id`. | `event_type` (what changed), `request_id` (work request ID). |
| **2 – Process work request** | `event_type`, `request_id`. Uses CMP API to fetch full work request. Uses **template mapping** (template name → workflow_id, title prefix, channel rules). | `proceed_status` (Accepted / Not Accepted), `work_request_id`, `template_name`, `workflow_id`, `workflow_name`, `status`, `assignee_name`, `assignee_id`, `campaign` (name), `supporting_activity` (comma‑separated string), `due_date`, `channel`, `task_title`. |
| **3 – Create main task** | Object from step 2 (work request metadata). Uses **campaign mapping** (campaign name → campaign_id). | `execution_status`, `task_id`, `cmp_task_url`, `campaign_id`. When there is supporting activity: also passes `supporting_activity`, `work_request_id`, `owner_email_address`, `due_date`, `task_title` (and related) to step 4. |
| **4 – Supporting activity** | Object from step 3 (task_id, task_title, campaign_id, supporting_activity, work_request_id, due_date, owner_email_address). Uses **supporting activity mapping** (activity name → workflow_id, template_id). Fetches work request again for brief summary. | `execution_status`, `task_urls` (list of created task URLs). |

**Known gaps in current design:**

- Step 1’s **declared** output is only `event_type` and `request_id`; the workflow branches on “contains status”, which is not a separate field—can cause ambiguity.
- Step 3 sometimes used a different key for the object passed to step 4 (“Supporting Activity” vs `supporting_activity`), which can break the link.
- Some outputs used in practice (e.g. `task_urls`, pass-through fields for step 4) are not fully reflected in the declared output schema—documentation/contract gap.

---

## 3. Existing issues and gaps (to clarify with stakeholders)

These are documented from analysis, testing, and prior stakeholder notes. They are grouped by **data/context**, **mappings and config**, **process/expectations**, and **environment**.

### 3.1 Data and context gaps

| Issue | What we know | What we need from stakeholders |
|-------|----------------|--------------------------------|
| **Template IDs and names** | Automation uses a fixed **template mapping** (template name → workflow_id, title prefix, channel rules). Some templates have different display names vs. internal names. | Which template names appear in the work request form? Any new templates planned? Who owns the list and how should it be updated? |
| **Campaign list** | Work request form uses a **dropdown not linked to CMP campaigns**; automation maps **campaign name (string) → campaign_id** via an internal table. Tests showed wrong campaign when name didn’t match (e.g. fallback used). | How is the dropdown populated? Same list as CMP campaigns or different? Who maintains campaign names and how often do they change? |
| **Supporting activity types** | Automation has a **supporting_activity mapping** (e.g. “Emailing”, “Paid Search”) to workflow_id and brief template_id. Some entries have empty `workflow_id` or `template_id`. | Is the list of supporting activity types stable? Any new types or renames? What should happen if a value is not in the mapping? |
| **Form field names and meaning** | We read fields such as “Campaign”, “Supporting Activity”, “Deliverable Due Date”, “Event End Date”, “End Date”, “Social Media Account”. | Are these names and meanings agreed and stable? Any optional vs mandatory rules we should enforce before creating tasks? |
| **“Requester” vs “assignee” vs “owner”** | Automation uses **assignee** from the work request as the **task owner**. Tests expected “work request owner” (e.g. requester) but saw a different user. | Who should be the **task owner** in CMP: the person who accepted/assigned the request, the requester, or someone else? What does “owner” mean in your process? |
| **Missing or placeholder data** | Demos showed values like “choose” or incomplete fields (e.g. zero attendees) causing odd behaviour. | What are the real-world rules for missing or placeholder values? Should the system block creation, use defaults, or notify someone? |

### 3.2 Mappings and configuration issues

| Issue | What we know | What we need from stakeholders |
|-------|----------------|--------------------------------|
| **Campaign not in mapping** | If campaign name is not in the mapping, current behaviour is undefined; tests showed wrong campaign (e.g. fallback). | When campaign is unknown or not selected: prefer **fallback campaign** (e.g. “undefined” / “others”) for later reassignment, or **explicit failure** (no task, clear message)? |
| **Template not in mapping** | New or renamed templates lead to undefined behaviour. | How often do new templates appear? Who decides when to add them to the automation? |
| **Supporting activity unmapped or empty** | Some mapping entries have empty workflow_id/template_id; unmapped values were not clearly defined. | For unmapped or mis-typed supporting activity: skip that item, fail the whole step, or use a default? |
| **Campaign list source** | Today the campaign list is hard-coded in the automation. Prior discussion mentioned an external source (e.g. Excel) for easier updates. | Do you want campaign (and optionally template/supporting activity) lists to be updated without code changes? If yes, in what form (e.g. shared file, admin UI)? |

### 3.3 Process and expectations

| Issue | What we know | What we need from stakeholders |
|-------|----------------|--------------------------------|
| **When to run automation** | Webhook fires on **any** work request change; we only **proceed** when the change is to “status”. So we still run step 1 when only assignee (or other field) changes, then stop. | Should automation run only when status changes to Accepted, or also on other events? Is “run then stop” acceptable for non-status changes, or do you want to avoid running at all (e.g. webhook filtering)? |
| **Accepted without assignee** | Current flow can reach “create task” with no assignee; task creation then fails (owner_id null). | When a work request is Accepted but has no assignee: should we **(a)** never create a task and stop silently, **(b)** stop and notify someone (e.g. CC), or **(c)** create task with a default owner? |
| **Due date and start date** | Due date is taken from form fields (e.g. “Deliverable Due Date”); start date is not explicitly set by the automation. Tests reported due date not inherited and start date not matching creation. | How should due date and start date be set on the task? Must they always match the work request? What if the form leaves them blank? |
| **Required fields** | Some request types (e.g. Social Paid) expect “Social Media Account”; when missing, task naming is incomplete. | Which fields are **mandatory** per request type before a task should be created? Are you open to defining a short “intake schema” (mandatory fields, validation) so the automation receives consistent data? |
| **Supporting activity linking** | Supporting-activity tasks are linked back to the work request via an API call. | Is “link supporting-activity tasks to the work request” still the required behaviour? Any change expected? |
| **Emails** | Success/failure emails go to task owner and a fixed CC list (currently Optimizely addresses). | Who should receive success and failure emails (recipients and CC)? Should CC be Swiss Re–only for production? |

### 3.4 Environment and observed failures

| Issue | What we know | What we need from stakeholders |
|-------|----------------|--------------------------------|
| **Automation not triggering (Test)** | In retests, automation often did not run at all in Test (no task, no email). | Is the Test environment configured the same as Production (webhook URL, enabled workflow, CMP config)? Can we align Test and Prod so we can validate behaviour reliably? |
| **Email tool behaviour** | There have been duplicate or repeated send_email calls (e.g. many identical “task created” emails). | Have you seen duplicate or missing emails in Production? Any specific rules (e.g. one email per task, never more than one per run)? |
| **Duplicate tasks on retry** | If the webhook or workflow is retried, supporting-activity step could create duplicate tasks. | Is “create at most once per work request” a requirement? Are retries or duplicate webhooks possible in your setup? |

---

## 4. Targeted questions for the meeting

Use these to gather information and remove ambiguity. Phrase them in your own words; adjust order by priority. Avoid leading toward “how many agents” or “which tools”—focus on **what** should happen and **who** decides.

### 4.1 Current process and workflow

1. Walk us through the **ideal** path from “someone submits a work request” to “the right task(s) exist in CMP and the right people are notified.” Who does what, in what order?
2. In practice, where do things **go wrong** most often (e.g. wrong campaign, wrong owner, missing task, duplicate task)? Can you give one or two concrete examples?
3. When a work request is **modified** (e.g. assignee added, status changed), what **should** happen? Should the automation run only for certain changes (e.g. status → Accepted), or is “run for any change and then decide” acceptable?
4. Who is the **owner** of a task in your process—the person who will do the work (assignee), the person who requested it, or someone else? How should we treat “no assignee” when the request is already Accepted?

### 4.2 Campaign and intake

5. How is the **campaign** chosen on the work request form today (dropdown, free text, other)? Is that list the same as the list of campaigns in CMP, or maintained separately?
6. When someone submits a request **without** selecting a campaign (or selects something that doesn’t exist in CMP), what should happen—create under a single “unspecified” campaign for later reassignment, or not create a task until campaign is clear?
7. Which fields on the work request form are **mandatory** before a task should be created? Are you open to defining and enforcing a minimal set (e.g. assignee, due date, campaign) so the automation gets consistent data?
8. Have you considered (or would you consider) a **single source of truth** for campaign names and IDs (e.g. shared file or list) that both the form and the automation could use, so updates don’t require code changes?

### 4.3 Owner, assignee, and notifications

9. In tests we saw the **task owner** in CMP differ from the person we expected (e.g. work request assignee). Does CMP have a specific way you want the task owner set (e.g. a particular field or API), or could there be permissions/config overriding it?
10. Who should receive **emails** when a task is created successfully vs. when something fails? Should the current CC list stay, or move to Swiss Re–only for production?
11. If the automation **fails** (e.g. campaign not found, task creation error), who should be notified and how? Is “email to CC only when owner is unknown” acceptable?

### 4.4 Supporting activities and templates

12. Is the list of **supporting activity** types (e.g. Emailing, Paid Search, Webpage) fixed for the near term, or will new types be added? Who decides and how do we keep the automation in sync?
13. For each supporting activity, should we always create a task with a **brief** pre-filled from the work request, or only when a specific template exists? What should happen if a supporting activity type has no workflow or template configured?
14. Are the **work request template names** (e.g. “Social Organic”, “Web requests (MARKETING)”) stable? When you add a new template in CMP, who will tell the team so we can add it to the automation?

### 4.5 Success criteria and priorities

15. What does **“working”** mean for you for this automation—e.g. “every accepted work request gets exactly one main task and the right supporting tasks, under the right campaign, with the right owner and one email”? Any other must-haves?
16. For **March/April demos or go-live**, which scenarios must work reliably (e.g. Social Organic only, or Social + Web + one supporting activity)? We can prioritise those in testing and configuration.
17. How do you want to handle **edge cases** (e.g. unknown campaign, unknown template, missing assignee)—explicit failure with a clear message, or fallback behaviour (e.g. default campaign) and fix later?

### 4.6 Environment and operations

18. In **Test**, we’ve seen the automation often not trigger at all. Can you confirm how Test is set up (webhook, workflow enabled, CMP config) and whether it should mirror Production?
19. Do you ever see **duplicate** tasks or **duplicate** emails in Production? If so, in what situations?
20. Is there a requirement that the same work request should **never** get duplicate supporting-activity tasks (e.g. if the webhook or workflow runs twice)?

---

## 5. After the meeting

- **Document answers** in a simple table (e.g. question ID, answer, decision, action owner).
- **Update** any assumptions in this doc and in `docs/client_questions_demo.md` so the next round (e.g. V2 behaviour, test plan) stays aligned with stakeholder intent.
- **Flag** any new gaps or follow-up questions for a short follow-up or async clarification so the team can propose “how the system should function” with minimal ambiguity.

---

## References

- **Current behaviour (original):** `analysis/behavior_contract.md`
- **Risks and mitigations:** `analysis/risk_register.md`
- **Refactor scope:** `analysis/refactor_scope.md`
- **Existing client Qs and answer log:** `docs/client_questions_demo.md`
- **Test findings:** `cmp_testing/task_creation_test.md`, `cmp_testing/task_creation_retest.md`
- **Stakeholder context and expectations:** `notes/plan_ahead.md`

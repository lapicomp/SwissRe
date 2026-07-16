Task Creation Agent – Clear Technical Report
System
Optimizely CMP + Opal Automation
Objective
Automatically generate tasks from accepted Work Requests and structure them correctly inside CMP campaigns.

1. Full Automation Flow (Expected)
The automation should run in the following order.
Work Request Submitted
        ↓
Work Request Accepted
        ↓
Opal Task Creation Agent Trigger
        ↓
Create Main Task
        ↓
Populate fields (Campaign, Due Date, Owner, etc)
        ↓
Optional Event Logic
        ↓
Create Milestone
        ↓
Create Supporting Activity Tasks
        ↓
Populate Briefs with Work Request data
        ↓
Send Notification

2. Detailed Step Logic
Step 1 — Initiate Task Creation
Trigger:
Automation starts when:
    • Work Request Status = ACCEPTED
    • AND Request Type = defined automation type
Input:
Work Request object
Output:
Opal opens Create Task modal
Expected Result:
    • Task creation process initiated
    • Parent campaign identified

Step 2 — Fill Campaign
Logic:
The task must inherit the campaign from the Work Request.

Task.Campaign = WorkRequest.Campaign
Input:
    → Work Request → Campaign custom field
Output:
    → Task is created under the correct campaign
Failure observed in tests:
Task created in wrong campaign:
    → Expected: [ES CMP] Test Campaign
    → Actual:   [ES-CMP] Test Opal Usecase Campaign

Root cause likely:
    • campaign hardcoded
    • wrong field reference

Step 3 — Fill Start & Due Date
Logic:
    • Task.DueDate = 
    • WorkRequest.DeliverableDueDate
    • OR
    • WorkRequest.PublishingDueDate
Input:
    • Deliverable Due Date
    • Publishing Due Date
Output
    • Correct due date in task.
    • Make sure start date is per the workflow creation date and due date is deliverable due date. 
Risk
If fields are optional and empty → task may be created without due date. 

Step 4 — Create Main Task
Logic:
Create a task and assign ownership.
    • Task.Owner = WorkRequest.Assignee
Expected Result:
Main Task created
    • Task owner = Work Request assignee
Failure observed
Owner incorrectly set.

Meaning system likely uses:
    • Default owner instead of:
        ○ WorkRequest.Assignee

Step 5 — Event Requests (Conditional Logic)
Only runs if:
    • RequestType = Event
Logic:
Add Event Manager as task owner in the Task Brief → Key People section.
    • TaskBrief.KeyPeople.EventManager = EventManagerField
Output:
Event Manager appears in Task Brief.


Step 6 — Edit Brief
Logic:
System opens the Task Brief.
Add additional information:
    • WorkRequest.AdditionalInformation
Save brief.
Output:
Brief saved for the first time.
This step is important because CMP often requires the first brief save before other automation steps work.

Step 7 — Create Milestone
Logic
Create milestone inside campaign.
    • Milestone.Campaign = Task.Campaign
    • Milestone.Name = WorkRequest.Name
Then attach task:
    • Task → Milestone
Output:
    • Milestone created
    • Task linked to milestone

Step 8 — Create Supporting Activity Tasks
Input fields:
    • Supporting Activities
    • Additional Information
    • Mapping XLS
Logic:
Automation reads:
    → Supporting Activities field
Example value:
    → 5 Social Organic posts
    → 1 Webpage
    → 3 Emails
Then system must:
    → Create tasks based on count
Example:
    → Create 5 Social Organic tasks
    → Create 1 Webpage task
    → Create 3 Email tasks
Each task must use mapping rules from:
    → Mapping XLS
Mapping includes:
    → Task title
    → Workflow
    → Brief template
    → Naming convention
Output:
Multiple tasks created with correct workflows.


Step 9 — Add Work Request Inputs into Briefs
Logic:
Extract Work Request information:
    → Request Name
    → Description
    → Topic
    → Country
    → Due Date
    → Additional Information
Add them into:
    → TaskBrief.Description
Important rule:
This applies only to:
    → Supporting activity tasks
    → NOT the main task.



3. Expected Final System State
When the automation finishes successfully:

Campaign
   └ Milestone
        ├ Main Task (from request)
        ├ Supporting Task 1
        ├ Supporting Task 2
        ├ Supporting Task 3
        └ etc

Each supporting task contains:
    → Correct workflow
    → Correct title
    → Correct brief template
    → Description populated from Work Request

4. Failures Observed in Testing
Wrong campaign mapping
Symptoms:
    - Task created under wrong campaign
Likely Cause:
    - Campaign field not read from Work Request
Fix:
Task.Campaign = WorkRequest.Campaign

Owner mapping incorrect
Symptoms:
    - Task owner ≠ Work Request assignee
Likely Cause:
    - Default owner override
Fix:
Task.Owner = WorkRequest.Assignee


Task created but not linked to correct campaign context
Symptoms:
    - Task exists but not visible under expected campaign
Likely Cause:
    - Task created globally
Fix:
Ensure: ParentCampaignID assigned correctly

5. Most Likely Root Cause Pattern
Based on the failures:
The automation likely breaks very early in the flow.
Meaning the problem is probably:
    - Trigger configuration
    - OR
    - Campaign field mapping
Because when the main task fails, all downstream steps fail too.

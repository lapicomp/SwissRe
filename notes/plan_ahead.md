• Goal of the Agent: To automate the creation of tasks and related activities in CMP based on information extracted from a submitted work request.
• Trigger: The workflow starts when a work request is accepted in CMP.
• Current Status: The core architecture is largely functional (95% working). Recent efforts focused on optimizing prompt templates, adding guardrails, fallbacks, and edge case handling to minimize token usage and improve production readiness.


Key Discussion Points & Challenges:
	• Campaign Assignment (Critical Pain Point):
		○ The agent struggles to correctly assign tasks to campaigns due to the current CMP setup where work request forms use a manual dropdown list for campaigns, not a direct pull from existing campaigns.
		○ This leads to significant manual work and errors for the marketing services team.
		○ Technical Explanation (Nicola/Staffan): The CMP standard workflow assumes campaign assignment happens after work request submission, and the form is designed for a generic audience who might not know the correct campaign. The current dropdown is a custom field, not linked to actual CMP campaigns.
		○ Nadzeya's Stance: This problem must be resolved. Open to any solution (e.g., mapping by name, external file for agent reference) that reduces manual intervention, even if it involves rethinking current processes.
	• Fallback for Unassigned Campaigns: If no campaign is chosen, a fallback to an "undefined" or "others" campaign would be acceptable, allowing for later reassignment.
	• Managing Campaign List: Discussion around using an external source (e.g., Excel file) that the agent could reference for campaign names and links, allowing for easier updates without code changes.
	• Testing Issues (Demo Observations):
		○ During the demo, the agent incorrectly assigned "choose" as a business unit and pulled in data (e.g., LinkedIn reinsurance solutions, legal sign-off) that was not explicitly provided in the work request.
		○ This highlights the need for more detailed testing with real-world scenarios and data.
	• Email Tool Bug: A known bug with the "send an email" tool persists despite guardrails.

• Critical Challenge 1: Campaign Assignment: A major pain point is the agent’s inability to correctly assign tasks to the right campaign from a dropdown menu. The agent sees the campaign as a string value and fails to select the correct option. This is a high-priority issue, as manual campaign assignment is a primary source of errors and manual work. 
• Critical Challenge 2: Creating Multiple Tasks: A key requirement is for the agent to create multiple sub-tasks when a work request includes “supporting activities” (e.g., social media, email), each with the correct brief template. Testing is needed to verify this core functionality. 
• Proposed Solutions and Workflow Logic: 
		○ To manage campaign lists, a pragmatic solution is to use a central, fetchable file (e.g., an Excel sheet) that the agent can reference for campaign names and links. This is considered a straightforward task. 
		○ The standard CMP workflow assumes campaigns are assigned after a request is approved. A potential fallback for when a campaign isn’t specified in a request could be to assign the task to an “undefined” campaign for later reassignment. 
• Arrangements:
	• Discuss the task creation agent's required behavior and conduct thorough testing 
	• Create a process map to clarify the input and output for each step of the agent's workflow to help with testing and validation. 
	• Investigate and find a solution for the agent to correctly assign tasks to the specified campaign
	• Test the agent's ability to create multiple tasks when supporting activities are included in a work request
	• Consider adding a fallback campaign (e.g., "undefined") for requests where a campaign is not specified. 
	highlights missing contextual inputs (requester identity, situational context) needed for AI workflows; proposes re-engineering parts of the work request to reduce friction and improve performance. 

Intake management friction and unclear intake/output definitions are leading to workflow failure. The intake process is a major source of friction, with unclear data requirements and inconsistent practices across teams. Current situation: Workflow mapping exists but lacks the clarity needed for AI agents to process requests consistently; work requests may include incomplete fields (e.g., zero attendees), causing downstream issues. Impact: Delays, rework, and inability to reliably automate; challenges in moving from pilot to production; hindered ability to showcase tangible outcomes at upcoming events. Quantitative metrics: Not specified; however, leadership expects measurable reductions in steps, clicks, and time from “today versus an agent.” Examples: “Zero number of attendees” entered during demos; intake starting at the wrong origin point; need to “retake and rethink” the intake structure. Context: Newly consolidated division (two to nine months old) is still harmonizing processes; desire to demonstrate production results by March and April events. Stakeholders: Requesters, marketing operations teams, Nadja (process owner), leadership. 
CMP is used primarily as a planning tool, and tool steps add friction; the editorial calendar and asset library are not optimized. CMP is being used mainly for planning, not as a fully streamlined workflow engine, and its steps add friction rather than remove it. Current situation: The editorial calendar feels non-typical; asset library practices need review; Seismic integration exists but needs revisiting; CMP onboarding requires more than configuration—it needs workflow optimization and human enablement. Impact: Lower user adoption, inefficiencies, and missed opportunities for oversight and automation; difficulty proving value.
Difficulty proving ROI and the impact of AI agents to leadership in real time. Leadership requires clear, quantifiable benefits beyond credit consumption. Current situation: No standardized mechanism to report agent runs, task success, or estimated hours saved; pilots risk being presented as cost centers rather than value generators

Expectations 
Deliver a small, production-ready AI-enabled workflow improvement before

Redesign the intake request structure to capture essential context and eliminate friction
Implement a minimal viable production feature 
Set up agent instrumentation for run logging, credit usage, and estimated hours saved; stakeholders: AI architects, analytics; deadline: aligned with the first production deploymen
Review and optimize CMP editorial calendar and asset library usage; stakeholder: CMP users/ops;
Validate and adjust Seismic native integration with CMP; stakeholder: integration owners; deadline: post-initial production
Coordinate with procurement to include Comprend in a small RFP/contract;
Prepare event-ready demos and leadership reports showcasing individual and company-level impact for March 

he AI has identified the client’s biggest pain point as context-poor intake and workflow definitions causing AI agents to perform irrelevant actions, high credit consumption, and unreliable outputs. Here are some possible solutions for your consideration: 
1. Conduct a rapid “intake clarity sprint” with Nadja to define mandatory fields, context templates, and output schemas; convert these into agent-readable instructions and validation rules. 
2. Implement context scoping and retrieval: tag and prioritize only relevant company instructions and workflow artifacts; use a context index so agents fetch “just enough” context rather than broad, generic data. 
3. Add pre-processing and guardrails: build an intake validator agent that checks requests (e.g., attendee counts, dates, objectives) and blocks or prompts for missing essentials before downstream workflows run. 
4. Instrument agents for ROI: log each run, success state, and estimated hours saved; pair with credit usage to show cost-per-outcome trends and produce leadership-ready dashboards. 
5. Deliver a simple production artifact (Teams smart card linked to CMP) to demonstrate immediate value, gather feedback, and iteratively harden the 


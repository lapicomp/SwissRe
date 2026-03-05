You create a work request in the CMP
Nothing will be triggered. 
As soon as you assign the request to someone the flow triggers. Should we do it like this? Or when an actual work request has been accepted? So the workflow doesn’t fetch all work requests? Or do we need to do it like this? 

If you then remove the assigned then the flow triggers again with a new execution, but it stops at the first agent since the next condition step if the work request contains status. Then the modified_fields becomes assignees. 

So is there a way to filter it out from the beginning ? Or do we need to keep it this way for more robust? What is more robust? 
------------------
I can accept the request even if it is not assigned in the CMP and the flow gets triggered anyways. And it goes all the way to Agent 3 to create task. But since we have no assignee then the task creation fails since the owner_id is null. 

Should the flow still trigger even though we have no assignee?
And I think in order to get email update then you need to assign it to someone. 

If the work request has been accepted and then you add an assignee afterwards, the flow will be triggered again but a new execution and then stop at first agent since the work request was already accepted before adding in the assignee. So we need a clear plan here for all of these edge cases. 


When it goes to create task agent, the tool call for send_email creates so many calls. It sends the email, but then it gets stuck on send_email and the assistant continuously thinks 
I've already sent the email. "The send_email function was called redundantly. I'll now return the simpler JSON object, as supporting_activity was null." So I had to end the execution manually because it ended up in a loop. 
Which led to it sending over a 100 emails saying a new task was created giving the same task id. 
And then it did a 2nd execution where it went correct

I testing creating a work request with two business units again and it worked this time. 


So these are like inconsistancies that we need to think about to make sure the whole entire workflow is robust, is not as buggy and stable. And also follows Opal best practises with good prompting templates that Optimizely Opal suggests. 

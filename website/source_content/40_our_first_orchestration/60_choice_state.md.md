+++
title = "Introducing the Choice State type"
chapter = false
weight = 60
+++

A Choice state adds branching logic to a state machine. You can think of this like a *switch *statement common in many programming languages. A Choice state has an array of rules.  Each rule contains two things: an expression that evaluates some boolean expression, and a reference to the next state to transition to if this rule matches successfully. All of the rules are evaluated in order and the first rule to match successfully causes the state machine to transition to the next state defined by the rule.

In our example workflow, we want to wait for a human to review an application if either the name or address check comes back as flagged. Otherwise, we want to automatically approve the application.  Let’s add in a Choice state that implements this flow.

Here is what our updated flow will look like after we're done with this step:
![Adding review required check](/images/workflow-add-review-required-sm.png)

### In this step, we will

* Add a new placeholder state called 'Pending Review'

* Add a ‘Review Required?’ state that uses the Choice state type to transition to the Pending Review state if either the name or the address checks return with a flag

* Update the Check Address step to transition to the ‘Review Required?’ state

### Make these changes

➡️ Step 1. Replace `serverless.yml` with ___CLIPBOARD_BUTTON def5ea473552142257ef1b5a047ba98dd01749c2:serverless.yml|

➡️ Step 2. Run:

```bash
sls deploy
```

We just added two new states to our workflow: ‘Review Required?’ and Pending Review.  The ‘Review Required?’ state examines its input (which is the output from the Check Address state) and runs through a series of checks. You can see that there’s an array of two choice rules in the state’s definition, each of which specifies what state name to go to next if its rule matches successfully. There is also a default state name specified to transition to in the event of no rule matches.  

One of our Choices rules says that if the value inside the input located at `checks.name.flagged` is true, then the next state should be Pending Review. The other choice rule expresses something similar, except it looks at `checks.address.flagged` to see if its true, in which case it also transitions to the Pending Review state. Finally, our choice state’s default value indicates that if none of our choice rules match, the state machine should transition to the Approve Application state.

{{% notice tip %}}
For a deeper discussion on the behavior and types of comparisons supported by the Choice state, see our developer guide https://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-choice-state.html
{{% /notice %}}


### Try it out

Let’s try some executions to see our Choice state in action:

➡️ Step 1. Back in the Step Functions web console, click ‘New execution’

➡️ Step 2. Try a valid application by pasting this as input:

`{ "application": { "name": "Spock", "address": "123 Enterprise Street" } }`

➡️ Step 3. Click ‘Start execution’. 

    Notice how the ‘Review Required?’ state transitions to the Approve Application state. That’s because our name and our address both contained valid values.  

➡️ Step 4. Try another execution with this invalid application (flagged for an evil name):

`{ "application": { "name": "evil Spock", "address": "123 Enterprise Street" } }`

Notice how this time, because we passed in a troublesome name (remember, our name checking logic will flag anything with the string ‘evil’ in the name), our workflow routes to the Pending Review State.

➡️ Step 5. Finally, for the sake of completeness, let’s do one more execution with this invalid address:

`{ "application": { "name": "Spock", "address": "Somewhere" } }`
   
Once again, notice how we route to the Pending Review state gain, this time because we passed in a troublesome address (our address checking logic will flag anything that does not match the number(s)-space-letter(s) pattern)


Thanks to the Choice state, we are now routing our workflow the way we want. But, we still have placeholder Pass states for our Approve Application and Pending Review steps. We’ll hold off on implementing the Approve Application step until later in the workshop (since we already know how to integrate with a Lambda function call from a step function). Instead, we’ll keep our learning momentum going and learn how to implement our Pending Review state. 

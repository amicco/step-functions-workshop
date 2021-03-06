+++
title = "Pausing an execution and waiting for an external callback"
chapter = false
weight = 20
+++

Step Functions does its work by integrating with various AWS services directly, and you can control these AWS services using three different service integration patterns: 

* Call a service and let Step Functions progress to the next state immediately after it gets an HTTP response. 

    You’ve already seen this integration type in action. It’s what we’re using to call the Data Checking Lambda function and get back a response.
    
* Call a service and have Step Functions wait for a job to complete. 
    
    This is most commonly used for triggering batch style workloads, pausing, then resuming execution after the job completes. We won’t use this style of service integration in this workshop.
    
* Call a service with a task token and have Step Functions wait until that token is returned along with a payload.
    
    This is the integration pattern we want to use here, since we want to make a service call, and then wait for an asynchronous callback to arrive sometime in the future, and then resume execution.


Callback tasks provide a way to pause a workflow until a task token is returned. A task might need to wait for a human approval, integrate with a third party, or call legacy systems. For tasks like these, you can pause a Step Function execution and wait for an external process or workflow to complete.

In these situations, you can instruct a Task state to generate a unique task token (a unique ID that references a specific Task state in a specific execution), invoke your desired AWS service call, and then pause execution until the Step Functions  service receives that task token back via an API call from some other process.

We’ll need to make a few updates to our workflow in order for this to work. 

### In this step, we will

* Make our Pending Review state invoke our Account Applications Lambda function using a slightly different Task state definition syntax which includes a `.waitForTaskToken` suffix. This will generate a task token which we can pass on to the Account Applications service along with the application ID that we want to flag for review. 

* Make the Account Applications Lambda function update the application record to mark it as pending review and storing the taskToken alongside the record. Then, once a human reviews the pending application, the Account Applications service will make a callback to the Step Functions API, calling the SendTaskSuccesss endpoint, passing back the task token along with the relevant output from this step which, in our case, will be data to indicate if the human approved or rejected the application. This information will let the state machine decide what to do next based on what decision the human made.

* Add another Choice state called 'Review Approved?' that will examine the output from the Pending Review state and transition to the Approve Application state or a Reject Application state (which we’ll also add now).



### Make these changes

➡️ Step 1. Replace `account-applications/flag.js` with <span class="clipBtn clipboard" data-clipboard-target="#id278b0babefb143aafbbf1bb5c773a62fcd3f374faccountapplicationsflagjs">this content</span> (click the gray button to copy to clipboard). 
{{< expand "Click to view diff" >}} {{< safehtml >}}
<div id="diff-id278b0babefb143aafbbf1bb5c773a62fcd3f374faccountapplicationsflagjs"></div> <script type="text/template" data-diff-for="diff-id278b0babefb143aafbbf1bb5c773a62fcd3f374faccountapplicationsflagjs">commit 278b0babefb143aafbbf1bb5c773a62fcd3f374f
Author: Gabe Hollombe <gabe@avantbard.com>
Date:   Wed Oct 16 10:58:50 2019 +0800

    Call out to Lambda from Pending Review state, add Review Approved? choice state that transitions to Approve or Reject pass states. Create a review lambda that calls back to Step Functions with review decision in SendTaskSuccess

diff --git a/account-applications/flag.js b/account-applications/flag.js
index 3e700d5..8bbdcb1 100644
--- a/account-applications/flag.js
+++ b/account-applications/flag.js
@@ -10,7 +10,7 @@ const dynamo = new AWS.DynamoDB.DocumentClient();
 const AccountApplications = require('./AccountApplications')(ACCOUNTS_TABLE_NAME, dynamo)
 
 const flagForReview = async (data) => {
-    const { id, flagType } = data
+    const { id, flagType, taskToken } = data
 
     if (flagType !== 'REVIEW' && flagType !== 'UNPROCESSABLE_DATA') {
         throw new Error("flagType must be REVIEW or UNPROCESSABLE_DATA")
@@ -32,6 +32,7 @@ const flagForReview = async (data) => {
         {
             state: newState,
             reason,
+            taskToken
         }
     )
     return updatedApplication
</script>
{{< /safehtml >}} {{< /expand >}}
{{< safehtml >}}
<textarea id="id278b0babefb143aafbbf1bb5c773a62fcd3f374faccountapplicationsflagjs" style="position: relative; left: -1000px; width: 1px; height: 1px;">'use strict';
const REGION = process.env.REGION
const ACCOUNTS_TABLE_NAME = process.env.ACCOUNTS_TABLE_NAME

const AWS = require('aws-sdk')
AWS.config.update({region: REGION});

const dynamo = new AWS.DynamoDB.DocumentClient();

const AccountApplications = require('./AccountApplications')(ACCOUNTS_TABLE_NAME, dynamo)

const flagForReview = async (data) => {
    const { id, flagType, taskToken } = data

    if (flagType !== 'REVIEW' && flagType !== 'UNPROCESSABLE_DATA') {
        throw new Error("flagType must be REVIEW or UNPROCESSABLE_DATA")
    }

    let newState
    let reason
    if (flagType === 'REVIEW') {
        newState = 'FLAGGED_FOR_REVIEW'
        reason = data.reason
    }
    else {
        reason = JSON.parse(data.errorInfo.Cause).errorMessage
        newState = 'FLAGGED_WITH_UNPROCESSABLE_DATA'
    }

    const updatedApplication = await AccountApplications.update(
        id,
        {
            state: newState,
            reason,
            taskToken
        }
    )
    return updatedApplication
}

module.exports.handler = async(event) => {
    try {
        const result = await flagForReview(event)
        return result
    } catch (ex) {
        console.error(ex)
        console.info('event', JSON.stringify(event))
        throw ex
    }
};
</textarea>
{{< /safehtml >}}

➡️ Step 2. Create `account-applications/review.js` with <span class="clipBtn clipboard" data-clipboard-target="#id278b0babefb143aafbbf1bb5c773a62fcd3f374faccountapplicationsreviewjs">this content</span> (click the gray button to copy to clipboard). 
{{< expand "Click to view diff" >}} {{< safehtml >}}
<div id="diff-id278b0babefb143aafbbf1bb5c773a62fcd3f374faccountapplicationsreviewjs"></div> <script type="text/template" data-diff-for="diff-id278b0babefb143aafbbf1bb5c773a62fcd3f374faccountapplicationsreviewjs">commit 278b0babefb143aafbbf1bb5c773a62fcd3f374f
Author: Gabe Hollombe <gabe@avantbard.com>
Date:   Wed Oct 16 10:58:50 2019 +0800

    Call out to Lambda from Pending Review state, add Review Approved? choice state that transitions to Approve or Reject pass states. Create a review lambda that calls back to Step Functions with review decision in SendTaskSuccess

diff --git a/account-applications/review.js b/account-applications/review.js
new file mode 100644
index 0000000..74b3186
--- /dev/null
+++ b/account-applications/review.js
@@ -0,0 +1,47 @@
+'use strict';
+const REGION = process.env.REGION
+const ACCOUNTS_TABLE_NAME = process.env.ACCOUNTS_TABLE_NAME
+
+const AWS = require('aws-sdk')
+AWS.config.update({region: REGION});
+
+const dynamo = new AWS.DynamoDB.DocumentClient();
+const stepfunctions = new AWS.StepFunctions();
+
+const AccountApplications = require('./AccountApplications')(ACCOUNTS_TABLE_NAME, dynamo)
+
+const updateApplicationWithDecision = (id, decision) => {
+    if (decision !== 'APPROVE' && decision !== 'REJECT') {
+        throw new Error("Required `decision` parameter must be 'APPROVE' or 'REJECT'")
+    }
+
+    switch(decision) {
+        case 'APPROVE': return AccountApplications.update(id, { state: 'REVIEW_APPROVED' })
+        case 'REJECT': return AccountApplications.update(id, { state: 'REVIEW_REJECTED' })
+    }
+}
+
+const updateWorkflowWithReviewDecision = async (data) => {
+    const { id, decision } = data
+
+    const updatedApplication = await updateApplicationWithDecision(id, decision)
+
+    let params = {
+        output: JSON.stringify({ decision }),
+        taskToken: updatedApplication.taskToken
+    };
+    await stepfunctions.sendTaskSuccess(params).promise()
+
+    return updatedApplication
+}
+
+module.exports.handler = async(event) => {
+    try {
+        const result = await updateWorkflowWithReviewDecision(event)
+        return result
+    } catch (ex) {
+        console.error(ex)
+        console.info('event', JSON.stringify(event))
+        throw ex
+    }
+};
\ No newline at end of file
</script>
{{< /safehtml >}} {{< /expand >}}
{{< safehtml >}}
<textarea id="id278b0babefb143aafbbf1bb5c773a62fcd3f374faccountapplicationsreviewjs" style="position: relative; left: -1000px; width: 1px; height: 1px;">'use strict';
const REGION = process.env.REGION
const ACCOUNTS_TABLE_NAME = process.env.ACCOUNTS_TABLE_NAME

const AWS = require('aws-sdk')
AWS.config.update({region: REGION});

const dynamo = new AWS.DynamoDB.DocumentClient();
const stepfunctions = new AWS.StepFunctions();

const AccountApplications = require('./AccountApplications')(ACCOUNTS_TABLE_NAME, dynamo)

const updateApplicationWithDecision = (id, decision) => {
    if (decision !== 'APPROVE' && decision !== 'REJECT') {
        throw new Error("Required `decision` parameter must be 'APPROVE' or 'REJECT'")
    }

    switch(decision) {
        case 'APPROVE': return AccountApplications.update(id, { state: 'REVIEW_APPROVED' })
        case 'REJECT': return AccountApplications.update(id, { state: 'REVIEW_REJECTED' })
    }
}

const updateWorkflowWithReviewDecision = async (data) => {
    const { id, decision } = data

    const updatedApplication = await updateApplicationWithDecision(id, decision)

    let params = {
        output: JSON.stringify({ decision }),
        taskToken: updatedApplication.taskToken
    };
    await stepfunctions.sendTaskSuccess(params).promise()

    return updatedApplication
}

module.exports.handler = async(event) => {
    try {
        const result = await updateWorkflowWithReviewDecision(event)
        return result
    } catch (ex) {
        console.error(ex)
        console.info('event', JSON.stringify(event))
        throw ex
    }
};
</textarea>
{{< /safehtml >}}

➡️ Step 3. Replace `serverless.yml` with <span class="clipBtn clipboard" data-clipboard-target="#id278b0babefb143aafbbf1bb5c773a62fcd3f374fserverlessyml">this content</span> (click the gray button to copy to clipboard). 
{{< expand "Click to view diff" >}} {{< safehtml >}}
<div id="diff-id278b0babefb143aafbbf1bb5c773a62fcd3f374fserverlessyml"></div> <script type="text/template" data-diff-for="diff-id278b0babefb143aafbbf1bb5c773a62fcd3f374fserverlessyml">commit 278b0babefb143aafbbf1bb5c773a62fcd3f374f
Author: Gabe Hollombe <gabe@avantbard.com>
Date:   Wed Oct 16 10:58:50 2019 +0800

    Call out to Lambda from Pending Review state, add Review Approved? choice state that transitions to Approve or Reject pass states. Create a review lambda that calls back to Step Functions with review decision in SendTaskSuccess

diff --git a/serverless.yml b/serverless.yml
index eec141d..acc14c6 100644
--- a/serverless.yml
+++ b/serverless.yml
@@ -30,6 +30,14 @@ functions:
       ACCOUNTS_TABLE_NAME: ${self:custom.applicationsTable}
     role: FlagRole
 
+  ReviewApplication:
+    name: ${self:service}__account_applications__review__${self:provider.stage}
+    handler: account-applications/review.handler
+    environment:
+      REGION: ${self:provider.region}
+      ACCOUNTS_TABLE_NAME: ${self:custom.applicationsTable}
+    role: ReviewRole
+
   FindApplications:
     name: ${self:service}__account_applications__find__${self:provider.stage}
     handler: account-applications/find.handler
@@ -144,6 +152,22 @@ resources:
           - { Ref: LambdaLoggingPolicy }
           - { Ref: DynamoPolicy }
 
+    ReviewRole:
+      Type: AWS::IAM::Role
+      Properties:
+        AssumeRolePolicyDocument:
+          Version: '2012-10-17'
+          Statement:
+            - Effect: Allow
+              Principal:
+                Service:
+                  - lambda.amazonaws.com
+              Action: sts:AssumeRole
+        ManagedPolicyArns:
+          - { Ref: LambdaLoggingPolicy }
+          - { Ref: DynamoPolicy }
+          - { Ref: StepFunctionsPolicy }
+
     RejectRole:
       Type: AWS::IAM::Role
       Properties:
@@ -250,6 +274,7 @@ resources:
                     Action: 'lambda:InvokeFunction'
                     Resource:
                         - Fn::GetAtt: [DataCheckingLambdaFunction, Arn]
+                        - Fn::GetAtt: [FlagApplicationLambdaFunction, Arn]
 
     ProcessApplicationsStateMachine:
       Type: AWS::StepFunctions::StateMachine
@@ -299,8 +324,36 @@ resources:
                         "Default": "Approve Application"
                     },
                     "Pending Review": {
-                        "Type": "Pass",
-                        "End": true
+                      "Type": "Task",
+                      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
+                      "Parameters": {
+                          "FunctionName": "#{flagApplicationLambdaName}",
+                          "Payload": {
+                              "id.$": "$.application.id",
+                              "flagType": "REVIEW",
+                              "taskToken.$": "$$.Task.Token"
+                          }
+                      },
+                      "ResultPath": "$.review",
+                      "Next": "Review Approved?"
+                    },
+                    "Review Approved?": {
+                        "Type": "Choice",
+                        "Choices": [{
+                                "Variable": "$.review.decision",
+                                "StringEquals": "APPROVE",
+                                "Next": "Approve Application"
+                            },
+                            {
+                                "Variable": "$.review.decision",
+                                "StringEquals": "REJECT",
+                                "Next": "Reject Application"
+                            }
+                        ]
+                    },
+                    "Reject Application": {
+                         "Type": "Pass",
+                         "End": true
                      },
                     "Approve Application": {
                         "Type": "Pass",
@@ -310,4 +363,5 @@ resources:
               }
             - {
               dataCheckingLambdaArn: !GetAtt [DataCheckingLambdaFunction, Arn],
+              flagApplicationLambdaName: !Ref FlagApplicationLambdaFunction,
             }
\ No newline at end of file
</script>
{{< /safehtml >}} {{< /expand >}}
{{< safehtml >}}
<textarea id="id278b0babefb143aafbbf1bb5c773a62fcd3f374fserverlessyml" style="position: relative; left: -1000px; width: 1px; height: 1px;">service: StepFunctionsWorkshop

plugins:
  - serverless-cf-vars

custom:
  applicationsTable: '${self:service}__account_applications__${self:provider.stage}'

provider:
  name: aws
  runtime: nodejs10.x
  memorySize: 128
  stage: dev

functions:
  SubmitApplication:
    name: ${self:service}__account_applications__submit__${self:provider.stage}
    handler: account-applications/submit.handler
    environment:
      REGION: ${self:provider.region}
      ACCOUNTS_TABLE_NAME: ${self:custom.applicationsTable}
      APPLICATION_PROCESSING_STEP_FUNCTION_ARN: { Ref: "ProcessApplicationsStateMachine" }
    role: SubmitRole

  FlagApplication:
    name: ${self:service}__account_applications__flag__${self:provider.stage}
    handler: account-applications/flag.handler
    environment:
      REGION: ${self:provider.region}
      ACCOUNTS_TABLE_NAME: ${self:custom.applicationsTable}
    role: FlagRole

  ReviewApplication:
    name: ${self:service}__account_applications__review__${self:provider.stage}
    handler: account-applications/review.handler
    environment:
      REGION: ${self:provider.region}
      ACCOUNTS_TABLE_NAME: ${self:custom.applicationsTable}
    role: ReviewRole

  FindApplications:
    name: ${self:service}__account_applications__find__${self:provider.stage}
    handler: account-applications/find.handler
    environment:
      REGION: ${self:provider.region}
      ACCOUNTS_TABLE_NAME: ${self:custom.applicationsTable}
    role: FindRole

  RejectApplication:
    name: ${self:service}__account_applications__reject__${self:provider.stage}
    handler: account-applications/reject.handler
    environment:
      REGION: ${self:provider.region}
      ACCOUNTS_TABLE_NAME: ${self:custom.applicationsTable}
    role: RejectRole

  ApproveApplication:
    name: ${self:service}__account_applications__approve__${self:provider.stage}
    handler: account-applications/approve.handler
    environment:
      REGION: ${self:provider.region}
      ACCOUNTS_TABLE_NAME: ${self:custom.applicationsTable}
    role: ApproveRole

  DataChecking:
    name: ${self:service}__data_checking__${self:provider.stage}
    handler: data-checking.handler
    role: DataCheckingRole

resources:
  Resources:
    LambdaLoggingPolicy:
      Type: 'AWS::IAM::ManagedPolicy'
      Properties:
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - logs:CreateLogGroup
                - logs:CreateLogStream
                - logs:PutLogEvents
              Resource:
                - 'Fn::Join':
                  - ':'
                  -
                    - 'arn:aws:logs'
                    - Ref: 'AWS::Region'
                    - Ref: 'AWS::AccountId'
                    - 'log-group:/aws/lambda/*:*:*'

    DynamoPolicy:
      Type: 'AWS::IAM::ManagedPolicy'
      Properties:
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: "Allow"
              Action:
                - "dynamodb:*"
              Resource:
                - { "Fn::GetAtt": ["ApplicationsDynamoDBTable", "Arn" ] }
                - 'Fn::Join':
                    - '/'
                    -
                        - { "Fn::GetAtt": ["ApplicationsDynamoDBTable", "Arn" ] }
                        - '*'

    StepFunctionsPolicy:
      Type: 'AWS::IAM::ManagedPolicy'
      Properties:
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            -
              Effect: "Allow"
              Action:
                - "states:StartExecution"
                - "states:SendTaskSuccess"
                - "states:SendTaskFailure"
              Resource:
                - { Ref: ProcessApplicationsStateMachine }

    SubmitRole:
      Type: AWS::IAM::Role
      Properties:
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - lambda.amazonaws.com
              Action: sts:AssumeRole
        ManagedPolicyArns:
          - { Ref: LambdaLoggingPolicy }
          - { Ref: DynamoPolicy }
          - { Ref: StepFunctionsPolicy }

    FlagRole:
      Type: AWS::IAM::Role
      Properties:
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - lambda.amazonaws.com
              Action: sts:AssumeRole
        ManagedPolicyArns:
          - { Ref: LambdaLoggingPolicy }
          - { Ref: DynamoPolicy }

    ReviewRole:
      Type: AWS::IAM::Role
      Properties:
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - lambda.amazonaws.com
              Action: sts:AssumeRole
        ManagedPolicyArns:
          - { Ref: LambdaLoggingPolicy }
          - { Ref: DynamoPolicy }
          - { Ref: StepFunctionsPolicy }

    RejectRole:
      Type: AWS::IAM::Role
      Properties:
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - lambda.amazonaws.com
              Action: sts:AssumeRole
        ManagedPolicyArns:
          - { Ref: LambdaLoggingPolicy }
          - { Ref: DynamoPolicy }

    ApproveRole:
      Type: AWS::IAM::Role
      Properties:
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - lambda.amazonaws.com
              Action: sts:AssumeRole
        ManagedPolicyArns:
          - { Ref: LambdaLoggingPolicy }
          - { Ref: DynamoPolicy }

    FindRole:
      Type: AWS::IAM::Role
      Properties:
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - lambda.amazonaws.com
              Action: sts:AssumeRole
        ManagedPolicyArns:
          - { Ref: LambdaLoggingPolicy }
          - { Ref: DynamoPolicy }

    DataCheckingRole:
      Type: AWS::IAM::Role
      Properties:
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - lambda.amazonaws.com
              Action: sts:AssumeRole
        ManagedPolicyArns:
          - { Ref: LambdaLoggingPolicy }

    ApplicationsDynamoDBTable:
      Type: 'AWS::DynamoDB::Table'
      Properties:
        TableName: ${self:custom.applicationsTable}
        AttributeDefinitions:
          -
            AttributeName: id
            AttributeType: S
          -
            AttributeName: state
            AttributeType: S
        KeySchema:
          -
            AttributeName: id
            KeyType: HASH
        BillingMode: PAY_PER_REQUEST
        GlobalSecondaryIndexes:
            -
                IndexName: state
                KeySchema:
                    -
                        AttributeName: state
                        KeyType: HASH
                Projection:
                    ProjectionType: ALL

    StepFunctionRole:
      Type: 'AWS::IAM::Role'
      Properties:
        AssumeRolePolicyDocument:
            Version: '2012-10-17'
            Statement:
                -
                  Effect: Allow
                  Principal:
                      Service: 'states.amazonaws.com'
                  Action: 'sts:AssumeRole'
        Policies:
            -
              PolicyName: lambda
              PolicyDocument:
                Statement:
                  -
                    Effect: Allow
                    Action: 'lambda:InvokeFunction'
                    Resource:
                        - Fn::GetAtt: [DataCheckingLambdaFunction, Arn]
                        - Fn::GetAtt: [FlagApplicationLambdaFunction, Arn]

    ProcessApplicationsStateMachine:
      Type: AWS::StepFunctions::StateMachine
      Properties:
        StateMachineName: ${self:service}__process_account_applications__${self:provider.stage}
        RoleArn: !GetAtt StepFunctionRole.Arn
        DefinitionString:
          !Sub
            - |-
              {
                "StartAt": "Check Name",
                "States": {
                    "Check Name": {
                        "Type": "Task",
                        "Parameters": {
                            "command": "CHECK_NAME",
                            "data": { "name.$": "$.application.name" }
                        },
                        "Resource": "#{dataCheckingLambdaArn}",
                        "ResultPath": "$.checks.name",
                        "Next": "Check Address"
                    },
                    "Check Address": {
                        "Type": "Task",
                        "Parameters": {
                            "command": "CHECK_ADDRESS",
                            "data": { "address.$": "$.application.address" }
                        },
                        "Resource": "#{dataCheckingLambdaArn}",
                        "ResultPath": "$.checks.address",
                        "Next": "Review Required?"
                    },
                    "Review Required?": {
                        "Type": "Choice",
                        "Choices": [
                          {
                            "Variable": "$.checks.name.flagged",
                            "BooleanEquals": true,
                            "Next": "Pending Review"
                          },
                          {
                            "Variable": "$.checks.address.flagged",
                            "BooleanEquals": true,
                            "Next": "Pending Review"
                          }
                        ],
                        "Default": "Approve Application"
                    },
                    "Pending Review": {
                      "Type": "Task",
                      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
                      "Parameters": {
                          "FunctionName": "#{flagApplicationLambdaName}",
                          "Payload": {
                              "id.$": "$.application.id",
                              "flagType": "REVIEW",
                              "taskToken.$": "$$.Task.Token"
                          }
                      },
                      "ResultPath": "$.review",
                      "Next": "Review Approved?"
                    },
                    "Review Approved?": {
                        "Type": "Choice",
                        "Choices": [{
                                "Variable": "$.review.decision",
                                "StringEquals": "APPROVE",
                                "Next": "Approve Application"
                            },
                            {
                                "Variable": "$.review.decision",
                                "StringEquals": "REJECT",
                                "Next": "Reject Application"
                            }
                        ]
                    },
                    "Reject Application": {
                         "Type": "Pass",
                         "End": true
                     },
                    "Approve Application": {
                        "Type": "Pass",
                        "End": true
                    }
                }
              }
            - {
              dataCheckingLambdaArn: !GetAtt [DataCheckingLambdaFunction, Arn],
              flagApplicationLambdaName: !Ref FlagApplicationLambdaFunction,
            }
</textarea>
{{< /safehtml >}}

➡️ Step 4. Run:

```bash
sls deploy
```

### Try it out

Now we should be able to submit an invalid application, see that our application gets flagged for review, manually approve or reject the review, and then see our review decision feed back into the state machine for continued execution.

Let’s test this:

➡️ Step 1. Submit an invalid application so it gets flagged. Run:

```bash
sls invoke -f SubmitApplication --data='{ "name": "Spock", "address": "123EnterpriseStreet" }'
```

➡️ Step 2. Check to see that our application is flagged for review. Run:

```bash
sls invoke -f FindApplications --data='{ "state": "FLAGGED_FOR_REVIEW" }' 
```

➡️ Step 3. Copy the application’s ID from the results, which we’ll use in a step below to provide a review decision for the application.

➡️ Step 4. In Step Functions web console, refresh the details page for our state machine, and look for the most recent execution. You should see that it is labeled as ‘Running’. 

➡️ Step 5. Click in to the running execution and you’ll see in the visualization section that the Pending Review state is in-progress. This is the state machine indicating that it’s now paused and waiting for a callback before it will resume execution.

➡️ Step 6. To trigger this callback that it’s waiting for, act as a human reviewer and approve the review (we haven't built a web interface for this, so we'll just invoke another function in the Account Applications service. Take care to paste the ID you copied in Step 3 above into this command when you run it, replacing REPLACE_WITH_APPLICATION_ID. 

Run with replacement:

```bash
sls invoke -f ReviewApplication --data='{ "id": "REPLACE_WITH_APPLICATION_ID", "decision": "APPROVE" }'
```

➡️ Step 7. Go back to the execution details page in the Step Functions web console (you shouldn’t need to refresh it), and notice that the execution resumed and, because we approved the review, the state machine transitioned into the Approve Application state after examining the input provided to it by our callback.  You can click on the the ‘Review Approved?‘ step to see our review decision passed into the step’s input (via the SendTaskSuccess callback that `account-applications/review.js` called).


Pretty cool, right?

Finally, all we need to do to finish implementing our example workflow is to replace our Approve Application and Reject Application steps.  Currently they’re just placeholder Pass states, so let’s update them with Task states that will invoke the ApproveApplication and RejectApplication Lambda functions we’ve created..
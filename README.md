# ServerlessComplexEventScheduling
Streamlining Operations with AWS Systems Manager Calendar and EventBridge Rules

## Introduction

AWS Eventbridge provides a serverless way of scheduling jobs within the cloud. There are options based on schedule and on event basis. Sometimes these jobs need to be modified based on calendar events. For example, 
1. Process payroll every Friday, except if Friday is a holiday, in which case schedule it on Thursday.
2. Process files from an external vendor every Monday morning. If there is a long weekend, then files should be processed only the following working day.

There is no direct way to use EventBridge to utilize a calendar for these type of scheduling.

Another service called AWS Systems Manager Calendar offers a familiar calendar interface to schedule events that an be used to trigger eventbride. This guide explores the integration of these services for efficient holiday scheduling.

## Integration Use-Case: Holiday Scheduling

Integrating Systems Manager Calendar with EventBridge Rules is effective for pausing or modifying actions during holidays. In this guide we have a job that triggers a lambda every 2 minutes. We will create an event in the change calendar, Thanksgiving day (more of a 10 min thanksgiving snack ), during which we do not want to job to execute. We will see how integration of these 2 services helps us achive that. 

## Understanding the Components

### AWS Systems Manager Calendar

Systems Manager Calendar allows defining operational periods like holidays for managing tasks. [Learn more about Systems Manager Calendar](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-calendar.html).

### AWS EventBridge Rules

EventBridge facilitates event-driven architectures and triggers AWS services in response to events. [Explore EventBridge Rules](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule.html).



## Step-by-Step Integration Guide

### Step 1: Set Up AWS Systems Manager Calendar

Systems Manager Change Calendar lets you set up date and time ranges (referred to as events) to specify when actions can be performed in your AWS account. When you create a Change Calendar entry, you are creating a Systems Manager document of the type Change Calendar that contains iCalendar 2.0 data in plaintext format. A Change Calendar entry can be one of two types:

**Open by default (DEFAULT_OPEN)**: Actions that are tracking Change Calendar can run by default but are blocked from running during associated events. During events, the state of a DEFAULT_OPEN calendar is CLOSED.
**Closed by default (DEFAULT_CLOSE)**: Actions that are tracking Change Calendar do not run by default but can run during events associated with the calendar entry. During events, the state of a DEFAULT_CLOSED calendar is OPEN.

Let’s use the Thanksgiving example. On ThanksGiving Day, you don’t want your job which usually runs every day to run. In other words, there is only one event when the job must be blocked.

You create a Change Calendar entry of DEFAULT_OPEN  that allows jobs to run and then you create an event for "Thanksgiving day snack", which changes the calendar state to CLOSED during the event.


The solution described in this blog post includes the following steps:

1. Create a Systems Manager Change Calendar entry using the DEFAULT_OPEN type.
2. Create a Systems Manager Automation document to suspend the pipeline release.
3. Create an EventBridge rules to excte the job and to monitor the calendar’s state and trigger the Automation document when the calendar state changes.

- 1. **Create a Holiday Calendar**: 
- **Define your holiday calendar in Systems Manager.**
[Guide to create calendars](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-calendar.html).The Calendar is either DEFAULT_OPEN or DEFAULT_CLOSED. For holiday calendar, we will keep the calendar DEFAULT_OPEN. A Calendar has events with a specific start and end times. These events are used to trigger EventBridge - at the start of the event, transitioning from Open to Closed and at the end of the event, transitioning from Closed to Open.

Click on the "Details" Tab on the Calendar and open the "Calendar Use" section. The ARN of this calendar is used in the later stages

- **Create Thanksgiving day snack event**:
Create an event on the calendar by clicking the Create Event Button. Define the start time 30 minutes later than now and ending at 40 minutes later than now. Lets call this "Thanksgiving day snack"

- 2. **Create an automation document**: [Systems Manager Automation](https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-ref-sys.html) documents or runbooks help define a series of steps that can be done on your infrastructure and supports a wide range of AWS infrastructure APIs. Automations also support workflows using the new Visual design tool and can be used for manual steps such as an approval before the change event can start.

You will find "Automations" under "Change Management" section. 
* Click on "Create Runbook"
* Explore the new visual design interface. You can get more details on different elements of the runbook [here](https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-documents.html)
* For our runbook, click on the "{} Code" button and paste the code below:

```
    {
    "description": "Disables the specified Amazon EventBridge rule",
    "schemaVersion": "0.3",
    "parameters": {
        "EventBusName": {
        "type": "String",
        "default": "default",
        "description": "The event bus associated with the rule"
        },
        "RuleName": {
        "type": "String",
        "description": "The name of the rule to disable"
        },
        "assumeRole": {
        "type": "AWS::IAM::Role::Arn",
        "description": "RoleToExecuteThisWith"
        }
    },
    "assumeRole": "{{ assumeRole }}",
    "mainSteps": [
            {
            "name": "DisableRule",
            "action": "aws:executeAwsApi",
            "isEnd": true,
            "inputs": {
                "Service": "events",
                "Api": "DisableRule",
                "Name": "{{ RuleName }}",
                "EventBusName": "{{ EventBusName }}"
            }
            }
        ]
    }
```
* We will use the DisableRule API usinf the action "aws:executeApi". This docment needs 3 input parameters

| Parameter      | Type         | Description                           | Default       |
| ---------------| -------------|---------------------------------------|---------------|
| EventBusName   | String       |The arn of the event bus               | default       |
| RuleName       | String              | Name of the rule to disable    | |
| assumeRole     | AWS::IAM::Role::Arn | Arn of the IAM role to use     | |

* Lets name this runboook "Disable Events" 
* Create a similar runbook to enable events using the code below called "Enable Events"
```
{
  "description": "Disables the specified Amazon EventBridge rule",
  "schemaVersion": "0.3",
  "parameters": {
    "EventBusName": {
      "type": "String",
      "default": "default",
      "description": "The event bus associated with the rule"
    },
    "RuleName": {
      "type": "String",
      "description": "The name of the rule to disable"
    },
    "assumeRole": {
      "type": "AWS::IAM::Role::Arn",
      "description": "RoleToExecuteThisWith"
    }
  },
  "assumeRole": "{{ assumeRole }}",
  "mainSteps": [
    {
      "name": "EnableRule",
      "action": "aws:executeAwsApi",
      "isEnd": true,
      "inputs": {
        "Service": "events",
        "Api": "EnableRule",
        "Name": "{{ RuleName }}",
        "EventBusName": "{{ EventBusName }}"
      }
    }
  ]
}
```



### Step 3: Create an EventBridge Rules
For this POC lets define a rule that triggers a job that runs every 2 minutes.

- **Define the Rule to run the job**: Navigate to EventBrodge -> Rules an [create a new rule](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule.html) based on a schedule  to run every 2 minutes. For the target, you can create a sample hello world lambda to log a message whenever it is triggerred. We will use this to test out the integration later. Lets call this "Job Rule"

- **Define the rules to track calendar events**
* Create a new rule called "Disable Job Based on Calendar Event" based on a "Rule with an event pattern"
* In the Event Pattern paste the code below replacing the arn with the arn from the HolidayCalendar that was created earlier (note the escape character for the slash)

```
{
  "source": ["aws.ssm"],
  "detail-type": ["Calendar State Change"],
  "resources": ["arn:aws:ssm:us-east-1:454340502151:document\/HolidayCalendar"],
  "detail": {
    "state": ["CLOSED"]
  }
}
```

* This rule is triggerred by the switch to an "CLOSED" state from the Holiday calendar
* For the Target Type select "AWS Service" and select the "Disable Events" runbook that you created earlier
* For this use case purposes we will hard code the parameter values as constants. 
* Paste the arn of the eventbus that has the "Job rule" in the EventBusName 
* Paste the name of the rule "Job rule" in the RuleName field
* Create a new IAM role to execute that has full access to eventbridge and provide the arn in the assumeRole field
* Create a new IAM role to execute this rule that has full access to the Systems Manager Calendar.
* Create a similar rule called "Enable Job Based on Calendar Event" following steps above with the using code below for event pattern. Make sure you select the "Enable Events" runbook or this rule
```
{
  "source": ["aws.ssm"],
  "detail-type": ["Calendar State Change"],
  "resources": ["arn:aws:ssm:us-east-XXXXXXXXX:document\/HolidayCalendar"], 
  "detail": {
    "state": ["OPEN"]
  }
}
```

Here is how the integration works:
1. .The Holiday Calendar is "DEFAULT OPEN" state
2. At the start of the event "Thanksgiving day snack" the calendar sends a state change event to eventbridge to trigger the "Disable Job Based on Calendar Event" rule. This rule runs the runbook that in turn disables the "Job rule"
4. At the end of the event "Thanksfgiving day snack" the calendar sends a state change event to eventbridge to trigger the "Enable Job Based on Calendar Event" rule. This rule runs the runbook that in turn enables the "Job rule".

### Step 4: Testing and Validation

- **Test the Setup**: Adjust the calendar event to simulate holidays and observe the rule’s behavior.
- **Monitor and Adjust**: Monitor with AWS CloudWatch. [About CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html).

## Best Practices and Considerations

- **Regularly Update the Holiday Calendar**: Keep the calendar updated with your organization's schedule.
- **Use Tags for Organization**: Effective for managing multiple calendars and rules.
- **Security and Compliance**: Ensure alignment with organizational standards. Limit access to all IAM roles to the ones absolutely necessary

## Conclusion
While Systems Manager itself requires an agent to run on EC2 instances and other computes, combining AWS Systems Manager Calendar with AWS EventBridge Rules enables complex event scheduling based on calendar events in a serverless way. While this blog gives a console way of setting up this integration, CLI and APIs can be used to automate deployment of the Calendar and the Rules.

Happy automated serverless complex scheduling! 🎉📆

---

For advanced applications or support, consult AWS certified professionals or refer to AWS documentation.


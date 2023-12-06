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

Let’s use the Thanksgiving example. On ThanksGiving Day, you don’t want your job which usually runs every day to run. In other words, there is only one day when the job must be blocked.

You create a Change Calendar entry of DEFAULT_OPEN  that allows deployments and then you create an event for New Year’s Day, which changes the calendar state to CLOSED during the event. You can monitor this change in the default state of the calendar with Amazon EventBridge rules that trigger the configured workflows.

In this blog post, you’ll create a public calendar of DEFAULT_OPEN with public holidays as calendar events. Using Amazon EventBridge, you’ll track the CLOSED events, causing a change in the calendar’s state and triggering an automation workflow to suspend the run job rule.

The solution described in this blog post includes the following steps:

1. Create a Systems Manager Change Calendar entry using the DEFAULT_OPEN type.
2. Create a Systems Manager Automation document to suspend the pipeline release.
3. Create an AWS Identity and Access Management (IAM) policy and a role to delegate permissions to the Systems Manager Automation document.
4. Create an IAM role to delegate permissions to EventBridge events, which invokes the Systems Manager Automation document.
5. Create an EventBridge rule to monitor the calendar’s state and trigger the Automation document when the calendar state changes.

- 1. **Create a Holiday Calendar**: Define your holiday calendar in Systems Manager. [Guide to create calendars](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-calendar.html).The Calendar is either DEFAULT_OPEN or DEFAULT_CLOSED. For holiday calendar, we will keep the calendar DEFAULT_OPEN. A Calendar has events with a specific start and end times. These events are used to trigger EventBridge - at the start of the event, transitioning from Open to Closed and at the end of the event, transitioning from Closed to Open.

- 2. **Create an automation document**: [Systems Manager Automation] (https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-ref-sys.html) documents or runbooks help define a series of steps that can be done on your infrastructure and supports a wide range of AWS infrastructure APIs. Automations also support workflows using the new Visual design tool and can be used for manual steps such as an approval before the change event can start.

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

- **Input Holiday Dates**: Add specific holiday dates including specific times you do not want the rules to be executed.

Click on the "Details" Tab on the Calendar and open the "Calendar Use" section. The ARN of this calendar is used in the later stages

### Step 2: Create an EventBridge Rule
For this POC lets define a rule that triggers a job that runs every 2 minutes.

- **Define the Rule**: Set up an EventBridge rule for tasks like data backup. [Creating EventBridge Rules](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule.html).
- **Set Up the Trigger**: Configure triggers based on events 

### Step 3: Integrate Systems Manager Calendar with EventBridge

- **Link the Calendar to the Rule**: Use the calendar's state as a condition in your EventBridge rule.
- **Configure Rule State**: The rule should respond to the calendar state, enabling or disabling as needed.

### Step 4: Testing and Validation

- **Test the Setup**: Adjust the calendar to simulate holidays and observe the rule’s behavior.
- **Monitor and Adjust**: Monitor with AWS CloudWatch. [About CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html).

## Best Practices and Considerations

- **Regularly Update the Holiday Calendar**: Keep the calendar updated with your organization's schedule.
- **Use Tags for Organization**: Effective for managing multiple calendars and rules.
- **Security and Compliance**: Ensure alignment with organizational standards.

## Conclusion

Combining AWS Systems Manager Calendar with AWS EventBridge Rules enhances resource efficiency and operational control. Follow these steps for successful integration.

Happy automated holiday scheduling! 🎉📆

---

For advanced applications or support, consult AWS certified professionals or refer to AWS documentation.


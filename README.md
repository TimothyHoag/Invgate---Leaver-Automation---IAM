# JML Leaver Automation: IAM + ITSM Workflow with n8n, Active Directory, InvGate, and Slack

## Overview

This project demonstrates a leaver/offboarding workflow that connects identity and access management (IAM) with IT service management (ITSM). Filing a leaver request in InvGate triggers an n8n workflow that:

- Reads the leaver's identity details from the InvGate ticket
- Disables the target Active Directory account
- Removes group memberships
- Posts timestamped audit comments back to the InvGate ticket at each step
- Sends a Slack notification
- Logs results and exceptions

The goal is not just to automate tasks, but to show the operational reasoning behind each step in a controlled offboarding process.

## Why I Built This

Leaver processes are often handled manually across multiple systems, which creates risk. Common problems include:

- Delayed account disablement
- Incomplete removal of access
- Poor visibility into whether offboarding was completed
- Inconsistent documentation between IAM and service management teams

I built this project to explore how a structured workflow can reduce manual effort, improve consistency, and create a more auditable offboarding process.

This project also reflects the overlap between:

- Identity and access management
- IT operations
- Workflow automation
- IT service management

## Project Goals

- Reduce manual steps in employee offboarding
- Improve consistency in leaver handling
- Create traceable workflow activity
- Integrate technical deprovisioning with ITSM records
- Demonstrate practical automation design, not just scripting
- Document the reasoning behind the control flow

## Workflows

- **[workflows/jml-leaver-flow-v1.json](workflows/jml-leaver-flow-v1.json)** — the primary leaver/offboarding workflow.
- **[workflows/jml-joiner-flow-v1.json](workflows/jml-joiner-flow-v1.json)** — a joiner/onboarding workflow draft using the same pattern.
- **[workflows/jml-combined-flow-v1.json](workflows/jml-combined-flow-v1.json)** — both flows combined into a single export.
- **[workflows/ad-ssh-test-workflow.json](workflows/ad-ssh-test-workflow.json)** — a minimal workflow for verifying SSH/Active Directory connectivity.

## Architecture

### High-Level Flow

1. A requester (or HR) files a leaver request against the "Leaver" catalog item in InvGate.
2. An InvGate automation rule fires on ticket creation and calls the n8n production webhook, passing the employee name, username, manager, last day, and the InvGate `request_id`.
3. n8n's Webhook node receives the payload; `Set Leaver Data` extracts and maps the fields.
4. n8n posts a "ticket received" comment back to the InvGate ticket, confirming automation has started.
5. The Active Directory account is disabled over SSH (`Disable-ADAccount`).
6. n8n comments on the ticket confirming the account is disabled.
7. Group memberships are removed over SSH (`Remove-ADGroupMember`).
8. A final "leaver complete" comment is posted to the ticket.
9. A Slack notification is sent to the relevant channel or stakeholder.
10. The workflow records success, failure, or exception details.

```text
InvGate: New request created (Leaver catalog item)
  -> Automation rule calls n8n webhook (request_id + employee fields)
  -> Webhook trigger -> Set Leaver Data
  -> Comment: Ticket Received
  -> Disable AD Account (SSH / PowerShell)
  -> Comment: Account Disabled
  -> Remove AD Groups (SSH / PowerShell)
  -> Comment: Leaver Complete
  -> Notify Slack
```

### Core Components

- **InvGate:** Ticket intake, webhook trigger, and audit trail (comments)
- **n8n:** Workflow orchestration
- **Active Directory and PowerShell (over SSH):** Account disable and group removal
- **Slack:** Operational notifications
- **Logging and error handling:** Visibility and troubleshooting

## Workflow Walkthrough

### 1. Request Intake

The process begins when a leaver request is filed against the Leaver catalog item in InvGate. Required fields are captured as InvGate custom fields on that catalog item:

- Employee name
- Username
- Manager
- Last day
- (InvGate assigns the `request_id` used for all audit comments)

### 2. Trigger

An InvGate automation rule (event: new request created, condition: Leaver catalog item) calls the n8n webhook via a "Call Web Service" action, authenticated with a shared header secret. This replaces manual form submission or a separate trigger step — the ticket itself is the trigger.

### 3. Field Mapping

The `Set Leaver Data` node reads the webhook payload and maps employee name, username, manager, last day, and `request_id` into the fields the rest of the workflow uses.

### 4. Disable Account

The first core access-control action is disabling the Active Directory account (`Disable-ADAccount` over SSH). This helps reduce the risk of continued access while later cleanup actions are completed. n8n posts a confirmation comment to the InvGate ticket immediately after.

### 5. Remove Group Memberships

After the account is disabled, the workflow removes its group memberships (`Remove-ADGroupMember` over SSH). This step is important because:

- Group memberships often drive application and resource access.
- Removing them helps clean up residual entitlements.
- It reduces the chance of access persisting if the account is later re-enabled in error.

### 6. Audit Trail via InvGate Comments

Rather than creating a separate ticket, the workflow posts timestamped comments back to the same InvGate ticket that triggered it — one at intake, one after disable, one after group removal. This provides:

- Operational visibility on the originating ticket
- A service management trail without duplicate records
- A handoff point when manual review is needed

### 7. Send a Slack Notification

A Slack notification is sent to the relevant team or channel once the flow completes. This communicates that:

- The request was received.
- The offboarding action was completed.
- An exception occurred and requires review.

### 8. Log Results

The workflow records the outcome of each major step for troubleshooting and auditability.

## Control Logic and Design Decisions

### Why trigger from the InvGate ticket instead of a form?

The ticket is already the system of record for the request. Triggering directly off ticket creation avoids a duplicate intake step and keeps the InvGate ticket as the single source of truth for the request and its audit trail.

### Why disable the account first?

Disabling the account immediately reduces risk. If offboarding is delayed or interrupted, the user should not retain active sign-in access.

### Why remove groups after disabling the account?

Group removal is still important, but it comes after disablement because the highest-priority control is preventing sign-in.

### Why comment on the existing ticket instead of creating a new one?

A technical action without a service record can be difficult to trace later. Commenting on the originating ticket keeps the full history — request, actions taken, and outcome — in one place instead of splitting it across records.

### Why send Slack notifications?

Offboarding often involves multiple stakeholders. Slack adds visibility without requiring people to check the ticket or workflow status manually.

### Why document the reasoning, not just the build?

Automation is more useful when people understand the control decisions behind it. The workflow matters, but the design logic matters too.

## Setup

Import the workflow JSON files from `workflows/` into n8n. None of the credentials, tenant URLs, or webhook secrets are committed to this repository — configure them as n8n credentials or environment-specific settings after import. Placeholder values in the JSON (e.g. `REPLACE_WITH_YOUR_CREDENTIAL_ID`, `REPLACE_WITH_YOUR_CREDENTIAL_NAME`, `YOUR_INVGATE_TENANT`) mark where this is required. On the InvGate side, the Leaver catalog item needs the custom fields above and an automation rule configured to call the n8n webhook on ticket creation.

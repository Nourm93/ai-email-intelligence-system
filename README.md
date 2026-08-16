# AI Email Intelligence System

A production-style AI email processing workflow built with **n8n**,
**Gmail**, **OpenAI**, **Anthropic Claude**, and **Notion**.

The system automatically receives new emails, removes duplicates,
classifies their importance and category, extracts structured
information from important messages, validates AI-generated JSON,
retries failed generations, falls back to a second AI provider when
necessary, normalizes the final output, and stores the result in Notion.

## Overview
<img width="1197" height="691" alt="Bildschirmfoto 2026-08-15 um 18 07 31" src="https://github.com/user-attachments/assets/2b0a86f9-89e8-49b3-8c8f-6bab4bdb03c7" />

The goal of this project is to demonstrate how an LLM can be used as one
component inside a reliable automation system rather than as a
standalone chatbot.

The workflow combines event-driven automation, structured AI outputs,
validation, branching, retry logic, multi-model fallback, data

## Successful Workflow Execution

The workflow was tested end-to-end with a sample email. The execution below shows the successful processing path through classification, validation, AI summarization, normalization, and Notion storage.
<img width="3978" height="2378" alt="IMG_5725" src="https://github.com/user-attachments/assets/c9104895-2cb3-48da-aa28-9ad20a57f526" />


## Architecture

``` text
Gmail Trigger
     |
     v
Get a Message
     |
     v
Remove Duplicates
     |
     v
OpenAI Classifier
     |
     v
Validate Classification
     |
     v
Important?
     |
    Yes
     |
     v
Claude Summarizer
     |
     v
Validate Summarizer
   /       \
Valid      Error
 |           |
 |           v
 |      Retry Claude
 |           |
 |           v
 |      Validate Retry
 |        /       \
 |     Valid      Error
 |       |          |
 |       |          v
 |       |     OpenAI Fallback
 |       |          |
 |       |          v
 |       |   Validate Fallback
 |       |          |
 +-------+----------+
         |
         v
 Merge Valid Results
         |
         v
  Normalize Output
         |
         v
       Notion
```

Emails classified as not important do not enter the expensive
summarization branch.

## Key Features

-   Gmail-based email ingestion
-   Full-message retrieval
-   Duplicate prevention using the Gmail message ID
-   AI-based importance classification
-   Eight email categories
-   Structured JSON output
-   Strict validation of AI responses
-   Conditional processing of important emails
-   Claude-based summarization and information extraction
-   Automatic retry after validation failure
-   OpenAI fallback model
-   Multi-path result merging
-   Final output normalization
-   Relative deadline conversion
-   Sender email extraction
-   Notion database storage
-   Credential-free public workflow template

## Email Categories

The classifier uses exactly these categories:

``` text
work
personal
finance
appointment
security
newsletter
promotion
other
```

## Workflow Details

### 1. Gmail Trigger

The workflow starts when Gmail detects a new message. The public
template contains a placeholder sender filter:

``` text
-from:YOUR_EMAIL@example.com
```

This can be used to prevent test emails sent from the same account from
triggering unwanted loops.

### 2. Get a Message

The Gmail trigger provides the event. The next node retrieves the
complete email so downstream nodes can work with the full text or HTML
content.

### 3. Remove Duplicates

The workflow removes messages already seen in previous executions using
the Gmail message ID.

This reduces the risk of processing and storing the same email more than
once.

### 4. OpenAI Classifier

The first AI stage decides whether an email is important and assigns a
category.

Expected structure:

``` json
{
  "important": true,
  "category": "work",
  "reason": "The email contains a deadline and requires action."
}
```

An email is treated as important when it contains signals such as a
deadline, task, request, appointment, financial information, security
issue, or required response.

Newsletters and promotions are normally treated as non-important.

### 5. Validate Classification

A JavaScript validation node parses the model output and verifies that:

-   `important` is a Boolean.
-   `category` is one of the eight allowed categories.
-   `reason` is a non-empty string.
-   The response is valid JSON.

Invalid output raises an error instead of silently entering the business
logic.

### 6. Important? Branch

Only emails with:

``` text
important = true
```

continue to the summarization pipeline.

This avoids unnecessary LLM calls for low-value messages.

### 7. Claude Summarizer

Important emails are analyzed by Claude.

The model is instructed to return only:

``` json
{
  "summary": "maximum 3 short sentences",
  "action_items": [
    "action 1",
    "action 2"
  ],
  "deadline": "deadline if mentioned, otherwise null",
  "priority": "low, medium, or high"
}
```

The model is explicitly instructed not to invent information.

### 8. Validate Summarizer

The output is parsed and validated before it can continue.

The validator checks:

-   `summary` is a non-empty string.
-   `action_items` is an array.
-   Every action item is a non-empty string.
-   `deadline` is either a string or `null`.
-   `priority` is exactly `low`, `medium`, or `high`.

### 9. Retry Claude Summarizer

If the first summarizer output fails validation, the error output is
routed to a second Claude request.

The retry prompt is deliberately stricter and emphasizes that the model
must return only valid JSON matching the required schema.

### 10. OpenAI Fallback

If the retry also fails validation, the original email is sent to an
OpenAI fallback model.

This provides provider/model redundancy instead of allowing a single
malformed response to terminate the processing path.

### 11. Merge Valid Results

Successful output can come from three paths:

``` text
Primary Claude
Retry Claude
OpenAI Fallback
```

The Merge node reunifies those paths before normalization.

### 12. Normalize Output

The final JavaScript node creates one consistent interface for the
storage layer.

The normalized object has this shape:

``` json
{
  "subject": "Example subject",
  "sender": "sender@example.com",
  "category": "work",
  "summary": "Short factual summary.",
  "action_items": [
    "First task",
    "Second task"
  ],
  "deadline": "2026-08-16T13:00:00.000Z",
  "priority": "high",
  "processed_at": "2026-08-15T12:00:00.000Z"
}
```

This node also performs defensive normalization.

#### String Cleaning

``` javascript
function cleanString(value, fallback = '') {
  return typeof value === 'string'
    ? value.trim()
    : fallback;
}
```

This prevents unexpected non-string values from propagating into Notion.

#### Category Validation

Only the eight supported categories are accepted. Unexpected values fall
back to:

``` text
other
```

#### Priority Validation

Only:

``` text
low
medium
high
```

are accepted. Unexpected values fall back to `low`.

#### Sender Extraction

Gmail may return a sender in this form:

``` text
Max Mustermann <max@example.com>
```

A regular expression extracts:

``` text
max@example.com
```

so it can be stored in a Notion Email property.

#### Deadline Normalization

The workflow supports simple relative deadline expressions such as:

``` text
tomorrow at 13:00
morgen um 13:00
today at 16:30
heute um 16:30
```

and converts them into an ISO timestamp that can be stored in Notion.

Already-valid ISO dates are passed through.

Unknown natural-language date formats return `null` instead of guessing.

> Note: the current deadline parser intentionally supports a limited set
> of relative expressions. A production deployment could replace this
> with a dedicated date parser or require the LLM to emit ISO-8601
> directly.

### 13. Notion Storage

The normalized result is stored as a new Notion database page.

Recommended database properties:

  Property        Notion Type   Source
  --------------- ------------- --------------------------------
  Email Subject   Title         `subject`
  Summary         Text          `summary`
  Priority        Select        `priority`
  Category        Select        `category`
  Action          Text          `action_items` joined with `;`
  Deadline        Date          `deadline`
  Sender          Email         `sender`
  Processed At    Date          `processed_at`

For `Priority`, create:

``` text
low
medium
high
```

For `Category`, create:

``` text
work
personal
finance
appointment
security
newsletter
promotion
other
```

## Example

### Incoming Email

``` text
Subject: Urgent: Customer presentation due tomorrow

Hi,

Please check the latest sales figures, update the Excel file,
and send me the corrected version by tomorrow at 13:00.

The final customer presentation is scheduled for tomorrow afternoon.
```

### Classification

``` json
{
  "important": true,
  "category": "work",
  "reason": "The email contains actionable tasks and a deadline."
}
```

### Extracted Information

``` json
{
  "summary": "The customer presentation must be prepared for tomorrow. The sales figures and Excel file need to be reviewed and updated.",
  "action_items": [
    "Check the latest sales figures",
    "Update the Excel file",
    "Send the corrected file"
  ],
  "deadline": "tomorrow at 13:00",
  "priority": "high"
}
```

### Normalized Result

``` json
{
  "subject": "Urgent: Customer presentation due tomorrow",
  "sender": "sender@example.com",
  "category": "work",
  "summary": "The customer presentation must be prepared for tomorrow. The sales figures and Excel file need to be reviewed and updated.",
  "action_items": [
    "Check the latest sales figures",
    "Update the Excel file",
    "Send the corrected file"
  ],
  "deadline": "2026-08-16T13:00:00.000Z",
  "priority": "high",
  "processed_at": "2026-08-15T12:00:00.000Z"
}
```

The timestamps above are illustrative examples.

## Tech Stack

-   **n8n** --- workflow orchestration
-   **Gmail** --- email trigger and message retrieval
-   **OpenAI** --- email classification and fallback summarization
-   **Anthropic Claude** --- primary email summarization
-   **JavaScript** --- validation and normalization
-   **JSON** --- structured communication between AI and workflow nodes
-   **Notion** --- structured storage and review

## Setup

### Prerequisites

You need:

-   an n8n instance
-   a Gmail account connected to n8n
-   an OpenAI API credential
-   an Anthropic API credential
-   a Notion integration
-   a Notion database with the required properties

### Import the Workflow

Import:

``` text
AI_Email_Intelligence_System_PUBLIC_FINAL.json
```

into n8n.

### Configure Credentials

Because credentials are intentionally removed from the public template,
reconnect:

-   Gmail OAuth
-   OpenAI API
-   Anthropic API
-   Notion API

Never commit API keys or access tokens to a public repository.

### Configure Gmail

Replace:

``` text
YOUR_EMAIL@example.com
```

with the appropriate address if you want to use the sender exclusion
filter.

### Configure Notion

Select your own Notion database in the final Notion node.

Ensure that the Notion integration has access to that database.

Create the database properties listed in the **Notion Storage** section
above.

### Test

Send a controlled test email containing:

-   a clear task
-   a deadline
-   enough context for a summary

Then verify:

1.  Gmail triggers the workflow.
2.  The email is classified.
3.  Important mail enters the summarizer path.
4.  The AI output passes validation.
5.  The final normalized object contains all expected fields.
6.  A new Notion page is created.

Also test edge cases such as:

-   an unimportant newsletter
-   an email without a deadline
-   an email without action items
-   malformed AI output
-   a duplicate message

-   ## How to Run This Workflow

This section explains how another user can run and test the AI Email Intelligence System locally using n8n.

### 1. Install Docker

Install Docker Desktop on your computer and make sure Docker is running.

### 2. Start n8n with Docker

Create a persistent Docker volume:

```bash
docker volume create n8n_data
```

Start n8n:

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

After n8n starts, open the following address in your browser:

```text
http://localhost:5678
```

You should now see the n8n interface.

### 3. Import the Workflow

Download the public workflow file from this repository:

```text
workflow/AI_Email_Intelligence_System_PUBLIC_FINAL.json
```

Then, inside n8n:

1. Open **Workflows**.
2. Select **Import from File**.
3. Choose the downloaded JSON workflow file.
4. Save the imported workflow.

### 4. Configure Your Credentials

For security reasons, the public workflow does not contain working credentials, API keys, or private account information.

Create and connect your own credentials for:

- Gmail
- OpenAI
- Anthropic Claude
- Notion

Never share API keys, passwords, OAuth tokens, or other credentials inside a public GitHub repository.

### 5. Configure the Gmail Trigger

Open the **Gmail Trigger** node in the imported workflow.

The public workflow contains the following placeholder filter:

```text
-from:YOUR_EMAIL@example.com
```

Replace:

```text
YOUR_EMAIL@example.com
```

with your own email address if you want to prevent emails sent from your own account from triggering the workflow.

For example:

```text
-from:john@example.com
```

The Gmail Trigger monitors the connected Gmail account and starts the workflow when a new email matching the configured conditions is received.

Make sure that:

- your Gmail credential is connected
- the Gmail Trigger is enabled
- the filter contains the correct email address
- the workflow is active when testing automatic execution

### 6. Configure the Notion Database

Create a Notion database that contains the properties required by the workflow.

Recommended properties:

| Property | Notion Type |
| --- | --- |
| Email Subject | Title |
| Summary | Text |
| Priority | Select |
| Category | Select |
| Action | Text |
| Deadline | Date |
| Sender | Email |
| Processed At | Date |

Create the following options for **Priority**:

```text
low
medium
high
```

Create the following options for **Category**:

```text
work
personal
finance
appointment
security
newsletter
promotion
other
```

Connect your Notion integration to this database.

Then open the final Notion node in the n8n workflow and select your own Notion database.

Make sure the Notion integration has permission to access the database.

### 7. Test the Workflow

Before using the workflow in production, perform a controlled test.

Send a test email to the Gmail account connected to the workflow.

Example test email:

```text
Subject: Urgent: Customer presentation due tomorrow

Hi,

please check the latest sales figures, update the Excel file,
and send me the corrected version by tomorrow at 13:00.

The customer presentation is scheduled for tomorrow afternoon.

Thanks.
```

This test email contains:

- a clear business task
- multiple action items
- a deadline
- enough context for AI classification and summarization

The expected processing path is:

```text
Gmail Trigger
↓
Get a Message
↓
Remove Duplicates
↓
OpenAI Classifier
↓
Validate Classification
↓
Important?
↓
Claude Summarizer
↓
Validate Summarizer
↓
Retry Claude if required
↓
OpenAI Fallback if required
↓
Merge Valid Results
↓
Normalize Output
↓
Notion
```

If the primary Claude output is valid, the retry and fallback paths are not required.

If validation fails, the workflow automatically attempts recovery through the configured retry and fallback logic.

### 8. Verify the Workflow Execution

After sending the test email, open the workflow execution in n8n.

A successful execution should show the relevant processing nodes completing successfully.

Check that:

- Gmail detected the message
- the complete email was retrieved
- duplicate prevention completed
- the email was classified correctly
- important emails entered the summarization branch
- the AI output passed validation
- the final data was normalized
- the Notion node completed successfully

### 9. Verify the Result in Notion

Open your Notion database.

A successfully processed email should create a new database entry containing information similar to:

```text
Email Subject: Urgent: Customer presentation due tomorrow
Priority: high
Category: work
Deadline: extracted and normalized deadline
Summary: AI-generated factual summary
Action Items: extracted tasks
Sender: original sender email address
Processed At: workflow processing timestamp
```

The exact summary and action-item wording may vary because the content is generated by an AI model.

### 10. Test a Non-Important Email

It is also recommended to test the classification branch with a non-important message.

For example:

```text
Subject: Weekly Newsletter

Hi,

here is this week's newsletter with our latest articles and updates.

Have a great weekend.
```

The classifier should normally categorize this type of message as:

```json
{
  "important": false,
  "category": "newsletter",
  "reason": "The email is informational and does not require immediate action."
}
```

Because the email is not important, it should not continue through the expensive AI summarization pipeline.

### 11. Test Duplicate Prevention

To verify duplicate handling, make sure the same Gmail message is not processed repeatedly.

The workflow uses the Gmail message ID to identify messages that have already been processed during the relevant workflow execution logic.

This helps prevent duplicate Notion entries and unnecessary AI API calls.

### 12. Activate the Workflow

After all tests complete successfully, activate the workflow in n8n.

From that point on, new emails matching the Gmail Trigger conditions can automatically start the workflow.

The production flow becomes:

```text
New Gmail Message
        ↓
Automatic Classification
        ↓
Important?
     /       \
   No         Yes
   ↓           ↓
 Stop     AI Processing
              ↓
         Validation
              ↓
      Retry / Fallback
              ↓
        Normalization
              ↓
            Notion
```

## Troubleshooting

### Workflow Does Not Start

Check that:

- the Gmail credential is connected
- the workflow is active
- the Gmail Trigger is configured correctly
- the Gmail filter is correct
- the incoming email matches the trigger conditions

### Gmail Trigger Does Not Detect the Test Email

Check the Gmail Trigger filter.

If you use:

```text
-from:YOUR_EMAIL@example.com
```

emails sent from that address will be excluded.

For testing, send the message from another email account or temporarily adjust the filter.

### OpenAI Node Fails

Check:

- OpenAI credentials
- API key validity
- available API credits
- selected model
- API connectivity

### Claude Node Fails

Check:

- Anthropic credentials
- API key validity
- available API credits
- selected Claude model
- API connectivity

The workflow includes retry and fallback logic for invalid AI-generated output, but invalid or missing API credentials must still be fixed manually.

### AI Output Fails Validation

The workflow expects structured JSON.

If the primary AI response does not match the expected structure, the validation layer rejects it.

The workflow can then use:

```text
Primary AI
↓
Validation Failure
↓
Retry
↓
Validation
↓
Fallback Model
↓
Validation
```

This behavior is intentional and is one of the reliability mechanisms built into the project.

### Notion Node Fails

Check that:

- the Notion credential is connected
- the Notion integration has access to the database
- the correct database is selected
- database property names match the workflow
- database property types are correct
- the deadline contains a valid date value

### Workflow Works Manually but Not Automatically

Make sure the workflow is **activated**.

Manual execution is useful during development and testing, but the Gmail Trigger requires an active workflow for normal automatic operation.

## Security Reminder

Do not publish or share:

- Gmail credentials
- OpenAI API keys
- Anthropic API keys
- Notion API tokens
- OAuth tokens
- real customer emails
- private email content
- personally identifiable information

The workflow included in this repository is intended to be a credential-free template.

Each user should connect their own accounts and credentials after importing the workflow.

## Credential Setup Guide

The public workflow included in this repository does not contain active credentials, API keys, OAuth tokens, or private account information.

Each user must connect their own Gmail, OpenAI, Anthropic Claude, and Notion accounts before running the workflow.

> **Security:** Never commit API keys, access tokens, passwords, OAuth secrets, or other credentials to GitHub.

### 1. Gmail Credential

The Gmail Trigger and Gmail message nodes require access to your own Gmail account.

In n8n:

1. Open the **Gmail Trigger** node.
2. Go to **Credential to connect with**.
3. Create a new Gmail credential.
4. Follow the Google authorization process.
5. Sign in with the Gmail account you want the workflow to monitor.
6. Grant the permissions required by n8n.
7. Save the credential.
8. Select the same Gmail credential in the Gmail nodes that require Gmail access.

After connecting Gmail, configure the Gmail Trigger filter.

The public workflow contains the placeholder:

```text
-from:YOUR_EMAIL@example.com
```

Replace:

```text
YOUR_EMAIL@example.com
```

with your own email address.

Example:

```text
-from:john@example.com
```

This can help prevent emails sent from your own address from triggering the workflow.

For testing, you can send an email from a different email account to the Gmail account connected to n8n.

---

### 2. OpenAI Credential

The workflow uses OpenAI for AI-based email processing and as part of the fallback architecture.

You must use your own OpenAI API account and API key.

#### Create an OpenAI API Key

1. Sign in to the OpenAI API platform.
2. Open the API key section of your account or project.
3. Create a new secret API key.
4. Copy the key and store it securely.

Your API key is a secret.

Never:

- publish it on GitHub
- include it inside the workflow JSON
- put it inside the README
- send it to another user
- expose it in screenshots

#### Connect OpenAI to n8n

In n8n:

1. Open an OpenAI node used by the workflow.
2. Open **Credential to connect with**.
3. Create a new OpenAI credential.
4. Paste your own OpenAI API key into the required API key field.
5. Save the credential.
6. Select this credential in the OpenAI nodes used by the workflow.

Repeat this for any OpenAI node that is not already using the credential.

Make sure your OpenAI API account has access to the model configured in the workflow and sufficient API usage capacity for testing.

---

### 3. Anthropic Claude Credential

The workflow uses Anthropic Claude as part of the AI summarization and reliability pipeline.

You must provide your own Anthropic API credentials.

#### Create an Anthropic API Key

1. Sign in to the Anthropic Console.
2. Open the API key section.
3. Create a new API key.
4. Copy the key.
5. Store it securely.

Do not publish the Anthropic API key or include it in the repository.

#### Connect Claude to n8n

In n8n:

1. Open the Claude/Anthropic node.
2. Open **Credential to connect with**.
3. Create a new Anthropic credential.
4. Enter your Anthropic API key.
5. Save the credential.
6. Select the credential in the Claude nodes used by the workflow.

Make sure your Anthropic account has access to the model configured in the workflow and sufficient API usage capacity.

The workflow may use Claude more than once because the reliability architecture includes retry logic.

Therefore, verify that all relevant Claude nodes use your own credential.

---

### 4. Notion Credential

The final processed email data is stored in a Notion database.

You need:

- a Notion account
- a Notion database
- permission for the integration to access that database
- a Notion credential connected to n8n

#### Create a Notion Connection

In Notion:

1. Open **Settings**.
2. Open **Connections**.
3. Create or configure a connection for API access.
4. Select the workspace that contains your database.
5. Configure the required permissions.
6. Obtain the integration/access token if required by your n8n credential method.
7. Store the token securely.

The connection must have sufficient permission to create content in the database used by the workflow.

#### Give the Connection Access to the Database

Creating a Notion connection alone is not enough.

The connection must also have access to the page or database used by the workflow.

Open the relevant Notion page/database and add the connection to it using Notion's connection/access settings.

Without database access, n8n may authenticate successfully but still fail when trying to create a database entry.

#### Connect Notion to n8n

In n8n:

1. Open the final **Notion** node.
2. Open **Credential to connect with**.
3. Create a new Notion credential.
4. Complete the required authentication or enter the required integration token.
5. Save the credential.
6. Select your Notion database in the node.

Make sure the database properties match the fields expected by the workflow.

---

### 5. Verify All Credentials

Before executing the complete workflow, verify that every external service is connected.

You should have working credentials for:

```text
Gmail
OpenAI
Anthropic Claude
Notion
```

Open each external-service node and confirm that it uses your own credential.

Do not assume that importing the workflow automatically configures credentials.

The workflow JSON provides the automation logic, but every user must configure their own external accounts.

---

### 6. API Usage and Costs

OpenAI and Anthropic API usage may generate charges depending on the models, account, and amount of processing used.

API access and consumer chatbot subscriptions are separate services.

Before running large tests or processing real email traffic:

- review your API account settings
- review current API pricing
- configure appropriate spending limits where available
- monitor API usage
- start with a small number of test emails

This project does not provide API credits.

Each user is responsible for their own API accounts and usage costs.

---

### 7. Test Credentials Before Production Use

Do not immediately activate the workflow for a busy Gmail inbox.

First perform a controlled test.

Recommended sequence:

```text
Connect Gmail
      ↓
Connect OpenAI
      ↓
Connect Anthropic Claude
      ↓
Connect Notion
      ↓
Verify Notion Database Access
      ↓
Send One Test Email
      ↓
Execute Workflow
      ↓
Inspect Every Node
      ↓
Verify Notion Output
      ↓
Activate Workflow
```

If a node fails, inspect that node before continuing.

Common causes include:

- missing credentials
- invalid or expired API keys
- insufficient API access
- incorrect Gmail authorization
- incorrect Gmail filter
- missing Notion database permissions
- incorrect Notion database selection
- unavailable AI model
- insufficient API balance or usage limits

---

### 8. Credential Security

Credentials should always remain private.

Never include real secrets in:

```text
README.md
workflow JSON files
GitHub commits
screenshots
example files
documentation
public issue reports
```

If an API key or access token is accidentally published, revoke or rotate it immediately through the corresponding service.

The public workflow in this repository is designed to contain workflow logic only.

Users are expected to supply their own credentials after importing the workflow.

## Reliability Design

This project deliberately adds reliability layers around probabilistic
AI output:

``` text
LLM Output
   |
   v
Validation
   |
   +---- valid --------------------+
   |                               |
   +---- invalid -> Retry           |
                      |             |
                      v             |
                  Validation        |
                      |             |
                      +-- invalid -> Fallback Model
                                      |
                                      v
                                  Validation
                                      |
                                      v
                                Normalization
```

The important design principle is:

> LLM output is treated as untrusted input until it has been parsed and
> validated.

## Security

The public workflow template does **not** include working credentials.

Before publishing an n8n export, review it for:

-   email addresses
-   credential references
-   API keys
-   access tokens
-   Notion database IDs and URLs
-   webhook IDs
-   instance-specific metadata
-   real customer or email content
-   personally identifiable information

Use synthetic test data in screenshots and documentation.

## Current Limitations

-   The deadline normalizer handles a deliberately limited set of
    relative date expressions.
-   AI classification and extraction can still make semantic mistakes
    even when the JSON schema is valid.
-   The current workflow focuses on important-email storage rather than
    handling the non-important branch.
-   Final failure handling after all AI validation attempts could be
    extended with an error database, alert, or dead-letter workflow.
-   Production deployments should add monitoring, cost tracking,
    rate-limit handling, and broader automated tests.

## Possible Future Improvements

-   Require ISO-8601 deadlines directly from the extraction model.
-   Add Gmail labels after successful processing.
-   Add an explicit dead-letter/error path.
-   Add Slack or email notifications for high-priority messages.
-   Track which model produced the successful result.
-   Store validation/retry metrics.
-   Add automated workflow tests.
-   Add human approval for sensitive actions.
-   Support attachments and document extraction.
-   Add multilingual date parsing.

## What This Project Demonstrates

This project demonstrates practical experience with:

-   event-driven automation
-   workflow orchestration
-   API integrations
-   LLM prompting
-   structured outputs
-   JSON parsing
-   schema-style validation
-   defensive programming
-   branching logic
-   retries
-   fallback strategies
-   multi-model AI systems
-   data normalization
-   database integration
-   security-conscious workflow sharing

Instead of relying on a single prompt and assuming the answer is
correct, the workflow treats AI as one component inside a controlled
software pipeline.

## Repository Structure

A simple repository structure can be:

```text
ai-email-intelligence-system/
├── README.md
├── LICENSE
├── workflow/
│   └── AI_Email_Intelligence_System_PUBLIC_FINAL.json
└── examples/
    └── example-email.md

## License

This project is licensed under the MIT License.

See the [LICENSE](LICENSE) file for details.

------------------------------------------------------------------------

Built as a portfolio project to explore reliable, production-style AI
workflow engineering with n8n.

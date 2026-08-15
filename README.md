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
normalization, and database storage.

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

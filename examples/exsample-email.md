# Example Email

This example demonstrates how the AI Email Intelligence System processes an incoming business email and converts it into structured information.

## Incoming Email

**Subject:** Urgent: Customer presentation must be completed by tomorrow

**From:** project.manager@example.com

**Body:**

Hi,

the presentation for our customer Müller GmbH must be completed by tomorrow at 13:00.

Please check the latest sales figures, update the Excel file, and send me the corrected version together with a short summary of the most important changes.

The final presentation is expected at 16:00.

Thanks.

---

## Expected AI Processing Result

```json
{
  "summary": "The customer presentation must be completed urgently. The latest sales figures need to be reviewed, the Excel file updated, and the corrected version sent together with a summary of the most important changes.",
  "priority": "high",
  "category": "work",
  "deadline": "2026-08-16T13:00:00",
  "action_items": [
    "Review the latest sales figures",
    "Update the Excel file",
    "Send the corrected Excel file",
    "Provide a summary of the most important changes"
  ]
}
```

## Expected Workflow Behavior

The workflow should:

1. Receive the email through the Gmail Trigger.
2. Remove duplicate messages.
3. Classify the email as important.
4. Generate structured data using the AI processing pipeline.
5. Validate the generated JSON.
6. Use the fallback AI provider if the primary processing fails.
7. Normalize the final result.
8. Create a new entry in the Notion database.

## Expected Notion Entry

The resulting Notion database entry should contain:

- **Email Subject:** Urgent: Customer presentation must be completed by tomorrow
- **Priority:** High
- **Category:** Work
- **Deadline:** August 16, 2026 at 13:00
- **Summary:** AI-generated summary of the email
- **Action Items:** The extracted tasks listed above

---

This example is provided for demonstration purposes only. All names, email addresses, and business information are fictional.

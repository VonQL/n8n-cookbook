# Automation ideas

A backlog of common office tasks that make good n8n templates, grouped by area and listed roughly from easiest to hardest within each group. Ideas that already exist as templates in this repo are marked **✅ Built** and link to their folder.

Each idea lists the n8n nodes it would most likely use. These are starting points, not fixed designs.

## Already built

| Template | What it covers |
|---|---|
| [Email opportunity auto-reply](../templates/email-opportunity-auto-reply/) | Auto-acknowledges genuine business opportunities and ignores ads, newsletters and spam |
| [Human-in-the-loop email reply](../templates/email-human-in-the-loop-reply/) | Drafts replies to sensitive emails and waits for your approval before sending |
| [Email triage: auto-reply or ask me first](../templates/email-triage-auto-reply-or-approval/) | Combines both: sorts every email into "needs me", "opportunity" or "ignore" |
| [Invoice and receipt PDFs to Google Sheets](../templates/invoice-receipt-pdf-to-sheets/) | Reads PDF attachments from Gmail and logs vendor, dates, tax and total as one row each |
| [Recurring report from Google Sheets](../templates/recurring-sheets-report/) | Weekly or monthly totals, breakdown and comparison emailed with a short AI summary |
| [Form submissions to documents](../templates/form-to-document/) | Fills a Google Docs quote, contract or certificate template from a form and emails the PDF |

## Email and communication

- **✅ Built: Auto-reply to opportunities and approve sensitive replies.** See the three templates above.
- **Morning inbox digest.** Summarize the last day's inbox into one digest sent to Telegram or Slack.
  *Nodes:* Schedule Trigger, Gmail (get many), Anthropic / AI Agent, Telegram or Slack.
- **Save attachments to Drive.** Save email attachments such as invoices and receipts to Google Drive, then log each one in a sheet.
  *Nodes:* Gmail Trigger (download attachments), IF / Filter, Google Drive (upload), Google Sheets (append row).
- **Follow-up reminders.** Remind you about sent emails that got no reply within X days.
  *Nodes:* Schedule Trigger, Gmail (search sent mail), Gmail (get thread), Code / Filter, Slack, Telegram or Gmail (send).

## Meetings and calendar

- **Meeting prep brief.** Before each meeting, send a brief with attendees, recent emails with them and the agenda.
  *Nodes:* Schedule Trigger, Google Calendar (get events), Gmail (search), Anthropic / AI Agent, Gmail or Slack.
- **Meeting notes to action items.** Turn meeting notes into action items and add them to a task app.
  *Nodes:* Google Drive Trigger or Webhook, Google Docs (get), Anthropic (structured output), Todoist, Asana, Notion or Trello.
- **Daily agenda and conflict alerts.** Send each morning's agenda and flag overlapping or back-to-back meetings.
  *Nodes:* Schedule Trigger, Google Calendar (get events), Code, Slack, Telegram or Gmail.

## Documents and data

- **✅ Built: Invoice and receipt extraction.** See [invoice-receipt-pdf-to-sheets](../templates/invoice-receipt-pdf-to-sheets/). Pull invoice or receipt data from PDFs into a spreadsheet for expense reports.
  *Nodes:* Gmail Trigger or Google Drive Trigger, Extract from File (PDF), Anthropic (structured output), Google Sheets.
- **✅ Built: Recurring reports.** See [recurring-sheets-report](../templates/recurring-sheets-report/). Generate weekly or monthly reports from Sheets or a database.
  *Nodes:* Schedule Trigger, Google Sheets / Postgres / MySQL, Code or Summarize, Anthropic (narrative), Gmail or Slack.
- **✅ Built: Form submissions to documents.** See [form-to-document](../templates/form-to-document/). Turn form submissions into documents such as contracts, quotes and certificates.
  *Nodes:* n8n Form Trigger or Google Forms / Typeform Trigger, Google Docs (copy template, replace text), Google Drive (export PDF), Gmail.

## Approvals and HR

- **Approval flows.** Run leave, expense and purchase requests through an approval step, reusing the human-in-the-loop pattern from [Human-in-the-loop email reply](../templates/email-human-in-the-loop-reply/).
  *Nodes:* n8n Form Trigger, Gmail or Slack (send and wait for approval), IF / Switch, Google Sheets (log decision).
- **New hire onboarding.** Create onboarding checklists and accounts for new hires.
  *Nodes:* Form Trigger or HR system Webhook, Google Workspace Admin, Slack (invite), Notion / Asana (create tasks), Gmail (welcome).
- **Timesheet reminders and payroll prep.** Remind people to submit timesheets and collect them for payroll.
  *Nodes:* Schedule Trigger, Google Sheets, Filter, Slack or Gmail, Code (totals).

## Sales and customers

- **Lead capture.** Capture leads from web forms or email into a CRM, then send a welcome email.
  *Nodes:* Webhook / Form Trigger or Gmail Trigger, Anthropic (extract fields), HubSpot / Pipedrive / Google Sheets, Gmail.
- **Support ticket routing.** Route support tickets to the right person and send an automatic acknowledgement, building on the [email triage](../templates/email-triage-auto-reply-or-approval/) template.
  *Nodes:* Gmail Trigger or helpdesk Webhook, Anthropic (classify), Switch, Slack, Gmail (acknowledge).
- **Overdue invoice reminders.** Send polite reminders for unpaid invoices past their due date.
  *Nodes:* Schedule Trigger, Google Sheets / Stripe / QuickBooks, Filter (overdue), Gmail.

## Suggested next templates

The **morning inbox digest** and **save attachments to Drive** are the easiest next steps, since both reuse the Gmail and Claude setup from the existing templates.

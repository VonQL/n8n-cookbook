# Automation ideas

A backlog of common office tasks that make good n8n templates, grouped by area and listed roughly from easiest to hardest within each group. Ideas that already exist as templates in this repo are marked **✅ Built** and link to their folder.

Each idea lists the n8n nodes it would most likely use. These are starting points, not fixed designs.

## Already built

| Template | What it covers |
|---|---|
| [Email opportunity auto-reply](../templates/email-opportunity-auto-reply/) | Auto-acknowledges genuine business opportunities and ignores ads, newsletters and spam |
| [Human-in-the-loop email reply](../templates/email-human-in-the-loop-reply/) | Drafts replies to sensitive emails and waits for your approval before sending |
| [Email triage: auto-reply or ask me first](../templates/email-triage-auto-reply-or-approval/) | Combines both: sorts every email into "needs me", "opportunity" or "ignore" |
| [Morning inbox digest](../templates/email-morning-inbox-digest/) | Sends one Telegram message each morning summarizing the last day's inbox |
| [Save email attachments to Drive](../templates/email-attachments-to-drive/) | Saves attachments to a Google Drive folder and logs each one in a sheet |
| [Follow-up reminders](../templates/email-follow-up-reminders/) | Reminds you on Telegram about sent emails that got no reply, with a suggested follow-up |

## Email and communication

- **✅ Built: Auto-reply to opportunities and approve sensitive replies.** See the first three templates above.
- **✅ Built: Morning inbox digest.** Summarize the last day's inbox into one digest sent to Telegram. See [email-morning-inbox-digest](../templates/email-morning-inbox-digest/).
- **✅ Built: Save attachments to Drive.** Save email attachments such as invoices and receipts to Google Drive, then log each one in a sheet. See [email-attachments-to-drive](../templates/email-attachments-to-drive/).
- **✅ Built: Follow-up reminders.** Remind you about sent emails that got no reply within X days. See [email-follow-up-reminders](../templates/email-follow-up-reminders/).

## Meetings and calendar

- **Meeting prep brief.** Before each meeting, send a brief with attendees, recent emails with them and the agenda.
  *Nodes:* Schedule Trigger, Google Calendar (get events), Gmail (search), Anthropic / AI Agent, Gmail or Slack.
- **Meeting notes to action items.** Turn meeting notes into action items and add them to a task app.
  *Nodes:* Google Drive Trigger or Webhook, Google Docs (get), Anthropic (structured output), Todoist, Asana, Notion or Trello.
- **Daily agenda and conflict alerts.** Send each morning's agenda and flag overlapping or back-to-back meetings.
  *Nodes:* Schedule Trigger, Google Calendar (get events), Code, Slack, Telegram or Gmail.

## Documents and data

- **Invoice and receipt extraction.** Pull invoice or receipt data from PDFs into a spreadsheet for expense reports.
  *Nodes:* Gmail Trigger or Google Drive Trigger, Extract from File (PDF), Anthropic (structured output), Google Sheets.
- **Recurring reports.** Generate weekly or monthly reports from Sheets or a database.
  *Nodes:* Schedule Trigger, Google Sheets / Postgres / MySQL, Code or Summarize, Anthropic (narrative), Gmail or Slack.
- **Form submissions to documents.** Turn form submissions into documents such as contracts, quotes and certificates.
  *Nodes:* n8n Form Trigger or Google Forms / Typeform Trigger, Google Docs (copy template, replace text), Google Drive (export PDF), Gmail.

## Approvals and HR

- **Approval flows.** Run leave, expense and purchase requests through an approval step, reusing the human-in-the-loop pattern from [Human-in-the-loop email reply](../templates/email-human-in-the-loop-reply/).
  *Nodes:* n8n Form Trigger, Gmail or Slack (send and wait for approval), IF / Switch, Google Sheets (log decision).
- **New hire onboarding.** Create onboarding checklists and accounts for new hires.
  *Nodes:* Form Trigger or HR system Webhook, Google Workspace Admin, Slack (invite), Notion / Asana (create tasks), Gmail (welcome).
- **Timesheet reminders and payroll prep.** Remind people to submit timesheets and collect them for payroll.
  *Nodes:* Schedule Trigger, Google Sheets, Filter, Slack or Gmail, Code (totals).

## Sales and customers

- **✅ Built: Lead capture.** Capture leads from web forms or email into a CRM, then send a welcome email. See [Lead capture to CRM](../templates/lead-capture-crm-welcome/).
  *Nodes:* Form Trigger, Gmail Trigger, Anthropic (extract fields), Google Sheets, Gmail.
- **✅ Built: Support ticket routing.** Route support tickets to the right person and send an automatic acknowledgement, building on the [email triage](../templates/email-triage-auto-reply-or-approval/) template. See [Support ticket routing](../templates/support-ticket-routing/).
  *Nodes:* Gmail Trigger, Anthropic (classify), Switch, Gmail (notify, acknowledge), Slack (optional).
- **✅ Built: Overdue invoice reminders.** Send polite reminders for unpaid invoices past their due date. See [Overdue invoice reminders](../templates/overdue-invoice-reminders/).
  *Nodes:* Schedule Trigger, Google Sheets, Code (overdue check), Gmail.

## Suggested next templates

The **morning inbox digest** and **save attachments to Drive** are the easiest next steps, since both reuse the Gmail and Claude setup from the existing templates.

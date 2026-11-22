# Overdue invoice reminders

**What it does:** Every weekday morning, reads your invoices from a Google Sheet and sends polite payment reminders for unpaid invoices that are past their due date. Each invoice gets up to three reminders, each a little firmer:

| Reminder | Sent when (default) | Tone |
|---|---|---|
| 1 | 1 day overdue | Friendly: "a friendly reminder that invoice INV-104 was due on..." |
| 2 | 7 days overdue | Follow-up: asks when you can expect payment |
| 3 | 14 days overdue | Final: asks them to pay or say what's wrong |

After the last reminder it stops emailing the client and tells you the invoice **needs a personal follow-up**. You also get one summary email per run listing what was sent, what needs you, and any rows in the sheet that need fixing. On days with nothing to report, nothing is sent.

**It starts in test mode.** While `dryRun` is `true`, every reminder goes to **you** (with the client's address in the subject) and the sheet is not changed. Set it to `false` when the reminders look right.

No AI is used. Reminders about money use fixed wording, so they can never invent an amount or a date.

**Trigger:** Schedule Trigger, Monday to Friday at 09:00 in the workflow's time zone.

**Credentials needed:**
- Google Sheets OAuth2 on `Read Invoices` and `Record Reminder`.
- Gmail OAuth2 on `Send Reminder` and `Email Me the Summary`. Reminders are sent from this account, so use the address your clients know.

## How it works

```
Every Weekday at 9:00 → Settings → Check Settings → Read Invoices → Find Due Reminders
                                                                       ├→ Reminder Due? ├─ yes → Send Reminder → Live Run? ├─ yes → Record Reminder
                                                                       │                │                                   └─ no  → Test Run (sheet unchanged)
                                                                       │                └─ no  → No Reminder Today
                                                                       └→ Build Summary → Email Me the Summary
```

**Find Due Reminders** looks at every row and skips invoices whose **Status** is `Paid`, `Void`, `Cancelled` or `Written off`, and invoices that aren't due yet. For each overdue invoice it checks how many reminders were already sent and when:

- the next reminder is due when the invoice is at least that many days overdue (`reminderDays`) **and** at least `minDaysBetween` days have passed since the last one. If the workflow was off for a while, it catches up one reminder at a time instead of sending all three at once.
- once all reminders have been sent, it's listed in your summary under "Needs a personal follow-up" every run until you mark it paid.
- a row with no valid due date or email address is listed under "Rows to fix".

After a real (not test) reminder goes out, **Record Reminder** updates **Reminders sent** and **Last reminder** on that row.

## Setup

> **Before the first run, fill in `ownerEmail`, `signature`, `companyName` and `sheetId` in Settings.** Until they are set, **Check Settings** stops the run with an error naming the missing fields.

1. Create a Google Sheet with a tab named **Invoices** and this header row (exact spelling, any order):

   | Invoice # | Client | Contact name | Email | Amount | Currency | Issue date | Due date | Status | Payment link | Reminders sent | Last reminder | Notes |
   |---|---|---|---|---|---|---|---|---|---|---|---|---|

   - **Due date**: a real date cell (type `2026-11-30`, or pick from the date picker). Dates are read as serial numbers, so day and month can never be swapped.
   - **Status**: leave empty or `Unpaid` until it's paid, then set `Paid`.
   - **Amount** and **Currency**: e.g. `1200` and `USD`. A 3-letter currency code is formatted properly (`$1,200.00`).
   - **Contact name** is used in the greeting (falls back to **Client**). **Payment link** is optional (Stripe, PayPal, your invoicing tool). Leave **Reminders sent** and **Last reminder** empty; the workflow fills them in.
   - Each **Invoice #** must be unique, because it's used to find the row to update.
2. Import `workflow.json` into n8n (**Workflows → Import from File**).
3. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `ownerEmail` | Where the summary (and test reminders) go |
   | `signature`, `companyName` | Sign-off on every reminder |
   | `sheetId` | The long ID in the sheet's URL: `docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit` |
   | `sheetName` | Tab name, `Invoices` by default |
   | `reminderDays` | Days overdue for reminders 1, 2, 3..., comma separated. `1, 7, 14` by default. Add or remove numbers to change how many reminders are sent |
   | `minDaysBetween` | Minimum days between two reminders for the same invoice |
   | `dryRun` | `true` sends reminders to you and leaves the sheet alone. Set to `false` to go live |
4. Connect your Google Sheets and Gmail credentials.
5. Check the time zone: **Workflow settings → Timezone**. "Overdue" and the 09:00 schedule both use it.
6. Add a couple of test rows (one due yesterday, one 10 days ago, one already paid) and click **Execute workflow**. You should get the test reminders and a summary marked `[TEST]`.
7. When you're happy, set `dryRun` to `false`, delete the test rows and activate the workflow.

## Tuning

- **Wording:** edit `reminderText()` in **Find Due Reminders**. The three versions are clearly marked.
- **Different schedule:** change the cron expression on the trigger, e.g. `0 9 * * 1` for Mondays only.
- **Copy yourself on every reminder:** in **Send Reminder**, add **Options → BCC** with your address.
- **Stripe or QuickBooks instead of a sheet:** replace **Read Invoices** with the Stripe node (*Invoice*, open invoices) or the QuickBooks Online node (*Invoice → Get Many*), then map their fields to the column names **Find Due Reminders** expects (`Invoice #`, `Email`, `Amount`, `Currency`, `Due date`, `Status`...). Keep track of reminders sent in a sheet, or Stripe's own invoice reminders may be simpler there.
- **Want to approve the final reminder?** Put the approval steps from [email-human-in-the-loop-reply](../email-human-in-the-loop-reply/) in front of **Send Reminder** for stage 3.

## Troubleshooting

**No reminders and no summary:** nothing was due. Check the **Due date** cells are real dates (right-aligned in Sheets) and that **Status** isn't `Paid`.

**The same reminder is sent every day:** you're still in test mode (`dryRun` is `true`), which never updates the sheet. Or **Record Reminder** can't find the row: check the **Invoice #** column name and that each number is unique.

**Read Invoices fails with "Sheet with name ... not found":** check `sheetName` matches the tab exactly, and that the Google Sheets credential can open the file.

## Privacy note

Invoice data stays in your Google Sheet and Gmail. Nothing is sent to an AI service.

**Tested with n8n version:** built for n8n 1.x (Schedule Trigger 1.2, Google Sheets 4.5, Gmail 2.1, Code 2). Not yet run live; please report issues.

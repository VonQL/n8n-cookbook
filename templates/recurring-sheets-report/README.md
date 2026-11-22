# Recurring report from Google Sheets

**What it does:** Every Monday morning (or on the 1st of each month), reads a Google Sheet, totals last week's or last month's rows, compares them with the period before, breaks them down by a column such as *Category* or *Client*, and emails you a short report. Claude Haiku writes a 3 to 5 sentence summary on top; **all numbers come from the workflow's own code**, not from the AI.

Example email:

> **Weekly expenses report**
> Nov 9 to Nov 15, 2026
>
> Spending was USD 1,450.00 across 3 items, up 44.9% on the previous week. Software made up most of it at USD 1,150.00, led by one USD 1,200.00 licence from Acme...
>
> | Total | $1,450.00 |
> |---|---|
> | Items | 3 |
> | Previous period | $1,000.50 (1 items) |
> | Change | +44.9% |
>
> followed by **By group** (total, items, share, change vs previous) and **Largest items** tables.

It works with any sheet that has a date column and an amount column: expenses, sales, invoices sent, hours logged, orders. The defaults match the sheet made by [invoice-receipt-pdf-to-sheets](../invoice-receipt-pdf-to-sheets/), so the two work together out of the box.

**Trigger:** Schedule Trigger, every Monday at 8 AM in your n8n instance's time zone.

**Credentials needed:**
- Google Sheets OAuth2 on `Read Sheet`.
- Anthropic API on `Anthropic Chat Model`. Create an API key in the [Claude Console](https://console.anthropic.com/settings/keys) and paste it into a new **Anthropic** credential in n8n. Never put the key in the workflow itself.
- Gmail OAuth2 on `Send Report`.

## How it works

```
Every Monday 8 AM → Settings → Read Sheet → Calculate Report → Any Rows?
                                                                ├─ yes → Write Summary (AI) → Build Email → Send Report
                                                                └─ no  ─────────────────────→ Build Email → Send Report
```

1. **Read Sheet** reads every row of the tab (row 1 is the header).
2. **Calculate Report** finds the last *full* period. On Monday Nov 16 with `period` = `week` that is Mon Nov 9 to Sun Nov 15; with `month` on Dec 1 it is November. It then works out:
   - total, number of items and average for that period, and the same for the period before
   - change in percent
   - totals per group (`groupColumn`), each group's share and its own change
   - the 5 largest items, labelled with `labelColumn`
   - how many rows were skipped because their date or amount couldn't be read
3. **Write Summary** gets only those computed numbers and is told never to calculate or invent figures. If Claude is unavailable, the report is still sent, just without the summary.
4. **Build Email** puts the summary and tables into a simple HTML email, and **Send Report** sends it.

If no rows fall in the period, you still get an email saying so, which also tells you when the column settings are wrong.

## Setup

> **Before the first run, fill in `spreadsheetId` and `sendTo` in Settings.** Until both are set, **Calculate Report** stops the run with an error naming them. If a column name in Settings doesn't exist in the sheet, it stops with an error listing the sheet's real columns.

1. Import `workflow.json` into n8n (**Workflows → Import from File**).
2. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `reportTitle` | Email heading and subject, e.g. `Weekly sales report` |
   | `period` | `week` (Monday to Sunday) or `month` |
   | `spreadsheetId` | The long ID in the sheet's URL: `docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit` |
   | `sheetName` | Tab name |
   | `dateColumn` | Header of the date column, e.g. `Date` |
   | `amountColumn` | Header of the amount column, e.g. `Total` or `Amount` |
   | `groupColumn` | Header to break the total down by, e.g. `Category`, `Client` or `Product` |
   | `labelColumn` | Header used to name items in *Largest items*, e.g. `Vendor` or `Description` |
   | `currency` | ISO code used to format amounts, e.g. `USD`, `EUR`, `PHP` |
   | `dateOrder` | How to read dates like `03/04/2026`: `MDY` (US) or `DMY`. ISO dates (`2026-04-03`) and real Sheets dates always work |
   | `sendTo` | Who gets the report. Separate several addresses with commas |
3. Connect the Google Sheets, Anthropic and Gmail credentials.
4. Click **Execute workflow** once to get a report right away and check the numbers against your sheet.
5. For a monthly report, set `period` to `month` and change **Every Monday 8 AM** to *Months*, day *1*, 8 AM (and rename it).
6. Activate the workflow.

## Tuning

- **Different day or time:** edit **Every Monday 8 AM**. The report always covers the last full week or month before the run, whatever the day.
- **Amounts like `$1,234.50`, `(45.00)` or `1.234,50`** are understood. Negative values (refunds, credit notes) reduce the total.
- **More than one sheet or a database:** replace **Read Sheet** with a **Postgres**, **MySQL** or **Airtable** node that returns rows with the same column names. **Calculate Report** doesn't care where the rows came from.
- **Slack or Telegram instead of email:** swap **Send Report** for a Slack or Telegram node and send `{{ $('Write Summary').first().json.text }}` as the message.
- **Your own wording:** edit the **Write Summary** system message, e.g. to flag any group that grew more than 20%.
- **Cost:** one Claude Haiku call per report, with only the summary numbers (never the raw rows) sent to the AI. The template uses `claude-haiku-5-5`. If it isn't in your n8n model dropdown, switch the model field to **By ID** and type `claude-haiku-5-5`.

## Troubleshooting

**"Column ... not found in the sheet":** the column names in Settings must match the header row exactly, including capitals and spaces. The error lists the real ones.

**The report says no rows, but the sheet has data:** check that the dates fall in the last full period (not the current one), and that `dateOrder` matches how your dates are written. A high *rows skipped* count means the date or amount column can't be read.

**Wrong week:** the schedule and the week boundaries use the n8n instance's time zone. Set it in **Workflow settings → Timezone**.

**Tested with n8n version:** built for n8n 1.x (Schedule Trigger 1.2, Google Sheets 4.5, Code 2, Basic LLM Chain 1.5, Anthropic Chat Model 1.3, Gmail 2.1). Not yet run live; please report issues.

## Privacy note

Only the computed summary (totals, group names, the 5 largest items and their labels) is sent to Anthropic's API, not the whole sheet. By default, Anthropic does not use API inputs to train its models. Remove `labelColumn` from the **Calculate Report** output if item names are sensitive.

# Invoice and receipt PDFs to Google Sheets

**What it does:** Watches Gmail for emails with PDF attachments, reads each PDF, and uses Claude Haiku to pull out the invoice or receipt data into one row of a Google Sheet, ready for expense reports or your bookkeeper.

| Column | Example |
|---|---|
| Vendor, Invoice Number | `Acme Corp`, `INV-0042` |
| Invoice Date, Due Date | `2026-11-20`, `2026-12-20` |
| Currency, Subtotal, Tax, Total | `USD`, `1000`, `200`, `1200` |
| Category, Description | `software_subscriptions`, `Annual design tool licence` |
| Payment Status | `paid`, `unpaid` or `unknown` |
| Status | `ok`, or `check: ...` with the reason a person should look at it |

PDFs that are not invoices or receipts (brochures, contracts, tickets) are skipped. PDFs it can't read, and results it isn't sure about, are **still logged** with a `check:` status, so nothing goes missing quietly.

**Trigger:** Gmail Trigger, polls every minute for new mail with a PDF attachment (`has:attachment filename:pdf`), including invoices from `noreply` billing systems.

**Credentials needed:**
- Gmail OAuth2 on `New Email` and `Label as Processed`. Use the **same** credential on both: a Gmail message ID only exists in the mailbox that received it.
- Anthropic API on `Anthropic Chat Model`. Create an API key in the [Claude Console](https://console.anthropic.com/settings/keys) and paste it into a new **Anthropic** credential in n8n. Never put the key in the workflow itself.
- Google Sheets OAuth2 on `Add to Sheet`.

## How it works

```
New Email → Settings → Split PDFs → Read PDF Text → Prepare Text → Has Text?
                                                                     ├─ yes → Extract Invoice Data (AI) → Is Invoice or Receipt?
                                                                     │                                      ├─ yes → Build Row ─┐
                                                                     │                                      └─ no  → Not an Invoice
                                                                     └─ no  → Unreadable PDF ───────────────────────────────────┤
                                                                                                                                 ▼
                                                                                              Add to Sheet → Label as Processed
```

1. **Split PDFs** turns every PDF attachment into its own item (an email with three receipts gives three rows) and ignores logos, images and other files.
2. **Read PDF Text** uses n8n's built-in **Extract from File** node. No AI cost so far. Password-protected or broken PDFs don't stop the run.
3. **Has Text?** sends scanned PDFs (photos with no text layer) straight to **Unreadable PDF**, which logs the file with `check: no text found (scanned image?)`.
4. **Extract Invoice Data** makes one Claude Haiku call per PDF with a strict schema. It copies values exactly, uses `null` instead of guessing, writes dates as `YYYY-MM-DD` and amounts as plain numbers.
5. **Build Row** sets **Status** to `ok`, or to `check:` when the total, date or vendor is missing, confidence is below `minConfidence`, or the AI left a note (for example "totals don't add up").
6. **Add to Sheet** uses *Append or Update* on the **Key** column (`vendor|invoice number`), so the same invoice emailed twice updates its row instead of adding a duplicate.
7. **Label as Processed** adds a Gmail label so you can see which emails were logged. Optional: it continues quietly if the label isn't set.

## Setup

> **Before the first run, fill in `spreadsheetId` in Settings and create the header row.** Until `spreadsheetId` is set, **Split PDFs** stops the run with an error naming it.

1. Create a Google Sheet with a tab named `Expenses` and paste this **header row** into row 1 (column names must match exactly; order doesn't matter):

   ```
   Key	Status	Type	Vendor	Invoice Number	Invoice Date	Due Date	Currency	Subtotal	Tax	Total	Payment Status	Category	Description	Vendor Tax ID	Confidence	File Name	From	Email Subject	Received	Email Link
   ```

   (Copy the line above and paste it into cell A1. The tabs split it into columns.)
2. Import `workflow.json` into n8n (**Workflows → Import from File**).
3. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `spreadsheetId` | The long ID in the sheet's URL: `docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit` |
   | `sheetName` | Tab name, `Expenses` by default |
   | `minConfidence` | 0 to 1. Rows below it get `check: low confidence` |
   | `processedLabelId` | Optional. ID of a Gmail label such as `Invoice logged` |
   | `maxChars` | Longest PDF text sent to the AI. 15,000 characters covers most multi-page invoices |
4. Connect the Gmail, Anthropic and Google Sheets credentials. n8n may auto-select a different Gmail credential on import, so check both Gmail nodes.
5. Optional: create the Gmail label. To find its ID, add a temporary Gmail node with **Label → Get Many**, run it, and copy the `id` (looks like `Label_123...`).
6. Test before activating: from a second address, email yourself a real invoice PDF, a receipt, a non-invoice PDF and a photo-style scanned receipt. Run the workflow manually, then check the sheet and the **Status** column.
7. Activate the workflow.

## Tuning

- **Only some senders or a label?** Edit the search in **New Email → Filters → Search**, e.g. `has:attachment filename:pdf label:invoices` or `from:(billing@acme.com OR receipts@stripe.com)`.
- **Your own categories?** Edit the `category` list in **Invoice Schema** and in the **Extract Invoice Data** system message together.
- **Scanned receipts:** this template reads the PDF's text layer, which photos don't have. To read them too, replace the **Unreadable PDF** branch with n8n's **Anthropic** node (*Analyze Document*), which sends the PDF itself to Claude, or run the PDFs through an OCR service first.
- **Drive instead of Gmail:** swap **New Email** for a **Google Drive Trigger** (*File Created* in a folder) followed by **Google Drive → Download**, and change **Split PDFs** to read the downloaded file. Everything after that stays the same.
- **Save the PDFs too:** add a **Google Drive → Upload** node after **Split PDFs** and put its `webViewLink` in a new column.
- **Cost:** only PDFs with text reach the AI, text is cut to `maxChars`, and each PDF costs one Claude Haiku call. The template uses `claude-haiku-5-5`. If it isn't in your n8n model dropdown, switch the model field to **By ID** and type `claude-haiku-5-5`.

## Troubleshooting

**Add to Sheet fails with "Column names were updated" or columns stay empty:** the header row doesn't match. Paste the header line from Setup into row 1 again.

**Add to Sheet fails with "Sheet with name ... not found":** `sheetName` must match the tab name exactly.

**Nothing happens for a test email:** the trigger only sees mail that arrived after the last poll. Send a new email, or open **New Email** and click **Fetch Test Event**. Send tests from a second address: the search skips mail in your Sent folder, which includes emails you send to yourself.

**Amounts look wrong:** check the row's **Status** and **Confidence**, then the PDF. Invoices with several tax rates or several invoices in one PDF are the usual cause; the AI is told to lower its confidence for these.

**Tested with n8n version:** built for n8n 1.x (Gmail Trigger 1.2, Gmail 2.1, Extract from File 1, Google Sheets 4.5, Basic LLM Chain 1.5, Anthropic Chat Model 1.3). Not yet run live; please report issues.

## Privacy note

The text of every PDF attachment is sent to Anthropic's API for extraction. By default, Anthropic does not use API inputs to train its models. Invoices contain names, addresses and sometimes bank details, so check this fits your obligations, narrow the Gmail search to the senders you need, or swap in a self-hosted model (e.g. Ollama).

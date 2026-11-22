# Save email attachments to Drive and log them in a sheet

**What it does:** Whenever an email with an attachment arrives in Gmail, each file is saved to one Google Drive folder with a tidy, sortable name and logged as a row in a Google Sheet. Invoices, receipts, contracts and statements end up in one place without you dragging files around.

| Received | From | Subject | File | Type | Size (KB) | Drive link | Email link | Saved at |
|---|---|---|---|---|---|---|---|---|
| 2026-11-21 10:02 | Acme Ltd <billing@acme.com> | Your November invoice | `2026-11-21_Acme-Ltd_INV-123.pdf` | Invoice | 85 | link | link | 2026-11-21 10:03 |

Signature logos, inline icons and file types you don't want (calendar invites, `.exe`, and so on) are skipped.

**Trigger:** Gmail Trigger, polls every minute for new email with an attachment (read or unread, any label, not from you).

**Credentials needed:**
- Gmail OAuth2 on `New Email with Attachment` and `Label as Saved`. **Both must use the same Gmail credential.** A Gmail message ID only exists in the mailbox that received it, so a different credential on the label node fails with "Requested entity was not found".
- Google Drive OAuth2 on `Upload to Drive`.
- Google Sheets OAuth2 on `Log in Sheet`.

No AI is used, so there is no API cost. To read the amounts and dates inside invoices, pair this with an invoice extraction template.

## How it works

```
New Email with Attachment → Settings → Split Attachments → Upload to Drive → Log in Sheet → Label as Saved
```

1. **New Email with Attachment** downloads every attachment of each new email.
2. **Split Attachments** stops the run with a clear error while `driveFolderId` or `spreadsheetId` still say `REPLACE_ME`. It then turns each email into one item per attachment and keeps only:
   - files whose extension is in `allowedExtensions`, and
   - images at least `minImageSizeKB` in size (smaller ones are almost always logos and icons).

   It names each file `YYYY-MM-DD_Sender-Name_original-name.ext` and guesses a **Type** (`Invoice`, `Receipt`, `Statement`, `Quote`, `Contract` or `Other`) from the file name and subject, in English, Spanish, German, French and Italian.
3. **Upload to Drive** saves the file into your folder.
4. **Log in Sheet** appends one row with the details and links to both the file and the email.
5. **Label as Saved** (optional) adds a Gmail label such as `Saved to Drive`, so you can see at a glance which emails were handled. If the label isn't set up, this step is skipped without stopping the workflow.

## Setup

> **Before the first run, fill in `driveFolderId` and `spreadsheetId` in Settings.** Until both are set, **Split Attachments** stops the run with an error naming the missing fields.

1. Import `workflow.json` into n8n (**Workflows → Import from File**).
2. **Create the Drive folder**, e.g. `Email attachments`. Open it and copy the ID from the address bar: `https://drive.google.com/drive/folders/`**`1AbC...xyz`**.
3. **Create the Google Sheet.** Name the first tab `Attachments` and put these headers in row 1, exactly as written:

   ```
   Received | From | Subject | File | Type | Size (KB) | Drive link | Email link | Saved at
   ```

   Copy the spreadsheet ID from its address: `https://docs.google.com/spreadsheets/d/`**`1XyZ...abc`**`/edit`.
4. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `driveFolderId` | The folder ID from step 2 |
   | `spreadsheetId` | The spreadsheet ID from step 3 |
   | `sheetName` | The tab name (default `Attachments`) |
   | `myAddress` | Your Gmail address, so the email links open in the right Google account |
   | `allowedExtensions` | Comma-separated file types to keep. Leave empty to keep everything |
   | `minImageSizeKB` | Images smaller than this are skipped (default 20) |
   | `savedLabelId` | Optional. ID of a Gmail label such as `Saved to Drive` |
5. Connect your Gmail, Google Drive and Google Sheets credentials. Pick the **same** Gmail credential on both Gmail nodes. n8n may auto-select a different one on import, so check each node.
6. Optional: to find the label ID, add a temporary Gmail node with **Label → Get Many**, run it, and copy the `id` (looks like `Label_123...`).
7. Test: from another account, send yourself an email with a PDF and an image attached. Click **Execute workflow**, then check the folder and the sheet.
8. Activate the workflow.

## Tuning

- **Only invoices and receipts?** Narrow the search on **New Email with Attachment → Filters → Search** with Gmail terms, e.g. `has:attachment -from:me (invoice OR receipt OR factura)` or `has:attachment from:(@acme.com OR @stripe.com)`.
- **A folder per month?** Use separate tabs or folders per year, or sort the sheet by **Received**. File names start with the date, so Drive's name sort is already chronological.
- **Also save your own sent attachments?** Remove `-from:me` from the search.
- **Skip the label step?** Delete **Label as Saved**; nothing else depends on it.

## Troubleshooting

**Nothing is saved, and Split Attachments has no output:** the email had no attachment of an allowed type, or only small images. Check the binary tab on **New Email with Attachment**, then `allowedExtensions` and `minImageSizeKB`.

**Log in Sheet fails with "Sheet with name ... not found":** `sheetName` must match the tab name exactly, including capitals.

**Columns end up empty or in the wrong place:** the header row must use the exact names listed in step 3. Open **Log in Sheet** and click **Refresh** next to the columns if you renamed any.

**Label as Saved fails with "Requested entity was not found":** both Gmail nodes must use the same credential. The workflow still continues, so files are saved either way.

**The same file is saved twice:** n8n only remembers which emails it has seen while the workflow stays active. Deactivating and reactivating it, or running it by hand, can process recent emails again.

## Privacy note

Attachments are copied into your own Google Drive and nothing is sent to an AI provider. Check who the Drive folder and the sheet are shared with, since invoices and contracts often contain personal or financial data.

**Tested with n8n version:** built for n8n 1.x (Gmail Trigger 1.2, Gmail 2.1, Google Drive 3, Google Sheets 4.5). Not yet run live; please report issues.

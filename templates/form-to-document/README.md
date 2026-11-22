# Form submissions to documents (quotes, contracts, certificates)

**What it does:** Turns a form submission into a finished PDF. The form picks a document type, the workflow copies the matching Google Docs template, fills in its `{{placeholders}}`, exports it as a PDF and emails it.

| Document type | Typical use | Example template text |
|---|---|---|
| **Quote** | price quote for a project or service | *Quote {{doc_number}} for {{client_name}}, {{company}}. {{title}}: {{amount}}. Valid until {{date}}.* |
| **Contract** | service agreement or simple contract | *This agreement between {{sender_name}} and {{client_name}} starts on {{date}}. Scope: {{details}}. Fee: {{amount}}.* |
| **Certificate** | course completion, award, attendance | *This certifies that {{client_name}} completed {{title}} on {{date}}.* |

**By default the PDF is emailed to you for review**, with a link to the editable Google Doc, and nothing goes to the client. Set `sendToClient` to `true` to email the client directly, with you in CC.

**Trigger:** n8n Form Trigger. n8n hosts the form for you and gives you a URL to share or embed.

**Credentials needed:**
- Google Drive OAuth2 on `Copy Template` and `Export PDF`.
- Google Docs OAuth2 on `Fill Placeholders`.
- Gmail OAuth2 on `Email Client` and `Email Me to Review`.

No AI is used: the document text is your own template, so contracts say exactly what you wrote.

## How it works

```
Document Request Form → Settings → Prepare Document → Copy Template → Fill Placeholders → Export PDF → Send to Client?
                                                                                                        ├─ yes → Email Client (you in CC)
                                                                                                        └─ no  → Email Me to Review
```

1. **Prepare Document** picks the template for the chosen type, checks the client email, and builds every placeholder value: a document number such as `Q-20261122-7K3F`, dates written out (`December 1, 2026`), and the amount formatted in your currency (`$2,500.00`).
2. **Copy Template** makes a copy of the template in your output folder, named like `Quote Q-20261122-7K3F - Ana Santos`. The template itself is never changed.
3. **Fill Placeholders** replaces each placeholder in the copy.
4. **Export PDF** downloads the copy as a PDF, and one of the two Gmail nodes sends it as an attachment.

## Placeholders

Put these anywhere in your Google Docs templates, including tables, headers and footers. Each is replaced everywhere it appears. Unused ones are fine.

| Placeholder | Filled with |
|---|---|
| `{{doc_type}}` | `Quote`, `Contract` or `Certificate` |
| `{{doc_number}}` | e.g. `Q-20261122-7K3F` (`C-` for contracts, `CERT-` for certificates) |
| `{{client_name}}` | Client name from the form |
| `{{client_email}}` | Client email |
| `{{company}}` | Company (may be empty) |
| `{{title}}` | Project, service or course name |
| `{{amount}}` | Amount with currency, e.g. `$2,500.00` (empty if not given) |
| `{{date}}` | Date from the form, e.g. `December 1, 2026`: valid-until date for quotes, start date for contracts, completion date for certificates |
| `{{details}}` | The *Details* text from the form |
| `{{today}}` | The date the document was made |
| `{{sender_name}}` | `senderName` from Settings |

Placeholders must be typed exactly, with no spaces inside the braces, and in a single formatting run (if part of `{{client_name}}` is bold and part isn't, Google Docs may not match it; retype it).

## Setup

> **Before the first run, fill in `outputFolderId`, `myEmail`, `senderName` and the template ID for each document type you use** in Settings. Until they are set, **Prepare Document** stops with an error naming the missing fields. You only need templates for the types you use; remove the others from the form's dropdown.

1. In Google Docs, create a template for each document type you need, using the placeholders above. Design them however you like: logo, layout and fonts are kept.
2. Create a Google Drive folder for the finished documents.
3. Import `workflow.json` into n8n (**Workflows → Import from File**).
4. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `quoteTemplateId`, `contractTemplateId`, `certificateTemplateId` | The ID in each template's URL: `docs.google.com/document/d/`**`THIS_PART`**`/edit` |
   | `outputFolderId` | The ID in the folder's URL: `drive.google.com/drive/folders/`**`THIS_PART`** |
   | `myEmail` | Where review copies go, and the CC on client emails |
   | `senderName` | Your name or business name, for `{{sender_name}}` and the email sign-off |
   | `currency` | ISO code for `{{amount}}`, e.g. `USD`, `EUR`, `PHP` |
   | `sendToClient` | `false` (default): PDF comes to you to check and forward. `true`: PDF goes straight to the client |
5. Connect the Google Drive, Google Docs and Gmail credentials.
6. Open **Document Request Form**, copy the **Test URL**, click **Execute workflow**, open the URL and submit a test.
7. Check the PDF in your inbox and the Google Doc in your output folder.
8. Activate the workflow, then share the form's **Production URL**.

## Tuning

- **Keep the form private:** the form URL is public, and anyone who has it can create documents. For an internal tool, open **Document Request Form → Authentication → Basic Auth**. Leave `sendToClient` off unless you trust everyone who can open the form, since otherwise it would email whatever address is typed in.
- **Your own fields:** add a field to the form, add it to `values` in **Prepare Document** (e.g. `phone: clean(s['Phone'])`), and add a matching *Replace All Text* line to **Fill Placeholders** with `{{phone}}`.
- **More document types:** add the option to the form dropdown, a `...TemplateId` field to Settings, and the type to the two lookups at the top of **Prepare Document**.
- **Use your existing form tool:** replace **Document Request Form** with a **Google Forms** (via Google Sheets Trigger), **Typeform** or **Jotform** trigger, and rename the fields in **Prepare Document** to match.
- **Approve before sending:** keep `sendToClient` off and forward the review email, or add a Gmail **Send and Wait for Response** step before **Email Client**, as in [email-human-in-the-loop-reply](../email-human-in-the-loop-reply/).
- **Log every document:** add a **Google Sheets → Append** node after **Export PDF** with the document number, client, amount and Doc link.
- **Signatures:** for contracts, send the PDF through an e-signature service (DocuSign, Dropbox Sign, PandaDoc all have n8n nodes) instead of Gmail.

## Troubleshooting

**Copy Template fails with "File not found":** the template ID is wrong, or the Google Drive credential's account can't open the template. Share the template with that account.

**A placeholder is still visible in the PDF:** it was typed differently from the table above (spaces, different name) or split by formatting. Retype it in one go.

**Fill Placeholders shows `{{client_name}}` turned into an error:** the *Text to Replace* fields must stay in **Fixed** mode. In Expression mode n8n tries to evaluate `{{ }}`.

**The form opens but nothing happens on submit:** the Test URL only works while you've clicked **Execute workflow**. Use the Production URL once the workflow is active.

**Tested with n8n version:** built for n8n 1.x (Form Trigger 2.2, Google Drive 3, Google Docs 2, Gmail 2.1, Code 2). Not yet run live; please report issues.

## Privacy note

Form answers and the finished documents stay in your own n8n, Google Drive and Gmail. Nothing is sent to an AI service.

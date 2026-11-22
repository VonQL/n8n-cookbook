# Lead capture: form or email to CRM, with welcome email

**What it does:** Catches new sales leads from two places, saves them to a simple CRM in Google Sheets, tells you about each one and sends the lead a welcome email.

| Where the lead comes from | What happens |
|---|---|
| **Your web form** (n8n's built-in form, which you can link to or embed) | saved straight to the CRM, no AI needed |
| **Your inbox** | Claude Haiku decides whether the email is a real sales inquiry and pulls out the name, company, phone, interest, budget and timeline |

Then, for each lead:

- **New contact:** a row is added to the **Leads** sheet, you get a "New lead" email, and the lead gets a welcome email (a reply in the same thread for email leads).
- **Already in the sheet** (same email address): only **Last contact** and **Last message** are updated. No second welcome email.

**Triggers:** n8n Form Trigger (on submit) and Gmail Trigger (polls every minute for new unread inbox mail; Promotions, Social and Forums are excluded).

**Credentials needed:**
- Gmail OAuth2 on `New Email`, `Reply with Welcome`, `Label as Lead`, `Send Welcome Email` and `Notify Me`. **`Reply with Welcome` and `Label as Lead` must use the same credential as `New Email`**: a Gmail message ID only exists in the mailbox that received it, so a different credential fails with "Requested entity was not found".
- Google Sheets OAuth2 on `Read Existing Leads`, `Add to CRM` and `Update Last Contact`.
- Anthropic API on `Anthropic Chat Model`. Create a key in the [Claude Console](https://console.anthropic.com/settings/keys) and paste it into a new **Anthropic** credential in n8n. Never put the key in the workflow itself.

## How it works

```
Website Form ─┐
              ├→ Settings → Prepare Lead → Route by Source
New Email ────┘                              ├─ Skip  → Skipped (bulk, automated, reply, bad address)
                                             ├─ Form  ───────────────────────────────┐
                                             └─ Email → Extract Lead (AI) → Is a Lead? ─┤ (no → Not a Lead)
                                                                                       ▼
        Lead → Read Existing Leads → New or Known? → Is New?
                                                       ├─ new   → Add to CRM → Notify Me
                                                       │                     → Came by Email? ├─ yes → Reply with Welcome → Label as Lead
                                                       │                                      └─ no  → Send Welcome Email
                                                       └─ known → Update Last Contact
```

1. **Prepare Lead** turns both sources into one shape. For email it applies cheap rules first (no AI cost): it skips `noreply` and notification senders, mailing-list or bulk headers, your own mail, and replies or forwards (`Re:`, `Fwd:`), since those belong to conversations you already have.
2. **Extract Lead (AI)** makes one Claude Haiku call per email and returns `is_lead`, `confidence`, the contact details and a short `summary`. Vendors pitching to you, recruiters, support requests from existing customers, invoices, newsletters and spam are **not** leads. It never invents details; anything not in the email stays empty.
3. **Is a Lead?** keeps emails with `is_lead` true and `confidence` ≥ `minConfidence`.
4. **Lead** builds the record and the welcome email text. **New or Known?** reads the sheet once per run and matches on email address, so the same person filling in the form twice (or emailing after the form) never gets a second welcome.
5. The welcome email looks like this:

   > Hi Ana,
   >
   > Thanks for getting in touch about a new website for your bakery. I've received your message and will get back to you personally within 1 business day.
   >
   > If it's easier, you can book a short call at a time that suits you: https://cal.com/you
   >
   > Best regards,
   > Your Name

   The booking line only appears when `bookingLink` is set. Edit the text in the **Lead** node (`welcomeMessage`).

## Setup

> **Before the first run, fill in `myAddress`, `ownerEmail`, `signature` and `sheetId` in Settings.** Until they are set, **Prepare Lead** stops the run with an error naming the missing fields.

1. Create a Google Sheet with a tab named **Leads** and this header row (exact spelling, any order):

   | Email | Name | Company | Phone | Source | Interest | Budget | Timeline | Summary | Status | First contact | Last contact | Last message |
   |---|---|---|---|---|---|---|---|---|---|---|---|---|

   New leads get `Status` = `New`. Change it by hand as they move along (Contacted, Won, Lost...).
2. Import `workflow.json` into n8n (**Workflows → Import from File**).
3. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `myAddress` | Your Gmail address, so your own mail is never treated as a lead |
   | `ownerEmail` | Where "New lead" notifications go. Usually the same address |
   | `signature` | Name used to sign the welcome email |
   | `sheetId` | The long ID in the sheet's URL: `docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit` |
   | `sheetName` | Tab name, `Leads` by default |
   | `welcomeSubject` | Subject of the welcome email for form leads (email leads get a reply in their thread) |
   | `replyWithin` | Promised response time in the welcome email |
   | `bookingLink` | Optional Calendly, Cal.com or similar link. Leave empty to leave it out |
   | `minConfidence` | 0 to 1. Raise it for fewer, surer email leads |
   | `leadLabelId` | ID of a Gmail label such as `Lead`. To find it, add a temporary Gmail node with **Label → Get Many**, run it and copy the `id`. If you skip this, everything else still works |
4. Connect your Gmail, Google Sheets and Anthropic credentials, and check each Gmail node uses the right one (n8n may auto-select a different one on import).
5. Test:
   - Open **Website Form**, click **Test URL**, and submit the form with your own second address.
   - Send yourself a project inquiry from another account, plus a newsletter and a sales pitch. Run the workflow and check **Extract Lead (AI)** (`reason` says why each one was or wasn't a lead).
   - Check the sheet, the notification and the welcome emails. Submit the form again with the same address: the row's **Last contact** should change and no second welcome email should arrive.
6. Activate the workflow, then share the form's **Production URL** (or embed it on your site).

## Tuning

- **Only one inbox address for sales?** Add `to:sales@yourdomain.com` to the **New Email** search.
- **Use a real CRM instead of Sheets:** replace **Read Existing Leads**, **Add to CRM** and **Update Last Contact** with the HubSpot (*Contact → Create or Update*), Pipedrive (*Person* and *Lead*) or Airtable nodes, keeping the same fields. HubSpot's create-or-update already de-duplicates on email.
- **Other web forms** (Typeform, Tally, Webflow, WordPress): add their n8n trigger, or a **Webhook** node, and feed it into **Settings** with fields named `Name`, `Email`, `Company`, `Phone` and `How can we help?` (or adjust **Prepare Lead**).
- **Want to see the welcome email before it goes?** Swap **Reply with Welcome** for the approval steps from [email-human-in-the-loop-reply](../email-human-in-the-loop-reply/).
- **Running the [email triage](../email-triage-auto-reply-or-approval/) template on the same inbox?** Both would answer the same inquiry. Use one of them for first-contact auto-replies, or narrow one of the Gmail searches.
- **Cost:** form leads cost nothing. Only email that passes the rule filter reaches the AI, bodies are cut to 6,000 characters, and each email is one Claude Haiku call (`claude-haiku-5-5`). If the model isn't in your dropdown, switch the field to **By ID** and type `claude-haiku-5-5`.

## Troubleshooting

**Reply with Welcome fails with "Requested entity was not found":** use the same Gmail credential as **New Email**. Also unpin stale data on **New Email**.

**Read Existing Leads fails with "Sheet with name ... not found":** check `sheetName` matches the tab name exactly, and that the Google Sheets credential can open the file.

**Every lead is treated as new:** the header in the sheet must be exactly `Email`.

**Form submissions show nothing in the sheet:** the form's **Test URL** only works while the editor is listening (click **Execute workflow** first). The **Production URL** only works once the workflow is active.

## Privacy note

Lead details are stored in your Google Sheet. Emails that pass the rule filter are sent to Anthropic's API for extraction; by default, Anthropic does not use API inputs to train its models. If you collect leads from the EU or UK, mention in your form how you use the data.

**Tested with n8n version:** built for n8n 1.x (Form Trigger 2.2, Gmail Trigger 1.2, Gmail 2.1, Google Sheets 4.5, Switch 3.2, Basic LLM Chain 1.5, Anthropic Chat Model 1.3). Not yet run live; please report issues.

# Support ticket routing with auto-acknowledgement

**What it does:** Watches your support inbox. Claude Haiku reads each email, decides which team should handle it and how urgent it is, then:

- emails the right person a short ticket summary with the original message (urgent tickets are flagged and also go to an escalation address),
- replies to the customer with an acknowledgement and a ticket number,
- ignores newsletters, notifications, sales pitches and spam.

| Team | Typical emails |
|---|---|
| `billing` | invoices, charges, refunds, payment methods, cancellations with money involved |
| `technical` | bugs, errors, outages, how-to questions, integrations |
| `account` | login and password problems, access, account changes, data export or deletion |
| `sales` | new purchases, quotes, upgrades, questions before buying |
| `general` | everything else, or when it is unclear |

Priority is `urgent`, `high`, `normal` or `low`. Only `urgent` (can't work at all, security problem, money taken wrongly, legal threat, hard deadline today) takes the urgent path. The AI is told to judge the real impact, not the word "URGENT" in a subject line.

This builds on [email triage: auto-reply or ask me first](../email-triage-auto-reply-or-approval/), which is for a personal inbox. Use this one for a shared support address.

**Trigger:** Gmail Trigger, polls every minute for new unread inbox mail (Promotions, Social and Forums are excluded).

**Credentials needed:**
- Gmail OAuth2 on `New Support Email`, `Alert Urgent`, `Notify Team`, `Send Acknowledgement` and `Label as Ticket`. **`Send Acknowledgement` and `Label as Ticket` must use the same credential as `New Support Email`**, signed in to the support inbox: a Gmail message ID only exists in the mailbox that received it, so a different credential fails with "Requested entity was not found".
- Anthropic API on `Anthropic Chat Model`. Create a key in the [Claude Console](https://console.anthropic.com/settings/keys) and paste it into a new **Anthropic** credential in n8n. Never put the key in the workflow itself.
- Optional: Slack on `Post to Slack` (the node is disabled until you turn it on).

## How it works

```
New Support Email → Settings → Prepare Ticket → Worth Checking?
                                                  ├─ no  → Skipped (bulk or automated)
                                                  └─ yes → Classify Ticket (AI) → Assign Ticket → Route Ticket
                                                                                                    ├─ Urgent       → Alert Urgent ─┐
                                                                                                    ├─ Normal       → Notify Team ──┴→ Post to Slack (optional) → Acknowledge?
                                                                                                    │                                                              ├─ yes → Send Acknowledgement → Label as Ticket
                                                                                                    │                                                              └─ no  → Follow-up (no new acknowledgement)
                                                                                                    └─ Not a ticket → Not a Ticket
```

1. **Prepare Ticket** (cheap rules, no AI cost) skips `noreply` and notification senders, mailing-list or bulk headers, and mail from the support address itself. It also gives each Gmail thread a ticket number such as `T-4F2A9C`, taken from the thread ID, so follow-ups in the same thread keep the same number.
2. **Classify Ticket (AI)** makes one Claude Haiku call that returns:
   | Field | Meaning |
   |---|---|
   | `is_support`, `is_spam` | whether this needs the support team at all |
   | `team`, `priority`, `sentiment` | routing, as above; sentiment is `positive`, `neutral`, `frustrated` or `angry` |
   | `topic` | a short phrase for the acknowledgement ("the failed card payment") |
   | `customer_ref` | an order, invoice or account number from the email, never invented |
   | `summary`, `reason` | what the customer needs, and why it went to this team and priority |
3. **Assign Ticket** picks the team's address from Settings (empty addresses fall back to `generalEmail`) and writes both emails.
4. **Alert Urgent** / **Notify Team** email the assignee. The subject reads like `[URGENT] [T-4F2A9C] technical, urgent: Site is down`, and the body has the summary, the reason, a link to the thread in Gmail and the original message.
5. **Send Acknowledgement** replies in the customer's thread:

   > Hi Cara,
   >
   > Thanks for contacting Acme. We've received your message about the site outage and logged it as ticket T-4F2A9C.
   >
   > We've marked it as urgent, and someone from our team will get back to you within 4 business hours. If you have anything to add, just reply to this email.
   >
   > Best regards,
   > The Acme Support Team

   Follow-ups (`Re:`, `Fwd:`) are routed again so the assignee sees them, but they never get a second acknowledgement, which also keeps it from looping with another auto-responder.

## Setup

> **Before the first run, fill in `myAddress`, `companyName`, `signature` and `generalEmail` in Settings.** Until they are set, **Prepare Ticket** stops the run with an error naming the missing fields.

1. Import `workflow.json` into n8n (**Workflows → Import from File**).
2. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `myAddress` | The support inbox address, so its own mail is never treated as a ticket |
   | `companyName` | Used in the acknowledgement and the AI instructions |
   | `signature` | Sign-off on the acknowledgement, e.g. `The Acme Support Team` |
   | `generalEmail` | Gets `general` tickets, and any team without its own address |
   | `billingEmail`, `technicalEmail`, `accountEmail`, `salesEmail` | Optional. One address per team, or several separated by commas |
   | `escalationEmail` | Optional. Also gets every urgent ticket, e.g. a manager or on-call address |
   | `urgentResponseTime`, `normalResponseTime` | What the acknowledgement promises. Only promise what your team can keep |
   | `sendAcknowledgement` | Set to `false` to route tickets without replying to customers |
   | `ticketLabelId` | ID of a Gmail label such as `Ticket`. To find it, add a temporary Gmail node with **Label → Get Many**, run it and copy the `id`. If you skip this, everything else still works |
   | `slackChannel` | Channel for the optional Slack post, e.g. `#support` |
3. Connect your Gmail and Anthropic credentials, and check each Gmail node uses the right one (n8n may auto-select a different one on import).
4. Optional Slack: select **Post to Slack**, press **D** (or right-click → **Activate**) to enable it, and connect a Slack credential. To use Slack *instead of* email notifications, disable **Alert Urgent** and **Notify Team** the same way; disabled nodes pass the ticket straight through.
5. Test before activating: from another account, send a billing question, a "the app is down and we can't work" email, a password problem, a newsletter and a sales pitch. Run the workflow manually, then check the team emails, the acknowledgements and `reason` on **Classify Ticket (AI)**.
6. Activate the workflow.

## Tuning

- **Only one address?** If the support address forwards into a bigger inbox, add `to:support@yourdomain.com` to the **New Support Email** search.
- **Different teams:** change the `team` list in the **Classify Ticket (AI)** system message and in **Ticket Schema**, then add a matching address in **Settings** and in the `teamEmail` map in **Assign Ticket**.
- **Treat angry customers as urgent:** in **Assign Ticket**, change `const urgent = ai.priority === 'urgent';` to also include `|| ai.sentiment === 'angry'`.
- **Faster urgent alerts:** swap **Alert Urgent** for a Telegram, Slack, Microsoft Teams or SMS (Twilio) node.
- **Log tickets in a sheet or helpdesk:** add a Google Sheets *Append Row* (or a Zendesk, Freshdesk or HubSpot ticket node) after **Assign Ticket**.
- **Wording:** edit `ackMessage` and `notifyBody` in **Assign Ticket**. The acknowledgement is in English; translate it there if your customers write in another language.
- **Cost:** only mail that passes the rule filter reaches the AI, bodies are cut to 6,000 characters, and each email is one Claude Haiku call (`claude-haiku-5-5`). If the model isn't in your dropdown, switch the field to **By ID** and type `claude-haiku-5-5`.

## Troubleshooting

**Send Acknowledgement fails with "Requested entity was not found":** use the same Gmail credential as **New Support Email**. Also unpin stale data on the trigger.

**The "Open in Gmail" link doesn't open the thread:** it only works for someone signed in to the support inbox, and `/u/0/` is the first account in their browser. Change it in **Assign Ticket** if needed.

**The acknowledgement goes out but nobody is notified:** check the team addresses in Settings, and that **Alert Urgent** and **Notify Team** are not disabled.

## Privacy note

Support emails that pass the rule filter are sent to Anthropic's API for classification; by default, Anthropic does not use API inputs to train its models. Ticket notifications contain the full original message, so send them only to people who should see customer data.

**Tested with n8n version:** built for n8n 1.x (Gmail Trigger 1.2, Gmail 2.1, Switch 3.2, Basic LLM Chain 1.5, Anthropic Chat Model 1.3, Slack 2.2). Not yet run live; please report issues.

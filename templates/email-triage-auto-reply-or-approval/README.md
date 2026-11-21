# Email triage: auto-reply or ask me first

**What it does:** One workflow that handles your whole Gmail inbox, with Claude Haiku sorting every email into one of three paths:

| Path | What kind of email | What happens |
|---|---|---|
| **Needs me** | payments, invoices, refunds, sensitive or personal data, contracts and legal matters, complaints, account security, commitments on price or deadlines, HR, press | Haiku drafts a careful reply and the workflow **waits for your approval** before sending |
| **Opportunity** | genuine work offers, client requests, partnerships, recruiters with a real role | a short, polite acknowledgement is sent automatically |
| **Ignore** | ads, newsletters, cold sales pitches, spam, personal mail, anything else | nothing |

If an email is both (for example a new client who also asks you to sign their contract), **Needs me wins**: you get the approval request and no automatic reply is sent.

This combines [email-opportunity-auto-reply](../email-opportunity-auto-reply/) and [email-human-in-the-loop-reply](../email-human-in-the-loop-reply/). Use one of those if you only want one behavior.

**Trigger:** Gmail Trigger, polls every minute for new unread inbox mail (Promotions, Social and Forums tabs are excluded).

**Credentials needed:**
- Gmail OAuth2 on all six Gmail nodes: `New Email`, `Ask for Approval`, `Send Approved Reply`, `Label as Human-approved`, `Send Auto-reply`, `Label as Auto-replied`. **They must all use the same Gmail credential**, signed in to the inbox you want to watch. A Gmail message ID only exists in the mailbox that received it, so a different credential on a reply node fails with "Requested entity was not found".
- Anthropic API (on both `Anthropic Chat Model` nodes). Create an API key in the [Claude Console](https://console.anthropic.com/settings/keys) and paste it into a new **Anthropic** credential in n8n. Never put the key in the workflow itself.

## How it works

```
Part 1 (every new email)
New Email → Settings → Prepare Email → Worth Checking?
                                         ├─ no  → Skipped (bulk or automated)
                                         └─ yes → Handle Each Email Separately (calls Part 2 once per email)

Part 2 (one email)
Handle One Email → Classify & Draft (AI) → Route Email
                                             ├─ Needs me    → First Draft → Pending Draft → Ask for Approval (waits) → Read Decision → Route Decision
                                             │                               ▲                                                     ├─ Send        → Send Approved Reply → Label as Human-approved
                                             │                               └──────── Revised Draft ← Redraft Reply (AI) ◄────────┤─ Redraft
                                             │                                                                                     ├─ Don't reply → Not Replying
                                             │                                                                                     └─ No answer   → Timed Out (left for you)
                                             ├─ Opportunity → Send Auto-reply → Label as Auto-replied
                                             └─ Ignore      → Ignored
```

1. **Prepare Email** (cheap rules, no AI cost) skips `noreply` and notification senders, mailing-list or bulk headers, and mail from your own address. Replies (`Re:`, `Fwd:`) still reach the AI, because an ongoing invoice or contract thread may need you, but they **never get the automatic reply**, so it can't loop with another auto-responder.
2. **Handle Each Email Separately** calls this same workflow once per email. n8n's "send and wait" waits for one response per run, so this gives every flagged email its own approval, and one unanswered approval never blocks the next email.
3. **Classify & Draft** makes one Claude Haiku call that returns:
   | Field | Meaning |
   |---|---|
   | `category` | `opportunity`, `marketing`, `spam`, `personal` or `other` |
   | `confidence`, `topic` | how sure it is of the category, and a short phrase for the auto-reply ("the website redesign project") |
   | `needs_human` | true if a person must review the reply |
   | `categories`, `risk` | why it needs you (`payment`, `invoice`, `legal_contract`, `complaint`...) and `low` / `medium` / `high` |
   | `is_spam` | junk or deceptive mail; never answered |
   | `summary`, `reason`, `warning` | what the sender wants, why it was routed this way, and any red flags |
   | `draft_reply` | a safe draft for the approval path: it never confirms payments, accepts terms, admits liability or repeats sensitive data, and uses `[placeholders]` instead of invented amounts or dates |
4. **Route Email** checks its rules in order and runs only the first match:
   1. **Needs me:** `needs_human` is true and it isn't spam.
   2. **Opportunity:** category is `opportunity`, not spam, `confidence` ≥ `minConfidence`, and it's a first contact (not a reply).
   3. **Ignore:** everything else.
5. **Opportunity path:** **Send Auto-reply** answers in the same Gmail thread:

   > Hi Ana,
   >
   > Thanks for reaching out about the website redesign project. I've received your message and will get back to you personally within 1-2 business days.
   >
   > Best regards,
   > Your Name

6. **Needs me path:** **Ask for Approval** emails you (`approverEmail`) the sender, subject, categories, risk, summary, any warning and the draft, with a **Review reply** button. The form lets you choose *Send reply* (with the reply box pre-filled so you can edit in place; leave it empty to send the draft as shown), *Redraft with AI using my notes* (you get a new approval email, as many rounds as you like), or *Don't reply*. If you don't answer within 3 days, nothing is sent.

Suspicious but plausible emails, such as an unexpected bank-detail change or an urgent payment request from "your CEO", are **not** dropped as spam. They go to the approval path with a warning, because those are the ones a human most needs to see.

## Setup

1. Import `workflow.json` into n8n (**Workflows → Import from File**).
2. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `myAddress` | Your Gmail address, so the workflow never handles your own mail |
   | `approverEmail` | Where approval requests go. Usually the same address |
   | `signature` | Name used to sign both auto-replies and AI drafts |
   | `replyWithin` | Promised response time in the auto-reply |
   | `minConfidence` | 0 to 1. Raise it for fewer, safer auto-replies |
   | `approvedLabelId` | ID of a Gmail label such as `Human-approved` |
   | `autoReplyLabelId` | ID of a Gmail label such as `Auto-replied` |
3. Connect your Gmail and Anthropic credentials. Pick the **same** Gmail credential on all six Gmail nodes. n8n may auto-select a different one on import, so check each node.
4. **Save the workflow.** **Handle Each Email Separately** calls the saved version of this workflow by its own ID (`{{ $workflow.id }}`), so save before the first test and after every change.
5. Create the two Gmail labels. To find their IDs, add a temporary Gmail node with **Label → Get Many**, run it, and copy each `id` (looks like `Label_123...`). If you skip this, replies still send; the label steps continue on error.
6. Make sure your n8n instance is reachable at its public URL (`WEBHOOK_URL`). The **Review reply** button links back to n8n.
7. Test before activating: from another account, send yourself a job offer, an invoice, a "please update our bank details" email, a newsletter and an obvious scam. Run the workflow manually and check the executions list. Flagged emails show up as separate *waiting* executions until you answer. Check `reason` on **Classify & Draft** to see why each email went where it did.
8. Activate the workflow.

## Tuning

- **Too many approval emails?** Remove categories you're happy to handle yourself from the **Classify & Draft** system message, or add a `risk` condition to the *Needs me* rule in **Route Email** (e.g. only `high`).
- **Too many or too few auto-replies?** Change `minConfidence`, or tighten the category rules in the system message.
- **Wait time:** **Ask for Approval → Options → Limit Wait Time** (default 3 days). Remove it to wait forever.
- **Prefer to review auto-replies too?** Point the *Opportunity* output of **Route Email** at **First Draft** instead, and every reply goes through approval.
- **Missing invoices from billing systems?** Many come from `noreply` or bulk senders, which **Prepare Email** skips because they can't be replied to. Remove the `automatedSender` / `bulkHeaders` checks if you want them.
- **Cost:** only mail that passes the rule filter reaches the AI, bodies are cut to 6,000 characters, and each email costs one Claude call (plus one per redraft). The template uses `claude-haiku-5-5`, Anthropic's fastest and cheapest current model. If it isn't in your n8n model dropdown, switch the model field to **By ID** and type `claude-haiku-5-5`.

## Other approval channels

Email is the default because it needs no extra setup. n8n has the same **Send and Wait for Response** operation on Telegram (fastest on mobile), Slack, Discord, Microsoft Teams and WhatsApp Business nodes. Swap **Ask for Approval** for one of them and keep the form fields named **Decision**, **Reply to send** and **Notes for redraft** (or update **Read Decision**).

## Troubleshooting

**A reply node fails with "Requested entity was not found":** all six Gmail nodes must use the same Gmail credential. Also check the test email still exists, and unpin stale data on **New Email**.

**Handle Each Email Separately fails, or nothing happens after it:** save the workflow and run again.

**The Review reply link opens `localhost`:** set n8n's `WEBHOOK_URL` to its public address (see [n8n docs](https://docs.n8n.io/hosting/configuration/configuration-examples/webhook-url/)).

**Send yourself tests from a second address.** The trigger skips mail from you (`-from:me`), which also keeps your approval emails out of the workflow.

## Privacy note

Email content that passes the rule filter, including any sensitive details, is sent to Anthropic's API for triage and drafting. By default, Anthropic does not use API inputs to train its models. Because this workflow handles sensitive mail, check that this fits your own obligations, or swap in a self-hosted model (e.g. Ollama). Approval emails contain the summary and draft, so send them only to an inbox you trust.

**Tested with n8n version:** built for n8n 1.x (Gmail Trigger 1.2, Gmail 2.1 with Send and Wait, Execute Workflow 1.1, Switch 3.2, Basic LLM Chain 1.5, Anthropic Chat Model 1.3). Not yet run live; please report issues.

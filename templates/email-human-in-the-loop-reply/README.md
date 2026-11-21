# Human-in-the-loop email reply

**What it does:** Watches your Gmail inbox for emails that need a human: payments, invoices, refunds, sensitive or personal data, contracts and legal matters, complaints, account security and other cases where a wrong automatic answer could cost money or trust. For each one, Google Gemini drafts a careful reply, then the workflow **pauses and emails you** the draft. You approve it, edit it, ask the AI for a redraft, or decide not to reply. Nothing is sent to the original sender until you choose.

**Trigger:** Gmail Trigger, polls every minute for new unread inbox mail (Promotions, Social and Forums tabs are excluded).

**Credentials needed:**
- Gmail OAuth2 (on `New Email`, `Ask for Approval`, `Send Approved Reply`, `Label as Human-approved`). **All four must use the same Gmail credential**, signed in to the inbox you want to watch. A Gmail message ID only exists in the mailbox that received it, so a different credential on the reply node fails with "Requested entity was not found".
- Google Gemini (PaLM) API (on both `Google Gemini Chat Model` nodes). Create a free API key at [Google AI Studio](https://aistudio.google.com/app/apikey) and paste it into a new **Google Gemini(PaLM) Api** credential in n8n. Never put the key in the workflow itself.

## How it works

```
Part 1 (every new email)
New Email → Settings → Prepare Email → Worth Checking?
                                         ├─ no  → Skipped (bulk or automated)
                                         └─ yes → Handle Each Email Separately (calls Part 2 once per email)

Part 2 (one email)
Handle One Email → Check & Draft (AI) → Needs a Human?
                                          ├─ no  → No Human Needed
                                          └─ yes → First Draft → Pending Draft → Ask for Approval (waits) → Read Decision → Route Decision
                                                                     ▲                                                     ├─ Send        → Send Approved Reply → Label as Human-approved
                                                                     └──────── Revised Draft ← Redraft Reply (AI) ◄────────┤─ Redraft
                                                                                                                           ├─ Don't reply → Not Replying
                                                                                                                           └─ No answer   → Timed Out (left for you)
```

1. **Prepare Email** (cheap rules, no AI cost) skips mail that can't or shouldn't be answered: `noreply` and notification senders, mailing-list or bulk headers, and mail from your own address. Unlike the [auto-reply template](../email-opportunity-auto-reply/), replies (`Re:`) are kept, because an ongoing invoice or contract thread is exactly what needs you, and nothing is sent without your approval, so there is no reply loop.
2. **Handle Each Email Separately** calls this same workflow once per email. n8n's "send and wait" only waits for one response per run, so without this step two emails arriving in the same poll would share one approval. It also means an unanswered approval never blocks the next email.
3. **Check & Draft** asks Gemini, in one call, for:
   | Field | Meaning |
   |---|---|
   | `needs_human` | true if a person must review the reply |
   | `categories` | any of `payment`, `invoice`, `refund_dispute`, `sensitive_info`, `legal_contract`, `complaint`, `account_security`, `commitment`, `hr_personal`, `media_pr`, `other_judgment` |
   | `risk` | `low`, `medium` or `high` |
   | `is_spam` | obvious junk only; dropped |
   | `summary`, `reason` | what the sender wants, and why it was flagged |
   | `warning` | red flags, such as an unexpected bank-detail change or a fake-looking invoice |
   | `draft_reply` | a safe draft: it never confirms payments, accepts terms, admits liability or repeats sensitive data, and uses `[placeholders]` instead of invented amounts or dates |

   Suspicious but plausible emails (an urgent payment request from "your CEO") are **not** dropped as spam. They are sent to you with a warning, because those are the ones a human most needs to see.
4. **Ask for Approval** emails you (`approverEmail`) the sender, subject, categories, risk, summary, any warning and the draft, with a **Review reply** button. The button opens a form:
   - **Decision:** *Send reply*, *Redraft with AI using my notes*, or *Don't reply*
   - **Reply to send:** pre-filled with the draft so you can edit it in place. If your n8n version doesn't pre-fill it, leave it empty to send the draft as shown in the email.
   - **Notes for redraft:** what to change, e.g. "firmer tone, ask for the invoice number"
5. **Route Decision**:
   - *Send reply*: answers the original email in the same Gmail thread with your (possibly edited) text, then adds a label.
   - *Redraft*: Gemini rewrites the reply using your notes (and your edits, if any) and you get a new approval email. Repeat as often as you like.
   - *Don't reply*: stops. The email stays in your inbox.
   - *No answer in time*: after 3 days the wait ends and nothing is sent.

## Setup

1. Import `workflow.json` into n8n (**Workflows → Import from File**).
2. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `myAddress` | Your Gmail address, so the workflow never handles your own mail |
   | `approverEmail` | Where approval requests go. Usually the same address; use another one (e.g. an assistant or a separate inbox) if you prefer |
   | `signature` | Name the AI signs replies with |
   | `labelId` | ID of a Gmail label such as `Human-approved` (see below) |
3. Connect your Gmail and Google Gemini credentials on the nodes listed above. Pick the **same** Gmail credential on all four Gmail nodes. n8n may auto-select a different one on import, so check each node.
4. **Save the workflow.** **Handle Each Email Separately** calls the saved version of this workflow by its own ID (`{{ $workflow.id }}`), so it must be saved before the first test, and re-saved after every change.
5. Create a Gmail label (e.g. `Human-approved`). To find its ID, add a temporary Gmail node with **Label → Get Many**, run it, and copy the `id` (looks like `Label_123...`). If you skip this, the reply still sends; the label step continues on error.
6. Make sure your n8n instance is reachable at its public URL (`WEBHOOK_URL`). The **Review reply** button links back to n8n; on a local install without a public URL, the link only works on the same machine.
7. Test before activating: from another account, send yourself an invoice, a "please update our bank details" email, a complaint and a normal question. Run the workflow manually, and check the executions list: each flagged email shows up as a separate *waiting* execution until you answer.
8. Activate the workflow.

## Tuning

- **Wait time:** open **Ask for Approval → Options → Limit Wait Time** (default 3 days). Remove the option to wait forever.
- **Too many approval emails?** Remove categories you're happy to handle yourself from the **Check & Draft** system message, or add a condition on `risk` in **Needs a Human?** (e.g. only `high`).
- **Missing invoices from billing systems?** Many are sent from `noreply` or bulk senders and are skipped by **Prepare Email**. Those can't be replied to anyway; to be notified about them, remove the `automatedSender` / `bulkHeaders` checks.
- **Combine with the auto-reply template:** connect the `false` output of **Needs a Human?** to an automatic reply, so routine mail is answered automatically and risky mail waits for you.
- **Cost:** only mail that passes the rule filter reaches the AI, bodies are cut to 6,000 characters, and each email costs one Gemini call (plus one per redraft). `gemini-2.5-flash` is covered by Gemini's free tier at low volume. If that model name isn't offered in your n8n version, pick any current Flash model from the node's dropdown.

## Other approval channels

Email is the default here because it needs no extra setup: the approval request arrives in the same inbox, from your own Gmail account. n8n has the same **Send and Wait for Response** operation on other nodes, so you can swap **Ask for Approval** for one of them and keep the rest of the workflow:

- **Telegram:** fastest on mobile. Needs a bot token from @BotFather and your chat ID.
- **Slack:** good for teams; post to a private channel or DM.
- **Discord, Microsoft Teams, WhatsApp Business:** also support send-and-wait.

Keep the form fields named **Decision**, **Reply to send** and **Notes for redraft** (or update **Read Decision**), since that's what the rest of the workflow reads.

## Troubleshooting

**Send Approved Reply fails with "Requested entity was not found"**

1. **Credential mismatch (most common).** All four Gmail nodes must show the same Gmail credential.
2. **The test email is gone.** If you deleted it after fetching the test event, the ID no longer exists. Send a new one.
3. **Stale pinned data.** Unpin **New Email** and fetch a fresh event.

**Handle Each Email Separately fails, or nothing happens after it**

The workflow wasn't saved before running. Save it and run again.

**The Review reply link opens `localhost`**

Set n8n's `WEBHOOK_URL` to its public address (see [n8n docs](https://docs.n8n.io/hosting/configuration/configuration-examples/webhook-url/)).

**Send yourself tests from a second address.** The trigger skips mail from you (`-from:me`), which also keeps your approval emails out of the workflow.

## Privacy note

Email content that passes the rule filter, including any sensitive details in it, is sent to Google's Gemini API for triage and drafting. On Gemini's free tier, Google may use that content to improve its products; a paid (billing-enabled) key avoids this. Because this template deliberately handles sensitive mail, consider a paid key or a self-hosted model (e.g. Ollama). Approval emails contain the summary and draft, so send them only to an inbox you trust.

**Tested with n8n version:** built for n8n 1.x (Gmail Trigger 1.2, Gmail 2.1 with Send and Wait, Execute Workflow 1.1, Basic LLM Chain 1.5, Google Gemini Chat Model 1). Not yet run live; please report issues.

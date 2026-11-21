# Email opportunity auto-reply

**What it does:** Watches your Gmail inbox and sends a short, polite acknowledgement only when an email is a genuine business opportunity, work offer or other income-generating request. Ads, newsletters, upsells, cold sales pitches and spam are ignored.

**Trigger:** Gmail Trigger, polls every minute for new unread inbox mail (Promotions, Social and Forums tabs are excluded).

**Credentials needed:**
- Gmail OAuth2 (on `New Email`, `Send Reply`, `Label as Auto-replied`). **All three must use the same Gmail credential**, signed in to the inbox you want to watch. A Gmail message ID only exists in the mailbox that received it.
- Google Gemini (PaLM) API (on `Google Gemini Chat Model`). Create a free API key at [Google AI Studio](https://aistudio.google.com/app/apikey) and paste it into a new **Google Gemini(PaLM) Api** credential in n8n. Never put the key in the workflow itself. Any n8n chat model node works if you'd rather swap it for OpenAI, Anthropic, Ollama, etc.

## How it works

```
New Email → Settings → Prepare Email → Worth Classifying?
                                         ├─ no  → Skipped (bulk or automated)
                                         └─ yes → Classify Email (AI) → Genuine Opportunity?
                                                                         ├─ yes → Send Reply → Label as Auto-replied
                                                                         └─ no  → Not an Opportunity
```

1. **Prepare Email** (cheap rules, no AI cost) skips mail that is clearly not from a person writing to you directly:
   - `noreply`, `notifications@`, `newsletter`, bounce senders
   - mailing-list or bulk headers (`List-Unsubscribe`, `Precedence: bulk`, `Auto-Submitted`)
   - replies and forwards (`Re:`, `Fwd:`). It answers only first contact, so it never loops on its own replies.
   - mail from your own address
2. **Classify Email** asks Gemini for one category: `opportunity`, `marketing`, `spam`, `personal` or `other`, plus `is_spam`, a `confidence` from 0 to 1, a short `topic` and a one-line `reason`. The prompt explicitly treats "pay us to partner" pitches and "earn $X from home" offers as marketing or spam, not opportunities.
3. **Genuine Opportunity?** replies only when category is `opportunity`, `is_spam` is false, and confidence is at least `minConfidence` (default 0.8).
4. **Send Reply** answers in the same Gmail thread:

   > Hi Ana,
   >
   > Thanks for reaching out about the website redesign project. I've received your message and will get back to you personally within 1-2 business days.
   >
   > Best regards,
   > Your Name

## Setup

1. Import `workflow.json` into n8n (**Workflows → Import from File**).
2. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `myAddress` | Your Gmail address, so the workflow never replies to itself |
   | `signature` | Name used to sign replies |
   | `replyWithin` | Promised response time in the reply |
   | `minConfidence` | 0 to 1. Raise it for fewer, safer replies |
   | `labelId` | ID of a Gmail label such as `Auto-replied` (see below) |
3. Connect your Gmail and Google Gemini credentials on the nodes listed above. Pick the **same** Gmail credential on all three Gmail nodes. n8n may auto-select a different one on import, so check each node.
4. Create a Gmail label (e.g. `Auto-replied`). To find its ID, add a temporary Gmail node with **Label → Get Many**, run it, and copy the `id` (looks like `Label_123...`). If you skip this, the reply still sends; the label step is set to continue on error.
5. Test before activating: send yourself a few emails from another account (a job offer, a newsletter, an obvious scam) and run the workflow manually. Check the `reason` field on **Classify Email** to see why each was sorted the way it was.
6. Activate the workflow.

## Tuning

- **Too many replies?** Raise `minConfidence` to 0.9, or tighten the rules in the **Classify Email** system message.
- **Missing real opportunities?** Look at the `reason` on skipped items. Recruiters who use mailing tools may be caught by the `List-Unsubscribe` rule; remove that check in **Prepare Email** if that matters to you.
- **Prefer to review first?** Change **Send Reply** operation from *Reply* to *Create Draft* (Draft resource) to have replies wait in your Drafts.
- **Cost:** only mail that passes the rule filter reaches the AI, and bodies are cut to 4,000 characters. `gemini-2.5-flash` costs a fraction of a cent per email and is covered by Gemini's free tier at low volume. If that model name isn't offered in your n8n version, pick any current Flash model from the node's dropdown.

## Troubleshooting

**Send Reply fails with "The resource you are requesting could not be found / Requested entity was not found"**

Gmail can't find the message ID in the account that Send Reply is using. The ID is taken from the trigger output (`id`, mapped to `messageId` in **Prepare Email**), so check these:

1. **Credential mismatch (most common).** Open **New Email**, **Send Reply** and **Label as Auto-replied**. All three must show the same Gmail credential. If you have two Gmail credentials, for example "Gmail account" and "Gmail account 2", the trigger may read one inbox while the reply looks in another.
2. **The test email is gone.** If you deleted the email, or emptied it from Trash, after fetching the test event, the old ID no longer exists. Send a new test email, click **Fetch Test Event** on **New Email** again, and re-run.
3. **Stale pinned data.** If **New Email** has pinned output (pin icon) from another account or an older email, unpin it and fetch a fresh event.

**Send yourself tests from a second address.** The trigger skips mail from you (`-from:me`), and **Prepare Email** skips mail from `myAddress`.

## Privacy note

Email content that passes the rule filter is sent to Google's Gemini API for classification. On Gemini's free tier, Google may use that content to improve its products; a paid (billing-enabled) key avoids this. Use a self-hosted model (e.g. Ollama) if that is a concern.

**Tested with n8n version:** built for n8n 1.x (Gmail Trigger 1.2, Gmail 2.1, Basic LLM Chain 1.5, Google Gemini Chat Model 1). The classifier has been run live with Gemini; please report issues.

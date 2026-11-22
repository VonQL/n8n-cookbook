# Morning inbox digest (Telegram)

**What it does:** Every morning, Claude Haiku reads the email that arrived in your Gmail inbox over the last 24 hours and sends you **one Telegram message** that you can read in under a minute:

> ☀️ **Inbox digest, Monday 23 November**
> Two emails need a reply; the most urgent is Ana's question about Friday's deadline.
>
> ✍️ **Needs your reply** (2)
> • **Ana Lopez**: Asks if the redesign can still ship Friday, needs an answer today. *open*
> • **Mark Chen**: Wants to book a call next week about the Q1 retainer. *open*
>
> ❗ **Important** (1)
> • **Acme Billing**: Invoice #4411 for €1,200 is due 30 November. *open*
>
> 📄 **FYI** (3)
> • **Google**: New sign-in to your account from a Mac in Manila. *open*
> • ...
>
> 🗑️ 14 newsletters, promotions and notifications skipped.

Every line links straight to the email in Gmail. On a quiet morning you get a short "no new email" note instead (or nothing, if you delete that node).

**Trigger:** Schedule, every day at 07:30 (in the workflow's time zone).

**Credentials needed:**
- Gmail OAuth2 on `Get Recent Emails`, signed in to the inbox you want summarized.
- Anthropic API on `Anthropic Chat Model`. Create an API key in the [Claude Console](https://console.anthropic.com/settings/keys) and paste it into a new **Anthropic** credential in n8n. Never put the key in the workflow itself. If you already use the [email triage](../email-triage-auto-reply-or-approval/) templates, reuse the same Gmail and Anthropic credentials.
- Telegram API on `Send to Telegram` (a bot token, see Setup).

## How it works

```
Every Morning → Settings → Check Settings → Get Recent Emails → Build Email List → Any Emails?
                                                                                    ├─ yes → Write Digest (AI) → Format Digest → Send to Telegram
                                                                                    └─ no  → Nothing New ──────────────────────────┘
```

1. **Check Settings** stops the run with a clear error while `myAddress` or `telegramChatId` still say `REPLACE_ME`, and builds the Gmail search: `in:inbox -from:me after:<24 hours ago>` plus `extraQuery`.
2. **Get Recent Emails** fetches up to `maxEmails` messages matching that search, read or unread.
3. **Build Email List** numbers the emails and trims each body (1,500 characters, or 400 for newsletters and automated senders) so the AI call stays cheap.
4. **Write Digest** makes **one** Claude Haiku call for the whole inbox. It puts every email in one bucket (`needs_reply`, `important`, `fyi` or `ignore`) with a summary of at most 20 words, plus a one-line overview.
5. **Format Digest** turns that into the Telegram message. Links are built from Gmail's own thread IDs, never by the AI, so they always point to the right email. Any email the AI forgets to mention is listed under FYI, so nothing disappears silently. Long digests are split into several messages (Telegram's limit is 4,096 characters).

## Setup

> **Before the first run, fill in `myAddress` and `telegramChatId` in Settings.** Until both are set, **Check Settings** stops the run with an error naming the missing fields.

1. Import `workflow.json` into n8n (**Workflows → Import from File**).
2. **Create a Telegram bot** (2 minutes):
   1. In Telegram, open [@BotFather](https://t.me/BotFather), send `/newbot` and follow the prompts. It replies with a **bot token**.
   2. In n8n, create a **Telegram** credential with that token and pick it on **Send to Telegram**.
   3. Open a chat with your new bot and press **Start** (a bot can't message you until you do).
   4. Find your **chat ID**: send any message to [@userinfobot](https://t.me/userinfobot) and copy the `Id` it shows. For a group, add the bot to the group and use the group's ID (it starts with `-`).
3. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `myAddress` | Your Gmail address. Used for the "open" links, so they open in the right Google account |
   | `telegramChatId` | Where the digest is sent (step 2.4) |
   | `lookbackHours` | How far back to look. 24 for a daily digest; 72 on a Monday-only schedule |
   | `maxEmails` | Upper limit of emails per digest (default 60) |
   | `extraQuery` | Extra Gmail search terms. The default skips the Promotions and Social tabs; leave it empty to include them |
   | `language` | Language of the summaries, e.g. `English`, `Filipino`, `Spanish` |
4. Connect your Gmail and Anthropic credentials.
5. Set the time on **Every Morning**, and check the time zone under **Workflow → Settings → Timezone**. The schedule and the date in the digest both use it.
6. Click **Execute workflow** once and check Telegram.
7. Activate the workflow.

## Tuning

- **Prefer Slack?** Replace **Send to Telegram** with a **Slack → Send Message** node and use `{{ $json.text }}` as the text. Slack uses its own formatting, so in **Format Digest** change `<b>x</b>` to `*x*` and links to `<url|open>`, or simply remove the HTML tags.
- **Twice a day?** Add a second time to **Every Morning** and set `lookbackHours` to 12.
- **Only unread mail?** Add `is:unread` to `extraQuery`.
- **Different buckets?** Edit the bucket list in the **Write Digest** system message, and the `sections` list in **Format Digest**.
- **Cost:** one Claude Haiku call per day, whatever the number of emails. The template uses `claude-haiku-5-5`, Anthropic's fastest and cheapest current model. If it isn't in your n8n model dropdown, switch the model field to **By ID** and type `claude-haiku-5-5`.

## Troubleshooting

**Telegram says "chat not found":** open a chat with your bot and press **Start**, then check `telegramChatId`. Group IDs start with `-`.

**Telegram says "can't parse entities":** a character in a summary broke the HTML formatting. **Format Digest** escapes `<`, `>` and `&`, so this should not happen; if it does, please report the email that caused it. As a quick fix, remove `parse_mode` from **Send to Telegram → Additional Fields**.

**The digest is in the wrong time zone, or arrives at the wrong hour:** set **Workflow → Settings → Timezone**.

**The "open" links open the wrong Google account:** set `myAddress` to the exact address of the inbox.

## Privacy note

The sender, subject and the start of each email from the last day are sent to Anthropic's API to write the digest. By default, Anthropic does not use API inputs to train its models. The digest itself goes to Telegram, so send it only to a chat you control. If that doesn't fit your obligations, swap in a self-hosted model (e.g. Ollama) or add `-label:private`-style terms to `extraQuery`.

**Tested with n8n version:** built for n8n 1.x (Schedule Trigger 1.2, Gmail 2.1, Basic LLM Chain 1.5, Anthropic Chat Model 1.3, Telegram 1.2). Not yet run live; please report issues.

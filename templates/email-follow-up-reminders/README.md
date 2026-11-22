# Follow-up reminders for unanswered emails (Telegram)

**What it does:** Every morning, it looks through the emails **you sent** and finds threads where nobody has replied for `waitDays` days (3 by default). Claude Haiku drops the ones that never needed an answer, such as "thanks!" or an FYI, and writes a friendly follow-up for the rest. You get one Telegram message:

> ⏰ **2 sent emails still waiting for a reply**
>
> **Website redesign: final quote**
> To Ana Lopez · 6 days · *open thread*
> Waiting for: approval of the final quote
> *Suggested follow-up:*
> Hi Ana, just bringing this back to the top of your inbox. Did you get a chance to look at the quote? Happy to walk through it on a call if that helps. Best, Von
>
> **Workshop dates in March**
> To Mark Chen · 3 days · *open thread*
> ...

Copy the suggestion into a reply in Gmail, tweak it and send. Nothing is ever sent for you.

**Trigger:** Schedule, every day at 09:00 (in the workflow's time zone).

**Credentials needed:**
- Gmail OAuth2 on `Get Sent Emails` and `Get Thread`. Use the same credential on both.
- Anthropic API on `Anthropic Chat Model`. Create an API key in the [Claude Console](https://console.anthropic.com/settings/keys) and paste it into a new **Anthropic** credential in n8n. Never put the key in the workflow itself. If you already use the [email triage](../email-triage-auto-reply-or-approval/) templates, reuse the same Gmail and Anthropic credentials.
- Telegram API on `Send to Telegram` (a bot token, see Setup).

## How it works

```
Every Morning → Settings → Check Settings → Get Sent Emails → Unique Threads → Get Thread → Find Unanswered → Worth Chasing? (AI) → Build Reminder → Send to Telegram
```

1. **Check Settings** stops the run with a clear error while `myAddress`, `telegramChatId` or `signature` still say `REPLACE_ME`, and builds the Gmail search `in:sent newer_than:30d -label:no-follow-up`.
2. **Get Sent Emails** and **Unique Threads** list the threads you wrote in recently (up to `maxThreads`).
3. **Get Thread** loads each whole thread, so replies from the other side are seen too.
4. **Find Unanswered** (plain rules, no AI cost) keeps a thread only when:
   - the latest message (ignoring drafts) was sent **by you**,
   - it went to at least one real person (not just yourself, and not a `noreply` address), and
   - it's been exactly `waitDays` days, or a multiple of it if `repeatReminders` is on. With 3 days, you're reminded on day 3, 6, 9 and so on, not every single morning.
5. **Worth Chasing?** makes one Claude Haiku call per remaining thread. It reads your last message and returns `expects_reply` (did you ask something?), `waiting_for` (a short phrase), a `nudge` in the email's own language signed with your `signature`, and a one-line `reason`.
6. **Build Reminder** sends one Telegram message listing the threads that need a nudge, longest-waiting first. If nothing needs chasing, **no message is sent**.

## Setup

> **Before the first run, fill in `myAddress`, `telegramChatId` and `signature` in Settings.** Until all three are set, **Check Settings** stops the run with an error naming the missing fields.

1. Import `workflow.json` into n8n (**Workflows → Import from File**).
2. **Create a Telegram bot** (skip if you already have one, e.g. from the [morning inbox digest](../email-morning-inbox-digest/)):
   1. In Telegram, open [@BotFather](https://t.me/BotFather), send `/newbot` and follow the prompts. It replies with a **bot token**.
   2. In n8n, create a **Telegram** credential with that token and pick it on **Send to Telegram**.
   3. Open a chat with your new bot and press **Start**.
   4. Send any message to [@userinfobot](https://t.me/userinfobot) and copy the `Id` it shows. That's your chat ID.
3. Open **Settings** and fill in:
   | Field | Meaning |
   |---|---|
   | `myAddress` | Your Gmail address. Used to recognize your own messages and for the "open thread" links |
   | `telegramChatId` | Where reminders are sent |
   | `signature` | Name used to sign the suggested follow-ups |
   | `waitDays` | Days without a reply before you're reminded (default 3) |
   | `repeatReminders` | `true`: remind again every `waitDays` days. `false`: remind once only |
   | `lookbackDays` | How far back to look at sent mail (default 30). Older threads are dropped |
   | `maxThreads` | Upper limit of threads checked per run (default 50) |
   | `skipLabel` | Threads with this Gmail label are never chased (default `no-follow-up`) |
   | `includeNudges` | `false` to get only the list, without suggested follow-ups |
4. Connect your Gmail and Anthropic credentials.
5. Optional: create the Gmail label `no-follow-up`. Add it to any thread you don't want reminders for.
6. Check the time zone under **Workflow → Settings → Timezone**. Keep **Every Morning** at **once a day**: reminders are counted in whole days, so running it more often repeats them, and skipping days can miss one.
7. Click **Execute workflow** once. If you have no threads that are exactly 3 days old, temporarily set `repeatReminders` to `true` and `waitDays` to `1` to see some results, then change them back.
8. Activate the workflow.

## Tuning

- **Stop reminders for one thread:** give it the `no-follow-up` label in Gmail.
- **Different wait times per person?** Duplicate the workflow, or add a rule to **Find Unanswered**, e.g. `const wait = to.some(p => p.address.endsWith('@bigclient.com')) ? 1 : waitDays;`.
- **Prefer Slack or email?** Replace **Send to Telegram** with **Slack → Send Message** or **Gmail → Send**, using `{{ $json.text }}`. For Slack, remove the HTML tags in **Build Reminder**.
- **Want ready-made drafts in Gmail?** Add a **Gmail → Draft → Create** node after **Worth Chasing?** with the thread ID from `$('Find Unanswered').item.json.threadId` and the `nudge` as the message. Note that with `repeatReminders` on, this creates a new draft each time.
- **No AI?** Delete **Worth Chasing?** and connect **Find Unanswered** to **Build Reminder**, then remove `.filter(x => x.ai.expects_reply)` there. You'll get every unanswered thread, including the "thanks!" ones.
- **Cost:** one Claude Haiku call per thread that passes the rules, usually only a handful per day. The template uses `claude-haiku-5-5`. If it isn't in your n8n model dropdown, switch the model field to **By ID** and type `claude-haiku-5-5`.

## Troubleshooting

**No Telegram message arrives:** that's normal when nothing needs chasing. Open the last execution and check **Find Unanswered** (threads that matched the day rules) and **Worth Chasing?** (`reason` explains each decision).

**Telegram says "chat not found":** open a chat with your bot and press **Start**, then check `telegramChatId`.

**Threads where the other person did reply still show up:** they probably replied in a new thread or from a different address. Add the `no-follow-up` label to the old thread.

**Get Thread fails with "Requested entity was not found":** use the same Gmail credential on both Gmail nodes.

## Privacy note

The recipient names, subject and your last message in each candidate thread are sent to Anthropic's API to decide whether a follow-up is needed and to draft it. By default, Anthropic does not use API inputs to train its models. Reminders go to Telegram, so send them only to a chat you control.

**Tested with n8n version:** built for n8n 1.x (Schedule Trigger 1.2, Gmail 2.1, Basic LLM Chain 1.5, Anthropic Chat Model 1.3, Telegram 1.2). Not yet run live; please report issues.

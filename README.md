# DeadlineBot

A WhatsApp bot that tracks your assignment deadlines the way you'd actually text a friend about them — no app to open, no dashboard to check. Text it a deadline, it logs it. Text it "done," it marks it off. It nags you automatically as due dates close in, with escalating urgency, and then shuts up once you've submitted.

Built in [n8n](https://n8n.io/) on top of the WhatsApp Cloud API, OpenAI, and Google Sheets.

> Curious about the build itself — the wrong turns, the invisible-checkbox bug that ate an evening, the Meta webhook fight? Read [`docs/build-story.md`](docs/build-story.md).

## Why

University deadlines were scattered across Moodle and Google Classroom, and I never remembered to check either. I *do* check WhatsApp constantly — so instead of building another to-do app I'd forget to open, I built something that lives inside a habit I already have.

## How it works

DeadlineBot is two independent workflows sharing one Google Sheet as the source of truth.

```mermaid
flowchart TD
    subgraph Intake["Task intake & classification (WhatsApp-triggered)"]
        A[WhatsApp message in] --> B[Fetch pending tasks from Sheet]
        B --> C[LLM: classify intent → strict JSON]
        C --> D[Split into individual tasks/matches]
        D --> E{Route by intent}
        E -->|new_task| F[Log new task to Sheet]
        E -->|submission_update| G[Mark task(s) submitted]
        E -->|query| H[Answer from pending tasks]
        E -->|unclear| I[Ask for clarification]
        F --> J[Confirm on WhatsApp]
        G --> J
    end

    subgraph Reminder["Reminder engine (schedule-triggered, runs hourly)"]
        K[Hourly schedule] --> L[Fetch pending tasks from Sheet]
        L --> M[Decide who's due for a nag]
        M --> N[Send WhatsApp template reminder]
        N --> O[Stamp Last Reminder Sent]
    end

    B -. reads/writes .-> Sheet[(Google Sheet)]
    F -. writes .-> Sheet
    G -. writes .-> Sheet
    L -. reads .-> Sheet
    O -. writes .-> Sheet
```

### 1. Task intake & classification
Every incoming WhatsApp message is sent, along with your current pending-task list, to an LLM with a strict system prompt. It classifies the message into exactly one of four intents and returns structured JSON:

| Intent | What it means | What happens |
|---|---|---|
| `new_task` | One or more new assignments/quizzes/projects with due dates | Each task is logged as its own row |
| `submission_update` | Something was turned in | Matching pending task(s) are marked submitted |
| `query` | A question about deadlines ("what's due this week?") | Answered directly from the pending-task list |
| `unclear` | Doesn't confidently fit the above | Bot asks a clarifying question |

The bot understands natural, messy phrasing on purpose:
- **Multiple tasks in one text** ("comp networks, oop, and data structures due wednesday") get split into separate rows.
- **Vague submissions** ("all done," "all my quizzes are submitted," or just "done") are resolved against the pending list — by task type, by name, by "everything," or falling back to the single soonest-due task.
- **Batch confirmations collapse** — marking 5 tasks done in one message produces one WhatsApp reply, not five.

### 2. Reminder engine
Runs independently on an hourly schedule and checks every pending task against its due date:

- **> 48 hours out** → silent
- **Inside 48 hours** → reminded roughly once a day
- **Inside 24 hours** → reminded roughly every 7 hours
- **Overdue** → exactly **one** reminder, then it stops nagging for that task

Because proactive WhatsApp messages outside a 24-hour user-reply window must use an approved [WhatsApp Message Template](https://developers.facebook.com/docs/whatsapp/message-templates), reminders are sent via a pre-approved template rather than free text.

## Stack

- **[n8n](https://n8n.io/)** — workflow orchestration (self-hosted or cloud)
- **[WhatsApp Cloud API](https://developers.facebook.com/docs/whatsapp/cloud-api)** (Meta) — inbound trigger + outbound messages/templates
- **OpenAI** (via n8n's LangChain node) — intent classification with structured JSON output
- **Google Sheets** — the entire database: one sheet, one row per task

## Repo contents

```
workflow/deadline-bot.json   n8n workflow export, ready to import
docs/build-story.md          the unfiltered story of building this
README.md                    you are here
```

## Setup

The workflow file in this repo has all personal identifiers (phone numbers, sheet IDs) stripped out — you'll wire up your own accounts on import.

### 1. Prerequisites
- An n8n instance (self-hosted or [n8n Cloud](https://n8n.io/cloud/))
- A Meta developer app with the WhatsApp product added, plus a WhatsApp Business Account
- An OpenAI API key
- A Google account (for Sheets)

### 2. Google Sheet
Create a sheet with these columns: `Course Name`, `Assigned Task`, `Due Date`, `Submitted`, `Guidelines (if any)`, `Last Reminder Sent`.

> ⚠️ Don't use a checkbox column type for `Submitted`. An unchecked checkbox in Google Sheets isn't an empty cell — it's a literal `FALSE`, and it will break any "find the next empty row" logic. Use plain text `true`/`false`.

### 3. WhatsApp Cloud API
This is the fiddly part — see [`docs/build-story.md`](docs/build-story.md) for the full account of what tripped me up. In short, you'll need to:
1. Create a Meta app → add the WhatsApp product.
2. Subscribe your app to your WhatsApp Business Account (this isn't exposed in the setup UI — it's a manual `POST` in [Graph API Explorer](https://developers.facebook.com/tools/explorer/)).
3. Set up a Meta **System User** to generate a long-lived access token (the default token expires every 24 hours).
4. Register your n8n production webhook URL as the callback URL — Meta only allows one at a time, and switching between n8n's test and production listeners will silently point it at the wrong one.
5. Submit a **Message Template** (e.g. named `deadline_reminder`, in `en`) with three body parameters for task/course/due-date, and wait for Meta's approval — this is required for the proactive reminder half of the workflow.

### 4. Import the workflow
1. In n8n, import [`workflow/deadline-bot.json`](workflow/deadline-bot.json).
2. Reconnect credentials on each node: WhatsApp Trigger, WhatsApp (send), OpenAI, Google Sheets.
3. In the **Log New Task**, **Mark Task(s) Submitted**, **Fetch Pending Tasks**, and **Get row(s) in sheet** nodes, point `documentId` at your own sheet.
4. In every WhatsApp send node, set `phoneNumberId` to your WhatsApp number's ID and `recipientPhoneNumber` to your own number.
5. Update the template name in **Send Reminder on Whatsapp** to match whatever you got approved by Meta.
6. Activate the workflow.

### 5. Test
Text your WhatsApp number something like `"OOP assignment due friday 5pm"` and confirm it lands in the sheet. Then hand-edit a row's due date into the reminder window and trigger the schedule node manually to confirm reminders fire.

## Design notes

- **One structured LLM call instead of an autonomous agent.** I deliberately avoided an AI Agent node that calls tools and reasons freely. This bot needs to be dependable every single day of a semester — I didn't want to debug an opaque agent trace at 11pm before a deadline. One LLM call with a strict JSON schema, routed by an explicit Switch node, is slower to build but far easier to reason about when something breaks.
- **Google Sheets as the database.** Deliberately low-tech — this is a single-user tool, and a spreadsheet is trivial to inspect and hand-edit while debugging.

## Known limitations

- Single-user by design (one hardcoded recipient number) — not built for multi-tenant use.
- Reminder cadence is checked on an hourly poll, not real-time.
- Requires Meta's manual approval of a message template before reminders can be sent.

## License

[MIT](LICENSE)

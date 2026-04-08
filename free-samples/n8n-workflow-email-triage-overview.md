# Build an Email Triage Automation with n8n + Claude

**What you'll build:** An automation that monitors your inbox, uses Claude to classify and summarize incoming emails, and routes them to the right place — all without you touching them until a decision is needed.

This overview walks through the full logic. You can build it manually using the steps below, or grab the ready-to-import workflow JSON from the link at the bottom.

---

## What This Workflow Does

Every time a new email arrives in your inbox, the workflow:

1. Pulls the email content (subject + body)
2. Sends it to Claude with a classification prompt
3. Routes the email based on Claude's response:
   - **Action required** → creates a task in your project management tool (Notion, Todoist, etc.)
   - **Newsletter / low priority** → moves to a designated folder and marks as read
   - **Meeting request** → drafts a reply with your availability or routes to your calendar tool
   - **Needs reply** → drafts a response for your review and saves it as a draft

The result: you open your inbox and see only the emails that genuinely need your attention, each with a draft reply or next action already prepared.

---

## Step-by-Step Build Guide

### Step 1: Set Up Your Email Trigger

In n8n, create a new workflow and add an **Email Trigger (IMAP)** node (or a **Gmail Trigger** if you're using Gmail).

- Set it to poll every [5 minutes] or use a webhook-based trigger if your provider supports it
- Filter to trigger only on new, unread messages in your inbox
- In the output, you'll use: `subject`, `text` (the plain-text body), and `from`

### Step 2: Clean the Email Content

Add a **Code node** (JavaScript) after the trigger. Email bodies often contain quoted reply chains, footers, and HTML artifacts that waste tokens and confuse Claude.

```javascript
// Strip quoted reply chains and excessive whitespace
const body = $input.item.json.text || "";
const cleaned = body
  .split('\n')
  .filter(line => !line.trim().startsWith('>'))  // remove quoted lines
  .join('\n')
  .replace(/\s{3,}/g, '\n\n')                   // collapse whitespace
  .substring(0, 2000);                           // cap at 2000 chars

return [{ json: { ...$input.item.json, cleanedBody: cleaned } }];
```

This keeps your Claude API costs low and improves classification accuracy.

### Step 3: Send to Claude for Classification

Add an **HTTP Request node** configured for the Anthropic API.

- **Method:** POST
- **URL:** `https://api.anthropic.com/v1/messages`
- **Headers:** `x-api-key: YOUR_KEY`, `anthropic-version: 2023-06-01`, `content-type: application/json`

**Request body:**

```json
{
  "model": "claude-3-haiku-20240307",
  "max_tokens": 200,
  "messages": [{
    "role": "user",
    "content": "Classify this email into exactly one category: ACTION_REQUIRED, MEETING_REQUEST, NEEDS_REPLY, NEWSLETTER, or LOW_PRIORITY.\n\nThen write a one-sentence summary of what the email is about.\n\nRespond in this exact format:\nCATEGORY: [category]\nSUMMARY: [summary]\n\nEmail:\nFrom: {{$json.from}}\nSubject: {{$json.subject}}\nBody: {{$json.cleanedBody}}"
  }]
}
```

*Note: Claude Haiku is the right model here — it's fast, cheap, and more than capable for classification tasks. Save Claude Sonnet or Opus for the draft-reply step.*

### Step 4: Parse Claude's Response

Add another **Code node** to extract the category and summary from Claude's text output:

```javascript
const content = $input.item.json.content[0].text;
const categoryMatch = content.match(/CATEGORY:\s*(\w+)/);
const summaryMatch = content.match(/SUMMARY:\s*(.+)/);

return [{
  json: {
    ...$input.item.json,
    category: categoryMatch ? categoryMatch[1] : 'LOW_PRIORITY',
    summary: summaryMatch ? summaryMatch[1].trim() : 'No summary'
  }
}];
```

### Step 5: Route with a Switch Node

Add a **Switch node** with conditions on `category`:

- `ACTION_REQUIRED` → connect to a Notion/Todoist/Linear node that creates a task
- `MEETING_REQUEST` → connect to a Google Calendar node or a draft-reply node
- `NEEDS_REPLY` → connect to Claude again (this time with Sonnet) to draft a reply, then save as Gmail draft
- `NEWSLETTER` → connect to a Gmail node that labels the email and marks it read
- `LOW_PRIORITY` → connect to a Gmail node that archives or labels it

### Step 6: Draft Replies with Claude Sonnet

For emails in the `NEEDS_REPLY` branch, add a second **HTTP Request node** calling Claude:

Prompt structure:
```
You are drafting a professional email reply on behalf of [Your Name].

Original email:
From: {{$json.from}}
Subject: {{$json.subject}}
Body: {{$json.cleanedBody}}

Write a concise, professional reply. Keep it under 150 words. Do not include a subject line. Start with a greeting. End with "Best," and a signature placeholder.
```

Then pass the output to a **Gmail node** set to "Create Draft" — the draft lands in your drafts folder, ready for one-click review and send.

---

## Expected Results

In practice, a well-tuned version of this workflow will:

- Handle 70–85% of incoming email without you touching it
- Reduce inbox noise by auto-archiving newsletters and low-priority messages
- Give you pre-drafted replies for emails that need responses
- Surface only the genuinely important messages, with context

The whole workflow takes about 45–90 minutes to build from scratch. Or you can skip the setup entirely.

---

> **Get the ready-to-import workflow JSON**
> The n8n x Claude Automation Pack includes this complete email triage workflow (import it in one click), plus 4 more automations: lead enrichment, meeting notes summarizer, Slack digest builder, and content repurposing pipeline.
> **[Get the Automation Pack for $29.99 → payhip.com/openclaw](https://payhip.com/openclaw)**

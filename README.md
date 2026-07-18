# Lead Triage with Claude: Form Intake → Classification → Human Approval → Google Sheets

An n8n workflow template that turns raw inbound leads into classified, reply-drafted, human-approved records — without ever letting the model act on its own.

**Flow:** form submission → Claude classifies the lead and drafts a reply → a reviewer approves or rejects by email → approved leads (with the draft) are appended to a Google Sheet. Failures at the AI step retry automatically and alert Slack instead of dying silently.

## What it does

1. A public n8n form collects a lead (name, email, company, timeline, free-text request).
2. Claude (`claude-sonnet-5`, via the Anthropic Messages API) classifies the lead as `hot`, `warm`, `cold`, or `spam`, explains why in one sentence, and drafts a short personalized reply.
3. The classification and draft are emailed to a human reviewer. The workflow pauses until they click **Approve** or **Reject**.
4. On approval, the lead, classification, and draft are appended to a Google Sheet. On rejection, nothing is stored.
5. If the Claude call fails after 3 attempts, or returns output that doesn't match the expected JSON contract, a Slack alert fires with the error message. The execution stays in n8n's log and can be retried from the UI.

## Who it's for

Anyone getting inbound leads through a form who wants AI to do the sorting and first-draft work, but doesn't trust AI output to reach a customer or a CRM unreviewed. That's the correct instinct — this template is built around it.

## Node-by-node walkthrough

| Node | Type | What it does |
|---|---|---|
| Lead Intake Form | Form Trigger | Hosts the intake form. Swap for a Webhook node if leads come from your own site. |
| Normalize Lead Fields | Set | Renames form labels to stable keys (`lead_name`, `lead_email`, ...) and stamps `submitted_at`. The rest of the workflow never depends on form wording. |
| Claude: Classify Lead + Draft Reply | HTTP Request | POST to `https://api.anthropic.com/v1/messages` with model `claude-sonnet-5`. Auth via a Header Auth credential (`x-api-key`) — the key is never stored in the workflow JSON. **Retry on fail: 3 tries, 5 s apart.** On final failure, execution routes to the error branch instead of stopping. |
| Parse Claude Response | Code | Enforces the output contract: extracts `content[0].text`, parses it as JSON (tolerating stray markdown fences), and whitelists the category against `hot/warm/cold/spam`. Malformed output throws, which routes to the error branch. Merges the original lead fields back in. |
| Approval: Email Draft to Reviewer | Gmail (Send and Wait) | Emails the classification and draft to a reviewer with Approve/Reject buttons, then pauses the workflow until one is clicked. |
| Approved? | If | Branches on the reviewer's decision (`data.approved`). |
| Log Lead to Google Sheets | Google Sheets | Appends the full record: timestamp, lead fields, category, reasoning, draft reply, status. Swap for your CRM's node (HubSpot, Pipedrive, Airtable) — fields are already normalized. |
| Rejected: No Action | NoOp | Rejected leads end here. Replace with an archive or notification step if you want a paper trail of rejections. |
| Alert: Claude Step Failed | Slack | One message per failed run: which step failed and the error text, with a pointer to the n8n execution log. |

## Why the retry + human-approval pattern matters

**Retries.** LLM APIs return transient errors under load — 429 rate limits, 529 overloaded, occasional 5xx. A demo workflow that works on Tuesday and drops leads on Thursday isn't an automation, it's a liability. Three attempts with a 5-second gap absorbs the common transient failures without any custom code — it's a checkbox in n8n's node settings that most templates skip.

**Fail loud.** When retries are exhausted, the error branch posts to Slack instead of letting the execution fail silently. The difference between "we lost leads for two days" and "we got pinged at 14:03" is this one branch.

**Contract enforcement.** The Code node treats Claude's output as untrusted input: parse it, validate the category against a fixed list, and throw (→ alert) if it doesn't conform. Prompt output drifts; downstream nodes shouldn't discover that at write time.

**Human-in-the-loop.** The model produces a *draft* and a *recommendation* — a human produces the decision. Nothing the model writes reaches the sheet (or, if you extend this, the customer) without an explicit click. This is what makes AI automation deployable in workflows where a wrong reply has a cost.

**Injection-aware prompting.** The lead's free-text message goes into the prompt, and lead forms get adversarial input. The system prompt instructs Claude to treat the message as data rather than instructions, the parser constrains what the output can be, and the approval step is the backstop for anything that slips through.

## Setup

See [SETUP.md](SETUP.md) for step-by-step import and credential instructions. Short version: import `workflow.json`, create four credentials (Anthropic header auth, Gmail, Google Sheets, Slack), set the reviewer email and spreadsheet ID, run a test submission.

## Customization ideas

- Replace Google Sheets with your CRM's node — the normalized fields map directly.
- Replace the Gmail approval with Slack's Send-and-Wait for in-channel approvals.
- Add a Gmail "send" node after the If branch to actually send the approved draft to the lead — deliberately left out so the template has no external side effects out of the box.
- Tighten or extend the category list in both the system prompt and the parser (they must stay in sync).

---

Want this adapted to your stack — different CRM, different intake, different approval channel? Kelson Works builds fixed-price Claude automations; a multi-system workflow like this one is $1,850, delivered working in 14 days. Pick a package and fill the intake form — no sales calls.

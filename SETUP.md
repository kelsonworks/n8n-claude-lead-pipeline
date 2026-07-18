# Setup

Exact steps to import the workflow and configure credentials. Time required: about 20 minutes, most of it Google OAuth clicking.

## Prerequisites

- n8n **1.60 or later** (Cloud or self-hosted). The approval step uses the Gmail node's "Send and Wait for Response" operation, which older versions don't have.
- An **Anthropic API key** (console.anthropic.com → API Keys).
- A **Gmail account** for sending approval emails (and Google Cloud OAuth credentials if self-hosting n8n).
- A **Google Sheet** you can write to.
- A **Slack workspace** with permission to create an app (optional — you can swap the alert node for email).

## 1. Import the workflow

**Option A — from file:**
1. In n8n, go to **Workflows → Add workflow**.
2. Open the workflow menu (three dots, top right) → **Import from File**.
3. Select `workflow.json`.

**Option B — paste:**
1. Copy the entire contents of `workflow.json`.
2. On a blank workflow canvas, press `Ctrl+V` / `Cmd+V`.

After import you'll see red warning triangles on four nodes — that's expected. They're the credential placeholders you configure next.

## 2. Create credentials

### 2a. Anthropic API key (Header Auth)

The Claude call is a plain HTTP Request node authenticated with a header credential, so the key lives in n8n's encrypted credential store, never in the workflow JSON.

1. **Credentials → Add credential → Header Auth**.
2. Name (header name field): `x-api-key`
3. Value: your Anthropic API key.
4. Save. Call the credential something recognizable, e.g. `Anthropic x-api-key`.
5. Open the **Claude: Classify Lead + Draft Reply** node and select this credential under *Credential for Header Auth*.

The `anthropic-version: 2023-06-01` header is already set in the node — don't remove it, the API rejects requests without it.

### 2b. Gmail (approval emails)

1. **Credentials → Add credential → Gmail OAuth2**.
2. On n8n Cloud: click *Sign in with Google* and authorize. Self-hosted: follow n8n's Gmail OAuth guide (Google Cloud project → OAuth consent screen → client ID/secret → paste into n8n → authorize).
3. Open the **Approval: Email Draft to Reviewer** node, select the credential, and change `sendTo` from `reviewer@example.com` to the address that should approve leads.

### 2c. Google Sheets

1. Create a spreadsheet with a tab named `Leads` (or change the tab name in the node).
2. Put these headers in row 1, exactly:
   `Timestamp | Name | Email | Company | Timeline | Request | Category | Reasoning | Draft reply | Status`
3. **Credentials → Add credential → Google Sheets OAuth2** and authorize (same flow as Gmail).
4. Open the **Log Lead to Google Sheets** node, select the credential, and replace `REPLACE_WITH_YOUR_SPREADSHEET_ID` with your spreadsheet's ID (the long string in the sheet URL between `/d/` and `/edit`). Alternatively switch the Document field to *From list* and pick it.

### 2d. Slack (failure alerts)

1. Create a Slack app at api.slack.com/apps → **OAuth & Permissions** → add bot scope `chat:write` → install to workspace → copy the Bot User OAuth Token.
2. **Credentials → Add credential → Slack API**, paste the token.
3. Open the **Alert: Claude Step Failed** node, select the credential, and set the channel (default is `#alerts` — change it or create that channel, and `/invite` the bot into it).

Don't use Slack? Delete this node, add an email node in its place, and reconnect the two error-branch arrows to it.

## 3. Test run

1. Click **Test workflow** (bottom of the canvas).
2. Open the **Lead Intake Form** node and copy the *Test URL*. Open it in a browser and submit a realistic fake lead.
3. Watch the execution: the Claude node should return a classification, then the workflow will pause at the approval node.
4. Check the reviewer inbox — you'll get an email with the classification, the draft reply, and Approve/Reject buttons. Click **Approve**.
5. Verify a new row appeared in the `Leads` sheet with Status `approved`.
6. Repeat and click **Reject** — the run should end at *Rejected: No Action* with nothing written.

**Test the failure path:** temporarily break the Anthropic credential (e.g. change `x-api-key` to `x-api-key-broken`), submit a lead, and wait — the node retries 3 times about 5 seconds apart (roughly 15 seconds total), then the Slack alert fires. Restore the credential afterward.

## 4. Activate

1. Toggle the workflow to **Active**.
2. Use the form's *Production URL* (in the Form Trigger node) as your live intake link — the test URL only works while you're watching the canvas.

## 5. Tuning

| What | Where |
|---|---|
| Retry count / delay | **Claude: Classify Lead + Draft Reply** → Settings tab → *Retry On Fail*, *Max Tries*, *Wait Between Tries* |
| Approval timeout | **Approval: Email Draft to Reviewer** → Options → *Limit Wait Time* (e.g. auto-cancel after 48 h so runs don't pause forever) |
| Categories | The system prompt in the HTTP Request node's JSON body **and** the `allowed` array in **Parse Claude Response** — keep them in sync |
| Reply length / tone | The system prompt in the HTTP Request node |
| Model | Leave as `claude-sonnet-5` unless you have a reason to change it; if you do, update it in the JSON body expression |

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| 401 `authentication_error` from the Claude node | Wrong or missing API key in the Header Auth credential; header name must be exactly `x-api-key` |
| 404 `not_found_error` mentioning the model | Model id typo — it must be `claude-sonnet-5` |
| "Claude did not return valid JSON" alert | The model returned prose instead of the JSON contract; check the execution log for the raw output, tighten the system prompt if it recurs |
| Approval email never arrives | Gmail credential not authorized, or the email landed in spam; check the node's output in the execution view |
| Google Sheets 403 | The authorized Google account can't edit the target spreadsheet |
| Rows land in the wrong columns | Row-1 headers don't exactly match the column names in step 2c |

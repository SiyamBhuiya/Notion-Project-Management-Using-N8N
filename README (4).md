# 🤖 CEO Project Management Automation

> A voice-to-Notion automation pipeline built with a custom Gemini Gem, Google Drive, and n8n — so the CEO can log project updates just by talking.

---

## 🧠 How It Works — The Big Picture

This system lets you submit structured project updates using nothing but natural language. You talk (or type) to a **custom-trained Gemini Gem**, it converts your input into a strict JSON payload, drops it into Google Drive as a Doc file, and an n8n workflow picks it up and fans it out across Notion, Google Sheets, and Gmail — automatically.

```
You speak/type
     │
     ▼
┌─────────────────────────────┐
│   Custom Gemini Gem         │  ← Trained to output a specific JSON schema
│   (Google AI Studio)        │
└─────────────────────────────┘
     │  Creates a Google Doc named  "n8n_bridge_push"
     │  containing the JSON payload
     ▼
┌─────────────────────────────┐
│   Google Drive              │  ← Detects the new file (polls every minute)
└─────────────────────────────┘
     │
     ▼
┌─────────────────────────────┐
│   n8n Workflow              │  ← Reads, validates, routes, and writes
│   (Project_Management_Final)│
└─────────────────────────────┘
     │
     ├──▶ Notion Page (Meeting Log, Decision Log, Open Questions, Status Table)
     ├──▶ Google Sheets (CEO Executive Dashboard)
     ├──▶ Gmail (Open Questions digest + Delegation emails + Alerts)
     └──▶ Google Drive (Deletes the bridge file after processing)
```

---

## ✨ The Gemini Gem — Custom AI Input Layer

A **custom Gem** was created in [Google AI Studio](https://aistudio.google.com/) trained specifically to:

1. Accept natural language project updates from the CEO (typed or pasted)
2. Parse the intent and extract structured fields
3. Output a **single, strict JSON object** matching the workflow's expected schema
4. Create a **Google Doc** in the root of Google Drive named exactly `n8n_bridge_push`
5. Write the JSON payload as plain text content inside that document

When you say **"send it"**, the Gem creates the Doc — and Google Drive does the rest.

### JSON Schema the Gem Produces

```json
{
  "project_name": "Project Alpha",
  "update_type": "meeting_log",
  "content": "Discussed Q2 delivery scope with engineering leads.",
  "status": "On Track",
  "owner": "Jane Doe",
  "blocker": "Waiting on vendor confirmation",
  "next_milestone": {
    "task": "Finalize technical spec",
    "date": "2026-05-10"
  },
  "decisions": [
    "Approved budget increase to $50k",
    "Moved launch date to June"
  ],
  "open_questions": [
    "Who owns the compliance review?",
    "Is the staging environment ready?"
  ],
  "delegations": [
    {
      "to": "John Smith",
      "task": "Draft compliance checklist",
      "due_date": "2026-05-05"
    }
  ]
}
```

### Valid Field Values

| Field | Valid Values |
|---|---|
| `update_type` | `meeting_log`, `decision`, `status_update`, `decision_log` |
| `status` | `On Track`, `At Risk`, `Blocked` |
| `decisions` | Array of strings (can be empty `[]`) |
| `open_questions` | Array of strings (can be empty `[]`) |
| `delegations` | Array of `{ to, task, due_date }` objects (can be empty `[]`) |

---

## ⚙️ n8n Workflow — Node by Node

### Stage 1 — Trigger & Ingestion

| Node | Type | What It Does |
|---|---|---|
| **Google Drive Trigger** | Trigger | Polls the root Drive folder every minute for new Google Docs |
| **Search files and folders** | Google Drive | Looks for a file named `n8n_bridge_push` in root |
| **Download Doc File** | Google Drive | Downloads the raw Doc file by ID |
| **Get Document Content** | Google Docs | Reads the plain text content from the Doc |
| **Extract Plain Text** | Code (JS) | Parses the raw text, strips whitespace, and returns clean JSON |

### Stage 2 — Validation

| Node | Type | What It Does |
|---|---|---|
| **Validate & Normalise Input** | Code (JS) | Checks all required fields exist, validates `update_type` and `status` enums, normalises `next_milestone`, stamps a timestamp |
| **Input Valid?** | IF | Routes valid payloads forward; invalid payloads to the alert branch |
| **Alert: Invalid Payload** | Gmail | Sends an error email with the raw payload if validation fails, then deletes the bridge file |

### Stage 3 — Notion Lookup

| Node | Type | What It Does |
|---|---|---|
| **Search Notion for Project** | Notion | Full-text searches Notion for a page matching `project_name` |
| **Project Found in Notion?** | Code (JS) | Maps the search result to `{ found: true/false, page_id }` |
| **Filter** | Filter | Filters Notion results to exact project name match |
| **If** | IF | Guards against false-positive Notion matches |
| **Notion Page Found?** | IF | Routes to data-write path (found) or alert path (not found) |
| **Alert: Project Not Found** | Gmail | Notifies the CEO if the project doesn't exist in Notion |

### Stage 4 — Notion Writes (parallel branches)

All four branches fire simultaneously after `Get Page Blocks`:

**Branch A — Status Table**

| Node | What It Does |
|---|---|
| **Find Table Block ID** | Finds the second table block in the Notion page |
| **Get Table block** | Fetches all rows of that table via Notion API |
| **Get Row ID** | Maps field labels to row IDs and builds PATCH payloads |
| **Update Row** | PATCHes each row with fresh values (status, owner, milestone, blocker, etc.) |

**Branch B — Decision Log**

| Node | What It Does |
|---|---|
| **Find Decision Log Block ID** | Searches page blocks for a heading containing "decision log" |
| **Create decision json** | Builds Notion block children (bulleted list + timestamp + divider) |
| **Append Inside Decision Log Section** | PATCHes the block to append decision bullets |

**Branch C — Open Questions**

| Node | What It Does |
|---|---|
| **Get Open Question log** | Searches page blocks for a heading containing "open question" |
| **Create question json** | Builds bulleted list blocks for each question |
| **Append Inside Decision Log Section1** | Appends question bullets into the section |

**Branch D — Meeting Log**

| Node | What It Does |
|---|---|
| **Find Meeting Log Block ID** | Searches page blocks for a heading containing "meeting log" |
| **Create Meeting Log Body** | Builds a paragraph block with `▶` prefix + timestamp + divider |
| **Append Inside Meeting Log** | Appends the meeting note into the section |

### Stage 5 — Google Sheets

| Node | What It Does |
|---|---|
| **Read CEO Dashboard Sheet** | Reads all rows from the "Executive Dashboard V2" sheet |
| **Find Project Row in Sheet** | Fuzzy-matches the project name to a row index |
| **Row Found in Sheet?** | Routes to update (found) or alert (not found) |
| **Update CEO Dashboard Sheet** | Updates Status, Owner, Blocker, Next Milestone, Last Updated, Notes |
| **Alert: Project Not Found1** | Emails the CEO if the sheet row is missing |

### Stage 6 — CEO Email Digest

| Node | What It Does |
|---|---|
| **Has Open Questions?** | Skips the digest email if `open_questions` is empty |
| **Email CEO: Open Questions Digest** | Sends a formatted HTML email listing all open questions requiring a decision |

### Stage 7 — Delegation

| Node | What It Does |
|---|---|
| **Has Delegations?** | Skips if `delegations` is empty; notifies via email if truly empty |
| **Read Contacts Table** | Reads the CONTACT Google Sheet (Name → Email lookup) |
| **Resolve Delegation Emails** | Maps each delegation's `to` field to an email address from the contact sheet |
| **Valid Delegation?** | Routes valid email targets to send; invalid to error |
| **Send Delegation Email** | Sends a formatted task-assignment email to the delegate |
| **Delegation Error** | Alerts the CEO if a delegate's name wasn't found in the contact sheet |

### Stage 8 — Cleanup

Every terminal path ends with a **Delete a file** node that removes the `n8n_bridge_push` Doc from Google Drive, keeping the Drive root clean and preventing re-processing.

---

## 🔗 Integrations & Credentials Required

| Service | Credential Type | Used For |
|---|---|---|
| Google Drive | OAuth2 | Trigger, file search, download, delete |
| Google Docs | OAuth2 | Reading Doc content |
| Google Sheets | OAuth2 | CEO Dashboard + Contacts table |
| Notion | API Key | Page search, block reads, block writes |
| Gmail | OAuth2 | All outbound emails |

---

## 🗂️ External Resources

| Resource | Description |
|---|---|
| CEO Executive Dashboard (Google Sheet) | `Executive Dashboard V2` tab — one row per project |
| CONTACT (Google Sheet) | `Sheet1` tab — Name/Email lookup for delegation |
| Notion Workspace | One page per project with standard heading sections |

### Required Notion Page Structure

Each project page must have these headings (case-insensitive):

- A **table block** (second table on the page) with rows: Project Name, Owner, Status, Next Milestone, Target Date, Blocker, Last Updated
- A heading containing **"Decision Log"**
- A heading containing **"Open Questions"**
- A heading containing **"Meeting Log"**

---

## 🚨 Error Handling & Alerts

All errors send a Gmail alert to the CEO's inbox with the full raw payload for debugging, then delete the bridge file. Handled error cases:

| Scenario | Alert Sent |
|---|---|
| JSON payload is malformed or missing required fields | ✅ |
| Project not found in Notion | ✅ |
| Project not found in CEO Dashboard sheet | ✅ |
| Delegate name not in contact list | ✅ |
| No delegations found | ✅ (notification email) |

---

## 🚀 Setup Guide

### 1. Configure the Gemini Gem

1. Go to [Google AI Studio](https://aistudio.google.com/) → **Gems**
2. Create a new Gem and paste in a system prompt that:
   - Defines the JSON schema above
   - Instructs Gemini to create a Google Doc named `n8n_bridge_push` in the root of Drive
   - Instructs it to write the JSON as plain text inside the Doc
   - Triggers on the phrase "send it"
3. Connect the Gem to your Google account with Drive + Docs access

### 2. Prepare Google Sheets

**CEO Executive Dashboard** — Create a sheet named `Executive Dashboard V2` with columns:
`Project Name | Status | Owner | Next Milestone | Blocker | Last Updated | Notes`

**CONTACT Sheet** — Create a sheet with columns:
`Name | Email`

### 3. Prepare Notion

- Create one page per project
- Add the standard section headings (Decision Log, Open Questions, Meeting Log)
- Add a status summary table (second table on the page) with the required rows

### 4. Import the n8n Workflow

1. Open your n8n instance
2. Go to **Workflows → Import**
3. Upload `Project_Management_Final.json`
4. Reconnect all credentials (Google Drive, Docs, Sheets, Notion, Gmail)
5. Activate the workflow

---

## 📋 Usage

1. Open the Gemini Gem
2. Describe your project update in natural language, e.g.:

   > *"Project Alpha update — we had a meeting today. Status is On Track. Owner is Jane Doe. Next milestone is finalising the tech spec by May 10th. We decided to increase the budget to 50k and move the launch to June. Open questions: who owns compliance review? Is staging ready? Delegate the compliance checklist to John Smith by May 5th. No blockers."*

3. Say or type **"send it"**
4. Gemini creates the Doc → n8n picks it up within 60 seconds → everything updates automatically

---

## 🏷️ Tags

`automation` · `notion` · `project-management` · `CEO Automation` · `n8n` · `gemini` · `google-drive` · `google-sheets` · `gmail`

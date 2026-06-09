# linkedin-outreach-automation
An end-to-end LinkedIn outreach automation system built for B2B SaaS teams. This project automates the full outreach cycle: ingesting leads, analyzing their LinkedIn activity with AI, leaving reactions and comments on their posts, and sending personalized DMs or connection requests — all while respecting LinkedIn rate limits and working-hour constraints.

---

## 📌 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Workflow 1 — Campaign Orchestrator](#workflow-1--campaign-orchestrator)
- [Workflow 2 — Per-Lead Engagement Engine](#workflow-2--per-lead-engagement-engine)
- [Database Schema (Supabase)](#database-schema-supabase)
- [Rate Limiting & Safety Mechanisms](#rate-limiting--safety-mechanisms)
- [Setup & Configuration](#setup--configuration)
- [Environment Variables & Credentials](#environment-variables--credentials)
- [How to Trigger a Campaign](#how-to-trigger-a-campaign)
- [Flow Diagrams](#flow-diagrams)

---

## Overview

This system allows a B2B SaaS platform to run automated, AI-personalized LinkedIn outreach campaigns at scale. When a campaign is triggered via a webhook (from a frontend app or form), the system:

1. Ingests a list of leads with their LinkedIn profile URLs
2. Validates and normalizes each profile identifier
3. Orchestrates per-lead actions using a child workflow
4. For each lead: fetches their recent LinkedIn posts, generates a personalized AI comment using Gemini, leaves a reaction, posts the comment, and sends a DM or connection request depending on connection distance
5. Tracks every action in Supabase and updates campaign counters throughout

---

## Architecture

```
Frontend / App / Form
        │
        ▼ (POST /webhook/startCampaign)
┌───────────────────────────────┐
│   Workflow 1: Orchestrator    │
│  - Validates leads            │
│  - Stores leads in DB         │
│  - Loops per lead             │
│  - Enforces rate limits       │
│  - Calls Workflow 2 per lead  │
└───────────┬───────────────────┘
            │  executeWorkflow (per lead)
            ▼
┌───────────────────────────────┐
│  Workflow 2: Engagement Engine│
│  - Fetches LinkedIn posts     │
│  - AI comment generation      │
│  - Leaves reaction            │
│  - Posts comment              │
│  - Sends DM or Invite         │
└───────────────────────────────┘
            │
            ▼
     Supabase Database
```

---

## Tech Stack

| Tool | Role |
|------|------|
| **N8N** | Workflow automation engine — orchestration, branching, looping, scheduling |
| **Unipile** | LinkedIn API integration — reading posts, leaving reactions/comments, sending DMs and connection requests |
| **Supabase** | PostgreSQL database — stores leads, campaign metadata, execution state, and user profiles |
| **Gemini (Google AI)** | Chat model — analyzes LinkedIn posts and generates personalized, context-aware comments |
| **Webhook** | Entry point — connects the frontend app or lead submission form to the automation |

---

## Workflow 1 — Campaign Orchestrator

**File:** `LinkedIn_Outreach_Workflow_1.json`

This is the parent workflow. It is triggered once per campaign run and is responsible for orchestrating the entire outreach sequence.

### Step-by-Step Flow

#### 1. Webhook Trigger
- **Node:** `Webhook2`
- Listens on `POST /webhook/startCampaign`
- Accepts a JSON payload containing:
  - `user_id` — the authenticated user's ID
  - `campaign_id` — the campaign to run
  - `leads` — an array of lead objects with `name`, `linkedin_profile`, `phone`, `email`, etc.

#### 2. Campaign Execution ID Registration
- **Node:** `Campaign Execution Id Update`
- Immediately writes the current N8N execution ID to the `campaign` table in Supabase, allowing the frontend to track or cancel the run.

#### 3. Lead Ingestion & Validation
- **Node:** `Split Out` → splits the `leads` array into individual items
- **Node:** `Loop Over Items1` → iterates over each lead
- **Node:** `If` → checks if the lead has a valid `linkedin_profile` (not empty, not "N/A")
  - **Valid lead path:** Continues to `Check If Lead Has Linkedin Id`
  - **Invalid lead path:** Skipped via a no-op node

#### 4. LinkedIn Identifier Normalization
- **Node:** `Check If Lead Has Linkedin Id`
- Custom JS that extracts the LinkedIn username from either a full profile URL (e.g. `https://linkedin.com/in/johndoe`) or a raw identifier string, producing a clean, consistent value for downstream use.

#### 5. Storing Leads in Supabase
- **Node:** `Get User Details` → fetches the user's profile from the `profiles` table
- **Node:** `Enter Leads detail in DB` → inserts each lead into the `LinkedIn Campaign` table with fields: `linkedin`, `full_name`, `gamaID`, `Campaign ID`, `mobile_number`, `email`, and default `NIL` statuses for `Invitation`, `DM`, and `LinkedIn HeadLine`

#### 6. Working Hours Gate
- **Node:** `Date & Time` → captures the current timestamp in IST (Indian Standard Time)
- **Node:** `Wait for next working Hour` → custom JS that checks if the current local hour falls within 9 AM–5 PM. If outside working hours, it calculates how many hours remain until 9 AM and returns a `waitHours` value.
- **Node:** `Wait3` → pauses the workflow for the calculated number of hours before proceeding

#### 7. Fetching Pending Leads
- **Node:** `fetch lead details` → queries the `LinkedIn Campaign` table for leads where `Campaign ID` and `gamaID` match, and `Invitation`, `DM`, and `LinkedIn HeadLine` are all still `NIL`.
- This ensures only unprocessed leads are picked up, making the workflow safely resumable after interruption.

#### 8. Per-Lead Processing Loop
- **Node:** `Loop Over Items2` → iterates over each pending lead
- **Node:** `Fetch Campaign Details` → fetches the campaign record including the current `linkedinLeadCounter`
- **Node:** `Rate limit check` → custom JS evaluating the counter:
  - Every **6th lead** (`count % 6 === 0`) → `is5BlockEnd = true`
  - Every **12th lead** (`count % 12 === 0`) → `is10BlockEnd = true`

#### 9. Rate Limit Branching & Waits
- **Node:** `If1` → if `is10BlockEnd` AND count ≠ 0 → long pause
  - **Node:** `Wait2` → **22 hours** (daily limit cooldown)
- **Node:** `If2` → if `is5BlockEnd` AND count ≠ 0 → medium pause
  - **Node:** `Wait1` → random **50–100 minutes** (batch cooldown)
- Default path (no block end):
  - **Node:** `Wait` → random **15–20 minutes** (per-lead human-like delay)

#### 10. Counter Update & User Fetch
- **Node:** `Update Campaign Details` → increments `linkedinLeadCounter` by 1 in Supabase
- **Node:** `Fetch User Details` → retrieves the user's `unipileID` needed to authenticate LinkedIn API calls

#### 11. Connection Distance Check
- **Node:** `If3` → checks the lead's `linkedinStatus`
  - **`Connected` (1st degree):** Calls Workflow 2 immediately to send a DM
  - **Not connected:** Waits 24 hours (`Wait4`) before calling Workflow 2 (used after a previous connection request was sent)

#### 12. Calling Workflow 2
- **Node:** `Call another workflow for each lead`
- Synchronously calls Workflow 2 for the current lead, passing: `linkedin`, `Campaign ID`, `gamaID`, `sno`, and `accountId` (Unipile ID)

#### 13. Campaign Completion
- After all leads are processed, updates the `campaign` table status to mark the run as complete.

---

## Workflow 2 — Per-Lead Engagement Engine

**File:** `LinkedIn_Outreach_Workflow_2.json`

This is the child workflow, called once per lead by Workflow 1. It handles all direct LinkedIn interactions for a single lead.

### Step-by-Step Flow

#### 1. Triggered by Workflow 1
Receives: `linkedin` (identifier), `Campaign ID`, `gamaID`, `sno`, `accountId` (Unipile ID).

#### 2. Fetch Lead's LinkedIn Posts
- Uses **Unipile's LinkedIn API** to retrieve the lead's recent public posts.
- Extracts post content, post URL, and engagement context.

#### 3. AI Analysis with Gemini
- Sends the lead's post content to **Gemini** with a structured prompt.
- Gemini analyzes the post's topic and tone and generates a personalized, professional comment that feels natural, is contextually relevant, and subtly reflects the sender's expertise.

#### 4. Leave a Reaction
- Uses **Unipile** to leave a LinkedIn reaction (e.g. Like or Insightful) on the lead's latest post.
- This creates a soft, non-intrusive first touchpoint before the comment.

#### 5. Post the AI-Generated Comment
- Uses **Unipile** to post the Gemini-generated comment on the lead's LinkedIn post.
- Updates the `LinkedIn Campaign` table in Supabase with the comment text and status.

#### 6. DM or Connection Request — Based on Distance
Checks the **connection distance** between the user and the lead via Unipile:

| Connection Distance | Action |
|--------------------|--------|
| **1st degree (Connected)** | Sends a **Direct Message (DM)** with a personalized outreach message |
| **2nd or 3rd degree (Not Connected)** | Sends a **Connection Request** with a custom introductory note |

#### 7. Update Lead Status in Supabase
Updates the lead's record in `LinkedIn Campaign`:
- `DM` or `Invitation` → set to the message/note content
- `LinkedIn HeadLine` → set to the lead's current LinkedIn headline
- Timestamps and status fields updated accordingly

---

## Database Schema (Supabase)

### `profiles` table
| Column | Description |
|--------|-------------|
| `id` | User ID (matches `user_id` from webhook) |
| `unipileID` | The user's Unipile account ID for LinkedIn API authentication |

### `campaign` table
| Column | Description |
|--------|-------------|
| `id` | Campaign ID |
| `executionid` | N8N execution ID stored for run tracking and cancellation |
| `linkedinLeadCounter` | Running count of leads processed (drives rate limiting logic) |
| `status` | Current campaign run status |

### `LinkedIn Campaign` table
| Column | Description |
|--------|-------------|
| `sno` | Serial number / primary key |
| `linkedin` | Lead's LinkedIn identifier |
| `full_name` | Lead's full name |
| `gamaID` | ID of the user who owns this campaign |
| `Campaign ID` | Associated campaign |
| `mobile_number` | Lead's phone number |
| `email` | Lead's email address |
| `Invitation` | Connection request status (`NIL` = not yet sent) |
| `DM` | DM status / content (`NIL` = not yet sent) |
| `LinkedIn HeadLine` | Lead's LinkedIn headline (`NIL` = not yet fetched) |

---

## Rate Limiting & Safety Mechanisms

LinkedIn enforces strict limits on automation. This system includes multiple layers of protection:

| Mechanism | Detail |
|-----------|--------|
| **Per-lead random delay** | Waits 15–20 minutes (randomized) between leads — mimics natural human browsing |
| **Batch pause every 5 leads** | After every 5th lead, waits a random 50–100 minutes |
| **Daily cooldown every 10 leads** | After every 10th lead, waits 22 hours before continuing |
| **Working hours enforcement** | Actions only execute between 9 AM and 5 PM IST; the workflow pauses overnight automatically |
| **Resumable lead queue** | Only fetches leads with `NIL` status — safe to re-trigger without duplicating actions |
| **Execution ID tracking** | N8N execution ID stored in Supabase, enabling cancellation from the frontend |

---

## Setup & Configuration

### Prerequisites

- N8N instance (self-hosted or N8N Cloud)
- Supabase project with the tables described above
- Unipile account with LinkedIn connected
- Google Gemini API access (via Google AI Studio or Vertex AI)

### Steps

1. **Import the workflows** into N8N:
   - Go to N8N → Workflows → Import from File
   - Import `LinkedIn_Outreach_Workflow_1.json`
   - Import `LinkedIn_Outreach_Workflow_2.json`

2. **Link the child workflow** in Workflow 1:
   - Open the `Call another workflow for each lead` node
   - Update the `workflowId` to match the ID of your imported Workflow 2

3. **Configure credentials** in N8N's credential manager:
   - Supabase (Project URL + Service Role Key)
   - Unipile (API Key + Base URL)
   - Google Gemini (API Key)

4. **Create Supabase tables** using the schema above.

5. **Activate Workflow 1** — Workflow 2 does not need to be separately activated (it runs as a subworkflow).

---

## Environment Variables & Credentials

All secrets are managed through N8N's built-in credential manager. No `.env` file is required.

| Credential | Used In |
|------------|---------|
| `Supabase` | Both workflows — all database reads and writes |
| `Unipile API Key` | Workflow 2 — post fetch, reactions, comments, DM/invite |
| `Google Gemini API` | Workflow 2 — AI comment generation |

---

## How to Trigger a Campaign

Send a `POST` request to your N8N webhook URL:

```
POST https://<your-n8n-instance>/webhook/startCampaign
Content-Type: application/json
```

```json
{
  "user_id": "uuid-of-the-user",
  "campaign_id": "uuid-of-the-campaign",
  "leads": [
    {
      "name": "Jane Smith",
      "linkedin_profile": "https://www.linkedin.com/in/janesmith/",
      "phone": "+919876543210",
      "email": "jane@example.com"
    },
    {
      "name": "John Doe",
      "linkedin_profile": "johndoe",
      "phone": "",
      "email": "john@example.com"
    }
  ]
}
```

The `linkedin_profile` field accepts either a full LinkedIn URL or a raw username/identifier. Leads with empty or `N/A` profiles are automatically skipped.

---

## Flow Diagrams

### Workflow 1 — High-Level Flow

```
Webhook Trigger
    │
    ├─► Register Execution ID in Supabase
    │
    └─► Split Leads Array
            │
         Loop over leads
            │
         ┌─ Has LinkedIn? ──No──► Skip
         │
        Yes
         │
         ▼
    Extract LinkedIn Identifier
         │
    Save Lead to Supabase (LinkedIn Campaign table)
         │
    Working Hours Check → Wait if outside 9AM–5PM IST
         │
    Fetch unprocessed leads from Supabase (NIL status only)
         │
         Loop over pending leads
            │
         Fetch campaign counter
            │
         Rate limit check
            ├─► Every 10 leads → Wait 22 hours
            ├─► Every 5 leads  → Wait 50–100 min (random)
            └─► Otherwise      → Wait 15–20 min (random)
                │
           Increment lead counter in Supabase
                │
           Fetch user's Unipile ID
                │
           Check connection distance
            ├─► Connected     → Call Workflow 2 (DM)
            └─► Not Connected → Wait 24h → Call Workflow 2 (Invite)
                │
           Continue loop
         │
    Mark campaign complete in Supabase
```

### Workflow 2 — Per-Lead Actions

```
Receive lead data from Workflow 1
    │
Fetch recent LinkedIn posts (Unipile)
    │
Analyze post content with Gemini AI
    │
Generate personalized comment
    │
Leave Reaction on post (Unipile)
    │
Post AI-Generated Comment (Unipile)
    │
Check connection distance
    ├─► 1st degree (Connected) → Send DM (Unipile)
    └─► 2nd / 3rd degree       → Send Connection Request with note (Unipile)
    │
Update lead record in Supabase
(DM/Invitation content, LinkedIn headline, timestamps)
```

---

## Notes

- This project is intended for legitimate B2B outreach and should be used in compliance with [LinkedIn's User Agreement](https://www.linkedin.com/legal/user-agreement) and applicable automation policies.
- The working-hours enforcement and randomized delays are intentional design choices — they reduce automation fingerprinting and protect the connected LinkedIn account from restrictions.
- The system is stateful and resumable: if a run is interrupted, re-triggering the webhook will automatically skip already-processed leads.
 Supabase &amp; Gemini.
[README.md](https://github.com/user-attachments/files/28749840/README.md)

# Real Estate Lead Triage (n8n Workflow)

An n8n workflow that reads new real estate leads from a Google Sheet, scores each one **High / Medium / Low** against the agency's property inventory and lead source, sorts them by priority, drafts a short follow-up for every High lead, and emails a triage report through Gmail.

Built to automate the daily question every sales team asks: **"Which leads do I call first?"**

---

## Workflow Overview

```
Run: Triage Today's Leads
        │
        ▼
Read Inventory Tab          (Google Sheets)
        │
        ▼
Read Leads Tab              (Google Sheets) ──(error)──► Report Sheet Read Error (deactivated)
        │
        ▼
Keep Only Status = New      (Filter)
        │
        ▼
Score and Sort Leads        (Code)
        │
        ▼
Draft Follow-Up for High Leads   (Edit Fields / Set)
        │
        ▼
Build Triage Report         (Code)
        │
        ▼
Send a message              (Gmail)
```

---

## Nodes

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | **Run: Triage Today's Leads** | Manual Trigger | Starts the run on demand ("Execute workflow"). |
| 2 | **Read Inventory Tab** | Google Sheets (`read: sheet`) | Loads the property inventory rows used to match leads to available listings. |
| 3 | **Read Leads Tab** | Google Sheets (`read: sheet`) | Loads all lead rows. On failure, the error output routes to the error-report node. |
| 4 | **Keep Only Status = New** | Filter | Drops every lead whose `Status` is not `New`, so already-handled leads are never re-triaged. |
| 5 | **Score and Sort Leads** | Code | Scores each new lead High / Medium / Low using the inventory and lead source, then sorts High → Medium → Low. |
| 6 | **Draft Follow-Up for High Leads** | Edit Fields (Set) | Adds a short two-line call/SMS follow-up message for each High lead. |
| 7 | **Build Triage Report** | Code | Assembles the sorted leads, suggested next actions and follow-ups into the report body. |
| 8 | **Send a message** | Gmail (`send: message`) | Emails the triage report. |
| - | **Report Sheet Read Error** | Edit Fields (Set), *deactivated* | Intended error branch for a failed Google Sheets read. See [Known Limitations](#known-limitations). |

---

## Data Flow (example run)

| Stage | Items |
|-------|-------|
| Read Inventory Tab | 3 |
| Read Leads Tab | 6 |
| Keep Only Status = New | 5 kept |
| Score and Sort Leads | 1 (leads packed into one item) |
| Draft Follow-Up for High Leads | 1 |
| Build Triage Report | 2 |
| Send a message | 2 |

---

## Prerequisites

- An **n8n** instance (cloud or self-hosted)
- A **Google account** with access to the leads spreadsheet
- n8n credentials:
  - **Google Sheets OAuth2**
  - **Gmail OAuth2**

---

## Google Sheet Structure

The workflow expects one spreadsheet with two tabs.

### Leads tab

| Column | Description |
|--------|-------------|
| Name | Lead's full name |
| Phone / Email | Contact details |
| Source | Where the lead came from (e.g. referral, website, ad) |
| Budget | Lead's stated budget |
| Interested Property / Area | What the lead is asking about |
| Status | `New`, or another status once handled |

### Inventory tab

| Column | Description |
|--------|-------------|
| Property | Listing name or ID |
| Location | Area of the listing |
| Price | Listing price |
| Availability | Whether the listing is still available |

> Match the column names above to your real sheet headers, and update the field names inside the **Score and Sort Leads** code node if they differ.

---

## Setup

1. **Import** the workflow JSON into n8n (**Workflows → Import from file**).
2. **Connect credentials** on both Google Sheets nodes and the Gmail node.
3. In **Read Inventory Tab** and **Read Leads Tab**, select your spreadsheet and the correct sheet tab.
4. In **Send a message**, set the recipient address and subject.
5. Click **Execute workflow** on the manual trigger to test.

---

## Scoring Logic

Each new lead is rated against the inventory and its source:

| Priority | Typical meaning |
|----------|-----------------|
| **High** | Strong match to an available listing (budget and location fit) and a high-intent source. Call first. |
| **Medium** | Partial match, or a lower-intent source. Follow up the same day or next. |
| **Low** | Weak match or unclear intent. Nurture, don't chase. |

Leads are sorted **High → Medium → Low** and each gets a suggested next action. Only **High** leads get a drafted two-line call/SMS follow-up.

> The exact point values and rules live in the **Score and Sort Leads** code node. Keep this table in sync with that node if you change the rules.

---

## Output

The Gmail message contains:

- Leads ordered High → Medium → Low
- A suggested next action for each lead
- A ready-to-send two-line follow-up for every High lead

---

## Known Limitations

- **Manual trigger only.** Nothing runs unless someone clicks Execute. Replace it with a **Schedule Trigger** (for example weekdays at 9:00) to make this a real daily process.
- **Error branch is deactivated.** If the Google Sheets read fails, the workflow stops silently and nobody is told. Enable it and connect it to a Gmail or Slack alert, or attach an Error Workflow.
- **Two emails per run.** The report node outputs 2 items and the Gmail node sends 2 messages. Confirm this is intended; otherwise merge to a single item before sending.
- **Status is never updated.** Triaged leads stay `New`, so tomorrow's run will score and email them again. Add a Google Sheets **Update Row** step to mark them `Triaged`.
- **No empty-state handling.** If no lead has `Status = New`, the Filter outputs nothing and the run ends without a message. Add an IF branch that sends a "no new leads today" note.

---

## Tech Stack

- [n8n](https://n8n.io)
- Google Sheets API
- Gmail API
- JavaScript (n8n Code nodes)

---

## License

MIT

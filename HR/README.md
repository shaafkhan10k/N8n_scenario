
# HR Recruitment Workflow

An [n8n](https://n8n.io) automation that screens job applicants end-to-end: it pulls unprocessed candidates from a Google Sheet, extracts their CV, enriches it with their public GitHub activity, scores them with Claude against a fixed rubric, routes them into a role-specific sheet, and emails the hiring team a ranked top-3 shortlist per role — with every result flagged for manual review, not auto-decided.

## How it works

```
Run HR Screening
  → Read Master Sheet (Processed = FALSE)
    → Loop Over Candidates
        → Download CV → Extract CV Text
        → Get GitHub Repos → Summarize GitHub
        → GPA >= 3?
            ├─ true  → Score Candidate (Claude) → Parse Score → Mark Scored
            │            → Route by Role → Append to <Role> Sheet
            └─ false → Mark Disqualified
        → back to Loop Over Candidates
  → (once loop is done) All Candidates Processed
    → for each role sheet: Read → Sort by Score → Top 3 → Build Digest (HTML) → Email hiring team
```

Roles supported out of the box: **AI/ML Engineer**, **Full Stack Web Development**, **Video Editing**, **Game Development**.

### Scoring rubric

Each candidate is scored 0–100 by Claude using:

| Criterion | Weight |
|---|---|
| CV fit | 40% |
| GitHub signal | 30% |
| GPA | 20% |
| Communication / presentation | 10% |

The model returns strict JSON (`score`, `cv_fit_notes`, `github_notes`, `gpa_contribution`, `overall_reasoning`), which is parsed and written back to the sheet.

A GPA gate (`>= 3`) runs before scoring — candidates below the cutoff are marked disqualified and skipped, saving an LLM call.

## What this workflow does **not** do

- It never auto-advances or rejects a candidate — every digest email is explicitly labeled "manual review required."
- It doesn't verify CV or GitHub content; it treats extracted text and public repo data as scoring signal, not fact.
- It doesn't dedupe or re-score candidates already marked `Processed = TRUE`.

## Setup

### 1. Import the workflow
Import `HR Recruitment Workflow.json` into your n8n instance (Workflows → Import from File).

### 2. Google Sheet structure
Create a Google Sheet with these tabs:

**`Master Response`** (your intake/applicant form responses) — needs at least:
| Column | Notes |
|---|---|
| `Name` | |
| `Email` | |
| `Role Applied For` | Must exactly match one of: `Ai / ML Engineer`, `Full Stack Web Development`, `Vedio Editing`, `Game Development` |
| `GPA` | Numeric |
| `CV Link` | Google Drive file link |
| `GitHub URL` | Full profile URL |
| `LinkedIn URL` | Optional, reference only — not scraped |
| `Processed` | `TRUE` / `FALSE` — the workflow only picks up `FALSE` rows |
| `Status` | Written by the workflow (`Scored` / `Disqualified`) |

**Role sheets** — one tab per role (`Ai / ML Engineer`, `Full Stack Web Development`, `Vedio Editing`, `Game Development`), each with columns:
`Name`, `Email`, `Score`, `GPA`, `CV Fit Notes`, `GitHub Notes`, `GPA Contribution`, `Overall Reasoning`, `GitHub URL`, `LinkedIn URL`

> Note: the "Video Editing" sheet/tab and role label are intentionally spelled `Vedio Editing` in this workflow to match the source form — keep the spelling consistent if you rename anything.

### 3. Credentials to connect
- **Google Sheets** — read/write access to the sheet above
- **Google Drive** — read access to download CVs
- **GitHub API** — a token is enough for public repo reads (`/users/{username}/repos`)
- **Gmail** — to send the digest emails
- **Anthropic API** — used by the `Score Candidate` node (Claude Sonnet)

### 4. Fill in placeholders
Before running:
- Set `documentId` on every Google Sheets node to your actual spreadsheet
- Replace `hiring-team@example.com` in each `Email <Role>` node with your real recipient(s)

### 5. Run it
Trigger manually via **Run HR Screening**, or replace the manual trigger with a Cron/Schedule node to run it on a recurring basis.

## Tech stack
n8n · Claude (Anthropic API) · Google Sheets · Google Drive · Gmail · GitHub API

## License
MIT (or update to match your preference).

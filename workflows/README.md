# Job Hunting Automation — What We Built

This folder contains 6 n8n workflows. n8n is a tool where you connect boxes (“nodes”) together, and data flows from box to box, left to right. Each box does one small job — fetch a webpage, run some code, call an AI, write a spreadsheet row. You don’t need to code to use n8n itself, but a couple of our boxes contain small JavaScript snippets to do the parsing work that no pre-built box could do.

---

## The 6 workflows, in the order you’d actually use them

| # | File | What it does |
|---|---|---|
| 1 | `linkedin_job_extractor.json` | Searches LinkedIn for jobs matching keywords you choose |
| 2 | `linkedin_profile_extractor.json` | Reads data off a public LinkedIn profile page |
| 3 | `yahya_profile_formatter.json` | Stores your resume data once, outputs it in different formats |
| 4 | `ai_job_hunter_resume_tailor.json` | **The main pipeline.** Weekly LinkedIn search → AI resume tailoring → 3 Google Sheets + WhatsApp |
| 5 | `whatsapp_job_bot.json` | WhatsApp bot — text commands to search jobs or trigger an application |
| 6 | `job_application_bot.json` | Analyzes a job form URL with AI and pre-fills all your answers |
| 7 | `un_jobs_extractor.json` | Searches UN Inspira and unjobs.org for data science roles |

If you only run one workflow, run **#4** — it’s the complete weekly pipeline. #5 and #6 work together for on-demand applying via WhatsApp. #7 covers UN-specific jobs.

---

## 1. `linkedin_job_extractor.json` — Job Search

**What it’s for:** You give it keywords (e.g. “data analyst”) and a location, it returns a list of matching LinkedIn job postings.

**The chain of boxes:**

```
Manual Trigger → Search Parameters → Fetch LinkedIn Jobs → Parse Job Listings
```

1. **Manual Trigger** — the “Play” button. You click it to start the workflow.
2. **Search Parameters** — type in what you’re searching for: `keywords`, `location`, `start`, `count`.
3. **Fetch LinkedIn Jobs** — sends a web request to LinkedIn’s public job search, pretending to be a regular browser.
4. **Parse Job Listings** — LinkedIn sends back raw HTML. This box picks out: job title, company, location, date posted, and the job link.

**Output:** one row per job found, with clean fields: `title`, `company`, `job_url`.

---

## 2. `linkedin_profile_extractor.json` — Read a Profile Page

**What it’s for:** Point it at any **public** LinkedIn profile URL and it pulls out the visible info — name, headline, current job, education, skills.

**The chain of boxes:**

```
Manual Trigger → Profile URL → Fetch Profile Page → Parse Profile Elements
```

**Limitation:** Only sees what anyone on the internet can see without being logged in.

---

## 3. `yahya_profile_formatter.json` — Your Resume Data, One Place

**What it’s for:** Stores your complete resume data once and transforms it into different output formats (LinkedIn About section, structured JSON for applications).

```
Manual Trigger → Profile Data Store ┬→ Format: LinkedIn About
                                      └→ Format: Resume JSON
```

**Edit only the Profile Data Store node** when your resume changes — everything downstream updates automatically.

---

## 4. `ai_job_hunter_resume_tailor.json` — The Full Weekly Pipeline ⭐

**What it’s for:** Runs automatically once a week. Searches LinkedIn for jobs that match your skills, asks GPT-4o-mini to write a tailored resume summary + cover note for each job, and logs everything into Google Sheets. Sends a WhatsApp summary when done.

**The chain of boxes:**

```
Weekly Schedule
      │
      ▼
Profile & Search Config        (your résumé data + 3 search terms)
      │
      ▼
Fetch LinkedIn Jobs            (runs 3×, once per search term)
      │
      ▼
Parse & Score Jobs             (extract job info + rate relevance)
      │
      ├──────────────────────────────┤
      ▼                                    ▼
Sheet 1: Jobs Found            Build AI Prompt
                                             │
                                             ▼
                                     AI Resume Tailor      (calls OpenAI)
                                             │
                                             ▼
                                     Parse AI Response
                                             │
                             ┌─────────────┴──────────────┐
                             ▼                              ▼
                     Sheet 2: Tailored Resumes    Sheet 3: Application Tracker
                             │
                     Collect All Jobs (aggregate)
                             │
                     Format WhatsApp Summary
                             │
                     Send WhatsApp Notification
```

### Setup checklist before this one will run

| Step | What | Where |
|---|---|---|
| 1 | Create a Google Sheet with **3 tabs** named exactly: `Jobs Found`, `Tailored Resumes`, `Application Tracker` | Google Sheets |
| 2 | Copy the Sheet ID from its URL (the long string between `/d/` and `/edit`) | — |
| 3 | Paste that ID into `YOUR_GOOGLE_SPREADSHEET_ID` in nodes “Sheet 1”, “Sheet 2”, “Sheet 3” | n8n workflow |
| 4 | Connect a Google account via OAuth and set that credential on the 3 Sheet nodes | n8n credentials |
| 5 | Get an OpenAI API key and paste it over `YOUR_OPENAI_API_KEY` in the “AI Resume Tailor” node | n8n workflow |
| 6 | Replace `YOUR_WHATSAPP_NUMBER` with your number (digits only, e.g. `+14155551234`) | n8n workflow |

---

## 5. `whatsapp_job_bot.json` — WhatsApp Bot

**What it’s for:** An always-on bot you text directly from WhatsApp to search for jobs on demand or trigger a job application fill guide.

**Commands you can send:**

| Command | What happens |
|---|---|
| `find AI engineer` | Searches LinkedIn and returns top 3 matches |
| `find data scientist in Nairobi` | Same, but in a specific location |
| `top` | Searches for top AI/ML jobs right now |
| `apply https://jobs.company.com/apply/123` | Analyzes the application form and pre-fills your answers |
| `help` | Shows the command list |

**The chain of boxes:**

```
WhatsApp Webhook → Parse Command → Route Command
                                          ├─ (search/top) → Search LinkedIn → Parse Results → Send WhatsApp Reply
                                          ├─ (help) → Build Help Reply → Send WhatsApp Reply
                                          ├─ (apply) → Handle Apply Command → Call Application Bot → Build Apply Reply → Send WhatsApp Reply
                                          └─ (unknown) → Build Unknown Reply → Send WhatsApp Reply
                                                                         │
                                                                  Respond to Twilio
```

The **apply** command internally calls the Job Application Form Filler workflow (#6 below) and sends you back a confirmation.

### Setup

1. Set your Twilio credential on the **Send WhatsApp Reply** node.
2. Activate the workflow. Copy the webhook URL shown on the Webhook node (looks like `https://yahyasl85.app.n8n.cloud/webhook/whatsapp-job-bot`).
3. In Twilio console → Messaging → WhatsApp Sandbox Settings, paste that URL into **“When a message comes in”**.
4. From your own WhatsApp, send `join <your-sandbox-word>` to the Twilio sandbox number **+1 415 523 8886** to register your number.

---

## 6. `job_application_bot.json` — Job Application Form Filler

**What it’s for:** Give it a job application URL. It fetches the page, uses GPT-4o-mini to identify every form field, maps your profile answers to each field, writes a tailored two-paragraph cover letter, and sends everything to your WhatsApp + Application Tracker sheet. You then open the form in Chrome and fill it in field-by-field using the guide — it takes a few minutes instead of 30.

**The chain of boxes:**

```
Application Webhook (POST /webhook/job-apply)
      │
      ▼
Fetch Application Page       (downloads the job form HTML)
      │
      ▼
Build Analysis Prompt        (your profile + HTML → OpenAI request body)
      │
      ▼
OpenAI Form Analysis         (GPT-4o-mini reads the form + profile, returns fill data)
      │
      ▼
Parse & Format Guide         (clean up response, build WhatsApp-friendly field list)
      │
      ▼
Log to Application Tracker   (save to Google Sheet: date, company, pre-filled fields, cover letter, steps)
      │
      ▼
WhatsApp Alert               (sends the step-by-step guide + cover letter excerpt to your phone)
      │
      ▼
Webhook Response             (returns JSON confirmation to the caller)
```

### What the AI returns for each application

- **Fields**: a mapping of every input field name → value to type (name, email, LinkedIn URL, years of experience, etc.)
- **Cover letter**: two paragraphs tailored to the specific role and company
- **Step-by-step guide**: ordered list of actions to complete the application
- **Form type**: whether the form can theoretically be auto-submitted or requires browser interaction
- **Estimated time**: how many minutes the manual fill will take

### How to trigger it

**Via WhatsApp bot**: text `apply https://jobs.example.com/apply/123` — the bot calls this workflow automatically.

**Via HTTP** (Postman, curl, another workflow):
```
POST https://yahyasl85.app.n8n.cloud/webhook/job-apply
Content-Type: application/json

{
  "job_url": "https://jobs.example.com/apply/data-scientist",
  "job_title": "Data Scientist",
  "company_name": "Acme Corp"
}
```

### Setup checklist

| Step | What | Where |
|---|---|---|
| 1 | Replace `YOUR_OPENAI_API_KEY` | OpenAI Form Analysis node |
| 2 | Replace `YOUR_GOOGLE_SPREADSHEET_ID` | Log to Application Tracker node |
| 3 | Connect Google Sheets OAuth2 credential | Log to Application Tracker node |
| 4 | Replace `YOUR_TWILIO_CREDENTIAL_ID` and `YOUR_WHATSAPP_NUMBER` | WhatsApp Alert node |
| 5 | Fill in your personal info (phone, address, university…) | Profile object inside Build Analysis Prompt node |

---

## 7. `un_jobs_extractor.json` — UN Job Search

**What it’s for:** Searches two UN job sources for data science roles and writes results to your Google Sheet.

- **UN Inspira** (inspira.un.org) — the official UN Secretariat job board, powered by PeopleSoft HRS. Requires a 2-step session flow: first GET to extract a security token, then POST the search.
- **unjobs.org** — a UN job aggregator covering 50+ agencies. Simpler HTML scraping.

Keywords searched: `data science`, `machine learning`, `data analyst`, `information management`, `statistics`.

**Output:** writes to the `Jobs Found` tab in your Google Sheet with `source` column set to either `UN Inspira` or `unjobs.org (UN Aggregator)`.

---

## How to import any workflow into n8n

1. Go to your n8n at `https://yahyasl85.app.n8n.cloud`
2. Click **⋯** (three dots menu) → **Import from URL**
3. Paste the raw GitHub URL for the file, e.g.:
```
https://raw.githubusercontent.com/yahyasl85/n8n/claude/n8n-job-extraction-setup-m6nwec/workflows/ai_job_hunter_resume_tailor.json
```

**Raw URLs for all workflows:**

| Workflow | Import URL |
|---|---|
| AI Job Hunter (main) | `.../workflows/ai_job_hunter_resume_tailor.json` |
| WhatsApp Bot | `.../workflows/whatsapp_job_bot.json` |
| Application Form Filler | `.../workflows/job_application_bot.json` |
| UN Jobs | `.../workflows/un_jobs_extractor.json` |
| LinkedIn Job Search | `.../workflows/linkedin_job_extractor.json` |
| Profile Formatter | `.../workflows/yahya_profile_formatter.json` |

(Replace `...` with `https://raw.githubusercontent.com/yahyasl85/n8n/claude/n8n-job-extraction-setup-m6nwec`)

---

## A few general n8n concepts that’ll help

- **Items**: data in n8n flows as a list of “items” (think: rows). If a box outputs 5 items, the next box runs once *per item* automatically — no loops needed.
- **Code nodes**: most boxes are pre-built. A few are small custom scripts (parsing HTML, building AI prompts) — you never need to edit the code, only the settings around it.
- **Expressions** (`{{ $json.something }}`): pull a value from the previous box’s output instead of a fixed value.
- **Credentials**: anything talking to an outside service (Google Sheets, OpenAI, Twilio) needs you to connect your account once in n8n. Never stored in the JSON files — that’s why you see placeholders like `YOUR_OPENAI_API_KEY`.

---

## Suggested setup order

1. Import and configure `ai_job_hunter_resume_tailor.json` (the weekly pipeline) — run it once manually to confirm Google Sheets and OpenAI work.
2. Import and configure `whatsapp_job_bot.json` — register the Twilio sandbox, test with `help`.
3. Import and configure `job_application_bot.json` — test by texting `apply [a real job URL]` to your WhatsApp bot.
4. Import `un_jobs_extractor.json` if you want UN job results in the same sheet.

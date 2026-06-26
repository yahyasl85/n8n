# Job Hunting Automation — What We Built

This folder contains 4 n8n workflows. n8n is a tool where you connect boxes ("nodes") together, and data flows from box to box, left to right. Each box does one small job — fetch a webpage, run some code, call an AI, write a spreadsheet row. You don't need to code to use n8n itself, but a couple of our boxes contain small JavaScript snippets to do the parsing work that no pre-built box could do.

This README explains each workflow in plain language: what it's for, how data moves through it, and what you need to fill in before running it.

---

## The 4 files, in the order you'd actually use them

| # | File | What it does |
|---|---|---|
| 1 | `linkedin_job_extractor.json` | Searches LinkedIn for jobs matching keywords you choose |
| 2 | `linkedin_profile_extractor.json` | Reads data off a public LinkedIn profile page |
| 3 | `yahya_profile_formatter.json` | Stores your resume data once, outputs it in different formats |
| 4 | `ai_job_hunter_resume_tailor.json` | **The big one.** Combines #1's job search with AI resume tailoring, writes results into Google Sheets |

If you only run one workflow, run **#4** — it's the complete pipeline. The other three are smaller building blocks that #4 borrows ideas from (you could also run them standalone for simpler tasks).

---

## 1. `linkedin_job_extractor.json` — Job Search

**What it's for:** You give it keywords (e.g. "data analyst") and a location, it returns a list of matching LinkedIn job postings.

**The chain of boxes:**

```
Manual Trigger → Search Parameters → Fetch LinkedIn Jobs → Parse Job Listings
```

1. **Manual Trigger** — the "Play" button. You click it to start the workflow (instead of it running automatically).
2. **Search Parameters** — a box where you type in what you're searching for: `keywords`, `location`, `start` (pagination), `count` (how many results).
3. **Fetch LinkedIn Jobs** — sends a web request to LinkedIn's public job search, pretending to be a regular browser (so LinkedIn doesn't immediately block it).
4. **Parse Job Listings** — LinkedIn sends back a big blob of raw webpage code (HTML). This box is a small script that picks out just the useful bits: job title, company, location, date posted, and the link to the job.

**Output:** one row per job found, with clean fields like `title`, `company`, `job_url`.

**Nothing to configure** — it works as-is, though you'll want to change the default keywords/location in Search Parameters to match what you're looking for.

---

## 2. `linkedin_profile_extractor.json` — Read a Profile Page

**What it's for:** Point it at any **public** LinkedIn profile URL and it pulls out the visible info — name, headline, current job, education, skills.

**The chain of boxes:**

```
Manual Trigger → Profile URL → Fetch Profile Page → Parse Profile Elements
```

1. **Manual Trigger** — click to start.
2. **Profile URL** — type the LinkedIn profile link here (e.g. `https://www.linkedin.com/in/your-username`).
3. **Fetch Profile Page** — downloads that profile's public webpage.
4. **Parse Profile Elements** — a script that reads the downloaded page two ways:
   - First, it looks for a hidden block of structured data LinkedIn embeds in every page (called JSON-LD) — this is the most reliable source.
   - If that's missing some fields, it falls back to scanning the raw HTML for specific patterns (like the headline text).

**Output:** one object with `name`, `headline`, `location`, `about`, `experience[]`, `education[]`, `skills[]`, etc.

**Limitation:** LinkedIn shows much less data to visitors who aren't logged in. This only sees what anyone on the internet could see without an account.

---

## 3. `yahya_profile_formatter.json` — Your Resume Data, One Place

**What it's for:** Instead of re-typing your résumé every time you need it in a different format, this stores it **once** in a single box, and two other boxes transform it into different outputs.

**The chain of boxes:**

```
Manual Trigger → Profile Data Store ─┬─→ Format: LinkedIn About
                                      └─→ Format: Resume JSON
```

1. **Manual Trigger** — click to start.
2. **Profile Data Store** — this is the master copy of your info: summary, skills (grouped into AI/Automation, Data Science, Tech Stack, Domain), work history, education, certifications. **This is the one box you edit** when your resume changes — everything downstream updates automatically.
3. Two boxes run in parallel off the same data:
   - **Format: LinkedIn About** — turns it into a ready-to-paste LinkedIn "About" section, with a character count check (LinkedIn's limit is 2,600 characters).
   - **Format: Resume JSON** — turns it into a clean, structured object suitable for feeding into a PDF generator, a job application form, or another automation.

**Why this matters:** this is the "single source of truth" pattern. Workflow #4 below reuses this same idea — there's one profile box, and everything else reads from it instead of having your info copy-pasted in five places (which would go out of date).

---

## 4. `ai_job_hunter_resume_tailor.json` — The Full Pipeline ⭐

**What it's for:** Runs automatically once a week. It searches LinkedIn for jobs that match your skills, scores how relevant each one is, asks an AI to write a tailored resume summary + cover note **for each individual job**, and logs everything into a Google Sheet you can browse.

**The chain of boxes:**

```
Weekly Schedule
      │
      ▼
Profile & Search Config        (your résumé data + 3 search terms)
      │
      ▼
Fetch LinkedIn Jobs            (runs once per search term = 3 times)
      │
      ▼
Parse & Score Jobs             (extract job info + rate relevance)
      │
      ├──────────────────────────────┐
      ▼                              ▼
Sheet 1: Jobs Found            Build AI Prompt
(Google Sheet, tab 1)                │
                                      ▼
                              AI Resume Tailor      (calls OpenAI)
                                      │
                                      ▼
                              Parse AI Response
                                      │
                      ┌───────────────┴───────────────┐
                      ▼                               ▼
              Sheet 2: Tailored Resumes      Sheet 3: Application Tracker
              (Google Sheet, tab 2)          (Google Sheet, tab 3)
```

### Box-by-box explanation

1. **Weekly Schedule** — instead of a manual "Play" button, this is a clock. It fires automatically every Monday at 8am, so you don't have to remember to run it.

2. **Profile & Search Config** — same idea as workflow #3's "Profile Data Store," but it also defines **3 search queries** to run (e.g. "AI engineer," "data scientist," "AI consultant"). Because it outputs 3 separate items, every box after it runs 3 times — once per search.

3. **Fetch LinkedIn Jobs** — same job-search request as workflow #1, just running 3× automatically (once per search term from step 2).

4. **Parse & Score Jobs** — same job-parsing logic as workflow #1, *plus* a relevance score: it counts how many of your skills appear in each job's title/company text. Higher score = better match.

5. **Sheet 1: "Jobs Found"** — every job found gets written as a new row here, regardless of score. This is your raw list — a record of everything the search turned up.

6. **Build AI Prompt** — *in parallel* with step 5, this box writes out instructions for the AI: here's the job, here's your résumé, please tailor it. It's just text formatting — no AI call happens yet.

7. **AI Resume Tailor** — this is the box that actually calls OpenAI's API (specifically GPT-4o-mini) and asks it to return a tailored summary, a list of key skills to emphasize, a short cover note, and an honest assessment of how good a fit the job is.

8. **Parse AI Response** — the AI replies with text formatted as JSON; this box unpacks it into clean fields.

9. **Sheet 2: "Tailored Resumes"** — each job's AI-written summary, skills, and cover note get logged here.

10. **Sheet 3: "Application Tracker"** — a row gets added per job with status `"To Apply"` and empty columns for `date_applied`, `interview_date`, `offer_status`, `notes` — so you can manually update this sheet as you actually apply and progress through interviews.

### Why three separate sheets instead of one?

- **Jobs Found** = "what's out there" (everything, unfiltered)
- **Tailored Resumes** = "what to say" (the AI's writing, reusable when you apply)
- **Application Tracker** = "where things stand" (your manual tracking — apply, interview, offer)

Keeping them separate means you can sort/filter Jobs Found by relevance score without cluttering your tracker, and you can update the tracker by hand without touching the AI-generated content.

### Setup checklist before this one will run

| Step | What | Where |
|---|---|---|
| 1 | Create a Google Sheet with **3 tabs**, named exactly: `Jobs Found`, `Tailored Resumes`, `Application Tracker` | Google Sheets |
| 2 | Copy the Sheet's ID from its URL (the long string between `/d/` and `/edit`) | — |
| 3 | Paste that ID into `YOUR_GOOGLE_SPREADSHEET_ID` — appears in 3 places: nodes "Sheet 1," "Sheet 2," "Sheet 3" | n8n workflow |
| 4 | Connect a Google account to n8n (OAuth) and set that credential on the same 3 Sheet nodes | n8n credentials |
| 5 | Get an OpenAI API key and paste it over `YOUR_OPENAI_API_KEY` in the "AI Resume Tailor" node | n8n workflow |
| 6 | (Optional) Edit the search keywords in "Profile & Search Config" if you want different job titles searched | n8n workflow |

Each sheet tab needs column headers matching the field names listed in that node (e.g. Sheet 1 needs columns: `job_id, title, company, location, posted_date, job_url, matched_skills, relevance_score, search_query, date_found, source`).

---

## A couple of general n8n concepts that'll help

- **Items**: data in n8n flows as a list of "items" (think: rows). If a box outputs 5 items, the next box runs once *per item* automatically — you don't write a loop yourself.
- **Code nodes**: most boxes are pre-built and just need settings filled in. A few boxes ("Parse Job Listings," "Build AI Prompt," etc.) are small custom scripts because no pre-built box does that exact parsing — but you never need to edit the code, only the settings around it (like the Profile Data Store).
- **Expressions** (`{{ $json.something }}`): you'll see these in some box settings — they mean "pull this value from the previous box's output" rather than a fixed value.
- **Credentials**: anything talking to an outside service (Google Sheets, OpenAI) needs you to connect your own account/API key once in n8n — these are never stored in the JSON file itself, which is why you see placeholders like `YOUR_OPENAI_API_KEY`.

---

## Suggested next step

Import `ai_job_hunter_resume_tailor.json` into your n8n instance, work through the setup checklist above, and run it once manually (there's no harm running it outside its Monday schedule) to see real data land in your Google Sheet. Once that round-trip works, it'll keep running itself weekly.

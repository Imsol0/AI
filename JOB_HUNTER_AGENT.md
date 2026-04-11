# Job Hunter Agent — Universal Meta-Prompt

> **For:** Cursor / Windsurf (or any agent with web search, web fetch, file I/O, and shell access)
> **What it does:** Parses your resume, runs deep job discovery across hidden ATS platforms, verifies every link, scores fit, deduplicates across runs, and outputs an interactive HTML dashboard + Excel pipeline.
> **How to use:** Drop your resume `.docx` into the project root. Open this file in Cursor/Windsurf. Tell the agent: *"Follow JOB_HUNTER_AGENT.md."*

---

## Operating Principles (read first, apply throughout)

1. **The user's value is depth, not breadth of job boards.** Anyone can search Indeed. Your job is to find roles on company ATS pages, niche boards, and direct career sites that the user would never discover alone.
2. **Every link must be verified live.** No exceptions. No guesses. No "probably still open." If you cannot fetch the URL and confirm the role is active, drop it.
3. **Build, don't describe.** Phases 3 and 4 require you to *write working Python code* — not pseudocode, not a description. The script must run end-to-end on first execution.
4. **Re-runnability is non-negotiable.** Running the pipeline twice in a row must show zero new jobs the second time. Dedup is correctness, not a nice-to-have.
5. **When uncertain, ask before acting.** Wrong assumptions about location, experience level, or work mode waste hours. Confirmation costs seconds.
6. **Batch tool calls.** Web searches and fetches should run in parallel wherever the agent supports it.

---

## Phase 0 — Resume Ingestion

### 0.1 Locate the resume
Scan the project root for `.docx` files.
- **Zero found:** Stop. Tell the user: *"Place your resume `.docx` in the project root and re-run."*
- **One found:** Use it. Confirm the filename to the user before parsing.
- **Multiple found:** List them and ask the user which is current.

### 0.2 Parse the resume
`.docx` is a zip — standard file-read tools won't work. Use Python:

```bash
python -c "import docx; doc = docx.Document('FILENAME.docx'); print('\n'.join(p.text for p in doc.paragraphs))"
```

If `python-docx` is missing: `pip install python-docx` (or `pip install --break-system-packages python-docx` on managed systems).

Also extract tables (resumes often put skills in tables):
```python
for table in doc.tables:
    for row in table.rows:
        for cell in row.cells:
            print(cell.text)
```

### 0.3 Build the candidate profile
From the parsed text, extract and **write to `candidate_profile.json`** in the project root so subsequent runs can reuse it:

```json
{
  "name": "...",
  "location_city": "...",
  "location_region": "...",
  "location_country": "...",
  "field": "...",
  "years_experience": 0,
  "education": [{"degree": "...", "school": "...", "year": "..."}],
  "certifications": ["..."],
  "hard_skills": ["..."],
  "soft_skills": ["..."],
  "languages": ["..."],
  "work_authorization": "unknown | citizen | PR | work_permit | needs_sponsorship",
  "current_or_last_role": "...",
  "notable_projects": ["..."]
}
```

If a field is genuinely absent from the resume, use `null` — don't invent.

### 0.4 Detect the field
If the resume's field is ambiguous (e.g., a recent grad with mixed experience), do **not** guess. Ask. Otherwise, classify into one of: `tech`, `data`, `cybersecurity`, `healthcare`, `finance`, `marketing`, `creative`, `engineering`, `legal`, `education`, `government`, `nonprofit`, `trades`, `hospitality`, `retail`, `science`, `other`. This classification drives Phase 1.6 (sector dorking).

---

## Phase 1 — Confirmation (Thorough)

Before any searching, present a single consolidated summary to the user and confirm. Use this exact structure:

```
Resume parsed. Here's what I found:

  Name:           [name]
  Location:       [city, region, country]
  Field:          [detected field]
  Experience:     [X years, level inferred]
  Top skills:     [top 5 hard skills]
  Certifications: [list or "none found"]

Before I start searching, I need to confirm 6 things:

1. Target roles — what job titles are you aiming for? (List 2-5. I'll
   expand into all variations automatically.)

2. Geographic scope — which cities/regions? Are you willing to commute?
   How far?

3. Work mode — on-site only, hybrid only, remote only, or any mix?

4. Experience level — entry, mid, senior, or open? (I won't include
   levels you don't ask for.)

5. Companies to EXCLUDE — anywhere you've already applied or want
   to skip?

6. Companies to PRIORITIZE — dream companies, target industries, or
   any "must include" employers?

Optional but useful:
  - Salary floor (I'll filter below it)
  - Authorization constraints (do you need sponsorship?)
  - Industries to AVOID (e.g., no defense, no gambling, no crypto)
```

**Wait for the user's response. Do not proceed until all six are answered.** If the user gives a partial answer, ask only the missing items.

Save the answers to `search_config.json` so re-runs reuse them.

---

## Phase 2 — Exhaustive Discovery

This is where most agents fail by stopping too early. **You must execute at least 20 distinct search queries** spanning the categories below. Track them — if you've run fewer than 20 by the end of this phase, keep going.

### 2.1 Title expansion
From the user's stated target roles, generate:
- **Primary titles** — exactly as user named them
- **Synonym titles** — industry equivalents (e.g., "Data Analyst" ↔ "BI Analyst", "Reporting Analyst", "Insights Analyst", "Analytics Specialist")
- **Adjacent titles** — same skills, different framing (e.g., "Data Analyst" ↔ "Operations Analyst", "Research Analyst")
- **Seniority variants** — "Junior X", "Associate X", "X I", "X II" (only at the levels user requested)
- **Industry-prefixed variants** — e.g., "Clinical Data Analyst", "Marketing Data Analyst", "Financial Data Analyst"

Output this list before searching so the user can sanity-check it.

### 2.2 ATS platform dorking — the highest-value step

Companies post on their ATS *before* (or instead of) syndicating to Indeed/LinkedIn. Run dorks against every major ATS. Use the user's `LOCATION` and rotate through their `TITLE` variants.

| ATS | Dork pattern |
|---|---|
| Workday | `site:myworkdayjobs.com "TITLE" "LOCATION"` |
| Greenhouse | `site:boards.greenhouse.io "TITLE" "LOCATION"` |
| Lever | `site:jobs.lever.co "TITLE" "LOCATION"` |
| Ashby | `site:jobs.ashbyhq.com "TITLE" "LOCATION"` |
| SmartRecruiters | `site:smartrecruiters.com "TITLE" "LOCATION"` |
| Workable | `site:workable.com "TITLE" "LOCATION"` |
| BambooHR | `site:bamboohr.com/careers "TITLE" "LOCATION"` |
| Breezy | `site:breezy.hr "TITLE" "LOCATION"` |
| JazzHR | `site:applytojob.com "TITLE" "LOCATION"` |
| iCIMS | `site:icims.com "TITLE" "LOCATION"` |
| Jobvite | `site:jobs.jobvite.com "TITLE" "LOCATION"` |
| Recruitee | `site:recruitee.com "TITLE" "LOCATION"` |
| ADP WFN | `site:workforcenow.adp.com "TITLE" "LOCATION"` |
| Paylocity | `site:recruiting.paylocity.com "TITLE" "LOCATION"` |
| Dayforce | `site:dayforcehcm.com "TITLE" "LOCATION"` |
| Pinpoint | `site:pinpointhq.com "TITLE" "LOCATION"` |
| Teamtailor | `site:teamtailor.com "TITLE" "LOCATION"` |
| Rippling | `site:ats.rippling.com "TITLE" "LOCATION"` |

**Rule:** Run at least 8 ATS dorks per scan. Rotate the title variants across them so you're not searching the same string 18 times.

### 2.3 Direct career page bypass
```
inurl:careers "TITLE" "LOCATION" -indeed -linkedin -glassdoor -ziprecruiter -monster
inurl:jobs "TITLE" "LOCATION" -indeed -linkedin -glassdoor
"we're hiring" "TITLE" "LOCATION" -indeed -linkedin
```

### 2.4 Regional and niche boards
Pick boards relevant to the user's `location_country`:

**Canada:** Built In Toronto/Vancouver/Ottawa, Communitech (Waterloo), Discover Technata (Kanata), London Tech Jobs, Knighthunter, GC Jobs, GC Digital Talent, Career Beacon, Jobillico, Eluta, Workopolis, IndeedFlex.

**USA:** Built In (per city), Dice, Wellfound (AngelList Talent), USAJobs, Handshake, "HN Who is Hiring" via hnhiring.com, Otta, Y Combinator Work at a Startup.

**UK / EU:** Otta, Welcome to the Jungle, CWJobs, Reed, Totaljobs, Honeypot (DE), Stepstone (DE).

**Australia:** Seek, EthicalJobs (nonprofit), Hatch (early career).

**Global remote:** RemoteOK, WeWorkRemotely, Remotive, Remote.co, Working Nomads — *only if the user said remote is OK*.

### 2.5 Sector-specific sources
Match to the field detected in Phase 0.4:

| Field | Sources |
|---|---|
| Healthcare | Hospital HR portals, regional health authority sites, NHS Jobs (UK), Health Match BC, Medical Jobs Canada |
| Government | City/municipal portals, provincial/state portals, federal portals, Apply To Education (school boards) |
| Education | University HR (`site:*.edu careers`, `site:*.ac.uk jobs`), HigherEdJobs, Times Higher Education |
| Finance/Banking | Big bank careers, Big 4 consulting careers, eFinancialCareers, CFA Institute job board |
| Tech/Startups | Wellfound, YC jobs, ProductHunt jobs, Hacker News hiring threads |
| Cybersecurity | ISC2 Career Center, CyberSeek, infosec-jobs.com, niche MSSP career pages |
| Data/ML | Kaggle Jobs, ai-jobs.net, ML jobs boards, research lab career pages |
| Creative/Design | Dribbble Jobs, Behance, Coroflot, Working Not Working, Authentic Jobs |
| Legal | Law society job boards, ZSA, RainmakerThinking, in-house counsel boards |
| Nonprofit | Idealist, Charity Village (CA), Work for Good, EthicalJobs |
| Trades | Local union boards, apprenticeship portals, industry association sites |

### 2.6 Hidden company discovery
- Search `"top [field] companies in [city]"` — get directories
- Search Clutch.co, G2, Cloudtango for service firms in user's domain
- Check the user's region's economic development corporation site for major employers
- Search Reddit: `site:reddit.com r/[city] hiring [field]`
- Check industry association member directories

### 2.7 Recency filters
For follow-up scans, append recency to dorks: `posted "last 7 days"`, `"posted today"`, or use search engine date filters where available.

### 2.8 Search budget check
Before moving to Phase 3, count your distinct queries. If under 20, return to 2.2-2.6 and run more. The user explicitly wants depth.

---

## Phase 3 — Verification & Scoring

### 3.1 Verify every link
For each candidate URL, fetch the page and confirm **all four**:

1. **HTTP 200** — not 403, 404, or redirect to a generic `/careers` landing page
2. **Role still listed** — page contains the job title and apply mechanism
3. **Not expired** — no "this position has been filled", "no longer accepting applications", "posting closed"
4. **Matches stated location** — the location on the page matches what the user wants (or is remote and user accepts remote)

If any check fails: **drop the link entirely.** Do not include "check the portal manually" entries.

### 3.2 Extract structured data per job
For every verified job, capture:

| Field | Notes |
|---|---|
| `company` | Full legal name; note parent company if subsidiary |
| `title` | Exact title as posted |
| `location` | City, region |
| `mode` | `on-site` / `hybrid` / `remote` |
| `level` | `intern` / `entry` / `mid` / `senior` |
| `salary_min`, `salary_max`, `salary_currency` | `null` if not posted — do **not** invent |
| `posted_date` | If shown |
| `closing_date` | If shown — flag urgent if ≤7 days away |
| `match_score` | 0-100, see 3.3 |
| `match_reasons` | Bullet list of why it scored that way |
| `fit_summary` | One sentence: why this user, this role |
| `tier` | 1-4, see 3.4 |
| `apply_url` | Verified-live direct link |
| `source_ats` | Which platform it came from |
| `verified_at` | ISO timestamp |

### 3.3 Match scoring rubric (concrete, not vibes)

Weight each factor and sum:

| Factor | Weight | How to score |
|---|---|---|
| Required hard skills overlap | 35 | (skills_user_has ÷ skills_required) × 35 |
| Years of experience match | 20 | Full marks if user is in posted range; -5 per year below; -3 per year above |
| Education requirement | 10 | Full marks if user meets, half if "preferred", 0 if hard-required and missing |
| Certifications | 10 | Full marks if user has all required; partial if some |
| Industry experience | 10 | Full if user has worked in same industry; half if adjacent |
| Location feasibility | 10 | Full if local or accepted remote; 0 if outside scope |
| Seniority match | 5 | Full if level matches user's preference exactly |

**Floor for inclusion: 60.** Anything below 60 is dropped unless the user explicitly asked for stretch roles.

Bands:
- **90-100** — apply this week
- **75-89** — strong fit, apply within 2 weeks
- **60-74** — stretch fit, apply if time permits

### 3.4 Tier classification

| Tier | Meaning |
|---|---|
| **1** | Direct match — title and skills are bullseye |
| **2** | Adjacent — same skills, different title or slight pivot |
| **3** | Enterprise/brand-name — broader scope, good résumé builder |
| **4** | Specialized — government, defense, regulated industries (may need clearance, longer hiring cycles) |

---

## Phase 4 — Pipeline Construction

Build all of this. Do not stub it. The user must be able to run `python scan_jobs.py` and get working output.

### 4.1 Files to create

```
project_root/
├── [their_resume].docx          ← already there
├── candidate_profile.json       ← from Phase 0.3
├── search_config.json           ← from Phase 1
├── job_tracker.json             ← dedup database (created on first run)
├── scan_jobs.py                 ← the pipeline
├── Jobs_Scan.xlsx               ← generated output
└── jobs.html                    ← generated output
```

### 4.2 `scan_jobs.py` requirements

The script must:

1. **Define `CURRENT_SCAN_JOBS`** as a list of dicts using the schema from 3.2. This is the only section you update between scans.
2. **Load `job_tracker.json`** if it exists; create empty if not. Structure:
   ```json
   {
     "scans": [{"date": "...", "total": 0, "new": 0}],
     "seen_keys": {
       "<md5>": {"company": "...", "title": "...", "first_seen": "..."}
     }
   }
   ```
3. **Compute keys** as `md5(company.lower().strip() + "|" + title.lower().strip())`.
4. **Tag each job** as `new` or `archived` by checking the tracker.
5. **Update the tracker** with new keys and append a scan record.
6. **Build `Jobs_Scan.xlsx`** using `openpyxl`:
   - Sheet 1 "Jobs": all roles, sorted by match_score desc, columns from 3.2 plus a "NEW?" column
   - NEW rows filled green; match score conditionally colored (≥90 green, 75-89 amber, 60-74 light red)
   - Column with `apply_url` as a clickable hyperlink
   - Header row frozen, autofilter on
   - Sheet 2 "Summary": scan timestamp, totals, urgent count (closing ≤7 days), top 10 by match
7. **Build `jobs.html`** as a self-contained dark-themed dashboard:
   - Hero with user's name, scan timestamp, and headline stats (total, new this scan, ≥90% matches, urgent)
   - Filter chips: All • New only • ≥90% match • Urgent • Hybrid • On-site • Remote • per-Tier
   - Search box with full-text filter (vanilla JS, no framework needed)
   - Two clearly separated sections: **Fresh Finds** (new this scan) and **Archive** (previously seen)
   - Each card: company, title, tag row (mode/level/salary/posted/closing), big match score circle, fit summary, "Apply" button → opens apply_url in new tab
   - NEW badge (green) and URGENT badge (red, animated) where applicable
   - All CSS inline; no external dependencies; no CDN calls
8. **Print a CLI summary** at the end: total / new / urgent / top 5.

### 4.3 Minimum quality bar
- Script must be **idempotent**: running it twice in a row → 0 new on the second run
- Excel hyperlinks must be **clickable** (not raw text)
- HTML must render **offline** (no CDN, no external fonts)
- All file writes use UTF-8

---

## Phase 5 — Serve & Verify

After generating files:

1. **Kill any process on port 8080**, then start a local server:
   ```bash
   python -m http.server 8080
   ```
   Background it so the user can keep working.
2. **Open `http://localhost:8080/jobs.html`** in the browser if the agent supports it.
3. **Sanity-check** by clicking through filters yourself if the agent has browser tools — confirm "New only" shows the right count, "Urgent" filters correctly.

---

## Phase 6 — Deliver

Present this to the user as the final message:

```
Scan complete.

  Total verified roles:     [N]
  New this scan:            [N]
  ≥90% match:               [N]
  Closing within 7 days:    [N]   ← apply first

Top 5 matches:
  1. [Company] — [Title] — [Match%] — [Salary] — [Location]
  2. ...

Hidden gems (not on Indeed/LinkedIn):
  • [Company] — found via [ATS/source]
  • ...

Files:
  • Dashboard:  http://localhost:8080/jobs.html
  • Excel:      ./Jobs_Scan.xlsx
  • Tracker:    ./job_tracker.json

To re-scan later:
  python scan_jobs.py     (after I add new finds to CURRENT_SCAN_JOBS)
  Or just say "scan again" and I'll run a fresh discovery round.

Action plan:
  This week:    Apply to all ≥90% matches and anything closing soon
  Next 2 weeks: Apply to 75-89% matches
  Ongoing:      Monitor [N] tier-4 portals for cyclical openings
```

---

## Re-run Protocol

When the user says "scan again", "find new jobs", or "re-run":

1. **Do not** rebuild `scan_jobs.py` from scratch — preserve `job_tracker.json` and `search_config.json`.
2. Run a fresh Phase 2 with **recency filters** ("last 7 days", "posted today").
3. Verify new candidates per Phase 3.
4. Append the new entries to `CURRENT_SCAN_JOBS` in `scan_jobs.py` (replace the old list — the tracker remembers what's been seen).
5. Run `python scan_jobs.py`. Dedup tags genuinely new vs. previously seen.
6. Show the user the delta: *"Found X new roles since last scan on [date]."*

---

## Failure Modes & Recovery

| Problem | Action |
|---|---|
| `python-docx` install fails | Try `--break-system-packages`; if still failing, try `apt install python3-docx` or ask user to use a venv |
| Resume parses but profile is sparse | Show what you got and ask the user to fill the gaps directly in `candidate_profile.json` |
| < 20 search queries possible (very narrow field) | Tell the user, then go deeper on the queries that *do* work — more title variants, more locations |
| 0 verified jobs after full scan | Don't fabricate. Tell the user the search came up empty, list the queries you ran, and ask whether to (a) broaden location, (b) drop a hard requirement, or (c) include stretch roles ≥50% |
| Job page requires JS to load | Try `web_fetch` first; if empty, try a different fetcher; if still empty, drop the link rather than guess |
| Port 8080 in use | Kill the existing process; if not possible, use 8081 and tell the user |
| User's location is rural / no local market | Suggest nearest metros and ask if they'll commute or relocate |

---

## Hard Rules (violations = broken output)

- **Never** include an unverified link.
- **Never** invent salaries, dates, or requirements.
- **Never** include roles below the 60% match floor unless the user opted in.
- **Never** include work modes the user excluded.
- **Never** include experience levels the user didn't ask for.
- **Never** stub `build_excel` or `build_html` — they must work end-to-end.
- **Never** skip the confirmation phase, even if the resume seems clear.
- **Always** dedup by `(company, title)` hash.
- **Always** preserve previous scans in the archive section.
- **Always** batch parallel searches when the agent supports it.

---

## Field Adaptation Cheatsheet

| Field | Title variants to expand into | Sector-specific sources to prioritize |
|---|---|---|
| Data / Analytics | Data Analyst, BI Analyst, Analytics Engineer, Reporting Analyst, Insights Analyst, Data Specialist | Kaggle, ai-jobs.net, company data team pages |
| Cybersecurity | SOC Analyst, Security Analyst, Pentester, Vulnerability Analyst, GRC Analyst, AppSec Engineer | ISC2, CyberSeek, MSSP careers, infosec-jobs.com |
| Software Eng | Software Engineer, Backend Engineer, Full Stack Developer, Platform Engineer, SDE | Wellfound, YC Jobs, HN Hiring, company eng blogs |
| Nursing | RN, Registered Nurse, Clinical Nurse, Nurse Practitioner, Charge Nurse | Hospital HR portals, health authority sites, NHS Jobs |
| Marketing | Marketing Manager, Growth Marketing, Content Strategist, Brand Manager, Digital Marketing | Marketing Brew, HubSpot jobs, agency career pages |
| Accounting | Staff Accountant, CPA, Auditor, Tax Analyst, Controller | Big 4 portals, CPA firm careers, eFinancialCareers |
| Mechanical Eng | Mechanical Engineer, Design Engineer, CAD Engineer, Manufacturing Engineer | Industry associations, plant career pages, ASME |
| UX / Product Design | UX Designer, Product Designer, UI Designer, Interaction Designer | Dribbble, Behance, Working Not Working |
| Teaching | Teacher, Instructor, Educator, Lecturer | School board portals, Apply to Education, HigherEdJobs |
| Legal | Associate, Counsel, Paralegal, Legal Analyst | Law society boards, ZSA, in-house counsel boards |

---

*End of meta-prompt. The agent should follow phases sequentially, batch parallel operations, and never fabricate data.*

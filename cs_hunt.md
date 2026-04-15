# Cyber Job Hunter — Cursor Meta-Prompt

> **For:** Cursor (or any agent with web search, web fetch, and file read).
> **What it does:** Parses your resume, asks one focused round of cyber-specific questions, runs deep ATS dorking, verifies every link, computes a real ATS keyword score against your resume, and outputs a single self-contained `jobs.html` dashboard.
> **No Python pipeline. No openpyxl. No pandoc. One markdown file in, one HTML file out.**
>
> **How to use:** Drop your resume `.docx` in the project root. Open this file in Cursor. Say: *"Follow CYBER_JOB_HUNTER.md."*

---

## Operating Principles

1. **Cyber is not one field.** Offensive, defensive, GRC, and AppSec hire on completely different signals. The agent must know which lane the user is in before searching.
2. **Depth over breadth.** Indeed and LinkedIn are the user's job — find roles on company ATS pages they'd never discover alone.
3. **Every link verified live.** No 404s, no expired postings, no "check the portal." Drop it if it doesn't load.
4. **No code pipeline.** No `scan_jobs.py`, no Excel, no openpyxl. Just one self-contained `jobs.html`. Token-efficient by design.
5. **ATS score is computed locally.** Read the resume text and the job description text, intersect cyber keywords, output a real percentage. No external libraries beyond `python-docx` for the resume.
6. **When uncertain, ask before acting.** One consolidated question block. Never guess offensive vs defensive.

---

## Phase 0 — Resume Ingestion

### 0.1 Find the resume
Scan project root for `.docx` files.
- **None:** Stop. Tell the user to drop their resume in the project folder.
- **One:** Use it.
- **Multiple:** List and ask which is current.

### 0.2 Parse it
`.docx` is a zip — standard read tools won't work. Use `python-docx`:

```bash
python -c "
import docx
doc = docx.Document('FILENAME.docx')
text = '\n'.join(p.text for p in doc.paragraphs)
for t in doc.tables:
    for r in t.rows:
        for c in r.cells:
            text += '\n' + c.text
print(text)
" > resume_text.txt
```

If `python-docx` is missing: `pip install python-docx` (add `--break-system-packages` if needed).

The output `resume_text.txt` is the **single source of truth** for ATS scoring in Phase 4. Do not delete it.

### 0.3 Extract profile
From `resume_text.txt`, identify:
- Name, location (city / region / country)
- Years of experience (estimate from work history dates)
- Education and graduation year (matters for internship eligibility)
- Certifications: look for **Security+, Network+, CySA+, PenTest+, CASP+, CISSP, CISM, CISA, CEH, OSCP, OSEP, OSWE, OSED, CRTP, CRTO, eJPT, eCPPT, PNPT, GSEC, GCIH, GPEN, GWAPT, GCIA, GCFA, GCFE, AWS Security, Azure SC-200/SC-100, CCSP**
- Hard skills: programming languages, tools (Burp, Nmap, Metasploit, Wireshark, Splunk, Sentinel, CrowdStrike, Nessus, Cobalt Strike, BloodHound, etc.), platforms (AWS, Azure, GCP), frameworks (MITRE ATT&CK, NIST CSF, ISO 27001, PCI-DSS)
- Notable projects, CTF placements, HackTheBox/TryHackMe ranks, bug bounty disclosures
- Work authorization (if discernible)

Hold this in memory — no JSON file needed.

---

## Phase 1 — Confirmation (One Block, Wait for Reply)

Present this **exact block** to the user. Do not search until they reply.

```
Resume parsed. Here's what I found:

  Name:           [name]
  Location:       [city, region, country]
  Experience:     [X years]
  Education:      [degree, school, year]
  Certifications: [list or "none found"]
  Top skills:     [top 6 detected]

Before I search, I need answers to all 8 questions below.
You can reply numbered, freeform — whatever's fastest.

1. CYBER LANE — which best describes the roles you want?
   (a) Offensive  — pentest, red team, bug bounty, exploit dev, AppSec offense
   (b) Defensive  — SOC, blue team, IR, threat hunt, detection eng, DFIR
   (c) GRC        — risk, compliance, audit, ISO/SOC2, policy, vCISO track
   (d) AppSec     — SAST/DAST, secure code review, product security
   (e) Cloud Sec  — AWS/Azure/GCP security, CSPM, IAM, container security
   (f) Identity   — IAM, IdP, Zero Trust, PAM
   (g) OT / ICS   — industrial, SCADA, critical infrastructure
   (h) Open       — multiple of the above (list which)

2. ROLE TYPE — pick all that apply:
   (a) Internship / Co-op
   (b) New grad / Junior / Entry
   (c) Mid-level (2-5 yrs)
   (d) Senior (5+ yrs)

3. LOCATION — which cities/regions? Willing to commute? How far?

4. WORK MODE — on-site, hybrid, remote, or any?

5. CLEARANCE — do you have or are you eligible for a security clearance?
   (Matters heavily for tier-4 defense/government roles.)

6. EXCLUDE — any companies you've already applied to or want to skip?

7. PRIORITIZE — dream companies, target industries, or "must include"?

8. SALARY FLOOR — minimum acceptable (or "open"). Currency assumed to
   match your location.

Optional but useful:
  • Industries to AVOID (defense, gambling, crypto, ad-tech, etc.)
  • Sponsorship needed? (yes / no / already authorized)
```

**Wait for the reply. Do not proceed until questions 1-8 are answered.** Re-ask only the missing items if the reply is partial.

Hold the answers in memory for the rest of the session.

---

## Phase 2 — Title Expansion (Lane-Specific)

Based on the user's answer to question 1, expand into the right title set. **Do not search defensive titles for an offensive user.** This is the most common mistake.

### Offensive lane
Penetration Tester, Pentester, Red Team Operator, Red Team Engineer, Offensive Security Engineer, Ethical Hacker, Security Consultant (Pentest), Adversary Simulation Engineer, Exploit Developer, Application Penetration Tester, Network Penetration Tester, Cloud Penetration Tester, Wireless Penetration Tester, Physical Penetration Tester, Purple Team Engineer.

### Defensive lane
SOC Analyst (I/II/III), Security Analyst, Security Operations Engineer, Threat Hunter, Threat Intelligence Analyst, Incident Responder, IR Consultant, DFIR Analyst, Digital Forensics Analyst, Detection Engineer, Detection & Response Engineer, Security Engineer (Blue Team), MDR Analyst, XDR Engineer, SIEM Engineer, Splunk Engineer, Sentinel Engineer.

### GRC lane
GRC Analyst, Risk Analyst, Compliance Analyst, IT Auditor, Information Security Auditor, ISO 27001 Lead Implementer, SOC 2 Analyst, Security Compliance Specialist, Privacy Analyst, Third-Party Risk Analyst, Vendor Risk Analyst, Policy Analyst, Cybersecurity Consultant (GRC), vCISO Associate.

### AppSec lane
Application Security Engineer, Product Security Engineer, AppSec Analyst, Secure Code Reviewer, SAST Engineer, DAST Engineer, DevSecOps Engineer, Security Champion, Software Security Engineer.

### Cloud Sec lane
Cloud Security Engineer, AWS Security Engineer, Azure Security Engineer, GCP Security Engineer, CSPM Engineer, Cloud Security Architect, Container Security Engineer, Kubernetes Security Engineer.

### Identity lane
IAM Engineer, IAM Analyst, Identity Engineer, Okta Engineer, Entra ID Engineer, PAM Engineer, CyberArk Engineer, Zero Trust Engineer, SSO Engineer.

### OT / ICS lane
OT Security Engineer, ICS Security Analyst, SCADA Security Engineer, Critical Infrastructure Security Analyst, Plant Security Engineer.

### Internship modifier
If the user picked role type (a), prepend each title with: `Intern`, `Co-op`, `Cybersecurity Intern`, `Security Intern`, `Summer Analyst`. Also search dedicated student programs (see Phase 3.4).

**Output the expanded title list to the user before searching** so they can sanity-check it.

---

## Phase 3 — Discovery (Minimum 20 Queries)

Track query count. Do not move to Phase 4 until you've executed at least 20 distinct searches across the categories below.

### 3.1 ATS dorking — the highest-value step

Companies post on their ATS *before* syndicating to job boards. Rotate title variants across these. Substitute the user's location.

| ATS | Pattern |
|---|---|
| Workday | `site:myworkdayjobs.com "TITLE" "LOCATION"` |
| Greenhouse | `site:boards.greenhouse.io "TITLE" "LOCATION"` |
| Lever | `site:jobs.lever.co "TITLE" "LOCATION"` |
| Ashby | `site:jobs.ashbyhq.com "TITLE" "LOCATION"` |
| SmartRecruiters | `site:smartrecruiters.com "TITLE" "LOCATION"` |
| Workable | `site:workable.com "TITLE" "LOCATION"` |
| BambooHR | `site:bamboohr.com/careers "TITLE" "LOCATION"` |
| Breezy | `site:breezy.hr "TITLE" "LOCATION"` |
| iCIMS | `site:icims.com "TITLE" "LOCATION"` |
| Jobvite | `site:jobs.jobvite.com "TITLE" "LOCATION"` |
| Recruitee | `site:recruitee.com "TITLE" "LOCATION"` |
| Pinpoint | `site:pinpointhq.com "TITLE" "LOCATION"` |
| Teamtailor | `site:teamtailor.com "TITLE" "LOCATION"` |
| ADP WFN | `site:workforcenow.adp.com "TITLE" "LOCATION"` |
| Dayforce | `site:dayforcehcm.com "TITLE" "LOCATION"` |
| Rippling | `site:ats.rippling.com "TITLE" "LOCATION"` |

**Rule:** at least 8 ATS dorks per scan, rotating title variants.

### 3.2 Direct career page bypass

```
inurl:careers "TITLE" "LOCATION" -indeed -linkedin -glassdoor -ziprecruiter
inurl:jobs "TITLE" "LOCATION" -indeed -linkedin -glassdoor
"we're hiring" "TITLE" "LOCATION" -indeed -linkedin
```

### 3.3 Cyber-specific boards

Always check these regardless of lane:

- **infosec-jobs.com** — best dedicated cyber board
- **CyberSeek** (US) — supply/demand mapping with live links
- **ISC2 Career Center**
- **Dice** — heavy on cleared/government cyber
- **ClearanceJobs.com** — only if user has clearance
- **NinjaJobs** — vetted infosec roles
- **Cybersecurity Jobsite** (UK/EU)
- **Canadian Cybersecurity Jobs** (CA)

### 3.4 Internship-specific sources (only if user picked role type a)

- **Handshake** — primary US/CA student board
- **RippleMatch**
- **WayUp**
- Big tech early-career pages: **Google STEP**, **Microsoft Explore**, **Meta University**, **Amazon Future Engineer**, **IBM Accelerate**
- **NSA Stokes / DoD CySP / CyberCorps SFS** (US, requires citizenship)
- **CSE Co-op Program** (Canada)
- **GCHQ Apprenticeships** (UK)
- University co-op portals if the user is a student

### 3.5 Regional sources

Pick by user's country:
- **Canada:** GC Jobs, GC Digital Talent, Communitech, Built In Toronto, Eluta, Career Beacon
- **USA:** USAJobs, Built In (per city), Wellfound, HN Hiring (hnhiring.com)
- **UK:** CWJobs, Reed, Totaljobs, CyberSecurityJobsite
- **EU:** Welcome to the Jungle, Otta, Honeypot (DE), Stepstone
- **Australia:** Seek, Hatch (early career)

### 3.6 Niche by lane

| Lane | Extra sources |
|---|---|
| Offensive | Pentester firms (Bishop Fox, NCC Group, Trustwave, SpecterOps, TrustedSec, Coalfire, SecureWorks, Praetorian, Mandiant, IOActive), bug bounty platforms careers (HackerOne, Bugcrowd, Intigriti) |
| Defensive | MSSP careers (Arctic Wolf, eSentire, Expel, Red Canary, Sophos, Rapid7, Critical Start), MDR vendors |
| GRC | Big 4 (Deloitte, PwC, EY, KPMG), boutique audit firms, ISO 27001 consultancies, SOC 2 boutiques (Vanta, Drata, Secureframe careers) |
| AppSec | Snyk, Veracode, Checkmarx, Semgrep, GitGuardian careers; product security teams at SaaS companies |
| Cloud Sec | Wiz, Orca, Lacework, Prisma Cloud, Aqua, Sysdig careers; cloud team pages at hyperscalers |
| Identity | Okta, Ping, ForgeRock, CyberArk, BeyondTrust, Delinea, SailPoint careers |
| OT/ICS | Dragos, Claroty, Nozomi Networks, plant operator HR portals, utility company careers |

### 3.7 Recency
For re-runs, append `posted "last 7 days"` or `"posted today"` to dorks.

### 3.8 Budget check
Count queries before moving on. Under 20 → keep searching.

---

## Phase 4 — Verification & ATS Scoring

### 4.1 Verify every link

For each candidate URL, fetch it and confirm **all four**:
1. Returns HTTP 200, not a generic redirect
2. The job title is visible on the page
3. No "expired", "filled", "no longer accepting" language
4. Location matches user's scope (or remote and user accepts remote)

If any check fails → **drop the link**. No "check the portal" entries.

### 4.2 Extract the job description text

When fetching each verified job page, capture the full job description text into a variable for that job. You'll grep it against the resume in 4.3.

### 4.3 ATS score — keyword grep, no libraries

This replaces traditional ATS matching. Pure string intersection between `resume_text.txt` (from Phase 0.2) and the fetched job description.

**Algorithm:**

1. **Build the cyber keyword universe** — a fixed list of ~150 cyber terms grouped by category:

   - **Tools:** burp, burpsuite, nmap, metasploit, wireshark, nessus, qualys, rapid7, nexpose, openvas, splunk, sentinel, crowdstrike, sentinelone, defender, carbon black, sumo logic, elastic, kibana, suricata, snort, zeek, bro, yara, sigma, mitre, cobalt strike, bloodhound, mimikatz, impacket, responder, hashcat, john, hydra, aircrack, kismet, sqlmap, gobuster, ffuf, nuclei, semgrep, snyk, checkmarx, veracode, sonarqube, trivy, clair, falco
   - **Platforms:** aws, azure, gcp, kubernetes, docker, terraform, ansible, jenkins, gitlab, github, okta, entra, active directory, ldap, kerberos
   - **Frameworks/Standards:** mitre att&ck, nist, nist csf, iso 27001, iso 27002, soc 2, pci-dss, hipaa, gdpr, ccpa, fedramp, cis, owasp, owasp top 10, sans top 25, cve, cvss, stride, dread
   - **Concepts:** penetration testing, red team, blue team, threat hunting, threat intelligence, incident response, dfir, forensics, malware analysis, reverse engineering, exploit development, vulnerability assessment, vulnerability management, siem, soar, edr, xdr, ndr, dlp, casb, sase, ztna, zero trust, iam, pam, sso, mfa, encryption, pki, tls, ssl, vpn, firewall, ids, ips, waf, soc, csirt, grc, risk assessment, compliance, audit, policy, secure code review, sast, dast, iast, sca, devsecops, cloud security, container security, api security, mobile security, network security, application security
   - **Languages:** python, bash, powershell, go, rust, c, c++, java, javascript, ruby, perl, sql
   - **Certifications:** security+, network+, cysa+, pentest+, casp+, cissp, cism, cisa, ceh, oscp, osep, oswe, osed, crtp, crto, ejpt, ecppt, pnpt, gsec, gcih, gpen, gwapt, gcia, gcfa, gcfe

2. **Filter to lane-relevant subset** based on Phase 1 question 1. (Don't penalize an offensive candidate for not having Splunk.)

3. **For each job:**
   - Lowercase both the job description and the resume text
   - From the lane-relevant keyword universe, find which keywords appear in the **job description** → that's the "required set" for this role
   - Find which of those required keywords also appear in the **resume** → that's the "matched set"
   - **ATS score = round(len(matched) / len(required) × 100)**
   - If `len(required) < 5`, the JD is too thin to score reliably → mark `ats_score = null` and flag "JD too sparse"

4. **Also capture:**
   - `matched_keywords` (list)
   - `missing_keywords` (list — what the JD asks for that the resume doesn't have; this is the gap the user can address)

### 4.4 Per-job data

Hold this in memory for each verified job (no JSON file needed — write straight to HTML in Phase 5):

| Field | Notes |
|---|---|
| company | Full name |
| title | Exact as posted |
| location | City, region |
| mode | on-site / hybrid / remote |
| level | intern / entry / mid / senior |
| salary | range or `null` (never invent) |
| posted | date if shown |
| closing | date if shown; flag urgent if ≤7 days |
| ats_score | 0-100 from 4.3 |
| matched_keywords | list |
| missing_keywords | list |
| fit_summary | one sentence: why this user, this role |
| tier | 1-4 (see 4.5) |
| apply_url | verified-live |
| source | which ATS or board |

### 4.5 Tier
- **1** Direct lane match, bullseye title and skills
- **2** Adjacent lane or slight pivot
- **3** Enterprise/brand-name, broader scope
- **4** Government, defense, cleared roles, regulated industries

### 4.6 Inclusion floor
Drop any job with `ats_score < 50`. The user wants signal, not noise. (Below 50 means the resume is missing more than half the JD's keywords — too much of a stretch.)

If `ats_score` is `null` (sparse JD), include it but sort it last.

---

## Phase 5 — Output: One HTML File

Generate a **single self-contained `jobs.html`** at the project root. No external dependencies, no CDN, no `.py` script, no Excel.

### 5.1 Structure

- **Hero header**
  - User's name + cyber lane (e.g., "Aman • Offensive Security • Kitchener, ON")
  - Scan timestamp
  - Headline stats: Total roles • New this scan • ≥85% ATS • Urgent (≤7 days)

- **Filter chips** (vanilla JS, click to toggle)
  - All • Internships • Entry • Mid • Senior
  - ≥85% ATS • 70-84% ATS • 50-69% ATS
  - Urgent • Hybrid • On-site • Remote
  - Tier 1 • Tier 2 • Tier 3 • Tier 4

- **Search box** — full-text filter across visible card content

- **Job cards** sorted by `ats_score` desc, each showing:
  - Company name + title (large)
  - Tag row: mode, level, salary, posted, closing
  - **Big circular ATS score** (color-coded: ≥85 green, 70-84 amber, 50-69 red)
  - Fit summary (one sentence)
  - **Expandable "Why this score"** showing matched + missing keywords as chips
    - Matched keywords → green chips
    - Missing keywords → red chips (these are the gaps to address)
  - **Apply Now** button → opens `apply_url` in new tab
  - URGENT badge (red, animated) if closing ≤7 days
  - Tier badge (small, top-right)

### 5.2 Design rules
- Dark theme, all CSS inline in `<style>`, all JS inline in `<script>`
- System font stack only — no Google Fonts
- Renders fully offline
- UTF-8

### 5.3 Re-run handling (no tracker file needed)
On re-runs, the agent re-generates `jobs.html` from scratch with the new scan results. To remember what was seen previously, the agent reads the **previous `jobs.html`** before overwriting and extracts apply URLs from it (they're in `href=` attributes). Any URL in the new scan that wasn't in the old file is tagged `NEW` with a green badge. Simple, file-based, zero JSON pipeline.

If `jobs.html` doesn't exist yet, every job is `NEW`.

---

## Phase 6 — Deliver

After writing `jobs.html`, present this final message to the user:

```
Scan complete.

  Lane:                  [offensive / defensive / GRC / etc.]
  Total verified roles:  [N]
  New this scan:         [N]
  ≥85% ATS match:        [N]
  Closing ≤7 days:       [N]   ← apply first

Top 5 by ATS score:
  1. [Company] — [Title] — [ATS%] — [Salary] — [Location]
  2. ...

Hidden gems (not on Indeed/LinkedIn):
  • [Company] — found via [ATS source]
  • ...

Biggest keyword gaps across your top matches:
  • [keyword]  (missing from [N] of your top roles)
  • [keyword]  (missing from [N])
  • [keyword]  (missing from [N])
  → Consider adding these to your resume if you have the experience.

Output: ./jobs.html
Open it directly in your browser, or run:
  python -m http.server 8080
  → http://localhost:8080/jobs.html

To scan again: just say "scan again" and I'll run a fresh discovery
round, then regenerate jobs.html with NEW badges on anything fresh.
```

---

## Re-run Protocol

When the user says "scan again" / "find new" / "re-scan":

1. Read existing `jobs.html` and extract all `apply_url` values into a set.
2. Run a fresh Phase 3 with recency filters (`"posted last 7 days"`).
3. Verify per Phase 4. Compute ATS scores against the same `resume_text.txt`.
4. Tag each result: if `apply_url` not in the old set → `NEW`.
5. Overwrite `jobs.html`.
6. Tell the user: *"Found [X] new roles since last scan. [Y] urgent."*

---

## Failure Modes

| Problem | Action |
|---|---|
| `python-docx` install fails | Try `--break-system-packages`, then a venv, then ask the user |
| Resume parses but profile is sparse | Show what you got, ask the user to confirm/correct inline |
| <20 queries possible (very narrow niche) | Tell the user, then go deeper on title variants and adjacent lanes |
| 0 verified jobs after full scan | Don't fabricate. Show the queries you ran and ask whether to broaden location, drop seniority constraints, or include adjacent lanes |
| Job page is JS-rendered and fetch returns empty | Try once more with a different fetcher; if still empty, drop it |
| JD too sparse for keyword scoring | Set `ats_score = null`, include the role, sort last |
| User picked clearance-required lane but has no clearance | Warn them, then exclude clearance-only roles unless they say otherwise |
| User is a student picking "internship" but resume shows 5 years of experience | Ask if they want internships, full-time entry, or both |

---

## Hard Rules (violations break the output)

- **Never** include an unverified link.
- **Never** invent salaries, dates, certifications, or requirements.
- **Never** include roles below 50% ATS unless `ats_score` is `null`.
- **Never** mix lanes — an offensive candidate gets offensive roles. Don't pad with SOC analyst jobs.
- **Never** include levels the user didn't ask for.
- **Never** include modes the user excluded.
- **Never** create a `.py` script, JSON tracker, or Excel file. One HTML file. That's it.
- **Always** read the resume text into `resume_text.txt` before scoring.
- **Always** compute ATS scores against the actual fetched JD text — no estimates.
- **Always** show matched + missing keyword chips so the user understands the score.
- **Always** wait for the Phase 1 reply before searching.
- **Always** batch parallel tool calls when supported.

---

*End of meta-prompt. The agent follows phases sequentially and never fabricates data.*

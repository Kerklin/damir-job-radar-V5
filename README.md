# Damir Job Radar v5

Automatic vacancy search for construction, infrastructure, WASH, procurement, project management and heritage roles, with explainable 0–100 fit scoring against Damir Hadžić's profile.

Three files. No API keys, no secrets, no database, no GitHub Actions, no paid services.

```text
app.py
requirements.txt
README.md
```

## What is new in v5

### UNHCR: HTTP 429 handling (reviewed and rebuilt)

| Layer | What it does |
|---|---|
| Fewer requests | Small Workday sites are read in one listing request instead of one search per keyword. |
| Session warm-up | The app first opens the careers page to receive the site's session cookies, then calls the job API with `Origin` and `Referer` headers, like a browser. |
| Browser-like connection | `curl_cffi` with a Chrome fingerprint. If it fails locally, the app switches to the standard client automatically. |
| Shared pacing | At least 1 second between requests to the same site, across all sources and all users. WFP and UNHCR share one pace because Workday sites share one front end. |
| Retry-After | Short waits (up to 30 s) are honoured inside the search. Longer waits end the search for that source, and its next automatic retry is scheduled no earlier than the site asked (between 20 minutes and 6 hours). |
| Bot check | A Cloudflare challenge is recognised. The app tries the other connection type once, then reports "blocked by bot protection" instead of retrying uselessly. |
| Circuit breaker | After any 429, the app stops fetching optional full vacancy texts for that source. Listings are kept and the status shows "OK (partial)". |
| Backup page | If Workday still refuses, UNHCR vacancies are read from `unjobs.org/organizations/unhcr`. Status: "OK (via fallback)". |
| Last good results | If everything fails, the last successful results stay visible until the retry. |

### UNICEF: source detection

- **Root cause found:** UNICEF now links vacancies as `jobs.unicef.org/jobs/<title-slug>` with no job number in the address. v4 only recognised `/job/<number>`. v5 recognises both forms, plus `jobId=` style links.
- Reads keyword search pages, category filter pages (for example WASH), the mobile listing, the main listing and the root search page.
- The same vacancy found in several layouts is merged by title, so it appears once with the best link, closing date and location.
- Reads the structured job data (schema.org JSON-LD) that career pages publish, for reliable closing dates, locations and full texts.
- Accepts any UNICEF careers address, including `www.unicef.org/careers/vacancies` and `jobs.unicef.org/`.
- Diagnosis: if UNICEF pages arrive with an unfilled template, the message says so.
- Backup page: `unjobs.org/organizations/unicef`.

### Other improvements

- **My profile** (sidebar): confirm your French or Italian level, so those language requirements count as matched. You can also set the year your experience started.
- **Filters:** grade type (international P/D, national NO/NPSA/LICA, consultancy, support), and sorting by score, closing date or newest.
- **New eligibility checks:** EU-citizens-only posts, UNDP tier/internal-only posts and rosters are flagged.
- **Tracker notes:** edit states and notes in My tracker; notes are kept when you change status.
- **Sources tab:** a "Backup URL" column, a "Test one source" button, and UNjobs organization or skills pages accepted as sources. An optional "UNjobs: WASH engineering" source is included, switched off by default.
- **Workday API addresses** (`/wday/cxs/...`) pasted by mistake are now understood correctly.
- **Two users opening the app at once** no longer search the same sites twice.

## Update your existing repository

1. Open the repository on GitHub → **Add file → Upload files**.
2. Drag in the three files → **Commit changes** (they replace the old ones).
3. Streamlit redeploys automatically. If not: app menu → **Reboot app**.
4. Open the **Sources** tab and press **Run test** for UNHCR and UNICEF.

## New deployment

1. Create a GitHub repository and upload the three files.
2. https://share.streamlit.io → **Create app** → repository, branch `main`, main file `app.py`.
3. **Advanced settings** → Python **3.12** → **Deploy**. No secrets needed.

## Run on your computer (Windows PowerShell)

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
streamlit run app.py
```

## Reading the Sources tab

| Status | Meaning | Action |
|---|---|---|
| OK / OK (cached) | Working | None |
| OK (partial) | Site asked to slow down; listings complete, some full texts skipped | None |
| OK (via fallback) | Official site refused or unreadable; backup page used | None. Links open the backup listing, which links to the official vacancy |
| Failed, showing last results | Temporary problem | None. Automatic retry shown in the message |
| Failed: rate limited | Site asked for fewer requests | Wait; retry is automatic |
| Failed: blocked by bot protection | Site refuses requests from the hosting server and no backup is set | Add a backup URL, or run the app on your computer |
| Failed: no vacancies read (…) | Page layout changed; the message shows what the page contained | Use Analyze vacancy meanwhile |

## Scoring (100 points)

Title 20 · competencies 25 · organization/sector 15 · experience 15 · education 10 · location/eligibility 5 · language 5 · contract 5.
85+ Excellent · 75+ Strong · 65+ Good · 50+ Possible.

- **Disqualified**: unrelated role family or explicit nationality restriction (hidden from main tabs; still in "Download all").
- **Review required**: strong match with a possible blocker (national post outside BiH, EU citizens only, internal/tier-restricted, licence, unlisted language).
- **Verify**: unknowns. They are never counted as matched.

## Limits

- These are public website endpoints and pages, not official APIs. They can change or block automated access without notice. Backups and last good results reduce the impact.
- Backup listings from UNjobs are third-party pages. Always apply through the official vacancy.
- Not searched automatically: LinkedIn, Indeed, UN Careers (Inspira), UNOPS. Paste those into **Analyze vacancy**.
- Streamlit Community Cloud keeps nothing between restarts. Download your tracker and settings regularly.
- If `curl_cffi` ever fails to install, delete that line from `requirements.txt`. The app works without it, with a higher chance of bot blocks.

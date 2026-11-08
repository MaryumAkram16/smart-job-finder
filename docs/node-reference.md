# Node Reference

Every node in [`workflow/smart-job-finder.json`](../workflow/smart-job-finder.json), in execution order.

| # | Node | Type | What it does |
|---|---|---|---|
| 1 | **Every 6 Hours** | Schedule Trigger | Starts the workflow every 6 hours. |
| 2 | **Search Terms** | Set | Defines `searchTerms`: "AI Automation Engineer", "Applied AI Engineer", "AI Engineer remote", "n8n automation engineer", "LLM engineer remote", "workflow automation engineer". |
| 3 | **Split Terms** | Split Out | Splits `searchTerms` into one item per term. Feeds JSearch API, SerpAPI Google Jobs, and Get My Resume. |
| 4a | **JSearch API** | HTTP Request | `GET https://jsearch.p.rapidapi.com/search` with `query = "<term> remote"`, `date_posted=3days`, `remote_jobs_only=true`, `num_pages=1`. Sends one request every 3 s (batch size 1). Auth: Multiple Headers Auth credential plus `X-RapidAPI-Host` header. `neverError` is on. |
| 4b | **Check JSearch API Status** | Code | Throws an error if the response has a status ≥ 400, or has a `message` string and no `data` array. Otherwise passes items through. |
| 5a | **SerpAPI Google Jobs** | HTTP Request | `GET https://serpapi.com/search` with `engine=google_jobs`, `q = "<term> remote"`, `chips=date_posted:week`. Auth: Query Auth credential. `neverError` is on. |
| 5b | **Check SerpAPI Status** | Code | Throws an error if the response has a status ≥ 400 or an `error` string. Otherwise passes items through. |
| 6 | **Merge & Normalize** | Code | Converts JSearch `data[]` and SerpAPI `jobs_results[]` into one shape: `id`, `title`, `company`, `location`, `description` (first 4000 chars), `url`, `posted`, `source`. Falls back to a `company-title` slug when there is no job ID. |
| 7 | **Get Seen Job IDs** | Google Sheets (read) | Reads all rows from the "Seen Jobs" sheet. Runs once, always outputs data, retries on failure (5 s wait). |
| 8 | **Remove Seen Jobs** | Code | Builds a set of `job_id`s seen in the last 7 days (by `date_seen`), then drops jobs that are already seen, duplicated within the batch, or missing `id`/`title`/`url`. |
| 9 | **Rule Pre-Filter** | Code | Free keyword filter on title + description. Drops jobs that contain a hard reject phrase (e.g. "us citizens only", "must relocate", "security clearance", "hybrid role"), a core-ML term (tensorflow, pytorch, cuda, computer vision, deep learning research), or are senior/staff/principal with 7+/8+/9+/10+ years or PhD. Keeps only jobs mentioning at least one stack term (n8n, python, openai, gpt, claude, anthropic, gemini, api, automation, llm, rag, webhook, zapier, make.com). |
| 10 | **Get My Resume** | Google Sheets (read) | Reads the resume sheet (expects a `resume_text` column). Runs once, always outputs data. |
| 11 | **Attach Resume (Merge)** | Merge | Combines jobs (input 1, from Rule Pre-Filter) with the resume row (input 2, from Get My Resume) so scoring has job fields and `resume_text`. |
| 12 | **OpenAI Deep Scoring** | HTTP Request | `POST https://api.openai.com/v1/chat/completions` with `gpt-4o-mini`, temperature 0.2, JSON response format. Rubric: stack match (40), remote-anywhere and any nationality (25), realistic experience level (20), no blockers (15). Also flags staffing-agency, vague talent pool, unpaid, or MLM-style posts. Returns `score`, `stack_match`, `location_ok`, `experience_fit`, `blockers`, `has_red_flags`, `red_flags`, `verdict`. |
| 13 | **Parse Scores** | Code | Parses each OpenAI response and merges the score fields onto the matching job from Rule Pre-Filter (by index). Skips responses that are not valid JSON. |
| 14 | **Threshold + Top 5** | Code | Keeps jobs with `score >= 50`, `location_ok === true`, and `has_red_flags !== true`; sorts by score descending and keeps the top 5. |
| 15 | **Format Email** | Code | Builds an HTML email with one card per job (title, score, company, location, source, verdict, stack match, experience fit, blockers note, "Apply Now" link). Returns `skip: true` if there are no jobs. |
| 16 | **Any Jobs Found?** | If | Continues only when `skip` is `false`. True branch goes to Send Gmail and Prep Seen IDs. |
| 17 | **Send Gmail** | Gmail | Sends the HTML to the configured address with subject `🎯 <count> new AI jobs matched (top: <topScore>/100)`. |
| 18 | **Prep Seen IDs** | Code | Builds `{ job_id, date_seen }` rows (today's date) from the Threshold + Top 5 output. |
| 19 | **Save Seen IDs** | Google Sheets (append) | Appends rows to the "Seen Jobs" sheet. The column mapping in the export only defines `job_id`; add a `date_seen` mapping if you want the 7-day window in Remove Seen Jobs to apply. |

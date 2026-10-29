# Daily remote-job agent

Each run it finds new remote jobs that pay $70K+, scores how well they fit you, and writes a
tailored resume and cover letter for the best few (default: 3 a day, matching your
1-3 applications/day goal). It **never applies for you**. You review everything first.

## Output (in `output/<date>/`)
- `summary.md` / `summary.csv`: table of company, role, salary, fit score, flags, and a link
  to each tailored application
- `jobs/<company>__<role>.md`: gaps to address, a "review before sending" list, tailored
  resume bullets, and the cover letter

## Setup (5 minutes)
1. `pip install -r requirements.txt`
2. Get an API key from the Anthropic Console and set it: `export ANTHROPIC_API_KEY=...`
3. Open `resume_facts.md` and fill in the [brackets] (CA credential, portfolio link, counts).
   This file is the **only** source the model may use. Keep it accurate.
4. Open `config.yaml` and review company slugs, title filters, and the salary floor.
5. Try it: `python job_agent.py --selftest` then `python job_agent.py --dry-run`
   (dry run fetches and filters, with no API cost), then `python job_agent.py`.

## Run it daily
**Mac/Linux (cron)**, 7:30 AM every day:
```
30 7 * * * cd /path/to/job-agent && /usr/bin/python3 job_agent.py >> agent.log 2>&1
```
Your computer must be on. For always-on, use GitHub Actions in a **private** repo
(`resume_facts.md` has your contact info). Save as `.github/workflows/daily.yml` and add
`ANTHROPIC_API_KEY` under Settings > Secrets:
```yaml
name: daily-job-agent
on:
  schedule: [{cron: "0 15 * * *"}]   # 8am Pacific (7am during standard time)
  workflow_dispatch:
jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.12"}
      - uses: actions/cache@v4
        with: {path: seen.db, key: "seen-${{ github.run_id }}", restore-keys: "seen-"}
      - run: pip install -r requirements.txt
      - run: python job_agent.py
        env: {ANTHROPIC_API_KEY: "${{ secrets.ANTHROPIC_API_KEY }}"}
      - uses: actions/upload-artifact@v4
        with: {name: "jobs-${{ github.run_id }}", path: output/}
```

## How it finds jobs (and why not LinkedIn/Indeed)
It reads the **public job-board APIs** companies publish through Greenhouse, Lever, and
Ashby, plus Remotive's public feed. LinkedIn and Indeed forbid scraping in their terms and
block bots, so they're left out on purpose; use their built-in email alerts alongside this.
To add a company, find its careers URL (e.g. `job-boards.greenhouse.io/khanacademy`) and add
the slug to `config.yaml`. A wrong slug just logs a 404 and is skipped.

## Safeguards against made-up claims
- The model sees only `resume_facts.md` and is told to use placeholders for unknowns.
- After generation, the script flags any "do not claim" term (Python, dbt, etc.), any number
  not found in your facts or the posting, and any leftover [placeholders]. These show up
  under **Review before sending**. Read them every time.

## Limits to know about
- Salary: uses the API's pay field when present, otherwise parses the text. Jobs whose salary
  was parsed from free text are flagged "verify". Jobs with no listed salary are skipped
  (change `allow_unknown_salary` to include them).
- "Remote" often means certain states only. Bare "Remote" listings are flagged "Confirm
  US/California eligibility".
- Fit scores are an LLM's judgment, so treat them as a sorting aid, not a verdict.
- Companies using Workday, iCIMS, or other systems without a public API aren't covered.
- Cost: scoring uses the small model on up to 40 jobs/day, and tailoring uses the larger
  model on at most 3. Both caps are in `config.yaml`.
- Nonprofits with no public board are best checked manually or via Idealist email alerts.
- This was tested offline with mocked responses. Live endpoints can change, so run
  `--dry-run` first and check the logs.

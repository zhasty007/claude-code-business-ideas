# Lending Desk Playbook: How to Build Each Idea in Claude Code

The detailed companion to [IDEAS.md](IDEAS.md). Each offer covers what it is, who buys it, how to build it in Claude Code, MVP scope, and pricing.

## A. Shared Claude Code toolkit (build once, use for every offer)

**Repo layout (one private GitHub repo for the team):**
```
lending-desk/
  CLAUDE.md              # team rules: no client data in git, coding style, how to run things
  .claude/
    skills/              # one folder per reusable skill (SKILL.md + scripts)
    agents/              # subagents: pdf-extractor, credit-analyst, reviewer
    commands/            # slash commands like /spread, /memo, /alco-report
    settings.json        # permissions and hooks shared by all 4 of you
  tools/                 # Python scripts the skills call (calculations live here, not in prompts)
  templates/             # Excel/Word templates for spreads, memos, reports
  sample-data/           # fake borrowers and portfolios for demos and tests
  tests/                 # pytest checks on every calculation
```

**Claude Code features you'll use over and over:**
- **CLAUDE.md:** tells Claude the team rules, e.g. "never write real client data to the repo; all formulas must live in tools/ with a test."
- **Skills:** packaged instructions plus scripts, like `spread-financials` or `draft-credit-memo`. Anyone on the team, and later a client's own Claude, can run them the same way every time.
- **Subagents:** focused helpers with their own context, e.g. one extracts numbers from PDFs, a second checks the math, a third reviews the output like a credit officer would.
- **Slash commands:** one-word workflows (`/spread borrower-folder/`) so non-technical staff can use them.
- **Hooks:** automatic guardrails, e.g. run `pytest` after every edit to `tools/`, or block any write to `sample-data/` that looks like a real SSN or account number.
- **MCP servers:** connect Claude to Google Drive, SharePoint, email, or a database so it can pick up borrower files and put finished reports back.
- **Headless mode (`claude -p`):** run a skill from a script or a scheduled job with no one at the keyboard, e.g. run every ALCO report on the 3rd business day of the month.
- **Claude Agent SDK:** when a service turns into a product, wrap the same skills in a small web app clients log into.

**Golden rule:** Claude reads documents, writes the narrative, and orchestrates the work. **Python does the math.** Every number in a deliverable comes from a tested script, so it's the same every time and you can show a bank exactly how it was calculated.

**Data safety:** use Claude through a commercial plan or the API (business terms, no training on your data), keep client files in the client's storage or an encrypted bucket, never in git. Build all demos on `sample-data/`.

---

## B. Commercial lending offers

### 1. Financial spreading automation
**What it is:** A lender drops a borrower's tax returns (1040, 1120, 1120-S, 1065, Schedules C/E/K-1), interim P&Ls, and balance sheets into a folder. Out comes a completed spread in the lender's Excel template: income statement, balance sheet, EBITDA, debt service coverage (DSCR), leverage, liquidity, and a global cash flow combining the business and its guarantors.

**Who buys:** Community banks, credit unions, and loan brokers with 1 to 5 credit analysts. Spreading is the most tedious part of their job, often 2 to 4 hours per deal.

**How to build it in Claude Code:**
1. Ask Claude Code to scaffold the repo above and a `tools/spread/` package.
2. Collect 5 to 10 sample returns (blank IRS forms filled with fake numbers). Have Claude write `tools/spread/extract.py`, which pulls line items from PDFs (pdfplumber for text-based PDFs, Claude's PDF/vision reading for scans) into a standard JSON schema, e.g. `{"form": "1120S", "year": 2025, "gross_receipts": ..., "depreciation": ...}`.
3. Have Claude write `tools/spread/calc.py` with the formulas: EBITDA, cash available for debt service, DSCR, debt/EBITDA, current ratio, global cash flow. The lending partner writes the formulas in plain English first; Claude turns them into code and pytest cases.
4. `tools/spread/excel.py` writes results into the lender's own template with openpyxl, so the output looks like what they already use.
5. Wrap it as a skill `.claude/skills/spread-financials/SKILL.md`: "Given a folder of borrower documents, identify each form, run extract.py, ask the user to confirm any field with low confidence, run calc.py, fill the template, and list every adjustment made." Add a `/spread` slash command.
6. Add a `pdf-extractor` subagent and a `spread-checker` subagent that re-adds totals and flags anything that doesn't tie (e.g. net income on the return doesn't match the P&L).

**MVP in about 2 to 3 weekends:** 1120-S and Schedule C only, one bank's template.
**Demo:** a 5 minute screen recording that turns a folder of PDFs into a finished spread.
**Pricing:** $75 to $150 per spread, or $750 to $2,000/month per bank for unlimited spreads.

### 2. Credit memo drafting assistant
**What it is:** Takes the spread from offer 1, plus the loan request, borrower background, collateral info, and the analyst's notes, and drafts a first pass of the credit memo in the bank's format: borrower overview, loan request, sources and uses, repayment analysis, collateral, guarantor strength, risk rating rationale, strengths and weaknesses, and conditions.

**Who buys:** The same banks. This is the natural upsell after spreading.

**How to build it in Claude Code:**
1. Get 2 or 3 anonymized memos from the bank and save them as `templates/memo/<bank>/example-*.md`. Claude learns the tone and section order from these.
2. Write the skill `.claude/skills/draft-credit-memo/SKILL.md` with rules: "Only use numbers from spread.json. Every ratio must cite its source cell. Flag anything outside the bank's policy (e.g. DSCR < 1.25x, LTV > 80%) in a Policy Exceptions section. Never invent facts; write [ANALYST TO CONFIRM] instead."
3. Put the bank's credit policy limits in `templates/memo/<bank>/policy.yaml` so the skill can check them automatically.
4. Add a `credit-reviewer` subagent that reads the draft as a skeptical chief credit officer and lists weaknesses and missing items.
5. Output to Word using a docx script and the bank's own memo template.

**MVP:** 1 weekend once offer 1 works.
**Pricing:** bundle with spreading at $1,500 to $3,000/month per bank, or $150 to $300 per memo.

### 3. Covenant and annual review monitoring
**What it is:** Tracks every commercial loan's covenants (minimum DSCR, max leverage, minimum liquidity, reporting deadlines). When a borrower sends financials, it recalculates the covenants, flags breaches, and drafts the annual review. It also keeps a "tickler" list of missing documents (tax returns, insurance certificates, rent rolls) and drafts reminder emails.

**Who buys:** Banks and credit unions with small loan admin teams. Missed covenant checks are a common exam finding, so this is an easy sell to a nervous compliance officer.

**How to build it in Claude Code:**
1. Have Claude design a small SQLite (later Postgres) database: loans, covenants, documents_required, documents_received, test_results.
2. Reuse `tools/spread/` from offer 1 to calculate covenant values from new financials.
3. A skill `covenant-test` loads a loan's covenants, runs the calculations, writes a pass/fail table, and drafts a waiver or breach memo if anything fails.
4. A skill `tickler-report` lists overdue documents by loan officer and drafts reminder emails. Use the Gmail or Outlook MCP connector to create drafts (never send automatically).
5. Schedule a weekly headless run (`claude -p "/tickler-report"`) with cron or GitHub Actions that emails the loan admin team a summary.
6. Later, use the Claude Agent SDK to build a simple web page where borrowers upload documents.

**MVP:** 2 to 3 weekends.
**Pricing:** $500 to $1,500/month based on number of loans.

### 4. Loan packaging for small business borrowers
**What it is:** A done-for-you service for business owners applying for an SBA 7(a), SBA 504, or conventional loan. You prepare a clean package: 3 years of historical spreads, 2 years of monthly projections, a debt service coverage analysis, sources and uses, a short business plan, a personal financial statement summary, and a document checklist.

**Who buys:** Small business owners, usually through referrals from CPAs, brokers, and banks that are tired of messy applications.

**How to build it in Claude Code:**
1. Reuse offer 1's spreading tools on the borrower's own documents.
2. Skill `build-projections`: interview the owner (a list of questions about revenue drivers, hiring, capex), then run `tools/projections.py` to build monthly projections with best, base, and worst case.
3. Skill `loan-package`: assembles everything into one PDF and a matching Excel file, using the SBA's required forms checklist as `templates/sba/checklist.md`.
4. Reuse the existing Invisible Expert writing style to draft the business plan narrative in the owner's voice.
5. A `lender-reviewer` subagent reads the package as a bank analyst would and lists what's missing before it goes out.

**MVP:** 2 weekends.
**Pricing:** $1,500 to $5,000 flat per package. Check your state's loan broker rules before charging success fees.

---

## C. Fixed income offers

### 5. Investment portfolio reporting for credit unions and community banks
**What it is:** Every month, small institutions present their investment portfolio to an ALCO (asset/liability) committee. Today someone builds that deck by hand from custodian statements. You automate it: holdings summary, book vs. market value, unrealized gains/losses, yield, duration and convexity, cash flow ladder by year, sector and rating mix, and rate shock tables (+/-100, 200, 300 bp).

**Who buys:** CFOs and investment officers at credit unions and community banks under about $1B in assets.

**How to build it in Claude Code:**
1. Get a sample holdings export (CSV/Excel from the custodian or broker) and save a fake version in `sample-data/portfolio/`.
2. Have Claude write `tools/bonds/`:
   - `pricing.py`: yield, price, modified duration, convexity from coupon, maturity, and price (use QuantLib-Python or plain numpy; the bond trading partner checks the results against Bloomberg or a broker sheet).
   - `cashflows.py`: projected principal and interest by month and year, including call dates and MBS prepayment assumptions (start simple).
   - `shock.py`: reprices the portfolio under parallel rate shifts.
   - `curve.py`: pulls the daily Treasury yield curve from treasury.gov or FRED.
3. Write pytest checks against 10 hand-verified bonds. This is where the bond trader's knowledge matters most.
4. Skill `alco-report`: load holdings, run all calculations, build charts (matplotlib), and write a short "what changed this month" commentary. Output to PowerPoint or PDF with the institution's branding.
5. Schedule a headless run on the 3rd business day of each month that pulls the new holdings file through the Google Drive or SharePoint MCP connector and drops the finished report back.

**MVP:** 3 to 4 weekends (the bond math needs careful testing).
**Pricing:** $750 to $2,500/month per institution.

### 6. Bond analytics tools for small RIAs and family offices
**What it is:** Small RIAs managing a few hundred million dollars often build bond ladders in Excel. You provide tools that load their holdings, show the maturity and call schedule, project income by month for each client, find gaps in the ladder, and run what-if scenarios (e.g. "what if these munis get called next year?").

**Who buys:** RIAs and family offices with 1 to 10 advisors.

**How to build it in Claude Code:**
1. Reuse `tools/bonds/` from offer 5.
2. Skill `ladder-builder`: given a target ladder (e.g. equal amounts maturing each year for 10 years) and current holdings, show gaps and the dollar amount needed per year. It shows the gaps and does not recommend specific bonds, so you stay a tool provider, not an advisor.
3. Skill `income-projection`: monthly coupon and principal income per client account, exported to Excel.
4. Build a simple dashboard with Streamlit (Claude Code can build this in an afternoon), protected by login, and host it for each client.
5. Later, use the Claude Agent SDK so advisors can ask plain-English questions like "which accounts have more than 20% maturing in 2027?"

**MVP:** 2 weekends after offer 5.
**Pricing:** $300 to $1,000/month per firm.

### 7. Market commentary engine
**What it is:** Advisors and small banks need regular "what's happening in rates and credit" commentary for clients and boards. You produce a weekly or monthly draft in each client's voice, with charts.

**Who buys:** RIAs, credit union investment officers, and community bank CFOs.

**How to build it in Claude Code:**
1. `tools/market_data.py` pulls Treasury yields, the fed funds rate, credit spreads, and inflation data from free sources like FRED and treasury.gov.
2. Skill `market-commentary`: builds 3 or 4 charts, then writes a 600 to 900 word commentary. The bond trader adds a short "trader's notes" paragraph each week, and Claude blends it in. That paragraph is your real differentiator.
3. Keep a voice file per client (`templates/voice/<client>.md`) with example past letters, as in the existing `debank` and `deep-dive-article` skills.
4. Run the `humanizer` skill as a last pass so it reads naturally.
5. A compliance check subagent flags phrases that sound like personalized advice or promises of returns.

**MVP:** 1 weekend.
**Pricing:** $300 to $800/month per client. This is a good add-on to offers 5 and 6.

---

## D. IT offers

### 8. Secure AI setup for regulated firms
**What it is:** A fixed-scope project to get a small bank, credit union, or RIA using AI safely: choose and set up business-tier Claude accounts, write an AI acceptable use policy, set up data handling rules, configure access controls and logging, connect approved tools (SharePoint, email), and train staff with real examples from their own workflows.

**Who buys:** The same institutions. This is often the first project, because they can't buy offers 1 to 7 until they have this in place.

**How to build it in Claude Code:**
1. Have Claude Code draft a reusable policy pack in `templates/it/`: AI acceptable use policy, data classification guide, vendor risk assessment, incident response addendum, and a staff training deck. The IT partner reviews and tightens it once; after that it's mostly find-and-replace per client.
2. A skill `ai-readiness-assessment` runs an interview (questions about systems, data types, regulators, current tools) and produces a gap report and a 30/60/90 day plan.
3. Ready-made Claude Code `settings.json` and `CLAUDE.md` templates for client use, with permissions locked down and hooks that block sensitive data patterns.
4. A skill `training-workshop` builds a hands-on session from the client's own processes (e.g. "summarize this loan file" using fake data).

**MVP:** 1 to 2 weekends for the policy pack.
**Pricing:** $5,000 to $15,000 per engagement, plus an optional $500/month support retainer.

### 9. Vendor due diligence readiness
**What it is:** Before a bank buys anything from you, it sends a security questionnaire (often 100 to 300 questions) and asks about SOC 2. You build your own answer package first, then sell the same service to other small fintech and software vendors trying to sell to banks.

**Who buys:** You first, then small software vendors and fintech startups.

**How to build it in Claude Code:**
1. Write your own security policies with Claude Code (access control, encryption, backups, incident response, vendor management, business continuity) in `templates/security/`.
2. Build an "answer library" in a simple YAML or Markdown file: one approved answer per common question, with references to the policy that backs it up.
3. Skill `questionnaire-filler`: reads a client's questionnaire (Excel or PDF), matches each question to the answer library, fills what it can, and lists the questions needing a human answer.
4. Skill `soc2-gap-check`: compares current controls to the SOC 2 criteria and produces a gap list and a remediation plan.

**MVP:** 2 weekends (doubles as your own security homework).
**Pricing:** $2,000 to $5,000 per questionnaire package, $10,000+ for SOC 2 readiness.

---

## E. The flagship: "lending desk in a box"
Combine offers 1, 2, 3, 5, and 8 into one subscription for community banks and credit unions.
- **Build order:** 8 (needed to get through the door) and 1 (fastest win) first, then 2, then 5, then 3.
- **Delivery:** start as a managed service, where you run the skills and deliver the output. Once 3 or more clients use the same workflow, build a client web portal using the Claude Agent SDK so their staff can run it themselves.
- **Pricing:** $5k to $15k setup plus $1k to $3k/month.

## F. General small business ideas (shorter)
- **Back-office automation sprints:** in the first week, Claude Code interviews the owner and maps the process. Then it builds Python scripts or Zapier/Make flows, plus a QuickBooks connection, and a short how-to guide. Reuse a `process-mapping` skill for every client.
- **AI readiness assessment:** same skill as offer 8, without the banking-specific regulations.
- **Custom skills for professional firms:** use the existing HCW law firm skills as a template. Use `skill-creator` to build and test new skills for each firm.
- **Productized services:** Invisible Expert plus content-engine for authority books, and a KPI dashboard skill that reads QuickBooks exports.
- **Part-time finance and operations retainers:** reuse the spreading and projection tools from offers 1 and 4 for monthly close and cash flow forecasts.
- **Small software products:** whatever repeats most across clients becomes an Agent SDK app.
- **Workshops:** run live demos of offers 1 and 5 at state banking and credit union league events.

## G. Suggested build calendar (first 12 weeks)
| Weeks | IT lead | Lending lead | Bond lead | Fourth partner |
|---|---|---|---|---|
| 1 to 2 | Repo, CLAUDE.md, hooks, security policies (8, 9) | Formulas and sample returns for spreading (1) | Bond math spec and test bonds (5) | LLC, pricing sheet, target list of 30 institutions |
| 3 to 6 | Hosting, client storage, Streamlit shell | Spreading MVP, then credit memo (1, 2) | `tools/bonds` and ALCO report MVP (5) | Book pilot meetings, demo videos |
| 7 to 12 | Client onboarding and support | Covenant tracking (3) | Commentary engine (7) | Case studies, league event pitch |

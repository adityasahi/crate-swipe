# ConnectRate

A contact review and learning system

## What It Does

Sales reps swipe through contacts one at a time and decide whether each contact fits their ideal customer profile (ICP). The system learns from every decision and uses that knowledge to automate the next list.

---

## How to Run this

```bash
# 1. Install dependencies (run once)
npm install

# 2. Start the app
npm start

# Opens at http://localhost:3000
```

**Requirements:** Node 16+, npm 8+. No other dependencies.

---

## The Three Phases

### Phase 1 — Swipe & Learn
 We upload CSV 1 (First Name, Last Name, Title, Company, contact Linkedin URl, email, phone). The system enriches each contact with seniority, department, company size, and industry using a deterministic lookup engine — no external API calls( could be done using an API and will be easier to load it onto a databaselike SQLite). Swipe through contacts using 5 verdict categories designed to capture nuance the system can actually learn from.

### Phase 2 — Automate With Memory
wpload CSV 2 (using the same format). The system scores every contact against the learned profile using a weighted composite fit rate across all 7 dimensions. Contacts scoring ≥ 68 are auto-approved, ≤ 32 are auto-rejected, and the uncertain middle is surfaced for manual review. The dashboard shows: "Auto-approved X, auto-rejected Y, need your input on Z."

### Phase 3 — ICP Summary
After both CSVs are processed, a full ICP profile view shows which titles, seniorities, departments, and industries consistently approved — plus the auto-decision rules that were derived.

---

## Data Model

### Verdict Categories (Phase 1)

| Key | Verdict | What It Signals |
|-----|---------|-----------------|
| 1 | Strong Yes | Right person + right ICP pattern — learn from both |
| 2 | Yes (Exception) | Like this person personally, but their pattern is off — learn person-fit only |
| 3 | Pattern Only | Right ICP profile, but not this specific person — learn pattern-fit only |
| 4 | Soft No | Borderline — low signal, de-weighted in learning |
| 5 | Hard No | Wrong profile entirely — negative signal |
| S | Skip | No decision |

The split between `pFit` (pattern fit) and `persF` (person fit) is what allows the system to distinguish "we like this type of person" from "we like this individual" — critical for accurate automation.

### Decision Schema

```ts
{
  ts:       string,           // ISO timestamp
  c:        EnrichedContact,  // full enriched contact snapshot
  verdict:  VerdictId,        // one of the 6 categories above
  note:     string,           // optional rep note
  pFit:     0 | 1,            // does this count as a pattern fit?
  persF:    0 | 1,            // does this count as a personal fit?
  autoScore?: number,         // 0–100 composite score at time of decision
  source:   "manual" | "auto_approve" | "auto_reject"
}
```

### Enrichment

Contacts are enriched client-side using deterministic lookup maps (no API keys, no latency):
- **Seniority** — regex on title (C-Suite, VP, Director, Manager, IC)
- **Department** — keyword matching on title
- **Company size** — static map of ~100 known B2B SaaS companies
- **Industry** — static map of the same companies

### Auto-Scoring Formula

```
score = (seniority_fit_rate × 0.35)
      + (department_fit_rate × 0.30)
      + (company_size_fit_rate × 0.20)
      + (industry_fit_rate × 0.15)

score ≥ 68  →  auto-approve
score ≤ 32  →  auto-reject
32 < score < 68  →  surface for manual review
```

Fit rates are only computed for dimension values with ≥ 2 examples. If a contact's dimension has no training data, it defaults to 50 (uncertain) rather than making a blind guess.

---

## Project Structure
This is how the project is structed in order to make it easy to find and edit code and files
```
src/
├── App.tsx                    # Root — 3-phase state machine
├── App.css                    # Full stylesheet
├── App.phase2.css             # Phase 2/3 additional styles
├── index.tsx                  # React entry point
├── types.ts                   # Shared TypeScript interfaces
├── enrichment.ts              # Deterministic enrichment engine
├── csvParser.ts               # CSV parser with column alias mapping
├── verdicts.ts                # Verdict config — labels, keys, colors, pFit flags
├── learning.ts                # Learning engine — fit rates, scoring, classification
├── phase2.ts                  # Phase 2 automation — processCSV2, automationRate
├── export.ts                  # CSV + JSON export helpers
├── hooks/
│   ├── useDecisions.ts        # Decision state + learning wrapper
│   └── useKeyboard.ts         # Global keyboard shortcut binding
└── components/
    ├── UploadScreen.tsx        # CSV 1 drag-and-drop upload
    ├── ContactCard.tsx         # Swipeable card with drag gestures
    ├── PatternPanel.tsx        # Live ICP pattern sidebar
    ├── LearningScreen.tsx      # Phase 1 results + continue CTA
    ├── Phase2Gate.tsx          # CSV 2 upload + auto-decision preview
    ├── Phase2Swipe.tsx         # Uncertain-only swipe queue (Phase 2)
    └── ICPSummary.tsx          # Full ICP view (Phase 3)
```

---

## Python Utilities (optional)
 this is something i wanted to include but ended up not including due to the size being only 200, and running python seems to be an over kill
```bash
# Enrich a raw contacts CSV offline
python3 icp_enrich_analyze.py --enrich # the name of the csv file

# Analyze an exported decisions CSV
python3 icp_enrich_analyze.py --analyze # the name of the csv file

# Score a new CSV against a saved profile
python3 icp_enrich_analyze.py --score  the name of the csv file  --profile icp_decisions_pattern_report.json
```

Requires Python 3.8+. No external packages — stdlib only.

---

## Tradeoffs

**Enrichment without an API** — Using lookup maps means coverage is limited to known companies. For unknown companies the system defaults to "Unknown" size and "B2B Software" industry. A real v2 would call Clearbit or Apollo for live enrichment.

**Thresholds are fixed** — Auto-approve at 68 and auto-reject at 32 are reasonable defaults but not tuned per-rep. A production system would calibrate these based on the rep's historical false-positive tolerance.

**Minimum sample size of 2** — Fit rates are only computed for dimension values with at least 2 decisions. This prevents a single fluke decision from creating a misleading rule, but it means early decisions carry less weight.

**No persistence** — Decisions live in React state. Refreshing the page resets everything. Export to CSV/JSON before closing.

# Future case

i have made the model based on use case where the weights can be changed and is only temporary, this can be improved using ML techniques again, the system will be weighed by samples and is something that can be corrected faster than it looks. this is only a sample of the work.

We can use real API calls and enrich any company where python program can handle rate limiting, retries and caching cleanly.

Using Flask and SQLite would help us save decisions and we can pick up where we left from, same goes for the users as well.

Right now I used a weighted formula to calculate but using logisitic regression or XGBoost, we can train a real classifier across multiple reps for decision making and the model improves with more feedback.

we can schedule a scoring engine where a new CSV drops in a folder, python code enriches it and then scores it and provides the results through email and can be scheduled to deliver by certain time of the  day as well.

Since you mentioned Claude to be your primary use, LLM integration should be not be an issue where we can use an API call and pass the contact to the LLM and have it study the swipes over the period of time and predict.

## How I Handled Ambiguity

The spec left several things intentionally undefined. Here's what I noticed and the deliberate call I made for each while reading the need:

**"What counts as a pattern vs a one-off decision?"**
I required a minimum of 2 decisions per dimension value beforeforming any rule. 1 decision can be considered a fluke so  the system should
not set up to auto-reject every "IC / Junior" title because one rep said no to a single SDR. Sample size gates confidence.

**"Yes/No is too coarse to learn from reliably"**
A rep saying yes to someone they like personally, despite wrong
ICP signals, can poison / causes issues with the pattern engine if treated as a
plain yes, therefore I ended up spliting  verdicts into pFit (pattern fit) and persF
(person fit) — a "yes exception" teaches personal fit only,
not pattern fit. This is the core design decision for this method.

**"When is confidence high enough to automate?"**
I chose ≥68% fit rate = auto-approve, ≤32% = auto-reject which leaves us with a 36% which can be surfaced for manual review. These thresholds
are deliberate where we can automate the obvious cases, but ensure that we do not ship a bad rep's bad taste at scale.

**"What is an ICP summary?"**
Undefined in the spec. I interpreted it as: the system's
learned best-fit value per dimension, the derived auto-rules,
and a verdict breakdown — so a sales manager can read it and
validate whether the machine learned the right thing and is available to download again for manual review or even LLM based on the prompt you choose to review it, making it easier to build an algorithm based on the feedback and data we get from the swipes, again can be automated to get a summary as pdf as well.

# crate-icp

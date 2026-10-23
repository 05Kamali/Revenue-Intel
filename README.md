# SpendNest — Revenue Intelligence Inbox


A local-first, production-grade Revenue Intelligence Inbox built with a clean SaaS interface, intuitive navigation, responsive layout, clear typography, and robust visual hierarchy. Designed to ingest raw omni-channel conversation records (telephony calls and virtual meetings), reconcile updates deterministically, track data quality, and surface actionable revenue signals for revenue operations, sales leadership, and customer success teams.

> 📖 **Looking for a comprehensive end-to-end architecture walkthrough?** See [PROJECT_GUIDE.md](file:///c:/Users/Kamali%20J/OneDrive/Folder/Volopay/Revenue/Revenue_Intelligence_Inbox_Submission/PROJECT_GUIDE.md) for an in-depth explanation of the entire system design, SaaS modules, data state machine, and analytical case studies.

---

## Table of Contents
1. [Setup and Run Steps](#setup-and-run-steps)
2. [How to Reproduce the Replay Checks](#how-to-reproduce-the-replay-checks)
   - [Option A: Automated 1-Click Verification Suite](#option-a-automated-1-click-verification-suite-recommended)
   - [Option B: Manual Step-by-Step Reproduction](#option-b-manual-step-by-step-reproduction)
3. [Where Outputs are Written](#where-outputs-are-written)
   - [Pre-Generated Physical Output Directory](#pre-generated-physical-output-directory)
   - [In-App Dynamic Manifests and Exports](#in-app-dynamic-manifests-and-exports)
4. [Metric Definitions and Denominators](#metric-definitions-and-denominators)
5. [Analysis Architecture: Pre-Generated vs. Runtime](#analysis-architecture-pre-generated-vs-runtime)
6. [Analytical Findings and Product Influence](#analytical-findings-and-product-influence)
7. [Deliberately Rejected Decisions](#deliberately-rejected-decisions)
8. [Approximate Time Spent and Effort Breakdown](#approximate-time-spent-and-effort-breakdown)

---

## Setup and Run Steps

This application was engineered with a **zero-dependency, local-first architecture**. It requires **no server installation, no build steps (`npm run build`), no Node.js runtime, no backend daemon, and no external API keys**.

### 1. Direct Browser Launch (`file://`)
1. Download or extract this project folder to your local machine.
2. Double-click [`index.html`](file:///c:/Users/Kamali%20J/OneDrive/Folder/Volopay/Revenue/Revenue_Intelligence_Inbox_Submission/index.html) (or right-click → **Open With** → Google Chrome, Microsoft Edge, Mozilla Firefox, or Safari).
3. The application will immediately launch in your browser. All embedded data fixtures (`data/01_initial.json`, `data/02_update.json`, and pre-compiled markdown dossiers) are loaded seamlessly via [`app-data.js`](file:///c:/Users/Kamali%20J/OneDrive/Folder/Volopay/Revenue/Revenue_Intelligence_Inbox_Submission/app-data.js).

### 2. Optional Local Static HTTP Server
If preferred or if corporate security policies restrict `file://` script execution:
```bash
# Using Node / npx:
npx serve .

# Or using Python 3:
python -m http.server 8080

# Or using PHP:
php -S localhost:8080
```
Then navigate to `http://localhost:8080` in your web browser.

### 3. Application State & Persistence
- **Storage Mechanism**: The application stores state in your browser profile's `window.localStorage` under the key `ri_inbox_v1` (with theme preference saved under `ri_theme_v1`).
- **Initial Boot**: On first load, the app automatically initializes with the 44 baseline records from `01_initial.json`.
- **Reset to Clean**: At any time, you can navigate to the **Import & Replay Lab** tab (`#imports`) and click **Reset to Clean Start** to return to an empty slate (0 records).
- **Dark / Light Mode**: Use the theme switcher toggle in the top-right header or sidebar to switch between Obsidian Dark Mode and Crisp Light Mode.
- **Universal Command Palette**: Press `Ctrl + K` (or `Cmd + K` on macOS) to instantly jump between views, trigger replay steps, toggle themes, or download manifests.

---

## How to Reproduce the Replay Checks

The hiring assignment specifies strict idempotency, ordering independence, and manual-override persistence contracts:
- Record identity is defined solely as the composite key: `(source_type, source_id)`.
- Batch deliveries wrap records with a `revision` integer (`01_initial.json` is revision 1; `02_update.json` is revision 2).
- Re-importing batches in any order must never produce duplicate records (total records remains **44**).
- Applying an older revision over a newer revision must be silently ignored.
- A source text update replaces source fields and marks existing analysis `pending_review` (stale), but **must strictly preserve human manager manual route overrides**.
- State must survive full browser reloads.

You can verify these invariants either through the automated in-app runner or manually step-by-step.

### Option A: Automated 1-Click Verification Suite (Recommended)
1. Open the application and navigate to the **Import & Replay Lab** view from the left sidebar (or click `#imports`).
2. Click the primary button: **Run Automated 5-Step Verification**.
3. The asynchronous test runner will automatically execute all 5 contract steps in sequence:
   - Resets state to 0.
   - Ingests `01_initial.json` (asserts 44 records, 17 usable transcripts).
   - Re-ingests `01_initial.json` in reverse array order (asserts 44 records, 0 duplicates).
   - Applies manual route override on `M07` to `customer_support` (asserts `manual_override: true`).
   - Ingests `02_update.json` (asserts 44 records, 18 usable transcripts, M07 rev 2, `pending_review` flag, preserved route `customer_support`).
   - Re-ingests `01_initial.json` in reverse order (asserts M07 remains at rev 2 and keeps `customer_support`).
   - Verifies `localStorage` serialization.
4. A live green-pass audit log will display on screen showing execution timestamps and assertion results.

---

### Option B: Manual Step-by-Step Reproduction

Follow the manual verification steps using the dedicated buttons in the **Import & Replay Lab**:

| Step | Action | Expected State & Verification Points |
| :--- | :--- | :--- |
| **1. Clean Start** | Click **Reset to Clean Start** | Active records reset to `0`. Dashboard metrics show zeros. |
| **2. Initial Import** | Click **Import 01_initial.json** | Ingests 44 records (30 calls, 14 meetings). **17 usable transcripts** (38.6% coverage). `M07` has no usable turns (`has_transcript: true`, but `turns: []`). |
| **3. Replay Reversed** | Click **Repeat Initial (Reversed Order)** | Ingests the 44 initial records in reversed array order. **Total records remains exactly 44**. Duplicate count is 0. Identical or older revisions are ignored. |
| **4. Override M07** | Click **Set M07 to Customer Support** (or open `M07` in Evidence Inspector and select Customer Support) | `M07` manual route is locked to `customer_support`. The badge `Manual Override Active` appears on the record. |
| **5. Ingest Update** | Click **Import 02_update.json** | Total unique records **remains 44** (does NOT become 45). Usable transcript count increases from **17 to 18** (coverage increases to 40.9%). `M07` source revision becomes `2` (now containing 27 turns). `M07` review state changes to `pending_review` with a visible warning: *"Source transcript updated (rev 1 → rev 2). Prior analysis marked stale pending human review."* **Crucially, the manual route `customer_support` remains intact**. |
| **6. Replay Initial** | Click **Repeat Initial (Reversed Order)** | Delivery with revision 1 arrives again. `M07` **ignores revision 1**, remaining at revision 2 with its updated 27 turns and preserved `customer_support` route. |
| **7. Persistence Check** | Refresh your browser (`F5` or `Ctrl+R`) | Open `M07` in the Evidence Inspector: revision remains `2`, 27 turns remain loaded, `customer_support` remains selected, and `pending_review` status persists. |

---

## Where Outputs are Written

Outputs are available both as pre-generated physical files in the repository and as dynamic, real-time downloads generated directly in the browser.

### Pre-Generated Physical Output Directory
All persistent file artifacts are housed inside [`exports/revenue-inbox-export/`](file:///c:/Users/Kamali%20J/OneDrive/Folder/Volopay/Revenue/Revenue_Intelligence_Inbox_Submission/exports/revenue-inbox-export/):

```
exports/revenue-inbox-export/
├── manifest.json              # Machine-readable catalog of all records, routes, review states, & relative paths
├── manifest.csv               # Tabular CSV spreadsheet version of the export manifest
├── manager-brief-2026-08-21.md# Executive daily intelligence brief for August 21, 2026
├── deal-follow-up/            # Deal progression dossiers (e.g., C03, C13, C14, C16, C18, C21, C22, C26, C27, C30)
│   ├── C03_deal-follow-up.md
│   └── ...
├── risk-review/               # Churn risk and dispute dossiers (e.g., C19, C25)
│   ├── C19_risk-review.md
│   └── ...
├── customer-support/          # Product defect & bug escalation dossiers (including M07)
│   └── M07_customer-support.md
├── sales-coaching/            # Cold-call coaching & objection handling dossiers (e.g., C01, C03, C06, C17, C20, C24)
│   ├── C01_sales-coaching.md
│   └── ...
└── marketing-feedback/        # Competitive positioning & market signals (e.g., C09, M02)
    └── ...
```

*Every `output_paths` entry in the manifests is strictly relative to the export root.*

### In-App Dynamic Manifests and Exports
Navigate to the **Exports & Manifest Hub** view (`#exports`) in the application:
1. **Interactive File Tree & Live Markdown Viewer**: Browse the folder hierarchy, click any dossier to view the rendered Markdown with syntax styling, or copy its raw Markdown to your clipboard.
2. **Download Current Export Manifest (JSON)**: Dynamically generates and downloads `manifest.json` computed from the active in-memory state.
3. **Download Current Manifest (CSV)**: Generates and downloads a spreadsheet-ready `manifest.csv`.
4. **Download Daily Manager Brief (Markdown)**: Downloads `manager-brief-2026-08-21.md`.
5. **Download Full Export Bundle (JSON)**: Generates a complete self-contained JSON archive containing the active manifest, summary metrics, and all Markdown dossier contents.

---

## Metric Definitions and Denominators

The dashboard calculates all 10 metrics **dynamically at runtime from the active dataset**. No totals are hardcoded.

You can inspect the live calculations and formulas in-app at any time by clicking the **"Audit & Denominators"** button in the dashboard header or opening the **Data Dictionary** tab.

| Metric | App Display Value | Exact Calculation & Denominator Logic |
| :--- | :--- | :--- |
| **1. Total Unique Source Records** | **44** | Distinct count of logical keys `(source_type, source_id)`. Base dataset contains 30 calls + 14 meetings = 44 records. |
| **2. Call & Meeting Counts** | **30 Calls / 14 Meetings** | Partitioned strictly by `source_type === 'call'` (30) and `source_type === 'meeting'` (14). Sums to 44. |
| **3. Activity by Day** | **Friday, 21 August 2026** | Normalized to business day in `Asia/Kolkata` timezone using `started_at` (calls) and `starts_at` (meetings). All 44 events occur on this day. |
| **4. Activity by Call Representative** | **Arun: 15, Leela: 8, Meera: 5, Kiran: 1, Dev: 1** | Grouped by `payload.rep_name` for records where `source_type === 'call'`. Represents verified telephone sales representatives. Total: 30 calls. |
| **5. Meeting Activity by Organiser (Displayed Separately)** | **team08: 3, team09: 3, team06: 2, team04: 2, team03: 2, team02: 1, team01: 1** | Grouped by `payload.organizer_email` for records where `source_type === 'meeting'`. **Displayed in a separate dedicated panel from call reps** because calendar organisers are scheduling metadata accounts, not verified sales reps or active conversation participants. Total: 14 meetings. |
| **6. Usable Supplied Transcript Count & Coverage %** | **17 / 44 (38.6%)** initially;<br>**18 / 44 (40.9%)** post-update | Records where conversation text is non-empty (`payload.transcript` has characters or `transcript_payload.turns` has length > 0). Denominator is all 44 source records. Missing text is an assessment redaction or upstream no-speech condition, not proof of zero activity. |
| **7. Routing Assignment Counts** | **23 routed assignments** across 5 categories:<br>• Deal Follow-up: 10<br>• Sales Coaching: 6<br>• Customer Support: 3<br>• Marketing Feedback: 2<br>• Risk Review: 2 | Sum of all assigned destinations. **May overlap** because a conversation can legitimately route to multiple departments (e.g., C03 routes to both Deal Follow-up and Sales Coaching). |
| **8. Records Awaiting Review** | **27 records** (initial and post-update) | Records with `review_state === 'needs_review'` or `review_state === 'pending_review'`. Includes records needing initial text triage or post-update analysis re-synthesis. |
| **9. Missing / Unusable Conversation Text** | **27 records** initially;<br>**26 records** post-update | Records where `has_usable_transcript === false` (null/empty transcript or empty turns array). Post-update, M07 receives 27 turns, reducing this count by 1. |
| **10. Stale or Pending Analysis** | **0** initially;<br>**1** post-update | Records where source text was revised under an existing analysis (`analysis_revision === null && review_state === 'pending_review'`). Flags M07 after `02_update.json` is ingested. |

---

## Analysis Architecture: Pre-Generated vs. Runtime

The assignment asks whether analysis is pre-generated or generated at runtime. Here is the exact architectural disclosure:

### What is Pre-Generated
- **Qualitative Evidence Dossiers**: Conversation summaries, verbatim quote extracts, 1-based turn citations, next actions, and initial suggested routing taxonomy are **pre-generated human-assisted review aids**.
- **Reasoning**: This adheres strictly to the hiring prompt constraint: *"Conversation summaries and classification_reason bodies are deliberately withheld... so they do not provide ready-made answers... No automatic model call is required."* Pre-generating verifiable, quote-grounded dossiers guarantees high-fidelity analytical depth without introducing unreliable hallucinations, runtime latency, or external cloud dependencies.

### What is Dynamically Computed at Runtime
- **Dashboard Metrics & Aggregations**: All 10 dashboard KPIs, call rep distributions, meeting organiser distributions, hourly timelines, and coverage ratios are computed in real-time from `state.records`.
- **Search & Multi-Faceted Filters**: Real-time multi-facet indexing across source type, rep/organiser, destination route, text usability, and manual override status.
- **Replay & Idempotency Engine**: Source deduplication, composite key tracking `(source_type, source_id)`, ordering-independent replay, and revision comparison.
- **Manual Override Protection**: Locking manual routes in `localStorage` and preserving them against subsequent batch updates.
- **Stale Analysis Lifecycle**: When an update arrives with new text (such as `M07` revision 2), the system dynamically detects that the transcript changed, resets `analysis_revision` to `null`, and marks `review_state: 'pending_review'`. It never pretends stale analysis matches new source text.
- **Dynamic Manifests**: Generation of real-time `manifest.json` and `manifest.csv` based on current client state.

---

## Analytical Findings and Product Influence

The application's evidence inspector and routing engine were directly shaped by three qualitative findings in the dataset:

### 1. M07 — Workaround is Not Resolution
- **Evidence**: Meeting `M07`, turns 8–15. The exact post-update transcript states: *"The workaround does not mean the direct upload issue has been resolved."*
- **Product Influence**: In the initial delivery, text was withheld. Upon ingesting `02_update.json`, the 27 turns reveal a high-severity direct upload defect. The app routes `M07` to **Customer Support**, flags the record as `pending_review`, provides an interactive **M07 Revision Diff Modal** comparing rev 1 vs. rev 2, and isolates engineering action items from sales pipeline tracking.

### 2. C03 — Tentative Interest with an Agreed Follow-Up
- **Evidence**: Telephone call `C03`, exact verbatim quote: *"Send the email and we can speak Monday or Tuesday."*
- **Product Influence**: The prospect agreed to a specific follow-up window but did not sign a contract or commit budget. The app assigns a **multi-destination route** (`deal_follow_up` + `sales_coaching`). The deal follow-up task specifies sending the agreed email before Monday, while coaching ensures the rep does not mark this deal as "closed-won" prematurely.

### 3. C01 — Explicit Rejection
- **Evidence**: Telephone call `C01`, exact verbatim quote: *"I didn't ask for this call, and I'm not interested."*
- **Product Influence**: Cold outreach encountered explicit boundary rejection. The app routes `C01` to **Sales Coaching** and suppression lists, advising sales management to coach reps on respecting prospect opt-outs rather than logging the call as an active opportunity.

---

## Deliberately Rejected Decisions

### 1. Rejecting Automatic Overwrite of Manual Routes on Data Ingestion
- **Rejected Alternative**: Automatically resetting a manager's chosen route whenever a newer source record arrives.
- **Rationale**: If a sales operations manager manually re-routed `M07` to `customer_support`, an automated system that silently overwrites this decision erases critical human domain knowledge.
- **Chosen Architecture**: The application preserves `manual_override: true` and retains the user's manual route, while setting `review_state: 'pending_review'` to inform the manager that new source text is available for inspection.

### 2. Rejecting Synthetic Deal Sizes and Rep Leaderboards
- **Rejected Alternative**: Inventing speculative CRM contract values, win probabilities, or ranking reps on leaderboards.
- **Rationale**: The dataset is an abridged sample without authoritative CRM opportunity records or historical deal outcomes. Ranking sales reps on a 44-record sample would introduce misleading management bias.
- **Chosen Architecture**: Displaying factual activity counts and verifiable transcript evidence without speculative metrics.

### 3. Rejecting Co-mingling of Meeting Organisers with Call Reps
- **Rejected Alternative**: Combining meeting organisers and telephone callers into a single "Rep Performance" table.
- **Rationale**: Meeting organisers are calendar metadata accounts (`team08`, `team09`), not verified attendees or closers.
- **Chosen Architecture**: Two completely distinct dashboard panels with an explicit boundary disclaimer.

---

## Approximate Time Spent and Effort Breakdown

Total estimated effort represented by this prototype: **~7.5 hours**.

| Phase / Component | Approximate Time | Details |
| :--- | :--- | :--- |
| **1. Data Contract & Schema Analysis** | 1.5 hours | Deep inspection of `Assignment.pdf`, `01_initial.json`, `02_update.json`, transcript payloads, timestamps, and redaction patterns. |
| **2. Architecture & Replay State Engine** | 2.0 hours | Designing deterministic composite key tracking `(source_type, source_id)`, ordering-independent idempotency, revision updates, manual override persistence in `localStorage`, and the automated 1-click test suite. |
| **3. Executive Dashboard & Metric Computation** | 1.5 hours | Building responsive 8-card KPI deck, separated call rep vs. organiser panels, hourly progression timeline, routing overlap matrix, and live audit modal. |
| **4. SaaS Interface & Evidence Inspector** | 1.5 hours | Implementing 4-pillar evidence cards, interactive simulated audio waveform scrubber, side-by-side M07 revision diff engine, universal `Ctrl+K` command palette, and Obsidian Dark / Crisp Light themes. |
| **5. Exports Hub, Verification Testing & Docs** | 1.0 hour | Interactive manifest explorer, live markdown viewer, dynamic file downloads, headless sandbox test runner (`scratch/test_inbox.js`), and comprehensive documentation (`README.md`, `DATA_CONTRACT.md`, `DATA_DICTIONARY.md`). |

---

*SpendNest Revenue Intelligence Inbox — Built for the Volopay Senior Full-Stack Engineer / Product Designer Assignment.*

# Project Research Summary

**Project:** DM Ops Hub — Email-to-Booking Automation Pipeline
**Domain:** LLM-native email pipeline (Microsoft Outlook + Claude + Google Sheets)
**Researched:** 2026-02-25
**Confidence:** MEDIUM-HIGH — Core design grounded in first-party project documents and prior session research; Python library versions and MCP server details require verification at install time.

## Executive Summary

DM Ops Hub is a bespoke email-to-booking automation system for Diversified Marketing, a South African promoter staffing agency. The system replaces a wholly manual process — Daniel and Dinah reading Outlook email chains, mentally tracking booking requests, and manually producing Activation Plans — with an LLM-native pipeline where Claude extracts structured booking data from casual, unstructured emails and writes it to a Google Sheets tracker the team already uses daily. No commercial tool handles this domain correctly: generic automation tools (Zapier, Make) require structured form intake that clients won't use, and enterprise CRMs are operationally overkill. Claude's ability to parse "we need 20 dry shifts in GP next month" into a validated booking record is the core value proposition.

The recommended approach is a 5-layer architecture: Outlook (email source) → Python polling script on Windows Task Scheduler (continuous monitoring, no AI) → MCP servers for Outlook and Google Sheets (Claude's API clients) → Claude intelligence layer (parsing, classification, extraction, AP generation) → Google Sheets as the single source of truth for all state. The polling script decouples continuous monitoring from Claude sessions, meaning emails are captured even when Claude is not running. This is the central architectural decision: the queue-based staging pattern using the Inbox Queue sheet as a buffer between lightweight polling and heavyweight AI processing.

The primary risks are trust erosion and silent failure. Trust erodes when Claude creates phantom booking rows (false positives), overwrites team manual edits (simultaneous edit conflicts), or extracts wrong field values without flagging uncertainty. Silent failure happens when OAuth tokens expire without alerting the team, when attachment-only emails are skipped without creating a review row, or when AP auto-generation fires on incomplete data. Both risk categories are fully preventable if the data model and error signalling are designed correctly before any write operations are coded. The correct sequencing is: infrastructure and auth first, then data model with column ownership rules and the three-state field model (extracted/pending/unsure), then parsing in shadow mode with human review, then live writes only after accuracy is validated on real emails.

---

## Key Findings

### Recommended Stack

The stack is deliberately minimal. Both MCP servers are Python-based, which locks the polling script runtime to Python 3.11+ — no second runtime to manage. The Anthropic SDK (claude-sonnet-4-5 by default, opus-4-5 only for genuinely ambiguous cases) provides the intelligence layer. MSAL for Python handles Microsoft OAuth with no manual token management. gspread handles simple Sheets row appends in the polling script; google-api-python-client handles formatting for AP generation. Windows Task Scheduler runs the poller — no Celery, no Redis, no cloud hosting required.

For Outlook MCP, ryaker/outlook-mcp (self-hosted) is recommended over Composio for production — Composio is acceptable for Phase 1 prototyping only, since it adds a third-party SaaS dependency to a business-critical email pipeline. For Google Sheets MCP, xing5/mcp-google-sheets is the primary recommendation. Both MCP servers authenticate against the same Azure App Registration and Google Cloud service account that the polling script uses.

**Core technologies:**
- **ryaker/outlook-mcp** (self-hosted): Claude's Outlook read access via Microsoft Graph API — eliminates third-party SaaS dependency on core pipeline
- **xing5/mcp-google-sheets**: Claude's Google Sheets read/write access — Python-based, service account auth, no user interaction required
- **Python 3.11+**: Single runtime for polling script and both MCP servers
- **MSAL for Python (1.28.x)**: Microsoft OAuth 2.0 token management — handles refresh automatically; do not implement manually
- **gspread (5.x) + google-api-python-client (2.x)**: Sheets access from polling script — gspread for data writes, google-api-python-client for AP formatting
- **Anthropic SDK (0.28.x+) + claude-sonnet-4-5**: Intelligence layer — Sonnet handles 95%+ of extraction at 5x lower cost than Opus
- **Windows Task Scheduler (built-in)**: Polling trigger every 15 minutes — no additional infrastructure needed

See `.planning/research/STACK.md` for full version compatibility, authentication architecture, and alternatives considered.

### Expected Features

The system has a clear v1 scope anchored by 10 P1 features that must all land for the team to abandon the current email-and-memory approach. The highest-complexity P1 item is LLM-based field extraction from casual/unstructured email text — this is also the system's core value proposition over any template-based tool. The Inbox Queue pattern (polling script + queue sheet) is architecturally foundational and must be treated as P1 infrastructure, not an optional enhancement.

**Must have — table stakes (v1 launch):**
- Email reading from Outlook inbox via MCP
- Booking request detection (LLM classification, not keyword-only)
- Field extraction from casual/unstructured email text
- Missing field detection with explicit flagging (three-state model: extracted/pending/unsure)
- Write new booking to Google Sheets tracker with idempotency check
- Booking status tracking (Received → Pending → Confirmed → AP Created)
- Audit trail: email source link on every booking row
- Per-employee to-do list generation from morning emails
- Inbox Queue pattern (Python polling script + queue sheet)

**Should have — differentiators (v1.x, after validation):**
- AP auto-generation trigger on "Confirmed" status (with field validation gate)
- Client brand colour on AP headers with holding company resolution
- Confidence scoring per extracted field
- Rate validation via dm-quoter skill
- Forwarded email chain awareness (pre-processing to split forwarding wrapper from original)

**Defer to v2+:**
- Cross-chain deduplication (heuristic subject+client+date matching)
- Smart role-based routing for to-do lists
- Webhooks/event-driven email processing (requires public HTTPS endpoint)
- WhatsApp monitoring
- Claude for Sheets add-on

**Anti-features (never build):**
- Auto-sending emails on behalf of team members
- Auto-confirming bookings based on email text
- Full natural language chat interface for the tracker
- Client-facing booking portal

See `.planning/research/FEATURES.md` for the full feature dependency tree and prioritization matrix.

### Architecture Approach

The architecture enforces a hard boundary between deterministic work (Python polling script) and judgment-required work (Claude). The polling script does only what can be expressed as `if X then Y` without ambiguity: check for new emails, apply keyword pre-filter, write email metadata to the Inbox Queue sheet, save the last-checked timestamp. Claude does everything requiring natural language understanding: classify email intent, extract booking fields, detect missing data, generate to-do lists, trigger AP creation. This boundary is non-negotiable — pulling Claude API calls into the polling script blurs failure modes and turns a lightweight background task into a slow, expensive process.

Google Sheets is the single source of truth for all booking state. No local database. No sync between Daniel's two machines. Team members edit the tracker directly; Claude reads and writes via the Sheets MCP. Column ownership must be defined before the first write: Claude owns extraction columns (Client, Brand, Email Source, Dates as extracted); team owns operational columns (Store List when sent later, Confirmed Dates, Assigned To, Notes).

**Major components:**
1. **Python Polling Script** — Runs every 15 min via Task Scheduler; authenticates via MSAL Client Credentials; fetches new emails from Graph API; applies keyword pre-filter; writes email metadata rows to Inbox Queue sheet; saves last-checked timestamp to state.json
2. **Outlook MCP Server (ryaker/outlook-mcp)** — Gives Claude read access to full email bodies during sessions; handles OAuth token refresh; used for `read_email` calls after Claude finds email IDs in the queue
3. **Google Sheets MCP Server (xing5/mcp-google-sheets)** — Gives Claude read/write access to Inbox Queue, Bookings Tracker, To-Do Sheets, and AP tabs
4. **Claude Intelligence Layer** — Email classification, field extraction, missing field detection, status determination, to-do generation, AP trigger validation; runs in Claude Code sessions
5. **Google Sheets (4 logical areas)** — Inbox Queue (staging, cleared after processing), Bookings Tracker (team-facing, primary operational surface), To-Do Sheets (per employee, regenerated each session), AP tabs (auto-generated, linked from tracker)

**Key patterns:**
- Queue-based email staging: decouple monitoring (continuous) from processing (on-demand)
- Claude as stateless processor: all state in Sheets; Claude reads on session start, writes results
- Partial data acceptance: write booking row immediately on detection, flag missing fields, track completeness percentage
- Polling over webhooks: local machine cannot receive Graph API push notifications; polling every 15 min is sufficient for this use case

Build order driven by dependency graph: Azure App Registration + Google Cloud project → MCP servers + Bookings Tracker schema → Polling script + auth verification → Claude email processing in shadow mode → Live writes after accuracy validation → AP auto-generation.

See `.planning/research/ARCHITECTURE.md` for full data flow diagrams, anti-patterns, and scaling considerations.

### Critical Pitfalls

Nine pitfalls identified, all preventable. The most severe are those that create silent failures (team doesn't know emails are missed) or erode trust through data corruption (overwrites, phantom rows).

1. **OAuth token refresh failure causes silent queue stagnation** — Add an `AUTH_FAILURE` sentinel row to the Inbox Queue on any 401/403 response; verify refresh token behaviour before Phase 2 begins; implement last-poll timestamp check so the team sees if the queue goes cold
2. **Forwarded email chains misattribute the client** — Build a pre-processing function to split forwarding wrapper from original email body; explicitly instruct Claude in the parsing prompt to extract from original content; cross-reference extracted client name against known client list (team member names = misparse signal)
3. **Over-automating before validating extraction accuracy** — Run Claude in shadow mode (writes to staging sheet, team reviews every row) for the first 50 real emails before enabling live tracker writes; set per-field accuracy thresholds before trusting automation
4. **False positive booking detection from non-booking emails** — Keyword match is pre-filter only; Claude makes the final classification; low-confidence classifications route to Review Queue tab, never directly to Bookings Tracker; never auto-update existing rows from follow-up emails
5. **Simultaneous edit data loss (Claude overwrites team manual edits)** — Define column ownership before writing any code; Claude-owned columns vs. team-owned columns; Claude checks last-modified timestamp before writing to any row; never overwrite team-owned columns programmatically
6. **Missing fields treated as empty rather than pending** — Implement three-state model from day one: "Claude extracted: [value]" / "Pending: client hasn't provided" / "?: Claude unsure — verify"; Completeness % column gates status progression to Confirmed
7. **AP auto-generation fires before all required fields are present** — Separate "Confirmed (Pending Data)" from "Confirmed (Ready for AP)"; validate required AP fields programmatically before triggering dm-activation-planner; add 15-minute delay between status set and AP trigger

See `.planning/research/PITFALLS.md` for remaining pitfalls (Google Sheets API rate limits, attachment-only emails), technical debt patterns, integration gotchas, and the "Looks Done But Isn't" production checklist.

---

## Implications for Roadmap

Based on research, the build order is determined by the component dependency graph, not by feature value. Nothing works until Azure App Registration and Google Cloud project exist. Claude can't process emails until the Bookings Tracker schema exists and MCP servers are authenticated. AP generation can't be trusted until extraction accuracy is validated on real emails. This produces a natural 4-phase structure.

### Phase 1: Infrastructure and Authentication Foundation

**Rationale:** Azure App Registration and Google Cloud project are blockers for everything. This is also the correct moment to establish security baselines (.env, .gitignore, service account sharing) because fixing credential leaks retroactively is costly. ARCHITECTURE.md explicitly identifies the Google Cloud setup as the "highest-priority first action" because it unblocks both this project and dm-activation-planner simultaneously.

**Delivers:**
- Azure App Registration with Mail.Read application permission (for polling script) and delegated permissions (for Outlook MCP)
- Google Cloud project with Sheets API enabled, service account created, JSON key secured
- Outlook MCP server (ryaker/outlook-mcp) installed, authenticated, verified reading emails in Claude Code
- Google Sheets MCP server (xing5/mcp-google-sheets) installed, authenticated, verified reading/writing Sheets
- Bookings Tracker Google Sheet created with full column schema (booking ID, client, brand, holding company, shift type, base rate, dates, store list, province, status, missing fields, completeness %, email source, AP link, last modified)
- .env.example committed to repo; secrets in .gitignore; mcp-config-example.json committed

**Features addressed:** Email reading infrastructure, Google Sheets write infrastructure (prerequisites for all P1 features)

**Pitfalls avoided:** Security mistakes (credential commits), OAuth token handling designed before first use

**Pitfall to verify before Phase 2:** Trigger an auth failure scenario and confirm the sentinel row appears in Inbox Queue

**Research flag:** Standard patterns — Azure App Registration and Google Cloud setup are well-documented. Skip `/gsd:research-phase` for this phase.

---

### Phase 2: Polling Script and Data Model

**Rationale:** The polling script must exist before Claude processing can be tested against real emails — it provides the Inbox Queue that Claude drains. The data model (three-state fields, column ownership, status enum) must be locked in before any write operations are coded. Retrofitting these after the first writes are live means rewriting the most consequential logic under pressure.

**Delivers:**
- `scripts/polling/email_poller.py`: MSAL Client Credentials auth, Graph API email fetch with `$select` (not full body on list call), keyword pre-filter, Inbox Queue append, state.json timestamp management, `AUTH_FAILURE` sentinel on 401/403
- `scripts/polling/keyword_filter.py`: configurable keyword list, returns boolean flag per email
- Forwarded email pre-processor: splits forwarding wrapper from original body; labels segments for Claude
- Windows Task Scheduler registration for poller (every 15 minutes)
- `schemas/booking.json`: three-state field model, required vs optional field lists, status enum values matching Google Sheets dropdown validation
- `schemas/sheet-columns.json`: column definitions, ownership map (Claude-owned vs team-owned), write rules
- Column ownership enforcement in write code: Claude never writes to team-owned columns
- Completeness % formula in Bookings Tracker
- Batch write logic with 429 error handling and retry-after

**Features addressed:** Inbox Queue pattern, idempotent processing, booking status tracking, missing field detection, audit trail (email source link)

**Pitfalls addressed:** OAuth token refresh failure (sentinel row + monitoring), forwarded email confusion (pre-processor), simultaneous edit data loss (column ownership), missing fields as empty (three-state model), Sheets API rate limits (batch writes + 429 handling)

**Research flag:** Standard Python + Graph API patterns. The forwarded email pre-processing function may benefit from testing against 5-10 real Sheilagh/Dinah forwarded emails before this phase closes — treat as a validation task, not a research task.

---

### Phase 3: Claude Email Processing in Shadow Mode

**Rationale:** PITFALLS.md is explicit: do not enable live tracker writes until Claude's extraction accuracy is validated on real DM emails. Shadow mode (Claude writes to a staging sheet; team reviews every row before it moves to the live tracker) is the only safe path to building team trust. This phase is also where prompts are refined iteratively against actual email variance, not hand-picked examples.

**Delivers:**
- `prompts/booking-detector.md`: classification prompt (new booking / follow-up to existing / non-booking); confidence routing (low confidence → Review Queue tab, not Bookings Tracker); negative examples including rate card updates, availability responses, internal ops emails
- `prompts/email-parser.md`: field extraction prompt with forwarded email handling instructions; three-state output format; known client list cross-reference instruction; HTML stripping pre-processing
- `prompts/todo-generator.md`: per-employee to-do list prompt; intent categorisation (booking / follow-up / FYI / action); role routing rules (Dinah: instore ops; Viviene/Rae: APs; Daniel: strategic items)
- Shadow mode staging sheet: Claude writes extracted bookings here; team reviews and approves before live tracker write
- Classification audit tab: logs every classification decision (email, sender, classification, confidence) for the first 100 emails
- Accuracy tracking against 50+ real historical emails; per-field accuracy scores documented
- Promotion to live mode only after accuracy thresholds met: Client >95%, Shift Type >90%, Dates >80%, Rate >85%

**Features addressed:** Booking request detection, field extraction from casual text, missing field detection, per-employee to-do lists, confidence scoring per field, forwarded email chain awareness

**Pitfalls addressed:** Over-automating before validating accuracy (shadow mode gate), false positive booking detection (classification prompt with confidence routing), forwarded email chain confusion (parsing prompt instructions)

**Research flag:** The email-parser prompt will need iterative refinement against real DM email variance — schedule 2-3 prompt iteration cycles before declaring this phase complete. No external research needed; the iteration is empirical.

---

### Phase 4: Live Pipeline and AP Auto-Generation

**Rationale:** Live writes to the Bookings Tracker are only enabled after Phase 3 accuracy validation. AP auto-generation is the highest-value v1.x feature and the hardest to get right — it depends on all upstream phases being stable. Triggering AP on bad data is worse than not having AP generation at all (PITFALLS.md, Pitfall 9). The dm-activation-planner skill already exists and needs only a Google Sheets pivot from Excel; the blocking dependency is reliable upstream data.

**Delivers:**
- Live write promotion: Claude writes directly to Bookings Tracker after accuracy gate passed
- Booking ID stability: lookup by ID value, not row number; row insertion robustness verified
- `prompts/ap-trigger.md`: AP trigger validation logic; field completeness check before triggering dm-activation-planner; "Confirmed (Pending Data)" vs "Confirmed (Ready for AP)" state separation; 15-minute delay between status set and trigger
- `config/brand-colours.json`: client/brand → hex colour lookup table (populated by Daniel)
- dm-activation-planner Google Sheets pivot: AP generation outputs to Google Sheets instead of Excel
- AP auto-sharing: generated AP sheets shared with all team member emails at creation time
- AP link written back to Bookings Tracker booking row
- Status advancement to "AP Created" after successful generation
- `docs/RUNBOOK.md`: day-to-day operation guide for Daniel

**Features addressed:** AP auto-generation, client brand colour on AP headers, holding company resolution, rate validation via dm-quoter (add alongside AP generation), live tracker writes with idempotency

**Pitfalls addressed:** AP fires before data complete (field validation gate + state separation), AP links requiring re-sharing (auto-share at creation), booking ID stability

**Research flag:** dm-activation-planner Google Sheets pivot may need a focused session to understand the current Excel output format and map it to Sheets equivalents. Flag for `/gsd:research-phase` during Phase 4 planning if the skill's current output structure is undocumented.

---

### Phase Ordering Rationale

- **Auth before everything:** No component functions without Azure App Registration and Google Cloud credentials. Setting them up first also enforces security baselines early when they're cheap to get right.
- **Data model before writes:** Column ownership, three-state fields, and status enums must be locked in before any code writes to the Sheets tracker. These are the hardest schema decisions to change retroactively.
- **Shadow mode before live mode:** The team's trust in the system is the primary success metric for v1. A single phantom booking or lost manual edit early in the rollout will cause the team to revert to email-and-memory. Shadow mode builds evidence before trust is demanded.
- **AP generation last:** AP generation depends on correct field extraction (Phase 3), a populated brand-colours.json (human task), and dm-activation-planner's Google Sheets pivot (existing skill refactor). All three must be complete before Phase 4 is attempted.
- **To-do lists alongside Phase 3:** Per-employee to-do lists share the same email processing infrastructure as booking detection. They can be built in parallel with Phase 3's classification and parsing work without adding a separate phase.

### Research Flags

Phases needing deeper research during planning:
- **Phase 4 (AP Integration):** If dm-activation-planner's current Excel output format is undocumented or complex, run `/gsd:research-phase` before Phase 4 planning to map the existing skill's output structure to the Google Sheets equivalent.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Infrastructure):** Azure App Registration and Google Cloud setup are thoroughly documented. ryaker/outlook-mcp and xing5/mcp-google-sheets both have README setup guides. Verify MCP server Python version requirements at install time.
- **Phase 2 (Polling Script):** Python + MSAL + Graph API is a well-established pattern. The forwarded email pre-processor is custom code, but the logic is straightforward string parsing — no research needed, just testing against real emails.
- **Phase 3 (Claude Processing):** Prompt engineering is iterative, not researchable. The accuracy thresholds and shadow mode gate are implementation decisions, not unknowns.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | Core MCP server choices verified from prior session research (2026-02-24). Python library versions reflect training knowledge (cutoff August 2025) — verify exact versions at install time via PyPI. MCP server Python version requirements (ryaker: 3.10+, xing5: 3.9+) are LOW confidence — check repo READMEs at install. |
| Features | HIGH | Feature analysis grounded in PROJECT.md (primary source) and prior architecture research. Domain is first-party — no competitor feature guesswork needed. Anti-feature list is well-reasoned against the specific operational constraints of a 5-person agency. |
| Architecture | HIGH | Based on prior session research (architecture-overview.md from 2026-02-24) and established patterns for Microsoft Graph + MCP + Google Sheets integrations. The queue-based staging pattern and polling-over-webhooks decision are sound for the local-first deployment constraint. |
| Pitfalls | MEDIUM | OAuth lifecycle behaviour, Sheets API rate limits, and Graph API pagination are well-documented in training data. Forwarded email chain structure and specific Claude extraction failure modes on DM's actual email corpus are inferred from general LLM automation patterns — validation against real DM emails is required during Phase 3. |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **MCP server version compatibility:** ryaker/outlook-mcp and xing5/mcp-google-sheets Python version requirements and dependency conflicts are LOW confidence. Check each repo's README and requirements.txt before Phase 1 installation. Resolve any conflicts before committing to the stack.
- **Composio vs ryaker decision point in Phase 1:** If ryaker/outlook-mcp proves difficult to configure (Azure App Registration complexity, token storage), Composio is an acceptable Phase 1 fallback to unblock email reading — but must be migrated to ryaker before Phase 4 goes live. Document the migration path explicitly if Composio is used in Phase 1.
- **dm-activation-planner current output format:** The existing skill outputs to Excel. The Google Sheets pivot required for Phase 4 AP generation depends on understanding the current template structure. If this is undocumented, a brief research session before Phase 4 planning is warranted.
- **Real DM email corpus for Phase 3 validation:** The shadow mode accuracy validation requires access to 20-50 real historical DM booking emails. Daniel must confirm availability of these before Phase 3 planning. Redaction may be needed for privacy.
- **Brand colour lookup table population:** The `config/brand-colours.json` file is a human-populated data dependency for AP generation. Daniel must provide the authoritative client/brand → hex colour mapping. This is not a technical task but it blocks Phase 4.

---

## Sources

### Primary (HIGH confidence)
- `.planning/PROJECT.md` — Booking data model, team structure, scope boundaries, daily workflow requirements
- `.planning/research/architecture-overview.md` (2026-02-24) — Authentication flows, monitoring strategy, Sheets data model
- `.planning/research/outlook-mcp-options.md` (2026-02-24) — Composio vs ryaker vs Zapier vs Anthropic connector comparison
- `.planning/research/google-sheets-mcp-options.md` (2026-02-24) — xing5 vs ringo380 vs Workspace MCP comparison

### Secondary (MEDIUM confidence)
- Training knowledge (cutoff August 2025) — Python library versions (MSAL 1.28.x, gspread 5.x, google-api-python-client 2.x, anthropic SDK 0.28.x+), Microsoft Graph API token lifecycle, Google Sheets API rate limits (300 read/write requests per minute per project), LLM automation pipeline failure modes
- Microsoft Graph API documentation (training data) — v1.0 vs beta endpoint stability, Client Credentials vs Authorization Code flow, pagination via `@odata.nextLink`, `$select` parameter behaviour

### Tertiary (LOW confidence)
- ryaker/outlook-mcp Python version requirement (3.10+) — unverified; check repo README at install
- xing5/mcp-google-sheets Python version requirement (3.9+) — unverified; check repo README at install
- Composio token expiry behaviour on their managed tier — not documented in available training data; test the failure case explicitly if Composio is used in Phase 1

---

*Research completed: 2026-02-25*
*Ready for roadmap: yes*

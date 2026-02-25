# Architecture Research

**Domain:** Email-to-booking automation pipeline (local, MCP-integrated, Claude intelligence layer)
**Researched:** 2026-02-25
**Confidence:** HIGH (based on existing architecture research, PROJECT.md discovery session, and established patterns for Microsoft Graph + MCP + Google Sheets integrations)

---

## Standard Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        LAYER 1: EMAIL SOURCE                         │
│                                                                      │
│   Microsoft Outlook (client emails, team forwards, email chains)     │
│                      ↓ Microsoft Graph API                           │
└────────────────────────────────────┬────────────────────────────────┘
                                     │
┌────────────────────────────────────▼────────────────────────────────┐
│                     LAYER 2: MONITORING / QUEUE                      │
│                                                                      │
│   Python Polling Script (Windows Task Scheduler, every ~15 min)      │
│   • Calls Graph API directly (no MCP needed here)                   │
│   • Detects new emails since last_checked timestamp                  │
│   • Writes raw email metadata → Inbox Queue tab in Google Sheets     │
│   • Lightweight: keyword filter, no NLP, no Claude calls             │
└────────────────────────────────────┬────────────────────────────────┘
                                     │ (queue persists between sessions)
┌────────────────────────────────────▼────────────────────────────────┐
│                     LAYER 3: MCP SERVERS                             │
│                                                                      │
│  ┌──────────────────────────┐   ┌──────────────────────────────┐    │
│  │  OUTLOOK MCP SERVER      │   │  GOOGLE SHEETS MCP SERVER    │    │
│  │  (Composio or ryaker)    │   │  (xing5/mcp-google-sheets)   │    │
│  │                          │   │                              │    │
│  │  list_emails             │   │  read_range                  │    │
│  │  read_email              │   │  write_range                 │    │
│  │  search_emails           │   │  format_cells                │    │
│  │  get_attachments         │   │  create_spreadsheet          │    │
│  └──────────────┬───────────┘   └────────────────┬─────────────┘   │
└─────────────────┼───────────────────────────────────┼───────────────┘
                  │                                   │
┌─────────────────▼───────────────────────────────────▼───────────────┐
│                   LAYER 4: CLAUDE INTELLIGENCE                        │
│                                                                      │
│  ┌─────────────────────┐   ┌──────────────────────────────────┐     │
│  │  EMAIL PARSER       │   │  BOOKING DETECTOR                │     │
│  │  • NLP field        │   │  • Is this a booking request?    │     │
│  │    extraction       │   │  • Classify: booking / follow-up │     │
│  │  • Partial data     │   │    / FYI / action-required       │     │
│  │    flagging         │   │  • Extract: client, brand,       │     │
│  │  • Missing field    │   │    holding co, shift type,       │     │
│  │    detection        │   │    rate, dates, stores, region   │     │
│  └──────────┬──────────┘   └────────────────┬─────────────────┘     │
│             │                               │                        │
│  ┌──────────▼──────────┐   ┌───────────────▼──────────────────┐     │
│  │  TO-DO GENERATOR    │   │  BOOKING WRITER                  │     │
│  │  • Per employee     │   │  • Writes to Bookings Tracker    │     │
│  │    (Daniel, Dinah,  │   │  • Status: Received / Pending    │     │
│  │    Viviene, Rae)    │   │  • Flags missing fields          │     │
│  │  • Prioritized by   │   │  • Links to source email         │     │
│  │    urgency          │   └────────────────┬─────────────────┘     │
│  └─────────────────────┘                    │                        │
│                                             │ (status = Confirmed    │
│                                             │  + all fields present) │
│                              ┌──────────────▼──────────────────┐     │
│                              │  AP AUTO-CREATOR                │     │
│                              │  • Reads confirmed booking row  │     │
│                              │  • Triggers dm-activation-      │     │
│                              │    planner skill                │     │
│                              │  • Applies client brand colours │     │
│                              │  • Creates Google Sheets AP     │     │
│                              │  • Writes AP link back to row   │     │
│                              └─────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────▼────────────────────────────────┐
│                     LAYER 5: STORAGE (Google Sheets)                  │
│                                                                      │
│  ┌───────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────┐  │
│  │  INBOX QUEUE  │  │  BOOKINGS    │  │  TO-DO       │  │  APs   │  │
│  │  (staging     │  │  TRACKER     │  │  SHEETS      │  │  (auto │  │
│  │  sheet, auto- │  │  (team-      │  │  (per        │  │  gen'd │  │
│  │  cleared)     │  │  facing)     │  │  employee)   │  │  tabs) │  │
│  └───────────────┘  └──────────────┘  └──────────────┘  └────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Component Responsibilities

| Component | Responsibility | Owner | Technology |
|-----------|----------------|-------|------------|
| Outlook | Email source; client bookings, team forwards arrive here | Microsoft | Hosted (cloud) |
| Python Polling Script | Continuous monitoring; detects new emails, queues them without NLP | Deterministic script | Python + Graph API |
| Windows Task Scheduler | Triggers polling script every ~15 min; runs without user interaction | OS-level | Windows built-in |
| Outlook MCP Server | Gives Claude read access to full email bodies and attachments | MCP layer | Composio or ryaker/outlook-mcp |
| Google Sheets MCP Server | Gives Claude read/write access to all tracker sheets | MCP layer | xing5/mcp-google-sheets |
| Claude Intelligence Layer | NLP parsing, field extraction, classification, to-do generation, AP triggers | AI | Claude Opus/Sonnet via Claude Code |
| Inbox Queue Sheet | Temporary staging area; email metadata before Claude processes it | Storage | Google Sheets tab |
| Bookings Tracker Sheet | Primary team-facing record of all bookings and their status | Storage | Google Sheets (main sheet) |
| To-Do Sheets | Per-employee action lists generated from email categorization | Storage | Google Sheets tabs |
| Activation Plans | Auto-generated campaign documents, one per confirmed booking | Output | Google Sheets (separate files) |

---

## Recommended Project Structure

```
dm-ops-hub/
├── .planning/                    # GSD planning files (research, roadmap)
├── scripts/
│   ├── polling/
│   │   ├── email_poller.py       # Main polling script (Graph API, queue writer)
│   │   ├── graph_client.py       # Microsoft Graph API wrapper (auth, requests)
│   │   ├── keyword_filter.py     # Lightweight pre-filter before Claude sees emails
│   │   └── scheduler_setup.md   # Instructions to register with Task Scheduler
│   └── utils/
│       ├── sheets_client.py      # Direct Sheets API wrapper (for polling script)
│       └── state.json            # last_checked timestamp, processed email IDs
├── prompts/
│   ├── email-parser.md           # Claude prompt: extract booking fields from email
│   ├── booking-detector.md       # Claude prompt: classify email type
│   ├── todo-generator.md         # Claude prompt: generate per-employee to-do lists
│   └── ap-trigger.md             # Claude prompt: trigger AP creation on confirmation
├── schemas/
│   ├── booking.json              # Booking data model (field names, types, required/optional)
│   └── sheet-columns.json        # Column definitions for Bookings Tracker sheet
├── config/
│   ├── mcp-config-example.json   # Example Claude MCP config (keys redacted)
│   └── brand-colours.json        # Client/holding company → hex colour mapping
└── docs/
    ├── SETUP.md                  # Full setup guide (Azure, Google Cloud, MCP servers)
    └── RUNBOOK.md                # Day-to-day operation guide for Daniel
```

### Structure Rationale

- **scripts/polling/:** The polling script is a standalone Python process that runs independently of Claude. Separating it from prompts makes the Claude-vs-script boundary explicit.
- **prompts/:** Plain-text prompt files that Claude uses during processing sessions. Version-controlled so prompt improvements are tracked in git.
- **schemas/:** JSON schemas enforce the data model contract between Claude's output and the Sheets columns. Claude checks extracted data against the booking schema to detect missing fields reliably.
- **config/:** Brand colour mapping lives here so AP generation doesn't hardcode client data in prompts. Easier to add new clients without editing prompt files.

---

## Architectural Patterns

### Pattern 1: Queue-Based Email Staging

**What:** The polling script writes email metadata to a Google Sheets "Inbox Queue" tab. Claude reads from this queue rather than polling Graph API directly during sessions.

**When to use:** Always — this is the core pattern for this system.

**Why it works:** Decouples monitoring (continuous, lightweight, no AI) from processing (on-demand or session-start, expensive AI calls). The queue survives Claude session ends/restarts. State is shared between Daniel's two machines via Google Sheets (no local file sync needed for queue state).

**Trade-offs:**
- Pro: No emails lost between sessions; queue acts as audit trail
- Pro: Polling script stays simple — no AI logic, just keyword matching and write
- Con: Slight latency (~15 min max) between email arrival and Claude processing
- Con: Queue sheet needs clearing after processing (handled by Claude's "clear processed items" step)

```python
# email_poller.py — deterministic, no AI
def poll_inbox():
    last_checked = load_last_checked()          # from state.json
    new_emails = graph_client.get_emails_since(last_checked)
    booking_candidates = keyword_filter(new_emails)
    sheets_client.append_to_queue(booking_candidates)
    save_last_checked(now())                    # prevent duplicate processing
```

### Pattern 2: Claude as Stateless Processor

**What:** Claude does not maintain state between processing runs. All state lives in Google Sheets. Claude reads what it needs, processes, writes results, and exits.

**When to use:** This system — local, session-based, multi-device.

**Why it works:** Google Sheets as the single source of truth means Daniel can access and edit the tracker from any device. Claude on either machine reads the same state. No local database to sync.

**Trade-offs:**
- Pro: Zero sync complexity between Daniel's two machines
- Pro: Team (Dinah, Viviene, Rae) can update the tracker without any special tools
- Con: Google Sheets API has rate limits (100 requests per 100 seconds per project — unlikely to be hit here)
- Con: Sheet manipulation is slower than a real database; acceptable for this volume

### Pattern 3: Partial Data Acceptance with Status Flags

**What:** Claude writes a booking row immediately when a booking email is detected, even with incomplete data. Missing fields are flagged explicitly. Status drives what can happen next.

**When to use:** Whenever incoming data is structurally unreliable (casual email formats, clients saying "I'll send the store list next week").

**Why it works:** Avoids losing the booking reference while waiting for complete data. Team can see the partial booking and follow up. Claude can re-process the email thread later when more info arrives.

**Status progression:**
```
Received    → Email detected, partial data extracted, team notified
Pending     → Claude flagged missing fields, team is chasing client
Confirmed   → All required fields present, AP generation can trigger
AP Created  → AP document generated and linked
Deployed    → Promoter ops chain initiated
```

**Required fields for "Confirmed":** client, shift type, base rate, dates (specific), store list (even if partial), province, shift count.

**Optional at booking time:** holding company (required for AP branding), extras, contact person.

### Pattern 4: Intelligence/Script Boundary — What Claude Does vs What Python Does

**What:** Hard separation between deterministic work (Python) and judgment-required work (Claude).

**The rule:** If the logic can be written as `if X then Y` without ambiguity, Python does it. If it requires reading natural language, inferring intent, handling format variation, or generating content, Claude does it.

| Task | Owner | Why |
|------|-------|-----|
| Check for new emails every 15 min | Python | Deterministic, scheduled, no judgment |
| Filter emails by keyword ("booking", "shifts", "campaign") | Python | Simple pattern match, no NLP |
| Write email ID + subject to queue | Python | Mechanical data transfer |
| Read email body and extract booking fields | Claude | Casual/varied language, requires NLP |
| Detect whether email is a booking vs FYI vs follow-up | Claude | Requires context, intent reading |
| Decide which fields are missing vs ambiguous | Claude | Requires understanding of booking model |
| Generate per-employee to-do items | Claude | Requires prioritization judgment |
| Apply client brand colours to AP headers | Claude + config | Colour lookup is deterministic; triggering AP is Claude |
| Increment booking ID | Python/Formula | Deterministic |
| Write to specific sheet cells | Claude via MCP | Claude decides what to write, MCP executes |

---

## Data Flow

### Flow 1: New Email → Queue

```
[Email arrives in Outlook]
    ↓
[Python Poller runs] (every 15 min via Task Scheduler)
    ↓ calls Microsoft Graph API
[GET /me/mailFolders/inbox/messages?$filter=receivedDateTime gt {last_checked}]
    ↓
[keyword_filter: does subject/snippet contain booking-related terms?]
    ├─ No  → skip (not written to queue)
    └─ Yes → write to Inbox Queue sheet:
             {email_id, subject, sender, received_at, snippet, status: "unprocessed"}
    ↓
[save last_checked timestamp to state.json]
```

### Flow 2: Queue → Bookings Tracker

```
[Claude session starts / on-demand trigger]
    ↓
[Claude reads Inbox Queue sheet via Sheets MCP]
    ↓ for each unprocessed row:
[Claude calls read_email via Outlook MCP to get full body]
    ↓
[Claude runs email-parser prompt: extract all booking fields]
    ├─ Booking detected?
    │     ↓ Yes
    │   [Claude writes new row to Bookings Tracker]
    │   [Status = "Received" if partial, "Pending" if missing critical fields]
    │   [Missing fields column populated: "Needs: store list, specific dates"]
    │     ↓ No
    │   [Categorize as: follow-up / FYI / action-required]
    └─ [Claude marks Inbox Queue row as "processed" and clears it]
```

### Flow 3: Bookings Tracker → AP Generation

```
[Claude checks Bookings Tracker for rows where Status = "Confirmed"]
    ↓
[Validate all required AP fields are present in that row]
    ↓
[Claude triggers dm-activation-planner skill]
    ↓ reads from booking row:
    [client, brand, holding company, store list, dates, shift type, rate, region]
    ↓
[AP Google Sheet created with client brand colours on headers]
    ↓
[Claude writes AP link back to booking row]
[Claude updates Status → "AP Created"]
```

### Flow 4: Team Updates → Claude Re-Processing

```
[Team member (Dinah/Viviene/Rae) manually updates booking row]
    e.g., adds store list, confirms specific dates, changes status
    ↓
[On next Claude session, Claude checks for rows where:]
    Status = "Pending" AND all required fields now present
    ↓
[Claude promotes Status → "Confirmed"]
[Triggers AP generation flow (Flow 3 above)]
```

### State Management

Google Sheets is the single source of truth. No local state files for booking data.

```
[Google Sheets]
    ↑ Python Poller writes email queue entries
    ↑ Claude reads email queue, writes booking rows
    ↑ Claude writes AP links, status updates
    ↑ Team reads and manually updates booking rows
    ↑ Formulas auto-calculate (Total Rate = Base + Extras)
```

The only local state: `state.json` in the polling script directory stores the `last_checked` timestamp. This is intentionally minimal — losing it only means re-scanning recent emails (duplicates filtered by email ID).

---

## Email Polling vs Webhooks: Decision

**Chosen: Polling (v1). Webhooks (v2 stretch goal).**

| Factor | Polling (v1) | Webhooks (v2) |
|--------|-------------|---------------|
| Setup complexity | Low — Python script + Task Scheduler | High — requires public HTTPS endpoint to receive push |
| Local machine compatibility | Full — runs entirely locally | Problematic — local machine can't receive Graph webhooks without ngrok or Azure relay |
| Latency | ~15 min max | Near-realtime (<5 sec) |
| Reliability | Self-healing — next poll catches anything missed | Single point of failure if webhook endpoint is down |
| Infrastructure cost | None | Requires cloud relay or tunneling service |
| Maintainability | Simple Python — team can understand and fix | Requires webhook subscription management, secret rotation |

**Why polling wins for this system:** The system runs locally on Daniel's machines. Microsoft Graph webhooks (change notifications) require a publicly accessible HTTPS endpoint to deliver push events. Running that locally requires either a cloud VM or a tunneling service (ngrok, Azure Relay) — both add infrastructure complexity that contradicts the "runs locally, no hosting costs" constraint. 15-minute latency is acceptable for this use case (email bookings are not real-time).

If latency ever becomes a problem, the upgrade path is: Python polling → Azure Function that receives Graph webhooks → writes to Sheets queue. Claude processing layer stays identical.

---

## MCP Server Integration Patterns

### Pattern: MCP Servers as Claude's API Clients

MCP servers are not middleware in the traditional sense. They are tool providers that Claude invokes during a session. Claude Code loads them at session start via `claude_desktop_config.json`.

```
[Claude Code session]
    ↓ on startup: loads MCP servers defined in config
    ↓ during processing:
    Claude calls: outlook_mcp.read_email(id="ABC123")
    Claude calls: sheets_mcp.write_range(sheet="Bookings", range="A5:R5", values=[...])
```

**Key constraint:** MCP tools are only available when Claude is actively running a session. The polling script cannot use MCP — it must call APIs directly.

**Implication for build order:** MCP servers must be installed and verified before Claude can process any emails. The polling script and Sheets API access can be tested independently.

### Pattern: Dual Authentication Paths

The polling script and the MCP servers authenticate separately to the same services:

```
Outlook:
  Polling script → Microsoft Graph API (Client Credentials or OAuth token stored in .env)
  Outlook MCP    → Microsoft Graph API (OAuth token managed by Composio or ryaker)

Google Sheets:
  Polling script → Sheets API (Service Account JSON key)
  Sheets MCP     → Sheets API (same Service Account JSON key)
```

Both paths can use the same Azure App Registration and same Google Cloud Service Account. This avoids managing separate credentials sets.

---

## Integration Points

### External Services

| Service | Integration Pattern | Auth | Notes |
|---------|---------------------|------|-------|
| Microsoft Graph API | REST (polling script direct) + MCP server (Claude sessions) | OAuth 2.0 (Outlook) | Same Azure App Registration for both; token refresh handled by MCP server |
| Google Sheets API | REST (polling script direct) + MCP server (Claude sessions) | Service Account JSON | Same service account for both; share each Sheet with service account email |
| dm-activation-planner | Claude skill invocation within same session | None (same process) | Skill already exists; needs Google Sheets pivot from Excel |
| Windows Task Scheduler | Calls `python email_poller.py` on schedule | OS-level (no auth) | Registers as Windows task; runs as current user |

### Internal Boundaries

| Boundary | Communication | Format | Notes |
|----------|---------------|--------|-------|
| Polling Script → Inbox Queue | Sheets API write | Row append: email_id, subject, sender, received_at, snippet | One row per email; polling script owns writes, Claude owns clears |
| Claude → Bookings Tracker | Sheets MCP write_range | Full row per booking | Claude owns initial write; team owns updates; both update Status |
| Claude → To-Do Sheets | Sheets MCP write_range | One tab per employee | Overwrite entire tab each session (not append) |
| Claude → AP Sheet | dm-activation-planner skill + Sheets MCP | New Google Sheet per booking | Claude writes AP Link back to booking row after creation |
| Bookings Tracker → Team | Direct Google Sheets UI | N/A | Team accesses via browser; no API needed for team reads |

---

## Scaling Considerations

This system is designed for a small team (5 users) with low email volume (estimated 10-50 booking emails per month). Scaling is not a concern for v1 or v2. The notes below are for awareness only.

| Scale | Architecture Adjustments |
|-------|--------------------------|
| Current (< 100 emails/month) | Polling + Google Sheets — no changes needed |
| Growth (100-1000 emails/month) | Add email deduplication to polling; consider dedicated Sheets for queue vs tracker |
| High volume (1000+/month) | Replace Sheets queue with a real database (SQLite local or Supabase); Sheets stays as team-facing UI only |

**First bottleneck:** Google Sheets API rate limits (100 requests per 100 seconds). At current projected volume, this won't be reached. If ever hit, add exponential backoff in the polling script.

**Second bottleneck:** Claude API costs. Each email processed requires a Claude API call. At $0.003-0.015 per email (Sonnet), 100 emails/month = ~$1.50/month. Not a concern.

---

## Anti-Patterns

### Anti-Pattern 1: Calling Claude from the Polling Script

**What people do:** Try to add NLP email classification directly into the polling Python script using Claude API calls.

**Why it's wrong:** Turns a lightweight background task into a slow, expensive, failure-prone process. If the Claude API is rate-limited or down, the poller breaks. Claude calls take seconds; a poller should complete in milliseconds. This also blurs the intelligence/script boundary unnecessarily.

**Do this instead:** Poller does keyword-only pre-filtering and queues emails. Claude processes the queue when available (session start or on-demand). Two separate processes with separate failure modes.

### Anti-Pattern 2: Using Claude's Memory as the State Store

**What people do:** Expect Claude to "remember" which emails it processed or what booking IDs it assigned across sessions.

**Why it's wrong:** Claude has no persistent memory between sessions. Every session starts fresh. If state exists only in Claude's context, it is lost when the session ends.

**Do this instead:** All state in Google Sheets. Processed email IDs in the Inbox Queue. Booking IDs as a sheet formula (auto-increment). Last-checked timestamp in state.json. Claude reads state at session start; writes state at session end.

### Anti-Pattern 3: Making the Bookings Tracker Claude-Only

**What people do:** Design the sheet so only Claude writes to it, treating team manual edits as exceptions or errors.

**Why it's wrong:** The explicit requirement is that both Claude and team members update the sheet. Dinah will add store lists, Viviene will correct booking details, Rae will update statuses. If Claude overwrites their changes on next run, the tool becomes adversarial.

**Do this instead:** Claude writes rows and updates specific columns (Status, AP Link, Email Source). Team has clear write ownership over other columns (Store List when "sent later", Assigned To, Notes). Claude checks Status before overwriting — if Status has advanced past what Claude set, Claude does not roll it back.

### Anti-Pattern 4: Blocking AP Generation on Perfect Data

**What people do:** Only trigger AP generation when every possible field is populated — including optional fields like contact person, extras breakdown, etc.

**Why it's wrong:** Clients routinely omit optional data. Waiting for perfection means APs are never generated. The AP needs: client, brand, store list, dates, shift type, rate, region. Everything else is nice-to-have.

**Do this instead:** Define a strict "confirmed fields" list (minimum required for AP). Mark optional fields clearly in schema. AP triggers when confirmed fields are present, regardless of optional field state.

### Anti-Pattern 5: Webhook Dependency for Local Deployment

**What people do:** Design the monitoring layer around Microsoft Graph webhooks (change notifications) because they are more "modern" and have lower latency.

**Why it's wrong:** Graph webhooks require a publicly accessible HTTPS endpoint. A local Windows machine behind a home/office router cannot receive inbound webhook calls without a cloud relay, ngrok tunnel, or Azure Function. This adds infrastructure cost and complexity that violates the "runs locally, no hosting" constraint.

**Do this instead:** Python polling on Task Scheduler. 15-minute latency is acceptable for booking emails. Upgrade to webhooks only if the system is ever migrated to a cloud host.

---

## Build Order and Dependencies

The component dependency graph drives phase sequencing:

```
1. Azure App Registration + Google Cloud Project
   (blocks everything — both are prerequisites)
        ↓
2. Google Sheets MCP Server (xing5/mcp-google-sheets)
   AND Outlook MCP Server (Composio or ryaker)
   (can be set up in parallel — both need their cloud projects first)
        ↓
3. Bookings Tracker Sheet structure
   (Claude needs this to exist before it can write bookings)
        ↓
4. Python Polling Script (email_poller.py + Graph API auth)
   (needs Azure App Registration; can be tested with Outlook MCP in parallel)
        ↓
5. Claude Email Processing (prompts + MCP integration)
   (needs Outlook MCP + Bookings Tracker sheet + polling queue working)
        ↓
6. AP Auto-Generation (dm-activation-planner pivot to Google Sheets)
   (needs confirmed bookings in tracker; needs brand-colours.json populated)
        ↓
7. To-Do Generation (lower risk, can build alongside step 5 or 6)
```

**Shared Google Cloud blocker:** Both this project and dm-activation-planner need a Google Cloud project. Setting it up in Phase 1 unblocks both simultaneously. This is the highest-priority first action.

---

## Sources

- PROJECT.md (discovery session findings — confidence HIGH, first-party)
- `.planning/research/architecture-overview.md` (prior architecture research — confidence HIGH)
- `.planning/research/outlook-mcp-options.md` (MCP server options — confidence HIGH)
- `.planning/research/google-sheets-mcp-options.md` (Sheets MCP options — confidence HIGH)
- Microsoft Graph API email polling patterns (training data — confidence MEDIUM; recommend verifying Graph subscription/webhook docs if latency requirements change)
- MCP server integration patterns (training data + existing research — confidence HIGH for this use case)

---

*Architecture research for: email-to-booking automation pipeline (dm-ops-hub)*
*Researched: 2026-02-25*

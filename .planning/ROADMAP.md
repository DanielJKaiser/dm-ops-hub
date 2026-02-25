# Roadmap: DM Ops Hub — Email-to-Booking Pipeline

## Overview

Four phases, each unblocked by the previous. Phase 1 establishes every credential and API connection the system depends on — nothing else is possible without it. Phase 2 builds the continuous monitoring backbone and locks in the data model before any write operations are coded. Phase 3 introduces Claude's intelligence layer in shadow mode, validating extraction accuracy against real DM emails before live writes are enabled. Phase 4 promotes to live operation and adds the AP auto-generation that closes the loop from email receipt to Activation Plan creation.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Infrastructure and Auth** - Establish every credential, MCP connection, and security baseline the system depends on
- [ ] **Phase 2: Polling Script and Data Model** - Continuous email monitoring, Inbox Queue pattern, and locked column ownership schema
- [ ] **Phase 3: Claude Processing in Shadow Mode** - Email classification, field extraction, to-do generation, and booking writes validated against real emails before going live
- [ ] **Phase 4: Live Pipeline and AP Generation** - Promote to live tracker writes and auto-generate Activation Plans on confirmed bookings

## Phase Details

### Phase 1: Infrastructure and Auth
**Goal**: Every API credential, MCP server, and security baseline is in place — Claude can read Outlook emails and read/write Google Sheets in a session
**Depends on**: Nothing (first phase)
**Requirements**: INFRA-01, INFRA-02, INFRA-03, INFRA-04, INFRA-05, INFRA-06
**Success Criteria** (what must be TRUE):
  1. Claude can call an Outlook MCP tool in a session and retrieve a real email body from the DM inbox
  2. Claude can call a Google Sheets MCP tool in a session and append a test row to the Bookings Tracker
  3. The Bookings Tracker Google Sheet exists with all required columns (ID through Completeness %) and correct data validation dropdowns for Status
  4. No secrets exist in the git repository — .env holds credentials, .gitignore excludes them, mcp-config-example.json is committed with placeholders
**Plans**: TBD

Plans:
- [ ] 01-01: Azure App Registration and Google Cloud project setup
- [ ] 01-02: MCP server installation and authentication verification
- [ ] 01-03: Bookings Tracker schema creation and security baseline

### Phase 2: Polling Script and Data Model
**Goal**: Emails are captured continuously via a background poller and written to the Inbox Queue — Claude has a queue to drain when a session starts, and the data model is locked before any booking writes occur
**Depends on**: Phase 1
**Requirements**: MON-01, MON-02, MON-03, MON-04, MON-05, MON-06, MON-07, BOOK-02, BOOK-03, BOOK-04, BOOK-05, BOOK-06, BOOK-07
**Success Criteria** (what must be TRUE):
  1. The polling script runs every 15 minutes via Windows Task Scheduler and new emails appear as rows in the Inbox Queue sheet within one polling cycle
  2. If the poller's OAuth token fails, an AUTH_FAILURE sentinel row appears in the Inbox Queue — the team can see the pipeline is broken without checking logs
  3. The Bookings Tracker has a functioning Completeness % column that reflects how many required fields are present per row
  4. Claude-owned columns and team-owned columns are defined and documented — Claude's write code cannot overwrite team-owned fields
  5. Reprocessing the same email ID twice does not create a duplicate row in the Bookings Tracker
**Plans**: TBD

Plans:
- [ ] 02-01: MSAL authentication, Graph API email fetch, and state.json timestamp management
- [ ] 02-02: Keyword pre-filter, Inbox Queue writes, AUTH_FAILURE sentinel, and forwarded email pre-processor
- [ ] 02-03: Windows Task Scheduler registration and end-to-end poller verification
- [ ] 02-04: Data model schema — three-state fields, column ownership, status enum, completeness formula

### Phase 3: Claude Processing in Shadow Mode
**Goal**: Claude classifies, extracts, and writes to a staging sheet — team reviews every row before it reaches the live tracker, building confidence in accuracy before live writes are enabled
**Depends on**: Phase 2
**Requirements**: EMAIL-01, EMAIL-02, EMAIL-03, EMAIL-04, EMAIL-05, EMAIL-06, BOOK-01, TODO-01, TODO-02, TODO-03
**Success Criteria** (what must be TRUE):
  1. Claude processes the Inbox Queue on session start and classifies each email as new booking request, follow-up to existing booking, or non-booking — low-confidence emails route to the Review Queue tab, not the tracker
  2. For a booking email like "we need 20 dry shifts in GP next month," Claude writes a staging row with extracted fields, flags missing fields using the three-state model (extracted / pending / unsure), and does not write directly to the live Bookings Tracker
  3. After accuracy validation against 50+ real DM emails with Client >95%, Shift Type >90%, Rate >85%, Dates >80% per-field accuracy, Claude writes directly to the live Bookings Tracker with BOOK-01 live
  4. Claude generates per-employee to-do lists (Daniel: strategic items, Dinah: booking ops, Viviene/Rae: AP creation) in their respective Google Sheets tabs from morning email reads
  5. Forwarded emails from Sheilagh or Dinah are correctly attributed to the original client sender, not the forwarder
**Plans**: TBD

Plans:
- [ ] 03-01: Email classification prompt and Review Queue routing
- [ ] 03-02: Field extraction prompt with three-state model and forwarded email handling
- [ ] 03-03: Shadow mode staging sheet, classification audit tab, and accuracy tracking
- [ ] 03-04: To-do list generation prompt and per-employee Sheets tab writes
- [ ] 03-05: Accuracy validation against real DM emails and live mode promotion

### Phase 4: Live Pipeline and AP Generation
**Goal**: The full pipeline runs live — booking emails are extracted to the tracker automatically, and confirmed bookings with complete data auto-generate Activation Plans with correct client brand colours
**Depends on**: Phase 3
**Requirements**: AP-01, AP-02, AP-03, AP-04, AP-05, AP-06
**Success Criteria** (what must be TRUE):
  1. When a booking row reaches "Confirmed (Ready for AP)" status in the tracker, Claude auto-generates an Activation Plan as a Google Sheets document — not Excel — within one session
  2. The AP headers use the correct client brand colours from the brand-colours lookup table; multi-brand bookings use the holding company colour (Astral, RFG, or RCL)
  3. The generated AP link is written back to the Bookings Tracker AP Link column and the booking status advances to "AP Created" automatically
  4. AP generation does not trigger for "Confirmed (Pending Data)" status — only "Confirmed (Ready for AP)" with all required fields validated passes the gate
**Plans**: TBD

Plans:
- [ ] 04-01: AP trigger validation logic, field completeness gate, and status state separation
- [ ] 04-02: dm-activation-planner Google Sheets pivot and brand-colours lookup table
- [ ] 04-03: AP link write-back, status advancement, and runbook documentation

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Infrastructure and Auth | 0/3 | Not started | - |
| 2. Polling Script and Data Model | 0/4 | Not started | - |
| 3. Claude Processing in Shadow Mode | 0/5 | Not started | - |
| 4. Live Pipeline and AP Generation | 0/3 | Not started | - |

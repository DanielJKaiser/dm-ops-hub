# Requirements: DM Ops Hub

**Defined:** 2026-02-25
**Core Value:** Claude reads casual Outlook booking emails, extracts structured data, and writes to a Google Sheets tracker the team uses daily — replacing email-and-memory with a single operational surface.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Infrastructure

- [ ] **INFRA-01**: Azure App Registration configured with Mail.Read application permission (polling) and delegated permissions (Outlook MCP)
- [ ] **INFRA-02**: Google Cloud project created with Sheets API enabled and service account JSON key generated
- [ ] **INFRA-03**: Outlook MCP server (ryaker/outlook-mcp) installed and authenticated — Claude can read emails in a session
- [ ] **INFRA-04**: Google Sheets MCP server (xing5/mcp-google-sheets) installed and authenticated — Claude can read/write Sheets
- [ ] **INFRA-05**: Bookings Tracker Google Sheet created with full column schema (ID, Date Received, Client, Brand, Holding Company, Shift Type, Base Rate, Extras, Total Rate, Provinces, Store List, Dates, Shift Count, Status, Assigned To, AP Link, Email Source, Notes, Last Updated, Completeness %)
- [ ] **INFRA-06**: Security baseline established — .env for secrets, .gitignore excludes credentials, mcp-config-example.json committed

### Email Monitoring

- [ ] **MON-01**: Python polling script fetches new emails from Outlook via Microsoft Graph API every 15 minutes
- [ ] **MON-02**: Polling script authenticates via MSAL Client Credentials flow with automatic token refresh
- [ ] **MON-03**: Polling script applies keyword pre-filter to flag booking-related emails
- [ ] **MON-04**: Flagged email metadata written to Inbox Queue sheet in Google Sheets
- [ ] **MON-05**: Polling script tracks last-checked timestamp in state.json to avoid reprocessing
- [ ] **MON-06**: Polling script writes AUTH_FAILURE sentinel row to Inbox Queue on any 401/403 response
- [ ] **MON-07**: Windows Task Scheduler configured to run polling script every 15 minutes

### Email Processing

- [ ] **EMAIL-01**: Claude reads Inbox Queue and processes flagged emails on session start or on-demand
- [ ] **EMAIL-02**: Claude classifies each email as: new booking request, follow-up to existing booking, or non-booking
- [ ] **EMAIL-03**: Claude extracts booking fields from casual/unstructured email text (client, brand, holding company, shift type, base rate, extras, provinces, stores, dates, shift count, contact person)
- [ ] **EMAIL-04**: Claude detects missing fields using three-state model: extracted / pending (client hasn't provided) / unsure (Claude uncertain — verify)
- [ ] **EMAIL-05**: Claude handles forwarded email chains by parsing the embedded original content, not the forwarding wrapper
- [ ] **EMAIL-06**: Low-confidence classifications route to Review Queue tab, not directly to Bookings Tracker

### Bookings Tracker

- [ ] **BOOK-01**: Claude writes new booking row to Bookings Tracker with all extracted fields and status "Received" or "Pending"
- [ ] **BOOK-02**: Idempotent processing — system checks processed email IDs before writing to prevent duplicate rows
- [ ] **BOOK-03**: Booking status workflow enforced: Received → Details Pending → Confirmed (Pending Data) → Confirmed (Ready for AP) → AP Created → Deployed
- [ ] **BOOK-04**: Column ownership enforced — Claude-owned columns (extraction data) vs team-owned columns (Store List updates, Assigned To, Notes) with no cross-writes
- [ ] **BOOK-05**: Audit trail — every booking row includes email source link (Outlook email ID/URL) that is immutable once set
- [ ] **BOOK-06**: Completeness % column calculates how many required fields are present per row
- [ ] **BOOK-07**: Team members can manually update team-owned columns (Store List, Confirmed Dates, Assigned To, Notes, Status) without risk of Claude overwrite

### To-Do Generation

- [ ] **TODO-01**: Claude reads all morning emails and categorizes by intent: booking request, follow-up needed, FYI/info only, action required
- [ ] **TODO-02**: Claude generates per-employee to-do lists: Daniel (strategic items, client comms), Dinah (booking ops, promoter deployment), Viviene (AP creation), Rae (AP creation)
- [ ] **TODO-03**: To-do lists written to per-employee tabs in Google Sheets

### Activation Plans

- [ ] **AP-01**: Claude triggers AP auto-generation when booking status reaches "Confirmed (Ready for AP)" and all required fields validated
- [ ] **AP-02**: AP generated as Google Sheets document (not Excel) via dm-activation-planner skill
- [ ] **AP-03**: AP headers use client brand colours from brand-colours lookup table
- [ ] **AP-04**: Multi-brand bookings use holding company branding (Astral, RFG, RCL) on AP headers
- [ ] **AP-05**: Generated AP link written back to Bookings Tracker AP Link column
- [ ] **AP-06**: Booking status auto-advances to "AP Created" after successful AP generation

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Enhanced Intelligence

- **ENH-01**: Cross-chain deduplication — detect when same booking arrives via Daniel direct email AND Dinah forward, using heuristic subject+client+date matching
- **ENH-02**: Smart role-based routing for to-do lists — context-aware assignment beyond static role rules
- **ENH-03**: Rate validation via dm-quoter skill — flag when extracted rate deviates from standard rate cards

### Extended Integrations

- **INT-01**: Claude for Sheets add-on installed on Bookings Tracker for team-facing Claude intelligence (data validation, flagging)
- **INT-02**: Microsoft Graph webhooks for event-driven email processing (replaces polling)
- **INT-03**: dm-promoter-ops integration — auto-trigger briefing docs and training after AP Created
- **INT-04**: dm-admin integration — invoicing chain from confirmed/completed bookings

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Sending emails on behalf of users | Risks brand damage if tone is wrong; legal/consent issues; team prefers to send as themselves |
| Auto-confirming bookings from email text | "Confirmed" in email is ambiguous; client confirmation ≠ operational readiness; false positives trigger bad APs |
| Natural language chat interface for tracker | Google Sheets IS the interface; team already knows it; adds unnecessary UI complexity |
| Client-facing booking portal | Clients prefer email; adds public web surface, onboarding, and form design burden |
| WhatsApp monitoring | Requires Business API, monthly cost, dedicated phone number; no MCP server exists |
| Mobile app for team | Google Sheets already works on mobile; separate app fragments the source of truth |
| AI-drafted email replies | Creates unreviewed content that can be accidentally sent; context drift risk |
| Calendar integration | Not needed for v1; bookings are date-tracked in Sheets already |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFRA-01 | — | Pending |
| INFRA-02 | — | Pending |
| INFRA-03 | — | Pending |
| INFRA-04 | — | Pending |
| INFRA-05 | — | Pending |
| INFRA-06 | — | Pending |
| MON-01 | — | Pending |
| MON-02 | — | Pending |
| MON-03 | — | Pending |
| MON-04 | — | Pending |
| MON-05 | — | Pending |
| MON-06 | — | Pending |
| MON-07 | — | Pending |
| EMAIL-01 | — | Pending |
| EMAIL-02 | — | Pending |
| EMAIL-03 | — | Pending |
| EMAIL-04 | — | Pending |
| EMAIL-05 | — | Pending |
| EMAIL-06 | — | Pending |
| BOOK-01 | — | Pending |
| BOOK-02 | — | Pending |
| BOOK-03 | — | Pending |
| BOOK-04 | — | Pending |
| BOOK-05 | — | Pending |
| BOOK-06 | — | Pending |
| BOOK-07 | — | Pending |
| TODO-01 | — | Pending |
| TODO-02 | — | Pending |
| TODO-03 | — | Pending |
| AP-01 | — | Pending |
| AP-02 | — | Pending |
| AP-03 | — | Pending |
| AP-04 | — | Pending |
| AP-05 | — | Pending |
| AP-06 | — | Pending |

**Coverage:**
- v1 requirements: 35 total
- Mapped to phases: 0
- Unmapped: 35 (awaiting roadmap creation)

---
*Requirements defined: 2026-02-25*
*Last updated: 2026-02-25 after initial definition*

# Phase 1: Infrastructure and Auth - Context

**Gathered:** 2026-02-25
**Status:** Ready for planning

<domain>
## Phase Boundary

Establish every API credential, MCP server connection, and security baseline the system depends on. After this phase, Claude can read Outlook emails and read/write Google Sheets in a session. The Bookings Tracker spreadsheet exists with full schema and formatting.

**Critical logistics:** Daniel is building on a desktop WITHOUT Outlook installed. Phase 1 work is split:
- **Desktop (current machine):** Google Cloud project, Azure App Registration (browser-based), Google Sheets MCP, Bookings Tracker creation, schema, security baseline
- **Work laptop (after push to GitHub):** Outlook MCP installation, authentication verification, end-to-end email reading test

The repo is already on GitHub (github.com/DanielJKaiser/dm-ops-hub). Push after desktop work is complete.

</domain>

<decisions>
## Implementation Decisions

### Outlook MCP Choice
- Claude's discretion on Composio vs ryaker/outlook-mcp — pick what's fastest to unblock Phase 2
- If starting with Composio, document the migration path to ryaker explicitly
- Azure App Registration walkthrough: step-by-step guidance with links — Daniel is not familiar with Azure portal
- Permissions: Mail.Read only for v1 (read-only). Register Mail.ReadWrite in the app but don't grant it yet — future expansion path
- Multiple inboxes eventually (Daniel, Dinah, Sheilagh) but start with Daniel's inbox only for testing
- Team laptops get the setup only once everything is 100% working and easily pullable from git

### Google Sheet Structure
- Separate spreadsheets, NOT one mega-spreadsheet with everything as tabs
- Main operational spreadsheet has 4 tabs:
  1. **Bookings Tracker** — the team's daily operational surface
  2. **Inbox Queue** — where the polling script writes raw flagged emails
  3. **Review Queue** — low-confidence classifications land here for human review
  4. **Config/Lookups** — brand colours, client list, status enum reference data
- APs are separate Google Spreadsheets, split by brand AND campaign
  - Multiple bookings that are actually the same campaign can be merged
  - Claude flags potential merges when separate bookings look like they belong together
  - This is a Phase 4 implementation concern but the tracker schema should accommodate AP links per booking
- To-do lists: separate spreadsheet or tabs added in Phase 3 (not Phase 1)
- Day one access: Daniel + Google service account only. Team added after system is proven.

### Credential Management
- .env file in repo root for all secrets (Azure Client ID, Client Secret, Google project details)
- .gitignore excludes .env, service account key, and actual MCP config files
- .env.example committed with placeholder values + setup documentation explaining what each value is and where to get it
- Claude's discretion: service account JSON key storage location (in-repo gitignored vs external path)
- Claude's discretion: MCP config approach (example file vs setup script)

### Bookings Tracker Details
- **Status flow (6 states):** Received → Details Pending → Details Confirmed → Ready for AP → AP Created → Deployed for Human Review
- **Column order (status-first):** Status → Client → Brand → Holding Company → Shift Type → Base Rate → Extras → Total Rate → Provinces → Store List → Dates → Shift Count → Assigned To → AP Link → Email Source → Notes → Completeness % → Date Received → Last Updated → ID
- **Colour-coded statuses from day one:**
  - Received = (colour TBD)
  - Details Pending = (colour TBD)
  - Details Confirmed = (colour TBD)
  - Ready for AP = (colour TBD)
  - AP Created = (colour TBD)
  - Deployed for Human Review = (colour TBD)
- **Frozen columns: minimal or none.** Previous experience with dm-activation-planner had frozen cells taking up ~40% of usable screen. Keep header row frozen only. Do NOT freeze columns. If the team wants column freezing later, they can set it themselves.
- Data validation dropdowns on Status column (the 6 states above)

### Claude's Discretion
- Composio vs ryaker decision for initial MCP server
- Service account JSON key storage location
- MCP config generation approach (example file vs script)
- Exact status colour palette for conditional formatting
- Column width and general sheet formatting details
- Whether to include Completeness % formula in Phase 1 or defer to Phase 2

</decisions>

<specifics>
## Specific Ideas

- Daniel wants step-by-step Azure portal walkthrough — not just "configure these permissions"
- The setup must be easily replicable: `.env.example` + docs so cloning on work laptop or team laptops is straightforward
- APs are split by brand AND campaign — this is how the team thinks about their work (not by booking ID)
- The tracker should feel like a clean operational tool, not a data dump — status colours help the team scan quickly
- Internal team usage only for now — no need to over-engineer access controls or sharing

</specifics>

<deferred>
## Deferred Ideas

- AP merge detection (flagging multiple bookings that are actually the same campaign) — Phase 4
- Team laptop deployment and multi-inbox setup — after v1 is proven
- To-do list tabs — Phase 3
- Mail.ReadWrite permissions activation — future phase when email flagging is needed

</deferred>

---

*Phase: 01-infrastructure-and-auth*
*Context gathered: 2026-02-25*

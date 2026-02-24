# DM Ops Hub — Architecture Overview

> Researched: 2026-02-24

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     MICROSOFT OUTLOOK                       │
│              (Client emails, team forwards)                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    Microsoft Graph API
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  OUTLOOK MCP SERVER                          │
│            (Composio or ryaker/outlook-mcp)                  │
│                                                              │
│  Tools: list_emails, search_emails, read_email,              │
│         get_attachments, list_folders                         │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                                                              │
│                    CLAUDE (Orchestrator)                      │
│                                                              │
│  ┌─────────────────┐  ┌──────────────────┐                  │
│  │ Email Parser     │  │ Booking Detector │                  │
│  │ • NLP extraction │  │ • Keyword match  │                  │
│  │ • Field mapping  │  │ • Pattern recog  │                  │
│  │ • Missing field  │  │ • Rate lookup    │                  │
│  │   detection      │  │ • Client match   │                  │
│  └────────┬────────┘  └────────┬─────────┘                  │
│           │                     │                             │
│  ┌────────▼────────┐  ┌────────▼─────────┐                  │
│  │ To-Do Generator │  │ Booking Writer   │                  │
│  │ • Per-employee  │  │ • Sheets API     │                  │
│  │ • Categorized   │  │ • Status track   │                  │
│  │ • Prioritized   │  │ • Brand detect   │                  │
│  └─────────────────┘  └────────┬─────────┘                  │
│                                │                             │
│                       ┌────────▼─────────┐                  │
│                       │ AP Auto-Creator  │                  │
│                       │ • Triggers on    │                  │
│                       │   "Confirmed"    │                  │
│                       │ • Brand colours  │                  │
│                       │ • Google Sheets  │                  │
│                       │   native output  │                  │
│                       └──────────────────┘                  │
│                                                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    Google Sheets API
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  GOOGLE SHEETS MCP SERVER                     │
│               (xing5/mcp-google-sheets)                      │
│                                                              │
│  Tools: create_spreadsheet, read_range, write_range,         │
│         format_cells, create_sheet                            │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
    ┌──────────────┐ ┌──────────┐ ┌──────────────┐
    │  BOOKINGS    │ │   APs    │ │  TO-DO       │
    │  TRACKER     │ │ (auto-   │ │  SHEETS      │
    │  (team-      │ │ generated│ │  (per-       │
    │   facing)    │ │ from     │ │  employee)   │
    │              │ │ bookings)│ │              │
    └──────────────┘ └──────────┘ └──────────────┘
```

## Monitoring Strategy

### V1: Session-Based + Polling Script

```
┌──────────────────────────────────────────────┐
│  WINDOWS TASK SCHEDULER                       │
│  (runs every 15 minutes)                      │
│                                               │
│  → Python script calls Graph API              │
│  → Checks for new emails since last check     │
│  → Flags booking-related emails               │
│  → Writes raw email data to "Inbox Queue"     │
│    sheet in Google Sheets                      │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  CLAUDE (on session start or on-demand)       │
│                                               │
│  → Reads "Inbox Queue" sheet                  │
│  → Processes flagged emails with NLP          │
│  → Extracts booking fields                    │
│  → Writes to Bookings Tracker                 │
│  → Generates to-do lists                      │
│  → Clears processed items from queue          │
└──────────────────────────────────────────────┘
```

### V2: Continuous (Future)
- Microsoft Graph webhooks (push notifications on new email)
- Triggers Claude Agent SDK workflow automatically
- No polling needed — event-driven

## Authentication Flow

### Outlook (Microsoft Graph)
1. Register Azure App (free)
2. Configure permissions: Mail.Read, Mail.ReadWrite (optional)
3. Generate Client Secret
4. OAuth 2.0 authorization code flow → access token + refresh token
5. MCP server handles token refresh automatically

### Google Sheets
1. Create Google Cloud Project (free tier)
2. Enable Sheets API
3. Create Service Account → download JSON key
4. Share target Google Sheets with service account email
5. MCP server authenticates with service account key (no user interaction)

## Data Flow: Email → Booking → AP

```
1. Email arrives in Outlook
   └─→ Polling script detects new email
       └─→ Flags as "booking-related" based on keywords
           └─→ Writes to Inbox Queue sheet

2. Claude processes Inbox Queue
   └─→ Reads email body via Outlook MCP
       └─→ Extracts: client, brand, shift type, dates, stores, rate
           └─→ Detects missing fields
               └─→ Writes to Bookings Tracker with status "Received" or "Pending"

3. Team reviews Bookings Tracker
   └─→ Adds missing info (store list, confirmed dates)
       └─→ Updates status to "Confirmed"

4. Claude detects "Confirmed" status with all required fields
   └─→ Triggers dm-activation-planner skill
       └─→ Creates AP Google Sheet with:
           - Client brand colours on headers
           - Holding company branding for multi-brand
           - Store list from bookings tracker
           - Shift schedule from confirmed dates
           - Rate from bookings tracker
       └─→ Links AP back to bookings tracker (AP Link column)
           └─→ Updates booking status to "AP Created"

5. Promoter Ops chain triggered
   └─→ dm-promoter-ops: briefing docs, training, sales capture
       └─→ dm-quoter: validates rates match rate card
           └─→ dm-admin: invoicing pipeline ready
```

## Technology Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| Email Access | Microsoft Graph API via Outlook MCP | Industry standard, full email access |
| Email Monitoring | Python script + Windows Task Scheduler | Simple, reliable, no infrastructure |
| Intelligence Layer | Claude (Opus/Sonnet) | NLP parsing of casual emails, booking detection |
| Bookings Tracker | Google Sheets via Sheets MCP | Team-accessible, real-time collaboration, familiar UI |
| AP Generation | dm-activation-planner skill (Google Sheets native) | Existing skill, pivoted from Excel to Sheets |
| Hosting | Local (Daniel's machines) | No cloud hosting costs, runs alongside Claude Code |
| Sync | GitHub (dm-ops-hub repo) | Cross-device, version controlled |

## Shared Dependency: Google Cloud

Both this project AND the Activation Planner System need a Google Cloud project:
- **This project:** Sheets API (bookings tracker, to-do sheets)
- **AP System:** Maps JavaScript API + Sheets API (mapping + AP generation)

Setting up Google Cloud unblocks BOTH projects simultaneously. This should be Phase 1.

# DM Ops Hub — Email-to-Booking Pipeline

> **Project:** dm-ops-hub
> **Owner:** Daniel Kaiser (Diversified Marketing)
> **Created:** 2026-02-24
> **Repo:** github.com/DanielJKaiser/dm-ops-hub

---

## What This Is

An operational automation hub for Diversified Marketing that starts with an **email intelligence layer** — Claude reads Outlook emails, extracts actionable items, auto-detects booking requests, writes them to a Google Sheets booking tracker, and ultimately auto-generates Activation Plans when bookings are confirmed.

This is a **team-facing tool** used by Daniel, Dinah Tsholo, Viviene, and Rae — not just a personal dashboard.

---

## The Problem

Today, booking requests arrive via email in varied, casual formats — sometimes a structured brief, sometimes a one-liner like "we need 20 dry shifts in GP next month." These emails arrive to Daniel directly, or through Dinah/Sheilagh who forward them. There is:

- **No single view** of all incoming booking requests and their status
- **No structured intake** — information is fragmented across emails, people, and devices
- **No state tracking** — requests get lost between "received" and "confirmed"
- **Manual AP creation** — once a booking is confirmed, someone (Dinah/Viviene/Rae) manually builds the Activation Plan
- **No audit trail** — hard to see what was requested vs what was booked vs what was executed

---

## What "Done" Looks Like

**End-to-end automation:** A client emails a booking request → Claude detects it, extracts details, writes a row to the bookings Google Sheet → when the booking reaches "confirmed" with all required details (store list, dates, shift type, rates) → Claude auto-generates an Activation Plan with client brand colours on the headers.

Additionally:
- Claude summarizes all morning emails into actionable **to-do lists per employee** (Daniel, Dinah, Viviene, Rae)
- The team uses the Google Sheet as their **primary daily booking tracker**
- Both Claude and team members can update the sheet (Claude auto-updates from emails, team manually updates what Claude can't see)

---

## Key Context from Discovery

### Email Characteristics
- **Format:** Casual and varied. No standard template across clients. Could be a one-liner, a brief with attachments, or a forwarded email chain.
- **Missing information is common:** Clients often don't provide specific dates (say "next month"), don't specify store lists (say "GP"), don't clarify shift type (dry vs wet), or promise to "share the store list the week before."
- **Flow:** Mixed — some clients email Daniel directly, some go through Dinah or Sheilagh who forward relevant requests.

### Current Process
- **Dinah handles 99% of instore shift bookings** operationally
- **Viviene, Dinah, and Rae collaborate on making APs**
- There is **no bookings tracker** — this will be built from scratch
- Booking state is tracked informally (email flags, memory, conversation threads)

### Booking Data Model (What Claude Needs to Extract)
From a booking email, Claude should attempt to extract:

| Field | Example | Often Missing? |
|-------|---------|----------------|
| Client | Sunbake | Rarely |
| Brand | (may be same as client) | Sometimes |
| Holding Company | RCL, RFG, Astral | For AP branding |
| Shift Type | Dry / Wet / Beverage | Sometimes |
| Base Rate | R550/shift | Sometimes |
| Extras | Giveaways (+R100/shift transport) | Often |
| Provinces / Regions | GP, KZN, WC | Usually provided |
| Specific Stores | Makro Woodmead, Spar Hillcrest | Often missing — "will send later" |
| Dates | 1-15 March | Often vague — "next month" |
| Shift Count | 50 shifts | Sometimes |
| Contact Person | (client-side contact) | Sometimes |

### Booking Status Flow
```
Email Received
    → Request Logged (partial info OK)
        → Details Pending (waiting for store list, dates, etc.)
            → Confirmed (all info received)
                → AP Created (auto-generated)
                    → Deployed (promoter ops chain triggered)
```

### AP Branding Rule
- Cell header colourings match the **logo of the company** in the brief
- If multi-brand, use the **holding company** branding (Astral, RFG, RCL)

### Monitoring Approach
- **Goal:** Continuous/scheduled monitoring (not just on-demand)
- **Realistic v1:** A lightweight polling script checks Outlook every X minutes via Microsoft Graph API, Claude processes flagged emails when detected
- **Fallback:** Session-based scanning — Claude reads inbox on session start and on-demand

### Employee To-Do Lists
Beyond booking detection, Claude should:
- Read all morning emails
- Categorize by: booking request, follow-up needed, FYI/info only, action required
- Generate per-employee to-do lists (Daniel, Dinah, Viviene, Rae)

---

## Technical Architecture

```
┌──────────────────────────────────────────────────────┐
│                    OUTLOOK                           │
│         (Microsoft Graph API via MCP)                │
└──────────────────┬───────────────────────────────────┘
                   │ reads emails
                   ▼
┌──────────────────────────────────────────────────────┐
│              CLAUDE (Intelligence Layer)              │
│  • Parses casual/varied email formats                │
│  • Extracts booking fields                           │
│  • Detects missing information                       │
│  • Categorizes: booking / follow-up / FYI / action   │
│  • Generates per-employee to-do lists                │
└────────┬─────────────────────────┬───────────────────┘
         │                         │
         ▼                         ▼
┌─────────────────────┐  ┌─────────────────────────────┐
│  GOOGLE SHEETS      │  │  TO-DO LISTS                │
│  (Bookings Tracker) │  │  (per employee)             │
│  • Team-facing      │  │  • Daniel: strategic items  │
│  • Status tracking  │  │  • Dinah: booking ops       │
│  • Both Claude +    │  │  • Viviene: AP creation     │
│    team update it   │  │  • Rae: AP creation         │
└────────┬────────────┘  └─────────────────────────────┘
         │ (status = Confirmed)
         ▼
┌──────────────────────────────────────────────────────┐
│           AUTO AP GENERATION                         │
│  • dm-activation-planner skill                       │
│  • Client brand colours on headers                   │
│  • Holding company branding for multi-brand          │
│  • Google Sheets native (not Excel)                  │
└──────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────┐
│           PROMOTER OPS CHAIN                         │
│  • dm-promoter-ops (briefing, training, comms)       │
│  • dm-quoter (rate validation)                       │
│  • dm-admin (invoicing from confirmed bookings)      │
└──────────────────────────────────────────────────────┘
```

---

## Dependencies & Prerequisites

| Dependency | Status | Blocks |
|------------|--------|--------|
| Azure App Registration (free) | Not started | Outlook MCP access |
| Google Cloud Project | Not started (also needed for AP system) | Google Sheets API |
| Outlook MCP Server | Not installed | Email reading |
| Google Sheets MCP Server | Not installed | Bookings tracker |
| dm-activation-planner skill | Exists (needs Google Sheets pivot) | Auto AP generation |

---

## Existing DM Skills That Connect

| Skill | How It Connects |
|-------|----------------|
| **dm-activation-planner** | Auto-generates APs from confirmed bookings |
| **dm-promoter-ops** | Triggered after AP: briefing docs, training, sales capture, WhatsApp comms |
| **dm-quoter** | Validates rates extracted from emails against rate cards |
| **dm-admin** | Invoicing chain starts from confirmed/completed bookings |
| **dm-campaign-reporter** | Reads AP completion data for feedback reports |
| **dm-roadmap** | Tracks this project's progress across sessions/devices |

---

## Team

| Person | Role in This System |
|--------|-------------------|
| **Daniel** | Overview, strategic decisions, client comms |
| **Dinah Tsholo** | Primary booking handler, promoter deployment, AP creation |
| **Viviene** | AP creation, operational support |
| **Rae** | AP creation, operational support |
| **Sheilagh** | Forwards client requests, invoicing |
| **Claude** | Email parsing, booking extraction, to-do generation, AP auto-creation |

---

## Scope Boundaries

**In scope:**
- Outlook email reading and parsing via MCP
- Booking request detection and field extraction
- Google Sheets bookings tracker (new, team-facing)
- Booking status workflow (received → pending → confirmed → AP created)
- Per-employee to-do list generation from emails
- Auto AP generation on booking confirmation
- Client brand colour application on AP headers

**Out of scope (for now):**
- Sending emails on behalf of users (read-only for v1)
- Calendar integration
- WhatsApp monitoring (would require Business API)
- Client-facing portal
- Mobile app for team

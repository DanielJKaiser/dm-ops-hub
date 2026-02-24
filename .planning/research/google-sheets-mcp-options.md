# Google Sheets MCP Integration Options

> Researched: 2026-02-24

## Overview

Google Sheets API requires a Google Cloud project with Sheets API enabled. Auth is typically via service account (server-to-server) or OAuth 2.0 (user-level).

**Prerequisite for all:** Google Cloud project (free tier) with Sheets API enabled.

**Note:** This same Google Cloud project is needed for the Activation Planner system (Maps JavaScript API, Sheets API). Setting it up unblocks both projects.

---

## Option 1: xing5/mcp-google-sheets (Recommended)

- **URL:** https://github.com/xing5/mcp-google-sheets
- **Type:** Python-based MCP server
- **Auth:** Service account (JSON key file)
- **Features:** Create/read/update/delete spreadsheets, cell manipulation, formatting, formula support
- **Pros:** Production-ready, service account auth (no user interaction needed for polling), well-maintained
- **Cons:** Python dependency

## Option 2: ringo380/claude-google-sheets-mcp

- **URL:** https://github.com/ringo380/claude-google-sheets-mcp
- **Type:** Comprehensive MCP server optimized for Claude CLI
- **Features:** Spreadsheet discovery, data manipulation, formatting
- **Pros:** Built specifically for Claude, comprehensive tool set
- **Cons:** Newer project, less battle-tested

## Option 3: Google Workspace MCP Server

- **URL:** https://workspacemcp.com/
- **Type:** Full Google Workspace suite (Gmail, Drive, Docs, Sheets, Calendar)
- **Features:** 100+ tools across all Google services
- **Pros:** If we ever need Gmail/Drive/Docs access, it's all there
- **Cons:** Heavier than needed for just Sheets

## Option 4: Claude for Sheets Add-on (Official)

- **URL:** https://support.claude.com/en/articles/13162029
- **Type:** Google Sheets add-on that runs Claude formulas inside cells
- **Features:** =CLAUDE("prompt", cell_reference) formulas
- **Use case:** Different — this puts Claude inside Sheets, not Sheets inside Claude
- **Relevance:** Could be useful for the team side (Dinah/Viviene/Rae using Sheets directly with Claude formulas for data validation)

---

## Recommendation

**Primary: xing5/mcp-google-sheets (Option 1)** — service account auth means the polling script can write to Sheets without user interaction. Production-ready.

**Secondary: Claude for Sheets Add-on (Option 4)** — install this on the bookings Sheet itself so the team gets Claude intelligence directly in their spreadsheet (e.g., validating booking data, flagging incomplete entries).

---

## Bookings Sheet Structure (Draft)

| Column | Description | Updated By |
|--------|-------------|-----------|
| ID | Auto-increment booking reference | Claude |
| Date Received | When the email arrived | Claude |
| Client | Client name | Claude |
| Brand | Product/brand name | Claude |
| Holding Company | Parent company (RFG, Astral, RCL) — drives AP branding | Claude / Team |
| Shift Type | Dry / Wet / Beverage | Claude |
| Base Rate | R/shift | Claude |
| Extras | Giveaways, transport, equipment | Claude |
| Total Rate | Base + Extras | Formula |
| Provinces | Target regions | Claude |
| Store List | Provided / Pending / Partial | Claude / Team |
| Dates | Campaign dates | Claude |
| Shift Count | Total planned shifts | Claude / Team |
| Status | Received / Pending / Confirmed / AP Created / Deployed | Claude / Team |
| Assigned To | Dinah / Viviene / Rae | Team |
| AP Link | Link to generated AP Google Sheet | Claude |
| Email Source | Link/reference to original email | Claude |
| Notes | Free text | Team |
| Last Updated | Timestamp | Auto |

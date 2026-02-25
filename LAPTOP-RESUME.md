# DM Ops Hub — Laptop Resume Guide

## What This Is

This project was initialized on the desktop and pushed to GitHub. Phase 1 execution started but hit a checkpoint requiring Azure Portal and Google Cloud Console access. This guide picks up where we left off.

## Current State

- **Project:** Fully initialized — PROJECT.md, research (8 files), REQUIREMENTS.md (35 reqs), ROADMAP.md (4 phases), STATE.md
- **Phase 1:** 3 plans created, Task 1 of Plan 01-01 committed (security baseline files: .gitignore, .env.example, mcp-config-example.json)
- **Blocked on:** Azure App Registration + Google Cloud project setup (browser work) + .env creation

## Step 1: Clone the Repo

```bash
git clone https://github.com/DanielJKaiser/dm-ops-hub.git
cd dm-ops-hub
```

## Step 2: Complete Cloud Setup (Browser)

### PART A: Azure App Registration

1. Go to https://portal.azure.com/#view/Microsoft_AAD_RegisteredApps/ApplicationsListBlade
2. Click **New registration**
3. Name: `dm-ops-hub` | Single tenant | Redirect URI: Mobile/desktop → `http://localhost:3333/callback`
4. Copy **Application (client) ID** and **Directory (tenant) ID** from Overview
5. Certificates & secrets → New client secret → Description: `dm-ops-hub-secret`, Expires: 24 months → Copy the **Value**
6. API permissions → Add permission → Microsoft Graph → **Delegated**: `offline_access`, `User.Read`, `Mail.Read`, `Mail.ReadWrite`
7. API permissions → Add permission → Microsoft Graph → **Application**: `Mail.Read`
8. Grant admin consent for delegated ONLY (offline_access, User.Read, Mail.Read). Do NOT consent Mail.ReadWrite.

### PART B: Google Cloud Project

1. Go to https://console.cloud.google.com/projectcreate → Name: `dm-ops-hub`
2. Enable **Google Sheets API** and **Google Drive API**
3. IAM & Admin → Service accounts → Create: `dm-ops-hub-sheets` → Keys → JSON → Save as `service-account-key.json` in repo root
4. Copy the service account email

### PART C: Google Drive Folder

1. Create folder `DM Ops Hub` in Google Drive
2. Share with service account email as **Editor**
3. Copy folder ID from URL

### PART D: Create .env

```bash
cp .env.example .env
```

Fill in real values:
- `MS_CLIENT_ID` — from Azure step 4
- `MS_CLIENT_SECRET` — from Azure step 5
- `MS_TENANT_ID` — from Azure step 4
- `GOOGLE_SERVICE_ACCOUNT_PATH=./service-account-key.json`
- `GOOGLE_DRIVE_FOLDER_ID` — from Drive step 3
- `BOOKINGS_TRACKER_SPREADSHEET_ID=placeholder` (filled later in Plan 01-03)

## Step 3: Resume with Claude

Open Claude Code in the repo directory and paste this:

```
/gsd:resume-work
```

Or if that doesn't pick up context, paste this directly:

```
I just cloned dm-ops-hub on my work laptop. The cloud setup is done:
- Azure App Registration created with Mail.Read permissions
- Google Cloud project with Sheets + Drive APIs enabled
- Service account key saved as service-account-key.json
- .env created with real credentials
- Google Drive folder shared with service account

Resume Phase 1 execution. Plan 01-01 Task 1 is done (security baseline committed). Task 2 (cloud setup checkpoint) is now complete. Continue with Wave 2: Plan 01-02 (MCP server installation) and Plan 01-03 (Bookings Tracker creation) in parallel.
```

## What Happens Next

1. Claude verifies .env and service account key exist
2. **Plan 01-02:** Installs Google Sheets MCP + Outlook MCP, verifies both connect
3. **Plan 01-03:** Creates Bookings Tracker spreadsheet with full schema, status dropdowns, colour coding
4. Phase 1 verification against success criteria
5. Auto-advances to Phase 2 (Polling Script and Data Model)

## File Structure Reference

```
dm-ops-hub/
├── .planning/
│   ├── PROJECT.md              ← Project context
│   ├── config.json             ← Workflow settings (yolo, standard, parallel)
│   ├── REQUIREMENTS.md         ← 35 v1 requirements
│   ├── ROADMAP.md              ← 4 phases, 15 plans
│   ├── STATE.md                ← Current position tracker
│   ├── research/               ← 8 research documents
│   │   ├── STACK.md, FEATURES.md, ARCHITECTURE.md, PITFALLS.md, SUMMARY.md
│   │   └── architecture-overview.md, outlook-mcp-options.md, google-sheets-mcp-options.md
│   └── phases/
│       └── 01-infrastructure-and-auth/
│           ├── 01-CONTEXT.md   ← Phase 1 decisions
│           ├── 01-RESEARCH.md  ← Phase 1 technical research
│           ├── 01-01-PLAN.md   ← Credentials + security (Task 1 done, Task 2 = checkpoint)
│           ├── 01-02-PLAN.md   ← MCP server installation
│           └── 01-03-PLAN.md   ← Bookings Tracker creation
├── .env.example                ← Template (committed)
├── .gitignore                  ← Secret exclusions (committed)
├── mcp-config-example.json     ← MCP config template (committed)
├── .env                        ← YOUR SECRETS (gitignored, create from .env.example)
├── service-account-key.json    ← Google SA key (gitignored, download from GCP)
└── LAPTOP-RESUME.md            ← This file
```

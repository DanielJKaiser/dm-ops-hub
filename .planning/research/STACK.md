# Stack Research

**Domain:** Email-to-Booking Automation Pipeline (Microsoft Outlook + Claude + Google Sheets)
**Researched:** 2026-02-25
**Confidence:** MEDIUM — Core MCP server choices verified from prior research (2026-02-24). Python library versions reflect training knowledge (cutoff August 2025); verify exact versions at install time. No live web verification available in this session.

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **ryaker/outlook-mcp** | Latest (GitHub) | MCP server for Outlook email read access via Microsoft Graph API | Self-hosted, full feature set (threading, headers, folders, attachments), no third-party dependency for a production system. Composio is faster for prototyping but adds an external SaaS layer to a core business pipeline. |
| **xing5/mcp-google-sheets** | Latest (GitHub) | MCP server for Google Sheets read/write | Python-based, service account auth (critical: no user interaction required for unattended polling), supports formatting and formula creation needed for AP generation. |
| **Python** | 3.11+ | Polling script runtime + MCP server runtime | Both MCP servers are Python-based. 3.11 minimum for performance improvements and better error messages. 3.12 safe as of 2025. |
| **Microsoft Graph API** | v1.0 | Email data source (REST) | The only official, supported way to access Outlook/Exchange programmatically. v1.0 is stable production endpoint; v2.0 (beta) not ready. |
| **MSAL for Python** | 1.28.x | Microsoft OAuth 2.0 token management for polling script | Official Microsoft Authentication Library. Handles token acquisition, refresh, and caching so the polling script stays authenticated without manual intervention. Do not implement OAuth manually. |
| **Google APIs Client Library for Python (google-api-python-client)** | 2.x | Direct Google Sheets API calls when MCP server isn't available | Needed for the polling script to write to the Inbox Queue sheet without routing through Claude. Complements the MCP server (which is for Claude's direct use). |
| **Claude API via Anthropic SDK** | 0.28.x+ | Intelligence layer for email parsing and booking extraction | Provides claude-opus-4-5 and claude-sonnet-4-5 access. Sonnet is the right model for this workload: cheaper per token, still excellent at structured extraction from messy text. Opus only for truly ambiguous multi-step reasoning. |
| **Windows Task Scheduler** | (built-in) | Schedule the polling script every 15 minutes | No additional tooling needed. Built into Windows. Python script + `.bat` launcher registered as a scheduled task. Simple and reliable for a local-first system. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| **gspread** | 5.x | Higher-level Python wrapper around Google Sheets API | Use in the polling script for writing to the Inbox Queue sheet. Simpler API than google-api-python-client for row appends and value reads. Do not use for cell formatting — use google-api-python-client directly for that. |
| **google-auth** | 2.x | Google service account credential management | Required by both gspread and google-api-python-client. Handles loading the service account JSON key and refreshing credentials. |
| **python-dotenv** | 1.x | Load .env variables for secrets management | Store CLIENT_ID, CLIENT_SECRET, TENANT_ID, and Google service account path in `.env`. Never hardcode credentials in script files. |
| **requests** | 2.31+ | HTTP fallback for Graph API calls | If msgraph-sdk adds too much complexity, direct HTTP calls to Graph API with requests + MSAL tokens is a valid and transparent alternative for the polling script. |
| **schedule** | 1.2.x | In-process polling loop (alternative to Task Scheduler) | Use only if Windows Task Scheduler integration is problematic. schedule runs a Python loop that calls the email check function every N minutes. Simpler to debug but requires the process to stay running. |
| **pytest** | 7.x+ | Test suite for polling script | Test email parsing logic, field extraction heuristics, and status transition logic in isolation before wiring to live APIs. Critical for booking detection accuracy. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| **Claude Code** | Primary development environment and runtime for Claude intelligence layer | Claude Code is the orchestrator. The polling script feeds data in; Claude Code sessions process it. This is the "always-on" interface for on-demand and session-based email processing. |
| **Azure Portal** | Azure App Registration for Graph API auth | Register at portal.azure.com. Create App Registration → generate Client Secret → configure Mail.Read + Mail.ReadWrite + User.Read permissions. This is a one-time setup. |
| **Google Cloud Console** | Google Cloud project, Sheets API enablement, service account creation | console.cloud.google.com. Enable Sheets API, create service account, download JSON key. Share target Sheets with the service account email address. One setup unblocks both this project and the AP system. |
| **uv** (or pip) | Python package management | uv is faster and handles virtual environments better than pip alone. Use `uv venv` + `uv pip install` for the polling script environment. |
| **GitHub (dm-ops-hub)** | Version control for scripts, config, and prompts | The polling script, `.env.example`, MCP config, and Claude prompts all live here. Enables cross-device sync (Daniel's work machine + any other device). |

---

## Installation

```bash
# Create isolated environment for polling script
uv venv .venv
source .venv/Scripts/activate  # Windows Git Bash
# OR: .venv\Scripts\activate  # Windows CMD

# Core: Microsoft Graph access
uv pip install msal requests

# Core: Google Sheets access
uv pip install gspread google-auth google-api-python-client

# Core: Anthropic SDK (for direct API calls if needed outside Claude Code)
uv pip install anthropic

# Utilities
uv pip install python-dotenv schedule

# Dev / testing
uv pip install pytest
```

MCP servers (installed separately into Claude's config, not the project venv):

```bash
# Outlook MCP — clone and configure
git clone https://github.com/ryaker/outlook-mcp
# Follow repo README for Claude Code config entry

# Google Sheets MCP
git clone https://github.com/xing5/mcp-google-sheets
# Follow repo README for Claude Code config entry
```

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| ryaker/outlook-mcp (self-hosted) | Composio Outlook MCP | If you need to demo the system in under 30 minutes and don't have Azure App Registration ready. Composio's managed OAuth is faster for prototyping. Migrate away for production — Composio adds a third-party dependency on a business-critical email pipeline. |
| ryaker/outlook-mcp (self-hosted) | Official Anthropic Microsoft 365 Connector | Only viable if Daniel upgrades to Claude Team or Enterprise plan AND has Microsoft Entra ID Global Admin access. Not the current setup. |
| xing5/mcp-google-sheets | Google Workspace MCP (workspacemcp.com) | If the project expands to need Gmail, Drive, Docs, and Calendar in addition to Sheets. Overkill for Sheets-only. |
| xing5/mcp-google-sheets | ringo380/claude-google-sheets-mcp | If xing5 becomes unmaintained. ringo380 is built specifically for Claude CLI but is a newer project with less production history. |
| MSAL for Python | Raw OAuth implementation | Never. MSAL handles token expiry, refresh, caching, and all the edge cases. Implementing OAuth manually against Microsoft's endpoints is weeks of debugging. |
| gspread | google-api-python-client directly | When you need full formatting control (cell colours, merged cells) or batch updates on large ranges. The polling script uses gspread for simple row appends; AP generation logic should use google-api-python-client for the formatting capabilities. |
| Windows Task Scheduler | Celery + Redis | Only if this becomes a multi-user SaaS product. Celery is a distributed task queue that adds significant infrastructure (Redis broker, worker processes). For a local-first single-machine pipeline, Task Scheduler is the right tool. |
| Windows Task Scheduler | GitHub Actions cron | If the script moves to a cloud host (e.g., small VPS). Not appropriate while the system is local-first on Daniel's machine. |
| claude-sonnet-4-5 (default model) | claude-opus-4-5 | Use Opus only for genuinely ambiguous extraction tasks — e.g., an email with no clear client name, contradictory dates, or unusual structure. Sonnet handles 95%+ of booking extractions at 5x lower cost. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **Zapier MCP for Outlook** | Adds a paid SaaS dependency (Zapier subscription) between Outlook and Claude. Creates a black-box automation layer that's hard to debug when emails are misclassified. No real advantage over direct Graph API access. | ryaker/outlook-mcp with direct Graph API |
| **Microsoft Graph REST SDK (beta endpoint)** | The `/beta` endpoint is undocumented, unsupported, and can break without notice. Only v1.0 is production-safe. | Microsoft Graph API v1.0 only |
| **OAuth 2.0 Authorization Code Flow for polling script** | Requires a user to click "Allow" in a browser and produces tokens that expire. An unattended polling script needs credentials that work with no human interaction. | MSAL Client Credentials flow (app-level auth, no user needed) for polling; delegation tokens only for interactive Claude Code sessions |
| **Service account auth for Outlook/Graph API** | Microsoft does not support service accounts the same way Google does. Graph API uses Azure App Registrations with app-level permissions (Client Credentials flow) or delegated user permissions. "Service account" is a Google concept. | Azure App Registration with Client Credentials (app-only auth) or delegated auth via MSAL |
| **Zapier / Make (Integromat) as the integration layer** | Both add monthly costs and another failure point. They also abstract away the logic, making it impossible to customize the booking detection heuristics or add nuanced Claude prompting. | Direct Graph API + Google Sheets API calls in Python |
| **Email parsing via regex alone** | Booking emails are casual, varied, and multilingual (SA English). Regex will break on every new client format. This is exactly the problem Claude solves. | Claude as the primary parser; regex only for pre-filtering (e.g., does the subject contain "booking", "shifts", "campaign"?) |
| **Google Apps Script for Sheets automation** | Apps Script is sandboxed, has strict quotas, and creates a second codebase in a different language. Hard to test, hard to version control, hard to integrate with Claude. | Python + google-api-python-client or gspread, all in the same repo |
| **Node.js for the polling script** | Both MCP servers (outlook-mcp and mcp-google-sheets) are Python. Running a Node.js polling script alongside Python MCP servers creates two runtime environments to manage. | Python 3.11+ for the entire polling layer |

---

## Stack Patterns by Variant

**If running session-based only (no polling script yet):**
- Use Composio Outlook MCP for fastest setup
- Claude reads inbox manually on session start
- No Task Scheduler, no polling script
- Good for Phase 1 validation before investing in infrastructure

**If adding continuous monitoring (Phase 2+):**
- Switch to ryaker/outlook-mcp (self-hosted)
- Write `email_poller.py` using MSAL Client Credentials flow
- Register as Windows Scheduled Task (every 15 minutes)
- Writes new emails to "Inbox Queue" tab in Google Sheet
- Claude processes queue on session start or on-demand

**If moving toward event-driven (V2 future):**
- Microsoft Graph webhooks (push notifications on new email)
- Requires a public HTTPS endpoint to receive webhook POST requests
- Use ngrok for development, a small VPS or Azure Function for production
- Eliminates polling entirely — Claude is triggered on email arrival

**If the AP generation needs complex formatting:**
- Use google-api-python-client batchUpdate for cell formatting (colours, borders, merged cells)
- gspread handles data writes; google-api-python-client handles formatting
- Both are in the same environment; no conflict

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| msal 1.28.x | Python 3.8–3.12 | MEDIUM confidence — verified against MSAL changelog knowledge as of Aug 2025. Verify at install. |
| gspread 5.x | google-auth 2.x | These two must be version-matched. gspread 5.x dropped support for older google-auth. |
| google-api-python-client 2.x | google-auth 2.x | Same google-auth version must satisfy both gspread and google-api-python-client. pip/uv resolves this automatically. |
| anthropic SDK 0.28.x+ | Python 3.8+ | The SDK follows semantic versioning. Pin to a minor version in requirements.txt; update deliberately when new model versions drop. |
| ryaker/outlook-mcp | Python 3.10+ | LOW confidence — no live version verification. Check repo README at install time. |
| xing5/mcp-google-sheets | Python 3.9+ | LOW confidence — no live version verification. Check repo README at install time. |

---

## Authentication Architecture

### Microsoft Graph (Outlook)

Two distinct auth contexts exist in this system:

**Context 1: Polling script (unattended)**
- Flow: Client Credentials (app-only)
- Credentials: CLIENT_ID + CLIENT_SECRET + TENANT_ID from Azure App Registration
- No user interaction, works headless on Task Scheduler
- Requires: `Mail.Read` application permission (not delegated) in Azure portal

**Context 2: Claude Code sessions (interactive)**
- Flow: OAuth 2.0 Authorization Code via ryaker/outlook-mcp
- MCP server handles token storage and refresh
- Requires: `Mail.Read` + `Mail.ReadWrite` delegated permissions

Both flows share the same Azure App Registration but use different permission types.

### Google Sheets

Single auth context for both polling script and MCP server:
- Flow: Service Account (JSON key file)
- No user interaction required
- Service account email must be added as an Editor to each target Sheet
- Key file stored at a fixed path, referenced in `.env`

---

## Sources

- `.planning/research/outlook-mcp-options.md` (2026-02-24) — Outlook MCP comparison, Composio vs ryaker vs Zapier vs Anthropic connector. HIGH confidence (project-specific prior research).
- `.planning/research/google-sheets-mcp-options.md` (2026-02-24) — Google Sheets MCP comparison, xing5 vs ringo380 vs Workspace MCP. HIGH confidence (project-specific prior research).
- `.planning/research/architecture-overview.md` (2026-02-24) — Authentication flows, monitoring strategy, data flow. HIGH confidence (project-specific prior research).
- `.planning/PROJECT.md` — Booking data model, team structure, scope boundaries. HIGH confidence (primary source of truth).
- Training knowledge (cutoff August 2025) — Python library versions (MSAL, gspread, google-api-python-client, anthropic SDK). MEDIUM confidence — exact versions should be verified at `pip install` time via PyPI.
- Microsoft Graph API documentation knowledge — v1.0 vs beta endpoint stability, Client Credentials vs Authorization Code flow. MEDIUM confidence — core OAuth behavior is stable but always verify permission scopes in Azure portal.

---

*Stack research for: DM Ops Hub — Email-to-Booking Automation Pipeline*
*Researched: 2026-02-25*

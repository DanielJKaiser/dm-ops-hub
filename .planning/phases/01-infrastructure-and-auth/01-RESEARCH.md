# Phase 1: Infrastructure and Auth - Research

**Researched:** 2026-02-25
**Domain:** Azure/Google Cloud credentials, MCP server setup, Google Sheets schema, security baseline
**Confidence:** HIGH

## Summary

Phase 1 establishes three pillars: (1) Azure App Registration for Outlook access via ryaker/outlook-mcp (littlebearapps fork), (2) Google Cloud project with service account for Sheets API via xing5/mcp-google-sheets, and (3) the Bookings Tracker spreadsheet with full column schema. The MCP ecosystem for both Outlook and Google Sheets is mature with well-documented setup paths. The key complexity is the split-machine workflow: desktop handles Google Cloud + Azure registration + Sheets work, while the work laptop handles Outlook MCP testing with actual mailbox access.

Both MCP servers use standard auth flows (OAuth 2.0 delegated for Outlook, service account key for Sheets) and install via npm/uvx respectively. The Bookings Tracker schema is fully specified in CONTEXT.md with 20 columns and 6 status states. Security baseline is straightforward: .env for secrets, .gitignore for exclusion, example files for replication.

**Primary recommendation:** Execute in two waves -- Wave 1 handles Azure App Reg + Google Cloud project setup (browser-based, no code), Wave 2 handles MCP server installation, Bookings Tracker creation, and security baseline (code/config on desktop, Outlook MCP verification deferred to laptop).

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Claude's discretion on Composio vs ryaker/outlook-mcp -- pick what's fastest to unblock Phase 2
- If starting with Composio, document the migration path to ryaker explicitly
- Azure App Registration walkthrough: step-by-step guidance with links -- Daniel is not familiar with Azure portal
- Permissions: Mail.Read only for v1 (read-only). Register Mail.ReadWrite in the app but don't grant it yet -- future expansion path
- Multiple inboxes eventually (Daniel, Dinah, Sheilagh) but start with Daniel's inbox only for testing
- Team laptops get the setup only once everything is 100% working and easily pullable from git
- Separate spreadsheets, NOT one mega-spreadsheet with everything as tabs
- Main operational spreadsheet has 4 tabs: Bookings Tracker, Inbox Queue, Review Queue, Config/Lookups
- APs are separate Google Spreadsheets, split by brand AND campaign
- Day one access: Daniel + Google service account only. Team added after system is proven.
- .env file in repo root for all secrets
- .gitignore excludes .env, service account key, and actual MCP config files
- .env.example committed with placeholder values + setup documentation
- Status flow (6 states): Received, Details Pending, Details Confirmed, Ready for AP, AP Created, Deployed for Human Review
- Column order (status-first): Status, Client, Brand, Holding Company, Shift Type, Base Rate, Extras, Total Rate, Provinces, Store List, Dates, Shift Count, Assigned To, AP Link, Email Source, Notes, Completeness %, Date Received, Last Updated, ID
- Colour-coded statuses from day one
- Frozen columns: minimal or none. Header row frozen only. Do NOT freeze columns.
- Data validation dropdowns on Status column

### Claude's Discretion
- Composio vs ryaker decision for initial MCP server
- Service account JSON key storage location
- MCP config generation approach (example file vs script)
- Exact status colour palette for conditional formatting
- Column width and general sheet formatting details
- Whether to include Completeness % formula in Phase 1 or defer to Phase 2

### Deferred Ideas (OUT OF SCOPE)
- AP merge detection (flagging multiple bookings that are actually the same campaign) -- Phase 4
- Team laptop deployment and multi-inbox setup -- after v1 is proven
- To-do list tabs -- Phase 3
- Mail.ReadWrite permissions activation -- future phase when email flagging is needed
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| INFRA-01 | Azure App Registration configured with Mail.Read application permission (polling) and delegated permissions (Outlook MCP) | outlook-mcp docs specify required delegated permissions: offline_access, User.Read, Mail.Read, Mail.ReadWrite (register but don't grant). Application permission Mail.Read needed for polling script (Phase 2). |
| INFRA-02 | Google Cloud project created with Sheets API enabled and service account JSON key generated | mcp-google-sheets requires SERVICE_ACCOUNT_PATH env var. Standard GCP flow: create project, enable Sheets API, create service account, download JSON key. |
| INFRA-03 | Outlook MCP server (ryaker/outlook-mcp) installed and authenticated -- Claude can read emails in a session | littlebearapps/outlook-mcp is the active fork. Install via npm, configure Claude Desktop/Code MCP config with client ID/secret env vars. Requires OAuth auth-server flow on port 3333. |
| INFRA-04 | Google Sheets MCP server (xing5/mcp-google-sheets) installed and authenticated -- Claude can read/write Sheets | Install via uvx (Python/uv). Requires SERVICE_ACCOUNT_PATH and DRIVE_FOLDER_ID env vars. Service account must be shared on target Drive folder. |
| INFRA-05 | Bookings Tracker Google Sheet created with full column schema | 20 columns defined in CONTEXT.md. 4 tabs in main spreadsheet. Data validation dropdowns for Status column. Conditional formatting for status colours. |
| INFRA-06 | Security baseline established -- .env for secrets, .gitignore excludes credentials, mcp-config-example.json committed | Standard .env pattern. .gitignore must exclude .env, *.json service account keys, actual MCP config. Commit .env.example and mcp-config-example.json with placeholders. |
</phase_requirements>

## Standard Stack

### Core

| Tool | Version | Purpose | Why Standard |
|------|---------|---------|--------------|
| littlebearapps/outlook-mcp | latest | Outlook email access via MCP | Active fork of ryaker/outlook-mcp; Node.js-based MCP server for Microsoft Graph API |
| xing5/mcp-google-sheets | latest (uvx) | Google Sheets read/write via MCP | Python-based MCP server; service account auth; official Google Sheets API wrapper |
| Azure App Registration | N/A | OAuth app for Microsoft Graph API | Required for any Outlook API access; provides client ID/secret for auth flows |
| Google Cloud Console | N/A | Service account + Sheets API | Required for Google API access; service account enables headless auth |

### Supporting

| Tool | Purpose | When to Use |
|------|---------|-------------|
| Node.js | Runtime for outlook-mcp | Required by outlook-mcp |
| Python/uv/uvx | Runtime for mcp-google-sheets | Required by mcp-google-sheets; uvx handles virtual env |
| dotenv | Secret management | .env file pattern for all credentials |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| littlebearapps/outlook-mcp | Composio | Composio is hosted service (simpler setup) but adds vendor dependency; ryaker fork is self-hosted and more transparent |
| xing5/mcp-google-sheets | isaacphi/mcp-gdrive | mcp-gdrive handles Drive+Sheets but is broader scope; xing5 is Sheets-focused and simpler |
| Service account auth | OAuth 2.0 user auth | Service account is headless (no browser popup); better for automation; user auth better for per-user access |

**Recommendation on Composio vs ryaker:** Use littlebearapps/outlook-mcp (the active fork of ryaker). It is self-hosted, transparent, well-documented, and avoids vendor lock-in. The setup is slightly more involved (Azure App Reg + local OAuth flow) but this registration is needed regardless for the Phase 2 polling script. Starting with the same Azure app from day one avoids credential duplication.

## Architecture Patterns

### Recommended Project Structure
```
dm-ops-hub/
├── .env                          # Secrets (gitignored)
├── .env.example                  # Placeholder template (committed)
├── .gitignore                    # Excludes secrets and configs
├── mcp-config-example.json       # MCP config template (committed)
├── scripts/
│   ├── setup-outlook-mcp.ps1     # Outlook MCP setup helper (Windows)
│   └── verify-mcp.sh             # MCP connection verification
└── .planning/                    # Planning artifacts
```

### Pattern 1: Split-Machine Workflow
**What:** Desktop handles all browser-based setup (Azure portal, GCP console) and Google Sheets work. Work laptop handles Outlook MCP testing.
**When to use:** When the development machine lacks Outlook.
**Implementation:**
1. Desktop: Azure App Reg (browser), GCP project (browser), Sheets MCP + Tracker (API)
2. Push to GitHub
3. Laptop: Clone, copy .env, install Outlook MCP, authenticate, test email read

### Pattern 2: MCP Config as Example File
**What:** Commit `mcp-config-example.json` with placeholder values. Actual MCP config lives outside repo or in a gitignored location.
**When to use:** Always -- never commit real credentials.
**Implementation:**
```json
{
  "mcpServers": {
    "outlook": {
      "command": "node",
      "args": ["/path/to/outlook-mcp/index.js"],
      "env": {
        "OUTLOOK_CLIENT_ID": "YOUR_AZURE_CLIENT_ID",
        "OUTLOOK_CLIENT_SECRET": "YOUR_AZURE_CLIENT_SECRET"
      }
    },
    "google-sheets": {
      "command": "uvx",
      "args": ["mcp-google-sheets@latest"],
      "env": {
        "SERVICE_ACCOUNT_PATH": "/path/to/service-account-key.json",
        "DRIVE_FOLDER_ID": "YOUR_DRIVE_FOLDER_ID"
      }
    }
  }
}
```

### Pattern 3: Service Account Folder Sharing
**What:** Google service account accesses Sheets through a shared Drive folder, not individual file sharing.
**When to use:** When multiple spreadsheets need service account access.
**Implementation:** Create a Drive folder, share it with the service account email, set DRIVE_FOLDER_ID. All spreadsheets in this folder are accessible.

### Anti-Patterns to Avoid
- **Committing secrets to git:** Even temporarily -- git history preserves them forever
- **Hardcoding paths:** Use .env for all path configuration; different machines have different layouts
- **Skipping .env.example:** Team members (and laptop clone) need to know what variables are required
- **Over-permissioning Azure app:** Start with Mail.Read only; register Mail.ReadWrite but don't consent

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Outlook email access | Custom Graph API client | littlebearapps/outlook-mcp | Handles OAuth flow, token refresh, MCP protocol |
| Google Sheets access | Custom Sheets API client | xing5/mcp-google-sheets | Handles auth, API wrapper, MCP protocol |
| Secret management | Custom config loader | .env + dotenv pattern | Standard, well-understood, works across Node/Python |
| Google Sheets schema | Manual column creation | Google Sheets API batch update | Programmatic creation ensures consistency |

## Common Pitfalls

### Pitfall 1: Azure Redirect URI Mismatch
**What goes wrong:** OAuth flow fails with redirect_uri mismatch error
**Why it happens:** Azure App Registration redirect URI doesn't match what outlook-mcp sends
**How to avoid:** Set redirect URI to `http://localhost:3333/callback` in Azure portal (Mobile and desktop applications platform)
**Warning signs:** "AADSTS50011: The redirect URI specified in the request does not match" error

### Pitfall 2: Service Account Not Shared on Folder
**What goes wrong:** mcp-google-sheets returns 403 or empty results
**Why it happens:** Service account email was not added as editor on the Drive folder
**How to avoid:** After creating service account, copy its email (looks like `name@project.iam.gserviceaccount.com`), share the Drive folder with it as Editor
**Warning signs:** "The caller does not have permission" errors

### Pitfall 3: Windows Path Issues with MCP Config
**What goes wrong:** MCP server fails to start with ENOENT errors
**Why it happens:** Windows backslash paths in JSON config; or spaces in path not quoted
**How to avoid:** Use forward slashes in JSON config even on Windows, or escape backslashes. Use absolute paths.
**Warning signs:** "ENOENT: no such file or directory" on server start

### Pitfall 4: uvx Not Found on Windows
**What goes wrong:** Google Sheets MCP server fails to start
**Why it happens:** uv/uvx not installed or not in PATH on Windows
**How to avoid:** Install uv first (`pip install uv` or `winget install astral-sh.uv`), verify `uvx --version` works
**Warning signs:** "'uvx' is not recognized as an internal or external command"

### Pitfall 5: Outlook MCP Auth Server Not Running
**What goes wrong:** OAuth flow hangs or fails during first authentication
**Why it happens:** outlook-mcp requires a local auth server on port 3333 for the initial OAuth handshake
**How to avoid:** Run `npm run auth-server` before attempting first authentication, complete the browser OAuth flow, then auth-server is no longer needed
**Warning signs:** Browser opens but auth callback fails; "connection refused" on localhost:3333

### Pitfall 6: Google Sheets API Not Enabled
**What goes wrong:** API calls return 403 "Google Sheets API has not been used in project"
**Why it happens:** Created GCP project but forgot to enable Sheets API
**How to avoid:** Explicitly enable both Google Sheets API and Google Drive API in GCP console
**Warning signs:** "Google Sheets API has not been used in project X before or it is disabled"

## Code Examples

### .env.example Template
```bash
# Azure / Microsoft Graph (Outlook MCP + Polling Script)
MS_CLIENT_ID=your-azure-application-client-id
MS_CLIENT_SECRET=your-azure-client-secret-value
MS_TENANT_ID=your-azure-tenant-id

# Google Cloud (Sheets MCP)
GOOGLE_SERVICE_ACCOUNT_PATH=./service-account-key.json
GOOGLE_DRIVE_FOLDER_ID=your-shared-drive-folder-id

# Bookings Tracker
BOOKINGS_TRACKER_SPREADSHEET_ID=your-spreadsheet-id
```

### .gitignore Entries
```
# Secrets
.env
*.json
!package.json
!package-lock.json
!mcp-config-example.json
!.planning/**/*.json

# Service account key
service-account-key.json
*-service-account*.json

# MCP config (actual)
claude_desktop_config.json
```

### MCP Config Example (mcp-config-example.json)
```json
{
  "mcpServers": {
    "outlook": {
      "command": "node",
      "args": ["C:/path/to/outlook-mcp/index.js"],
      "env": {
        "OUTLOOK_CLIENT_ID": "YOUR_AZURE_CLIENT_ID",
        "OUTLOOK_CLIENT_SECRET": "YOUR_AZURE_CLIENT_SECRET"
      }
    },
    "google-sheets": {
      "command": "uvx",
      "args": ["mcp-google-sheets@latest"],
      "env": {
        "SERVICE_ACCOUNT_PATH": "C:/path/to/service-account-key.json",
        "DRIVE_FOLDER_ID": "YOUR_DRIVE_FOLDER_ID"
      }
    }
  }
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| ryaker/outlook-mcp | littlebearapps/outlook-mcp | 2025 | Active fork with maintenance; same API |
| pip install mcp-google-sheets | uvx mcp-google-sheets@latest | 2025 | uvx handles isolated environments; no global install |
| Manual OAuth token management | MCP servers handle token refresh | 2025 | Transparent auth in MCP protocol |

## Open Questions

1. **outlook-mcp Node.js version requirement**
   - What we know: Uses npm, standard Node.js project
   - What's unclear: Minimum Node.js version requirement
   - Recommendation: Check package.json engines field during installation; Node 18+ should be safe

2. **Completeness % formula timing**
   - What we know: CONTEXT.md lists it as Claude's discretion
   - What's unclear: Whether formula depends on Phase 2 column ownership rules
   - Recommendation: Include a simple COUNTA-based formula in Phase 1; Phase 2 can refine if needed

3. **Conditional formatting via API**
   - What we know: Google Sheets API supports conditional formatting rules
   - What's unclear: Whether mcp-google-sheets exposes conditional formatting or if direct API call is needed
   - Recommendation: Plan for direct Sheets API batch update if MCP tool doesn't support formatting rules

## Sources

### Primary (HIGH confidence)
- /littlebearapps/outlook-mcp (Context7) - Installation, configuration, Azure setup, permissions
- /xing5/mcp-google-sheets (Context7) - Installation, service account auth, env vars, Claude Desktop config

### Secondary (MEDIUM confidence)
- Azure App Registration docs (Microsoft) - Delegated vs application permissions model
- Google Cloud Console - Service account creation, API enablement

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Both MCP servers verified via Context7 with current docs
- Architecture: HIGH - Standard credential/MCP patterns, well-documented
- Pitfalls: HIGH - Common OAuth/permission issues documented in official READMEs

**Research date:** 2026-02-25
**Valid until:** 2026-03-25 (stable ecosystem, unlikely to change rapidly)

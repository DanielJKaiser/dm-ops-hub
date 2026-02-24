# Outlook MCP Integration Options

> Researched: 2026-02-24

## Overview

All options use Microsoft Graph API under the hood. The MCP server wraps it so Claude can call tools like `list_emails`, `search_emails`, `read_email` directly.

**Prerequisite for all:** Azure App Registration (free) with Microsoft Graph permissions (Mail.Read minimum, Mail.ReadWrite for full access).

---

## Option 1: Composio Outlook MCP (Recommended for v1)

- **URL:** https://mcp.composio.dev/outlook
- **Why:** Managed OAuth — handles the authentication flow for you. 64+ tools out of the box.
- **Features:** Read, search, send, folder management, email rules, calendar
- **Setup:** Install via Composio, connect Outlook account through their OAuth flow
- **Pros:** Easiest setup, no self-hosting, handles token refresh
- **Cons:** Third-party dependency, may have usage limits on free tier

## Option 2: ryaker/outlook-mcp (Most Feature-Rich)

- **URL:** https://github.com/ryaker/outlook-mcp
- **Features:** Email management, headers/forensics, conversation threading, calendar, folder management, rules, contacts, categories, focused inbox, mailbox settings
- **Setup:** Azure App Registration → configure CLIENT_ID + CLIENT_SECRET → add to Claude config
- **Pros:** Self-hosted, full feature set, no third-party dependency
- **Cons:** More setup required (Azure portal config)

## Option 3: Zapier MCP for Outlook

- **URL:** https://zapier.com/mcp/microsoft-outlook
- **Features:** Connects Outlook to 6000+ apps. Can trigger workflows from email events.
- **Pros:** Could bridge Outlook → Google Sheets without custom code
- **Cons:** Requires Zapier subscription, adds another dependency layer

## Option 4: Official Anthropic Microsoft 365 Connector

- **URL:** https://support.claude.com/en/articles/12542951
- **Features:** SharePoint, OneDrive, Outlook, Teams — unified access
- **Limitation:** Requires Claude Team or Enterprise plan + Microsoft Entra ID Global Admin setup
- **Status:** Not viable for current setup (would need plan upgrade)

---

## Recommendation

**Start with Composio (Option 1)** for fastest time-to-value. If we hit limits or need more control, migrate to **ryaker/outlook-mcp (Option 2)** which is self-hosted and fully featured.

For the polling/continuous monitoring requirement, **Option 2** is better long-term because we can run the MCP server locally with a cron job or scheduled task that checks for new emails every X minutes.

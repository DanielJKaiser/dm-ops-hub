# Pitfalls Research

**Domain:** Email-to-booking automation pipeline (NLP extraction, Microsoft Graph API, Google Sheets)
**Researched:** 2026-02-25
**Confidence:** MEDIUM — External search unavailable. Based on documented API behaviour (Graph API docs, Google Sheets API docs), well-established LLM automation failure modes, and project-specific context from discovery. Flagged where verification with live docs is recommended.

---

## Critical Pitfalls

### Pitfall 1: OAuth Token Refresh Failure Silently Breaks the Polling Loop

**What goes wrong:**
The Python polling script authenticates with Microsoft Graph API using an OAuth access token (short-lived, ~60 minutes) and a refresh token (long-lived). If the refresh token expires, is revoked, or the Azure App registration changes permissions, the polling script silently fails — it throws an authentication exception, logs nothing visible to the team, and stops writing new emails to the Inbox Queue sheet. The team sees no new bookings in Sheets but doesn't know why. Bookings get missed until someone manually notices the queue is stale.

**Why it happens:**
Refresh tokens for Azure apps with `offline_access` scope can expire after 90 days of inactivity or immediately if a tenant admin revokes app consent. Developers wire up the happy path, test it on day one, and don't build retry logic or alerting for authentication failures. The polling script doesn't distinguish "no new emails" from "can't reach the API."

**How to avoid:**
1. Store the refresh token in a persistent, secure file (not in-memory only). On each polling cycle, proactively check if the access token is within 10 minutes of expiry before making API calls.
2. On any `401 Unauthorized` or `403 Forbidden` response, write a "AUTH_FAILURE" sentinel row to the Inbox Queue sheet with a timestamp. This gives the team a visible signal.
3. Set a Windows Task Scheduler alert or a simple check: if the Inbox Queue sheet has had no new rows written in X hours during business hours, flag it.
4. For ryaker/outlook-mcp (self-hosted), re-read the token handling code and confirm it persists refresh tokens across restarts.
5. For Composio (managed), check their documentation on what happens when a token expires — does their dashboard alert you, or does it silently drop?

**Warning signs:**
- Inbox Queue sheet has no new entries during a business day when email volume is normally high
- Last polling timestamp in the queue is more than 30 minutes old during business hours
- Python script exits without an error log entry for "checked emails successfully"

**Phase to address:** Phase 1 (Infrastructure Setup) — before any parsing logic is built, the auth persistence and failure signalling must be implemented.

---

### Pitfall 2: Forwarded Email Chains Confuse Claude's Extraction

**What goes wrong:**
A large fraction of DM's booking emails arrive via Sheilagh or Dinah forwarding a client email. Forwarded chains in Outlook add headers like "From: Sheilagh... Forwarded message... From: [Client]..." and often wrap the original email body inside a quoted block. Claude may:
- Attribute the booking to Sheilagh as the client (she's the sender of the email it reads)
- Extract fields from the forwarding note ("Please handle this one") instead of the original brief
- Miss the original attachment because Graph API returns the top-level email, and attachments are on the inner forwarded message which is body text, not a Graph attachment object
- Get confused by multiple quoted layers if the email was already a reply chain before being forwarded

**Why it happens:**
Microsoft Graph API returns the raw email body as a single blob. Forwarded email structure is not a standard — different email clients format forwarded headers differently. Claude sees "From:", "Subject:", "Date:" in the middle of the body and may not reliably distinguish these as quoted headers vs. a new email.

**How to avoid:**
1. In the Claude parsing prompt, include an explicit instruction: "If the email body contains forwarded message headers ('---------- Forwarded message ----------', 'From:', 'Sent:', 'To:', 'Subject:' in the body), extract booking information from the original forwarded content, not from the forwarding note."
2. Write a pre-processing function in the polling script that detects forwarded email patterns and splits the body into: (a) forwarding note, (b) original email. Pass both segments separately to Claude with labels.
3. For client detection: cross-reference extracted client name against a known client list stored in a Sheets tab. If the detected "client" matches a team member name (Sheilagh, Dinah, Daniel, Viviene, Rae), flag it as a misparse.
4. Never use the Graph API `from` field as the client. Always derive client from body content, and validate against the known client list.

**Warning signs:**
- Bookings appearing with client name "Sheilagh" or "Dinah" or "Daniel"
- Multiple bookings with identical content logged from different senders
- Bookings logged with no brand or client but a forwarding note in the Notes field

**Phase to address:** Phase 2 (Email Parsing Core) — the pre-processing and prompt must be designed with forwarding in mind from day one, not retrofitted later.

---

### Pitfall 3: Over-Automating Before Validating Extraction Accuracy

**What goes wrong:**
The team builds the full pipeline — polling, Claude parsing, Google Sheets write, and AP auto-generation — before spending meaningful time validating that Claude's field extraction is accurate on real DM emails. When the pipeline is live and trusted, the team discovers that 30-40% of bookings have wrong dates, wrong shift counts, or the wrong brand. They've already decommissioned their manual process. Fixing upstream extraction errors now requires retroactively correcting dozens of live booking rows.

**Why it happens:**
The demo looks perfect on hand-picked examples. Developers move to integration immediately because the hard part (parsing) "works." The casual, varied format of DM's emails ("we need dry in GP around the 15th-ish") is deceptively difficult — Claude handles it well on average but has systematic blind spots on specific constructs that only appear in production.

**How to avoid:**
1. Before going live, run Claude against a batch of 20-30 real historical DM booking emails (redacted if needed) and compare extracted fields against what actually happened. Track a per-field accuracy score.
2. Set a minimum accuracy threshold per field before that field is trusted for automation: Client (>95%), Shift Type (>90%), Dates (>80%), Rate (>85%), Store Count (>70%).
3. For any field below threshold, mark it as "human-required" — Claude extracts but flags it explicitly as "unverified: [value]" in the sheet. Team confirms before it's trusted.
4. Do not trigger AP auto-generation until extraction accuracy has been validated on at least 50 real bookings over 2+ weeks in production review mode.
5. Build the pipeline in two phases: (a) "shadow mode" where Claude writes to a staging sheet and the team reviews every row before it moves to the live tracker, then (b) "live mode" after accuracy is established.

**Warning signs:**
- Team spots two booking errors in the same week
- Claude's confidence in a field is low but the value is still written without a flag
- "It works on the test emails" — if test emails were carefully chosen, not random

**Phase to address:** Phase 2 (Email Parsing) — build in shadow/review mode from the start. Phase 3 (Validation Gate) — explicit accuracy measurement before live mode is enabled.

---

### Pitfall 4: False Positive Booking Detection from Non-Booking Emails

**What goes wrong:**
The polling script flags emails as "booking-related" based on keywords (shifts, bookings, promoters, GP, dry). DM receives other emails using this language that are not bookings: rate card updates from clients ("here are our new dry shift rates for 2026"), promoter availability responses ("available for dry shifts in GP"), internal ops emails ("reminder: 3 dry shifts cancelled"), and newsletters from industry contacts. Claude processes all of these, sometimes creating phantom booking rows or updating existing bookings with wrong data from a follow-up email that's actually a cancellation.

**Why it happens:**
Keyword matching has no concept of intent. "We have 20 dry shifts in GP" in a rate card context looks identical to the same phrase in a booking request. The false positive rate compounds over time as email vocabulary normalises around booking language.

**How to avoid:**
1. Keyword matching should be a first-pass filter only — broad enough to catch real bookings. Claude must make the final classification ("Is this a booking request, a follow-up to an existing booking, or neither?") on every flagged email.
2. Include a "booking intent" check in Claude's prompt: "Does this email represent a new booking request, a follow-up to an existing booking, or is it non-booking? Only classify as 'new booking' if there is a clear request for promotional staffing services."
3. Add a confidence threshold: if Claude classifies an email as "booking" with low confidence, write it to a "Review Queue" tab, not directly to the Bookings Tracker.
4. Log every classification decision (email subject, sender, classification, confidence) to an audit tab. Review the first 100 classifications manually.
5. Never overwrite an existing booking row automatically. If Claude detects a follow-up email that matches an existing booking, append a note to that row's Notes column — never silently update Shift Count, Status, or Rate.

**Warning signs:**
- Non-booking emails appearing in the Inbox Queue
- Booking rows with clients that don't exist in DM's actual client base
- Existing booking rows having their data changed unexpectedly

**Phase to address:** Phase 2 (Email Parsing) — the classification prompt and confidence routing must be built before any data writes to the live tracker.

---

### Pitfall 5: Google Sheets Data Integrity When Claude and Humans Edit Simultaneously

**What goes wrong:**
The Bookings Tracker is edited by both Claude (via the Sheets API) and the team (via the browser UI). Race conditions cause lost writes. The most common scenario: Dinah opens the sheet, updates a booking row's Status to "Confirmed" and adds the store list in the Notes column. Simultaneously, Claude's polling cycle fires, reads that same row (with the old data), and writes an update — overwriting Dinah's store list addition or resetting the status. Dinah's work disappears. She re-enters it. This happens again. Trust in the system erodes.

**Why it happens:**
Google Sheets has no row-level locking. The Sheets API `batchUpdate` and `values.update` operations are last-write-wins. The API does not know Dinah has the sheet open. The polling script does not check if a row was recently modified before overwriting it.

**How to avoid:**
1. Claude should never overwrite cells that are team-editable unless explicitly allowed. Define column ownership strictly: Claude-owned columns (Client, Brand, Email Source, Date Received) vs. team-owned columns (Status, Store List, Assigned To, Notes, Confirmed Dates). Claude only writes to Claude-owned columns, never to team-owned columns.
2. For columns that both Claude and team may update (e.g., Shift Count): use append-only logic — Claude appends "Claude: 50 shifts" to the Notes column rather than writing to the Shift Count cell, and the team sets the authoritative Shift Count after reviewing.
3. Add a "Last Modified By" column with formula `=MODIFIED()` or update it via API after every write. Claude should check if "Last Modified By" contains a team member name and a timestamp within the last 5 minutes before writing to any row.
4. Use Google Sheets' built-in row protection: protect the Claude-owned columns from manual editing (with a warning, not a lock) and protect the team-owned columns from API writes by validating in code before any API call.
5. Never run Claude's write operation during the morning email rush hours (8-10am) without a debounce — this is when the team is most likely to be in the sheet.

**Warning signs:**
- Team members reporting their edits disappearing
- Status fields reverting to previous values
- Notes column losing content unexpectedly

**Phase to address:** Phase 2 (Sheets Integration) — the column ownership model must be designed before the first write operation is coded, not added as a fix after complaints.

---

### Pitfall 6: Missing Fields Treated as Empty Rather Than "Pending"

**What goes wrong:**
DM's clients routinely omit critical information: "we'll send the store list next week," "dates TBC," "same rate as usual." If Claude's extraction writes blank values to the Bookings Tracker for missing fields, two problems emerge: (a) the team doesn't know if the field is missing because Claude failed to extract it, or because the client genuinely hasn't provided it yet, and (b) Claude may re-parse the same email on a future cycle and again write blank, losing any manual corrections the team has made.

**Why it happens:**
NLP extraction returns null for a field that doesn't exist in the email. The simplest implementation writes null as an empty cell. Empty looks the same as "not yet collected."

**How to avoid:**
1. Distinguish three states for every extractable field: (a) "Claude extracted: [value]", (b) "Not in email — client hasn't provided", (c) "Claude unsure — verify". Write these states into the cell, not just the bare value. E.g., the Dates cell could contain "1-15 March 2026" (extracted), "Pending: client said next month" (vague but noted), or "?" (Claude couldn't find any date reference).
2. For fields marked "Pending" or "?", the Status column should automatically gate at "Details Pending" and not advance to "Confirmed" until all required fields are filled.
3. Add a "Completeness %" column that formulaically calculates how many required fields are filled. This gives Dinah an instant visual of what needs chasing.
4. On re-parse, Claude should only write to a field if it's currently empty or marked "?" — never overwrite a field that already has a human-entered value.

**Warning signs:**
- Bookings reaching "Confirmed" status with empty Dates or Store List fields
- Team manually filling fields that Claude then overwrites
- Team unable to distinguish "Claude couldn't find this" from "client hasn't sent it yet"

**Phase to address:** Phase 2 (Data Model Design) — the three-state field model must be in the schema from day one.

---

### Pitfall 7: Google Sheets API Rate Limits During Heavy Processing Cycles

**What goes wrong:**
On startup or after a backlog clears, Claude may process a batch of 20-30 emails from the Inbox Queue and attempt to write to the Bookings Tracker in rapid succession. Google Sheets API has a per-project quota of 300 read requests per minute and 300 write requests per minute (as of current published limits). At scale, batched writes can trigger `429 Too Many Requests` errors. The Python script or MCP server returns an error, Claude doesn't know the write failed, and the booking is lost.

**Why it happens:**
Developers test with 2-3 emails and never see rate limit errors. The API accepts requests fine in testing. In production, a backlog of 20 emails processed at once hits the limit. The MCP server may not handle 429 responses with proper retry-after logic.

**How to avoid:**
1. Batch all Sheets writes into a single `batchUpdate` call per processing cycle, not one API call per field or per row. This reduces API call volume by 10-20x.
2. After each batch write, insert a 1-2 second delay before the next polling cycle's writes.
3. Never write to the Sheets API more than once per 10 seconds from the polling script.
4. Add explicit 429 error handling: catch the error, wait for the `Retry-After` header duration (or default 60 seconds), then retry. Log the retry attempt.
5. Implement an in-memory queue in the polling script: accumulate all pending writes for 30 seconds, then flush as a single batch.

**Warning signs:**
- Bookings present in the Inbox Queue that don't appear in the Tracker
- Python script logs showing HTTP 429 errors
- Gaps in Booking ID sequence (IDs skipped when writes failed silently)

**Phase to address:** Phase 2 (Sheets Integration) — rate limit handling must be in the first write implementation, not added after hitting the limit in production.

---

### Pitfall 8: Attachment-Only Bookings Are Silently Skipped

**What goes wrong:**
Some clients send booking briefs as PDF or Word attachments with the email body containing only "Please see attached." The polling script flags the email as booking-related (keyword match on "booking brief" or subject line), but when Claude tries to parse it, the Graph API `message.body` field returns "Please see attached." — no booking data. Claude classifies it as non-actionable and skips it. A real booking is missed.

**Why it happens:**
Graph API returns the email body text. Attachment content is separate — `message.attachments` provides the file bytes. Reading an attached PDF requires additional API calls (`GET /messages/{id}/attachments/{attachmentId}/$value`) and PDF parsing logic. Most implementations wire up body parsing only.

**How to avoid:**
1. For every email in the Inbox Queue, check `hasAttachments: true` in the Graph API message object. If true and the body is sparse (less than 50 words), flag it as "attachment-required" in the queue.
2. For PDF and DOCX attachments: implement basic text extraction. Python can handle this with `pdfplumber` (PDF) or `python-docx` (DOCX). Feed the extracted text to Claude alongside the email body.
3. For attachment types Claude can't parse (images, Excel, custom formats): write a row to the Bookings Tracker with Status "Manual Review Required" and a note "Booking brief in attachment — please extract manually."
4. Never silently skip an email flagged as booking-related. Every flagged email must produce either a booking row or a review row.

**Warning signs:**
- A client follows up "did you get my brief?" and the booking is not in the tracker
- Inbox Queue shows emails processed but no corresponding tracker rows
- Spike in "Please see attached" emails with no booking rows created that day

**Phase to address:** Phase 2 (Email Parsing) — attachment detection must be designed alongside body parsing, not deferred.

---

### Pitfall 9: AP Auto-Generation Firing Before All Required Fields Are Present

**What goes wrong:**
The AP auto-generator is triggered when a booking's Status column changes to "Confirmed." A team member sets Status to "Confirmed" before all required AP fields are present — the store list is in an email somewhere, the rate is "same as usual" (not a number), or the dates span was entered as "March" without specific days. Claude fires the dm-activation-planner skill, which creates a partially populated AP sheet. The promoter team receives an AP with blank store lists and no specific dates, which is operationally useless and creates confusion.

**Why it happens:**
"Confirmed" to the team means "the client has confirmed they want the campaign" — not "all data is present." There's a semantic gap between operational confirmation and data completeness for AP generation.

**How to avoid:**
1. Separate "Confirmed" into two states: "Confirmed (Pending Data)" and "Confirmed (Ready for AP)." The team sets the former; Claude automatically advances to the latter only when the Completeness % column reaches 100% of required AP fields.
2. Define the required AP fields explicitly: Client (required), Brand (required), Shift Type (required), Base Rate (required — must be a number, not "same as usual"), Dates (required — must be specific dates, not "March"), Store List (required — must have at least one store or a note that store list is in an attached tab).
3. Validate these fields programmatically before triggering the AP skill. If validation fails, write a "AP Blocked: missing [field]" note to the Notes column and do not fire the trigger.
4. Never trigger AP generation within the same processing cycle that sets Confirmed status. Add a 15-minute delay to give the team a chance to catch premature confirmations.

**Warning signs:**
- APs created with blank store lists or "TBC" dates
- Team manually editing generated APs immediately after creation
- AP generation errors when the skill encounters null required fields

**Phase to address:** Phase 3 (AP Generation) — the validation gate must be designed before the trigger is wired up.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Keyword matching only for booking detection (no Claude classification) | Fast, no API cost | High false positive rate; phantom bookings accumulate | Never — add Claude classification from day one |
| Writing Claude-extracted values directly to final cells without flagging confidence | Simpler schema | Team can't distinguish verified vs. uncertain data; trust erodes | Never — implement the three-state model (extracted/pending/unsure) |
| Hardcoding email polling interval (e.g., every 15 min, fixed) | Simple implementation | Can't tune when email volume spikes; no backpressure | Acceptable for v1, but add config variable, not magic number |
| Skipping attachment parsing in v1 | Reduces scope | Misses attachment-only bookings silently | Acceptable only if "attachment-required" flag is implemented so nothing is lost |
| Single Sheets API key stored in plain .env file | Easy setup | Credential leak risk if repo is accidentally public | Acceptable locally, but must be in .gitignore from day one |
| Processing all queued emails in one batch on startup | Gets through backlog quickly | Hits API rate limits; may cause Sheets write failures | Never batch more than 10 emails per cycle without rate limit handling |

---

## Integration Gotchas

Common mistakes when connecting to these external services.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Microsoft Graph API | Using `user.read` scope only for token validation, forgetting `Mail.Read` | Always list required permissions explicitly in Azure App Registration and verify them in the token's decoded scopes |
| Microsoft Graph API | Fetching full email body for every poll cycle even for non-booking emails | Use `$select=id,subject,from,receivedDateTime,hasAttachments,bodyPreview` for the list call; only fetch full body for flagged emails |
| Microsoft Graph API | Not handling paginated results (`@odata.nextLink`) for large inboxes | Always follow `nextLink` in the polling script; assume results are paginated |
| Google Sheets API | Using service account but forgetting to share the Sheet with the service account email | The Sheet must be explicitly shared (with editor access) to the service account's `client_email` from the JSON key file |
| Google Sheets API | Treating row numbers as stable identifiers | Rows shift when someone inserts rows above. Use a Booking ID column as the stable key, and always look up rows by ID value, not by row number |
| Google Sheets API | Writing one cell at a time in a loop | Use `batchUpdate` or `values.batchUpdate` to write all cells in one API call |
| Composio (managed MCP) | Assuming token refresh is always handled | Composio has per-account usage limits on free tier; token expiry behaviour on their side is less documented — test the failure case explicitly |
| Claude NLP extraction | Sending the full raw email HTML to Claude | Strip HTML tags and inline CSS before sending to Claude; the noise degrades extraction quality |

---

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Processing all inbox emails every poll cycle | Slow cycles, re-processing old emails | Track last processed email timestamp; only fetch emails since last check | Breaks at ~50 unread emails in inbox |
| Reading the entire Bookings Tracker sheet to find matching rows | Slow Sheets reads, API quota consumed | Use a dedicated "ID Index" column and search with `MATCH()` formula or binary search in code | Breaks at ~500 rows in the tracker |
| Storing the Inbox Queue as a Sheets tab (read by Claude per session) | Grows unboundedly; slow reads | Purge processed items daily; keep queue under 100 rows | Breaks at ~200 queue rows |
| Claude processing emails sequentially in a single context window | Context fills up with email content | Process one email per Claude invocation, not batch; results returned to polling script | Breaks at ~10 long emails in one session |
| Logging every Graph API call to a Sheets tab | Useful for debugging; Sheets fills up fast | Use a local log file for verbose API logging; write only summary metrics to Sheets | Breaks at ~1000 log rows |

---

## Security Mistakes

Domain-specific security issues for this project.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Committing the Google service account JSON key to the GitHub repo | Anyone with repo access can write to all DM's Sheets | Add `*.json` (or specifically `service-account.json`) to `.gitignore` before creating the file |
| Storing Microsoft Graph Client Secret in plain text in the repo | Anyone with repo access can access DM's Outlook emails | Use environment variables or Windows Credential Manager; never commit secrets |
| Polling script running as Daniel's personal account token for shared team use | If Daniel revokes the app or changes password, the entire team's automation breaks | Use a dedicated service account or app-only auth (client credentials flow) for the polling script |
| Logging full email bodies to Google Sheets queue | Client data (rates, contact details, business information) stored in Sheets with team-wide access | Log only email ID, subject, sender, and a booking-detection flag in the queue; fetch full body only at processing time |
| Granting Mail.ReadWrite instead of Mail.Read to the Graph API app | App could modify or delete emails | Use Mail.Read only for v1; only upgrade to ReadWrite if specific write-back features are required |

---

## UX Pitfalls

How the team's daily experience with the tracker can fail.

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Booking rows appearing mid-sheet (not at top) as Claude inserts chronologically | Dinah must scroll to find new bookings | Always insert new bookings at row 2 (below header), pushing older bookings down; newest is always at top |
| Claude's extracted data in cells without any attribution | Team can't distinguish Claude-written from human-written data | Add a "Source" indicator (e.g., cell comment "Auto-extracted by Claude 2026-02-25 08:14") to Claude-populated cells |
| Status dropdown values not matching what Claude sets programmatically | Claude writes "Received" but the dropdown validation only allows "New" | Define status values in a config used by both the validation dropdown and the API write code — single source of truth |
| To-do lists regenerated every session without preserving completed items | Team marks tasks done; next session they reappear | To-do sheets should diff against prior state; completed items (checked off) are not re-added |
| AP links in the tracker pointing to a Sheet that requires re-sharing | Viviene/Rae click the link, get "Request access" | Auto-share generated AP sheets with all team member emails at creation time |

---

## "Looks Done But Isn't" Checklist

Things that appear complete in demo but are missing critical production pieces.

- [ ] **Token refresh:** OAuth flow works on day one — verify it still works after 90 days by testing with an expired refresh token before going live
- [ ] **Forwarded email parsing:** Demo emails are direct from client — verify with a real Sheilagh/Dinah forward chain before declaring parsing done
- [ ] **Attachment handling:** "Please see attached" emails result in a Review row, not silent skips — verify with a real PDF brief attachment
- [ ] **Column ownership:** Claude never overwrites team-edited cells — verify by manually editing a row and running a polling cycle; confirm the edit is preserved
- [ ] **Rate limit handling:** Polling script handles 429 responses with retry — verify by mocking a 429 response in testing
- [ ] **AP generation gate:** AP does not fire on Confirmed status with incomplete required fields — verify by confirming a booking with missing store list
- [ ] **Booking ID stability:** Row insertions above a booking do not change its ID or cause Claude to update the wrong row — verify by inserting a manual row above row 3 and running a cycle
- [ ] **Status gate for AP:** "Confirmed (Pending Data)" does not trigger AP — verify with a test booking

---

## Recovery Strategies

When pitfalls occur despite prevention.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| OAuth token refresh failure (emails missed) | MEDIUM | Re-authenticate manually via the OAuth flow; check Graph API audit log for missed emails in the time window; manually process any missed bookings |
| Forwarded email misparse (wrong client) | LOW | Team corrects the row manually; add the misparse pattern to the parsing prompt as a specific negative example |
| Data overwrite (team edit lost) | LOW-MEDIUM | Google Sheets version history (File > Version History) lets you see what was in a cell before the overwrite; restore manually |
| False positive bookings in tracker | LOW | Delete the phantom row; add the email sender/subject pattern to a block-list checked before queue insertion |
| AP generated with incomplete data | MEDIUM | Delete the bad AP Sheet; correct the booking row data; re-trigger AP generation; notify the promoter team the previous AP was invalid |
| Google Sheets API credential revoked | HIGH | Re-create service account key; re-share Sheet; redeploy; no data is lost (Sheets are unaffected), but automation is down until fixed |
| Rate limit cascade (batch of writes failed) | MEDIUM | Identify failed writes from audit log; manually re-process the missing emails from the Inbox Queue; implement batch limiting before re-enabling |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| OAuth token refresh failure | Phase 1 (Infrastructure) | Test auth failure scenario before Phase 2 begins |
| Forwarded email chain confusion | Phase 2 (Email Parsing) | Run against 10 real forwarded emails; check client attribution |
| Over-automating before validating accuracy | Phase 2-3 (Shadow Mode Gate) | 50-email validation batch before enabling live writes |
| False positive booking detection | Phase 2 (Classification Prompt) | Review 100 classification decisions in audit tab |
| Simultaneous edit data loss | Phase 2 (Sheets Schema) | Column ownership rules documented and enforced in code |
| Missing fields treated as empty | Phase 2 (Data Model) | Verify three-state model works for "Dates" field with vague email |
| Google Sheets API rate limits | Phase 2 (Sheets Integration) | Simulate 25-email batch and confirm no 429 errors |
| Attachment-only bookings skipped | Phase 2 (Email Parsing) | Test with "Please see attached" email; confirm Review row created |
| AP fires before data complete | Phase 3 (AP Integration) | Trigger Confirmed with missing Store List; confirm AP blocked |

---

## Sources

- Project context: `.planning/PROJECT.md` — DM email characteristics, data model, team structure
- Architecture context: `.planning/research/architecture-overview.md` — polling design, auth flow, Sheets column model
- Microsoft Graph API documentation (training data, HIGH confidence for token lifecycle and pagination behaviour)
- Google Sheets API documentation (training data, HIGH confidence for rate limits and service account auth)
- LLM automation pipeline failure patterns — MEDIUM confidence, based on well-documented patterns in the Claude and GPT automation community
- Forwarded email structure behaviour — MEDIUM confidence, recommend verifying with a real Sheilagh forward before Phase 2 begins

---
*Pitfalls research for: DM Ops Hub — Email-to-Booking Automation Pipeline*
*Researched: 2026-02-25*

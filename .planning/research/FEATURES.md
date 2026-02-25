# Feature Research

**Domain:** Email-to-booking automation pipeline (promoter marketing agency)
**Researched:** 2026-02-25
**Confidence:** HIGH (domain analysis from project documents) / MEDIUM (competitor patterns from training knowledge)

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features the team assumes will work on day one. Missing these = system feels broken and will be abandoned in favour of the existing email-and-memory approach.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Email reading from Outlook inbox | Without this, nothing works | LOW | Via Microsoft Graph API through Outlook MCP server |
| Booking request detection | The core trigger — distinguish booking emails from general correspondence | MEDIUM | LLM classification; needs to handle forwarded emails and email chains, not just fresh inbound |
| Field extraction from casual text | Project is explicit that emails are unstructured ("20 dry shifts in GP next month") | HIGH | Hardest accuracy problem; LLM approach handles variance better than regex |
| Missing field detection and flagging | Common case — clients routinely omit store lists, specific dates, rates | MEDIUM | Must flag clearly, not silently assume or skip |
| Write extracted booking to Google Sheets | The tracker is the team's daily operational surface | LOW-MEDIUM | CRUD via Google Sheets MCP; schema already designed |
| Booking status tracking (Received → Pending → Confirmed → AP Created) | Without a status, nobody knows what stage a booking is at | LOW | Enum column + timestamp; team must be able to update manually too |
| Per-employee to-do list generation | Named explicitly in PROJECT.md as a core daily workflow | MEDIUM | Requires email categorisation by intent (action, FYI, follow-up, booking) |
| Auto-generate Activation Plan on "Confirmed" | The final deliverable that replaces hours of manual AP creation | HIGH | Triggers dm-activation-planner skill; must only fire when all required fields present |
| Link AP back to bookings tracker | Users need to navigate from tracker row → AP document | LOW | Store Sheet URL in AP Link column |
| Idempotent processing (don't create duplicate rows) | Team will be confused and lose trust if the same email creates two booking rows | MEDIUM | Track processed email IDs; check before writing |

### Differentiators (Competitive Advantage)

Features that separate this system from a generic email-parsing tool and make it specifically powerful for Diversified Marketing's promoter agency workflow.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Client brand colour inference on AP headers | Matches the actual practice of Dinah/Viviene/Rae — APs look correct without manual formatting | MEDIUM | Requires a brand colour lookup table (client → hex); holding company rule for multi-brand (Astral, RFG, RCL) |
| Holding company resolution for multi-brand bookings | One email can cover multiple brands under one parent; AP branding must reflect the parent, not individual brand | LOW-MEDIUM | Lookup table: brand → holding company → colour; must be maintainable by team |
| Rate validation against known rate cards | dm-quoter skill exists; can flag if extracted rate deviates from standard rates before booking is confirmed | MEDIUM | Prevents under-quoting errors that cost the business money |
| Forwarded email chain awareness | Many requests arrive as "Dinah — fwd: client email" — the system must parse the embedded original, not the forwarding wrapper | MEDIUM | Thread/chain parsing; identify original sender vs forwarder; extract booking intent from nested content |
| Confidence scoring per extracted field | "I extracted 'GP' for province but I'm unsure if this means Gauteng only or includes surrounding areas" — lets team know where to validate | MEDIUM | Per-field confidence flag; surfaces as "Review" flag in Sheets row rather than silent acceptance |
| Partial-info booking intake (not reject-until-complete) | Many bookings start as one-liners; the system must accept partial data and track what's outstanding rather than requiring a complete submission | LOW | Status = "Pending" when fields missing; Notes column lists outstanding fields; team fills gaps over time |
| Smart assignment routing | Dinah handles 99% of instore ops; Viviene/Rae do APs; Daniel gets strategic items — to-do lists should route by role, not blast everyone | MEDIUM | Routing rules per email category + team role; must handle ambiguous cases gracefully |
| Inbox Queue pattern (polling script → Sheet → Claude) | Decouples continuous monitoring from Claude sessions; team doesn't need Claude open to have emails captured | MEDIUM | Python script runs on Windows Task Scheduler; writes raw flagged emails to queue sheet; Claude drains queue on session start |
| Deduplication across forwarded chains | Same booking may arrive as direct email to Daniel AND as a Dinah forward — must not create two rows | MEDIUM | Normalise on subject + client + approximate date; flag suspected duplicates for human review rather than silently dropping |
| Audit trail: email source link on every booking row | Team can always trace a booking back to the original email that created it | LOW | Store Outlook email ID/URL in Email Source column; immutable once set |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem like natural extensions but create disproportionate complexity, maintenance burden, or false confidence.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Sending emails on behalf of users (auto-reply to clients) | "Can it ask the client for the missing store list automatically?" | Sending in the voice of Daniel/Dinah risks brand damage if tone is wrong; clients may receive confusing automated responses; legal/consent considerations | Flag missing fields clearly in tracker; let Dinah send the follow-up as herself with full context — she can use the to-do item as her prompt |
| Full natural language chat interface for the tracker | "Can I just ask it what bookings are confirmed this week?" | Adds significant UI complexity; the Google Sheets tracker IS the interface — team already knows it | Keep Claude as a background processor; use Sheets formulas and filters for queries; team uses the Sheet natively |
| Real-time email push (webhook-based, event-driven) | "Can it process emails the instant they arrive?" | Microsoft Graph webhooks require a public HTTPS endpoint to receive callbacks — not available on a local machine without a tunnel service; adds infrastructure complexity | Polling every 15 minutes (Task Scheduler) provides near-real-time for this use case; upgrade to webhooks only if polling lag becomes a real problem (it won't be) |
| AI-generated follow-up email drafts in inbox | "Draft a reply asking for the store list" | Creates unreviewed content that can be accidentally sent; requires email-sending permissions; context drift between what Claude knows and what the client actually said | Generate the required information as a to-do item with full context so Dinah/Daniel can write the actual reply themselves |
| Auto-confirm bookings based on email content | "If the client says 'confirmed, let's go ahead' mark it as Confirmed automatically" | The word "confirmed" in email is ambiguous; client-side confirmation is not the same as DM-side operational readiness (store list, rates, dates must all be verified); false positives would trigger AP generation incorrectly | Keep status transitions human-driven; flag "client appears to have confirmed — review and update status" in to-do list |
| WhatsApp monitoring | "Most clients actually prefer to WhatsApp us" | WhatsApp Business API requires business verification, monthly cost, and a dedicated phone number; no MCP server exists for it; out of scope for v1 | Document as a v2 possibility if email volume proves insufficient; for now, team manually transcribes WhatsApp bookings into the tracker |
| Mobile app for the team | "Rae wants to check the tracker on her phone" | Google Sheets already works on mobile; building a separate app duplicates the tracker and fragments the source of truth | Ensure the Google Sheet is mobile-friendly (freeze header row, appropriate column widths); use Google Sheets mobile app |
| Client-facing booking portal | "Could clients submit bookings directly instead of emailing?" | Adds a public web surface, client onboarding, form design, and removes the casual email channel that clients prefer | Email remains the intake channel; the system makes email processing invisible to clients while structured internally |

---

## Feature Dependencies

```
[Email Reading via Outlook MCP]
    └──requires──> [Azure App Registration]
    └──enables──> [Booking Request Detection]
                      └──enables──> [Field Extraction from Casual Text]
                                        └──enables──> [Missing Field Detection]
                                        └──enables──> [Write to Google Sheets Tracker]
                                                          └──enables──> [Status Tracking]
                                                          └──enables──> [AP Auto-Generation]
                                                                            └──requires──> [All Required Fields Present]
                                                                            └──requires──> [Brand Colour Lookup Table]

[Polling Script (Python + Task Scheduler)]
    └──requires──> [Azure App Registration]
    └──enables──> [Inbox Queue Pattern]
                      └──enables──> [Continuous Email Monitoring without Claude session]

[Google Sheets MCP]
    └──requires──> [Google Cloud Project + Service Account]
    └──enables──> [Write to Bookings Tracker]
    └──enables──> [AP Auto-Generation output]
    └──enables──> [To-Do Sheets per employee]

[Per-Employee To-Do Lists]
    └──requires──> [Email Reading via Outlook MCP]
    └──requires──> [Email intent categorisation (booking / follow-up / FYI / action)]
    └──requires──> [Team role routing rules]

[Client Brand Colour on AP Headers]
    └──requires──> [AP Auto-Generation]
    └──requires──> [Brand colour lookup table (maintainable by team)]
    └──requires──> [Holding company resolution logic]

[Rate Validation via dm-quoter]
    └──enhances──> [Field Extraction] (validates extracted rate against rate cards)
    └──requires──> [dm-quoter skill access]

[Audit Trail: Email Source Link]
    └──requires──> [Email Reading] (to capture email ID at time of extraction)
    └──enhances──> [Bookings Tracker] (every row links back to source)

[Idempotency / Deduplication]
    └──requires──> [Write to Google Sheets] (must check before writing)
    └──prevents──> duplicate rows from forwarded chains or re-processed emails

[Confidence Scoring per Field]
    └──enhances──> [Field Extraction]
    └──enhances──> [Missing Field Detection]
    └──feeds into──> [Status determination] (LOW confidence fields → "Pending")
```

### Dependency Notes

- **AP Auto-Generation requires all required fields present:** The trigger for AP creation must check that client, brand, holding company, shift type, base rate, provinces, store list, and dates are all non-empty with human-verified status = "Confirmed". Firing on partial data would create broken APs.
- **Polling Script enables Inbox Queue pattern:** Without the script, email capture only happens when Claude is actively running. The script decouples capture from processing, which is critical for a team that receives emails throughout the day.
- **Brand Colour Lookup Table is a data dependency, not a code dependency:** The AP branding feature is blocked until someone (Daniel) provides the authoritative colour → client/brand mapping. This is a human task, not a technical one. Must be in the tracker or a config file.
- **Idempotency must precede any write operation:** If the system writes first and checks later, the damage (duplicate rows) is already done. Check-before-write is non-negotiable.
- **To-Do Lists conflict with Auto-Send:** If the system auto-sends emails, the to-do list items for "follow up on this booking" become confusing — did the system already follow up? Keep auto-send out of scope to preserve clarity.

---

## MVP Definition

### Launch With (v1)

Minimum viable product — what's needed for the team to abandon the current email-and-memory approach.

- [ ] Email reading from Outlook inbox via Outlook MCP — without this, nothing works
- [ ] Booking request detection (LLM classification of email intent) — core trigger
- [ ] Field extraction from casual/unstructured email text — core value
- [ ] Missing field detection with explicit flags — partial info is the normal case, not an edge case
- [ ] Write new booking row to Google Sheets tracker — team's operational surface
- [ ] Idempotent processing (no duplicate rows) — trust-critical from day one
- [ ] Booking status column with valid enum values — the team needs to see and update status
- [ ] Audit trail: email source link on every booking row — supports team verification
- [ ] Per-employee to-do list generation from morning emails — explicitly requested by Daniel as daily workflow
- [ ] Inbox Queue pattern (polling script + queue sheet) — decouples monitoring from Claude sessions

### Add After Validation (v1.x)

Features to add once core pipeline is stable and the team has validated the tracker.

- [ ] AP Auto-Generation trigger on "Confirmed" status — add once the team has been using the tracker for 1-2 weeks and the data model is stable; premature AP generation on bad data is worse than no generation
- [ ] Client brand colour on AP headers — requires brand colour lookup table to be populated first; add alongside AP generation
- [ ] Confidence scoring per extracted field — add when the team starts flagging "Claude got this wrong" consistently; gives them a signal for which fields to always check
- [ ] Holding company resolution for multi-brand APs — add alongside AP generation; required for correct branding
- [ ] Rate validation via dm-quoter — add when Claude-extracted rates start causing confusion; adds a safety net

### Future Consideration (v2+)

Features to defer until the v1 pipeline has proven its value.

- [ ] Forwarded email chain deep parsing — add when the team reports that forwarded requests are being missed or misread
- [ ] Deduplication across forwarded chains (heuristic matching) — add when duplicates become a visible problem; v1 deduplication covers exact email ID matches; heuristic matching adds complexity
- [ ] Smart assignment routing (role-based to-do list routing) — v1 to-do lists are categorised by intent; role-specific routing adds another layer; add when the team requests it
- [ ] Webhooks/event-driven email processing — only if 15-minute polling lag proves operationally painful; requires public HTTPS endpoint infrastructure
- [ ] Claude for Sheets add-on on the bookings tracker — useful team-facing enhancement; add when core pipeline is stable

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Email reading (Outlook MCP) | HIGH | LOW | P1 |
| Booking request detection | HIGH | MEDIUM | P1 |
| Field extraction from casual text | HIGH | HIGH | P1 |
| Missing field detection + flagging | HIGH | MEDIUM | P1 |
| Write to Google Sheets tracker | HIGH | LOW-MEDIUM | P1 |
| Idempotent processing | HIGH | MEDIUM | P1 |
| Status tracking (enum column) | HIGH | LOW | P1 |
| Audit trail (email source link) | MEDIUM | LOW | P1 |
| Per-employee to-do lists | HIGH | MEDIUM | P1 |
| Inbox Queue / polling script | HIGH | MEDIUM | P1 |
| AP auto-generation on Confirmed | HIGH | HIGH | P2 |
| Brand colour on AP headers | MEDIUM | MEDIUM | P2 |
| Holding company resolution | MEDIUM | LOW | P2 |
| Confidence scoring per field | MEDIUM | MEDIUM | P2 |
| Rate validation (dm-quoter) | MEDIUM | MEDIUM | P2 |
| Forwarded chain deep parsing | MEDIUM | HIGH | P3 |
| Cross-chain deduplication (heuristic) | LOW | HIGH | P3 |
| Smart role-based routing | MEDIUM | MEDIUM | P3 |
| Claude for Sheets add-on | LOW | LOW | P3 |
| Webhooks / event-driven processing | LOW | HIGH | P3 |

**Priority key:**
- P1: Must have for launch — without these, the system doesn't replace the current process
- P2: Should have — add after v1 is validated, unlocks the full end-to-end value promise
- P3: Nice to have — future consideration, add based on team feedback

---

## Competitor Feature Analysis

This system has no direct commercial competitors (it is bespoke for Diversified Marketing's workflow). However, the feature categories map onto patterns from comparable systems:

| Feature | Generic Email-to-CRM Tools (Zapier/Make) | Enterprise Booking Systems (Salesforce) | DM Ops Hub Approach |
|---------|------------------------------------------|----------------------------------------|---------------------|
| Field extraction from casual text | Template matching only; breaks on unstructured text | Structured form intake required | LLM-based extraction handles casual, varied formats natively |
| Missing field handling | Fail silently or reject submission | Block submission until form complete | Accept partial, flag missing, allow progressive completion |
| Status workflow | Rigid pipeline stages | Configurable but complex to maintain | Simple enum in Google Sheets column; team updates directly |
| Document generation | Generic templates, no brand awareness | Complex template engine, IT-managed | Brand colour lookup table, holding company logic, native Google Sheets output |
| Multi-employee task routing | Broadcast notifications to all | Role-based with complex permissions | Per-employee to-do lists generated from email content and role rules |
| Audit trail | Event log in separate system | Full activity history in CRM | Email source link on every booking row; team-readable in same Sheet |
| Integration with existing tools | API-based, requires developer setup | Native integrations with Salesforce ecosystem | Direct integration with dm-activation-planner, dm-quoter, dm-admin skills |

The core competitive insight: commercial tools require either structured intake forms (which clients won't fill out) or expensive enterprise configuration. The LLM-native approach handles casual email text that no template-based system can match.

---

## Sources

- Project context: `.planning/PROJECT.md` (HIGH confidence — primary source)
- Architecture context: `.planning/research/architecture-overview.md` (HIGH confidence — prior research)
- MCP tooling: `.planning/research/outlook-mcp-options.md`, `.planning/research/google-sheets-mcp-options.md` (HIGH confidence — prior research)
- Email automation patterns and competitor analysis: Training knowledge on email parsing, CRM intake workflows, document generation pipelines (MEDIUM confidence — architecture knowledge cutoff August 2025; core patterns are stable)
- Note: Web search was not available for this session. Core feature analysis is grounded in the project's own requirements documents, which provide high-fidelity domain context that outweighs generic web search for a bespoke system.

---

*Feature research for: Email-to-booking automation pipeline (Diversified Marketing)*
*Researched: 2026-02-25*

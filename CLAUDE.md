# SC Request Project

## What This Is
A redesign of the Salesforce SC (Solutions Consultant) request process used by Account Executives at GoTo. Replaces a manual Salesforce flow with a smarter system: auto-assigned SC, M365 calendar availability check, optional Outlook invite, and structured reporting record.

## Repository
- **GitHub:** https://github.com/oneofthegeeks/sc-request
- **Live site:** https://oneofthegeeks.github.io/sc-request/
- **GitHub Pages source:** `docs/` folder on `main` branch

## Key Documents
| Document | Path |
|----------|------|
| Design Spec | `docs/superpowers/specs/2026-03-16-sc-request-design.md` |
| Salesforce Implementation Plan (Option A) | `docs/superpowers/plans/2026-03-16-salesforce-option-a.md` |
| Presentation Site | `docs/index.html` + `docs/styles.css` + `docs/app.js` |

## Chosen Approach
- **Primary:** Option A — Salesforce Native LWC (fully planned, ready to execute)
- **Also specced:** Option C — Slack Workflow (Node.js backend, specced but not planned)

---

## Presentation Site
Built with plain HTML/CSS/JS. GoTo-branded (Figtree font, #FFE900 yellow, #0d0d0d dark sidebar).

**14 sections total, split in sidebar:**

*Stakeholder Presentation (8 sections):*
- The Problem, Proposed Solution, Request Flow, Calendar Integration, Data Model, Email Redesign, Implementation Options, Next Steps

*Technical Reference (6 sections):*
- Architecture, Field Reference, SC Selection Logic, M365 Integration, Scope & Boundaries, Implementation Plan

---

## Salesforce Objects
| Object | Type | Purpose |
|--------|------|---------|
| `SC_Request__c` | New Custom (Master-Detail Opportunity) | Full request record — reporting + history |
| `SC_Pairing__c` | New Custom | AE→SC assignments — replaces spreadsheet |
| `Opportunity` | Modified | Add `SC_Request_Notes__c` field + SC_Request__c related list |
| `Event` | Standard | Created on submit, linked to Opportunity |
| `OpportunityTeamMember` | Standard | SC added as "Solution Consultant" (Read/Write) |

## Opportunity Fields Auto-Updated on Submit
- `Primary_Solution_Consultant__c` — SC selected
- `SC_Date_Requested__c` — meeting date/time
- `SC_On_Team__c` — set to true
- `Demo__c` — set to true only if Request Type = "Demo Request"
- `SC_Request_Notes__c` — AE's freeform details (NEW field, Sales Engineering Info section)
- **NOT touched:** `Pre_Sales_Stage__c`, `SC_Notes__c` (SC-owned)

## Picklists
**Request Type (14):** Biz Dev/Channel, CSM Activity, Demo Request, Discovery Call, Document Request, On-Site Meeting, Pro Serv Scoping, RFX/Tender, Security Questionnaire, Security Team Escalation, Strategic Planning, Technical Call, Training, Trial Engagement/POC

**Product Line (17):** Central, EnhancedAudio, GoTo Resolve, GoToAssist, GoToConnect, GoToMeeting, GoToMyPC, GoToRoom, GoToTraining, GoToWebcast, GoToWebinar, ITSG, MDM, None, Pro, Rescue, UCC

---

## Technical Architecture (Option A)
| Component | Type | Purpose |
|-----------|------|---------|
| `scRequestModal` | LWC | 3-step modal: SC+calendar → Request details → Review+submit |
| `scRequestAction` | LWC (wrapper) | Quick Action entry point; calls `notifyRecordUpdateAvailable` on close |
| `ScRequestController` | Apex | All DML in single transaction with Savepoint/rollback |
| `M365CalendarService` | Apex | M365 free/busy query + optional Outlook invite creation |
| `M365_Graph_API` | Named Credential | NamedUser principal type — per-user OAuth onboarding |
| SC Request Email Flow | Record-Triggered Flow | Fires on SC_Request__c insert; 4 email alerts |
| `TestDataFactory` | Apex test helper | Creates Users, Opps, pairings for all test classes |
| `M365CalendarMock` | Apex test mock | HttpCalloutMock for M365CalendarService tests |

## Key Implementation Notes
- Entry point: existing "Request SC" Quick Action in Opportunity action bar overflow menu (unchanged)
- `@salesforce/user/Id` used in LWC for current user ID (NOT `recordId`)
- `notifyRecordUpdateAvailable` (not eval) for Opportunity page refresh on modal close
- Savepoint/rollback pattern for all 5 DML operations in ScRequestController
- Duplicate SC pairing: use `CreatedDate DESC`, surface warning, email SC's manager
- SC field: one SC → many AEs is normal; one AE → only one active SC (validation rule)

## M365 Integration
- **Read endpoint:** `POST /v1.0/me/calendarView/getSchedule` — free/busy for AE + SC
- **Write endpoint:** `POST /v1.0/me/events` — create Outlook invite
- **Scopes:** `Calendars.Read` (free/busy) + `Calendars.ReadWrite` (invite creation)
- **Auth:** NamedUser Named Credential — each user authenticates once via OAuth onboarding flow
- **Business hours:** 8am–6pm org timezone, Mon–Fri only
- **Default view:** Next 5 mutual open slots; date picker for specific dates
- **Fallback:** Manual date/time entry always available if API unavailable

## Outlook Invite Body
- Invite body is **HTML** (`contentType: "HTML"`) — renders formatted in Outlook, Teams, mobile
- Contains same fields as the email notification: Opportunity name (linked to SFDC), Account, Primary Contact, Meeting Date & Time, Request Type, Product Line, Assigned SC, AE freeform notes, "View Opportunity" button
- Both AE and SC are added as `required` attendees on the M365 event
- Primary contact fetched via `OpportunityContactRole WHERE IsPrimary = true LIMIT 1` — best-effort, shows "—" if none
- `createCalendarEvent` signature takes: scUserId, aeUserId, subject, notes, startDt, endDt, oppName, accountName, aeName, scName, requestType, productLine, primaryContact, opportunityId

## Email Template
- HTML, GoTo brand blue (`#0070d2`) header
- Fields: Opportunity (link), Primary Contact, Request Type, Product Line, Assigned SC, Meeting Date+Time (actual time — not "See Meeting Invite")
- Amber highlighted notes section
- 3 CTA buttons: View Opportunity, View SC Request Record, View Event (all SFDC deep links)
- Recipient footer: AE, SC, AE Manager (`AE.ManagerId`), SC Manager (`SC.ManagerId`)
- Delivery: Record-Triggered Flow on SC_Request__c insert
- **Excluded:** Secondary Product, Requester location

## Implementation Plan — 4 Chunks, 10 Tasks
- **Chunk 1:** SC_Pairing__c + SC_Request__c objects + SC_Request_Notes__c on Opportunity
- **Chunk 2:** ScRequestController + M365CalendarService + test helpers
- **Chunk 3:** scRequestModal LWC + scRequestAction wrapper
- **Chunk 4:** Email Flow + Named Credential + Opportunity layout updates

## Open Questions
- **M365 OAuth (IT):** Confirm whether existing SSO/App Registration covers `Calendars.Read` + `Calendars.ReadWrite`, or if a new App Registration is needed. **This is the only remaining blocker before development can begin.**

## Out of Scope
- Customer-facing calendar invites (AE handles separately)
- `Pre_Sales_Stage__c` — not touched
- `SC_Notes__c` — SC-owned, not overwritten
- Cross-timezone calendar display (future iteration)
- SC confirmation/acceptance workflow (future)
- Slack Option C implementation plan (specced but not yet planned)

# SC Request System — Design Spec
**Date:** 2026-03-16
**Status:** Draft — Pending User Review
**Recommended Option:** A (Salesforce Native)
**Also Specced:** Option C (Slack Workflow)

---

## Background

Account Executives (AEs) currently request Solutions Consultant (SC) involvement on opportunities via a manual process in Salesforce. The AE clicks a "Request SC" button on the Opportunity action bar, which triggers a flow that collects meeting details, notifies stakeholders via email, creates a calendar event on the opportunity, and adds the SC to the opportunity team.

**Key pain points with the current process:**
- AE→SC pairing data lives outside Salesforce (a spreadsheet maintained by SC managers), requiring AEs to manually select from a full SC list every time
- Calendar scheduling is a separate step — AE must secure a time on the SC's calendar independently before or after submitting the SFDC request, with no coordination between the two steps
- No structured reporting record of SC requests beyond the Event and Activity timeline

---

## Goals

1. Preserve all functionality of the current SC request process
2. Auto-populate the assigned SC based on AE→SC pairing data (bring pairing into Salesforce)
3. Integrate M365/Outlook calendar availability into the request flow so AEs can schedule on the spot while on the phone with a customer
4. Create a structured `SC_Request__c` record for reporting and opportunity history
5. Maintain the familiar "Request SC" entry point in the existing Opportunity action bar

---

## Shared Foundation (Both Options)

### Salesforce Objects

#### New: `SC_Request__c` (Custom Object)
Stores the full record of every SC request for reporting, history, and display on the Opportunity.

| Field | Type | Description |
|-------|------|-------------|
| `Opportunity__c` | Lookup(Opportunity) | Parent opportunity |
| `Requested_SC__c` | Lookup(User) | SC selected for the request |
| `Request_Type__c` | Picklist | Demo, Discovery, etc. |
| `Product_Line__c` | Picklist | GoTo Connect, Resolve, etc. |
| `Request_Notes__c` | Long Text Area | AE's freeform details (current state, problem to solve, demo requirements) |
| `Meeting_Date__c` | DateTime | Selected meeting date and time |
| `Status__c` | Picklist | Submitted, Confirmed, Completed, Cancelled |
| `Submitted_By__c` | Lookup(User) | AE who submitted the request |
| `Submitted_Date__c` | Formula (DateTime) | Formula field using `CreatedDate` — no Apex required, always accurate |

#### New: `SC_Pairing__c` (Custom Object)
Stores AE→SC assignments. Replaces the external spreadsheet. Maintained by SC managers via a Salesforce list view.

| Field | Type | Description |
|-------|------|-------------|
| `AE__c` | Lookup(User) | The Account Executive |
| `SC__c` | Lookup(User) | Their assigned Solutions Consultant |
| `Active__c` | Checkbox | Whether the pairing is current |

**Initial population:** One-time CSV import using Salesforce Data Loader or the standard Import Wizard. The SC manager exports the existing spreadsheet to CSV, maps `AE__c` (User lookup by email) and `SC__c` (User lookup by email), and imports via the admin Data Import Wizard. The development team will provide a field mapping guide. Acceptance criteria: all current active pairings exist in `SC_Pairing__c` with `Active__c = true` before go-live.

#### Modified: `Opportunity`
New field added to Sales Engineering Info section:

| Field | Type | Description |
|-------|------|-------------|
| `SC_Request_Notes__c` | Long Text Area | NEW — AE's request details, auto-populated on SC request submission |

**Fields auto-populated on SC request submission (existing fields):**

| Field | Value Set |
|-------|-----------|
| `Primary_Solution_Consultant__c` | SC selected in request |
| `SC_Date_Requested__c` | Meeting date/time selected in request |
| `SC_On_Team__c` | Checked (true) |
| `Demo__c` | Checked (true) if Request Type = "Demo" |

**Fields NOT touched by this process:**
- `Pre_Sales_Stage__c` — used independently
- `SC_Notes__c` — SC-owned field, not overwritten

#### Standard: `Event`
Created on submission, linked to the Opportunity.

| Field | Value |
|-------|-------|
| Subject | `Request for [Request_Type__c] for [Account Name] w/ a Solutions Consultant` |
| Description | SC Request Notes (AE's freeform details) |
| StartDateTime | Meeting date/time selected |
| OwnerId | AE (submitter) |
| WhatId | Opportunity Id |
| Type | Matches Request Type selected |

#### Standard: `OpportunityTeamMember`
SC added to the Opportunity team on submission.

| Field | Value |
|-------|-------|
| UserId | Selected SC |
| TeamMemberRole | Solution Consultant |
| OpportunityAccessLevel | Read/Write |

---

### Picklists (Static — Dev Ticket to Update)

**Request Type**
- Biz Dev/Channel
- CSM Activity
- Demo Request
- Discovery Call
- Document Request
- On-Site Meeting
- Pro Serv Scoping
- RFX/Tender
- Security Questionnaire
- Security Team Escalation
- Strategic Planning
- Technical Call
- Training
- Trial Engagement/POC

**Product Line**
- Central
- EnhancedAudio
- GoTo Resolve
- GoToAssist
- GoToConnect
- GoToMeeting
- GoToMyPC
- GoToRoom
- GoToTraining
- GoToWebcast
- GoToWebinar
- ITSG
- MDM
- None
- Pro
- Rescue
- UCC

> **Note:** A dev ticket is required to add new values after deployment.

---

### SC Selection Logic

1. On load, query `SC_Pairing__c` for records where `AE__c = current user` and `Active__c = true`
2. If exactly one active pairing found → auto-populate SC field with the paired SC
3. If multiple active pairings found for the same AE (data integrity issue) → use the most recently created record (`CreatedDate DESC`), surface a warning to the AE: "Multiple active SC pairings found — please verify the SC selected is correct," and automatically send an email notification to the SC manager (resolved via the SC's `ManagerId`) to clean up the duplicate records.
4. If no pairing found → dropdown defaults to full SC list with no pre-selection
5. AE can override the auto-populated SC by selecting any active SC from the dropdown at any time

**Important:** One SC can be paired with many AEs — this is normal and expected. The constraint is only that one AE should not have more than one active SC pairing simultaneously.

**Data integrity rule:** `SC_Pairing__c` should have a validation rule enforcing that only one active pairing per AE can exist at a time (deactivate old before creating new).

---

### Calendar Availability (M365 Integration)

Queries Microsoft 365 free/busy API for both the AE and the selected SC simultaneously.

**Display behavior:**
- **Default view:** Show the next 5 open time slots that work for both AE and SC (upcoming business days, business hours only)
- **Date picker:** AE can select a specific date to see all open slots on that day
- **Duration options:** 30 min / 1 hour / 90 min (AE selects before slots are shown; default pre-selection: 1 hour)
- AE selects a slot → that date/time becomes the `Meeting_Date__c` for the request

**Notes:**
- Business hours default: 8am–6pm in the Salesforce org's configured timezone. Both AE and SC availability is evaluated against the same org timezone for simplicity. Cross-timezone handling deferred to a future iteration.
- If the selected SC is changed via override dropdown, calendar re-queries automatically

**Outlook Calendar Invite (Write):**
After the AE selects a time slot, a checkbox is presented in the form:
> ☑ "Send Outlook calendar invite to SC automatically"

- Default state: **checked** (opt-out model — most requests won't have a pre-existing invite)
- If checked on submit: the M365 integration creates an Outlook calendar event for both the AE and SC, with the meeting details and a link to the Salesforce Opportunity. The SC receives the invite in their Outlook inbox to accept or decline.
- If unchecked: no Outlook invite is sent. The AE is responsible for calendar coordination. The Salesforce Event record is still created regardless.
- This write action requires M365 OAuth scope: `Calendars.ReadWrite` (in addition to `Calendars.Read` for availability checks)

**Error / Fallback Behavior:**
- If the M365 API is unavailable or returns an error: display an inline error message ("Calendar availability is currently unavailable") and allow the AE to manually enter a date and time to proceed with submission
- If zero open slots are found in the next 5 business days: show the message "No mutual availability found in the next 5 days" and prompt the AE to use the date picker to check specific dates, or enter a time manually
- Manual date/time entry is always available as a fallback regardless of API status

---

### Notifications on Submit

Email sent to all four recipients containing full request details:

| Recipient | How Resolved |
|-----------|-------------|
| AE | Current user |
| SC | Selected SC from request |
| AE's Manager | `AE.ManagerId` (from SFDC User record) |
| SC's Manager | `SC.ManagerId` (from SFDC User record) |

**Email template:** HTML formatted email with the following structure:

- **Header banner:** "A New SC Request Has Been Submitted" (GoTo brand blue)
- **Submitted by line:** AE name + submission date/time
- **Two-column key details card:**
  - Opportunity name (link to Opportunity record in Salesforce)
  - Primary Contact (auto-pulled from Opportunity primary contact)
  - Request Type
  - Product Line
  - Assigned SC
  - Meeting Date & Time (actual time from calendar selection — not "See Meeting Invite")
- **Notes section:** AE's freeform request notes, visually highlighted
- **Three CTA buttons:** "View Opportunity →", "View SC Request Record →", "View Event →" (all deep links into Salesforce)
- **Recipient footer:** Lists all four recipients by name and role

**Fields intentionally excluded from email:** Requester location, Secondary Product (removed from process entirely).

**Email delivery mechanism:** Salesforce Record-Triggered Flow on `SC_Request__c` insert, using HTML email template.

---

## Option A — Salesforce Native (Recommended)

### Overview
A Lightning Web Component (LWC) replaces the current flow behind the existing "Request SC" button in the Opportunity action bar overflow menu. Clicking the button opens a Lightning modal with a multi-step form. An Apex controller handles all Salesforce writes on submission. M365 calendar is queried via a Named Credential and External Service (OAuth 2.0 Connected App).

### Entry Point
- **Location:** Existing "Request SC" button in the Opportunity action bar overflow menu (the `▼` dropdown at the right end of the action bar)
- **Label:** "Request SC" (unchanged — no retraining required)

### Form Flow (Lightning Modal)

**Step 1 — SC & Meeting Time**
- SC field: Auto-populated from `SC_Pairing__c`, editable dropdown override
- Duration: 30 min / 1 hour / 90 min (radio buttons)
- Calendar availability panel: Next 5 open slots shown automatically; date picker available to browse specific dates
- AE selects a time slot

**Step 2 — Request Details**
- Request Type: Picklist dropdown
- Product Line: Picklist dropdown
- Request Notes: Long text area (label: "What does the customer have today, what are they trying to solve, and what should the demo include?")

**Step 3 — Review & Submit**
- Summary of all selections
- Submit button

### Technical Components

| Component | Purpose |
|-----------|---------|
| LWC: `scRequestModal` | Multi-step modal form |
| Apex: `ScRequestController` | Handles all DML (SC_Request__c, Event, OpportunityTeamMember, Opportunity field updates) |
| Apex: `M365CalendarService` | Queries M365 free/busy via Named Credential |
| Connected App | OAuth 2.0 for M365 integration |
| Named Credential | Stores M365 auth token securely |
| Email Alert | Salesforce Flow (Record-Triggered) on `SC_Request__c` insert — preferred over Workflow Rules (deprecated) and Process Builder (deprecated). Flow sends 4 email alerts via Send Email action. |
| Custom Object: `SC_Request__c` | Request record |
| Custom Object: `SC_Pairing__c` | AE→SC pairing data |
| Custom Field: `SC_Request_Notes__c` on Opportunity | AE's request details |

### Opportunity Layout Changes
- Add `SC_Request_Notes__c` field to the Sales Engineering Info section
- Add `SC_Request__c` related list to the Related tab
- No other layout changes required (button stays in existing location)

### Pros
- AE never leaves Salesforce
- All data natively queryable and reportable in SFDC
- Manager relationships resolved automatically from User hierarchy
- SC_Pairing__c maintainable by SC managers without dev involvement

### Cons
- Higher build complexity (LWC + Apex + Connected App + Named Credentials)
- Slower iteration — all UI changes require Salesforce deployment
- M365 OAuth setup adds initial configuration overhead

---

## Option C — Slack Workflow (Alternative)

### Overview
A Slack app with a `/request-sc` slash command. The AE triggers it from any Slack channel or DM. A multi-step Slack modal collects all request information. A backend service (Node.js or Python) handles the Salesforce API writes (same objects and field updates as Option A) and queries M365 calendar. Calendar availability is presented as interactive Slack message buttons. Confirmation is sent via Slack DM and email.

### Entry Point
- **Trigger:** `/request-sc` slash command in any Slack channel or DM
- **Note:** AE must search for and select the Opportunity by name (type-ahead search against Salesforce API) — one additional step vs. Option A

### Form Flow (Slack Modal)

1. **Opportunity search** — type-ahead lookup against Salesforce Opportunities by name
2. **SC selection** — auto-populated from pairing data, override available
3. **Request Type & Product Line** — Slack dropdown selects
4. **Request Notes** — text input in modal
5. **Submit** → bot queries M365 and posts available time slots as interactive buttons in a follow-up DM
6. AE taps a time slot to confirm → all Salesforce writes fire

### Technical Components

| Component | Purpose |
|-----------|---------|
| Slack App | Hosts slash command and modal |
| Backend Service (Node.js with Express) | Orchestrates SFDC API + M365 API calls. Node.js selected for broad Slack Bolt SDK and jsforce (Salesforce) ecosystem support. Deployed as a lightweight hosted service (e.g., Railway, Heroku, or internal server). |
| Salesforce Connected App | OAuth for backend → SFDC API access |
| M365 App Registration | OAuth for backend → M365 calendar access |
| SFDC REST API | All object creates/updates (same as Option A) |

### Salesforce Writes (Identical to Option A)
All the same objects, fields, and automations fire — the Slack workflow is purely a different front-end. The backend service makes the same Salesforce API calls that the Apex controller makes in Option A.

### Pros
- Fast to build and iterate — no Salesforce deployment process
- AEs who live in Slack can trigger without opening a browser
- Easy to update and redeploy without touching Salesforce

### Cons
- AE must context-switch out of Salesforce to use it
- Opportunity lookup adds one extra step
- Requires maintaining a separate backend service and infrastructure
- Less visual calendar UI compared to Option A modal

---

## What's Out of Scope

- Customer-facing calendar invites (AE handles separately)
- Pre-Sales Stage field — not touched by this process
- SC Notes field — SC-owned, not overwritten
- SC confirmation/acceptance workflow (future consideration)
- Automatic calendar invite to the customer contact (AE handles customer-facing scheduling separately)
- Cross-timezone calendar display — availability slots are shown in the Salesforce org timezone for both AE and SC. If AE and SC are in different time zones, the AE is responsible for interpreting the time correctly. Full per-user timezone support is a future iteration.

---

## Open Questions for Stakeholders

> ✅ Items #1, #2, #3, and #6 are resolved.

1. ~~**Request Type full list**~~ ✅ Resolved — see Picklists section
2. ~~**Product Line full list**~~ ✅ Resolved — see Picklists section
3. ~~**OpportunityTeamMember access level**~~ ✅ Resolved — Read/Write confirmed
4. ~~**Email template**~~ ✅ Resolved — HTML formatted template defined, see Notifications section
5. **M365 OAuth ownership** — owned by internal IT team. SSO is already in place and may cover the required OAuth scopes (`Calendars.Read`, `Calendars.ReadWrite`). IT team to confirm whether an existing Connected App / App Registration can be extended or a new one is needed before development begins.
6. ~~**Duration of meeting default**~~ ✅ Resolved — 1 hour default confirmed

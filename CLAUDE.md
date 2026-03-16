# SC Request Project

## What This Is
A redesign of the Salesforce SC (Solutions Consultant) request process used by Account Executives at GoTo. The goal is to replace a manual Salesforce flow with a smarter system that auto-assigns SCs, integrates M365 calendar availability, and improves reporting.

## Key Documents
- **Design Spec:** `docs/superpowers/specs/2026-03-16-sc-request-design.md` — full requirements, data model, and design decisions
- **Presentation Site:** `docs/site/` — GitHub Pages site for stakeholder buy-in

## Chosen Approach
- **Primary:** Option A — Salesforce Native LWC (Lightning Web Component)
- **Also specced:** Option C — Slack Workflow (Node.js backend)

## Salesforce Objects
| Object | Type | Purpose |
|--------|------|---------|
| `SC_Request__c` | New Custom | Full request record for reporting + history |
| `SC_Pairing__c` | New Custom | AE→SC assignments (replaces spreadsheet) |
| `Opportunity` | Modified | Add `SC_Request_Notes__c` field + related list |
| `Event` | Standard | Created on submit, linked to Opportunity |
| `OpportunityTeamMember` | Standard | SC added as Solution Consultant (Read/Write) |

## Opportunity Fields Auto-Updated on Submit
- `Primary_Solution_Consultant__c` — SC selected
- `SC_Date_Requested__c` — meeting date/time
- `SC_On_Team__c` — checked
- `Demo__c` — checked if Request Type = Demo Request
- `SC_Request_Notes__c` — AE's freeform details (NEW field)
- **NOT touched:** `Pre_Sales_Stage__c`, `SC_Notes__c` (SC-owned)

## Picklists
**Request Type:** Biz Dev/Channel, CSM Activity, Demo Request, Discovery Call, Document Request, On-Site Meeting, Pro Serv Scoping, RFX/Tender, Security Questionnaire, Security Team Escalation, Strategic Planning, Technical Call, Training, Trial Engagement/POC

**Product Line:** Central, EnhancedAudio, GoTo Resolve, GoToAssist, GoToConnect, GoToMeeting, GoToMyPC, GoToRoom, GoToTraining, GoToWebcast, GoToWebinar, ITSG, MDM, None, Pro, Rescue, UCC

## Key Design Decisions
- "Request SC" button stays in existing Opportunity action bar overflow menu
- SC auto-populated from `SC_Pairing__c`; AE can override from full SC dropdown
- One SC → many AEs is normal; one AE → only one active SC (enforced by validation rule)
- Calendar: next 5 open slots + date picker; 1-hour default; org timezone; fallback to manual entry
- Outlook calendar invite: optional checkbox, checked by default (opt-out model)
- Event title format: `Request for [Type] for [Account Name] w/ a Solutions Consultant`
- Email: HTML formatted, 3 Salesforce deep links, actual meeting time (not "See Meeting Invite")
- Email recipients: AE, SC, AE Manager (via User.ManagerId), SC Manager (via User.ManagerId)
- Email delivery: Record-Triggered Flow on `SC_Request__c` insert
- M365 OAuth: IT-owned, SSO in place — confirm Calendars.Read + Calendars.ReadWrite scopes

## Out of Scope
- Customer-facing calendar invites
- Pre_Sales_Stage__c field
- SC Notes field (SC-owned)
- Cross-timezone calendar display (future)
- SC confirmation/acceptance workflow (future)
- Customer contact scheduling (AE handles separately)

## GitHub Pages Site
- Source: `docs/site/`
- Purpose: Stakeholder presentation for buy-in
- Sections mirror the spec: Problem, Proposed Solution, Flow Walkthrough, Data Model, Email Design, Implementation Options, Open Questions

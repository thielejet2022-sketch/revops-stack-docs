# RevOps Stack Overview
**Organization:** Curve Dental (Battery Ventures Portfolio)
**Function:** Revenue Operations — Build from Scratch
**ARR at Documentation:** ~$87M (three-brand structure)
**Reporting Line:** CFO
**Last Updated:** 2026-05-08
**Owner:** Jim Thiele, Director of Revenue Operations

---

## Table of Contents
1. [Stack Architecture Summary](#stack-architecture-summary)
2. [Salesforce (CRM)](#salesforce-crm)
3. [HubSpot (Marketing Automation)](#hubspot-marketing-automation)
4. [Gong (Conversation Intelligence)](#gong-conversation-intelligence)
5. [Chili Piper (Scheduling & Routing)](#chili-piper-scheduling--routing)
6. [Clay (Lead Enrichment)](#clay-lead-enrichment)
7. [Power BI (Reporting & Analytics)](#power-bi-reporting--analytics)
8. [Commission Automation](#commission-automation)
9. [Integration Map](#integration-map)
10. [Known Dependencies & Risks](#known-dependencies--risks)
11. [Change Log](#change-log)

---

## Stack Architecture Summary

```
[Clay]
  └─► Lead Enrichment → [HubSpot]
                             └─► MQL Routing → [Chili Piper]
                                                    └─► Booked Meeting → [Salesforce]
                                                                              ├─► [Gong] (call recording/scoring)
                                                                              ├─► [Power BI] (pipeline & forecast reporting)
                                                                              └─► [Commission Engine] (attainment calculation)
```

**Design principle:** HubSpot owns top-of-funnel and lead lifecycle. Salesforce owns opportunity and revenue lifecycle. Data flows one direction (HubSpot → Salesforce) to prevent sync conflicts. Power BI reads from Salesforce as its single source of truth.

---

## Salesforce (CRM)

### Purpose
Primary system of record for pipeline, opportunities, accounts, contacts, and closed/won revenue. Replaced a fragmented multi-tool approach across three acquired brands.

### Key Configurations

**Lead Routing**
- Assignment rules fire on `Lead Source`, `Company Size`, and `Dental Brand` (Curve Dental, Apteryx, RevenueWell)
- Routing logic: SMB (<10 seats) → round-robin SDR queue; Mid-Market (10–50) → AE direct assign; Enterprise (50+) → named AE
- Overflow rule: unworked leads >48hrs auto-reassign to queue manager

**Stage Gating**
| Stage | Gate Criteria | Required Fields |
|---|---|---|
| Discovery | Meeting held + notes logged | `Meeting_Date__c`, `Discovery_Notes__c` |
| Qualify | MEDDIC score ≥ 3/6 | `Pain_Identified__c`, `Decision_Process__c` |
| Propose | Demo completed + quote sent | `Demo_Date__c`, `Quote_ID__c` |
| Negotiate | Legal/procurement engaged | `Contract_Start__c`, `Stakeholder_Map__c` |
| Closed Won | MSA signed + activation date set | `Close_Date`, `Activation_Date__c` |

**Custom Objects**
- `Deal_Timing__c` — tracks forecast category changes over time (source for Power BI waterfall chart)
- `Churn_Signal__c` — CS-logged indicators linked to Account; feeds churn reduction workflow
- `Acquisition_Account__c` — Dental HQ acquisition mapping object; links legacy IDs to Salesforce Account IDs

**Eliminated:** Salesforce CPQ — removed due to complexity-to-value ratio for dental SaaS deal structure. Replaced with a simplified quoting process using Google Docs templates + DocuSign, with deal terms captured in custom Opportunity fields.

**Critical Notes**
- Three-brand architecture uses `Brand__c` picklist on Account and Opportunity — **do not delete or rename picklist values** without auditing all reports, workflows, and Power BI queries that filter on this field
- Fiscal year set to January start — matches Battery Ventures reporting calendar
- Territory hierarchy uses `Region__c` → `Sub-Region__c` → `Rep_Name__c` (not standard Salesforce territory management)

---

## HubSpot (Marketing Automation)

### Purpose
Owns all inbound lead capture, nurture sequences, MQL scoring, and hand-off to Salesforce. Built from scratch after consolidating three legacy brand email tools.

### Key Configurations

**Lead Scoring Model**
Weighted scoring across two dimensions:

*Fit Score (0–50 points)*
| Attribute | Points |
|---|---|
| Practice size 5–20 seats | 20 |
| Practice size 20+ seats | 30 |
| Decision-maker title (Owner/CEO/COO) | 15 |
| Target geography (US/Canada) | 10 |
| Existing competitor (Dentrix/Eaglesoft) | 5 |

*Engagement Score (0–50 points)*
| Action | Points |
|---|---|
| Demo page visit | 20 |
| Pricing page visit | 15 |
| Email click (last 30 days) | 10 |
| Webinar attendance | 10 |
| Free trial start | 25 |

**MQL Threshold:** Combined score ≥ 60 triggers MQL status and Chili Piper routing

**Lifecycle Stages**
`Subscriber → Lead → MQL → SQL → Opportunity → Customer`

SQL assignment happens in Salesforce, not HubSpot — HubSpot only manages through MQL.

**Active Sequences**
- `Inbound-Demo-Request` — 5-touch, 10 days, triggers on demo form submit
- `Free-Trial-Nurture` — 8-touch, 21 days, triggers on trial activation
- `Re-Engage-Cold-Lead` — 3-touch, 14 days, triggers on leads MQL > 90 days ago with no activity

**Salesforce Sync**
- Bi-directional sync via native HubSpot-Salesforce connector
- Contact sync: HubSpot Contact ↔ Salesforce Contact/Lead
- **One-way only:** Opportunity data flows Salesforce → HubSpot (read-only in HubSpot)
- Sync field exclusions: `Commission__c`, `Deal_Timing__c` — excluded to prevent overwrite conflicts

**Critical Notes**
- Brand segmentation uses HubSpot `Brand` property — must match Salesforce `Brand__c` picklist values exactly or sync breaks
- Do not create duplicate Contact records in HubSpot; deduplication relies on `Email` as unique identifier

---

## Gong (Conversation Intelligence)

### Purpose
Call recording, transcription, and scoring for all AE and SDR calls. Primary source for rep coaching data, win/loss analysis, and discovery quality scoring.

### Key Configurations

**Salesforce Integration**
- Gong calls automatically linked to Salesforce Opportunities via email/calendar match
- `Gong_Call_Count__c` (custom field on Opportunity) — populated via Gong API → Salesforce flow
- Used in Power BI to correlate call volume with win rate by stage

**Scorecards**
Three active scorecards:
1. `Discovery Quality` — 8 criteria, scored by managers weekly on sample basis
2. `Demo Effectiveness` — 6 criteria, auto-flagged for review when deal goes dark post-demo
3. `Objection Handling` — 5 criteria, triggered on any call containing competitor mentions

**Talk Track Tracking**
Keyword trackers active for:
- Competitor names: Dentrix, Eaglesoft, Carestream, Curve (self-mention monitoring)
- Pricing objections: "too expensive," "budget," "cost," "cheaper"
- Champion language: "I love," "this is exactly," "when can we start"

**Critical Notes**
- Gong workspace is unified across all three brands — call library is not brand-segmented
- Manager review assignments are manual (no auto-assignment built); requires weekly ops calendar reminder

---

## Chili Piper (Scheduling & Routing)

### Purpose
Inbound meeting scheduling and lead-to-rep routing at the MQL handoff point. Replaced a manual SDR-to-calendar process that created 4–6 hour response lag on inbound demo requests.

### Key Configurations

**Routing Rules (priority order)**
1. Account owner match — if inbound lead email domain matches existing Salesforce Account, route to Account owner
2. Brand match — route to rep queue aligned to `Brand__c` value
3. Geographic round-robin — fallback for unmatched leads, rotates by time zone coverage

**Form Connections**
- HubSpot demo request form → Chili Piper router → AE calendar
- HubSpot free trial form → Chili Piper router → SDR calendar (for onboarding call)
- Direct website booking widget → same routing logic as demo request form

**Meeting Outcomes**
Meeting held triggers:
- Salesforce task creation (auto)
- HubSpot sequence pause (auto)
- Gong call link (auto via calendar match)

**Critical Notes**
- Chili Piper fires on MQL threshold crossing in HubSpot — if HubSpot scoring model changes, re-validate trigger
- AE capacity caps are manually managed in Chili Piper — update when headcount changes
- Holiday/OOO overrides require manual configuration; no auto-sync with Google Calendar OOO status

---

## Clay (Lead Enrichment)

### Purpose
Automated lead enrichment for inbound MQLs and outbound prospecting lists. Replaced manual SDR research process that consumed ~6 hours/week per rep.

### Key Configurations

**Enrichment Waterfall (in order)**
1. Clearbit — company firmographics, tech stack, funding
2. LinkedIn (via Clay scraper) — title, tenure, seniority
3. Apollo — phone number, email verification
4. Fallback: manual flag for SDR review if confidence score < 70%

**Trigger Points**
- New HubSpot MQL → Clay enrichment flow → enriched data written back to HubSpot Contact properties
- Weekly outbound list pull → Clay table → enriched CSV → imported to Salesforce campaign

**Key Enrichment Fields Written Back to HubSpot**
| Clay Field | HubSpot Property | Notes |
|---|---|---|
| Company headcount | `Number of Employees` | Used in lead scoring fit dimension |
| Tech stack (Dentrix/Eaglesoft detected) | `Current_Software__c` | Triggers competitor sequence |
| Funding stage | `Funding_Stage__c` | Used for enterprise segmentation |
| LinkedIn title | `Job Title` | Overrides only if HubSpot field is blank |

**Critical Notes**
- Clay API credits are consumption-based — monitor monthly usage; enrichment costs spike during outbound campaign pushes
- LinkedIn scraping is subject to rate limits; batch enrichment runs are scheduled overnight

---

## Power BI (Reporting & Analytics)

### Purpose
Single source of truth for all pipeline, forecast, and performance reporting. Replaced four separate brand-level spreadsheet trackers that produced conflicting ARR numbers.

### Key Configurations

**Data Source**
- Primary: Salesforce (direct connector, refreshes every 4 hours)
- Secondary: HubSpot (CSV export, manual refresh weekly — candidate for automation)
- Commission data: Google Sheets (manual upload monthly)

**Core Dashboards**

| Dashboard | Primary Audience | Refresh Cadence |
|---|---|---|
| Pipeline Health | CRO, AEs | 4-hour auto |
| Forecast by Brand | CFO, VP Sales | 4-hour auto |
| Deal Velocity | RevOps, Sales Mgmt | 4-hour auto |
| SDR Activity | SDR Manager | 4-hour auto |
| Churn Signal Monitor | CS, CFO | Daily |
| Commission Attainment | Finance, AEs | Monthly (manual) |

**Forecast Model Logic**
- Weighted pipeline: `Amount × Stage_Probability__c`
- Stage probabilities are custom (not Salesforce defaults): Discovery 10%, Qualify 25%, Propose 50%, Negotiate 75%, Commit (custom stage) 90%
- "Commit" stage = rep-indicated best case; used for CFO weekly call
- Waterfall chart pulls from `Deal_Timing__c` history object — requires this object to be populated correctly at every stage change

**Critical Notes**
- All three brands share one Power BI workspace — brand filter is applied at report level via `Brand__c` slicer; removing this slicer shows blended data
- HubSpot → Power BI connection is the weakest link in the stack; full automation via direct connector is a pending project

---

## Commission Automation

### Purpose
Automated attainment calculation replacing a manual Excel-based process that produced errors and required 3–5 days to close each month.

### Architecture
`Salesforce (Closed Won data)` → `Google Apps Script` → `Google Sheets (Commission Model)` → `Finance review` → `Payroll submission`

### Logic
- Attainment calculated on `Close_Date` within calendar month
- Quota sourced from `Quota__c` field on User object in Salesforce
- Accelerators trigger at 100% and 125% attainment (rate defined in Sheets model, not in script)
- Multi-brand reps: attainment calculated per-brand, then blended for payout

### Known Limitations
- SPIFFs and manual adjustments still entered by hand in Google Sheets
- No real-time visibility for reps — self-service attainment tracking is a pending project

---

## Integration Map

| Source | Destination | Method | Direction | Cadence |
|---|---|---|---|---|
| HubSpot | Salesforce | Native connector | Bi-directional (Opp read-only) | Real-time |
| Salesforce | Gong | Native connector | Bi-directional | Real-time |
| HubSpot | Chili Piper | Native connector | One-way trigger | Event-based |
| Clay | HubSpot | Clay webhook | One-way write | Event-based |
| Salesforce | Power BI | Native connector | One-way read | 4-hour refresh |
| Salesforce | Google Apps Script | API | One-way read | Monthly |
| Google Sheets | Power BI | Manual CSV | One-way | Monthly |

---

## Known Dependencies & Risks

| Risk | Affected Systems | Severity | Notes |
|---|---|---|---|
| `Brand__c` picklist value rename | Salesforce, HubSpot, Power BI | 🔴 Critical | Breaks sync, routing, and all brand-level reports simultaneously |
| HubSpot scoring model change | Chili Piper routing | 🔴 Critical | MQL threshold change will break routing trigger |
| Clay API credit overrun | HubSpot enrichment | 🟡 Medium | Enrichment silently fails; leads enter pipeline unenriched |
| Gong-Salesforce Opp link failure | Power BI call volume reports | 🟡 Medium | Call count field stops populating; affects coaching data |
| Google Apps Script auth expiry | Commission automation | 🟡 Medium | Script fails silently; commission calc reverts to manual |
| HubSpot → Power BI manual refresh lag | Forecast reporting | 🟢 Low | Top-of-funnel data stale by up to 7 days |

---

## Change Log

| Date | System | Change | Owner |
|---|---|---|---|
| 2024-Q1 | Salesforce | Eliminated CPQ; migrated to custom quote fields | J. Thiele |
| 2024-Q1 | Power BI | Built unified three-brand forecast dashboard from scratch | J. Thiele |
| 2024-Q2 | HubSpot | Rebuilt lead scoring model; added competitor tech stack dimension | J. Thiele |
| 2024-Q2 | Clay | Implemented enrichment waterfall; replaced manual SDR research | J. Thiele |
| 2024-Q3 | Chili Piper | Deployed brand-based routing rules; reduced demo response lag from 4–6hrs to <15min | J. Thiele |
| 2024-Q3 | Gong | Deployed three scorecard templates; built Salesforce call count integration | J. Thiele |
| 2024-Q4 | Commission | Automated Google Apps Script attainment calc; replaced manual Excel process | J. Thiele |
| 2024-Q4 | Salesforce | Built `Deal_Timing__c` history object for Power BI waterfall chart | J. Thiele |

---

*This document is intended as a living reference. Update the Change Log and relevant system sections whenever configuration changes are made. Feed this file as context to any AI assistant working on RevOps troubleshooting or system design tasks.*

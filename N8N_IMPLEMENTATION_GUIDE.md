# Glamour Styling — N8N Workflow Implementation Guide

**Prepared for:** AI Engineer
**Project:** Glamour Styling End-to-End Business Automation
**Platform:** N8N (self-hosted or N8N Cloud)
**Reference site:** https://glamourstyling.oxyscale.ai

---

## Overview

This guide documents every workflow that needs to be built in N8N to automate the Glamour Styling business. The system covers five phases: Lead Capture & Quoting, Acceptance & Booking, Install & Campaign, Property Sold & Pickup, and Reporting.

Each workflow below describes the **trigger**, **step-by-step nodes**, **data transformations**, and **external integrations** required.

---

## Integrations Required (Credentials to Set Up First)

| Service | N8N Integration Method | Purpose |
|---|---|---|
| Google Calendar | OAuth2 (Google Calendar node) | Consult reminders, job events |
| Google Forms / Sheets | OAuth2 (Google Sheets node) | Lead capture form data |
| Monday.com | API key (Monday.com node) | CRM — all lead/job tracking |
| Xero | OAuth2 (Xero node) | Invoice generation |
| Gmail / SMTP | OAuth2 or SMTP | Sending all automated emails |
| Twilio or MessageBird | API key (HTTP Request node) | Sending SMS |
| Realestate.com.au | HTTP Request node (scraper/RSS) | Property status monitoring |
| Google Looker Studio | Google Sheets as data source | Dashboard (no direct N8N node needed) |
| FastField | Webhook from FastField | Partial style PDF attachment |
| Sortly (future) | HTTP Request node | Inventory sync (in discovery) |

---

## Workflow 1: Consultation Reminder (Vacant & Partial Style)

**Trigger:** Google Calendar event created/updated with "styling consult" in the description
**Purpose:** Send reminder to client 2 hours before the on-site consultation

### Steps

1. **Trigger — Google Calendar Trigger node**
   - Poll for new/updated events every 15 minutes
   - Filter: event description contains "styling consult"

2. **Function node — Calculate reminder time**
   - Parse event start time
   - Subtract 2 hours to get `reminderSendAt` timestamp

3. **Wait node**
   - Wait until `reminderSendAt` timestamp is reached

4. **IF node — Check if reminder time has arrived**
   - If current time >= `reminderSendAt`, proceed
   - Otherwise loop back to Wait

5. **Split node — Send via email AND SMS**

   **Branch A — Email (Gmail / SMTP node)**
   - To: client email (extracted from calendar event description or attendees)
   - Subject: `Reminder: Your styling consultation is in 2 hours`
   - Body: Branded HTML email — "Hi [Name], just a reminder that Kali will be arriving at [Address] at [Time]. See you soon!"

   **Branch B — SMS (HTTP Request node → Twilio API)**
   - To: client phone number (from calendar event or Monday.com lookup)
   - Message: `Hi [Name], just a reminder your styling consult with Kali is at [Time] today at [Address].`

---

## Workflow 2: Vacant Property Lead Form Submission

**Trigger:** Google Sheets Trigger — new row added when agent submits the lead form
**Purpose:** Auto-create Monday.com lead, send branded quote + hire agreement, notify all parties

### Form Fields Captured
- Client name, email, phone
- Agent name, agent email
- Property address
- Number of bedrooms
- Living zones selected (checkboxes)
- Indicative styling date
- Auction date

### Steps

1. **Trigger — Google Sheets Trigger node**
   - Watch for new rows in the "Vacant Leads" sheet
   - Map all columns to named variables

2. **Function node — Calculate quote pricing**
   - Base price by bedrooms:
     - 2 bed → $X
     - 3 bed → $Y
     - 4 bed → $Z
     - (Confirm exact pricing with Kali)
   - Add per living zone pricing
   - Outdoor furniture = $0 (included at no charge)
   - Output: `quoteTotal`, `lineItems` array

3. **HTTP Request node → Monday.com API (Create Item)**
   - Board: "Leads / CRM"
   - Item name: `[Property Address] — [Agent Name]`
   - Column values:
     - Client Name, Email, Phone
     - Agent Name, Agent Email
     - Property Address
     - Bedrooms, Living Zones
     - Indicative Styling Date, Auction Date
     - Status: `Quoted`
     - Quote Total

4. **Function node — Generate quote HTML/PDF**
   - Build branded HTML quote document containing:
     - Glamour logo and branding
     - Business story paragraph
     - Package description
     - Line item breakdown (bedrooms + living zones)
     - Total price
     - Outdoor furniture note ("included at no charge")
     - Hire agreement text embedded for e-signature
   - Option A: Use N8N HTML node + PDF conversion (e.g., Gotenberg or html-pdf via Execute Command node)
   - Option B: Pre-built PDF template with Carbone.io or PDF.co via HTTP Request

5. **Gmail node — Send quote to client**
   - To: client email
   - Subject: `Your Glamour Styling Quote — [Property Address]`
   - Body: Branded HTML email with quote summary
   - Attachment: Generated quote PDF with embedded hire agreement

6. **Gmail node — Notify Kali**
   - To: Kali's email
   - Subject: `New Lead: [Property Address] via [Agent Name]`
   - Body: Full lead details summary

7. **Gmail node — Notify managing agent**
   - To: agent email
   - Subject: `Quote Sent: [Client Name] — [Property Address]`
   - Body: Confirmation that quote was sent to client, with next steps

8. **HTTP Request node → Monday.com API (Update Item)**
   - Attach quote PDF link to the Monday.com item
   - Update column: "Quote Sent Date" = today

---

## Workflow 3: Auto Follow-Up on Unaccepted Quotes

**Trigger:** Schedule Trigger — runs daily at 9:00 AM
**Purpose:** Send follow-up email to leads where quote has not been accepted within 2–3 days; archive leads older than 30 days

### Steps

1. **Trigger — Schedule Trigger node**
   - Cron: `0 9 * * *` (every day at 9 AM)

2. **HTTP Request node → Monday.com API (Get Items)**
   - Board: "Leads / CRM"
   - Filter: Status = `Quoted`
   - Return all matching items with `Quote Sent Date`

3. **Split in Batches node**
   - Process each lead one at a time

4. **Function node — Age calculation**
   - Calculate days since `Quote Sent Date`
   - Flag as `needsFollowUp` if 2–3 days old
   - Flag as `archive` if 30+ days old

5. **Switch node**
   - Route A: `needsFollowUp` = true → send follow-up email
   - Route B: `archive` = true → update Monday.com status
   - Route C: neither → no-op (do nothing)

6. **Route A — Gmail node (Follow-up email)**
   - To: client email
   - Subject: `Following up on your Glamour Styling quote`
   - Body: Warm, professional follow-up reiterating value and offering to answer questions

7. **Route A — HTTP Request node → Monday.com API**
   - Update: "Follow-Up Sent" column = today's date

8. **Route B — HTTP Request node → Monday.com API**
   - Update: Status = `Did Not Proceed`

---

## Workflow 4: Partial Style Lead (FastField Integration)

**Trigger:** Webhook — FastField sends POST request after Kali completes on-site consultation form
**Purpose:** Attach FastField PDF to correct Monday.com job item

### Steps

1. **Trigger — Webhook node**
   - Method: POST
   - FastField configured to POST to this N8N webhook URL on form submission
   - Payload contains: job ID or property address, PDF file URL or base64

2. **HTTP Request node — Download PDF**
   - Fetch PDF from FastField URL

3. **HTTP Request node → Monday.com API (Find Item)**
   - Search for item by property address or job reference
   - Return item ID

4. **HTTP Request node → Monday.com API (Upload File)**
   - Upload FastField PDF as a file column on the matched item

5. **HTTP Request node → Monday.com API (Update Item)**
   - Mark column: "FastField PDF Attached" = Yes

---

## Workflow 5: Quote Accepted — Install Date Confirmation

**Trigger:** Webhook or Monday.com Status Change — hire agreement signed by client
**Purpose:** Trigger install date confirmation form and begin booking process

> **Note:** The hire agreement e-signature mechanism (e.g., DocuSign, PandaDoc, or embedded PDF) must send a webhook to N8N on completion, OR a Monday.com automation can update a status column which N8N polls.

### Steps

1. **Trigger — Webhook node** (from e-signature platform on hire agreement acceptance)
   OR
   **Trigger — Schedule node** polling Monday.com for status change to `Accepted`

2. **HTTP Request node → Monday.com API (Update Item)**
   - Status: `Accepted — Awaiting Install Date`

3. **Gmail node — Send install date confirmation form to Kali**
   - Subject: `Quote Accepted: [Property Address] — Confirm Install Date`
   - Body: Prompt Kali to call client/agent to confirm install date and key access details
   - Include link to "Install Date Confirmation Form" (Google Form)

4. **Trigger — Google Sheets Trigger** (separate mini-workflow: fires when Kali submits confirmation form)
   - Fields captured: Confirmed install date, access method (keys / key safe), key safe code, special notes

5. **HTTP Request node → Monday.com API (Update Item)**
   - Update columns: Install Date, Access Details, Key Code

---

## Workflow 6: Invoice Auto-Generation in Xero

**Trigger:** Monday.com item updated — Install Date column populated (from Workflow 5)
**Purpose:** Auto-create invoice in Xero with correct line items and due date

### Steps

1. **Trigger — Webhook from Monday.com**
   - Configure Monday.com automation to call N8N webhook when Install Date column is filled
   - OR use a Schedule Trigger polling Monday.com for items with Install Date newly set

2. **HTTP Request node → Monday.com API (Get Item)**
   - Retrieve full item details: client name, email, line items, install date, quote total

3. **Function node — Calculate due date**
   - Due date = Install Date minus 3 days

4. **Xero node — Create Invoice**
   - Contact: client name + email (create contact if not exists)
   - Line items: match accepted quote line items
   - Due date: calculated above
   - Status: `AUTHORISED`

5. **HTTP Request node → Monday.com API (Update Item)**
   - Column: "Xero Invoice ID" = invoice ID returned by Xero
   - Column: "Invoice Status" = `Unpaid`

---

## Workflow 7: Payment Reminder — 7 Days Before Install

**Trigger:** Schedule Trigger — runs daily at 9:00 AM
**Purpose:** Send payment reminder to clients whose install is in 7 days and invoice is unpaid

### Steps

1. **Trigger — Schedule Trigger node**
   - Cron: `0 9 * * *`

2. **HTTP Request node → Monday.com API (Get Items)**
   - Filter: Install Date = today + 7 days AND Invoice Status = `Unpaid`

3. **Split in Batches node**

4. **Gmail node — Payment reminder email**
   - To: client email
   - Subject: `Friendly reminder — payment due for [Property Address]`
   - Body: Warm, non-confrontational reminder with Xero invoice link and amount due

5. **HTTP Request node (Twilio) — Payment reminder SMS**
   - To: client phone
   - Message: `Hi [Name], just a friendly reminder your invoice for [Address] is due in 3 days. Please check your email for payment details.`

---

## Workflow 8: Invoice Paid → Referral Reminder

**Trigger:** Xero Webhook — invoice status changed to `PAID`
**Purpose:** Notify Kali to pay the managing agent's referral fee

### Steps

1. **Trigger — Webhook node**
   - Xero configured to POST to N8N webhook on invoice status change

2. **IF node — Check status is PAID**
   - If status = `PAID`, proceed
   - Otherwise, stop

3. **HTTP Request node → Monday.com API (Find Item)**
   - Match by Xero invoice ID or client name + address
   - Return item with agent name, property address, referral status

4. **IF node — Check referral not already paid**
   - If "Referral Paid" column is empty or No, proceed

5. **Gmail node — Notify Kali**
   - Subject: `Referral Due: [Agent Name] — [Property Address]`
   - Body: `[Agent Name] is owed a $500 referral for [Property Address]. Payment method: gift card or bank transfer only.`

6. **HTTP Request node → Monday.com API (Update Item)**
   - Invoice Status column: `Paid`

---

## Workflow 9: Photo Timing SMS to Agent

**Trigger:** Schedule Trigger — daily at 8:00 AM
**Purpose:** Ask managing agent when property photos are scheduled (7 days before install)

### Steps

1. **Trigger — Schedule Trigger node**
   - Cron: `0 8 * * *`

2. **HTTP Request node → Monday.com API (Get Items)**
   - Filter: Install Date = today + 7 days AND "Photo Date Confirmed" = empty

3. **Split in Batches node**

4. **HTTP Request node (Twilio) — SMS to agent**
   - To: agent phone
   - Message: `Hi [Agent Name], just confirming — when are photos scheduled for [Property Address]? Please reply with the date so we can coordinate the install.`

   > **Alternative:** Send a Google Form link via SMS for the agent to submit photo date, which auto-populates Monday.com via a Google Sheets trigger.

5. **HTTP Request node → Monday.com API (Update Item)**
   - Column: "Photo SMS Sent" = today

---

## Workflow 10: Google Calendar Job Event Creation

**Trigger:** Monday.com Webhook — Install Date confirmed / status moves to `Booked`
**Purpose:** Auto-create a Google Calendar event for the install job

### Steps

1. **Trigger — Webhook from Monday.com**
   - Monday automation fires when Install Date column is filled

2. **HTTP Request node → Monday.com API (Get Item)**
   - Retrieve: property address, install date, access method, key code, package type, client name

3. **Google Calendar node — Create Event**
   - Title: `INSTALL: [Property Address]`
   - Date: Install Date
   - Description: `Client: [Name] | Access: [Method] | Key Code: [Code] | Package: [Package Type]`
   - Color: coordinate with Kali's colour system (use calendar colour ID — confirm with Kali)
   - Attach picking order PDF link in description once generated (see Workflow 11)

---

## Workflow 11: Picking Order Auto-Generation

**Trigger:** Same Monday.com Webhook as Workflow 10 (can run in parallel)
**Purpose:** Auto-generate a printable picking sheet from the accepted quote

### Steps

1. **Trigger — Webhook from Monday.com** (same trigger as Workflow 10)

2. **HTTP Request node → Monday.com API (Get Item)**
   - Retrieve: package type, bedrooms, living zones, any add-ons

3. **Function node — Map package to furniture list**
   - Use a lookup table mapping package + room selections to specific furniture items
   - Example:
     - Master Bedroom = [King bed frame, mattress, side tables ×2, lamp ×2, linen set, rug]
     - Living Area = [Sofa, coffee table, rug, cushions ×4, artwork]
   - Build full array of all required items with quantity and room label

4. **HTTP Request node → PDF Generator (e.g., PDF.co or Carbone.io)**
   - Pass furniture list into a picking sheet template
   - Template includes: job address, install date, item checklist, quantity, room label
   - Returns PDF URL or base64

5. **HTTP Request node → Monday.com API (Upload File)**
   - Attach picking order PDF to the job item

6. **Google Calendar node — Update Event**
   - Append picking order PDF link to the event description

---

## Workflow 12: AI Property Scanner — Realestate.com.au Monitor

**Trigger:** Schedule Trigger — every 30–60 minutes
**Purpose:** Detect when a staged property goes "Under Offer" or "Sold" and trigger appropriate workflows

> **Important note for engineer:** Realestate.com.au does not have an official public API. Recommended options:
> - **Option A (Recommended):** Use a third-party property data API such as **Domain API**, **PropTrack**, or **Pricefinder** which provide listing status data.
> - **Option B:** HTTP Request node targeting the specific property URL (for internal business use only; review ToS).
> - **Option C (Fallback):** Manual trigger — Kali updates Monday.com status when notified externally; N8N watches for that status change instead of scraping.

### Steps (using Option A — third-party property API)

1. **Trigger — Schedule Trigger node**
   - Cron: `*/30 * * * *` (every 30 minutes)

2. **HTTP Request node → Monday.com API (Get Items)**
   - Filter: Status = `In Progress` (currently installed properties)
   - Return all items with property address + listing URL

3. **Split in Batches node**

4. **HTTP Request node → Property Data API**
   - Query listing by address or listing ID
   - Return current status field

5. **Switch node — Route by detected status**
   - Route A: Status = `Under Offer` → notify Kali
   - Route B: Status = `Sold` → trigger Workflow 14
   - Route C: No change → do nothing

6. **Route A — Gmail node + Twilio SMS to Kali**
   - `[Property Address] is now Under Offer. Begin planning stock reallocation.`

7. **Route A — HTTP Request node → Monday.com API**
   - Update status column: `Under Offer`

---

## Workflow 13: Campaign Expiry Check (7 Weeks)

**Trigger:** Schedule Trigger — daily at 9:00 AM
**Purpose:** Fire alert if a property has been installed for 7 weeks without selling

### Steps

1. **Trigger — Schedule Trigger node**
   - Cron: `0 9 * * *`

2. **HTTP Request node → Monday.com API (Get Items)**
   - Filter: Status = `In Progress` AND Install Date <= today minus 49 days (7 weeks)
   - Exclude items where "Expiry Alert Sent" is already populated

3. **Split in Batches node**

4. **Gmail node — Email to managing agent**
   - Subject: `Campaign Update: [Property Address]`
   - Body: `Hi [Agent Name], we wanted to check in on [Address]. The styling campaign is approaching its 8-week window. Would you like to discuss an extension?`

5. **Gmail node — Alert to Kali**
   - Subject: `Campaign Expiring: [Property Address]`
   - Body: Summary of property, install date, agent contact details

6. **HTTP Request node → Monday.com API (Update Item)**
   - Column: "Expiry Alert Sent" = today's date
   - Column: Status = `Campaign Expiring`

---

## Workflow 14: Property Sold — Close-Out Workflow

**Trigger:** Routed from Workflow 12 (Route B), OR Monday.com Webhook when status set to `Sold`
**Purpose:** Send congratulations, notify all parties, update CRM

### Steps

1. **Trigger — Webhook from Monday.com** (status changed to `Sold`) or incoming call from Workflow 12

2. **HTTP Request node → Monday.com API (Get Item)**
   - Get all details: client, agent, property address, package, install date

3. **Gmail node — Congratulations to client**
   - Subject: `Congratulations on the sale of [Property Address]!`
   - Body: Warm congratulations, mention staging contribution, advise furniture pickup will be scheduled soon

4. **Gmail node — Notify managing agent**
   - Subject: `[Property Address] — Sale Confirmed`
   - Body: Confirm sale received, advise pickup will be coordinated with Kali shortly

5. **Gmail node — Internal alert to Kali**
   - Subject: `SOLD: [Property Address]`
   - Body: Full summary — client name, agent name, package type, install date, action required for pickup scheduling

6. **HTTP Request node → Monday.com API (Update Item)**
   - Status: `Property Sold`
   - Column: "Sold Date" = today

> **Pickup scheduling is managed manually by Kali.** N8N does not auto-schedule the pickup. Kali reviews the "Property Sold" view in Monday.com and coordinates based on team availability and location.

---

## Workflow 15: Post-Pickup Thank-You & Google Review Request

**Trigger:** Monday.com Webhook — status changed to `Pickup Complete`
**Purpose:** Send final thank-you and request Google review

### Steps

1. **Trigger — Webhook from Monday.com** (status = `Pickup Complete`)

2. **HTTP Request node → Monday.com API (Get Item)**
   - Retrieve: client name, email, agent name, agent email, property address

3. **Gmail node — Thank-you to client**
   - Subject: `Thank you from Glamour Styling — [Property Address]`
   - Body: Warm sign-off, acknowledge the journey, invite them to reach out for future properties

4. **Gmail node — Thank-you to managing agent**
   - Subject: `Thanks for the referral — [Property Address]`
   - Body: Acknowledge agent partnership, reinforce referral relationship

5. **Gmail node — Satisfaction survey to client** *(smart review filter)*
   - Subject: `How did we do? — Glamour Styling`
   - Body: "Please share your experience" with link to a Google Form rating (1–5 stars)

6. **Trigger — Google Sheets Trigger** *(second mini-workflow watching survey responses)*
   - When new survey response arrives:
     - **IF rating >= 4** → send follow-up Gmail with direct Google review link
     - **IF rating < 4** → capture feedback internally, notify Kali, do NOT send Google link

7. **HTTP Request node → Monday.com API (Update Item)**
   - Status: `Complete`
   - Column: "Review Requested" = Yes

---

## Workflow 16: Good Luck Message on Install Day

**Trigger:** Schedule Trigger — daily at 7:00 AM
**Purpose:** Send warm "good luck" message to clients whose install is today

### Steps

1. **Trigger — Schedule Trigger node**
   - Cron: `0 7 * * *`

2. **HTTP Request node → Monday.com API (Get Items)**
   - Filter: Install Date = today AND "Good Luck Sent" = empty

3. **Split in Batches node**

4. **Gmail node — Good luck email to client**
   - Subject: `Today's the day — Glamour Styling is on the way!`
   - Body: Warm, branded message — "We're so excited to style [Address] today. Best of luck with your campaign!"

5. **HTTP Request node (Twilio) — Good luck SMS to client**
   - To: client phone
   - Message: `Hi [Name]! The Glamour team is heading to [Address] today — so excited for you!`

6. **HTTP Request node → Monday.com API (Update Item)**
   - Column: "Good Luck Sent" = today

---

## Monday.com Board Structure

**Board Name: "Leads & Jobs — CRM"**

| Column Name | Type | Purpose |
|---|---|---|
| Item Name | Text (auto) | Property address — agent name |
| Client Name | Text | |
| Client Email | Email | |
| Client Phone | Phone | |
| Agent Name | Text | |
| Agent Email | Email | |
| Agent Phone | Phone | |
| Property Address | Text | |
| Bedrooms | Number | |
| Living Zones | Text / Dropdown | |
| Property Type | Dropdown | Vacant / Partial Style |
| Indicative Style Date | Date | From lead form |
| Install Date | Date | Confirmed install date |
| Auction Date | Date | |
| Quote Total | Number | |
| Status | Status | Quoted / Accepted / Booked / In Progress / Under Offer / Campaign Expiring / Property Sold / Pickup Complete / Complete / Did Not Proceed |
| Quote Sent Date | Date | |
| Follow-Up Sent | Date | |
| Access Details | Text | Keys / key safe info |
| Key Code | Text | |
| Xero Invoice ID | Text | |
| Invoice Status | Dropdown | Unpaid / Paid |
| Referral Paid | Dropdown | No / Yes — Gift Card / Yes — Bank Transfer |
| Referral Paid Date | Date | |
| Photo SMS Sent | Date | |
| Photo Date | Date | |
| Good Luck Sent | Date | |
| Sold Date | Date | |
| Review Requested | Checkbox | |
| FastField PDF | File | |
| Picking Order PDF | File | |
| Expiry Alert Sent | Date | |
| Quote PDF Link | Text | URL to quote document |

---

## N8N Workflow Summary Table

| # | Workflow Name | Trigger Type | Key Integrations |
|---|---|---|---|
| 1 | Consultation Reminder | Google Calendar Trigger | Gmail, Twilio |
| 2 | Vacant Lead Form Submission | Google Sheets Trigger | Monday.com, Gmail, PDF generator |
| 3 | Quote Follow-Up (unaccepted) | Schedule (daily 9 AM) | Monday.com, Gmail |
| 4 | FastField Partial Style PDF | Webhook (FastField) | Monday.com |
| 5 | Quote Accepted — Confirm Install Date | Webhook or Monday poll | Monday.com, Gmail, Google Forms |
| 6 | Invoice Auto-Generation | Monday.com Webhook | Xero, Monday.com |
| 7 | Payment Reminder 7 Days Pre-Install | Schedule (daily 9 AM) | Monday.com, Gmail, Twilio |
| 8 | Invoice Paid → Referral Reminder | Webhook (Xero) | Monday.com, Gmail |
| 9 | Photo Timing SMS to Agent | Schedule (daily 8 AM) | Monday.com, Twilio |
| 10 | Google Calendar Job Event | Monday.com Webhook | Google Calendar, Monday.com |
| 11 | Picking Order Generation | Monday.com Webhook | PDF generator, Monday.com, Google Calendar |
| 12 | AI Property Scanner | Schedule (every 30 min) | Property API, Monday.com, Gmail, Twilio |
| 13 | Campaign Expiry Check | Schedule (daily 9 AM) | Monday.com, Gmail |
| 14 | Property Sold Close-Out | Monday.com Webhook | Monday.com, Gmail |
| 15 | Post-Pickup Thank-You & Review | Monday.com Webhook | Monday.com, Gmail, Google Forms |
| 16 | Good Luck Install Day Message | Schedule (daily 7 AM) | Monday.com, Gmail, Twilio |

---

## Recommended Build Order

Build in this order to deliver value incrementally and get the highest-ROI workflows live first.

**Sprint 1 — Core Lead Flow**
1. Workflow 2: Vacant Lead Form → Quote → Monday.com → Email notifications
2. Workflow 6: Invoice Auto-Generation in Xero
3. Workflow 7: Payment Reminder 7 Days Pre-Install

**Sprint 2 — Booking & Operations**
4. Workflow 5: Quote Accepted → Install Date Confirmation
5. Workflow 10: Google Calendar Event Creation
6. Workflow 11: Picking Order Generation
7. Workflow 1: Consultation Reminders

**Sprint 3 — Campaign Management**
8. Workflow 16: Good Luck Message on Install Day
9. Workflow 9: Photo Timing SMS to Agent
10. Workflow 12: Property Scanner (confirm API approach with Kali first)
11. Workflow 13: Campaign Expiry Check (7 Weeks)

**Sprint 4 — Close-Out & Reputation**
12. Workflow 8: Invoice Paid → Referral Reminder
13. Workflow 14: Property Sold Close-Out
14. Workflow 15: Post-Pickup Thank-You & Google Review

**Sprint 5 — Edge Cases & Partial Styles**
15. Workflow 3: Quote Follow-Up (unaccepted)
16. Workflow 4: FastField Integration

---

## Questions to Clarify with Kali Before Building

1. **Pricing table** — Exact pricing per bedroom count and per living zone (needed for Workflow 2)
2. **Quote/hire agreement template** — Does Glamour have a branded Word/PDF template, or does one need to be designed from scratch?
3. **E-signature platform** — How does Kali currently handle hire agreement signing? (DocuSign, PandaDoc, embedded PDF link, or manual?)
4. **Property scanner** — Is Kali willing to use a paid third-party API (e.g., Domain API), or should this default to a manual Monday.com status update trigger?
5. **Google review link** — Kali needs to provide the exact Google Business Profile review URL for use in Workflow 15
6. **SMS sender name/number** — Confirm Twilio account setup and preferred sender name or number
7. **Referral amount** — Confirm referral is always $500 flat regardless of job size
8. **Partial style quoting form fields** — Confirm full field list for Kali's partial style quoting form (Workflow 4)
9. **N8N hosting** — Self-hosted or N8N Cloud? (Affects webhook URL format and setup)
10. **Google Calendar colour system** — What colours map to which job types? (Needed for Workflow 10)
11. **Furniture picking list** — Provide a complete list of furniture items per bedroom and per living zone so the picking order lookup table can be built (Workflow 11)

---

*Document prepared by Oxyscale AI*
*Reference: https://glamourstyling.oxyscale.ai*

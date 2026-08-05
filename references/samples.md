# Official Zoho CRM Widget Code Samples

Source: `https://www.zohocrm.dev/explore/widgets/codesamples`

The actual downloadable zip files require a logged-in Zoho Developer Space session.
Metadata below is extracted from the published dataset (`code_samples_file_list_new_v1.json`).

Use this reference to:
- Match a user's use case to the closest official sample
- Identify which widget type and location fits the task
- Point users to the right Kaizen help article for deep-dive context

---

## Sample 1 — Enhancing Mass Communication with a Custom Button Widget

**Widget type:** Custom Button — List View  
**Module:** Leads  
**Event:** OnLoad

**Use case:** A custom button widget that sends bulk SMS notifications and pre-recorded calls to
multiple selected Lead records from the List View. Lets sales teams run mass outreach campaigns
directly from CRM without switching tools.

**Key patterns:**
- List View button placement (`crm.record.detail.button` on list context)
- Iterating over selected `EntityId` array from `PageLoad` data
- Calling external communication APIs via `ZOHO.CRM.CONNECTION.invoke`

**Reference:** [Kaizen #156](https://help.zoho.com/portal/en/community/topic/kaizen-156-enhancing-mass-communication-in-zoho-crm-with-a-custom-button-widget)

---

## Sample 2 — Blog Feed Updates in Zoho CRM Dashboard

**Widget type:** Dashboard  
**Event:** OnLoad

**Use case:** A dashboard tile widget that displays the most recent blog posts of your preferred
products or services, refreshed daily via a scheduled Zoho Function. Keeps sales teams
informed about product news without leaving CRM.

**Key patterns:**
- Dashboard widget placement (no Entity or EntityId context)
- Fetching external RSS/blog data via `ZOHO.CRM.FUNCTIONS.execute` (scheduled function)
- Using `ZOHO.CRM.API.getOrgVariable` to store the last-fetched timestamp

**Reference:** [Kaizen #140](https://help.zoho.com/portal/en/community/topic/kaizen-140-integrating-blog-feed-into-zoho-crm-dashboard)

---

## Sample 3 — Fetching Desk Ticket Details in CRM using ZRC Widget

**Widget type:** Record detail (Contact page)  
**Event:** OnLoad

**Use case:** Displays all open Zoho Desk support tickets for a Contact record directly inside
CRM, so sales reps stay informed about ongoing support issues without switching apps.

**Key patterns:**
- ZRC (Zoho Remote Call) unified syntax for cross-product API calls
- Zoho Desk connection via `ZOHO.CRM.CONNECTION.invoke`
- Showing related-service data in a record detail sidebar

---

## Sample 4 — End-to-End EmbeddedApp with Zylker APIs

**Widget type:** EmbeddedApp (general)  
**Event:** OnLoad

**Use case:** A complete reference implementation demonstrating the full ZRC API surface:
CRM API calls (users, roles, profiles, user creation), external API calls using Zylker APIs
with FormData and baseUrl, connection-based API calls, and `zrc.createInstance` for reusable
API client configuration.

**Key patterns:**
- `zrc.createInstance` for reusable API clients across multiple calls
- ZRC CRM API: `users`, `roles`, `profiles`, user creation
- ZRC external API calls with `FormData`, `baseUrl`, `connection`, and `responseType` options
- Comprehensive error handling across multiple API types

---

## Sample 5 — Full EmbeddedApp with Zylker APIs (Image Upload Flow)

**Widget type:** EmbeddedApp (general)  
**Event:** OnLoad

**Use case:** Extends Sample 4 with an image download-and-upload flow — downloads an image
from a CRM record, processes it, and uploads it to a webhook URL using Zylker APIs.
Demonstrates file handling within the widget SDK.

**Key patterns:**
- `zrc.createInstance` for reusable API clients
- CRM API calls: users, roles, profiles
- External GET/POST with `FormData`
- Binary file (image) download and re-upload to an external endpoint

---

## Sample 6 — Mandating Subform Data for Blueprint Transitions

**Widget type:** Blueprint  
**Event:** OnLoad

**Use case:** Enforces that subform data from a related module is filled in before a Blueprint
transition can proceed on a Deals record. Prevents sales reps from advancing a deal stage
without completing required linked data.

**Key patterns:**
- Blueprint widget placement — widget renders inside the transition panel
- Reading subform/related module data via `ZOHO.CRM.API.getRelatedRecords`
- Blocking or allowing `ZOHO.CRM.BLUEPRINT.proceed` based on validation result
- Using `ZDK.Client.showAlert` or `ZOHO.CRM.UI.Popup` to communicate errors

**Reference:** [Kaizen #181](https://help.zoho.com/portal/en/community/topic/kaizen-181-mandating-subform-data-for-blueprint-transitions-using-widget)

---

## Sample 7 — Geolocating Leads' Addresses in Zoho CRM

**Widget type:** Custom Related List — Leads detail page  
**Event:** OnLoad

**Use case:** A Related List widget on the Lead detail page that renders an interactive map
showing the lead's location, provides directions from the current user's location, and
highlights other leads in the same street or city.

**Key patterns:**
- Related List widget placement (`crm.record.detail.related`)
- `ZOHO.CRM.API.getRecord` to fetch lead address fields
- `ZOHO.CRM.API.searchRecord` with criteria to find nearby leads
- Embedding a third-party maps API (e.g. Google Maps, Leaflet) with CSP whitelisting

**Reference:** [Kaizen #114](https://help.zoho.com/portal/en/community/topic/kaizen-114-geocoding-leads-addresses-in-zoho-crm)

---

## Sample 8 — Timer and Worklog Widget

**Widget type:** Custom Button (rendered in a Flyout via Client Script)  
**Event:** OnLoad

**Use case:** A timer widget that lets support or sales reps accurately track time spent on
multiple tasks. Runs inside a CRM Flyout (triggered by Client Script), logs work sessions,
and writes worklog entries back to CRM records.

**Key patterns:**
- Widget rendered inside a Client Script Flyout (`ZDK.Client.openPopup` / `NotifyAndWait`)
- Persistent timer state across widget lifecycle using `ZOHO.CRM.API.getOrgVariable`
- Writing worklog data back via `ZOHO.CRM.API.insertRecord` or `addNotes`
- Requires companion Client Script: `https://www.zohocrm.dev/explore/client-script/codesamples/render-widget-in-flyouts`

**Reference:** [Kaizen #187](https://help.zoho.com/portal/en/community/topic/kaizen-187-building-a-timer-and-worklog-widget-part-1)

---

## Sample 9 — Widget Between Blueprint Transition States

**Widget type:** Blueprint — Loans module detail page  
**Trigger field:** "Check Profile" transition button  
**Event:** OnLoad

**Use case:** Displays a loan applicant's past loan history during the approval Blueprint
transition, so approvers can make informed decisions without navigating away from the
transition panel.

**Key patterns:**
- Blueprint widget renders when a specific transition button is clicked
- `ZOHO.CRM.API.getRelatedRecords` to pull loan history from a related module
- Read-only data display — widget informs the approver, `ZOHO.CRM.BLUEPRINT.proceed` proceeds after review
- No user input captured; widget is informational only

**Reference:** [Kaizen #95](https://help.zoho.com/portal/en/community/topic/kaizen-95-how-is-a-widget-used-in-a-blueprint)

---

## Sample 10 — Render Widget in a Pop-up using Client Script

**Widget type:** Custom Button / triggered by picklist field change  
**Event:** OnLoad

**Use case:** Demonstrates two-way communication between a widget and Client Script — a Client
Script opens a widget in a popup, the widget collects data, and passes that data back to the
Client Script to update CRM fields.

**Key patterns:**
- `ZOHO.embeddedApp.on("NotifyAndWait", ...)` to receive the popup trigger with context
- `ZDK.Client.sendResponse(data.id, payload)` to pass data back to Client Script
- Widget as a data-collection UI triggered by another CRM event
- Requires companion Client Script: `https://www.zohocrm.dev/explore/client-script/codesamples/render-widgets-using-client-script`

**Reference:** [Zoho Community — Render Widgets using Client Script](https://help.zoho.com/portal/en/community/topic/render-widgets-using-client-script)

---

## Sample 11 — Escalation Workflow for Customer Complaints

**Widget type:** Related List widget + Custom Button widget (two widgets, one zip)  
**Event:** OnLoad

**Use case:** A support escalation flow built across two coordinated widgets:
1. A Related List widget shows a list of customer complaints
2. When the "Escalate" button is clicked, a confirmation box appears, then a popup widget
   captures escalation details (tier, notes, assignee)

**Key patterns:**
- `ZDK.Client.showConfirmation` for the confirmation step
- `ZDK.Client.openPopup` to open the escalation-detail widget as a popup
- Widget-to-widget communication: Related List widget triggers Button widget via Client Script
- The Button widget has API name `EscalationPopup` — referenced by the Related List widget
- Demonstrates the full `ZDK.Client` UI helper suite (`showMessage`, `showAlert`, `showConfirmation`, `openPopup`)
- Requires two separate widgets installed from the same zip file

---

## Pattern Index

Use this to find the right sample for a given requirement:

| Requirement | Relevant sample(s) |
|-------------|-------------------|
| Act on multiple selected records | #1 (mass communication) |
| Dashboard / analytics tile | #2 (blog feed) |
| Cross-product data (Desk, Books, etc.) | #3 (Desk tickets via ZRC) |
| ZRC / unified API syntax | #3, #4, #5 |
| File upload/download in a widget | #5 |
| Validate before Blueprint proceeds | #6 (subform mandate), #9 (loan history) |
| Map / geolocation in CRM | #7 |
| Time tracking | #8 (timer worklog) |
| Widget opened by Client Script | #8, #10 |
| Widget ↔ Client Script data exchange | #10 (NotifyAndWait / sendResponse) |
| Multi-widget coordination | #11 (escalation, two widgets) |
| ZDK.Client UI helpers showcase | #11 |
| Confirmation + popup flow | #11 |

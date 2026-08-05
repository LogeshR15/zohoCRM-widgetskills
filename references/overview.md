# Zoho CRM Widgets — Overview

## What is a Widget?

Widgets are embeddable UI components you create and add to Zoho CRM. Use them to display data
from third-party applications or perform actions that aren't natively available in CRM — for
example, a softphone dialpad, an analytics heatmap, a website monitoring panel, or any external
service you want to surface inside CRM without switching tabs.

Widgets are built with HTML, CSS, and JavaScript using the CRM JS SDK. They run inside a
sandboxed iframe injected by CRM and communicate with the CRM host via the SDK's postMessage
layer. Widgets support ZRC, enabling seamless API calls using unified syntax across contexts.

---

## Pre-requisites

**CRM Edition**
Widgets are available on **Professional, Enterprise, and Ultimate** editions of Zoho CRM only.
They are not available on the Free or Standard editions.

**Developer Permissions**
The CRM profile used to create and manage widgets must have Developer Permissions enabled:
> Setup → Users and Control → Security Control → Profiles → select profile → enable **Developer Permissions**

**Browser support**
Check the Zoho CRM supported browser versions page for the current list. Widgets run inside
CRM's iframe and inherit the same browser compatibility constraints.

**ZET CLI (for local development)**
```bash
npm install -g zoho-extension-toolkit   # requires Node.js v6+, npm v3+
zet --version                           # verify: 1.0.28 or later
```

---

## Limits

| Edition | Max widgets per org |
|---------|-------------------|
| Professional | 200 |
| Enterprise | 200 |
| Ultimate | 200 |

These are org-level limits across all widget types combined.

---

## Types of Widgets

Eight placement types are available. Each maps to a different `location` string in
`plugin-manifest.json`:

| Widget type | `location` string | Where it appears |
|-------------|------------------|-----------------|
| Widget in a Dashboard | `crm.dashboard` | CRM Analytics / Dashboard tab — draggable tile |
| Widget in a Web tab | `crm.webtab` | A dedicated full-page tab in the CRM navigation |
| Widget in a Custom Button | `crm.record.detail.button` | Button in the record detail view action bar |
| Widget in a Custom Related List | `crm.record.detail.related` | Related list section on a record detail page |
| Widget in a Wizard | `crm.wizard` | Embedded inside a CRM Wizard step |
| Widget in a Signal | `crm.signal` | Signal notification panel |
| Widget in Settings | `crm.settings` | A tab inside CRM Setup / Settings |
| Widget in a Blueprint | `crm.blueprint` | Embedded inside a Blueprint transition panel |

Additional locations available in `plugin-manifest.json` for record-level placements:

| `location` string | Where it appears |
|------------------|-----------------|
| `crm.record.detail` | Record detail page — sidebar panel |
| `crm.record.list` | List view — toolbar button |
| `crm.record.create` | Create record form |
| `crm.homepage` | CRM Homepage — draggable widget tile |

> **Note:** The `location` strings for Dashboard, Web tab, Wizard, Signal, and Blueprint widgets
> have not been verified against an authoritative Zoho source. Confirm exact strings in the
> Zoho CRM Developer Space or your ZET project after `zet init` before relying on them.

---

## What Widgets Can and Cannot Do

**Widgets can:**
- Read, create, update, and delete CRM records via the JS SDK
- Call external APIs (with domains whitelisted in `cspDomains`)
- Execute Deluge/serverless functions via `ZOHO.CRM.FUNCTIONS.execute`
- Trigger Blueprint transitions and Wizard steps
- Open popups, resize their own iframe, and navigate to CRM records
- Communicate with the CRM host and other widgets via `ZOHO.CRM.EVENTS.dispatch`

**Widgets cannot:**
- Run in the background or on a schedule (use Zoho Functions or Workflow for that)
- Access other browser tabs, the file system, or local device APIs
- Use `localStorage`, `sessionStorage`, or cookies (iframe is sandboxed)
- Make unauthenticated API calls to CRM — all calls use the logged-in user's session
- Exceed the logged-in user's CRM field-level and module-level permissions

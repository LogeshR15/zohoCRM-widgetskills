# zohoCRM-widgetskills — Claude Code Skill

A Claude Code skill for building Zoho CRM widgets using the ZET CLI (`zoho-extension-toolkit`).

## What it covers

- **Scaffolding** — `zet init --zoho-service CRM --project-name <name>` and project structure
- **plugin-manifest.json** — schema, widget locations, config fields, validation rules
- **CRM JS SDK v1.5 / v1.4** — every namespace, transcribed from Zoho's published JSDoc dataset:
  - `ZOHO.embeddedApp` — `init()` and all **6** events (`PageLoad`, `Dial`, `DialerActive`, `Notify`, `NotifyAndWait`, `ContextUpdate`)
  - `ZOHO.CRM.API` — all **28** methods: records (`getRecord`, `getAllRecords`, `insertRecord`, `updateRecord`, `upsertRecord`, `deleteRecord`, `searchRecord`, `coql`), related lists (`getRelatedRecords`, `updateRelatedRecords`, `delinkRelatedRecord`), notes & files (`addNotes`, `attachFile`, `uploadFile`, `getFile`), users & profiles (`getUser`, `getAllUsers`, `getProfile`, `getAllProfiles`, `updateProfile`), approvals (`getApprovalRecords`, `getApprovalById`, `getApprovalsHistory`, `approveRecord`), blueprint (`getBluePrint`, `updateBluePrint`), plus `getAllActions` and `getOrgVariable`
  - `ZOHO.CRM.CONFIG` — **3** methods: `getCurrentUser`, `getOrgInfo`, `getUserPreference`
  - `ZOHO.CRM.META` — **6** methods: `getFields`, `getModules`, `getLayouts`, `getRelatedList`, `getCustomViews`, `getAssignmentRules`
  - `ZOHO.CRM.HTTP` — `get`, `post`, `put`, `patch`, `delete`
  - `ZOHO.CRM.CONNECTION` — `invoke` · `ZOHO.CRM.CONNECTOR` — `invokeAPI`, `authorize`
  - `ZOHO.CRM.FUNCTIONS` — `execute` · `ZOHO.CRM.BLUEPRINT` — `proceed` · `ZOHO.CRM.WIZARD` — `post`
  - `ZOHO.CRM.UI` — `Resize`, `Popup`, `Record` (incl. `populate`), `Widget`, `Dialer`
  - `ZOHO.CRM.EVENTS` — `dispatch`
  - `ZDK.Client` — 1 method in v1.4, **8** in v1.5 (`showMessage`, `showAlert`, `showConfirmation`, `getInput`, `openPopup`, `showLoader`, `hideLoader`, `sendResponse`)
  - `$Client` — `close`
- **A "does not exist" table** — the widely-repeated method names that were never real, each mapped to its actual equivalent
- **Zoho's own documented contradictions** — `RelatedListName` vs `RelatedList`, `Id` vs `LayoutId`, `Content` vs `content`, and the non-uniform response envelopes
- **Dev workflow** — `zet run`, cert trust, CRM sandbox, `zet validate`, `zet pack`, Sigma deploy
- **Error handling** — payload-vs-rejection checking, envelope normalizer, retries, init failures
- **Security** — secrets via `getOrgVariable`, Connections over stored keys, XSS, CSP, permission gating
- **Troubleshooting** — silent event-name typos, calls that fail, CORS, i18n + locale, performance

## Accuracy

`sdk.md`, `error-handling.md`, `security.md`, and `troubleshooting.md` are verified against Zoho's
published JSDoc dataset (`widget-sdk-1.5.json`) and cross-checked against the shipped bundle at
`live.zwidgets.com/js-sdk/1.4/`. Each file states its provenance and how to re-derive it.

`manifest.md` and `workflow.md` carry an **unverified** banner — their contents have not been
checked against an authoritative source yet. Contributions welcome.

Why this matters: the doc site is a client-rendered SPA, so ordinary HTML fetches of
`zohocrm.dev/explore/widgets/...` return only a page shell. That is the likely root cause of the
inaccurate material about this SDK in circulation. The fix is to read the underlying dataset —
`SKILL.md` documents the two commands.

## Install in Claude Code

```bash
/install-skill https://github.com/LogeshR15/zohoCRM-widgetskills
```

Or clone manually into your Claude skills directory:

```bash
git clone https://github.com/LogeshR15/zohoCRM-widgetskills ~/.claude/skills/zohoCRM-widgetskills
```

Then invoke with:

```
/zohoCRM-widgetskills
```

## Prerequisites

- Node.js v6+, npm v3+
- ZET CLI: `npm install -g zoho-extension-toolkit`
- Zoho CRM account with widget/extension permissions
- (Optional) ZohoCRM MCP tools connected for live record data during development

## File structure

```
SKILL.md                   — skill entry point, triggers, wrong-vs-right method table
references/
  sdk.md                   — ✅ CRM JS SDK v1.5/v1.4 verified API reference
  error-handling.md        — ✅ payload checking, envelope normalizer, retries
  security.md              — ✅ secrets, XSS, CSP, permission gating, compliance
  troubleshooting.md       — ✅ debugging, silent failures, i18n, performance
  manifest.md              — ⚠️ plugin-manifest.json schema (unverified)
  workflow.md              — ⚠️ ZET CLI dev workflow and deploy steps (unverified)
```

## License

MIT

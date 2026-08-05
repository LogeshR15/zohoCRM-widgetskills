# zohoCRM-widgetskills — Claude Code Skill

A Claude Code skill for building Zoho CRM widgets using the ZET CLI (`zoho-extension-toolkit`).

## What it covers

- **Scaffolding** — `zet init --zoho-service CRM --project-name <name>` and project structure
- **plugin-manifest.json** — full schema, all widget locations, config fields, validation rules
- **CRM JS SDK v1.5** — complete reference for every namespace:
  - `ZDK.Client`, `ZOHO.embeddedApp`
  - `ZOHO.CRM.API` — getRecord, searchRecord, addRecord, updateRecord, deleteRecord, getRelatedRecords, getUser, getAllUsers
  - `ZOHO.CRM.CONFIG` — getFields, getLayouts, getModules, getPickListValues, getParameter
  - `ZOHO.CRM.META` — getEnvironment, getPortalInfo, getAppInfo
  - `ZOHO.CRM.HTTP` — get, post, put, delete, patch
  - `ZOHO.CRM.CONNECTION` / `CONNECTOR` — OAuth connector requests
  - `ZOHO.CRM.FUNCTIONS` — execute Deluge/serverless functions
  - `ZOHO.CRM.BLUEPRINT` — getTransitions, proceed
  - `ZOHO.CRM.WIZARD` — proceed
  - `ZOHO.CRM.UI` — Popup, Record, Widget, Dialer, Resize
  - `ZOHO.CRM.$Client` — event bus
- **Dev workflow** — `zet run`, cert trust, CRM sandbox, `zet validate`, `zet pack`, Sigma deploy
- **Error handling** — API error patterns, retry logic, graceful degradation, monitoring
- **Security** — protecting secrets, input validation, CSP/CORS, auth best practices, compliance
- **Troubleshooting** — common issues, debugging tips, performance optimization

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
SKILL.md                   — skill entry point and triggers
references/
  manifest.md              — plugin-manifest.json schema reference
  sdk.md                   — CRM JS SDK v1.5 full API reference
  workflow.md              — ZET CLI dev workflow and deploy steps
  error-handling.md        — API error patterns, retry strategies, graceful degradation
  security.md              — security best practices, secrets management, input validation
  troubleshooting.md       — common issues, debugging, performance optimization
```

## License

MIT

---
name: zohoCRM-widgetskills
description: "Zoho CRM widget development — ZET CLI (zoho-extension-toolkit), plugin-manifest.json schema, CRM JS SDK v1.5 (ZOHO.CRM.API, UI, CONFIG, FUNCTIONS, HTTP, CONNECTION, CONNECTOR, BLUEPRINT, META, WIZARD, $Client, ZDK.Client), local dev workflow, pack and deploy to Sigma. Trigger on 'build a CRM widget', 'create a widget', 'CRM extension', 'zet init', 'plugin-manifest', 'ZOHO.CRM.API', 'ZOHO.embeddedApp', 'ZDK.Client', or any Zoho CRM widget question."
compatibility: "Requires Node.js v6+ and npm v3+. Install ZET: npm install -g zoho-extension-toolkit. Verify: zet --version (current: 1.0.28)."
metadata:
  version: "1.0.0"
---

## How It Works

1. **Check for existing project** — look for `plugin-manifest.json` in the working directory. If missing, scaffold with `zet init`; never create the manifest manually.
2. **Check MCP tools** — if `mcp__ZohoCRM__*` tools are visible, use them to pull live record data for testing widget logic (field names, IDs, deal stages). Never ask the user to copy-paste IDs.
3. **Identify what to build** — widget type/location, data it needs, UI framework preference (vanilla JS, React, or plain HTML).
4. **Load the relevant reference** — manifest schema, SDK methods, dev workflow, or deploy steps.
5. **Write the widget** — scaffold `app/widget.html` and any supporting JS/CSS. Always initialize with `ZOHO.embeddedApp.on("PageLoad", ...)` + `ZOHO.embeddedApp.init()` before any API calls.
6. **Dev loop** — `zet run` → open `https://127.0.0.1:5000` → authorize cert → test inside CRM widget sandbox.
7. **Deploy** — `zet validate` → fix errors → `zet pack` → upload the generated `.zip` to Sigma Marketplace or CRM extension install.

## Pre-flight: Never Do This

- Do NOT run `zet init` if `plugin-manifest.json` already exists — it overwrites the project.
- Do NOT call any `ZOHO.CRM.*` API before `ZOHO.embeddedApp.init()` resolves.
- Do NOT use `http://` for the widget URL in production — CRM only loads HTTPS widget URLs.
- Do NOT create `key.pem`/`cert.pem` manually — `zet run` generates them automatically.
- Do NOT put secrets in `plugin-manifest.json` config values unless `"secure": true`.

## ZET CLI — Non-interactive Init (CRM is supported)

```bash
# CRM supports non-interactive init — use this instead of interactive zet init
zet init --zoho-service CRM --project-name MyWidgetName

# Other commands
zet run            # Start HTTPS dev server at https://127.0.0.1:5000
zet validate       # Validate manifest + file rules before packing
zet pack           # Create widget.zip for Sigma upload
zet push           # Push to Sigma (requires zet login first)
zet pull           # Fetch extension from Sigma
zet cloud_run      # Run in Sigma cloud sandbox
zet cloud_stop     # Stop Sigma cloud run
zet login          # Authenticate with Sigma account
zet whoami         # Show current login + workspace
zet list_workspace # Switch workspace
```

## MCP Integration

When `mcp__ZohoCRM__*` tools are available, use them to enrich widget development:

| Task | MCP tool to use |
|------|----------------|
| Get field names for a module | `mcp__ZohoCRM__get_crm_record` on a sample record |
| List deal stages for config | `mcp__ZohoCRM__get_pipeline_stages` |
| Test widget data fetch live | `mcp__ZohoCRM__search_crm_records` |
| Verify record IDs for testing | `mcp__ZohoCRM__list_deals` or `search_crm_records` |

## References

| Reference | Load when the query is about… |
|-----------|-------------------------------|
| `references/manifest.md` | `plugin-manifest.json` schema — service, modules.widgets, modules.connectors, config fields, location names, validation rules |
| `references/sdk.md` | Full CRM JS SDK v1.5 — ZDK.Client, ZOHO.embeddedApp, ZOHO.CRM.API, UI, CONFIG, FUNCTIONS, HTTP, CONNECTION, CONNECTOR, BLUEPRINT, META, WIZARD, $Client — method signatures + examples |
| `references/workflow.md` | Dev workflow — `zet run`, cert trust, widget sandbox, `zet validate` errors, `zet pack` output, Sigma upload, cloud_run |
| `references/error-handling.md` | Handling API errors — permission errors, retry patterns, missing records, timeouts, graceful degradation, monitoring |
| `references/security.md` | Security best practices — protecting secrets, input validation, CSP/CORS, auth patterns, compliance, dependency management |
| `references/troubleshooting.md` | Common issues and fixes — widget not loading, ZOHO undefined, API calls failing, CORS errors, slow performance |

## Triggers

Use this skill for: "build a CRM widget", "create a widget", "CRM extension", "Zoho widget", "zet init", "zet run", "plugin-manifest.json", "ZOHO.CRM.API", "ZOHO.embeddedApp", "ZDK.Client", "widget location", "widget SDK", "embed a widget in CRM", "zoho-extension-toolkit", or any question about Zoho CRM widget development, ZET CLI commands, or the CRM JS SDK.

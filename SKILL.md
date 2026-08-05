---
name: zohoCRM-widgetskills
description: "Zoho CRM widget development — ZET CLI (zoho-extension-toolkit), plugin-manifest.json schema, CRM JS SDK v1.5/v1.4 (ZOHO.CRM.API, META, CONFIG, UI, HTTP, CONNECTION, CONNECTOR, FUNCTIONS, BLUEPRINT, WIZARD, EVENTS, ZDK.Client, $Client), local dev workflow, pack and deploy to Sigma. Trigger on 'build a CRM widget', 'create a widget', 'CRM extension', 'zet init', 'plugin-manifest', 'ZOHO.CRM.API', 'ZOHO.embeddedApp', 'ZDK.Client', or any Zoho CRM widget question."
compatibility: "Requires Node.js v6+ and npm v3+. Install ZET: npm install -g zoho-extension-toolkit. Verify: zet --version (current: 1.0.28)."
metadata:
  version: "2.0.0"
---

## How It Works

1. **Check for existing project** — look for `plugin-manifest.json` in the working directory. If missing, scaffold with `zet init`; never create the manifest manually.
2. **Check MCP tools** — if `mcp__ZohoCRM__*` tools are visible, use them to pull live record data for testing widget logic (field names, IDs, deal stages). Never ask the user to copy-paste IDs.
3. **Identify what to build** — widget type/location, data it needs, UI framework preference (vanilla JS, React, or plain HTML).
4. **Load the relevant reference** — manifest schema, SDK methods, dev workflow, or deploy steps.
5. **Write the widget** — scaffold `app/widget.html` and any supporting JS/CSS. Always initialize with `ZOHO.embeddedApp.on("PageLoad", ...)` + `ZOHO.embeddedApp.init()` before any API calls.
6. **Dev loop** — `zet run` → open `https://127.0.0.1:5000` → authorize cert → test inside CRM widget sandbox.
7. **Deploy** — `zet validate` → fix errors → `zet pack` → upload the generated `.zip` to Sigma Marketplace or CRM extension install.

## Never invent an SDK method

`references/sdk.md` is transcribed from Zoho's published JSDoc dataset and is the **only**
authority in this skill. Widely-circulated third-party material about this SDK documents methods
that do not exist. Before writing any `ZOHO.*` call, confirm the name against that file — it
includes a "Does not exist — do not use" table for the common fabrications.

The traps that bite most often:

| Wrong | Right |
|-------|-------|
| `ZOHO.CRM.API.addRecord` | `ZOHO.CRM.API.insertRecord` |
| `ZOHO.CRM.CONFIG.getFields` / `getModules` / `getLayouts` | same names on `ZOHO.CRM.META` |
| `ZOHO.CRM.CONFIG.getParameter` | `ZOHO.CRM.API.getOrgVariable` |
| `ZOHO.CRM.CONFIG.getPickListValues` | `META.getFields` → `field.pick_list_values` |
| `ZOHO.CRM.META.getEnvironment` / `getPortalInfo` / `getAppInfo` | `CONFIG.getCurrentUser` / `CONFIG.getOrgInfo` / the `PageLoad` payload |
| `CONNECTION.makeRequest` / `CONNECTOR.makeRequest` | `CONNECTION.invoke(name, req)` / `CONNECTOR.invokeAPI(ns, data)` |
| `BLUEPRINT.getTransitions` | `API.getBluePrint` (+ `API.updateBluePrint`) |
| `WIZARD.proceed` | `WIZARD.post(record_data)` |
| `UI.Dialer.open` / `.close` | `maximize()` / `minimize()` / `notify()` — all zero-arg |
| `ZDK.Client.on` / `.getConfig` / `.resize` | `ZOHO.embeddedApp.on` / `API.getOrgVariable` / `UI.Resize` |
| a `Fields` param on `getRecord` | no such param — use `API.coql` to project |

`ZOHO.CRM.CONFIG` has exactly **3** methods. `ZOHO.CRM.META` has **6**. `ZOHO.CRM.API` has **28**.

## SDK version and script tag

Target **v1.5** — it is a superset of v1.4 (identical `ZOHO.CRM.*`, plus 7 extra `ZDK.Client`
UI helpers). Pin the version explicitly; Zoho's own "latest" aliases disagree with each other.

```html
<script src="https://live.zwidgets.com/js-sdk/1.5/ZohoEmbededAppSDK.min.js"></script>
```

The filename misspelling `ZohoEmbededAppSDK` (one `d`) is authentic.

## Events — all six

`ZOHO.embeddedApp.on` does not validate event names, so a typo fails **silently and permanently**.
The complete set: `PageLoad`, `Dial`, `DialerActive`, `Notify`, `NotifyAndWait`, `ContextUpdate`.
There is no `App.Load`. Subscribe before calling `init()`.

In `PageLoad`, `data.EntityId` is a **string** for related-list placements and an **array** for
button placements — branch on `Array.isArray()`.

## Pre-flight: Never Do This

- Do NOT run `zet init` if `plugin-manifest.json` already exists — it overwrites the project.
- Do NOT call any `ZOHO.CRM.*` API before the `PageLoad` handler fires.
- Do NOT assume a resolved Promise means success — status lives in the payload (`code`/`status`/`message`), and the envelope shape differs per method.
- Do NOT code against numeric CRM error codes as if the widget SDK defined them; it publishes none.
- Do NOT use `http://` for the widget URL in production — CRM only loads HTTPS widget URLs.
- Do NOT create `key.pem`/`cert.pem` manually — `zet run` generates them automatically.
- Do NOT put secrets in `plugin-manifest.json` config values unless `"secure": true`.
- Do NOT assume `ZDK.Client` / `$Client` exist — CRM injects them only for Client Script and popup contexts. Feature-detect.
- Do NOT leave a v1.5 loader showing — an active loader blocks every other pop-up. Pair `showLoader()` with `hideLoader()`.

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

| Reference | Verified | Load when the query is about… |
|-----------|:--------:|-------------------------------|
| `references/overview.md` | ⚠️ | **Load first for any new user or "what is a widget" question.** Widget definition, CRM edition requirements (Professional/Enterprise/Ultimate), Developer Permissions setup, 200-widget org limit, all 8 widget placement types with location strings, and what widgets can/cannot do |
| `references/sdk.md` | ✅ | **Load before writing any `ZOHO.*` call.** All 28 API methods, CONFIG(3), META(6), HTTP(5), CONNECTION, CONNECTOR, FUNCTIONS, BLUEPRINT, WIZARD, UI + 4 sub-namespaces, EVENTS, ZDK.Client (v1.4 vs v1.5), $Client — exact signatures, response shapes, the "does not exist" table, and the places Zoho's docs contradict themselves |
| `references/error-handling.md` | ✅ | Payload-vs-rejection checking, envelope normalizer, empty-vs-permission-denied, retries, init failures, shape traps |
| `references/security.md` | ✅ | Secrets via `getOrgVariable`, Connections over stored keys, XSS, CSP, permission gating, audit logging, compliance |
| `references/troubleshooting.md` | ✅ | Widget not loading, `ZOHO` undefined, silent event typos, calls that fail, CORS, i18n + locale, performance |
| `references/workflow.md` | ⚠️ | Dev workflow — `zet run`, cert trust, widget sandbox, `zet validate` errors, `zet pack` output, Sigma upload, cloud_run |
| `references/manifest.md` | ⚠️ | `plugin-manifest.json` schema — service, widgets, connectors, config fields, locations, validation rules |

✅ = transcribed from Zoho's published JSDoc dataset and cross-checked against the shipped SDK
bundle. ⚠️ = not yet verified against an authoritative source; treat specifics as provisional.

### Re-verifying the SDK reference

The doc site is a client-rendered SPA, so fetching a page's HTML yields only a shell — which is why
so much inaccurate third-party material about this SDK exists. Pull the dataset the pages render
from instead:

```bash
curl -s https://www.zohocrm.dev/fingerprint_config.json \
  | jq -r '.default["dxh-data-store"]["widget-sdk-1.5.json"]'
# then: https://www.zohocrm.dev/dxh-data-store/widget-sdk-1.5_<hash>_.json
```

Doc page URLs use **dots**, not underscores (`/explore/widgets/v1.5/ZOHO.CRM.API`). `$Client` is
documented on the `ZDK.Client` page. The changelog route is plural (`/changelogs`).

## Triggers

Use this skill for: "build a CRM widget", "create a widget", "CRM extension", "Zoho widget", "zet init", "zet run", "plugin-manifest.json", "ZOHO.CRM.API", "ZOHO.embeddedApp", "ZDK.Client", "widget location", "widget SDK", "embed a widget in CRM", "zoho-extension-toolkit", or any question about Zoho CRM widget development, ZET CLI commands, or the CRM JS SDK.

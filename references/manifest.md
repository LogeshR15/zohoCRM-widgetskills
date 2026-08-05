# plugin-manifest.json — Schema Reference

> **⚠ Verification status: UNVERIFIED.** Unlike `references/sdk.md`, the contents of this file have
> **not** been checked against an authoritative Zoho source. The widget `location` strings, config
> field rules, `cspDomains` keys, and `zet validate` limits below should be confirmed against the
> Sigma / ZET manifest documentation before you rely on them. Treat the `location` table in
> particular as a starting point — verify each string against your CRM build.
>
> Verified-source method, for whoever picks this up: the SDK reference was rebuilt from Zoho's own
> JSON data store (see the provenance note at the top of `references/sdk.md`). The manifest schema
> lives in a different doc property and needs its own equivalent pass.

## Minimal valid CRM manifest

```json
{
  "service": "CRM",
  "modules": {
    "widgets": [
      {
        "location": "crm.record.detail",
        "url": "app/widget.html"
      }
    ],
    "connectors": []
  },
  "config": []
}
```

## Full schema

### Top-level keys

| Key | Type | Required | Notes |
|-----|------|----------|-------|
| `service` | string | Yes | Must be `"CRM"` exactly (case-sensitive) |
| `modules` | object | Yes | Must contain both `widgets` and `connectors` keys |
| `config` | array | Yes | Can be empty `[]`; holds install-time config parameters |
| `cspDomains` | object | No | Content Security Policy — domains widget can load resources from |

### modules.widgets[]

Each entry in `modules.widgets` must have both `location` and `url`. Both can't be empty strings.

| Key | Type | Required | Notes |
|-----|------|----------|-------|
| `location` | string | Yes | Widget placement in CRM UI (see location table below) |
| `url` | string | Yes | Relative path to HTML entry point, e.g. `"app/widget.html"` |

### Widget locations

| Location string | Where it appears in CRM |
|----------------|------------------------|
| `crm.record.detail` | Record detail page — sidebar panel |
| `crm.record.detail.related` | Related list section in record detail view |
| `crm.record.list` | List view — toolbar button |
| `crm.record.create` | Create record form |
| `crm.homepage` | CRM Homepage — draggable widget tile |
| `crm.settings` | Setup / Settings page tab |
| `crm.record.detail.button` | Custom button inside the record detail view |

Multiple widgets can be registered by adding more entries to the array.

### modules.connectors[]

Connectors link to Zoho Connections (OAuth integrations). Can be empty.

| Key | Type | Required | Notes |
|-----|------|----------|-------|
| `name` | string | Yes | Internal connector name |
| `service` | string | Yes | Zoho service identifier (e.g. `"zoho_sheet"`, `"zoho_mail"`) |

### config[]

Config parameters are shown to the admin at install time. Two variants:

**Static config** (value hardcoded, not shown to admin):
```json
{
  "name": "api_endpoint",
  "value": "https://api.example.com"
}
```

**User-defined config** (admin fills in at install):
```json
{
  "name": "api_key",
  "userdefined": true,
  "type": "text",
  "default": "Enter your API key here",
  "mandatory": true,
  "secure": true
}
```

#### User-defined config fields

| Key | Type | Required | Notes |
|-----|------|----------|-------|
| `name` | string | Yes | Config key name (read at runtime via `ZOHO.CRM.API.getOrgVariable` — **not** `CONFIG.getParameter`, which does not exist) |
| `userdefined` | boolean | Yes | Must be `true` for admin-configurable params |
| `type` | string | Yes | Input type: `"text"`, `"select"`, `"checkbox"`, `"multiselectbox"`, `"password"` |
| `default` | string | Yes | Placeholder or default value shown in install UI |
| `mandatory` | boolean | Yes | Whether admin must fill it before install completes |
| `secure` | boolean | Yes | If `true`, value is encrypted at rest — use for secrets/tokens |
| `options` | array | For `select`/`checkbox` | List of choice strings, e.g. `["Option A", "Option B"]` |

### cspDomains (optional)

```json
{
  "cspDomains": {
    "script-src": ["https://cdn.example.com"],
    "style-src": ["https://fonts.googleapis.com"],
    "img-src": ["https://images.example.com"],
    "connect-src": ["https://api.example.com"]
  }
}
```

## Full example with all keys

```json
{
  "service": "CRM",
  "modules": {
    "widgets": [
      {
        "location": "crm.record.detail",
        "url": "app/widget.html"
      },
      {
        "location": "crm.record.list",
        "url": "app/list-widget.html"
      }
    ],
    "connectors": [
      {
        "name": "my_connector",
        "service": "zoho_sheet"
      }
    ]
  },
  "config": [
    {
      "name": "base_url",
      "value": "https://api.myservice.com"
    },
    {
      "name": "auth_token",
      "userdefined": true,
      "type": "text",
      "default": "Paste your auth token",
      "mandatory": true,
      "secure": true
    },
    {
      "name": "sync_mode",
      "userdefined": true,
      "type": "select",
      "default": "auto",
      "mandatory": false,
      "secure": false,
      "options": ["auto", "manual", "scheduled"]
    }
  ],
  "cspDomains": {
    "connect-src": ["https://api.myservice.com"]
  }
}
```

## Validation rules (zet validate)

- `service` must be `"CRM"` (not lowercase, not missing)
- `modules` must have both `widgets` key and `connectors` key — even if empty arrays
- Each widget must have non-empty `location` and `url`
- Each connector must have non-empty `name` and `service`
- User-defined config must have: `type`, `default`, `mandatory` (boolean), `secure` (boolean)
- `select`/`checkbox` config must have non-empty `options` array
- Max 250 files, max total size 20 MB, max individual file size 5 MB
- Allowed extensions: `.html`, `.js`, `.css`, `.json`, `.txt`, `.md`, `.xml`, `.jpg`, `.png`, `.gif`, `.svg`, `.mp3`, `.mp4`, `.woff`, `.woff2`, `.ttf`, `.eot`

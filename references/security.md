# Security Best Practices for Zoho CRM Widgets

## 1. Protect Secrets & API Keys

### DON'T: Hardcode secrets in widget HTML/JS

```javascript
// ❌ NEVER DO THIS
const apiKey = "sk_live_1a2b3c4d5e6f7g8h";
const dbPassword = "MySecurePassword123";
```

### DO: Use secure config parameters

In `plugin-manifest.json`:
```json
{
  "config": [
    {
      "name": "api_key",
      "userdefined": true,
      "type": "password",
      "default": "Paste your API key",
      "mandatory": true,
      "secure": true
    },
    {
      "name": "webhook_url",
      "userdefined": true,
      "type": "text",
      "default": "https://your-service.com/webhook",
      "mandatory": true,
      "secure": true
    }
  ]
}
```

In widget — the read method is `ZOHO.CRM.API.getOrgVariable`:

```javascript
// Admin enters the key at install time; widget reads it at runtime.
// NOTE: getOrgVariable publishes no parameter table. Two calling forms exist,
// and their response casing differs — handle both.
ZOHO.CRM.API.getOrgVariable("api_key")
  .then(function (res) {
    var s = res && res.Success;
    var apiKey = s ? (s.Content !== undefined ? s.Content : s.content) : null;
    if (!apiKey) throw new Error("api_key not configured");
    return makeAuthenticatedRequest(apiKey);   // keep in memory only
  });

// Batch form — returns { Success: { content: { key: { value } } } }
ZOHO.CRM.API.getOrgVariable({ apiKeys: ["api_key", "webhook_url"] })
  .then(function (res) {
    var c = res && res.Success && (res.Success.content || res.Success.Content) || {};
    var apiKey = c.api_key && c.api_key.value;
  });
```

> There is **no `ZOHO.CRM.CONFIG.getParameter`** — that method does not exist in any SDK version.
> `ZOHO.CRM.CONFIG` has exactly three methods (`getCurrentUser`, `getOrgInfo`,
> `getUserPreference`). See `references/sdk.md`.

### Best practices for secrets

- **`secure: true`** — CRM encrypts the value at rest (database)
- **Type `password`** — CRM masks input; value is still plain text in memory
- **Never log secrets** — Exclude from error reports, console output, CSP headers
- **Rotate regularly** — Prompt admins to update credentials periodically
- **Use short-lived tokens** — Prefer temporary access tokens over permanent API keys
- **Don't pass a secret to `HTTP.*` if a Connection can do it** — `ZOHO.CRM.CONNECTION.invoke`
  and `ZOHO.CRM.CONNECTOR.invokeAPI` keep OAuth credentials server-side, so the token never
  enters widget JS at all. Prefer them over hand-managed keys.

## 2. Input Validation & XSS Prevention

### DON'T: Render untrusted data directly

```javascript
// ❌ VULNERABLE
const userData = response.data[0].Last_Name;
document.getElementById("content").innerHTML = userData;  // XSS risk
```

### DO: Escape or use safe DOM methods

```javascript
// ✓ SAFE — Escape HTML entities
function escapeHtml(text) {
  const div = document.createElement("div");
  div.textContent = text;
  return div.innerHTML;
}

const userData = response.data[0].Last_Name;
document.getElementById("content").innerHTML = escapeHtml(userData);

// ✓ OR — Use textContent for plain text
document.getElementById("content").textContent = userData;

// ✓ OR — Use DOM APIs (createElement, appendChild)
const el = document.createElement("span");
el.textContent = userData;
document.getElementById("content").appendChild(el);
```

### Template escaping

```javascript
// ❌ VULNERABLE
const html = `<h1>${userName}</h1>`;

// ✓ SAFE
function renderTemplate(name) {
  const div = document.createElement("div");
  div.innerHTML = `<h1>${escapeHtml(name)}</h1>`;
  return div;
}
```

### Validate external data

```javascript
function validateEmail(email) {
  const re = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return re.test(email);
}

function validatePhone(phone) {
  return /^\d{1,15}$/.test(phone.replace(/[\s\-\(\)]/g, ""));
}

// Validate before passing to CRM API
if (!validateEmail(userData.Email)) {
  throw new Error("Invalid email format");
}
```

## 3. Content Security Policy (CSP) & CORS

### Whitelist external domains

```json
{
  "cspDomains": {
    "script-src": ["https://cdn.example.com"],
    "style-src": ["https://fonts.googleapis.com"],
    "img-src": ["https://images.example.com", "data:"],
    "connect-src": ["https://api.example.com", "https://webhook.example.com"]
  }
}
```

### Minimize whitelisting

- Only add domains you absolutely need
- Use HTTPS for all external URLs
- Avoid wildcard domains (`https://*`)
- Reach for a Connection/Connector before adding a domain plus a stored key

### Prefer the SDK over raw `fetch`

A browser `fetch` from the widget iframe is subject to the remote server's CORS policy, which you
usually don't control. The SDK's request methods are mediated by CRM, which is what lets them
attach credentials at all:

```javascript
// ❌ Fails whenever the remote server doesn't send permissive CORS headers
fetch("https://external-api.com/data").then(r => r.json());

// ✓ Mediated by CRM — and resolves the RAW remote response, not a {data:…} envelope
ZOHO.CRM.HTTP.get({
  url: "https://external-api.com/data",
  headers: { Accept: "application/json" }
}).then(function (res) {
  var payload = typeof res === "string" ? JSON.parse(res) : res;
});

// ✓✓ Best when the target needs auth — the OAuth token never enters widget JS
ZOHO.CRM.CONNECTION.invoke("my_connection", {
  url: "https://external-api.com/data",
  method: "GET",
  param_type: 1
});
```

> Whether `ZOHO.CRM.HTTP.*` targets must also appear in `cspDomains.connect-src` is not stated in
> the SDK reference. Treat listing them as the safe default — it costs nothing and removes a
> whole class of silent failure.

## 4. Authentication & Token Management

### Use CRM OAuth, not manual auth

```javascript
// ❌ Store user credentials
const username = "admin@company.com";
const password = "password123";

// ✓ Leverage CRM's built-in auth
// ZOHO.CRM.API calls automatically use the logged-in user's token
ZOHO.CRM.API.getRecord({ ... });
```

### Respect user permissions

Calls run as the logged-in user and are bound by their module and field permissions. The SDK
documents **no error codes**, and a permission denial is not guaranteed to reject — check the
resolved payload:

```javascript
ZOHO.CRM.API.updateRecord({
  Entity: "Accounts",
  APIData: { id: recordId, Annual_Revenue: 1000000 }
})
.then(function (res) {
  var entry = res && res.data && res.data[0];
  if (!entry || String(entry.status).toLowerCase() !== "success") {
    // Covers permission denial, validation failure, and locked records alike —
    // the SDK does not let you distinguish them
    showError(entry && entry.message || "Update was not applied");
    return;
  }
  showSuccess();
})
.catch(function (err) {
  console.error("updateRecord rejected:", err);   // reject shape is undocumented
  showError("Update failed");
});
```

Don't offer an action the user can't perform. Gate the UI on their profile instead of letting the
write fail:

```javascript
ZOHO.CRM.CONFIG.getCurrentUser().then(function (user) {
  // user is a FLAT object: { id, full_name, email, profile: {name, id}, role: {...}, status }
  if (user.profile && user.profile.name !== "Administrator") hideAdminControls();
});
```

### Avoid storing tokens in localStorage

```javascript
// ❌ INSECURE
localStorage.setItem("crm_token", accessToken);

// ✓ Safe — CRM manages the token
// Just use ZOHO.CRM.API directly; token is managed by the sandbox
```

## 5. Secure Communication

### Always use HTTPS

```javascript
// ❌ Development with HTTP
const url = "http://127.0.0.1:5000/app/widget.html";

// ✓ Production with HTTPS
const url = "https://my-widget-service.example.com/app/widget.html";
```

### Sign webhook payloads

If your widget receives webhooks from an external service:

```javascript
// External service signs the payload with a secret
// Your backend verifies the signature

function verifySignature(payload, signature, secret) {
  const crypto = require("crypto");
  const hash = crypto
    .createHmac("sha256", secret)
    .update(JSON.stringify(payload))
    .digest("hex");
  
  return hash === signature;
}
```

## 6. Rate Limiting & Abuse Prevention

### Implement client-side rate limits

```javascript
const apiCallQueue = [];
const maxConcurrent = 3;
let activeCount = 0;

function queued(fn) {
  return new Promise(function(resolve, reject) {
    apiCallQueue.push({ fn, resolve, reject });
    processQueue();
  });
}

function processQueue() {
  while (activeCount < maxConcurrent && apiCallQueue.length > 0) {
    const { fn, resolve, reject } = apiCallQueue.shift();
    activeCount++;
    
    Promise.resolve(fn())
      .then(resolve)
      .catch(reject)
      .finally(function() {
        activeCount--;
        processQueue();
      });
  }
}

// Usage
queued(() => ZOHO.CRM.API.getRecord(...));
queued(() => ZOHO.CRM.API.updateRecord(...));
```

### Handle rate limiting gracefully

Widget calls share the CRM REST API's rate limits, but the SDK does not document how a limit
surfaces — so retry on any failure with a bounded count rather than matching a code you can't
verify:

```javascript
function makeApiCall(fn, maxRetries = 3, delay = 1000) {
  return fn().catch(function (err) {
    if (maxRetries <= 1) throw err;
    return new Promise(function (resolve) {
      setTimeout(() => resolve(makeApiCall(fn, maxRetries - 1, delay * 2)), delay);
    });
  });
}
```

Prefer fewer, larger calls over many small ones — `getAllRecords` with `per_page` beats N
`getRecord` calls, and `coql` beats client-side filtering.

## 7. Data Privacy & Compliance

### Minimize data collection

- Don't cache sensitive data (email, phone, financial info)
- Clear sensitive data from memory after use
- Never put record data in a URL, a log payload, or an external analytics call

**Field-level minimization is not available on `getRecord`.** There is no `Fields` parameter on
`getRecord`, `getAllRecords`, or `searchRecord` — the SDK returns the full record whether you want
it or not. So minimize on the way *out*, not on the way in:

```javascript
// getRecord returns everything — project down before doing anything else with it
ZOHO.CRM.API.getRecord({ Entity: "Contacts", RecordID: id })
  .then(function (res) {
    var full = res.data[0];
    var safe = { name: full.Last_Name, company: full.Account_Name };   // drop the rest
    render(safe);        // never pass `full` to logging, analytics, or an external POST
  });
```

If you genuinely need a narrow projection server-side, `coql` accepts an explicit select list:

```javascript
ZOHO.CRM.API.coql({
  select_query: "select Last_Name, Account_Name from Contacts where id = " + id
});
```

### Comply with local laws

- **GDPR** — Clear personal data on user request; log data processing
- **CCPA** — Enable data export/deletion; document data usage
- **HIPAA** — Don't store PHI in plain text; use encryption

### Audit logging

```javascript
// Resolve the acting user once at PageLoad and reuse it.
// CONFIG.getCurrentUser() resolves a FLAT object — not wrapped in users[].
let actor = null;
ZOHO.CRM.CONFIG.getCurrentUser().then(function (user) {
  actor = { id: user.id, email: user.email };
});

function logAction(action, entity, recordId) {
  return ZOHO.CRM.HTTP.post({
    url: "https://audit-log-service.com/log",   // must be in cspDomains.connect-src
    headers: { "Content-Type": "application/json" },
    body: {                                     // HTTP.* takes an Object, not a JSON string
      timestamp: new Date().toISOString(),
      action: action,                           // "view" | "edit" | "delete"
      entity: entity,
      recordId: recordId,
      userId: actor && actor.id
      // Deliberately no field values — log WHAT was touched, never the contents
    }
  }).catch(function () { /* audit logging must never break the widget */ });
}

// Usage
logAction("view", "Contacts", recordId, { fields_accessed: ["Email", "Phone"] });
```

## 8. Dependency Management

### Avoid eval() and dynamic code

```javascript
// ❌ DANGEROUS
eval(userInput);
new Function(userInput)();
```

### Pin dependency versions

In your widget project:
```json
{
  "dependencies": {
    "lodash": "4.17.21",
    "axios": "1.4.0"
  }
}
```

### Keep dependencies updated

```bash
npm audit        # Check for known vulnerabilities
npm audit fix    # Auto-fix where possible
npm outdated     # See available updates
```

## 9. Testing & Security Scanning

### Manual security checklist

- [ ] No hardcoded secrets (API keys, passwords, tokens)
- [ ] All external URLs use HTTPS
- [ ] Whitelisted only necessary domains in CSP
- [ ] User input is escaped before rendering
- [ ] Errors don't expose sensitive information
- [ ] Secrets are stored in secure config (not manifest)
- [ ] Rate limiting & abuse prevention in place
- [ ] Audit logging for sensitive operations

### Automated scanning

```bash
# Check for common security issues
npm install -g snyk
snyk test

# Audit npm dependencies
npm audit

# Code scanning (if using GitHub)
# Settings → Code security → Enable GitHub Advanced Security
```

## 10. Incident Response

### If a secret is exposed

1. **Immediately rotate** — Update the API key in the manifest config
2. **Revoke old credentials** — In your external service, disable the old key
3. **Audit logs** — Check if the key was used maliciously
4. **Notify users** — If their data may be affected
5. **Update plugin** — Redeploy the widget to CRM

### Report a vulnerability

If you discover a security vulnerability in this skill, email the maintainer with:
- Description of the vulnerability
- How to reproduce it
- Suggested fix
- Your contact info (optional)

Do NOT open a public GitHub issue for security vulnerabilities.

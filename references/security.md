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

In widget:
```javascript
// Admin enters the key at install time; widget retrieves it safely
ZOHO.CRM.CONFIG.getParameter("api_key")
  .then(function(apiKey) {
    // Use apiKey only in memory — never log or expose it
    return makeAuthenticatedRequest(apiKey);
  });
```

### Best practices for secrets

- **`secure: true`** — CRM encrypts the value at rest (database)
- **Type `password`** — CRM masks input; value is still plain text in memory
- **Never log secrets** — Exclude from error reports, console output, CSP headers
- **Rotate regularly** — Prompt admins to update credentials periodically
- **Use short-lived tokens** — Prefer temporary access tokens over permanent API keys

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
- Prefer same-origin requests (communicate via ZOHO.CRM.HTTP which is always allowed)
- Use HTTPS for all external URLs
- Avoid wildcard domains (`https://*`)

### ZOHO.CRM.HTTP vs CORS

```javascript
// ❌ This may fail due to CORS (blocked by browser)
fetch("https://external-api.com/data")
  .then(r => r.json());

// ✓ This respects CSP and works
ZOHO.CRM.HTTP.get({
  url: "https://external-api.com/data",
  headers: { "Accept": "application/json" }
})
.then(r => r.data);
```

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

```javascript
// Always handle permission errors gracefully
ZOHO.CRM.API.updateRecord({
  Entity: "Accounts",
  APIData: { id: recordId, Annual_Revenue: 1000000 }
})
.catch(function(err) {
  if (err.code === 1002 || err.code === 1003) {
    // User lacks edit permission
    showError("You don't have permission to edit this record");
    return;
  }
  throw err;
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

```javascript
function makeApiCall(fn, maxRetries = 3, delay = 1000) {
  return fn().catch(function(err) {
    if (err.code === 429 && maxRetries > 0) {
      // Rate limited — backoff and retry
      return new Promise(function(resolve) {
        setTimeout(() => {
          resolve(makeApiCall(fn, maxRetries - 1, delay * 2));
        }, delay);
      });
    }
    throw err;
  });
}
```

## 7. Data Privacy & Compliance

### Minimize data collection

- Only request fields your widget actually needs
- Don't cache sensitive data (email, phone, financial info)
- Clear sensitive data from memory after use

```javascript
// ✓ Request only what you need
ZOHO.CRM.API.getRecord({
  Entity: "Contacts",
  RecordID: id,
  Fields: ["Last_Name", "Company"]  // Not requesting Email, Phone, etc.
});
```

### Comply with local laws

- **GDPR** — Clear personal data on user request; log data processing
- **CCPA** — Enable data export/deletion; document data usage
- **HIPAA** — Don't store PHI in plain text; use encryption

### Audit logging

```javascript
function logAction(action, entity, recordId, details) {
  const entry = {
    timestamp: new Date().toISOString(),
    action: action,           // "view", "edit", "delete"
    entity: entity,
    recordId: recordId,
    userId: getCurrentUserId(),
    details: details
  };
  
  // Send to secure logging service
  return ZOHO.CRM.HTTP.post({
    url: "https://audit-log-service.com/log",
    body: JSON.stringify(entry)
  });
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

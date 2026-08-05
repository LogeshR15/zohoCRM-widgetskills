# Troubleshooting Guide for Zoho CRM Widgets

## Widget Not Loading

### Symptom: Blank widget or "Loading…" persists

**Check 1: SDK script is present**
```html
<!-- Required in widget HTML <head> -->
<script src="https://live.zwidgets.com/js-sdk/1.0/ZohoEmbededAppSDK.min.js"></script>
```

**Check 2: Initialization is called at script load time**
```javascript
// ❌ DON'T wait for DOMContentLoaded
document.addEventListener("DOMContentLoaded", function() {
  ZOHO.embeddedApp.init();  // Too late
});

// ✓ DO call init immediately after registering the event handler
ZOHO.embeddedApp.on("PageLoad", function(data) {
  // Handle page load
});
ZOHO.embeddedApp.init();  // At script load time
```

**Check 3: Browser console for errors**
1. Right-click inside the widget iframe → Inspect
2. Open DevTools Console tab
3. Look for red error messages
4. Common errors:
   - `Uncaught ReferenceError: ZOHO is not defined` — SDK script didn't load
   - `Cannot read property 'init' of undefined` — SDK script missing or blocked

**Check 4: Network tab for failed requests**
1. DevTools → Network tab
2. Reload the page
3. Look for failed requests (red icons)
4. Check for 404 (not found) or 403 (forbidden) on:
   - The widget HTML file
   - The SDK script (`ZohoEmbededAppSDK.min.js`)
   - Any external CSS/JS resources

**Fix**: Ensure `dev server is running` (`zet run`) and widget URL is correct in CRM.

---

## "ZOHO is undefined" Error

### Symptom: Console shows `Uncaught ReferenceError: ZOHO is not defined`

**Cause**: Widget code runs before the SDK script loads.

**Solution**:
```javascript
// ❌ This runs before SDK loads
ZOHO.embeddedApp.init();

// ✓ Wrap in a check
function doWork() {
  if (typeof ZOHO === "undefined") {
    // SDK not loaded yet, retry after delay
    setTimeout(doWork, 100);
    return;
  }
  ZOHO.CRM.API.getRecord({ ... });
}

// ✓ OR — Ensure SDK script is the first <script> in <head>
<head>
  <script src="https://live.zwidgets.com/js-sdk/1.0/ZohoEmbededAppSDK.min.js"></script>
  <script src="your-app.js"></script>  <!-- Your code runs after SDK loads -->
</head>
```

---

## API Calls Not Working

### Symptom: `ZOHO.CRM.API.getRecord()` doesn't return data

**Check 1: You're inside a PageLoad event**
```javascript
// ❌ Calling API at the top level
ZOHO.CRM.API.getRecord({ ... });  // Fails silently

// ✓ Calling API inside PageLoad handler
ZOHO.embeddedApp.on("PageLoad", function(data) {
  ZOHO.CRM.API.getRecord({
    Entity: data.Entity,
    RecordID: data.EntityId
  }).then(function(response) {
    console.log(response.data);
  });
});
```

**Check 2: Module API name is correct**
```javascript
// ❌ Using display name (what users see in CRM)
ZOHO.CRM.API.getRecord({
  Entity: "Lead",  // ❌ Wrong
  RecordID: id
});

// ✓ Using API name (from CRM setup)
ZOHO.CRM.API.getRecord({
  Entity: "Leads",  // ✓ Correct
  RecordID: id
});
```

**To find the correct API name**:
1. In CRM, go to **Setup → Modules**
2. Hover over the module name → note the API name in tooltip
3. Or use MCP tools: `mcp__ZohoCRM__get_crm_record` to check live modules

**Check 3: Error handling in place**
```javascript
ZOHO.CRM.API.getRecord({ ... })
  .then(function(response) {
    console.log("Success:", response);
  })
  .catch(function(err) {
    console.error("Error:", err.code, err.message);
    // Now you'll see what went wrong
  });
```

**Common API errors**:
- `code: 1001` — Record locked or doesn't exist
- `code: 1002` — No permission to read
- `code: 1003` — Field doesn't exist or no permission
- `code: 1004` — Invalid module name
- `401` — Token expired, reload widget

---

## CORS / CSP Errors

### Symptom: Failed to fetch data from external API

Console error: `Blocked by Content Security Policy` or `No 'Access-Control-Allow-Origin'`

**Solution**: Whitelist domain in `plugin-manifest.json`

```json
{
  "cspDomains": {
    "connect-src": ["https://api.example.com"],
    "script-src": ["https://cdn.example.com"],
    "img-src": ["https://images.example.com"]
  }
}
```

Then redeploy and test:
```bash
zet validate
zet pack
# Upload dist/widget.zip to CRM
```

**Note**: You don't need to whitelist `ZOHO.CRM.HTTP` calls — they're always allowed.

---

## Widget Appears But Data Is Blank

### Symptom: Widget loads, but shows no data

**Check 1: Are errors being caught silently?**
```javascript
// ❌ No error handling
ZOHO.CRM.API.getRecord({ ... })
  .then(function(response) {
    document.getElementById("content").innerHTML = response.data[0].Last_Name;
  });

// ✓ With error handling
ZOHO.CRM.API.getRecord({ ... })
  .then(function(response) {
    if (!response.data || response.data.length === 0) {
      throw new Error("No record found");
    }
    document.getElementById("content").innerHTML = response.data[0].Last_Name;
  })
  .catch(function(err) {
    console.error("Failed to load:", err);
    document.getElementById("content").innerHTML = "Error: " + err.message;
  });
```

**Check 2: Is the response data structure correct?**
```javascript
ZOHO.CRM.API.getRecord({ Entity: "Leads", RecordID: id })
  .then(function(response) {
    console.log("Full response:", response);
    console.log("Data array:", response.data);
    console.log("First record:", response.data[0]);
    // Now inspect the actual structure in DevTools
  });
```

**Check 3: Field API names**
```javascript
// ❌ Using display name
var name = response.data[0].Last Name;  // Spaces don't work in JS

// ✓ Using API name (underscores or camelCase)
var name = response.data[0].Last_Name;  // ✓ Correct
var name = response.data[0].last_name;  // May also work depending on config
```

---

## "Record not found" After Filtering

### Symptom: Widget works on some records, not others

**Likely cause**: User doesn't have read permission on those records.

**Solution**: Handle gracefully
```javascript
ZOHO.CRM.API.getRecord({ ... })
  .then(function(response) {
    if (!response.data || response.data.length === 0) {
      document.getElementById("content").innerHTML =
        "No data available. You may not have permission to view this record.";
      return;
    }
    render(response.data[0]);
  });
```

---

## Dev Server Issues

### Symptom: `zet run` fails or hangs

**Check 1: Port 5000 is available**
```bash
lsof -i :5000  # Check what's using port 5000

# If another process uses it, either:
# 1. Kill the process: kill -9 <PID>
# 2. Use a different port: Edit server/index.js, change port to 5001
```

**Check 2: ZET CLI is up to date**
```bash
zet --version
npm install -g zoho-extension-toolkit@latest
```

**Check 3: node_modules are installed**
```bash
npm install
zet run
```

**Check 4: Manifest is valid**
```bash
zet validate
```

---

## "Cert Not Trusted" When Opening https://127.0.0.1:5000

### Symptom: Browser warning about self-signed certificate

**This is normal** for local development. The dev server generates a self-signed cert.

**Solution**:
1. Visit `https://127.0.0.1:5000` in your browser
2. Click **Advanced** → **Proceed to 127.0.0.1 (unsafe)**
3. You should see the app directory listing (confirming the cert is now trusted)
4. Now register the widget in CRM using `https://127.0.0.1:5000/app/widget.html`

For production, the widget URL must use a **real HTTPS certificate** (not self-signed).

---

## Changes Not Reflecting in CRM

### Symptom: Code is updated but CRM shows old version

**Check 1: Hard refresh the page**
- In CRM, press **Ctrl+Shift+R** (Windows) or **Cmd+Shift+R** (Mac)
- This clears browser cache and reloads

**Check 2: Is `zet run` still running?**
```bash
# Check if dev server is running
lsof -i :5000

# If not, restart it
zet run
```

**Check 3: Did you save the file?**
- Ensure your editor saved changes to the file
- Check file modification time: `ls -l app/widget.html`

**Check 4: For HTML changes, reload CRM page entirely**
- Navigate away from the record, then back
- Or close and reopen the widget popup

---

## Translation (i18n) Not Working

### Symptom: Widget shows keys like `widget.title` instead of translated text

**Check 1: Translations file exists**
```
app/
  translations/
    en.json    ← Must exist
```

**Check 2: File is valid JSON**
```bash
cat app/translations/en.json
# Should parse without errors
```

**Check 3: Accessing translations correctly**
```javascript
// ❌ Won't work — ZOHO.CRM.META doesn't auto-translate
const label = ZOHO.CRM.META.translate("widget.title");

// ✓ Fetch translations manually
fetch("app/translations/en.json")
  .then(r => r.json())
  .then(translations => {
    document.getElementById("title").textContent = translations.widget_title;
  });
```

---

## Multiple Widgets on Same Record

### Symptom: Multiple widgets load, but they interfere with each other

**Solution**: Namespace your event listeners
```javascript
// ❌ Generic event names may conflict
document.addEventListener("custom-event", handler);

// ✓ Use widget-specific names
document.addEventListener("widget-abc-custom-event", handler);
ZOHO.CRM.$Client.trigger("widget-abc-custom-event", data);
```

---

## Widget Performance Slow

### Symptom: Widget is sluggish, takes time to render

**Check 1: How many API calls are you making?**
```javascript
// ❌ This makes 100 sequential API calls
records.forEach(function(record) {
  ZOHO.CRM.API.getRecord({ RecordID: record.id });  // Slow!
});

// ✓ Batch into a single call or use getAllRecords
ZOHO.CRM.API.getAllRecords({ Entity: "Contacts", per_page: 200 });
```

**Check 2: Are you caching results?**
```javascript
const cache = {};

function getCachedRecord(id) {
  if (cache[id]) return Promise.resolve(cache[id]);
  
  return ZOHO.CRM.API.getRecord({ RecordID: id })
    .then(function(response) {
      cache[id] = response.data[0];
      return cache[id];
    });
}
```

**Check 3: Debounce rapid API calls**
```javascript
function debounce(fn, delay) {
  let timeoutId;
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}

const debouncedSearch = debounce(function(query) {
  ZOHO.CRM.API.searchRecord({ Type: "word", Query: query });
}, 500);
```

---

## Getting Help

If you've tried the above and still have issues:

1. **Check the browser console** (DevTools → Console)
2. **Enable debug logging**:
   ```javascript
   console.log("Widget loaded");
   console.log("PageLoad data:", data);
   ```
3. **Review the full error** — don't just look at the first line
4. **Check ZET-debug.log** — in your widget project root:
   ```bash
   cat ZET-debug.log
   ```
5. **Consult the references**:
   - `references/sdk.md` — API method signatures
   - `references/manifest.md` — Manifest schema
   - `references/error-handling.md` — Error patterns
6. **Test with MCP tools** — If available, use `mcp__ZohoCRM__*` tools to verify live data

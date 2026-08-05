# Error Handling Patterns for Zoho CRM Widgets

## Common Error Scenarios

### 1. API Errors

All ZOHO.CRM.API calls reject with an error object: `{ code, message, details, status }`.

#### Handle missing records

```javascript
ZOHO.CRM.API.getRecord({
  Entity: "Leads",
  RecordID: data.EntityId
})
.then(function(response) {
  if (!response.data || response.data.length === 0) {
    // Record doesn't exist or user lacks permission
    showError("Record not found or no access");
    return;
  }
  const record = response.data[0];
  render(record);
})
.catch(function(err) {
  logError("getRecord failed", err);
  showError("Failed to load record: " + err.message);
});
```

#### Handle permission errors

```javascript
ZOHO.CRM.API.updateRecord({
  Entity: "Leads",
  APIData: { id: recordId, Status: "Qualified" }
})
.catch(function(err) {
  if (err.code === 1002 || err.code === 1003) {
    // Permission denied
    showError("You don't have permission to update this record");
  } else if (err.code === 1001) {
    // Record locked
    showError("Record is currently locked");
  } else {
    showError("Update failed: " + err.message);
  }
});
```

#### Retry with exponential backoff

```javascript
function retryWithBackoff(fn, maxRetries = 3, delay = 1000) {
  return fn().catch(function(err) {
    if (maxRetries > 0 && (err.code === 429 || err.status === 429)) {
      // Rate limited — wait and retry
      return new Promise(function(resolve, reject) {
        setTimeout(function() {
          resolve(retryWithBackoff(fn, maxRetries - 1, delay * 2));
        }, delay);
      });
    }
    return Promise.reject(err);
  });
}

// Usage
retryWithBackoff(function() {
  return ZOHO.CRM.API.getAllRecords({ Entity: "Contacts", per_page: 200 });
})
.then(function(response) { /* success */ })
.catch(function(err) { /* failed after retries */ });
```

### 2. SDK Not Available (Initialization)

The CRM SDK loads asynchronously. Protect against race conditions:

```javascript
// DON'T: Call API before init
ZOHO.CRM.API.getRecord(...);  // ❌ SDK may not be ready

// DO: Wait for PageLoad event
ZOHO.embeddedApp.on("PageLoad", function(data) {
  // Now ZOHO namespace is available
  ZOHO.CRM.API.getRecord(...);  // ✓ Safe
});
ZOHO.embeddedApp.init();
```

### 3. HTTP Request Errors

`ZOHO.CRM.HTTP` methods also reject on network or permission errors:

```javascript
ZOHO.CRM.HTTP.get({
  url: "https://api.example.com/data",
  parameters: { limit: 10 }
})
.then(function(response) {
  if (response.status >= 400) {
    // HTTP error status
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  return response.data;
})
.catch(function(err) {
  if (err.message.includes("net::ERR_NAME_NOT_RESOLVED")) {
    showError("Network error — check URL or CSP domains");
  } else if (err.message.includes("403") || err.message.includes("401")) {
    showError("Authentication failed");
  } else {
    showError("Request failed: " + err.message);
  }
});
```

### 4. CONFIG Parameter Errors

`ZOHO.CRM.CONFIG.getParameter` may fail if the parameter doesn't exist:

```javascript
ZOHO.CRM.CONFIG.getParameter("api_key")
.then(function(value) {
  if (!value || value.trim() === "") {
    showError("API key not configured. Contact admin.");
    return;
  }
  useApiKey(value);
})
.catch(function(err) {
  // Parameter not found in config
  showError("Config parameter missing: " + err.message);
});
```

### 5. Cross-Origin / CSP Errors

Widget iframe is sandboxed. External requests require CSP domains:

```javascript
// In plugin-manifest.json
{
  "cspDomains": {
    "connect-src": ["https://api.example.com", "https://cdn.example.com"]
  }
}

// In widget JS
ZOHO.CRM.HTTP.post({
  url: "https://api.example.com/webhook",  // ✓ Whitelisted
  body: JSON.stringify({ ... })
})
.catch(function(err) {
  // If error mentions CSP or CORS:
  // 1. Add domain to manifest cspDomains
  // 2. Redeploy and test
});
```

## Error Recovery Patterns

### Graceful degradation

```javascript
ZDK.Client.on("App.Load", function(data) {
  // Try to load enriched data; fall back to minimal view
  loadEnrichedData()
    .catch(function() {
      // Load just the record ID + name, skip enhancements
      return loadBasicData(data.EntityId);
    })
    .then(render);
});

function loadEnrichedData() {
  return ZOHO.CRM.API.getRecord({
    Entity: data.Entity,
    RecordID: data.EntityId,
    Fields: ["Last_Name", "Email", "Phone", "Company", "Custom_Field_1"]
  });
}

function loadBasicData(id) {
  return ZOHO.CRM.API.getRecord({
    Entity: data.Entity,
    RecordID: id,
    Fields: ["Last_Name"]
  });
}
```

### Error boundary (UI stability)

```javascript
function safeRender(data) {
  try {
    const html = buildHTML(data);
    document.getElementById("content").innerHTML = html;
  } catch (err) {
    console.error("Render failed:", err);
    document.getElementById("content").innerHTML =
      "<p style='color:red;'>Display error. Check browser console.</p>";
  }
}
```

### Queue failed operations

```javascript
const opQueue = [];

function queueOp(fn) {
  opQueue.push(fn);
  processQueue();
}

function processQueue() {
  if (opQueue.length === 0) return;
  
  const op = opQueue.shift();
  op()
    .then(function() {
      processQueue();  // Process next
    })
    .catch(function(err) {
      console.error("Operation failed:", err);
      opQueue.unshift(op);  // Retry later
      setTimeout(processQueue, 5000);
    });
}

// Usage
queueOp(function() {
  return ZOHO.CRM.API.updateRecord({ ... });
});
```

## Debugging with error codes

| Error Code | Meaning | Recovery |
|------------|---------|----------|
| 1001 | Record locked | Retry later or show message |
| 1002 | Insufficient permission | Check user role, skip operation |
| 1003 | Field validation failed | Show validation message to user |
| 1004 | Invalid module/field name | Verify manifest config |
| 1005 | Mandatory field missing | Show form error |
| 401 | Unauthorized (expired token) | Reload widget or re-auth |
| 403 | Forbidden (CSP/CORS) | Check manifest cspDomains |
| 429 | Rate limited | Use exponential backoff |
| 500+ | Server error | Log and retry with backoff |

## Monitoring & Logging

Send errors to an external service for analysis:

```javascript
function logWidgetError(context, err) {
  const payload = {
    timestamp: new Date().toISOString(),
    context: context,
    error: {
      code: err.code,
      message: err.message,
      stack: err.stack
    },
    userAgent: navigator.userAgent
  };
  
  ZOHO.CRM.HTTP.post({
    url: "https://your-logging-service.com/api/widget-errors",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload)
  }).catch(function() {
    // Logging itself failed — silent fail
    console.error("Failed to log error:", err);
  });
}
```

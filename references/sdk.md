# Zoho CRM JS SDK v1.5 — Full Reference

## SDK Script Tag

```html
<!-- Always the first script in <head> -->
<script src="https://live.zwidgets.com/js-sdk/1.0/ZohoEmbededAppSDK.min.js"></script>
```

---

## Initialization — REQUIRED before any API call

```javascript
// Pattern 1 — Classic (v1.0 compatible)
ZOHO.embeddedApp.on("PageLoad", function(data) {
  // data.Entity      — module name e.g. "Leads", "Contacts"
  // data.EntityId    — current record ID (detail/edit views)
  // data.ButtonPosition — "DetailView", "ListView", "QuickCreate"
  // data.Location    — widget location string
  doWork(data);
});
ZOHO.embeddedApp.init();

// Pattern 2 — ZDK.Client (v1.5, preferred for new widgets)
ZDK.Client.on("App.Load", function(data) {
  // same data shape as PageLoad
  doWork(data);
});
```

> **Rule:** Call `ZOHO.embeddedApp.init()` (or `ZDK.Client.on`) unconditionally at script load.
> Never gate it behind a DOM ready check — CRM injects the page load event before DOMContentLoaded.

---

## ZDK.Client

The modern top-level client introduced in v1.5.

```javascript
// Listen for events from CRM host
ZDK.Client.on(eventName, callback)

// Send a message to the parent CRM page (cross-origin postMessage wrapper)
ZDK.Client.trigger(eventName, data)

// Resize the widget iframe
ZDK.Client.resize({ height: 400 })           // px number or string
ZDK.Client.resize({ height: "400px" })

// Close the widget popup/panel
ZDK.Client.close()

// Read a config parameter defined in plugin-manifest.json
ZDK.Client.getConfig(paramName)              // returns Promise<string>

// Set a runtime config value (session-scoped, not persisted)
ZDK.Client.setConfig(paramName, value)
```

### ZDK.Client events

| Event | When it fires |
|-------|--------------|
| `"App.Load"` | Widget iframe fully loaded and CRM context injected |
| `"App.Redirect"` | User navigates away from current record |
| `"App.Message"` | Custom message sent from another widget via `ZDK.Client.trigger` |

---

## ZOHO.CRM.API

All methods return a `Promise`. Resolve value is always `{ data: [...] }` for list results,
or `{ data: [record] }` for single record results.

### getRecord

```javascript
ZOHO.CRM.API.getRecord({
  Entity: "Leads",       // module API name
  RecordID: "4895937000000123456",
  Fields: ["Last_Name", "Email", "Phone"]  // optional; omit for all fields
})
.then(function(response) {
  var record = response.data[0];
  console.log(record.Last_Name);
});
```

### getAllRecords

```javascript
ZOHO.CRM.API.getAllRecords({
  Entity: "Contacts",
  sort_order: "asc",        // "asc" | "desc"
  sort_by: "Last_Name",
  converted: "false",       // "true" | "false" | "both"
  approved: "true",
  page: 1,
  per_page: 200,            // max 200
  Fields: ["Last_Name", "Email"]
})
.then(function(response) {
  var records = response.data;
});
```

### searchRecord

```javascript
// Search by criteria (COQL-like)
ZOHO.CRM.API.searchRecord({
  Entity: "Leads",
  Type: "criteria",
  Query: "(Email:equals:user@example.com)",
  Fields: ["Last_Name", "Email", "Lead_Status"]
})
.then(function(response) { /* response.data */ });

// Search by phone
ZOHO.CRM.API.searchRecord({ Entity: "Contacts", Type: "phone", Query: "+919876543210" });

// Search by email
ZOHO.CRM.API.searchRecord({ Entity: "Contacts", Type: "email", Query: "user@example.com" });

// Full-text word search
ZOHO.CRM.API.searchRecord({ Entity: "Leads", Type: "word", Query: "Acme Corp" });
```

#### Criteria syntax

```
(FieldAPIName:operator:value)
AND / OR to combine: ((Field1:equals:A)and(Field2:contains:B))

Operators: equals, not_equals, starts_with, ends_with, contains,
           not_contains, is_empty, is_not_empty, greater_than,
           less_than, greater_equal, less_equal, between
```

### addRecord

```javascript
ZOHO.CRM.API.addRecord({
  Entity: "Leads",
  APIData: {
    Last_Name: "Smith",
    Company: "Acme Corp",
    Email: "smith@acme.com",
    Lead_Source: "Web Site"
  },
  Trigger: ["workflow", "approval", "blueprint"]  // optional
})
.then(function(response) {
  var newId = response.data[0].details.id;
});
```

### updateRecord

```javascript
ZOHO.CRM.API.updateRecord({
  Entity: "Leads",
  APIData: {
    id: "4895937000000123456",   // required
    Lead_Status: "Contacted",
    Phone: "+919876543210"
  },
  Trigger: ["workflow"]          // optional
})
.then(function(response) { /* response.data */ });
```

### deleteRecord

```javascript
ZOHO.CRM.API.deleteRecord({
  Entity: "Leads",
  RecordID: "4895937000000123456"
})
.then(function(response) { /* response.data[0].code === "SUCCESS" */ });
```

### getRelatedRecords

```javascript
ZOHO.CRM.API.getRelatedRecords({
  Entity: "Contacts",
  RecordID: data.EntityId,
  RelatedList: "Notes",
  page: 1,
  per_page: 20
})
.then(function(response) { var notes = response.data; });
```

### updateRelatedRecord

```javascript
ZOHO.CRM.API.updateRelatedRecord({
  Entity: "Contacts",
  RecordID: "4895937000000123456",
  RelatedList: "Notes",
  RelatedRecordID: "4895937000000789012",
  APIData: { Note_Content: "Updated note content" }
});
```

### deleteRelatedRecord

```javascript
ZOHO.CRM.API.deleteRelatedRecord({
  Entity: "Contacts",
  RecordID: "4895937000000123456",
  RelatedList: "Notes",
  RelatedRecordID: "4895937000000789012"
});
```

### getUser / getAllUsers

```javascript
ZOHO.CRM.API.getUser({ ID: "4895937000000111111" })
.then(function(response) { var user = response.users[0]; });

ZOHO.CRM.API.getAllUsers({ type: "ActiveUsers", page: 1, per_page: 200 })
.then(function(response) { var users = response.users; });
// type options: "AllUsers", "ActiveUsers", "DeactiveUsers", "ConfirmedUsers",
//               "NotConfirmedUsers", "DeletedUsers", "ActiveConfirmedUsers",
//               "AdminUsers", "ActiveConfirmedAdmins", "CurrentUser"
```

### getAllProfiles

```javascript
ZOHO.CRM.API.getAllProfiles()
.then(function(response) { var profiles = response.profiles; });
```

---

## ZOHO.CRM.CONFIG

Read metadata about the CRM org and module configuration.

### getFields

```javascript
ZOHO.CRM.CONFIG.getFields({ Entity: "Leads" })
.then(function(response) {
  var fields = response.fields;
  // Each field: { api_name, display_label, data_type, pick_list_values?, ... }
});
```

### getLayouts

```javascript
ZOHO.CRM.CONFIG.getLayouts({ Entity: "Leads" })
.then(function(response) { var layouts = response.layouts; });
```

### getModules

```javascript
ZOHO.CRM.CONFIG.getModules({})
.then(function(response) {
  var modules = response.modules;
  // Each: { api_name, module_name, singular_label, plural_label, ... }
});
```

### getRelatedList

```javascript
ZOHO.CRM.CONFIG.getRelatedList({ Entity: "Contacts" })
.then(function(response) { var relatedLists = response.related_lists; });
```

### getPickListValues

```javascript
ZOHO.CRM.CONFIG.getPickListValues({ Entity: "Leads", FieldAPIName: "Lead_Status" })
.then(function(response) {
  var values = response.pick_list_values;  // array of { display_value, actual_value }
});
```

### getCustomViews

```javascript
ZOHO.CRM.CONFIG.getCustomViews({ Entity: "Leads" })
.then(function(response) { var views = response.custom_views; });
```

### getParameter (install-time config)

```javascript
// Read a config value defined in plugin-manifest.json config[]
ZOHO.CRM.CONFIG.getParameter("api_key")
.then(function(value) { console.log(value); });
```

---

## ZOHO.CRM.META

Get runtime environment and context info.

```javascript
// Current DC, portal, and logged-in user
ZOHO.CRM.META.getEnvironment()
.then(function(response) {
  // response: { deployment, dc, portal: { id, name }, user: { id, name, email } }
  var dc = response.dc;         // "US", "EU", "IN", "AU", "JP", "CN"
  var userId = response.user.id;
});

// Portal (org) info
ZOHO.CRM.META.getPortalInfo()
.then(function(response) {
  // response: { portal_id, portal_name, org_id, ... }
});

// Widget placement context
ZOHO.CRM.META.getAppInfo()
.then(function(response) {
  // response: { location, entity, entity_id, page_type }
});
```

---

## ZOHO.CRM.HTTP

Make authenticated HTTP requests from the widget (uses CRM OAuth token automatically).

```javascript
// GET
ZOHO.CRM.HTTP.get({
  url: "https://www.zohoapis.com/crm/v8/Leads",
  parameters: { fields: "Last_Name,Email", per_page: 10 },
  headers: { "Content-Type": "application/json" }
})
.then(function(response) { console.log(response.data); });

// POST
ZOHO.CRM.HTTP.post({
  url: "https://www.zohoapis.com/crm/v8/Leads",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ data: [{ Last_Name: "Test", Company: "Acme" }] })
})
.then(function(response) { /* response.data */ });

// PUT
ZOHO.CRM.HTTP.put({ url, headers, body });

// DELETE
ZOHO.CRM.HTTP.delete({ url, headers });

// PATCH
ZOHO.CRM.HTTP.patch({ url, headers, body });
```

---

## ZOHO.CRM.CONNECTION

Make requests through a Zoho Connection (configured connector with OAuth).

```javascript
ZOHO.CRM.CONNECTION.makeRequest({
  url: "https://sheet.zoho.com/api/v2/spreadsheets",
  type: "GET",
  parameters: {},
  headers: {},
  connection: "my_connector"   // matches connector name in plugin-manifest.json
})
.then(function(response) { console.log(response.data); });
```

---

## ZOHO.CRM.CONNECTOR

Similar to CONNECTION — make requests via a registered connector (external OAuth services).

```javascript
ZOHO.CRM.CONNECTOR.makeRequest({
  url: "https://api.externalservice.com/data",
  type: "POST",
  parameters: { key: "value" },
  headers: { "Content-Type": "application/json" },
  connector: "my_connector"
})
.then(function(response) { /* response */ });
```

---

## ZOHO.CRM.FUNCTIONS

Execute a Deluge or serverless function from within the widget.

```javascript
ZOHO.CRM.FUNCTIONS.execute({
  api_name: "my_function_api_name",
  arguments: JSON.stringify({ param1: "value1", record_id: data.EntityId })
})
.then(function(response) {
  // response.details.output — stringified return value from the function
  var result = JSON.parse(response.details.output);
});
```

---

## ZOHO.CRM.BLUEPRINT

Interact with Blueprint transitions on a record.

### getTransitions

```javascript
ZOHO.CRM.BLUEPRINT.getTransitions({
  Entity: "Leads",
  RecordID: data.EntityId
})
.then(function(response) {
  var transitions = response.blueprint.transitions;
  // Each transition: { id, name, criteria_met, next_field_value }
});
```

### proceed

```javascript
ZOHO.CRM.BLUEPRINT.proceed({
  Entity: "Leads",
  RecordID: data.EntityId,
  TransitionId: "transition_id_here",
  Data: {
    // Any fields required by the transition
    Lead_Status: "Qualified",
    Description: "Transition note"
  }
})
.then(function(response) { /* response.code === "SUCCESS" */ });
```

---

## ZOHO.CRM.WIZARD

Trigger a wizard action from within a widget (used inside wizard widgets).

```javascript
ZOHO.CRM.WIZARD.proceed({
  button_id: "Next",
  current_fieldname_vs_value: {
    "Field_API_Name": "value",
    "Another_Field": "another_value"
  }
})
.then(function(response) { /* response */ });
```

---

## ZOHO.CRM.UI

Control the widget's container and navigate CRM from within the widget.

### Resize

```javascript
ZOHO.CRM.UI.Resize({ height: "400", width: "600" })   // strings, in px
```

### UI.Popup

```javascript
// Close the widget popup without reloading
ZOHO.CRM.UI.Popup.close();

// Close popup AND reload the parent CRM page
ZOHO.CRM.UI.Popup.closeReload();
```

### UI.Record

```javascript
// Open a record detail page
ZOHO.CRM.UI.Record.open({
  Entity: "Contacts",
  RecordID: "4895937000000123456"
});

// Open create record form (optionally pre-fill fields)
ZOHO.CRM.UI.Record.create({
  Entity: "Leads",
  data: { Last_Name: "Smith", Company: "Acme" }
});

// Open edit view for a record
ZOHO.CRM.UI.Record.edit({
  Entity: "Leads",
  RecordID: "4895937000000123456"
});
```

### UI.Widget

Open a secondary widget in a popup overlay from within the current widget.

```javascript
ZOHO.CRM.UI.Widget.open({
  url: "app/secondary-widget.html",
  size: { height: "600", width: "800" },
  features: "popup"
});
```

### UI.Dialer

Integrate with Zoho PhoneBridge / telephony.

```javascript
// Open dialer with a number pre-filled
ZOHO.CRM.UI.Dialer.open({ phoneNumber: "+919876543210" });

// Close dialer
ZOHO.CRM.UI.Dialer.close();

// Notify CRM of call events
ZOHO.CRM.UI.Dialer.notify({
  eventType: "callInitiated",   // "callInitiated" | "callAnswered" | "callEnded"
  callId: "unique-call-id",
  fromNumber: "+10123456789",
  toNumber: "+919876543210"
});
```

---

## ZOHO.CRM.$Client

Low-level postMessage event bus. Use for cross-widget communication or intercepting CRM host events.

```javascript
// Subscribe to a CRM host event
ZOHO.CRM.$Client.on("entity.save", function(data) {
  // fired when user saves the current record
  console.log(data.EntityId);
});

// Fire a custom event (received by other widgets on the page via .on)
ZOHO.CRM.$Client.trigger("custom.event.name", { payload: "value" });
```

---

## Complete minimal widget example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>CRM Widget</title>
  <script src="https://live.zwidgets.com/js-sdk/1.0/ZohoEmbededAppSDK.min.js"></script>
  <style>
    body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; padding: 16px; }
    #status { color: #666; font-size: 14px; }
    #content { margin-top: 12px; }
  </style>
</head>
<body>
  <div id="status">Loading…</div>
  <div id="content"></div>

  <script>
    ZOHO.embeddedApp.on("PageLoad", function(data) {
      document.getElementById("status").textContent = "Widget loaded";

      ZOHO.CRM.API.getRecord({
        Entity: data.Entity,
        RecordID: data.EntityId
      })
      .then(function(response) {
        var record = response.data[0];
        document.getElementById("content").innerHTML =
          "<strong>" + record.Last_Name + "</strong><br>" + (record.Email || "No email");
      })
      .catch(function(err) {
        document.getElementById("status").textContent = "Error: " + JSON.stringify(err);
      });
    });

    ZOHO.embeddedApp.init();
  </script>
</body>
</html>
```

---

## Error handling pattern

```javascript
ZOHO.CRM.API.getRecord({ Entity: "Leads", RecordID: id })
.then(function(response) {
  if (!response.data || response.data.length === 0) {
    // No record found or no permission
    return;
  }
  handleRecord(response.data[0]);
})
.catch(function(error) {
  // error: { code, message, details, status }
  console.error("CRM API error:", error.code, error.message);
});
```

---

## SDK Limitations & Considerations

- All API calls execute in the context of the logged-in CRM user — respect their field-level and module-level permissions.
- `ZOHO.CRM.API.*` calls are subject to CRM API rate limits (same as REST API).
- `ZOHO.CRM.HTTP` only works for URLs whitelisted in `cspDomains.connect-src` in the manifest.
- `ZOHO.CRM.FUNCTIONS.execute` round-trips through the server — not suitable for latency-sensitive UI interactions.
- Widget iframes are sandboxed — no `localStorage`, no cookies, no `window.opener`.
- Always call `ZOHO.embeddedApp.init()` at script load time, not inside DOMContentLoaded.
- For production widgets, the `url` in `plugin-manifest.json` must be an HTTPS URL (not localhost).

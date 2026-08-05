# Error Handling Patterns for Zoho CRM Widgets

## What the official docs actually guarantee

Read this first — it constrains everything below.

- Every SDK method returns a **Promise**. No method documents its rejection shape.
- The docs publish **no error-code table** and carry **no `@throws` annotations** anywhere in the
  SDK dataset. Any numeric CRM error code you use must be verified against the
  [CRM REST API status docs](https://www.zoho.com/crm/developer/docs/api/v8/status-codes.html) for
  your API version — the widget SDK does not define its own.
- Success and failure are reported **inside the resolved value**, via `code` / `status` / `message`.
  A resolved Promise does **not** mean the operation succeeded.
- The casing of those fields is inconsistent across namespaces: `CONNECTION.invoke` resolves
  `code: "SUCCESS"`, `FUNCTIONS.execute` resolves `code: "success"`.
- Response envelopes differ per method (`{data:[…]}`, bare array, flat object, `{users:[…]}`, or a
  raw remote response from `HTTP.*`). There is no single shape to switch on.
- The **only** documented failure mode in the whole SDK reference is
  `ZOHO.CRM.API.getApprovalRecords` returning **204** to a non-approver, and to a standard user
  calling `others_awaiting`.

Consequence: **inspect the payload, don't rely on `.catch()`.**

```javascript
// ❌ Assumes rejection signals failure
ZOHO.CRM.API.updateRecord({ ... })
  .then(() => showSuccess())
  .catch(() => showError());

// ✓ Check the payload — a resolved Promise can still be a failure
ZOHO.CRM.API.updateRecord({ Entity: "Leads", APIData: { id: id, Lead_Status: "Contacted" } })
  .then(function (res) {
    var entry = res && res.data && res.data[0];
    var ok = entry && String(entry.status).toLowerCase() === "success";
    ok ? showSuccess() : showError(entry && entry.message);
  })
  .catch(function (err) {
    // Undocumented shape — log defensively, never index into it blindly
    console.error("updateRecord rejected:", err);
    showError("Request failed");
  });
```

A reusable normalizer for the inconsistent envelopes:

```javascript
// Returns { ok, message, entries } across all the documented response shapes
function normalize(res) {
  if (res == null) return { ok: false, message: "Empty response", entries: [] };

  // upsertRecord resolves a bare array
  if (Array.isArray(res)) {
    return { ok: res.every(e => String(e.status).toLowerCase() === "success"),
             message: res.map(e => e.message).join("; "), entries: res };
  }
  // getRecord / insertRecord / updateRecord / deleteRecord → { data: [...] }
  if (Array.isArray(res.data)) {
    return { ok: res.data.length > 0 &&
                 res.data.every(e => e.status == null ||
                                     String(e.status).toLowerCase() === "success"),
             message: res.data.map(e => e.message).filter(Boolean).join("; "),
             entries: res.data };
  }
  // getUser / getAllUsers / getAllProfiles / getProfile → { users: [...] } / { profiles: [...] }
  var listKey = ["users", "profiles", "fields", "modules", "layouts",
                 "related_lists", "custom_views", "assignment_rules", "actions"]
                .find(k => Array.isArray(res[k]));
  if (listKey) return { ok: true, message: null, entries: res[listKey] };

  // approveRecord / updateBluePrint / CONNECTION.invoke → flat object
  if (res.status || res.code) {
    return { ok: String(res.status ?? res.code).toLowerCase() === "success",
             message: res.message, entries: [res] };
  }
  // HTTP.* → raw remote response; caller must interpret
  return { ok: true, message: null, entries: [res] };
}
```

---

## Empty results vs. permission denials

An empty result and "you can't see this" look identical from a widget. Both surface as an absent
record, not an error.

```javascript
ZOHO.CRM.API.getRecord({ Entity: data.Entity, RecordID: id })
  .then(function (res) {
    var record = res && res.data && res.data[0];
    if (!record) {
      // Record deleted, id wrong, or the user lacks read permission — indistinguishable
      showMessage("No data available for this record.");
      return;
    }
    render(record);
  })
  .catch(function (err) {
    console.error("getRecord failed:", err);
    showMessage("Could not load this record.");
  });
```

Since v1.4, `FUNCTIONS.execute` runs **as the current user, not as admin**. A function that worked
during development under an admin login can fail for standard users with no change to your widget.
Handle a null/error `details.output` as a real possibility:

```javascript
ZOHO.CRM.FUNCTIONS.execute("my_function", { arguments: JSON.stringify({ id: recordId }) })
  .then(function (res) {
    var out = res && res.details && res.details.output;
    if (out == null) { showMessage("Action unavailable for your role."); return; }
    var parsed;
    try { parsed = JSON.parse(out); }
    catch (e) { console.error("Non-JSON function output:", out); return; }
    render(parsed);
  });
```

The documented 204 case:

```javascript
ZOHO.CRM.API.getApprovalRecords({ type: "others_awaiting" })
  .then(function (res) {
    // Standard users and non-approvers get an empty 204 here by design
    var rows = (res && res.data) || [];
    if (rows.length === 0) { showMessage("No approvals visible to your role."); return; }
    render(rows);
  });
```

---

## Retry with backoff

Rate limits are shared with the CRM REST API, but the widget SDK does not document how a limit
surfaces. Retry on *any* failure with a cap, rather than on a specific code you can't verify:

```javascript
function retry(fn, attempts = 3, delay = 1000) {
  return fn().catch(function (err) {
    if (attempts <= 1) return Promise.reject(err);
    return new Promise(function (resolve) {
      setTimeout(() => resolve(retry(fn, attempts - 1, delay * 2)), delay);
    });
  });
}

retry(() => ZOHO.CRM.API.getAllRecords({ Entity: "Contacts", per_page: "200" }))
  .then(handle)
  .catch(err => console.error("Gave up after retries:", err));
```

Keep concurrency bounded — a widget that fans out one call per row will hit limits:

```javascript
function pool(tasks, limit = 3) {
  var i = 0, results = [];
  function worker() {
    if (i >= tasks.length) return Promise.resolve();
    var idx = i++;
    return tasks[idx]()
      .then(r => { results[idx] = r; })
      .catch(e => { results[idx] = { error: e }; })
      .then(worker);
  }
  return Promise.all(Array.from({ length: Math.min(limit, tasks.length) }, worker))
               .then(() => results);
}
```

---

## Initialization failures

`ZOHO.embeddedApp.on` is a generic passthrough — the SDK stores the handler in a map without
validating the event name. **A misspelled event fails silently and forever.** Use the exact six:
`PageLoad`, `Dial`, `DialerActive`, `Notify`, `NotifyAndWait`, `ContextUpdate`.

```javascript
// ❌ Never fires. No error, no warning.
ZOHO.embeddedApp.on("pageload", handler);
ZOHO.embeddedApp.on("App.Load", handler);   // not a real event

// ✓
ZOHO.embeddedApp.on("PageLoad", handler);
ZOHO.embeddedApp.init();
```

Guard against `PageLoad` never arriving (bad widget URL, untrusted dev cert, SDK blocked):

```javascript
var loaded = false;
ZOHO.embeddedApp.on("PageLoad", function (data) { loaded = true; start(data); });
ZOHO.embeddedApp.init();

setTimeout(function () {
  if (!loaded) {
    document.getElementById("status").textContent =
      "Widget did not receive CRM context. Check the widget URL and that the SDK script loaded.";
  }
}, 8000);
```

`ZDK.Client` and `$Client` are **injected by CRM at runtime** and are absent from the SDK bundle.
In an ordinary widget they are `undefined`. Always feature-detect:

```javascript
function toast(msg, type) {
  if (typeof ZDK !== "undefined" && ZDK.Client && ZDK.Client.showMessage) {
    ZDK.Client.showMessage(msg, { type: type || "info" });   // v1.5, Client Script / popup context
  } else {
    document.getElementById("status").textContent = msg;      // fallback
  }
}
```

A v1.5 loader will block every subsequent pop-up if you leak it — always pair it:

```javascript
ZDK.Client.showLoader({ template: "spinner", message: "Fetching…" });
doWork()
  .catch(function (err) { console.error(err); })
  .then(function () { ZDK.Client.hideLoader(); });   // runs on both paths
```

---

## Handling the shape traps

The docs contradict themselves in specific places (full list in `references/sdk.md`). Two are
likely to bite at runtime:

```javascript
// Related-list key: tables say RelatedListName, examples use RelatedList
function getRelated(entity, recordId, listName) {
  var base = { Entity: entity, RecordID: recordId, per_page: 200 };
  return ZOHO.CRM.API.getRelatedRecords(Object.assign({}, base, { RelatedList: listName }))
    .then(r => (r && r.data) ? r : Promise.reject(new Error("empty")))
    .catch(() => ZOHO.CRM.API.getRelatedRecords(
      Object.assign({}, base, { RelatedListName: listName })));
}

// getOrgVariable: "Content" for the single form, "content" for the batch form
function readOrgVar(res) {
  var s = res && res.Success;
  if (!s) return null;
  return s.Content !== undefined ? s.Content : s.content;
}
```

`PageLoad` delivers `EntityId` as a **string** in related-list placements and an **array** in button
placements:

```javascript
var ids = Array.isArray(data.EntityId) ? data.EntityId
        : data.EntityId ? [data.EntityId] : [];
```

`HTTP.*` resolves the **raw remote response** — an XML string for XML endpoints. Never assume JSON:

```javascript
ZOHO.CRM.HTTP.get({ url: endpoint, params: { scope: "crmapi" }, headers: { Authorization: token } })
  .then(function (res) {
    if (typeof res === "string") {
      if (res.trim().startsWith("<")) { handleXml(res); return; }
      try { res = JSON.parse(res); } catch (e) { console.error("Unparseable:", res); return; }
    }
    handleJson(res);
  });
```

---

## Keeping the UI alive

Render inside a boundary so one bad field can't blank the widget:

```javascript
function safeRender(fn) {
  try { fn(); }
  catch (err) {
    console.error("Render failed:", err);
    document.getElementById("content").textContent = "Display error — see console.";
  }
}
```

Degrade instead of failing when an enrichment call is optional:

```javascript
Promise.all([
  ZOHO.CRM.API.getRecord({ Entity: data.Entity, RecordID: id }),
  ZOHO.CRM.API.getRelatedRecords({ Entity: data.Entity, RecordID: id, RelatedList: "Notes" })
    .catch(() => null)          // notes are nice-to-have
]).then(function (results) {
  renderRecord(results[0]);
  if (results[1]) renderNotes(results[1]);
});
```

## Logging

Log locally first — `console.log` output appears in the widget iframe's own console (right-click
inside the widget → Inspect). If you ship errors to an external service, the destination must be
listed in `cspDomains.connect-src` in `plugin-manifest.json`, and the logger must never throw:

```javascript
function logError(context, err) {
  console.error(context, err);
  ZOHO.CRM.HTTP.post({
    url: LOG_ENDPOINT,                       // must be in cspDomains.connect-src
    headers: { "Content-Type": "application/json" },
    body: { context: context, message: String(err && err.message || err) }
  }).catch(function () { /* logging must never break the widget */ });
}
```

Never include record field values, tokens, or org-variable contents in a log payload — see
`references/security.md`.

# Zoho CRM Widgets JS SDK — Verified Reference

**Provenance.** Every signature below is transcribed from Zoho's published JSDoc data
(`widget-sdk-1.4.json` / `widget-sdk-1.5.json`, the datasets that render
`zohocrm.dev/explore/widgets/<ver>/*`) and cross-checked against the shipped bundle at
`https://live.zwidgets.com/js-sdk/1.4/ZohoEmbededAppSDK.min.js`.

**To re-verify** (the doc site is a client-rendered SPA, so plain HTML fetches return only a shell):

```bash
# 1. Get the current fingerprint for the dataset
curl -s https://www.zohocrm.dev/fingerprint_config.json | jq -r '.default["dxh-data-store"]["widget-sdk-1.5.json"]'
# 2. Fetch the dataset (substitute the hash)
curl -s "https://www.zohocrm.dev/dxh-data-store/widget-sdk-1.5_<hash>_.json"
```

Doc page URLs use **dots, not underscores**: `/explore/widgets/v1.5/ZOHO.CRM.API`.
`$Client` has no page of its own — it is documented on the `ZDK.Client` page.
The changelog route is plural: `/explore/widgets/v1.5/changelogs`.

---

## Version & CDN

| Version | CDN URL |
|---------|---------|
| **1.5** (target this) | `https://live.zwidgets.com/js-sdk/1.5/ZohoEmbededAppSDK.min.js` |
| 1.4 | `https://live.zwidgets.com/js-sdk/1.4/ZohoEmbededAppSDK.min.js` |
| 1.3 | `https://live.zwidgets.com/js-sdk/1.3/ZohoEmbededAppSDK.min.js` |
| 1.2 | `https://live.zwidgets.com/js-sdk/1.2/ZohoEmbededAppSDK.min.js` |
| 1.1 | `https://live.zwidgets.com/js-sdk/1.1/ZohoEmbededAppSDK.min.js` |
| 1.0.5 | `https://live.zwidgets.com/js-sdk/1.0.5/ZohoEmbededAppSDK.min.js` |

The filename misspelling is authentic: `ZohoEmbededAppSDK` — one `d` in "Embeded".

```html
<!-- First script in <head> -->
<script src="https://live.zwidgets.com/js-sdk/1.5/ZohoEmbededAppSDK.min.js"></script>
```

**v1.5 vs v1.4:** every `ZOHO.CRM.*` namespace is byte-identical between the two. The only
difference is `ZDK.Client`, which goes from 1 method (v1.4) to 8 (v1.5). v1.5 also introduces
ZRC and multi-page (soft-navigation) support, which are documented outside the Widget SDK
sidebar and are not covered here.

> Zoho's own data is self-contradictory about which release is "latest": `jssdk.json` aliases
> latest → 1.4, while `widget-changelog.json` aliases latest → 1.5, and both carry a "Latest"
> badge. Pin an explicit version in your script tag rather than relying on any alias.

---

## Initialization

```javascript
// Subscribe BEFORE calling init()
ZOHO.embeddedApp.on("PageLoad", function (data) {
  console.log(data);
  // business logic here
});

ZOHO.embeddedApp.init();
```

`init()` takes no parameters and has no documented return value. The subscribe-then-init
ordering is the only constraint the docs state (as an inline comment in the official example).

> `ZOHO.embeddedApp.on` is a generic passthrough — the SDK stores handlers in a map and wires
> them via the CRM host. Event names are **not** validated by the SDK, so a typo fails silently
> rather than throwing.

### Events

Six events, exact case:

| Event | Fires when | Added |
|-------|-----------|-------|
| `PageLoad` | An entity page (detail page) loads | — |
| `Dial` | The Call icon inside Zoho CRM is clicked | — |
| `DialerActive` | The softphone window is toggled | — |
| `Notify` | A Client Script flyout notify call is triggered | v1.2 |
| `NotifyAndWait` | A Client Script flyout notify call is triggered synchronously | v1.2 |
| `ContextUpdate` | The Wizard's form is modified | v1.3 |

Only `PageLoad` has a documented payload. Two shapes, depending on placement:

```jsonc
// Related list placement
{ "Entity": "Leads", "EntityId": "3000000032096" }

// Button placement — EntityId is an ARRAY
{
  "EntityId": ["3000000040011", "3000000032101", "3000000032096"],
  "Entity": "Leads",
  "ButtonPosition": "ListView"
}
```

`EntityId` is a string in the related-list shape and an **array** in the button shape. Branch on
`Array.isArray(data.EntityId)` rather than assuming either.

`NotifyAndWait` supplies a `data.id` used to reply:

```javascript
ZOHO.embeddedApp.on("NotifyAndWait", function (data) {
  ZDK.Client.sendResponse(data.id, { choice: "mail", value: "example@zoho.com" });
});
```

---

## ZOHO.CRM.API — 28 methods

All return a `Promise`. Response envelopes are **not** uniform — see each method.

### Records

#### getRecord(config)

| Param | Type | Req | Default |
|-------|------|-----|---------|
| `config.Entity` | String | ✓ | — |
| `config.RecordID` | String | ✓ | — |
| `config.approved` | String | — | `True` |

```javascript
ZOHO.CRM.API.getRecord({ Entity: "Leads", RecordID: "1000000030136", approved: "both" })
  .then(function (data) { var record = data.data[0]; });
```

Resolves `{ data: [ <record> ] }` — no `info` block. There is **no `Fields` parameter**; the
full record is returned.

#### getAllRecords(config)

| Param | Type | Req | Notes |
|-------|------|-----|-------|
| `config.Entity` | String | ✓ | |
| `config.sort_order` | String | — | `asc` \| `desc` |
| `config.converted` | String | — | |
| `config.approved` | String | — | |
| `config.page` | String | — | |
| `config.per_page` | String | — | |

Resolves `{ data: [...], info: { per_page, count, page, more_records } }`.
No `sort_by` and no `Fields` parameter exist. No `per_page` maximum is documented — `200` appears
in sample responses but is never stated as a cap.

#### insertRecord(config)

**Not `addRecord`.** `addRecord` does not exist.

| Param | Type | Req | Notes |
|-------|------|-----|-------|
| `config.Entity` | String | ✓ | |
| `config.Trigger` | list | ✓ | `"workflow"`, `"approval"`, `"blueprint"`. Omit ⇒ all execute. `[]` ⇒ none execute. |
| `config.APIData` | Object | ✓ | Single object, or an array for bulk |

```javascript
ZOHO.CRM.API.insertRecord({
  Entity: "Leads",
  APIData: { Company: "Zylker", Last_Name: "Peterson" },
  Trigger: ["workflow"]
}).then(function (data) { var id = data.data[0].details.id; });
```

Resolves `{ data: [ { code, details, message: "record added", status } ] }` — one entry per record.

#### updateRecord(config)

| Param | Type | Req | Notes |
|-------|------|-----|-------|
| `config.Entity` | String | ✓ | |
| `config.Trigger` | list | ✓ | Same semantics as `insertRecord` |
| `config.APIData` | Object | ✓ | Declared `String` in the docs, but examples pass an Object |

The record id goes **inside `APIData`** — there is no top-level `RecordID`:

```javascript
ZOHO.CRM.API.updateRecord({
  Entity: "Leads",
  APIData: { id: "1000000049031", Company: "Zylker", Last_Name: "Peterson" }
});
```

Resolves `data[0].message === "record updated"`.

#### upsertRecord(config)

| Param | Type | Req | Notes |
|-------|------|-----|-------|
| `config.Entity` | String | ✓ | |
| `config.Trigger` | list | ✓ | |
| `config.APIData` | Object | ✓ | |
| `config.duplicate_check_fields` | Object | ✓ | Declared Object; example passes an **array**: `["Website", "Mobile"]` |

Resolves a **bare top-level array** (no `data` wrapper), each entry carrying `duplicate_field`
and `action`:

```jsonc
[ { "code": "SUCCESS", "duplicate_field": "Mobile", "action": "update",
    "details": {}, "message": "record updated", "status": "success" } ]
```

#### deleteRecord(config)

`config.Entity` (String, ✓), `config.RecordID` (String, ✓).
Resolves `data[0].message === "record deleted"`.

#### searchRecord(config, page, per_page)

**The only method with positional parameters beyond `config`.**

| Param | Type | Req | Notes |
|-------|------|-----|-------|
| `config.Entity` | String | ✓ | |
| `config.Type` | String | ✓ | `email` \| `phone` \| `word` \| `criteria` |
| `config.Query` | String | ✓ | |
| `config.delay` | boolean | ✓ | |
| `page` | String | ✓ | **Positional**, not inside `config` |
| `per_page` | String | ✓ | **Positional**, not inside `config` |

```javascript
ZOHO.CRM.API.searchRecord({ Entity: "Leads", Type: "phone", Query: "123456789", delay: false });
ZOHO.CRM.API.searchRecord({ Entity: "Leads", Type: "email", Query: "test@zoho.com" });
ZOHO.CRM.API.searchRecord({ Entity: "Leads", Type: "word",  Query: "ZohoCrop" });
ZOHO.CRM.API.searchRecord({ Entity: "Leads", Type: "criteria", Query: "(Company:equals:Zoho)" });
ZOHO.CRM.API.searchRecord({
  Entity: "Leads", Type: "criteria",
  Query: "((Company:equals:Zoho)or(Company:equals:zylker))"
});
```

Criteria form is `(FieldAPIName:operator:value)`, combined with `and` / `or`. No operator list is
published on this page — the CRM REST API criteria operators apply. No `Fields` parameter exists.

#### coql(queryObject)

`queryObject.select_query` (String, ✓). Added in v1.2.

```javascript
ZOHO.CRM.API.coql({
  select_query: "select Last_Name, First_Name, Full_Name from Contacts where Last_Name = 'Boyle' and First_Name is not null limit 2"
});
```

Resolves `{ data: [...], info: { count, more_records } }`.

### Related records

Note the naming: the parameter tables declare **`RelatedListName`**, but every official example
passes **`RelatedList`**. The docs are internally inconsistent here; if one fails, try the other.

#### getRelatedRecords(config)

`config.Entity` ✓, `config.RecordID` ✓, `config.RelatedListName` ✓, `config.page` (Number, opt),
`config.per_page` (Number, opt).

```javascript
ZOHO.CRM.API.getRelatedRecords({
  Entity: "Leads", RecordID: "1000000030136", RelatedList: "Notes", per_page: 200
});
```

#### updateRelatedRecords(config)

**Plural.** `updateRelatedRecord` (singular) does not exist.
`config.Entity` ✓, `config.RecordID` ✓, `config.RelatedListName` ✓, `config.RelatedRecordID` ✓,
`config.APIData` ✓ (declared String, example passes Object).
Resolves `data[0].message === "relation updated"`.

#### delinkRelatedRecord(config)

**This is the delete operation for relations.** `deleteRelatedRecord` does not exist.
`config.Entity` ✓, `config.RecordID` ✓, `config.RelatedListName` ✓, `config.RelatedRecordID` ✓.

### Notes & files

#### addNotes(config)

`config.Entity` (String ✓), `config.RecordID` (**Long** ✓ — the only `RecordID` not typed String),
`config.Title` (String ✓), `config.Content` (String ✓).

#### attachFile(config)

`config.Entity` ✓, `config.RecordID` ✓, `config.File.Name` (String ✓),
`config.File.Content` (object ✓ — a Blob in the official example).
Resolves `data[0].message === "attachment uploaded successfully"`.

#### uploadFile(config)

No parameter table is published; the shape comes only from the example.

```javascript
var file = document.getElementById("attachmentinput").files[0];
ZOHO.CRM.API.uploadFile({
  CONTENT_TYPE: "multipart",
  PARTS: [{ headers: { "Content-Disposition": "file;" }, content: "__FILE__" }],
  FILE: { fileParam: "content", file: file }
});
```

Resolves `{ data: [ { details: { name, id }, ... } ] }`. That `id` is the file id consumed by
`getFile` and by `updateBluePrint`'s `$file_id`.

#### getFile(config)

No parameter table published. Example passes `{ id: "<64-char file hash>" }`.
Resolves with the file as a binary string.

### Users & profiles

#### getUser(config)
`config.ID` (String ✓ — UserID). Resolves `{ users: [ <user> ] }`.

#### getAllUsers(config)

`config.Type` (String ✓ — **capital T**), `config.page` (number, opt), `config.per_page` (number, opt).

Allowed `Type` values: `AllUsers`, `ActiveUsers`, `DeactiveUsers`, `ConfirmedUsers`,
`NotConfirmedUsers`, `DeletedUsers`, `ActiveConfirmedUsers`, `AdminUsers`,
`ActiveConfirmedAdmins`.

> `CurrentUser` is **not** in this list. To get the logged-in user, use
> `ZOHO.CRM.CONFIG.getCurrentUser()`.

#### getAllProfiles()
No parameters. Resolves `{ profiles: [...] }`.

#### getProfile(config)
`config.ID` (String ✓ — ProfileID). Resolves `{ profiles: [ { ..., permissions_details, sections } ] }`.

#### updateProfile(config)
`config.ID` ✓, `config.APIData` ✓.

```javascript
ZOHO.CRM.API.updateProfile({
  ID: "1000000028942",
  APIData: { profiles: [ { permissions_details: [ { id: "...", enabled: false } ] } ] }
});
```

### Approvals

#### getApprovalRecords(config)
`config.type` (string ✓ — **lowercase t**, unlike `getAllUsers`): `awaiting` | `others_awaiting`.

Callable with no argument to get the current user's pending approvals. Documented behaviour:
a caller who is not the approver receives a **204** response; `others_awaiting` is generally
limited to Super Admin / administrator, and standard users get an empty 204.

#### getApprovalById(config)
`config.id` (string ✓).

#### getApprovalsHistory()
No parameters.

#### approveRecord(config)
`config.Entity` ✓, `config.RecordID` ✓, `config.actionType` ✓
(`approve` | `delegate` | `resubmit` | `reject`), `config.comments`, `config.user` (delegate only).

> `comments` and `user` are marked required in the type table but described as optional /
> delegate-only in their descriptions.

Resolves a **flat object** (not array-wrapped):
`{ code: "SUCCESS", details: { id }, message: "Record approved successfully", status: "success" }`.

### Blueprint (read/write live on API, not on BLUEPRINT)

#### getBluePrint(config)
Note the capital **P**, unlike the `ZOHO.CRM.BLUEPRINT` namespace.
`config.Entity` ✓, `config.RecordID` ✓.
Resolves `{ blueprint: { process_info: {...}, transitions: [...] } }`.

#### updateBluePrint(config)
`config.Entity` ✓, `config.RecordID` ✓, `config.BlueprintData` ✓.

```javascript
ZOHO.CRM.API.updateBluePrint({
  Entity: "Leads", RecordID: "1000000030136",
  BlueprintData: {
    blueprint: [ { transition_id: "1000000031001",
                   data: { Phone: "8940372937", Notes: "Updated via blueprint" } } ]
  }
});
```

`data` also accepts attachments — `{ "Attachments": { "$file_id": ["<hash>"] } }` — or a link —
`{ "Attachments": { "$link_url": "facebook.com" } }`.
Resolves flat: `{ code: "SUCCESS", message: "transition updated successfully", status: "success" }`.

### Misc

#### getAllActions(config)
`config.Entity` ✓, `config.RecordID` ✓. Resolves `{ actions: [ { http_method, name, href, params? } ] }`.
The `approvals` action only appears when an approval is pending **and** the caller has admin access.

#### getOrgVariable(...)

**This is how a widget reads its install-time configuration.** No parameter table is published;
two calling forms appear in the examples.

```javascript
// Single variable — note capital "Content"
ZOHO.CRM.API.getOrgVariable("variableNamespace");
// → { "Success": { "Content": "12345" } }

// Batch — note lowercase "content"
ZOHO.CRM.API.getOrgVariable({ apiKeys: ["key1", "key2", "key3"] });
// → { "Success": { "content": { "apikey": { "value": "..." }, "authtoken": { "value": "..." } } } }
```

The `Content` / `content` casing difference between the two forms is in the published docs. Guard
for both.

---

## ZOHO.CRM.CONFIG — 3 methods

This namespace is **only** these three. The metadata methods (`getFields`, `getModules`,
`getLayouts`, `getRelatedList`, `getCustomViews`) live in `ZOHO.CRM.META`.

| Method | Params | Resolves |
|--------|--------|----------|
| `getCurrentUser()` | none | Flat user object — **not** wrapped in `users[]`: `{ confirm, full_name, role{name,id}, profile{name,id}, last_name, alias, id, first_name, email, zuid, status }` |
| `getOrgInfo()` | none | Org info. The published sample output is a copy-paste error showing `{ "Success": { "Content": "12345" } }` — treat the real shape as unverified. |
| `getUserPreference()` | none | `{ "mode": "day" }` — current user's Day/Night theme. Added in v1.4. |

---

## ZOHO.CRM.META — 6 methods

| Method | Params | Resolves (top-level key) |
|--------|--------|--------------------------|
| `getModules()` | none | `{ modules: [...] }` |
| `getFields(config)` | `config.Entity` ✓ | `{ fields: [...] }` |
| `getLayouts(config)` | `config.Entity` ✓, `config.Id` (opt) | `{ layouts: [...] }` |
| `getRelatedList(config)` | `config.Entity` ✓ | `{ related_lists: [...] }` |
| `getCustomViews(config)` | `config.Entity` ✓, `config.Id` (opt) | `{ categories: [...], custom_views: [...] }` |
| `getAssignmentRules(config)` | `config.Entity` ✓ | `{ assignment_rules: [...] }` |

```javascript
ZOHO.CRM.META.getFields({ Entity: "Contacts" })
  .then(function (data) { data.fields.forEach(f => console.log(f.api_name, f.field_label)); });
```

Each `getFields` entry includes `api_name`, `field_label`, `data_type`, `json_type`, `read_only`,
`length`, `pick_list_values`, `view_type{view,edit,quick_create,create}`, and more — so
**`getFields` is where picklist values come from.** There is no `getPickListValues` method.

> `getLayouts` declares its optional key as `config.Id`, but its own official example passes
> `"LayoutId"`. `getCustomViews` declares and uses `Id` consistently. One of the two `getLayouts`
> forms is wrong in the published docs; test against your org.
>
> `getLayouts`, `getRelatedList`, and `getCustomViews` all carry the copy-pasted return
> description "Resolved with data of Assignment rules matching with Entity". Ignore it — each
> returns its own key as tabled above.

---

## ZOHO.CRM.HTTP — 5 methods

`get`, `post`, `put`, `patch`, `delete` — all lowercase. `delete` is a plain function name despite
being a reserved word, so call it as `ZOHO.CRM.HTTP.delete({...})`.

| Method | Declared params |
|--------|-----------------|
| `get(request)` | `request.params`, `request.headers` |
| `post(request)` | `request.params`, `request.headers`, `request.body` |
| `put(request)` | `request.params`, `request.headers`, `request.body` |
| `patch(request)` | `request.params`, `request.headers`, `request.body` |
| `delete(request)` | `request.params`, `request.headers`, `request.body` |

The key is **`params`**, not `parameters`. (`ZOHO.CRM.CONNECTION.invoke` uses `parameters` — the
two namespaces genuinely differ.)

> `request.url` is used in every official example but is **never declared as a parameter** on any
> of the five methods. It is required in practice.

```javascript
ZOHO.CRM.HTTP.get({
  url: "https://crm.zoho.com/crm/private/xml/Users/getUsers",
  params:  { scope: "crmapi", type: "AllUsers" },
  headers: { Authorization: "******" }
}).then(function (data) { console.log(data); });

ZOHO.CRM.HTTP.put({
  url: "https://crm.zoho.com/crm/v2/Contacts",
  headers: { Authorization: "******" },
  body: { data: apidata }
});
```

These resolve with the **raw remote response, un-enveloped** — an XML string for an XML endpoint,
the target API's JSON otherwise. Do not assume a `{ data: [...] }` wrapper.

---

## ZOHO.CRM.CONNECTION — 1 method

### invoke(conn_name, req_data)

Two positional arguments.

| Param | Type | Req | Default |
|-------|------|-----|---------|
| `conn_name` | String | ✓ | — |
| `req_data.url` | String | ✓ | — |
| `req_data.method` | String | — | `GET` |
| `req_data.parameters` | Object | — | — |
| `req_data.headers` | Object | — | — |
| `req_data.param_type` | Integer | ✓ | — (**1** = params, **2** = payload) |

```javascript
ZOHO.CRM.CONNECTION.invoke("mailchimp4", {
  url: "http://mailchimp.api/sample_api",
  method: "POST",
  parameters: { param1: "paramvalue1" },
  headers: { header1: "headervalue1" },
  param_type: 1
});
```

Resolves `{ code: "SUCCESS", details: { CODE: 200, message, status }, message: "Connection invoked successfully", status: "success" }`.

There is no `makeRequest` method.

---

## ZOHO.CRM.CONNECTOR — 2 methods

### invokeAPI(nameSpace, data)

| Param | Type | Notes |
|-------|------|-------|
| `nameSpace` | String | e.g. `"MailChimp.sendSubscription"` |
| `data.VARIABLES` | Object | Values for placeholders in the connector API |
| `data.CONTENT_TYPE` | Object | `"multipart"` for multipart requests |
| `data.PARTS` | Array | Multipart part configs |
| `data.FILE` | Object | `{ fileParam, file }` |

For multipart, the file placeholder is the literal string `"__FILE__"` in a part's `content`.

### authorize(nameSpace)
`nameSpace` (String ✓). Prompts the connector authorize window; resolves `true` on success.

---

## ZOHO.CRM.FUNCTIONS — 1 method

### execute(func_name, req_data)

Two positional arguments — **not** a single config object.

`req_data` has no documented sub-parameters. The example shows one key, `arguments`, whose value
is a **JSON string**:

```javascript
ZOHO.CRM.FUNCTIONS.execute("custom_function4", {
  arguments: JSON.stringify({ mailid: "user@example.com" })
}).then(function (data) {
  var output = data.details.output;   // the function's return value
  var type   = data.details.type;     // e.g. "VOID"
  var execId = data.details.id;
});
```

Resolves `{ code: "success", details: { type, output, id }, message: "function executed successfully" }`.
Note `code` is **lowercase** `"success"` here, while `CONNECTION.invoke` returns uppercase
`"SUCCESS"`.

**v1.4 behavioural change:** execution scope moved from admin to **the current user**. A function
that relied on admin privileges may now fail for standard users.

---

## ZOHO.CRM.BLUEPRINT — 1 method

### proceed()

**No parameters.** Advances the blueprint to the next state.

```javascript
ZOHO.CRM.BLUEPRINT.proceed();
```

Added in v1.1. There is no `getTransitions` on this namespace — read transitions with
`ZOHO.CRM.API.getBluePrint(config)` and commit a specific transition with
`ZOHO.CRM.API.updateBluePrint(config)`.

---

## ZOHO.CRM.WIZARD — 1 method

### post(record_data)

`record_data` (Object ✓) — field data to set on the record in the wizard. Added in v1.1.

```javascript
ZOHO.CRM.WIZARD.post({ field_api_name1: "field_value", field_api_name2: "field_value" });
```

There is no `proceed` on this namespace. To observe wizard form edits, subscribe to the
`ContextUpdate` event.

---

## ZOHO.CRM.UI

All `UI` methods resolve with `true | false`.

### Resize(dimensions)

`dimensions.height` (Integer ✓), `dimensions.width` (Integer ✓) — in px. The official example
passes strings.

```javascript
ZOHO.CRM.UI.Resize({ height: "200", width: "1000" });
```

v1.3 added related-list height resize; v1.4 added Wizard resize support.

### UI.Popup — 2 methods

Both zero-argument: `close()`, `closeReload()` (close and reload the underlying view).

### UI.Record — 4 methods

| Method | Params |
|--------|--------|
| `open(data)` | `data.Entity` ✓, `data.RecordID` ✓ |
| `edit(data)` | `data.Entity` ✓, `data.RecordID` ✓ |
| `create(data)` | `data.Entity` ✓ **only** |
| `populate(RecordData)` | `RecordData` (Object ✓) |

```javascript
ZOHO.CRM.UI.Record.open({ Entity: "Leads", RecordID: "1000000036062" });
ZOHO.CRM.UI.Record.create({ Entity: "Leads" });

// Prefill the open entity form — create() takes no field data
ZOHO.CRM.UI.Record.populate({
  Annual_Revenue: "500", Description: "Populating test data", Phone: "85663655785"
});
```

`create()` has **no `data` field for prefilling** — use `populate()` for that.

**Undocumented but shipped:** `open`, `edit`, and `create` all forward a `Target` key (the v1.0.5
changelog records `Target="_blank"` support for the Record API). `create` also forwards `RecordID`.
Neither appears in the parameter tables.

### UI.Widget — 1 method

### open(...)
No parameters are formally documented. The shape comes only from the example:

```javascript
ZOHO.CRM.UI.Widget.open({
  Entity: "WebTab1_Widget",
  Message: { arg1: "Argument 1", arg3Nested: { subArg1: "SubArgument 1" } }
});
```

`{ Entity: <webtab/widget name>, Message: <arbitrary object, nesting allowed> }`. This opens a
**WebTab widget** — it is not a generic "open an HTML file in a popup" call. For that, use
`ZDK.Client.openPopup()` (v1.5).

### UI.Dialer — 3 methods

All **zero-argument**, all resolve `true | false`:

| Method | Effect |
|--------|--------|
| `maximize()` | Maximizes the CallCenter window |
| `minimize()` | Minimizes the CallCenter window |
| `notify()` | Notifies the user with an audible sound |

There is no `open()`, no `close()`, and `notify()` takes **no** call-metadata object.

---

## ZOHO.CRM.EVENTS — 1 method

Present in the SDK dataset but **absent from the doc sidebar**, so treat it as semi-public.

### dispatch(event_name, event_data)
`event_name` (String ✓), `event_data` (Object ✓). No documented return value.

---

## ZOHO.CRM.ACTION — undocumented namespace

Present in the raw JSDoc data since **v1.0.5** and confirmed in every shipped bundle including v1.5,
but **absent from `filtered-widget-sdk.json`** (the public API dataset that renders the doc pages).
Treat as an internal namespace — use only if you know the specific CRM context that requires it.

Two methods confirmed from the v1.5 bundle source:

### setConfig(object)

Sends a `CUSTOM_ACTION_SAVE_CONFIG` postMessage to the CRM host. Used by widgets embedded inside
CRM Action flows to persist configuration.

```javascript
ZOHO.CRM.ACTION.setConfig({ key: "value" });
```

### enableAccountAccess(object)

Sends an `ENABLE_ACCOUNT_ACCESS` postMessage to the CRM host. Likely used in enterprise
cross-account extension contexts.

```javascript
ZOHO.CRM.ACTION.enableAccountAccess({ /* object */ });
```

Neither method has documented parameters, return values, or error shapes in any version of the
JSDoc dataset. Both return a Promise (same internal postMessage wrapper as all other SDK methods).
Do not use unless you are certain of the host context that requires it — there is no public
guidance on when these succeed or fail.

---

## ZDK.Client

Injected by CRM at runtime for widgets rendered from Client Script or via `openPopup`. It is
**not** in the SDK bundle, so it is `undefined` in an ordinary widget context — feature-detect
before use.

### v1.4 — 1 method

#### sendResponse(request_uuid, [data])
`request_uuid` (String ✓) — the id from the `NotifyAndWait` event. `data` (Any, optional).

### v1.5 — 8 methods

> All the pop-up style methods share one constraint: **an active loader blocks other pop-ups.**
> Call `hideLoader()` before showing a message, alert, confirmation, or input.

#### showMessage(message, [options])
`message` (String ✓). `options.type`: `info` (default) | `error` | `warning` | `success`.

Markdown supported: `_italics_`, `*bold*`, `__underline__`, `~strikeout~`, `` `code` ``,
`# Heading1`, `### Heading3`, `!blockquote`, `[link](https://…)`.
**Image markdown is not supported by `showMessage`** (it is by `showAlert`).

```javascript
ZDK.Client.showMessage("This is an *important* warning.", { type: "warning" });
```

#### showAlert(message, [heading], [accept_message])
`accept_message` defaults to `Okay`. Supports image markdown:
`ZDK.Client.showAlert("![alt](https://link-to/sample.png)")`.

#### showConfirmation(message, [accept_message], [reject_message])
Defaults `Yes, Proceed` / `Cancel`. **Returns Boolean.**

```javascript
var ok = await ZDK.Client.showConfirmation("Are you *sure*?", "Yes. Got it!", "Nope");
```

#### getInput(options, [heading], [accept_message], [reject_message])

`options.type`: `text` (default) | `number` | `textarea` | `picklist` | `multiselectpicklist`.
`options.label`, `options.default_value` (String, or Array for `multiselectpicklist`),
`options.list_options` (`[{ actual_value, display_value }]`).

Limits: **7 input options max**; `text` 120 chars; `number` 50 digits; `textarea` 2000 chars;
picklist / multiselectpicklist 2000 options, each option 120 chars.

> Documented as returning `Object`, but every official example shows an **array** of values
> (e.g. `["120", "Batch A"]`). The first parameter is typed `Object` while the examples pass an
> array. Both are inconsistencies in the published docs — code defensively.

#### openPopup(config, [data])

Opens another widget in a popup and awaits its `$Client.close()` value. **Requires widget SDK ≥ 1.2**
— the only explicit minimum-version note in the docs.

| Key | Req | Default | Notes |
|-----|-----|---------|-------|
| `config.api_name` | ✓ | — | API name of the widget (Custom Button type) |
| `config.type` | ✓ | — | `"widget"` |
| `config.header` | — | widget name | `undefined` hides the header |
| `config.close_icon` | — | `true` | |
| `config.close_on_escape` | — | `false` | |
| `config.animation_type` | — | `1` | 1 top, 2 right, 3 left, 4 bottom, 5 fade, 6 zoom |
| `config.height` | — | `70vh` | px or vh |
| `config.width` | — | `60vw` | px or vw |
| `config.top` | — | `0` | px or `center` |
| `config.left` | — | `center` | px or `center` |
| `config.bottom` | — | — | overrides `top` |
| `config.right` | — | — | overrides `left` |
| `data` | — | — | delivered as the child widget's `PageLoad` event data |

Returns `object | string | number | boolean` — whatever the child passed to `$Client.close()`.

```javascript
var result = await ZDK.Client.openPopup(
  { api_name: "sample_widget", type: "widget", header: "Sample Widget",
    animation_type: 4, height: "450px", width: "450px" },
  { data: "passed to the child's PageLoad" }
);
```

#### showLoader(config)
`config.type`: `page` (default). `config.template`: `standard` (default) | `spinner` |
`vertical-bar` — spinner and vertical-bar are recommended for long-running API calls.
`config.message`: 240 char limit.

#### hideLoader()
No parameters.

#### sendResponse(request_uuid, [data])
As v1.4; description updated to "Send response to widget".

---

## $Client — 1 method

### close([response])
`response` (Any, optional) — passed back to whatever opened this widget (Client Script flyout, or
the `openPopup` caller). Added in v1.2.

```javascript
$Client.close({ choice: "mail", value: "example@zoho.com" });
```

There is no `$Client.on(...)`. **`$Client` and `ZDK.Client` expose no subscribable events** —
confirmed by the absence of any `@event`/`@fires` tags in the dataset. All widget events go through
`ZOHO.embeddedApp.on`.

---

## Does not exist — do not use

These appear in third-party guides and in earlier revisions of this reference. None are in the
official dataset or the shipped bundle:

| Non-existent | Use instead |
|--------------|-------------|
| `ZOHO.CRM.API.addRecord` | `insertRecord` |
| `ZOHO.CRM.API.updateRelatedRecord` (singular) | `updateRelatedRecords` |
| `ZOHO.CRM.API.deleteRelatedRecord` | `delinkRelatedRecord` |
| `ZOHO.CRM.CONFIG.getFields` / `getModules` / `getLayouts` / `getRelatedList` / `getCustomViews` | same names on `ZOHO.CRM.META` |
| `ZOHO.CRM.CONFIG.getPickListValues` | `ZOHO.CRM.META.getFields` → `field.pick_list_values` |
| `ZOHO.CRM.CONFIG.getParameter` | `ZOHO.CRM.API.getOrgVariable` |
| `ZOHO.CRM.META.getEnvironment` / `getPortalInfo` / `getAppInfo` | `CONFIG.getCurrentUser`, `CONFIG.getOrgInfo`, and the `PageLoad` payload |
| `ZOHO.CRM.CONNECTION.makeRequest` | `CONNECTION.invoke(conn_name, req_data)` |
| `ZOHO.CRM.CONNECTOR.makeRequest` | `CONNECTOR.invokeAPI(nameSpace, data)` |
| `ZOHO.CRM.BLUEPRINT.getTransitions` | `ZOHO.CRM.API.getBluePrint` |
| `ZOHO.CRM.BLUEPRINT.proceed({...})` with args | `proceed()` — zero args; or `API.updateBluePrint` |
| `ZOHO.CRM.WIZARD.proceed` | `WIZARD.post(record_data)` |
| `ZOHO.CRM.UI.Dialer.open` / `.close` | `maximize()` / `minimize()` / `notify()` only |
| `ZOHO.CRM.UI.Widget.open({url, size, features})` | `open({Entity, Message})`, or `ZDK.Client.openPopup` (v1.5) |
| `ZOHO.CRM.$Client.on` / `.trigger` | `ZOHO.embeddedApp.on`; `$Client.close` |
| `ZDK.Client.on` / `.resize` / `.close` / `.getConfig` / `.setConfig` | `ZOHO.embeddedApp.on`; `ZOHO.CRM.UI.Resize`; `$Client.close`; `API.getOrgVariable` |
| `ZDK.Client.on("App.Load" \| "App.Redirect" \| "App.Message")` | `ZOHO.embeddedApp.on("PageLoad" \| …)` — see the event table |
| A `Fields` param on `getRecord` / `getAllRecords` / `searchRecord` | Not supported; full records are returned |
| A `sort_by` param on `getAllRecords` | Only `sort_order` exists |

---

## Documented inconsistencies to code around

The published docs contradict themselves in these specific places. Each is a real trap:

1. **`RelatedListName` vs `RelatedList`** — parameter tables say the former, all examples use the
   latter (`getRelatedRecords`, `updateRelatedRecords`, `delinkRelatedRecord`).
2. **`getLayouts` `Id` vs `LayoutId`** — table declares `Id`, its own example passes `LayoutId`.
3. **`getAllUsers` uses `Type`, `getApprovalRecords` uses `type`** — casing differs between methods.
4. **`getOrgVariable` returns `Content` (string form) but `content` (batch form).**
5. **Response envelopes are not uniform** — `getRecord` → `{data:[…]}`; `upsertRecord` → bare array;
   `approveRecord` / `updateBluePrint` → flat object; `getUser` → `{users:[…]}`;
   `CONFIG.getCurrentUser` → flat object; `HTTP.*` → raw remote response.
6. **`code` casing differs** — `"SUCCESS"` from `CONNECTION.invoke`, `"success"` from
   `FUNCTIONS.execute`.
7. **`APIData` is typed `String`** on `updateRecord` and `updateRelatedRecords` but every example
   passes an Object.
8. **`getFile`, `getOrgVariable`, `uploadFile`, `UI.Widget.open`** render with empty parameter lists
   yet all require an argument. Their shapes exist only in examples.
9. **`addNotes.RecordID` is typed `Long`** while every other `RecordID` is `String`.
10. **`attachFile`'s description** reads "To delink the relation between the records" — a
    copy-paste error. It does upload an attachment.

## Error handling

The docs publish **no error-code table and no `@throws` anywhere**. What is actually documented:

- Every method returns a Promise; reject shapes are undocumented.
- Success/failure is carried in the payload as `code` / `status` / `message` (with the casing
  inconsistencies noted above), so **inspect the resolved value — do not rely on rejection alone**.
- The one documented failure mode is `getApprovalRecords` returning **204** to non-approvers and to
  standard users calling `others_awaiting`.

See `references/error-handling.md` for defensive patterns built on these facts.

## Runtime constraints

- Calls execute as the **logged-in CRM user** and respect their module and field permissions.
- `ZOHO.CRM.API.*` shares the CRM REST API rate limits.
- External URLs must be allowed via `cspDomains` in `plugin-manifest.json`.
- `FUNCTIONS.execute` round-trips to the server — unsuitable for latency-sensitive UI.
- The widget runs in a sandboxed iframe.
- Production widget URLs must be HTTPS.
- Subscribe with `ZOHO.embeddedApp.on(...)` **before** calling `ZOHO.embeddedApp.init()`.

---

## Complete widget example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>CRM Widget</title>
  <script src="https://live.zwidgets.com/js-sdk/1.5/ZohoEmbededAppSDK.min.js"></script>
  <style>
    body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; padding: 16px; }
    #status { color: #666; font-size: 14px; }
  </style>
</head>
<body>
  <div id="status">Loading…</div>
  <div id="content"></div>

  <script>
    function text(el, s) { document.getElementById(el).textContent = s; }

    ZOHO.embeddedApp.on("PageLoad", function (data) {
      // Button placements deliver an ARRAY of ids
      var id = Array.isArray(data.EntityId) ? data.EntityId[0] : data.EntityId;

      if (!id) { text("status", "No record in context"); return; }

      ZOHO.CRM.API.getRecord({ Entity: data.Entity, RecordID: id })
        .then(function (response) {
          var record = response && response.data && response.data[0];
          if (!record) { text("status", "Record not found or not permitted"); return; }

          text("status", "");
          var name = document.createElement("strong");
          name.textContent = record.Last_Name || "(no name)";     // textContent, not innerHTML
          var email = document.createElement("div");
          email.textContent = record.Email || "No email";
          var box = document.getElementById("content");
          box.appendChild(name);
          box.appendChild(email);
        })
        .catch(function (err) {
          text("status", "Failed to load record");
          console.error("getRecord failed:", err);
        });
    });

    // Must come after .on(), at script load time
    ZOHO.embeddedApp.init();
  </script>
</body>
</html>
```

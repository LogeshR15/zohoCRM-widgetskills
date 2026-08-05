# PageLoad Event Payload Shapes by Widget Placement

Reference for `ZOHO.embeddedApp.on("PageLoad", function(data) { ... })`.

The `data` object structure varies significantly by where the widget is placed. Wrong assumptions here are the #1 source of silent runtime bugs.

---

## 1. Quick Reference Table

| Placement | `data.Entity` | `data.EntityId` | `data.related_list` | Notes |
|---|---|---|---|---|
| **Custom Button — List View** | string (module) | **array** of selected IDs | — | EntityId is always an array here |
| **Custom Button — Detail View** | string (module) | string (single ID) | — | |
| **Related List** | string (module) | string (parent record ID) | object (see shape below) | |
| **Record Detail (crm.record.detail)** | string (module) | string (single ID) | — | |
| **Blueprint Transition** | string (module) | string (single ID) | — | No transition metadata in PageLoad |
| **Create Form** | string (module) | absent / undefined | — | Record not yet created |
| **Record List View** | string (module) | absent / undefined | — | No record selected |
| **Dashboard** | absent / undefined | absent / undefined | — | Minimal/empty data |
| **Homepage** | absent / undefined | absent / undefined | — | |
| **Web Tab** | absent / undefined | absent / undefined | — | |
| **Settings Tab** | absent / undefined | absent / undefined | — | |
| **Popup (via Client Script)** | absent / undefined | absent / undefined | — | Only receives fields the opener explicitly passed |

---

## 2. EntityId: String vs Array (Critical)

**The single most common silent bug in CRM widgets.**

| Placement | `EntityId` type |
|---|---|
| List View button (one or more records selected) | `string[]` — always an array, even when one record is selected |
| All other record-context placements | `string` — a single record ID |

**Never** call `.split()`, `.trim()`, or string methods directly on `EntityId` from a button widget. **Never** call `.map()` or `.forEach()` on `EntityId` from a detail/related-list widget.

### Defensive pattern — use this every time

```javascript
ZOHO.embeddedApp.on("PageLoad", function(data) {
  const module = data.Entity;       // may be undefined on dashboard/web tab
  const rawId  = data.EntityId;    // may be string, array, or undefined

  // Normalize to array regardless of placement
  let recordIds = [];
  if (rawId) {
    recordIds = Array.isArray(rawId) ? rawId : [rawId];
  }

  // Now recordIds is always string[] (possibly empty)
  console.log("Module:", module);
  console.log("Record IDs:", recordIds);
});
```

---

## 3. Payload Shapes per Placement

### 3a. Custom Button — List View

Triggered when the user selects one or more records in a list view and clicks a custom button.

```javascript
// data shape
{
  Entity:   "Leads",            // module API name
  EntityId: ["id1", "id2"]     // ARRAY — even if only one record selected
}
```

```javascript
// Usage pattern
ZOHO.embeddedApp.on("PageLoad", function(data) {
  const module    = data.Entity;
  const recordIds = data.EntityId;  // always an array here
  // recordIds.forEach(id => { ... });
});
```

### 3b. Custom Button — Detail View

Triggered from the detail page of a single record.

```javascript
// data shape
{
  Entity:   "Contacts",       // module API name
  EntityId: "4876457000000" // STRING — single record ID
}
```

### 3c. Related List Widget

Placed inside the related list section of a record detail page.

```javascript
// data shape
{
  Entity:   "Contacts",         // parent module API name
  EntityId: "4876457000000",    // parent record ID (string)
  related_list: {
    id:    "4876457000001",     // related list ID
    label: "Deals",             // display label of the related list
    associated_feature_type: "module",  // type of the association
    associated_feature_resource: {
      name:     "Deals",        // display name of related module
      api_name: "Deals",        // API name of related module
      id:       "4876457000002" // related module ID
    }
  }
}
```

```javascript
// Usage pattern
ZOHO.embeddedApp.on("PageLoad", function(data) {
  const parentModule = data.Entity;
  const parentId     = data.EntityId;
  const relatedList  = data.related_list;
  const relatedName  = relatedList.associated_feature_resource.api_name;
  const relatedLabel = relatedList.label;
});
```

### 3d. Blueprint Transition Widget

Placed inside a Blueprint transition panel (e.g., approval/review step on a deal or custom object).

```javascript
// data shape
{
  Entity:   "Deals",           // module API name
  EntityId: "4876457000000"  // string — the record currently in the transition
}
// NOTE: Blueprint transition details (transition name, stage info) are NOT
// in PageLoad. Use ZOHO.CRM.API.getRecord() to fetch the record, then
// ZOHO.CRM.META.getLayouts() or Blueprint-specific APIs for transition context.
```

```javascript
// Usage pattern
ZOHO.embeddedApp.on("PageLoad", function(data) {
  const module   = data.Entity;
  const recordId = data.EntityId;
  ZOHO.CRM.API.getRecord({ Entity: module, RecordID: recordId })
    .then(function(response) {
      const record = response.data[0];
      // render transition UI using record fields
    });
});
```

### 3e. Popup Widget (triggered by Client Script)

Opened via `ZDKClient.popupConfig()` or the Client Script `openPopup` API. The widget receives **only** the custom fields the caller explicitly passes — there is no automatic Entity/EntityId injection.

```javascript
// Example: Client Script passes custom data when opening the popup
// The data shape is entirely determined by what the caller sends.
// Example shape if caller passed { max_rows: 10, product_category: "Gold" }:
{
  max_rows:         10,
  product_category: "Gold"
  // Entity and EntityId are NOT present unless the caller explicitly included them
}
```

```javascript
// Usage pattern — always guard every field
ZOHO.embeddedApp.on("PageLoad", function(data) {
  const maxRows         = data.max_rows         || 5;
  const productCategory = data.product_category || null;
  // Do not assume Entity or EntityId exist
});
```

### 3f. Dashboard / Homepage / Web Tab / Settings Tab

PageLoad fires but `data` carries no record context.

```javascript
// data shape — approximately
{}
// or
{ Entity: undefined, EntityId: undefined }
```

No Entity, no EntityId, no related_list. Do not attempt to call `ZOHO.CRM.API.getRecord()` from these placements without obtaining an ID through another means (e.g., a picker UI or URL param).

---

## 4. Fields That Are NOT in PageLoad

Developers commonly expect these to be present. They are not.

| Expected field | Reality |
|---|---|
| Record field values (name, phone, email, etc.) | Not present. Call `ZOHO.CRM.API.getRecord()` to fetch them. |
| Blueprint transition name or ID | Not in PageLoad. Use Blueprint-specific APIs after init. |
| Related list records | Only the `related_list` metadata (IDs/labels) is present, not the actual child records. |
| Current user info | Not in PageLoad. Call `ZOHO.CRM.CONFIG.getCurrentUser()` separately. |
| Organization info | Not in PageLoad. Call `ZOHO.CRM.CONFIG.getOrgInfo()` separately. |
| Custom button name / trigger source | Not present. You cannot tell which button triggered the widget from PageLoad alone. |
| Locale / currency / timezone | Not in PageLoad. Use `ZOHO.CRM.CONFIG.getOrgInfo()`. |
| Parent record fields (in related list) | Only the parent record ID is present. Fetch the record separately if you need field values. |

---

## 5. ZDK-based Widgets (No PageLoad Handler)

Widgets using `ZDK.Client` (the newer SDK pattern, e.g., Tickets-style widgets) often skip the PageLoad handler entirely.

```javascript
// Minimal ZDK init — no PageLoad needed
ZOHO.embeddedApp.init();
// Then use ZDK.Client methods directly:
// ZDK.Client.getAllRecords({ ... })
// ZDK.Client.getRelatedRecord({ ... })
```

If you are building with ZDK, do not add a PageLoad listener expecting record context — the ZDK methods carry their own context resolution internally.

---

## 6. Placement-to-manifest-type Mapping

For reference when reading `plugin-manifest.json`:

| `plugin-manifest.json` location type | PageLoad placement category |
|---|---|
| `crm.record.button` (listview) | Custom Button — List View |
| `crm.record.button` (detailview) | Custom Button — Detail View |
| `crm.relatedlist.button` | Custom Button — Detail View |
| `crm.related.list` | Related List Widget |
| `crm.record.detail` | Record Detail Widget |
| `crm.blueprint` | Blueprint Transition Widget |
| `crm.dashboard.component` | Dashboard Widget |
| `crm.homepage.component` | Homepage Widget |
| `crm.webtab` | Web Tab Widget |
| Popup (Client Script) | Popup Widget |

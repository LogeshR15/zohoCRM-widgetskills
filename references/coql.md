# COQL — CRM Object Query Language

**Provenance.** Syntax rules and limits are taken from Zoho's COQL API documentation.
Code examples are transcribed verbatim from the four official Zoho widget samples that use COQL:
Sample 1 (mass communication), Sample 6 (mandate subform / Blueprint), and Sample 7 (geolocation).
The `API.coql` vs `CONNECTION.invoke` tradeoff is observed behaviour from those samples, not
official guidance.

---

## What is COQL?

COQL is a SQL-like query language for reading Zoho CRM data. It maps to the REST endpoint
`POST /crm/v7/coql`. From a widget, you call it either through the SDK shorthand
`ZOHO.CRM.API.coql()` or through `ZOHO.CRM.CONNECTION.invoke()`.

**Read-only.** COQL is a SELECT-only language. There is no INSERT, UPDATE, or DELETE.

### When to use COQL instead of other SDK methods

| Situation | Use |
|-----------|-----|
| Fetch one record by ID | `ZOHO.CRM.API.getRecord({ Entity, RecordID })` |
| Full-text / keyword search on indexed fields | `ZOHO.CRM.API.searchRecord({ Entity, Type, Query })` |
| Fetch all records in a module (no filter) | `ZOHO.CRM.API.getAllRecords({ Entity })` |
| **Filter by specific IDs (bulk fetch)** | **COQL** |
| **Cross-field WHERE with AND/OR** | **COQL** |
| **Aggregate: COUNT, GROUP BY** | **COQL** |
| **Query subform modules** | **COQL** |
| **Pagination beyond the first page** | **COQL with OFFSET** |

Rule of thumb: if you would naturally write a SQL WHERE clause, reach for COQL.

---

## Syntax Reference

```sql
SELECT field1, field2, ...
FROM   ModuleName
WHERE  condition
GROUP BY field
ORDER BY field [ASC | DESC]
LIMIT  n
OFFSET n
```

All clauses except `SELECT` and `FROM` are optional.

### SELECT

List API-name fields separated by commas. There is no `SELECT *`.

```sql
SELECT First_Name, Last_Name, Email, Phone FROM Leads
```

Aggregate functions supported: `COUNT(field)`, `SUM(field)`, `AVG(field)`, `MIN(field)`, `MAX(field)`.

```sql
SELECT Document_Category, COUNT(id) FROM Document_Details WHERE id is not null
GROUP BY Document_Category
```

### FROM

The module API name (not the display label). Standard modules use their API names
(`Leads`, `Contacts`, `Deals`, `Accounts`, …). Subform modules also work — use the
subform's API name as it appears in the module's field metadata.

```sql
FROM Contacts
FROM Document_Details   -- subform module from Sample 6
```

### WHERE

Standard comparison operators: `=`, `!=`, `<`, `>`, `<=`, `>=`.
Logical operators: `and`, `or`. Use parentheses to control precedence.

```sql
WHERE Last_Name = 'Smith'
WHERE Amount > 50000 and Stage != 'Closed Lost'
WHERE (City = 'Chennai' or City = 'Mumbai') and Lead_Status = 'Contacted'
```

**`in` list** — filter by a set of values (strings or numbers):

```sql
WHERE id in ('5163XXXXXXXXXXX001', '5163XXXXXXXXXXX002', '5163XXXXXXXXXXX003')
```

Up to 200 values per `in` list.

**Null checks:**

```sql
WHERE Phone is not null
WHERE Secondary_Email is null
```

**`like` pattern matching** (% is wildcard):

```sql
WHERE Last_Name like 'Smi%'
```

**Date / DateTime literals:**

```sql
WHERE Created_Time >= '2025-01-01T00:00:00+05:30'
WHERE Closing_Date between '2025-01-01' and '2025-03-31'
```

Date format: `YYYY-MM-DD`. DateTime format: `YYYY-MM-DDThh:mm:ss+HH:MM`.

**Parent ID filter for subforms:**

```sql
WHERE Parent_Id = '5163XXXXXXXXXXX001'
```

### GROUP BY

One or more non-aggregate fields. Must also appear in SELECT.

```sql
SELECT Stage, COUNT(id) FROM Deals WHERE Closing_Date >= '2025-01-01' GROUP BY Stage
```

### ORDER BY

```sql
ORDER BY Created_Time DESC
ORDER BY Last_Name ASC, First_Name ASC
```

### LIMIT and OFFSET

```sql
LIMIT 50
LIMIT 200 OFFSET 200   -- page 2
```

Default LIMIT is 200. Maximum LIMIT is 200. Use OFFSET to paginate:
- Page 1: `LIMIT 200 OFFSET 0` (or omit OFFSET)
- Page 2: `LIMIT 200 OFFSET 200`
- Page n: `LIMIT 200 OFFSET (n-1)*200`

---

## Invocation Pattern 1 — `ZOHO.CRM.API.coql`

Use this for standard queries from record detail, button, related-list, and homepage widgets.
It is simpler and handles auth automatically.

```javascript
// Async / await style
const ids = recordIds.map(id => `'${id}'`).join(', ');
const select_query = `SELECT First_Name, Last_Name, Email FROM Contacts WHERE id in (${ids})`;

const response = await ZOHO.CRM.API.coql({ select_query });
// response === { data: [ { First_Name: '...', Last_Name: '...', Email: '...', id: '...' }, ... ] }
const records = response.data;
```

```javascript
// Promise / .then style (from Sample 7)
const coqlQuery = `select First_Name, Last_Name, City, Street
                   from Contacts
                   where Street = '${street}' or City = '${city}'`;

ZOHO.CRM.API.coql({ select_query: coqlQuery }).then(function (coqlResponse) {
    coqlResponse.data.forEach(function (record) {
        console.log(record.First_Name, record.City);
    });
});
```

### Response shape — `API.coql`

```javascript
{
  data: [
    { Field_Name: value, Another_Field: value, id: '5163XXXXXXXXXXX' },
    { ... }
  ]
}
```

When there are no matching records the `data` key is absent (or the SDK may return
`{ data: undefined }`). Always guard:

```javascript
const records = response.data ?? [];
```

---

## Invocation Pattern 2 — `ZOHO.CRM.CONNECTION.invoke`

Use this in Blueprint widgets, or when `API.coql` returns a scope/permission error,
or when you need aggregate functions (`COUNT`, `GROUP BY`) that `API.coql` does not surface.

Requires a named connection (`crm_oauth_connection` is the standard built-in one) declared in
`plugin-manifest.json`.

```javascript
// From Sample 6 (Blueprint mandate widget)
const query = {
    select_query: `select Document_Category, COUNT(id)
                   from Document_Details
                   where id is not null and Parent_Id = '${contactId}'
                   group by Document_Category`
};

ZOHO.CRM.CONNECTION.invoke("crm_oauth_connection", {
    method: "POST",
    url: "https://www.zohoapis.com/crm/v7/coql",
    parameters: query,
    param_type: 2,                                  // 2 = JSON body
    headers: { "Content-Type": "application/json" }
}).then(function (response) {
    // The actual CRM API response is nested under response.details.statusMessage
    const statusMessage = response.details.statusMessage;

    if (statusMessage === "") {
        // No results — empty string, not null or []
        return;
    }

    const records = statusMessage.data;   // array of result objects
    records.forEach(function (row) {
        console.log(row.Document_Category, row.count);
    });
});
```

### Response shape — `CONNECTION.invoke`

The CRM API response is nested inside the SDK wrapper:

```javascript
// Outer SDK envelope
{
  details: {
    statusMessage: {           // <-- actual CRM API response body lives here
      data: [ { ... }, ... ],
      info: { count: 5, more_records: false }
    }
  }
}

// When there are no results:
{
  details: {
    statusMessage: ""          // empty string — NOT null, NOT { data: [] }
  }
}
```

Always check for the empty-string case before accessing `.data`:

```javascript
const statusMessage = response.details.statusMessage;
if (!statusMessage || statusMessage === "") return;
const records = statusMessage.data ?? [];
```

### `plugin-manifest.json` connection declaration

```json
{
  "connections": [
    {
      "link_name": "crm_oauth_connection",
      "display_name": "CRM OAuth Connection",
      "connectors": ["zohocrm"],
      "scopes": { "zohocrm": ["ZohoCRM.modules.ALL"] }
    }
  ]
}
```

---

## Choosing Between the Two Patterns

| | `API.coql` | `CONNECTION.invoke` |
|---|---|---|
| Setup | None | Connection in manifest |
| Auth | Automatic | Via named connection |
| Blueprint widgets | May not work | Works |
| Aggregate / GROUP BY | May not surface | Works |
| Response path | `response.data` | `response.details.statusMessage.data` |
| Empty result | `response.data` is absent | `response.details.statusMessage === ""` |
| Recommended default | Yes | When `API.coql` is unavailable or insufficient |

---

## Common Query Patterns

### Bulk fetch by record IDs (from selected records)

Used in mass-communication, bulk-action widgets. `context.recordIds` comes from
the `PageLoad` event in list-view widgets.

```javascript
ZOHO.embeddedApp.on("PageLoad", async function (context) {
    const ids = context.recordIds.map(id => `'${id}'`).join(', ');
    const fields = ['First_Name', 'Last_Name', 'Email', 'Phone'];
    const select_query = `SELECT ${fields.join(', ')} FROM ${context.module} WHERE id in (${ids})`;

    const { data } = await ZOHO.CRM.API.coql({ select_query });
    const records = data ?? [];
    // ... process records
});
```

### Location / address search

```javascript
const select_query = `select First_Name, Last_Name, City, Street
                      from Contacts
                      where Street = '${street}' or City = '${city}'`;

const response = await ZOHO.CRM.API.coql({ select_query });
(response.data ?? []).forEach(record => { /* ... */ });
```

### Subform aggregate query

```javascript
const select_query = `select Document_Category, COUNT(id)
                      from Document_Details
                      where id is not null and Parent_Id = '${contactId}'
                      group by Document_Category`;

// Must use CONNECTION.invoke for this — API.coql may not support GROUP BY
```

### Paginated fetch (all records in a module)

```javascript
async function fetchAll(module, fields) {
    const results = [];
    let offset = 0;
    const limit = 200;

    while (true) {
        const select_query = `SELECT ${fields.join(', ')} FROM ${module} LIMIT ${limit} OFFSET ${offset}`;
        const response = await ZOHO.CRM.API.coql({ select_query });
        const page = response.data ?? [];
        results.push(...page);
        if (page.length < limit) break;
        offset += limit;
    }
    return results;
}
```

---

## Field Naming Rules

- Always use the **API name**, not the display label.
  - Display label: "Last Name" → API name: `Last_Name`
  - Display label: "Mobile" → API name: `Mobile`
- Custom fields end with `_c` in some older orgs; in current orgs they may not. Check the
  actual API name in CRM Setup > Modules and Fields.
- Lookup fields: use the field's own API name to get the referenced record's `id` and `name`.
  To get deeper fields from the related record you need a separate query — COQL does not
  support JOIN.
- Subform modules have their own field API names, separate from the parent module.

To discover API names programmatically:

```javascript
// List fields for a module
const meta = await ZOHO.CRM.META.getFields({ Entity: "Contacts" });
meta.fields.forEach(f => console.log(f.api_name, f.display_label));
```

---

## Limits and Gotchas

| Constraint | Value |
|------------|-------|
| Max query length | 25,000 characters |
| Max values in `in (...)` list | 200 |
| Max records returned per page | 200 |
| Pagination | LIMIT + OFFSET |
| Supported DML | SELECT only |
| JOINs | Not supported |
| Subqueries | Not supported |

**String quoting.** String literals must use single quotes. Double quotes are not valid COQL.

```sql
-- correct
WHERE Last_Name = 'Smith'

-- wrong
WHERE Last_Name = "Smith"
```

**SQL injection.** Interpolating user input directly into a query string is unsafe. Sanitize or
escape single quotes in any user-supplied value before embedding:

```javascript
const safe = userInput.replace(/'/g, "\\'");
const select_query = `SELECT id FROM Contacts WHERE Last_Name = '${safe}'`;
```

**Empty result difference.** `API.coql` omits the `data` key on empty results; always
use `response.data ?? []`. `CONNECTION.invoke` sets `statusMessage` to the empty string `""`
on empty results; check for that before accessing `.data`.

**`API.coql` and aggregates.** The official samples route all `COUNT` / `GROUP BY` queries
through `CONNECTION.invoke`. Treat `API.coql` + aggregates as untested — use
`CONNECTION.invoke` when you need them.

**Blueprint widget scope.** Inside a Blueprint transition widget `ZOHO.CRM.API.*` methods
may throw scope errors. Use `CONNECTION.invoke` for all CRM API calls from Blueprint widgets.

**Data types in results.** All field values come back as strings or numbers depending on the
field type; booleans come back as `true`/`false` (boolean). Dates come back as strings in
`YYYY-MM-DD` format. DateTimes come back in ISO 8601 format with timezone offset.

**Module names are case-sensitive.** `FROM contacts` will fail; use `FROM Contacts`.

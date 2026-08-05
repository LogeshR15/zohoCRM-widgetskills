# Official Zoho CRM Widget Code Samples — Complete Code Reference

Source: `https://www.zohocrm.dev/explore/widgets/codesamples`  
Metadata: `code_samples_file_list_new_v1.json`

## How to Use This File

This file contains the complete, verbatim source code for all 9 documented Zoho CRM widget samples extracted from the official zip files. Use it to:

- Copy working code patterns directly into a new widget
- Identify which SDK version, widget type, and CRM APIs are used for a given use case
- Understand the PageLoad data shape for each widget placement type
- Compare approaches (e.g., Blueprint widgets in samples 6, 9; ZDK.Client in sample 11)

**SDK target for new widgets: v1.5.** All samples below that use an older SDK version (v1.2, v1.0.5) should have their `<script src>` updated to:
```
https://live.zwidgets.com/js-sdk/1.5/ZohoEmbededAppSDK.min.js
```

---

## Quick Reference: Pattern Index

| Requirement | Sample(s) |
|-------------|-----------|
| Act on multiple selected records from List View | #1 (mass communication) |
| Dashboard tile with no record context | #2 (blog feed) |
| Cross-product data via ZRC (Desk, Books, etc.) | #3 |
| File upload inside a widget | #6 (mandate subform) |
| Validate data before Blueprint proceeds | #6, #9 |
| Google Maps / geolocation in widget | #7 |
| Time tracking with custom module | #8 |
| Widget opened by Client Script (Flyout) | #8, #10 |
| Return data from widget to Client Script | #10 (`$Client.close`) |
| ZDK.Client UI helpers (confirmation, popup, message) | #11 |
| Two coordinated widgets in one extension | #11 |
| All 28 API methods in one widget | #11 (widget1.html) |
| COQL for multi-record queries | #1, #6, #7 |
| Plotly.js data visualization in widget | #9 (CIBIL gauge) |

## SDK Versions Used Across Samples

| Sample | SDK version in zip |
|--------|--------------------|
| 1 — Custom Button (Twilio) | v1.2 |
| 2 — Dashboard Blog Feed | v1.2 |
| 3 — Desk Tickets ZRC | **v1.5** |
| 6 — Mandate Subform Blueprint | v1.2 |
| 7 — Geolocation Map | v1.2 |
| 8 — Timer Worklog | **v1.0.5** (oldest) |
| 9 — Blueprint Loan History | v1.2 |
| 10 — Widget in Popup | v1.2 |
| 11 — Escalation ZDK | Non-standard CDN ⚠️ |

> **Warning:** Sample 11 uses `https://app-assets.web.app/widgetsdk/ZohoEmbededAppSDK.min.js` — not the official zwidgets.com CDN. Do not use this URL in production. Replace with the official v1.5 URL.

---

## Sample 1 — Custom Button Widget (Mass Communication)

**Widget type:** Custom Button — List View  
**Module:** Leads  
**SDK version:** v1.2  
**Event:** PageLoad (receives `Entity` and `EntityId` array of selected records)

**Use case:** Bulk SMS and pre-recorded voice calls to selected Lead records from the List View using Twilio. Fetches phone field values dynamically via COQL, sends each number through Twilio's REST API.

**Key SDK patterns:**
- `ZOHO.embeddedApp.on("PageLoad", fn)` receives `{ Entity, EntityId }` where `EntityId` is an array of selected record IDs
- `ZOHO.CRM.META.getFields({ Entity })` to discover phone-type fields dynamically
- `ZOHO.CRM.API.coql({ select_query })` to bulk-fetch phone numbers for selected IDs using `WHERE id in (...)` clause
- `ZOHO.CRM.UI.Resize({ height, width })` to size the widget popup
- Twilio API called directly via `fetch()` (no CRM connection needed since credentials are hardcoded — see gotchas)

**Files:** `widget.html`, `twilio.js`

### widget.html

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="UTF-8">
	<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH" crossorigin="anonymous">
	<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/choices.js/public/assets/styles/choices.min.css">
	<style>
		.custom-alert { position: fixed; top: 0; left: 0; right: 0; display: flex; justify-content: center; align-items: center; }
		.alert-content { padding: 10px; font-size: 14px; border-radius: 5px; text-align: center; display: flex; align-items: center; gap: 20px; }
		.alert-content.success { background-color: rgb(223, 254, 206); border-left: 3px solid rgb(104, 182, 62); }
		.alert-content.error { background-color: rgb(254, 206, 206); border-left: 3px solid rgb(182, 62, 62); }
		.btn-close { width: 5px !important; height: 5px !important; }
		.hidden { display: none; }
	</style>
</head>
<body>
	<div class="d-flex flex-column gap-4 p-4">
		<div>
			<label for="phone-fields-dropdown" class="form-label">Phone Fields</label>
			<select id="phone-fields-dropdown" class="form-select" multiple></select>
		</div>
		<ul class="nav nav-tabs" id="myTab" role="tablist">
			<li class="nav-item" role="presentation">
				<a class="nav-link active" id="sms-tab" data-bs-toggle="tab" href="#sms" role="tab">SMS</a>
			</li>
			<li class="nav-item" role="presentation">
				<a class="nav-link" id="voice-call-tab" data-bs-toggle="tab" href="#voice-call" role="tab">Voice Call</a>
			</li>
		</ul>
		<div class="tab-content">
			<div class="tab-pane show active" id="sms" role="tabpanel">
				<label for="sms-input" class="form-label">Message</label>
				<textarea class="form-control" id="sms-input" rows="3" placeholder="Enter Text Message"></textarea>
				<div class="d-flex justify-content-center mt-4">
					<button class="btn btn-primary" id="sms-button" onclick="onSubmit('sms')">Send SMS</button>
				</div>
			</div>
			<div class="tab-pane" id="voice-call" role="tabpanel">
				<label for="voice-call-input" class="form-label">Text to Speech</label>
				<textarea class="form-control" id="voice-call-input" rows="3" placeholder="Enter Voice Message"></textarea>
				<div class="d-flex justify-content-center mt-4">
					<button class="btn btn-primary" id="voice-call-button" onclick="onSubmit('voice-call')">Call now</button>
				</div>
			</div>
		</div>
	</div>
	<div id="custom-alert" class="custom-alert hidden">
		<div class="alert-content">
			<div id="alert-message"></div>
			<button type="button" class="btn-close" onclick="closeAlert()"></button>
		</div>
	</div>
	<script src="https://code.jquery.com/jquery-1.12.4.min.js" integrity="sha384-nvAa0+6Qg9clwYCGGPpDQLVpLNn0fRaROjHqs13t4Ggj3Ez50XnGQqc/r8MhnRDZ" crossorigin="anonymous"></script>
	<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js" integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz" crossorigin="anonymous"></script>
	<script src="https://cdn.jsdelivr.net/npm/choices.js/public/assets/scripts/choices.min.js"></script>
	<script src="https://live.zwidgets.com/js-sdk/1.2/ZohoEmbededAppSDK.min.js"></script>
	<script src="twilio.js"></script>
	<script>
		const context = {};
		async function getPhoneNumbers() {
			const phoneFieldsDropdown = document.getElementById('phone-fields-dropdown');
			const selectedValues = Array.from(phoneFieldsDropdown.options).filter(option => option.selected).map(option => option.value);
			if (!selectedValues.length) { return showAlert('error', 'Please select atleast one phone field'); }
			const ids = context.recordIds.map(x => `'${x}'`).join(', ');
			const select_query = `SELECT ${selectedValues.join(', ')} FROM ${context.module} WHERE id in (${ids})`;
			const { data } = await ZOHO.CRM.API.coql({ select_query });
			const phoneNumbers = [];
			data.forEach(x => { Object.keys(x).forEach(key => { if (key !== 'id' && x[key]) { phoneNumbers.push(x[key]); } }); });
			return phoneNumbers;
		}
		async function onSubmit(action) {
			const text = document.getElementById(`${action}-input`).value;
			if (!text?.length) { return showAlert('error', 'Message cannot be empty'); }
			const phoneNumbers = await getPhoneNumbers();
			if (!phoneNumbers) { return; }
			if (!phoneNumbers.length) { return showAlert('error', 'No phone numbers available in the selected phone fields for the selected records'); }
			switch (action) {
				case 'sms': await twilio.sendSMS(text, phoneNumbers); break;
				case 'voice-call': await twilio.makePhoneCall(text, phoneNumbers); break;
			}
			showAlert('success', 'Sent Successfully');
		}
		ZOHO.embeddedApp.on("PageLoad", function (data) {
			const { Entity, EntityId } = data;
			context.module = Entity;
			context.recordIds = EntityId;
			ZOHO.CRM.META.getFields({ Entity }).then(function (response) {
				const { fields } = response;
				const phoneFields = fields.filter(x => x.data_type === 'phone').map(x => ({ value: x.api_name, label: x.display_label }));
				new Choices('#phone-fields-dropdown', { removeItemButton: true, choices: phoneFields, placeholder: true, placeholderValue: 'Select Phone Fields to be used' });
				ZOHO.CRM.UI.Resize({height:"400", width:"700"});
			});
			twilio.init("$YOUR_ACCOUNT_SID", "$YOUR_AUTH_TOKEN", "+YOUR_TWILIO_NUMBER");
		})
		ZOHO.embeddedApp.init();
		function showAlert(type, msg) {
			const alert = document.getElementById("custom-alert");
			const content = alert.querySelector('.alert-content');
			content.classList.add(type);
			alert.querySelector('#alert-message').textContent = msg;
			alert.classList.remove('hidden');
		}
		function closeAlert() {
			const alert = document.getElementById("custom-alert");
			alert.classList.add('hidden');
			const content = alert.querySelector('.alert-content');
			content.classList.remove('error');
			content.classList.remove('success');
		}
	</script>
</body>
</html>
```

### twilio.js

```javascript
const twilio = {
	init: function(accountSid, authToken, fromNumber) {
		this.accountSid = accountSid;
		this.fromNumber = fromNumber;
		this.token = btoa(`${accountSid}:${authToken}`);
	},
	__makeRequest: async function(method, url, data) {
		return await fetch(`https://api.twilio.com/2010-04-01/Accounts/${this.accountSid}/${url}`, {
			method,
			headers: { 'Authorization': `Basic ${this.token}`, 'Content-Type': 'application/x-www-form-urlencoded' },
			body: data
		}).then(response => response.json());
	},
	sendSMS: async function(message, phoneNumbers) {
		for (let toNumber of phoneNumbers) {
			const data = new URLSearchParams({ Body: message, From: this.fromNumber, To: toNumber });
			await this.__makeRequest('POST', 'Messages.json', data);
		}
	},
	makePhoneCall: async function(text, phoneNumbers) {
		const twiml = `<?xml version="1.0" encoding="UTF-8"?><Response><Say>${text}</Say></Response>`;
		for (let toNumber of phoneNumbers) {
			const data = new URLSearchParams({ Twiml: twiml, From: this.fromNumber, To: toNumber });
			await this.__makeRequest('POST', 'Calls.json', data);
		}
	}
}
```

**Gotchas:**
- Twilio credentials (`$YOUR_ACCOUNT_SID`, `$YOUR_AUTH_TOKEN`) are hardcoded as placeholders. In production, use `ZOHO.CRM.CONNECTION.invoke` with a custom connection to avoid exposing secrets in widget code.
- `ZOHO.CRM.API.coql` with `WHERE id in (...)` requires IDs to be wrapped in single quotes within the string.
- The `Choices.js` multi-select is initialized inside `PageLoad` — do not call it before the SDK is ready.

---

## Sample 2 — Blog Feed Dashboard Widget

**Widget type:** Dashboard  
**Module:** N/A (org-level)  
**SDK version:** v1.2  
**Event:** PageLoad (no record context — dashboard widgets receive no `Entity` or `EntityId`)

**Use case:** Displays latest blog posts stored as HTML in a CRM org variable. A scheduled Deluge function updates the variable periodically; the widget reads and renders it. Demonstrates the dashboard widget pattern and using `ZOHO.CRM.CONNECTION.invoke` to call the CRM Settings API.

**Key SDK patterns:**
- Dashboard widget PageLoad receives no entity data — no need to destructure `data`
- `ZOHO.CRM.CONNECTION.invoke(conn_name, req_data)` with `param_type: 1` for GET requests
- Reading org variable by ID via `/crm/v6/settings/variables/{id}`
- Response path: `data.details.statusMessage.variables[0].value`

**Files:** `widget.html` only

```html
<html>
<head>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: sans-serif; }
        html, body { height: 100%; width: 100%; }
        #blogs-table-container { height: 100%; display: grid; place-items: center; }
        table { width: 100%; border-spacing: 0; border: 1px solid #EDF0F4; }
        thead { background-color: #f8f9fb; }
        th, td { padding: 10px; text-align: center; border-bottom: 1px solid #EDF0F4; }
        tbody tr:hover { background-color: #f4f7fe; }
    </style>
	<script src="https://live.zwidgets.com/js-sdk/1.2/ZohoEmbededAppSDK.min.js"></script>
</head>
<body>
	<div id="blogs-table-container"></div>
	<script type="text/javascript">
		ZOHO.embeddedApp.on("PageLoad", function () {
			var conn_name = "crm_oauth_connection";
			var req_data = {
				"method": "GET",
				"url": "https://www.zohoapis.com/crm/v6/settings/variables/5843104000002135015",
				"param_type": 1
			};
			ZOHO.CRM.CONNECTION.invoke(conn_name, req_data).then(function (data) {
				const div = document.getElementById("blogs-table-container");
				div.innerHTML = data.details.statusMessage.variables[0].value;
			})
		})
		ZOHO.embeddedApp.init();
	</script>
</body>
</html>
```

**Gotchas:**
- The org variable ID (`5843104000002135015`) is hardcoded — replace with your actual variable ID.
- The connection name `crm_oauth_connection` must be defined in the widget's `plugin-manifest.json` under `connections`.
- The value stored in the org variable is expected to be a pre-built HTML string (table markup). The Deluge function that populates it is not included in this sample.

---

## Sample 3 — Desk Ticket Details via ZRC Widget

**Widget type:** Record detail — Related List placement  
**Module:** Contacts  
**SDK version:** v1.5  
**Event:** PageLoad (receives `Entity`, `EntityId`, and `related_list` object)

**Use case:** Shows open Zoho Desk tickets for a Contact record by querying Desk's API via a CRM connection. Uses the ZRC (Zoho Resource Client) unified syntax available in v1.5 to fetch the Contact's email, then queries Desk for open tickets matching that email.

**Key SDK patterns:**
- v1.5 `zrc.get()` unified syntax for both CRM API calls and cross-product API calls
- `zrc.get(url, { connection: "connection_name" })` for authenticated external API calls
- Related list `PageLoad` data shape includes `data.related_list.associated_feature_resource`
- Fetches contact record first, extracts email, then queries Desk

**Files:** `widget.html` only

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Open Desk Tickets</title>
    <script src="https://live.zwidgets.com/js-sdk/1.5/ZohoEmbededAppSDK.min.js"></script>
    <style>
        body { font-family: "Segoe UI", Arial, sans-serif; font-size: 14px; margin: 0; padding: 20px; background-color: #fafafa; }
        h3 { margin-bottom: 12px; font-size: 16px; font-weight: 600; color: #333; }
        table { width: 100%; border-collapse: collapse; background: #fff; border-radius: 8px; overflow: hidden; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
        th, td { padding: 10px 12px; text-align: left; }
        th { background: #f5f7fa; font-weight: 600; color: #555; border-bottom: 1px solid #e1e5eb; }
        tr:nth-child(even) { background-color: #fafbfc; }
        tr:hover { background-color: #f0f6ff; }
        td { color: #444; border-bottom: 1px solid #eaeef3; }
        td a { color: #0073e6; font-weight: 500; text-decoration: none; }
        .status { display: inline-block; padding: 2px 8px; border-radius: 12px; font-size: 12px; font-weight: 500; color: #fff; background: #0073e6; }
        .priority-high { color: #d93025; font-weight: 600; }
        .priority-medium { color: #f9a825; font-weight: 600; }
        .priority-low { color: #188038; font-weight: 600; }
        .empty { background: #fff; border: 1px dashed #ccc; padding: 20px; text-align: center; border-radius: 8px; color: #777; }
    </style>
</head>
<body>
    <h3>Open Zoho Desk Tickets</h3>
    <div id="tickets">Loading...</div>
    <script>
        ZOHO.embeddedApp.on("PageLoad", async function (data) {
            const pageData = {
                Entity: data.Entity,
                EntityId: data.EntityId,
                related_list: {
                    associated_feature_resource: {
                        name: data.related_list.associated_feature_resource.name,
                        api_name: data.related_list.associated_feature_resource.api_name,
                        id: data.related_list.associated_feature_resource.id
                    },
                    associated_feature_type: data.related_list.associated_feature_type,
                    id: data.related_list.id,
                    label: data.related_list.label
                }
            }
            const contactRes = await zrc.get(`/crm/v8/${pageData.Entity}/${pageData.EntityId}`);
            const ZOHO_DESK_URL = "https://desk.zoho.com";
            const contactEmail = contactRes.data && contactRes.data.data[0] && contactRes.data.data[0].Email;
            if (!contactEmail) {
                document.getElementById("tickets").innerHTML = "<p>No email found for this Account.</p>";
                return;
            }
            try {
                const deskRes = await zrc.get(`${ZOHO_DESK_URL}/api/v1/tickets?limit=10&status=Open&email=${encodeURIComponent(contactEmail)}`, {
                    connection: "zoho_desk_connection_name",
                });
                if (!deskRes || !deskRes.data) { throw new Error("No response from Zoho Desk"); }
                const tickets = deskRes.data || [];
                let html = "<table><tr><th>ID</th><th>Subject</th><th>Status</th><th>Priority</th><th>Created</th></tr>";
                tickets.forEach(t => {
                    const link = `https://desk.zoho.com/support/yourportal#Cases/${t.id}`;
                    html += `<tr><td><a href="${link}" target="_blank">${t.ticketNumber}</a></td><td>${t.subject}</td><td>${t.status}</td><td>${t.priority || '-'}</td><td>${new Date(t.createdTime).toLocaleString()}</td></tr>`;
                });
                html += "</table>";
                document.getElementById("tickets").innerHTML = html;
            } catch (err) {
                console.error("Error fetching tickets:", err);
                document.getElementById("tickets").innerHTML = "<p>Failed to load tickets.</p>";
            }
        });
        ZOHO.embeddedApp.init();
    </script>
</body>
</html>
```

**Gotchas:**
- `zrc` is a global available only in SDK v1.5+. This will not work with v1.2 or older.
- The connection name `zoho_desk_connection_name` must be declared in `plugin-manifest.json`.
- `zrc.get("/crm/v8/...")` uses a relative path for CRM APIs; external APIs require the full URL plus a `connection` option.
- The Desk portal URL in the ticket link (`yourportal`) must be replaced with your actual Desk portal name.
- `data.related_list` is only present when the widget is placed as a Related List — not available in other placements.

---

## Sample 6 — Mandate Subform Data for Blueprint Transitions

**Widget type:** Blueprint — Deals module  
**Module:** Deals (reads/writes to Contacts via lookup)  
**SDK version:** v1.2  
**Event:** PageLoad (receives `Entity` and `EntityId` of the Deal record in transition)

**Use case:** Enforces document subform data collection before allowing a Blueprint state transition. On load, uses COQL via CONNECTION to check how many document categories are already filled in the Contact's `Document_Details` subform. If fewer than 4, shows a data-entry form with file upload. Only enables the "Move to Next State" button after all 4 categories are filled. Uses `ZOHO.CRM.BLUEPRINT.proceed()` to advance the transition.

**Key SDK patterns:**
- `ZOHO.CRM.BLUEPRINT.proceed()` to advance the Blueprint state
- `ZOHO.CRM.UI.Popup.close()` to close the Blueprint popup
- `ZOHO.CRM.API.uploadFile(config)` with multipart config for file uploads
- `ZOHO.CRM.CONNECTION.invoke` with `param_type: 2` and JSON body for POST requests (COQL)
- Subform update: include existing subform record IDs plus new entries in `APIData`

**Files:** `widget.html`, `widget.js`, `widget.css`

### widget.html

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="UTF-8">
	<title>Zoho CRM Blueprint Widget</title>
	<link rel="stylesheet" type="text/css" href="widget.css">
	<script src="https://live.zwidgets.com/js-sdk/1.2/ZohoEmbededAppSDK.min.js"></script>
</head>
<body>
	<div id="contentContainer" style="display: none;">
		<h2>Fill in document details for all categories</h2>
		<div id="subformContainer"></div>
		<div id="buttonContainer" class="button-container">
			<button class="nextStateButton">Move to Next State</button>
			<button class="closePopupButton">Close</button>
			<button class="saveButton" type="submit" form="subform">Save</button>
		</div>
	</div>
	<script src="widget.js"></script>
	<div id="messageContainer" style="display: none; text-align: center;">
		<div class="image-container">
			<img src="https://www.zoho.com/art/gallery/images/data-collection-game.png" alt="Data Collection" class="responsive-image">
		</div>
		<h2>You have already gathered all the document details!</h2>
		<div class="button-container">
			<button class="nextStateButton">Move to Next State</button>
			<button class="closePopupButton">Close</button>
		</div>
	</div>
</body>
</html>
```

### widget.js

```javascript
document.addEventListener('DOMContentLoaded', function () {
	let contactId;
	ZOHO.embeddedApp.on("PageLoad", function (data) {
		const recordId = data.EntityId;
		const moduleName = data.Entity;
		ZOHO.CRM.API.getRecord({ Entity: moduleName, RecordID: recordId }).then(function (response) {
			const record = response.data[0];
			contactId = record.Contact_Name.id;
			const query = { "select_query": `select Document_Category,COUNT(id) from Document_Details where id is not null and Parent_Id='${contactId}' group by Document_Category` };
			ZOHO.CRM.CONNECTION.invoke("crm_oauth_connection", {
				method: "POST", url: "https://www.zohoapis.com/crm/v7/coql",
				parameters: query, param_type: 2, headers: { "Content-Type": "application/json" }
			}).then(function (response) {
				const data = response.details.statusMessage.data;
				if (response.details.statusMessage === "" || (data && data.length < 4)) {
					createSubformUI();
					document.getElementById('contentContainer').style.display = 'block';
				} else if (data && data.length === 4) {
					document.getElementById('messageContainer').style.display = 'block';
					const nextStateButton = document.querySelector('.nextStateButton');
					nextStateButton.disabled = false;
					nextStateButton.style.backgroundColor = '';
				}
			});
		});
	});
	ZOHO.embeddedApp.init();

	function createSubformUI() {
		const subformContainer = document.getElementById('subformContainer');
		subformContainer.innerHTML = `
			<form id="subform">
				<table id="documentsTable">
					<thead><tr><th>Document Category</th><th>Document Name</th><th>Document Number</th><th>Upload</th></tr></thead>
					<tbody>
						<tr>
							<td><select id="documentCategory" name="documentCategory" required>
								<option value="Address Proof">Address Proof</option>
								<option value="Bank Statement">Bank Statement</option>
								<option value="ID Proof">ID Proof</option>
								<option value="Income Proof">Income Proof</option>
								<option value="Medical Proof">Medical Proof</option>
							</select></td>
							<td><input type="text" id="documentName" name="documentName" required></td>
							<td><input type="number" id="documentNumber" name="documentNumber" required></td>
							<td><input type="file" id="documentFile" name="upload" required></td>
						</tr>
					</tbody>
				</table>
				<button type="button" id="addRowButton" class="add-row-button">Add Row</button>
			</form>`;
		document.getElementById('addRowButton').addEventListener('click', addRow);
		document.getElementById('subform').addEventListener('submit', updateRecord);
		const nextStateButton = document.querySelector('.nextStateButton');
		nextStateButton.disabled = true;
		nextStateButton.style.backgroundColor = 'grey';
	}

	function addRow() {
		const table = document.getElementById('documentsTable').getElementsByTagName('tbody')[0];
		const newRow = table.insertRow();
		newRow.insertCell(0).innerHTML = `<select name="documentCategory" required><option value="Address Proof">Address Proof</option><option value="ID Proof">ID Proof</option><option value="Income Proof">Income Proof</option><option value="Medical Proof">Medical Proof</option></select>`;
		newRow.insertCell(1).innerHTML = `<input type="text" name="documentName" required>`;
		newRow.insertCell(2).innerHTML = `<input type="number" name="documentNumber" required>`;
		newRow.insertCell(3).innerHTML = `<input type="file" name="upload" required>`;
		if (table.rows.length >= 4) { document.getElementById('addRowButton').style.display = 'none'; }
	}

	async function uploadFile(file) {
		var config = {
			"CONTENT_TYPE": "multipart",
			"PARTS": [{ "headers": { "Content-Disposition": "file;" }, "content": "__FILE__" }],
			"FILE": { "fileParam": "content", "file": file }
		};
		let fileId;
		await ZOHO.CRM.API.uploadFile(config).then(function (response) { fileId = response.data[0].details.id; });
		return fileId;
	}

	async function getExisitingSubFormRecords(contactId) {
		let exisitingRecords = [];
		await ZOHO.CRM.API.getRecord({ Entity: "Contacts", RecordID: contactId }).then(function (response) {
			const record = response.data[0];
			if (record.Document_Details) { record.Document_Details.forEach(function (document) { exisitingRecords.push({id: document.id}); }); }
		});
		return exisitingRecords;
	}

	async function updateRecord(event) {
		event.preventDefault();
		const rows = document.querySelectorAll('#documentsTable tbody tr');
		const documentDetails = await Promise.all(Array.from(rows).map(async (row) => {
			const documentCategory = row.querySelector('select[name="documentCategory"]').value;
			const documentNumber = row.querySelector('input[name="documentNumber"]').value;
			const documentName = row.querySelector('input[name="documentName"]').value;
			const documentFile = row.querySelector('input[name="upload"]').files[0];
			const fileId = await uploadFile(documentFile);
			return { Document_Category: documentCategory, Document_Number: documentNumber, Document_Name: documentName, Upload: [{ file_id: fileId }] };
		}));
		const existingRecords = await getExisitingSubFormRecords(contactId);
		existingRecords.forEach(record => documentDetails.push(record));
		const payload = { id: contactId, Document_Details: documentDetails };
		ZOHO.CRM.API.updateRecord({ Entity: "Contacts", APIData: payload }).then(function (updateResponse) {
			const saveButton = document.querySelector('.saveButton');
			saveButton.textContent = "Saved";
			saveButton.disabled = false;
			saveButton.style.backgroundColor = 'grey';
			const nextStateButton = document.querySelector('.nextStateButton');
			nextStateButton.disabled = false;
			nextStateButton.style.backgroundColor = '';
		});
	}

	document.querySelectorAll('.nextStateButton').forEach(button => {
		button.addEventListener('click', function () {
			ZOHO.CRM.BLUEPRINT.proceed().then(function () {
				ZOHO.CRM.UI.Popup.close();
			});
		});
	});
	document.querySelectorAll('.closePopupButton').forEach(button => {
		button.addEventListener('click', function () { ZOHO.CRM.UI.Popup.close(); });
	});
});
```

**Gotchas:**
- When updating a subform, you must include the existing subform record IDs (`{ id: ... }`) along with new entries, or existing rows will be deleted.
- `ZOHO.CRM.API.uploadFile` uses a special config with `"__FILE__"` as a placeholder string — the actual file object goes in the `FILE.file` property.
- The COQL query against `Document_Details` (a subform) uses `Parent_Id` to filter by the parent Contact ID — this is a subform-specific COQL pattern.
- The widget uses `ZOHO.CRM.CONNECTION.invoke` for COQL (POST) rather than `ZOHO.CRM.API.coql` because subform COQL is done against v7, while `ZOHO.CRM.API.coql` may use a different version.
- `widget.css` content is not included in the extracted zip — add your own styles.

---

## Sample 7 — Geolocating Leads' Addresses (Map in Related List)

**Widget type:** Custom Related List — Leads detail page  
**Module:** Leads  
**SDK version:** v1.2  
**Event:** PageLoad (receives `Entity` and `EntityId` of the current Lead record)

**Use case:** Embeds a Google Maps view on a Lead's detail page showing: the lead's address as a destination marker, directions from the current user's browser location, and markers for nearby leads in the same city or street (fetched via COQL).

**Key SDK patterns:**
- `ZOHO.CRM.API.getRecord` to fetch the current Lead's address fields
- `ZOHO.CRM.API.coql` with `WHERE Street='...' or City='...'` to find nearby leads
- Browser `navigator.geolocation.getCurrentPosition` for user's current location
- Google Maps JS API loaded with `async defer` and `callback=initializeWidget`
- `ZOHO.embeddedApp.init()` called inside the Maps callback to ensure Maps is ready first

**Files:** `widget.html` only

```html
<html>
<head>
	<script src="https://live.zwidgets.com/js-sdk/1.2/ZohoEmbededAppSDK.min.js"></script>
</head>
<body>
	<script src="https://maps.googleapis.com/maps/api/js?key=YOUR_GOOGLE_MAPS_API_KEY&callback=initializeWidget" async defer></script>
	<div id="map"></div>
	<script type="text/javascript">
		function initializeWidget() {
			if ('geolocation' in navigator) {
				ZOHO.embeddedApp.on("PageLoad", function (data) {
					if (data && data.Entity) {
						ZOHO.CRM.API.getRecord({ Entity: data.Entity, RecordID: data.EntityId }).then(function (response) {
							const street = response.data[0].Street;
							const city = response.data[0].City;
							const coqlQuery = `select First_Name,Last_Name,City,Street from ${data.Entity} where Street='${street}' or City='${city}'`;
							ZOHO.CRM.API.coql({ 'select_query': coqlQuery }).then(function (coqlResponse) {
								navigator.geolocation.getCurrentPosition(function (position) {
									let map = new google.maps.Map(document.getElementById("map"), { zoom: 10, center: { lat: position.coords.latitude, lng: position.coords.longitude } });
									let directionsService = new google.maps.DirectionsService();
									let directionsDisplay = new google.maps.DirectionsRenderer();
									directionsDisplay.setMap(map);
									let geocoder = new google.maps.Geocoder();
									let destination = `${street},${city}`;
									geocoder.geocode({ address: destination }, function (results, status) {
										if (status === "OK" && results[0].geometry) {
											new google.maps.Marker({ map, position: results[0].geometry.location, title: "Destination" });
											let locations = coqlResponse.data.map(data => ({ address: `${data.Street},${data.City}`, title: data.First_Name + " " + data.Last_Name }));
											for (let loc of locations) { geocodeAndAddMarker(loc.address, loc.title, map); }
											calculateAndDisplayRoute(directionsService, directionsDisplay, { lat: position.coords.latitude, lng: position.coords.longitude }, destination);
										}
									});
								});
							});
						});
					}
				});
				ZOHO.embeddedApp.init();
			}
		}
		function geocodeAndAddMarker(address, title, map) {
			new google.maps.Geocoder().geocode({ address }, function (results, status) {
				if (status === "OK") { new google.maps.Marker({ map, position: results[0].geometry.location, title }); }
			});
		}
		function calculateAndDisplayRoute(directionsService, directionsDisplay, origin, destination) {
			directionsService.route({ origin, destination, travelMode: google.maps.TravelMode.DRIVING }, function (response, status) {
				if (status === google.maps.DirectionsStatus.OK) { directionsDisplay.setDirections(response); }
			});
		}
		document.getElementById("map").style.height = window.innerHeight + "px";
	</script>
</body>
</html>
```

**Gotchas:**
- `YOUR_GOOGLE_MAPS_API_KEY` must be replaced with a real key that has the Maps JS API, Geocoding API, and Directions API enabled.
- `ZOHO.embeddedApp.init()` and `ZOHO.embeddedApp.on("PageLoad", ...)` are called inside `initializeWidget` (the Maps callback), not at the top level. This ensures Maps is loaded before the SDK tries to init. If you reverse this order, Maps objects will be undefined when PageLoad fires.
- The COQL query concatenates field values directly into the string — this will break if the street or city contains a single quote. Sanitize or escape input for production use.
- `document.getElementById("map").style.height` is set before the map div is passed to the Maps API — this line runs at parse time, so the div exists in DOM by then.

---

## Sample 8 — Timer and Worklog Widget

**Widget type:** Custom Button rendered in a Flyout via Client Script  
**Module:** Cases (reads) + Timer_Entries (custom module, writes)  
**SDK version:** v1.0.5 (oldest version used across all samples)  
**Event:** PageLoad (no entity context — opened as a flyout by Client Script)

**Use case:** Time tracking widget. The user starts a timer (creates a `Timer_Entries` record with `Start_Time`), optionally enters a work description, selects a related Case, then stops the timer (updates the record with `End_Time`). Uses two custom view IDs (cvid) to fetch records — one for Cases, one for the latest Timer_Entries record.

**Key SDK patterns:**
- `ZOHO.CRM.CONFIG.getCurrentUser()` to get the logged-in user's ID
- `ZOHO.CRM.API.getAllRecords({ Entity, cvid, per_page })` to fetch records from a specific custom view
- `ZOHO.CRM.API.insertRecord` to create the timer entry on START
- `ZOHO.CRM.API.updateRecord` to write `End_Time` on STOP (fetches the latest record first)
- IST timezone formatting done manually (UTC+330 minutes offset)

**Files:** `widget.html` only

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <script src="https://crm.zoho.com/crm/javascript/v2/ZohoCRM.min.js"></script>
    <script src="https://live.zwidgets.com/js-sdk/1.0.5/ZohoEmbededAppSDK.min.js"></script>
    <style>
        body { font-family: Arial, sans-serif; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; background-color: #f0f0f0; }
        .timer-widget { background: #fff; padding: 20px; border-radius: 10px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); text-align: center; }
        .timer { font-size: 2rem; margin: 20px 0; }
        button { font-size: 1rem; padding: 10px 20px; border: none; border-radius: 5px; cursor: pointer; background-color: #007bff; color: white; }
        select, textarea { width: 100%; margin: 10px 0; }
        #errorMessage { color: red; font-size: 0.9rem; margin-top: 5px; }
    </style>
</head>
<body>
    <div class="timer-widget">
        <div class="timer" id="timer">00:00:00</div>
        <button id="toggleButton">START</button><br><br>
        <label for="workDescription">Work Description</label><br><br>
        <textarea id="workDescription" rows="3" placeholder="Enter work description"></textarea>
        <div id="errorMessage"></div><br><br>
        <label for="moduleRecords">Record</label><br>
        <select id="moduleRecords"></select>
    </div>
    <script>
        let timerInterval, elapsedTime = 0, isRunning = false, currentUserId = "";
        const casesCVID = "YOUR_CASES_CVID";
        const timerEntriesCVID = "YOUR_TIMER_ENTRIES_CVID";
        const timerModule = "Timer_Entries";
        const casesModule = "Cases";

        async function populateRecordsDropdown() {
            const recordsDropdown = document.getElementById("moduleRecords");
            recordsDropdown.innerHTML = "";
            const recordsResponse = await ZOHO.CRM.API.getAllRecords({ Entity: casesModule, cvid: casesCVID, per_page: 10 });
            if (recordsResponse.data && recordsResponse.data.length > 0) {
                recordsResponse.data.forEach(record => {
                    const option = document.createElement("option");
                    option.value = record.id;
                    option.textContent = record.Subject || "Unnamed Record";
                    recordsDropdown.appendChild(option);
                });
            }
        }

        function formatTime(t) {
            return [Math.floor(t/3600), Math.floor((t%3600)/60), t%60].map(n => String(n).padStart(2,'0')).join(':');
        }

        function getISTTime() {
            const now = new Date();
            const ist = new Date(now.getTime() + 330 * 60 * 1000);
            return `${ist.getUTCFullYear()}-${String(ist.getUTCMonth()+1).padStart(2,'0')}-${String(ist.getUTCDate()).padStart(2,'0')}T${String(ist.getUTCHours()).padStart(2,'0')}:${String(ist.getUTCMinutes()).padStart(2,'0')}:${String(ist.getUTCSeconds()).padStart(2,'0')}+05:30`;
        }

        async function createRecord(startTime) {
            const workDescription = document.getElementById("workDescription").value;
            const selectedRecordId = document.getElementById("moduleRecords").value;
            const selectedRecordText = document.getElementById("moduleRecords").options[document.getElementById("moduleRecords").selectedIndex].text;
            await ZOHO.CRM.API.insertRecord({ Entity: timerModule, APIData: { Start_Time: startTime, Owner: currentUserId, Work_Description: workDescription, Related_to_Case: selectedRecordId, Name: selectedRecordText } });
        }

        async function updateRecord(endTime) {
            const workDescription = document.getElementById("workDescription").value;
            const selectedRecordId = document.getElementById("moduleRecords").value;
            const response = await ZOHO.CRM.API.getAllRecords({ Entity: timerModule, cvid: timerEntriesCVID, per_page: 1 });
            const latestRecord = response.data[0];
            if (latestRecord) {
                await ZOHO.CRM.API.updateRecord({ Entity: timerModule, APIData: { id: latestRecord.id, End_Time: endTime, Work_Description: workDescription, Related_to_Case: selectedRecordId } });
            }
        }

        async function toggleTimer() {
            const button = document.getElementById('toggleButton');
            const currentTime = getISTTime();
            if (isRunning) {
                clearInterval(timerInterval);
                button.textContent = 'START';
                await updateRecord(currentTime);
                elapsedTime = 0;
                document.getElementById('timer').textContent = formatTime(elapsedTime);
                document.getElementById('workDescription').value = "";
            } else {
                elapsedTime = 0;
                timerInterval = setInterval(() => { elapsedTime++; document.getElementById('timer').textContent = formatTime(elapsedTime); }, 1000);
                button.textContent = 'STOP';
                await createRecord(currentTime);
            }
            isRunning = !isRunning;
        }

        document.getElementById('toggleButton').addEventListener('click', function () {
            if (document.getElementById('toggleButton').textContent.trim() === "STOP") {
                if (!document.getElementById('workDescription').value.trim()) {
                    document.getElementById('errorMessage').textContent = "Work Description is mandatory.";
                    return;
                }
                document.getElementById('errorMessage').textContent = "";
            }
            toggleTimer();
        });

        ZOHO.embeddedApp.on("PageLoad", async function () {
            const userResponse = await ZOHO.CRM.CONFIG.getCurrentUser();
            currentUserId = userResponse.users[0].id;
            populateRecordsDropdown();
        });
        ZOHO.embeddedApp.init();
    </script>
</body>
</html>
```

**Gotchas:**
- This sample loads two scripts: `ZohoCRM.min.js` (the old v2 CRM JS SDK) AND `ZohoEmbededAppSDK.min.js`. For new widgets, only the embedded app SDK is needed — do not include the v2 CRM JS SDK.
- `casesCVID` and `timerEntriesCVID` are hardcoded placeholder strings — replace with the actual custom view IDs from your org.
- The `updateRecord` function fetches the "latest" timer entry by using a custom view sorted by created time. If multiple users run timers simultaneously, this can fetch the wrong record. The custom view must be configured correctly.
- The IST offset is hardcoded (+05:30). For multi-timezone orgs, use `ZOHO.CRM.CONFIG.getCurrentUser()` to get timezone info or use the browser's `Intl` API.
- `Timer_Entries` is a custom module — it must be created in the org with the fields `Start_Time`, `End_Time`, `Work_Description`, `Related_to_Case`, `Owner`.

---

## Sample 9 — Widget Between Blueprint Transition States (Loan History)

**Widget type:** Blueprint — Loans module  
**Module:** Loans (custom), reads from Contacts and a `Loan_History` related list  
**SDK version:** v1.2  
**Event:** PageLoad (receives `Entity` and `EntityId` of the Loan record in transition)

**Use case:** During a loan approval Blueprint transition, displays the applicant's contact details, full loan history in a table (read from a related list), and a CIBIL credit score gauge chart (rendered with Plotly.js). Score is reduced if any loan has a "Defaults" status. Agent uses "Approve" (`ZOHO.CRM.BLUEPRINT.proceed()`) or "Decline" (`ZOHO.CRM.UI.Popup.close()`) buttons.

**Key SDK patterns:**
- `ZOHO.CRM.API.getRelatedRecords({ Entity, RecordID, RelatedList, page, per_page })` to fetch related list data
- `ZOHO.CRM.BLUEPRINT.proceed()` for approval
- `ZOHO.CRM.UI.Popup.close()` for decline / cancel
- Plotly.js gauge chart with color-coded ranges for credit score visualization
- Chained API calls: Loan record → Contact record → Loan History related list

**Files:** `index.html`, `index.js`, `index.css`

### index.html

```html
<!DOCTYPE html>
<html>
<head>
  <title>User Details</title>
  <link rel="stylesheet" href="index.css">
  <script src="https://live.zwidgets.com/js-sdk/1.2/ZohoEmbededAppSDK.min.js"></script>
  <script src="https://cdn.plot.ly/plotly-latest.min.js"></script>
  <script src="index.js"></script>
</head>
<body>
  <div class="container">
    <div class="add-flex">
      <div class="details">
        <h2>User Details</h2>
        <h3>Basic Details</h3>
        <p><strong>Name : </strong> <span id="name"></span></p>
        <p><strong>Email : </strong> <span id="email"></span></p>
        <p><strong>PAN ID : </strong> <span id="panId"></span></p>
        <p><strong>Address :</strong> <span id="address"></span></p>
      </div>
      <div class="container1"><div id="plotly-visualization"></div></div>
    </div>
    <div class="loan-history">
      <h2>Loan History</h2>
      <div class="loan-history-table">
        <table>
          <thead>
            <tr><th>S. No</th><th>Bank Name</th><th>Loan Amount</th><th>Loan Requested Date</th><th>Loan Issued Date</th><th>Loan Due Date</th><th>EMI Paid on</th><th>Loan Ends On</th><th>Loan Status</th></tr>
          </thead>
          <tbody id="loanHistoryTableBody"></tbody>
        </table>
      </div>
    </div>
    <div class="buttons">
      <button id="button1" onclick="closePopup()">Decline</button>
      <button id="button2" onclick="moveToNextState()">Approve</button>
    </div>
  </div>
</body>
</html>
```

### index.js

```javascript
ZOHO.embeddedApp.on("PageLoad", async function (data) {
    const loanRec = await ZOHO.CRM.API.getRecord({ Entity: data.Entity, RecordID: data.EntityId });
    const contactRec = await ZOHO.CRM.API.getRecord({ Entity: "Contacts", RecordID: loanRec.data[0].Recipient.id });
    const loanHistory = await ZOHO.CRM.API.getRelatedRecords({ Entity: "Contacts", RecordID: loanRec.data[0].Recipient.id, RelatedList: "Loan_History", page: 1, per_page: 200 });
    displayBasicDetails(contactRec.data[0]);
    processLoanHistory(loanHistory.data);
})
ZOHO.embeddedApp.init();

function moveToNextState() { ZOHO.CRM.BLUEPRINT.proceed(); }
function closePopup() { ZOHO.CRM.UI.Popup.close(); }

function displayBasicDetails(data) {
    document.getElementById('name').textContent = data.Full_Name;
    document.getElementById('email').textContent = data.Email;
    document.getElementById('panId').textContent = data.PAN_ID;
    document.getElementById('address').textContent = data.Mailing_Street + data.Mailing_Zip;
}

function processLoanHistory(loanHistory) {
    var count = 1, creditScore;
    var tbody = document.getElementById('loanHistoryTableBody');
    loanHistory.forEach(function (loan) {
        if (!loan.Bank_Name) return;
        var row = document.createElement('tr');
        [count++, loan.Bank_Name, loan.Amount, loan.Loan_Requested_Date, loan.Loan_Issued_Date, loan.Loan_Due_Date, loan.EMI_Paid_On, loan.Loan_Ends_On, loan.Loan_Status].forEach(val => {
            var td = document.createElement('td');
            td.textContent = val;
            row.appendChild(td);
        });
        if (loan.Loan_Status === 'Defaults') { row.classList.add('default-loan-row'); creditScore = Math.floor(Math.random() * 350); }
        tbody.appendChild(row);
    });
    creditScore = creditScore || Math.floor(Math.random() * 900);
    Plotly.newPlot("plotly-visualization", [{
        type: "indicator", mode: "gauge+number", value: creditScore,
        title: { text: "Credit Score", font: { size: 24 } },
        gauge: { axis: { range: [null, 850] }, bar: { color: "#1f77b4" }, steps: [{ range: [0,350], color: "red" }, { range: [350,550], color: "orange" }, { range: [550,700], color: "yellow" }, { range: [700,850], color: "green" }] }
    }], {});
}
```

**Gotchas:**
- The credit score uses `Math.random()` — this is a demo placeholder. In production, integrate with an actual credit bureau API.
- `index.js` is loaded via a `<script src>` in the `<head>` — this means the script executes before the DOM is ready. The `ZOHO.embeddedApp.on("PageLoad", ...)` handler is registered at parse time (before DOM), which is correct — PageLoad fires after the SDK is initialized, not at DOM parse time. However, `displayBasicDetails` and `processLoanHistory` reference DOM elements by ID, so they must be called only after DOM is ready (which they are, since they're called inside the async PageLoad handler).
- `Recipient` is the API name of the lookup field on the Loans module linking to Contacts — change this to your actual field API name.
- `Loan_History` is the API name of the related list on Contacts — verify this matches your org's configuration.
- `index.css` content is not included in the extracted zip.

---

## Sample 10 — Render Widget in a Pop-up using Client Script

**Widget type:** Custom Button / popup triggered by Client Script on picklist field change  
**Module:** Deals (or any module where the Client Script runs)  
**SDK version:** v1.2  
**Event:** PageLoad (receives `max_rows` and `product_category` from Client Script via `$Client.open`)

**Use case:** A Client Script detects a picklist change and opens this widget as a popup using `$Client.open`. The widget fetches Products filtered by category, displays them in a selectable table, and returns the selected products back to the Client Script using `$Client.close(selected_products)`. The Client Script then populates a subform with the returned data.

**Key SDK patterns:**
- Widget receives custom data from Client Script via `data` in PageLoad (`data.max_rows`, `data.product_category`)
- `ZOHO.CRM.API.searchRecord({ Entity, Type: "criteria", Query })` to filter products by category
- `$Client.close(returnValue)` to send data back to the calling Client Script
- Client Script side uses `$Client.open({ widget_api_name, ... }, inputData)` and `await`s the return value

**Files:** `widget.html`, `script/script.js`, `styles/style.css`

### widget.html

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <script src="https://live.zwidgets.com/js-sdk/1.2/ZohoEmbededAppSDK.min.js"></script>
  <title>Choose products</title>
  <link href="styles/style.css" rel="stylesheet" type="text/css" />
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">
</head>
<body>
  <div class="wgt_cp_popupContent">
    <section class="wgt_cp_contentWrap">
      <div hidden="true" id="noProductDiv"><pre>No products associated to the selected deal</pre></div>
      <table width="100%" cellspacing="0" id="pTable">
        <thead>
          <tr class="wgt_productTblHead">
            <th><input type="checkbox" id="selectAll" onclick="selectAllProducts(this)"></th>
            <th>Product Name</th><th>Product Category</th><th>Unit Price</th><th>Quantity in Stock</th>
          </tr>
        </thead>
        <tbody id="tbody"></tbody>
      </table>
    </section>
    <footer class="wgt_cp_footerWrap">
      <div class="wgt_selecProdWrapper">
        <span class="wgt_selectedProdCount" id="selectedCount">0 Products selected</span>
        <span class="wgt_clearSelProducts" onclick="clearSelection()">Clear</span>
      </div>
      <div><button type="button" class="wgt_nextBtn wgt_primaryBtn" onclick="closewidget()">Next</button></div>
    </footer>
  </div>
  <script src="./script/script.js"></script>
</body>
</html>
```

### script/script.js

```javascript
var count = 0;
var productData, maxRows = 0;

ZOHO.embeddedApp.on("PageLoad", async function (data) {
	maxRows = data.max_rows;
	const search_response = await ZOHO.CRM.API.searchRecord({ Entity: "Products", Type: "criteria", Query: "(Product_Category:equals:" + data.product_category + ")" });
	if (search_response && search_response.data && search_response.data.length) {
		productData = search_response.data;
		var htmlString = "";
		productData.forEach(({ id, Product_Category, Product_Name, Unit_Price, Qty_in_Stock }) => {
			htmlString += `<tr><td><input type='checkbox' onclick='selected(this)' id='${id}' class='products'></td><td>${Product_Name}</td><td>${Product_Category}</td><td>${Unit_Price}</td><td>${Qty_in_Stock}</td></tr>`;
		});
		document.getElementById("tbody").innerHTML = htmlString;
	} else {
		document.getElementById("pTable").hidden = true;
		document.getElementById("noProductDiv").hidden = false;
	}
});
ZOHO.embeddedApp.init();

function selected(element) { element.checked ? count++ : count--; document.getElementById("selectedCount").innerHTML = `${count} Products selected`; }
function selectAll(select_all) {
	for (let el of document.getElementsByClassName('products')) { el.checked = select_all; }
	count = select_all ? productData.length : 0;
	document.getElementById("selectedCount").innerHTML = `${count} Products selected`;
	document.getElementById("selectAll").checked = select_all;
}
function selectAllProducts(element) { selectAll(element.checked); }
function clearSelection() { selectAll(false); }
function closewidget() {
	if (count > maxRows) { alert("Selected product is greater than the maximum subform rows."); }
	else {
		var selected_products = [];
		for (let el of document.getElementsByClassName('products')) { el.checked && selected_products.push(productData.find(p => p.id === el.id)); }
		$Client.close(selected_products);
	}
}
```

**Gotchas:**
- `$Client.close(value)` is the mechanism for returning data to the Client Script. This is a global injected by the SDK when the widget is opened via `$Client.open`. It is NOT the same as `ZOHO.CRM.UI.Popup.close()`.
- `data.max_rows` and `data.product_category` come from the Client Script's `$Client.open` call — the widget has no way to know these values unless the Client Script passes them.
- `styles/style.css` content is not included in the extracted zip.
- The `selected` function name shadows the global `selected` concept — in strict mode this could conflict. Rename if needed.
- `ZOHO.CRM.API.searchRecord` with `Type: "criteria"` uses the CRM search API with criteria syntax `(field:operator:value)`. For multiple criteria, join with `and` or `or`: `(Field1:equals:val1)and(Field2:equals:val2)`.

---

## Sample 11 — Escalation Workflow (ZDK.Client Methods Showcase)

**Widget type:** Related List widget + Custom Button popup widget (two widgets in one extension package)  
**Module:** Cases / Contacts (the related list shows complaints; the popup captures escalation detail)  
**SDK version:** Non-standard CDN — `https://app-assets.web.app/widgetsdk/ZohoEmbededAppSDK.min.js`  
**Event:** PageLoad (no entity context used in the main widget — data is hardcoded for demo)

**Use case:** A Related List widget displays a table of support complaints with an "Escalate" button per row. Clicking Escalate calls `ZDK.Client.showConfirmation` for a confirm dialog, then `ZDK.Client.openPopup` to open a second widget (EscalationPopup) that collects escalation details. The popup returns data via `$Client.close`, and the parent widget shows a success message via `ZDK.Client.showMessage`.

**Key SDK patterns:**
- `ZDK.Client.showConfirmation(message, confirmLabel, cancelLabel)` — returns `true`/`false`
- `ZDK.Client.openPopup({ api_name, type, header, animation_type, width, height }, inputData)` — opens another widget as a popup and `await`s its return value
- `ZDK.Client.showMessage(message, { type })` — shows a toast notification (`'success'`, `'error'`, `'info'`)
- `$Client.close(data)` in the popup widget to return data to the caller
- Two widgets in one zip: the `plugin-manifest.json` must define both widget API names

**Files:** `tickets.html` (Related List widget), `escalationdetail.html` (popup widget — API name: `EscalationPopup`), `widget1.html` (all 28 API methods demo)

### tickets.html (Related List widget — main widget)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <script src="https://app-assets.web.app/widgetsdk/ZohoEmbededAppSDK.min.js"></script>
    <style>
        /* ... table styles, status badges, escalate button ... */
    </style>
</head>
<body>
    <div class="container">
        <div class="table-wrapper">
            <table>
                <thead><tr><th>ID</th><th>Subject</th><th>Status</th><th>Priority</th><th>Created</th><th>Action</th></tr></thead>
                <tbody>
                    <tr><td>TCK-0001</td><td>Unable to log in to portal</td><td>Open</td><td>High</td><td>20/09/2025</td><td><button onclick="openEscalationPopup('TCK-0001')">Escalate</button></td></tr>
                    <!-- more rows -->
                </tbody>
            </table>
        </div>
    </div>
    <script>
        ZOHO.embeddedApp.init();
        async function openEscalationPopup(ticketId) {
            let confirmed = await ZDK.Client.showConfirmation(
                'Escalating this case will mark it High Priority, notify the Customer Success Manager, and start SLA tracking. Do you want to continue?',
                'Yes. Proceed', 'No'
            );
            if (confirmed) {
                try {
                    const escalationResponse = await ZDK.Client.openPopup({
                        api_name: 'EscalationPopup',
                        type: 'widget',
                        header: 'Escalation Information',
                        animation_type: 1,
                        width: '500px',
                        height: '650px'
                    }, { data: '' });
                    if (escalationResponse.data.escalationLevel) {
                        ZDK.Client.showMessage('Case escalated successfully', { type: 'success' });
                    }
                } catch (error) {
                    console.error('Error in popup action', error);
                }
            }
        }
    </script>
</body>
</html>
```

### escalationdetail.html (popup widget — API name: EscalationPopup)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <script src="https://app-assets.web.app/widgetsdk/ZohoEmbededAppSDK.min.js"></script>
    <!-- form styles -->
</head>
<body>
    <div class="popup">
        <div class="popup-header"><h2>Escalation Details</h2></div>
        <div class="popup-body">
            <form id="escalationForm">
                <div class="form-group">
                    <label>Escalation Level:</label>
                    <select id="escalationLevel"><option value="">Select Level</option><option value="E1">E1</option><option value="E2">E2</option><option value="E3">E3</option></select>
                </div>
                <div class="form-group"><label>Reason:</label><input type="text" id="reason" placeholder="Enter escalation reason"></div>
                <div class="form-group"><label>Priority Update:</label><input type="text" id="priority" value="High" readonly></div>
                <div class="form-group"><label>Notes:</label><textarea id="notes" placeholder="Enter additional notes"></textarea></div>
            </form>
        </div>
        <div class="popup-footer">
            <button onclick="hidePopup()">Cancel</button>
            <button onclick="saveEscalation()">Save</button>
        </div>
    </div>
    <script>
        ZOHO.embeddedApp.init();
        function hidePopup(data) { $Client.close({ data }); }
        function saveEscalation() {
            const form = document.getElementById('escalationForm');
            const formData = { escalationLevel: form.escalationLevel.value, reason: form.reason.value, priority: form.priority.value, notes: form.notes.value };
            hidePopup(formData);
        }
    </script>
</body>
</html>
```

### widget1.html (all 28 API methods demo — key excerpt)

This file demonstrates every `ZOHO.CRM.API` method with a button per method. Key patterns shown:

```javascript
ZOHO.embeddedApp.on("PageLoad", function(data) { console.log("Widget Loaded"); });

// getRecord
ZOHO.CRM.API.getRecord({ Entity: "Leads", RecordID: "440280000000234001" }).then(console.log);

// coql
ZOHO.CRM.API.coql("select Last_Name, Account_Name, Email from Contacts where Last_Name != null").then(console.log);

// insertRecord
ZOHO.CRM.API.insertRecord({ Entity: "Leads", APIData: { Last_Name: "Smith", Email: "smith@example.com" } }).then(console.log);

// searchRecord
ZOHO.CRM.API.searchRecord({ Entity: "Leads", Type: "word", Query: "Smith" }).then(console.log);

// BLUEPRINT
ZOHO.CRM.API.getBluePrint("Deals", "440280000000234001").then(console.log);
ZOHO.CRM.API.updateBluePrint("Deals", "440280000000234001", { blueprint_transition_id: "TRANSITION_ID", data: { Phone: "1234567890" } }).then(console.log);
```

> **Note:** `widget1.html` also uses `ZOHO.CRM.CURRENT_RECORD.getRecord()` — an undocumented namespace not in the official JSDoc dataset. See `sdk.md` undocumented section for details.

**Gotchas:**
- **Do not use the `app-assets.web.app` CDN in production.** Replace with `https://live.zwidgets.com/js-sdk/1.5/ZohoEmbededAppSDK.min.js`.
- `ZDK.Client` methods (`showConfirmation`, `openPopup`, `showMessage`) are available in the v1.5 SDK as `ZDK.Client.*`. In older SDKs these are not available — use `ZOHO.CRM.UI.Popup` and `ZOHO.CRM.UI.Popup.close()` instead.
- The `openPopup` call uses `api_name: 'EscalationPopup'` — this must match the `api_name` field in the `plugin-manifest.json` for the second widget exactly.
- The ticket data in `tickets.html` is hardcoded (demo only) — in production, fetch from Desk or CRM using the appropriate API.
- `animation_type: 1` controls the popup entrance animation. Valid values are 1 (slide) and 2 (fade) — verify with current SDK docs.

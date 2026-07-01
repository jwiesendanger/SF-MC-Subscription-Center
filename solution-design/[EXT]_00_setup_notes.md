# UK Preference Center — Setup Notes

## Files in this folder

| File | Where it lives in MCE | Purpose |
|---|---|---|
| `parent_preference_center.html` | Parent BU > CloudPages | Preference UI, form renderer, unsub confirmation. No placeholders — publish directly. |
| `child_academic.html` | Academic child BU > CloudPages | Read/write CRM prefs for Ed Cloud org. Includes SLCM_Person_ID__c fallback. |
| `child_radar.html` | Alumni-Philanthropy child BU > CloudPages | Read/write CRM prefs for RADAR org. No SLCM fallback needed. |
| `child_generic.html` | Any other child BU > CloudPages | Template for Social Work, HR, Healthcare, Athletics, Extension. |
| `footer_content_block.html` | Parent BU > Content Builder > Content Blocks | Dynamic branded footer for all UK emails. |

---

## Critical AMPscript & SSJS Patterns (Verified in Production)

### 1. Use meta refresh — NOT Redirect() or HTTPRedirect()
Both `Redirect()` and `HTTPRedirect()` cause "The page content contains errors and cannot be processed"
in CloudPages context. The only reliable redirect mechanism is HTML meta refresh:

```html
<meta http-equiv="refresh" content="0;url=%%=v(@redirectURL)=%%"/>
```

### 2. ENT. prefix required for Synchronized DEs
`LookupRows` fails silently (rowcount=0) or throws a parse error without the `ENT.` prefix,
even when the DE External Key appears to match:

```
SET @row = LookupRows("ENT.Contact_Salesforce_13", "Id", @subKey)
```

Test with `ENT.` first. If rowcount returns 0, try without. If it errors, `ENT.` is required.

### 3. AttributeValue() for CloudPagesURL parameters — NOT QueryParameter()
Parameters encrypted by CloudPagesURL() must be read with `AttributeValue()`:

```
SET @subKey     = AttributeValue("_subscriberkey")   /* Salesforce Contact ID */
SET @sendingMID = AttributeValue("memberid")          /* sending BU MID */
SET @jobID      = AttributeValue("jobid")
SET @listID     = AttributeValue("listid")
SET @batchID    = AttributeValue("_batchid")
```

`QueryParameter("sub")`, `QueryParameter("_subscriberkey")`, `QueryParameter("mid")` all return empty.

### 4. Booleans from Sync DE arrive as "True"/"False" (capital T/F)
Not "true"/"false" (lowercase). Always compare with both cases:

```
IIF(Field(Row(@row,1), "CommPref_Newsletters__c") == "True", "true",
  IIF(Field(Row(@row,1), "CommPref_Newsletters__c") == "true", "true", "false"))
```

### 5. writeback URL must be hardcoded — NOT RequestURL()
`RequestURL()` inside `Concat()` causes a parse error. Hardcode the page's own public URL:

```
SET @myURL = "https://pub.pages.exacttarget.com/page/XXXXXXXX"
/* Then use: "&writeback=", UrlEncode(@myURL) */
```

### 6. All VAR declarations must be at the top
AMPscript does not allow VAR declarations inside IF blocks. All variables must be
declared at the beginning of the AMPscript block before any IF statements.

### 7. No nested function calls
`UrlEncode(v(@subKey))` and `UrlEncode(RequestURL())` both cause parse errors.
Assign the inner value to a variable first, then pass the variable to UrlEncode.

### 8. SSJS subscriber status: use api.updateItem() — NOT api.invoke("LogUnsubEvent")
`api.invoke("LogUnsubEvent")` is an AMPscript function, not a SOAP object — it fails silently
in SSJS WSProxy (page loads, but status never changes). The correct method is `api.updateItem()`.

The SSJS block runs on every form submit. It sets status to `Unsubscribed` or `Active`
depending on whether `globalunsub=1`, covering both unsub and re-subscribe via the preference center:

```javascript
Platform.Load("core", "1.1.5");
var formAction  = Platform.Request.GetFormField("formAction");
var globalUnsub = Platform.Request.GetFormField("globalunsub");
var mid         = Platform.Request.GetFormField("mid");
var subKey      = Platform.Request.GetFormField("sub");

if (formAction === "saveprefs" && mid && subKey) {
  var api       = new Script.Util.WSProxy();
  var newStatus = (globalUnsub === "1") ? "Unsubscribed" : "Active";
  api.setClientId({ ID: parseInt(mid, 10) });
  api.updateItem("Subscriber", {
    SubscriberKey: subKey,
    Status:        newStatus
  });
}
```

---

## Re-subscribe: additional step required for CRM-initiated opt-ins

The preference center handles unsub and re-subscribe automatically when the subscriber
uses the email link:
- Subscriber unsubscribes → MCE status changes to **Unsubscribed**
- Subscriber opens the email link, unchecks "Unsubscribe from all" and saves → MCE status changes back to **Active**

**Gap:** If someone changes `HasOptedOutOfEmail = false` directly in CRM (without going through
the preference center), the Sync DE updates on the next hourly sync — but MCE does NOT
automatically reactivate the subscriber. The subscriber stays suppressed in MCE even though
CRM shows opt-in.

To handle this, one of the following processes is needed:

### Option 1 — Automation Studio batch job (recommended for production)
Create an Automation Studio automation running daily:
1. Query Activity: query the Sync DE for contacts where `HasOptedOutOfEmail = false`
2. Script Activity (SSJS): for each contact, call `api.updateItem("Subscriber", { Status: "Active" })`
   with `api.setClientId({ ID: parseInt(mid, 10) })` for the correct BU MID

### Option 2 — Salesforce Flow
Create a Flow that fires when `HasOptedOutOfEmail` changes from `true` to `false`:
1. Invocable Apex callout to the MCE REST or SOAP API
2. Updates the Subscriber status to Active in the correct BU

### Option 3 — Manual
In MCE > Email Studio > All Subscribers, search for the subscriber and change status to Active
directly in the UI. Sufficient for demo or one-off support cases.

---

## Required Data Extensions

### 1. UK_PrefCenter_BrandConfig (Parent BU)

| Field | Type | Notes |
|---|---|---|
| MID | Number (PK) | Business Unit MID |
| OrgCode | Text 50 | Identifier: academic, radar, socialwork, hr, healthcare, athletics, extension |
| BUName | Text 200 | Display name: "UK College of Medicine", "UK Alumni Association", etc. |
| WriteBackURL | Text 500 | Full public URL of the child relay page (no trailing slash) |
| LogoURL | Text 500 | Hosted logo image URL (white version for dark backgrounds) |
| LogoAlt | Text 200 | Logo alt text |
| BrandColor | Text 10 | Hex color with #, e.g. #003DAA |
| FooterAddress | Text 300 | Mailing address string |
| FooterPhone | Text 30 | Phone number string |

Populate one row per sending BU.

### 2. Synchronized Contact DE (each child BU)

Must include these Contact fields:

| Salesforce Field | Type | Notes |
|---|---|---|
| Id | Text 18 | Salesforce Contact ID — primary key for LookupRows |
| Email | Text 254 | Email address |
| HasOptedOutOfEmail | Boolean | Standard opt-out field — written on global unsub |
| DoNotContact | Boolean | Standard do-not-contact field — also written on global unsub |
| SLCM_Person_ID__c | Text 50 | SAP/SLCM Person ID — Academic org only, confirm API name with Jordan |
| CommPref_Newsletters__c | Boolean | Preference field |
| CommPref_Events__c | Boolean | Preference field |
| CommPref_Alerts__c | Boolean | Preference field |
| CommPref_Academic__c | Boolean | Preference field |
| CommPref_Recruitment__c | Boolean | Preference field |
| CommPref_AlumniDonor__c | Boolean | Preference field |
| CommPref_HealthWellness__c | Boolean | Preference field |

---

## CRM Fields to Create (identical API names in all 7 orgs)

| Label | API Name | Type | Default |
|---|---|---|---|
| Pref: Newsletters & News | CommPref_Newsletters__c | Checkbox | Checked (true) |
| Pref: Events & Programs | CommPref_Events__c | Checkbox | Checked (true) |
| Pref: Alerts & System Notices | CommPref_Alerts__c | Checkbox | Checked (true) |
| Pref: Academic & Dept Communications | CommPref_Academic__c | Checkbox | Checked (true) |
| Pref: Recruitment & Admissions | CommPref_Recruitment__c | Checkbox | Checked (true) |
| Pref: Donor & Alumni Engagement | CommPref_AlumniDonor__c | Checkbox | Checked (true) |
| Pref: Health & Wellness | CommPref_HealthWellness__c | Checkbox | Checked (true) |

Default = Checked (true) so existing contacts are opted in until they explicitly opt out.

---

## Placeholders to Replace

| Placeholder | File | What to put |
|---|---|---|
| `REPLACE_WITH_PARENT_PREF_CENTER_URL` | all child pages | Published URL of parent_preference_center CloudPage |
| `REPLACE_WITH_THIS_PAGE_URL` | all child pages | Public URL of THIS child page (get after first publish, then re-publish) |
| `REPLACE_WITH_SYNC_DE_NAME` | all child pages | Synchronized Contact DE name with ENT. prefix, e.g. ENT.Contact_Salesforce_13 |
| `REPLACE_WITH_ORG_DISPLAY_NAME` | child_generic.html only | Display name shown in the org badge, e.g. "Social Work (Ed Cloud)" |

---

## Implementation Order

1. Create CommPref_* fields on Contact in each CRM org (7 orgs). Default = Checked.
2. Add fields to Sync DE config in each child BU. Run manual sync. Verify columns appear.
3. Create UK_PrefCenter_BrandConfig DE in Parent BU.
4. **Publish parent_preference_center.html** in Parent BU — note the URL.
5. For each child BU:
   a. Replace `REPLACE_WITH_PARENT_PREF_CENTER_URL` with URL from step 4
   b. Replace `REPLACE_WITH_SYNC_DE_NAME` with actual DE name (with ENT. prefix)
   c. Publish — note the URL
   d. Replace `REPLACE_WITH_THIS_PAGE_URL` with this page's own URL
   e. Re-publish
   f. Add a row to UK_PrefCenter_BrandConfig with this BU's MID and WriteBackURL
6. Add footer_content_block.html as a Content Block in Parent BU.
7. Insert the Content Block into all UK email templates.
8. **RADAR only — replace Experience Cloud footer link:**
   - In the Alumni-Philanthropy child BU, locate all active email templates and journeys
     that currently link to the RADAR Experience Cloud subscription center page.
   - Replace that link with a `CloudPagesURL()` call pointing to `child_radar.html`,
     passing `_subscriberkey`, `memberid`, `jobid`, `listid`, and `_batchid`.
   - The RADAR `Subscription__c` ledger object is not affected — it stays intact for Radar 2.0.

---

## Notes

### Non-CRM contacts — default behavior

Some audiences (Excel-uploaded lists, external industry contacts) have no Salesforce Contact record
and will never match a Sync DE lookup. When `LookupRows` returns 0 rows, all child pages default
to `HasOptedOutOfEmail = false` and all `CommPref_*` fields = `true` (opted in).

This is intentional: these contacts have never had the opportunity to set preferences, so the
safest default is to treat them as opted in to everything. The preference center will not error —
it simply shows the default state.

These contacts cannot write back to CRM on save because there is no Contact record to update.
The `UpdateSingleSalesforceObject` call will return 0 (no record found) and no write occurs.

**Long-term recommendation:** bring these audiences into CRM as Contact records so they can
be looked up and their preferences honored. The Ed Cloud rebuild is already planning to
account for external contacts. See the recommendations deck for the full pitch.

---

### SLCM ID — Confirm with Jordan
The Academic org uses `SLCM_Person_ID__c` as subscriber key for some contacts.
Confirm the exact API name with Jordan before go-live.
The Academic child page includes a fallback `LookupRows` by SLCM_Person_ID__c
if the Contact ID lookup returns 0 rows.

### RADAR Legacy
- RADAR Subscription__c object stays intact for Radar 2.0 (Experience Cloud, Aug 2027)
- This preference center writes CommPref_* on Contact only — the two systems coexist
- ENT.Contact_Salesforce_13 referenced in the old RADAR footer is replaced by this pattern

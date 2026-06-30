# Subscription Config Data Extension Schema
**Phase 2 | UK Centralized Subscription Center — Parent Business Unit**

Two data extensions live in the **parent BU** only. All subscriber-facing and admin pages read exclusively from these DEs at runtime — no org IDs, field API names, or labels are hardcoded anywhere else.

---

## DE 1: `SubscriptionConfig_Orgs`

Stores one record per registered CRM organization.

**Primary Key:** `OrgID`  
**Is Sendable:** No  
**Is Testable:** No

| Field Name | Data Type | Max Length | Required | Notes |
|---|---|---|---|---|
| `OrgID` | Text | 18 | ✅ | Salesforce 18-character Org ID (e.g., `00D8Z000001XxXx`). Primary key. |
| `DisplayName` | Text | 100 | ✅ | Human-readable name shown in the admin tool (e.g., `Human Resources`). |
| `InstanceURL` | Text | 255 | ✅ | Salesforce instance base URL — no trailing slash (e.g., `https://uk.my.salesforce.com`). |
| `MCBUMID` | Text | 20 | ✅ | Marketing Cloud MID of the child BU linked to this CRM org. Used for `updateItem` unsubscribe calls. |
| `SFClientID` | Text | 255 | ✅ | Connected App client ID for OAuth client credentials flow. |
| `SFClientSecret` | Text | 255 | ✅ | Connected App client secret. Stored plaintext — restrict DE access to admin tool cloud page only. |
| `IsActive` | Boolean | — | ✅ | `True` = org is live in the subscription center. `False` = deactivated (soft delete). |
| `CreatedDate` | Date | — | ✅ | Set at insert; never updated. |
| `ModifiedDate` | Date | — | ✅ | Updated on every write. |

**Notes:**
- Removing an org via the admin tool sets `IsActive = False` (soft delete). Hard deletes are not performed to preserve audit history.
- The parent cloud page filters `IsActive = True` at runtime.
- `SFClientID` / `SFClientSecret` are used server-side only (SSJS HTTP calls). They are never exposed to the browser.

---

## DE 2: `SubscriptionConfig_Fields`

Stores one record per registered subscription preference field, scoped to an org.

**Primary Key:** `FieldID`  
**Is Sendable:** No  
**Is Testable:** No

| Field Name | Data Type | Max Length | Required | Notes |
|---|---|---|---|---|
| `FieldID` | Text | 200 | ✅ | Composite primary key: `{OrgID}_{SFObjectName}_{FieldAPIName}`. Ensures uniqueness across orgs. |
| `OrgID` | Text | 18 | ✅ | Foreign key to `SubscriptionConfig_Orgs.OrgID`. |
| `SFObjectName` | Text | 100 | ✅ | Salesforce API object name (e.g., `Contact`, `HR_Record__c`). |
| `FieldAPIName` | Text | 100 | ✅ | Salesforce field API name (e.g., `Email_Opt_In_Benefits__c`). Must match the MC synchronized DE column name exactly. |
| `DisplayLabel` | Text | 100 | ✅ | Subscriber-facing label on the preference form (e.g., `Benefits & Wellness Updates`). Set by admin, not read from Salesforce. |
| `FieldType` | Text | 50 | ✅ | Salesforce field type as returned by the describe API (e.g., `boolean`, `checkbox`). Used to render the correct input control on the preference form. |
| `SortOrder` | Number | — | ✅ | Display order on the preference form. Lower = higher. Admin-configurable. Default 99. |
| `IsActive` | Boolean | — | ✅ | `True` = field is shown on the preference form. `False` = hidden (soft delete). |
| `CreatedDate` | Date | — | ✅ | Set at insert. |
| `ModifiedDate` | Date | — | ✅ | Updated on every write. |

**Notes:**
- `FieldID` is constructed by the admin tool as `{OrgID}_{SFObjectName}_{FieldAPIName}`. This guarantees uniqueness without requiring a separate sequence.
- Deactivating a field sets `IsActive = False`. The underlying CRM field and the synchronized DE column are **not** touched.
- The parent cloud page queries this DE filtered by `OrgID` and `IsActive = True`, ordered by `SortOrder`, to know which fields to render and what label to show.
- `SFObjectName` and `FieldAPIName` together identify the exact column in the synchronized contact DE for that BU. The synchronized DE column name is expected to be the `FieldAPIName` directly.

---

## Provisioning Steps

### Step 1 — Create `SubscriptionConfig_Orgs`

1. In the **parent BU**, navigate to **Contact Builder → Data Extensions → Create**.
2. Choose **Standard Data Extension**.
3. Name: `SubscriptionConfig_Orgs`
4. Set **Is Sendable** = No.
5. Add fields exactly as specified in the table above:
   - Set `OrgID` as the **Primary Key**.
   - Set `IsActive` type to **Boolean**.
   - Set `CreatedDate` and `ModifiedDate` type to **Date**.
   - All other fields are **Text** or **Number** as specified.
6. Save.

### Step 2 — Create `SubscriptionConfig_Fields`

1. Same BU and method as above.
2. Name: `SubscriptionConfig_Fields`
3. Add fields per the second table.
   - Set `FieldID` as the **Primary Key**.
   - `SortOrder` = **Number** (no decimals).
   - `IsActive` = **Boolean**.
   - `CreatedDate`, `ModifiedDate` = **Date**.
4. Save.

### Step 3 — Restrict DE Access

Both DEs should have their **sharing settings** restricted so only the admin tool cloud page and the parent subscription center cloud page can interact with them. No child BU should have write access to these DEs directly.

---

## Runtime Query Examples (SSJS — Parent Cloud Page)

```javascript
// Load active fields for a given org, ordered by SortOrder
var prox = new Script.Util.WSProxy();
var cols = ["FieldID", "OrgID", "SFObjectName", "FieldAPIName", "DisplayLabel", "FieldType", "SortOrder"];
var filter = {
  "LeftOperand":  { Property: "OrgID",    SimpleOperator: "equals", Value: orgId },
  "LogicalOperator": "AND",
  "RightOperand": { Property: "IsActive", SimpleOperator: "equals", Value: 1 }
};
var result = prox.retrieve("DataExtensionObject[SubscriptionConfig_Fields]", cols, filter);
var fields = result.Results || [];
```

```javascript
// Load org metadata by OrgID
var orgRows = prox.retrieve(
  "DataExtensionObject[SubscriptionConfig_Orgs]",
  ["OrgID", "DisplayName", "InstanceURL", "MCBUMID"],
  { Property: "OrgID", SimpleOperator: "equals", Value: orgId }
);
var org = (orgRows.Results && orgRows.Results.length > 0) ? orgRows.Results[0] : null;
```

# PRD: UK Centralized Subscription Center
**Salesforce Marketing Cloud — University of Kentucky**
**Version:** 1.2 | **Date:** 2026-06-29 | **Status:** Draft — POC Scope

---

## 1. Background & Problem Statement

The University of Kentucky currently operates email subscription management across three isolated silos:

1. **Legacy cloud pages (circa 2017)** — unmaintainable and no longer aligned with current platform capabilities.
2. **Radar subscriber portal** — high-friction experience that requires an external portal login, causing significant user abandonment.
3. **Unpersonalized profile centers** (Social Work, UK Healthcare, Human Resources) — not synchronized with the CRM; preference data lives in Marketing Cloud only.

The result is fragmented subscriber data across 14 business units, no unified subscriber experience, and ongoing risk of data drift between Marketing Cloud and Salesforce CRM.

> **POC Scope:** This version of the PRD covers a proof-of-concept implementation scoped to a single CRM organization — **Human Resources**. The architecture is designed to scale to all 14 business units; full rollout phases will be planned following POC validation.

---

## 2. Objective

Design and implement a centralized, passwordless, frictionless subscription preference center that:

- Serves all 14 UK business units from a single parent cloud page
- Uses CRM as the system of record for all preference data
- Writes subscriber preferences back to the originating CRM organization in real time
- Eliminates the need for external portal logins via encrypted token-based identity
- Introduces zero new HIPAA or FERPA regulatory exposure
- Provides an internal admin tool for managing CRM org registrations and subscription field configuration without requiring developer intervention

---

## 3. Proposed Architecture

### 3.1 Overview

The solution is a **3-layer decoupled architecture** hosted within the existing Marketing Cloud and Salesforce CRM infrastructure:

| Layer | Location | Responsibility |
|---|---|---|
| **UI Layer** | Parent Business Unit (Marketing Cloud) | Renders preference forms; handles all subscriber-facing interactions |
| **Relay Layer** | Child Business Unit cloud pages (one per BU) | Acts as translation bridge; routes data writes to the correct CRM org |
| **Data Layer** | Synchronized contact data extensions | Mirrors CRM contact fields into Marketing Cloud for form pre-population |

The **Parent Business Unit** is a centralized utility host only — it does not execute marketing sends and has no direct connection to any CRM organization. This design minimizes tenant overhead and isolates rendering from data writes.

### 3.2 Preference Management Flow

1. Subscriber clicks "manage preferences" in an email footer.
2. Footer uses `CloudPagesURL()` to encrypt subscriber context (subscriber key, BU identifier, CRM org ID) into a secure token.
3. Subscriber is redirected to the **child relay page** for that BU.
4. Child relay page decrypts the token and redirects to the **parent cloud page**, passing context.
5. Parent page queries the synchronized contact data extension and renders a pre-populated preference form.
6. On form submission, data is posted back through the child relay page, which writes preferences to the correct CRM organization.

### 3.3 Unsubscribe Flow

1. Subscriber clicks "unsubscribe" in an email footer — same token-encrypted redirect as above.
2. Child relay page refreshes to a confirmation screen rendered by the parent page.
3. On confirmation, a server-side JavaScript block in the parent BU calls the API `updateItem` method with the originating child BU's client ID, setting subscriber status to `unsubscribed` in that BU's All Subscribers list.
4. Simultaneously, the seven category checkbox fields are cleared on the synchronized data extension to maintain data alignment.

**Important:** Unsubscribe is scoped to the originating CRM organization only. It does not suppress the subscriber across all UK organizations.

### 3.4 Data Synchronization Timing

CRM-to-Marketing Cloud synchronization for preference data has a configured refresh interval of approximately **15 minutes**. The preference form will reflect CRM data as of the last sync; real-time CRM edits will not be visible until the next sync cycle completes.

---

## 4. Key Decisions

### 4.1 CRM Write-Back Architecture
Each business unit's child relay page writes subscriber preferences back to its own specific CRM organization. The parent business unit never writes directly to any CRM org.

### 4.2 Subscription Field Scope
Subscription category fields are not restricted to the Salesforce Contact object. Fields may reside on any Salesforce object, provided the field is accessible via the Marketing Cloud synchronized data extension for the originating business unit. The admin tool (see Section 4.7) is the mechanism by which fields are registered — including their object name, API name, and display label — rather than being hardcoded into any cloud page.

For the POC, fields are scoped to the **Human Resources CRM org** only. API name consistency across multiple orgs is not a POC requirement but must be enforced before full rollout.

- **Default value must be set to `True`** for all existing records. This ensures current subscribers begin as opted-in and prevents accidental mass opt-outs at launch.
- Default-to-`True` logic applies only to new records entering the system. Existing records retain their current values.

### 4.3 Global Unsubscribe Scope Limitation
"Unsubscribe from all emails" language on the preference center refers only to the CRM organization associated with the email that was received — not the entire University of Kentucky. Email copy and preference center UI language must accurately reflect this scoped behavior.

### 4.4 Unsubscribe Terminology
All unsubscribe language must be specific to the originating CRM or email set. Language implying a university-wide global opt-out is prohibited in this implementation scope. Governance will enforce this as part of the annual review process.

### 4.5 Default Subscription Status for New Records
New contact records entering the system default to `True` (opted-in) across all seven subscription category fields.

### 4.6 Preference Center Scope
The preference center is **not a global aggregator**. It displays subscription preferences relevant only to the specific CRM organization connected to the business unit from which the email originated. Subscribers will only see and manage preferences for that context.

### 4.7 Admin Tool as Configuration Authority
A secured cloud page in the parent business unit serves as the single interface for managing subscription center configuration. It is the only supported mechanism for:
- Registering or removing CRM organizations
- Adding or removing subscription fields (including selection of the Salesforce object and field API name)
- Updating field display labels

All configuration is persisted in a **Subscription Config Data Extension** in the parent BU. The subscriber-facing parent cloud page reads exclusively from this DE at runtime — no org or field configuration is hardcoded in any cloud page. Changes made via the admin tool take effect after the next CRM-to-MC sync cycle.

**Authentication:** The admin tool uses Marketing Cloud SSO. SSJS checks for an active MC business unit session before rendering the page. Users without a valid MC session are denied access. No external identity provider or additional infrastructure is required.

---

## 5. Implementation Phases

### Phase 1 — CRM Field Preparation
- **POC org: Human Resources.** Ensure the required subscription fields exist on the appropriate Salesforce object(s) in the HR CRM org.
- Default value for boolean fields set to `True` on existing records to prevent accidental opt-outs at launch.
- Validation: Confirm field API names and object references are correct before proceeding.

**Owner:** Human Resources CRM administrator

### Phase 2 — Subscription Config DE & Admin Tool
This phase establishes the configuration layer that all subsequent phases depend on.

**2a — Subscription Config Data Extension**
- Design and create the Subscription Config DE schema in the parent business unit.
- Schema must support: CRM org metadata (org ID, instance URL, display name, MID of linked MC BU), field definitions per org (Salesforce object name, field API name, display label, field type), and active/inactive status per org and per field.

**2b — Admin Tool Cloud Page**
- Build a secured cloud page in the parent business unit.
- The tool must allow an authorized user to:
  - Add a new CRM org (enter org ID, instance URL, display name, linked MC BU MID)
  - Remove an existing CRM org (and its associated field registrations)
  - Connect to a registered CRM org via Salesforce API and browse available objects and their fields
  - Select one or more fields from any object to include as subscription categories
  - Set or update the display label for each selected field
  - Deactivate or remove a field from the subscription center without deleting the underlying CRM field
- All changes persist to the Subscription Config DE.
- Access restricted via MC SSO check — SSJS validates an active Marketing Cloud session before rendering.
- **POC:** Register the Human Resources org only. Multi-org support is validated structurally but not exercised with live data until full rollout.

**Owner:** Salesforce implementation team

### Phase 3 — Map Selected Fields to Synchronized Data Extensions
- Using the field scope defined via the admin tool, map the selected CRM fields to the Marketing Cloud synchronized contact data extension for the HR BU.
- Confirm sync is active and data flows correctly from CRM to Marketing Cloud.
- Validate 15-minute sync cadence is operating as expected.
- **POC:** Scoped to the Human Resources BU only.

**Owner:** Marketing Cloud administrator / Salesforce implementation team  
**Dependency:** Phase 2 (admin tool must define field scope before mapping)

### Phase 4 — Publish Parent Cloud Page
- Build and publish the subscriber-facing parent cloud page in the parent business unit.
- Page must: decrypt incoming token, look up the originating org's field configuration from the Subscription Config DE, query the correct synchronized data extension, render a pre-populated preference form using the configured fields and labels, handle form submission by posting back to the originating child relay page, and render the unsubscribe confirmation screen.
- SSJS unsubscribe block must call `updateItem` using the correct child BU client ID.
- No org identifiers, field API names, or labels may be hardcoded — all must be read from the Subscription Config DE at runtime.

**Owner:** Salesforce implementation team  
**Dependency:** Phase 2 (config DE must be populated), Phase 3 (synced DEs must be mapped)

### Phase 5 — Link Child Relay Pages
- **POC:** Build one child relay page for the Human Resources BU only.
- Each relay page: receives the encrypted token from the email footer, redirects to the parent cloud page with context, and accepts form POST submissions to write preferences to its respective CRM org.

**Owner:** Salesforce implementation team  
**Dependency:** Phase 4

### Phase 6 — Generate Dynamic HTML Footer Content
- **POC:** Scoped to Human Resources email templates only.
- Footer uses `CloudPagesURL()` to generate the encrypted token linking to the HR child relay page.
- Validate footer links route correctly through relay → parent for the HR BU.
- Migrate existing HR unsubscribe data: export current suppression lists and reload into the new configuration.

**Owner:** Marketing Cloud administrator / HR email template owner  
**Dependency:** Phase 5

---

## 6. Governance Model

- **Annual review** led by Emily Brenzel's team to audit subscription categories against current campus communication needs.
- Adding, removing, or relabeling subscription fields is performed via the admin tool by an authorized administrator — not by modifying cloud page code directly.
- Changes to field API names in Salesforce still require coordination across all participating CRM organizations simultaneously; the admin tool does not create or rename fields in Salesforce.
- Changes require formal sign-off from business unit stakeholders before execution.
- Unsubscribe language must be reviewed annually to confirm it accurately reflects scoped (not global) behavior.

---

## 7. Supporting Automations

### 7.1 CRM Manual Override Correction (Daily Batch)
A daily Automation Studio batch process queries for contacts with an `Unsubscribed` Marketing Cloud status whose CRM record shows an active (opted-in) status — indicating a CRM administrator manually re-activated the record without a corresponding Marketing Cloud update. The process uses SSJS via WS Proxy to flip the Marketing Cloud subscriber status back to `Active`, keeping both systems aligned.

### 7.2 External System Suppression Ingestion (Interim)
For opt-outs originating from external systems (e.g., Constant Contact), an interim Automation Studio workflow ingests and parses suppression lists from those systems.

**Long-term recommendation:** Route external opt-outs through a MuleSoft integration flow that sets the `Has Opted Out of Email` flag on the Salesforce Contact record. Marketing Cloud synchronization will then pick up the suppression automatically.

### 7.3 Non-CRM Audience Contact Creation
Any individual emailed by the University of Kentucky must have a Contact record in Salesforce CRM (CRM anchor requirement). For audiences arriving via manual Excel uploads, implement a validation step — via MuleSoft flow or Automation Studio script — that:
1. Validates incoming records against the CRM.
2. Creates a minimal contact record (email address, first name, source tag) for any subscriber without an existing CRM record before initiating sends.

**Owner:** Emily Brenzel / MuleSoft team (see Open Items)

---

## 8. Out of Scope

- **Bounce/hold handling:** Automatic removal of email holds (caused by bounces) when a subscriber updates their email address in the CRM is a separate process and not included in this implementation.
- **Global cross-organization opt-out:** A university-wide suppression spanning all CRM organizations is a future-state capability targeted for the Data Cloud / Marketing Cloud Next phase.
- **Data Cloud integration:** Future phase. Data Cloud will serve as the cross-profile aggregator for global preferences once implemented.

---

## 9. Open Items & Risks

### ⚠️ OPEN: Query Performance Risk
**Issue:** Historical implementations experienced significant email queue delays when retrieving Salesforce objects during send execution. The proposed architecture queries a specific Contact ID for preference center rendering — it is unconfirmed whether this ID-based query pattern will trigger the same performance bottlenecks.

**Action required:** John López to evaluate potential query performance risks and provide confirmation that the ID-based lookup pattern does not impact email send throughput.

**Blocking:** Phase 4 (parent cloud page build) should not be finalized until this is confirmed. Also affects Phase 2b (admin tool) if the tool performs live field lookups against the Salesforce API.

---

### ⚠️ OPEN: Minimal Contact Record Requirements for Non-CRM Audiences
**Issue:** The preference center requires a CRM anchor for every emailed subscriber. The minimum required fields for a non-CRM-sourced contact record (e.g., Excel upload audiences) have not been formally defined.

**Action required:** John López and Emily Brenzel to define the minimum contact record fields required for non-CRM audiences to ensure FERPA/HIPAA compliance and accurate audit trails.

**Blocking:** Phase 6 (footer rollout and send activation for upload-based audiences) cannot proceed until the contact creation validation workflow is defined and implemented.

---

### ✅ RESOLVED: Admin Tool Authentication Mechanism
**Decision:** MC SSO check. SSJS validates an active Marketing Cloud business unit session before rendering the admin tool. No external identity provider required.

**Resolved:** 2026-06-29

---

## 10. Stakeholders

| Name | Organization | Role |
|---|---|---|
| John Wiesendanger | University of Kentucky | Project lead / IT |
| Jordan Adler | University of Kentucky | IT / architecture |
| Emily Brenzel | University of Kentucky | Marketing operations / governance |
| Zack Adams | University of Kentucky | IT |
| Kamryn Elswick | University of Kentucky | Stakeholder |
| John López | Salesforce | Solution architect (lead) |
| Clint Sheets | Salesforce | Implementation |
| Steve Buesink | Salesforce | Implementation |
| Nickey Revard | Salesforce | Implementation |
| Mariana Aguirre | Salesforce | Implementation |

---

*PRD drafted from architecture review session notes (June 26, 2026). Treat all decisions in Section 4 as aligned unless explicitly reopened by a stakeholder. Open items in Section 9 must be resolved before the phases noted.*

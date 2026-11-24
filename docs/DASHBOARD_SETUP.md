# Phasor dashboard setup

Org: **Phasor - Quanta** · [cloud.samsara.com](https://cloud.samsara.com)

## 1. Forms (required — API cannot create these)

Library → Forms → create / edit (exact titles + required labels in [FORM_TEMPLATES.md](FORM_TEMPLATES.md)):

1. **Asset Inventory** — `be639e48-d208-4e44-9be7-05913cc634f5`
2. **Asset Transfer** — `b985b684-d257-4e26-9d14-769f060cff5a`
3. **Asset Receive** — `e6531fce-5cd6-48b8-b71a-9a0ff737e36d`

Publish each template so `GET /form-templates` returns current revisions.

## 2. Automation Function (already deployed)

**`asset-transfer-assign`** is live with secret `SAMSARA_API_TOKEN` and Phasor params.

The Function routes by **form template id** once it has a submission. It does **not** require `WORKFLOW_ACTION` when the submission can be resolved. It accepts:

| Source | How submission is found |
|--------|-------------------------|
| Event param `FormSubmissionId` / `formSubmissionId` | Direct |
| Nested `_triggerData` / `trigger1.formSubmissionId` / `form.id` | Deep parse |
| Classic Alert (`SamsaraFunctionTriggerSource=alert`) | `alertIncidentTime` (+ optional `assetId`) near-time match |
| Automations Execute Function (`…=api`) | Near-time match on Transfer / Receive / Inventory templates |

### Required Automations / Alerts (dashboard)

> **2026-11-24 status:** completed Transfers are **not** producing Function runs at submit time. See [`AUTOMATION_NOT_FIRING.md`](AUTOMATION_NOT_FIRING.md) for correlation evidence. API cannot list/create Automations or set Function Visibility — Ryan must complete the checklist below in the dashboard.

Create **three** workflows (Automations *or* classic Alerts). Prefer **classic Alerts → Form Submitted → Execute Function** when possible: those pass `alertIncidentTime` / `alertConfigurationId`, which the Function already understands.

| # | Trigger | Template filter | Action |
|---|---------|-----------------|--------|
| 1 | **Form Submitted** | Asset Transfer | Execute Function → `asset-transfer-assign` |
| 2 | **Form Submitted** (or Form Updated → status completed) | Asset Receive | Execute Function → `asset-transfer-assign` |
| 3 | **Form Submitted** | Asset Inventory | Execute Function → `asset-transfer-assign` |

#### Ryan checklist (do in order)

1. **Settings → Developer → Functions → `asset-transfer-assign` → Visibility = customer users** (required for Execute Function). If the Function was not visible, re-open each Alert/Automation and re-select the Function.
2. Create or open the Transfer workflow: trigger **Form Submitted**, template **Asset Transfer** (`b985b684-…`), action **Execute Function → asset-transfer-assign**. Save and **Enable**.
3. Repeat for **Asset Receive** and **Asset Inventory**.
4. **Apply To** = all / org-wide (not a restrictive tag). No score/answer filters that block these forms.
5. Optional: map submission id → event param **`FormSubmissionId`** if the UI allows field mapping.
6. **Verify via Driver App** (not only dashboard complete): submit Transfer → expect a Function run within **~60s** and a new Asset Receive for the recipient. Dashboard-only completes have historically skipped some Form Submitted paths.
7. If still no run: leave Visibility + enablement as above and escalate platform delivery; meanwhile `POST /functions/asset-transfer-assign/runs` with `FormSubmissionId` for orphans.

Other details:

1. Use **Form Submitted** for Transfer and Inventory (fires once when the form reaches its submitted end state). Do **not** use Form Updated for Transfer unless you add a status filter — Form Updated fires on every edit.
2. For Receive, Form Submitted is enough if drivers submit to completion. If you rely on an approval transition, use Form Updated with status = completed.
3. Optional Function params (pin after you copy ids from an Alert/Automation URL or from Function logs):
   - `SUBMIT_AUTOMATION_CONFIGURATION_IDS` / `SUBMIT_ALERT_CONFIGURATION_IDS` — Transfer
   - `UPDATE_AUTOMATION_CONFIGURATION_IDS` / `UPDATE_ALERT_CONFIGURATION_IDS` — Receive
   - `INVENTORY_AUTOMATION_CONFIGURATION_IDS` / `INVENTORY_ALERT_CONFIGURATION_IDS` — Inventory
4. Optional `WORKFLOW_ACTION=submit|update|inventory` — only needed for manual/API test runs without a real submission template. Live Automations should rely on template routing.

### Expected effects

| Trigger | Effect |
|---------|--------|
| Asset Transfer submitted | Status **In Transit** + create prefilled **Asset Receive** for recipient driver |
| Asset Receive completed | Assign asset to ownership driver + status **Active** |
| Asset Inventory submitted | Status **In Inventory** (+ assign if person field present) |

### Redeploy code from this repo

```bash
./scripts/bundle.sh
# Then either upload dist/asset-transfer-assign.zip in Settings → Developer → Functions
# or PATCH /functions/asset-transfer-assign (codePackage) → PUT zip → POST …/deploy
```

## 3. Optional: manual assigner

Settings → Developer → Functions → **Create** `assign-asset-forms` only if you want one-shot assignment of blank forms:

| Setting | Value |
|---------|-------|
| Handler | `entrypoint.main` |
| Zip | `dist/assign-asset-forms.zip` |
| Secret | `SAMSARA_API_KEY` |

Manual test: `AssigneeId=63484`, `AssigneeType=user`, `FormKinds=transfer`.

## 4. Attribute

**Asset Lifecycle Status** was created via API (`ad06e5db-8657-43fe-a409-3e5559b33bfe`) with values In Inventory / In Transit / Active. No dashboard step required unless you want to rename it.

## 5. Verify a Transfer → Receive hook

1. From the **Driver App**, complete an **Asset Transfer** with Asset + recipient driver filled.
2. Within **~60 seconds**, check Function logs for `asset-transfer-assign` (`GET /functions/asset-transfer-assign/logs`) — expect `Function started`.
3. Confirm a new **Asset Receive** exists for the recipient (`formTemplateId=e6531fce-…`).
4. If no Function run appears, the Automation/Alert did not fire — follow the Ryan checklist in section 2 and [`AUTOMATION_NOT_FIRING.md`](AUTOMATION_NOT_FIRING.md). If a run appears but errors on submission id, ensure the near-time + deep parse build is deployed ([`AUTOMATION_HOOK_FIX.md`](AUTOMATION_HOOK_FIX.md)).

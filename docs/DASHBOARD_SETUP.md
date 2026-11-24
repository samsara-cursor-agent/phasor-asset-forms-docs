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
| 2 | **Form Completed** (preferred) or **Form Submitted** | Asset Receive | Execute Function → `asset-transfer-assign` |
| 3 | **Form Submitted** | Asset Inventory | Execute Function → `asset-transfer-assign` |

> **Receive note (2026-11-24):** Driver App Transfer + Receive Automations **are** firing (`activity`). Receive handler only applies ownership/**Active** when status is **`completed`**. Prefer **Form Completed**. If Assignments API is 403 for the org, status still goes Active; ownership must wait until `/fleet/assets/assignments` is enabled. Details: [`RECEIVE_AUTOMATION.md`](RECEIVE_AUTOMATION.md).

#### Ryan checklist (do in order)

1. **Settings → Developer → Functions → `asset-transfer-assign` → Visibility = customer users** (required for Execute Function). If the Function was not visible, re-open each Alert/Automation and re-select the Function.
2. Create or open the Transfer workflow: trigger **Form Submitted**, template **Asset Transfer** (`b985b684-…`), action **Execute Function → asset-transfer-assign**. Save and **Enable**.
3. **Asset Receive (required for ownership/Active):** trigger **Form Completed** (or Form Submitted if completes land as `completed`), template **Asset Receive** (`e6531fce-…`), action **Execute Function → asset-transfer-assign**. Save and **Enable**.
4. Repeat for **Asset Inventory** (Form Submitted).
5. **Apply To** = all / org-wide (not a restrictive tag). No score/answer filters that block these forms.
6. Optional: map submission id → event param **`FormSubmissionId`** if the UI allows field mapping.
7. **Verify via Driver App:** submit Transfer → Function run + new Receive for recipient (~60s). Recipient completes Receive → Function `action=update` + asset **Active** (~60s). See [`RECEIVE_AUTOMATION.md`](RECEIVE_AUTOMATION.md).
8. If Transfer/Receive run but assignment 403s: escalate **Asset Assignments API enablement** for the org; Function still sets Active after the 22:04 redeploy.

Other details:

1. Use **Form Submitted** for Transfer and Inventory (fires once when the form reaches its submitted end state). Do **not** use Form Updated for Transfer unless you add a status filter — Form Updated fires on every edit.
2. For Receive, prefer **Form Completed**. Form Submitted works when the Driver App lands `status=completed` (Phasor Driver App path does). Non-completed Receive runs log and no-op.
3. Function params already pinned from live Driver App Automations (2026-11-24):
   - `SUBMIT_AUTOMATION_CONFIGURATION_IDS=053a0af9-e0b0-4ee9-b49d-800f31f7d44c` — Transfer
   - `UPDATE_AUTOMATION_CONFIGURATION_IDS=3f1c8339-ce22-4e98-8816-f95962517de1` — Receive
   - Optional: `INVENTORY_AUTOMATION_CONFIGURATION_IDS` / `*_ALERT_CONFIGURATION_IDS`
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

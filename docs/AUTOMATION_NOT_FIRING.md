# Diagnosis: Automation does not invoke `asset-transfer-assign` (2026-11-24)

Org: **Phasor - Quanta** (`10011491`) · Function: **`asset-transfer-assign`** (deployed) · Transfer template: **`b985b684-d257-4e26-9d14-769f060cff5a`**

This is separate from the earlier payload-parsing bug (fixed when a run *does* occur as `trigger=api`). Here the Automation never creates a Function run at submit time.

## Verdict

**The Automation/Alert is not invoking the Function.** Function code is fine when called manually. No Function-side config change can create or enable the dashboard Automation — Ryan must fix it in the UI.

### Top 2 causes (with evidence)

| # | Likely cause | Evidence |
|---|--------------|----------|
| **1** | **Form Submitted Automation not actually enabled / wrong product surface / wrong template or Apply-To scope** | `GET /alerts/configurations` returns **empty** (no classic Alerts). Automations have **no public list/create API**. Four completed Transfers today produced **zero** Function starts at submit time (see table below). The only Function runs were manual `POST /functions/.../runs` (`trigger=api`) minutes later. |
| **2** | **Function Visibility does not allow customer users** (required for Execute Function as an Alert/Automation action) | [Samsara Functions docs](https://developers.samsara.com/docs/functions): “The Function must be visible to customer users to appear as an alert action.” / troubleshooting: “Cannot use as alert action → Visibility must allow customer access.” `GET /functions/{name}` does **not** return visibility; create/PATCH schemas also omit it (dashboard-only). A probe `PATCH` with `visibility: "customer"` returned 200 but visibility still cannot be read back via API — **confirm in Settings → Developer → Functions**. |

Secondary / contributing:

- All four Transfers were submitted by **dashboard user** `63484` (Ryan), not a Driver App driver. Docs and prior notes: dashboard-only completes have historically skipped some Form Submitted paths. Retest via **Driver App** after fixing enablement/visibility.
- API cannot create `status=completed` submissions (create allows `notStarted` only; PATCH statuses exclude `completed`), so a controlled API submit→watch loop cannot fully reproduce Form Submitted.

## Correlation proof (submit time → Function run)

| Transfer submission | Submitted at (UTC) | submittedBy | Function run within ±2 min of submit? | Later processing |
|---------------------|--------------------|-------------|----------------------------------------|------------------|
| `01eac0af-68d0-4ec0-9997-6268bad3284f` | 20:58:38 | user 63484 | **No** | Manual API run 21:07:50 (`87288f6e-…`) with FormSubmissionId |
| `4c1d768d-c6e9-483f-a0d5-23b9965c12c0` | 21:06:55 | user 63484 | **No** (no Automation start) | Manual empty-params API run 21:07:57 near-time matched it |
| `feb1e6fe-df79-42fa-a0f7-ec7fb1b49166` | 21:26:38 | user 63484 | **No** — zero log lines mention this id | Never auto-processed |
| `bcd1a50c-93a4-4da1-9afc-005736ed8142` | 21:32:45 | user 63484 | **No** — zero log lines mention this id | Never auto-processed |

Asset Receive forms created today (`a417d91a-…`, `d3688909-…`) appeared only at **21:07:5x** from those manual Function runs — not at Transfer submit times.

Function config snapshot: handler `entrypoint.main`, schedule **off**, secret `SAMSARA_API_TOKEN` present, Transfer template param correct, `effects` empty, code package **deployed**.

## API limits (what we cannot do from here)

| Capability | Result |
|------------|--------|
| List/create **Automations** (Organization → Automations) | Endpoints 404 — dashboard only |
| List classic **Alerts** | `GET /alerts/configurations` → `data: []` |
| Create Form Submitted → Execute Function Alert via API | Form Submitted is **not** in the public Alerts `triggerTypeId` list; Execute Function (`actionTypeId=14`) needs undocumented `ScriptExecution` params |
| Read/set Function **visibility** reliably | Not in GET response; not in documented PATCH body; must use dashboard |
| Complete a Transfer via API to retest Form Submitted | Create only allows `notStarted`; PATCH cannot set `completed` |

## Ryan checklist (exact dashboard steps)

### A. Function visibility (do this first — 30 seconds)

1. Open [cloud.samsara.com](https://cloud.samsara.com) → **Settings → Developer → Functions**.
2. Open **`asset-transfer-assign`**.
3. Set **Visibility** so **customer users** can use it (wording may be “Visible to customer users” / org users). Save.
4. Confirm the Function still shows as **Deployed**.

If Visibility was wrong, existing Automations may have saved without a valid Execute Function target — re-open the Automation and re-select the Function after fixing Visibility.

### B. Prefer classic Alert (more reliable payload for this Function)

Settings / Alerts (Workflows) → create **three enabled** alerts:

| Name (suggested) | Trigger | Conditions | Action |
|------------------|---------|------------|--------|
| Asset Transfer → Function | **Form Submitted** | Template = **Asset Transfer** (`b985b684-…`) | **Execute Function** → `asset-transfer-assign` |
| Asset Receive → Function | **Form Submitted** (or Form Updated + status completed) | Template = **Asset Receive** (`e6531fce-…`) | same Function |
| Asset Inventory → Function | **Form Submitted** | Template = **Asset Inventory** (`be639e48-…`) | same Function |

Checks:

- Alert is **Enabled** (not draft/paused).
- **Apply To** = organization-wide / all (not a tag that excludes dashboard or driver testers).
- No score/answer filter that would block these Transfers.
- Function appears in the Execute Function picker (if missing → fix Visibility).

### C. If using Organization → Automations instead

1. **Settings → Organization → Automations**.
2. Open the Transfer automation (or create one).
3. Trigger = **Form Submitted**, template = **Asset Transfer**.
4. Action = **Execute Function** → `asset-transfer-assign` (re-select after Visibility fix).
5. Optional but preferred: map submission id → Function event param **`FormSubmissionId`**.
6. Save and **Enable**. Repeat for Receive + Inventory.

Connected Workflows / Automations license + permissions are required for Forms triggers ([KB](https://kb.samsara.com/hc/en-us/articles/48838365816845-Forms-Triggers-and-Actions-in-Automations)).

### D. Verify (expect a Function run within ~60 seconds)

1. Prefer **Driver App**: assign or open **Asset Transfer**, fill Asset + recipient, **submit** to completion.
2. Within **~60 seconds**, open Function **Executions/Logs** or:

```bash
# Window covering the submit
curl -sS -G 'https://api.samsara.com/functions/asset-transfer-assign/logs' \
  --data-urlencode "startTime=<submit-30s>" \
  --data-urlencode "endTime=<submit+2m>" \
  -H "Authorization: Bearer $SAMSARA_API_TOKEN"
```

3. Expect a new run (`Function started`) and a new **Asset Receive** for the recipient (`formTemplateId=e6531fce-…`).
4. If **still zero runs** after Visibility + enabled Form Submitted Alert/Automation + Driver App submit → escalate as platform Form Submitted → Execute Function delivery (not Function code). Meanwhile process orphans manually:

```bash
curl -sS -X POST 'https://api.samsara.com/functions/asset-transfer-assign/runs' \
  -H "Authorization: Bearer $SAMSARA_API_TOKEN" -H 'Content-Type: application/json' \
  -d '{"paramsOverride":{"FormSubmissionId":"<transfer-submission-id>"}}'
```

## What we changed from this investigation

- **No Function code redeploy** (not a code bug for “no run created”).
- Attempted `PATCH` visibility via API; **cannot confirm** via GET — Ryan must set Visibility in the dashboard.
- Docs updated: this file + checklist cross-links in `DASHBOARD_SETUP.md`.

## Related

- Payload parsing when runs *do* fire: [`AUTOMATION_HOOK_FIX.md`](AUTOMATION_HOOK_FIX.md)
- Full setup: [`DASHBOARD_SETUP.md`](DASHBOARD_SETUP.md)

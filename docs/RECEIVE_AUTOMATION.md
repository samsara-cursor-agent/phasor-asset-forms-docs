# Asset Receive: Automation + Function diagnosis (2026-11-24)

Org: **Phasor - Quanta** (`10011491`) · Function: **`asset-transfer-assign`** · Receive template: **`e6531fce-5cd6-48b8-b71a-9a0ff737e36d`**

## Verdict

| Question | Answer |
|----------|--------|
| Is Receive Automation missing? | **No** — Driver App completed Receive **did** invoke the Function (`trigger=activity`, `automationConfigId=3f1c8339-…`). |
| Did Transfer Automation fire? | **Yes** — `trigger=activity`, `automationConfigId=053a0af9-…`. |
| Why did Receive look broken? | Function routed correctly to `action=update` / status `completed`, then **failed on `/fleet/assets/assignments` (403 not enabled)**. Before the 22:04 redeploy, that error aborted **before** setting status **Active**, so ownership + Active both looked like “didn’t fire”. |
| Function bug (near-time)? | **Yes (fixed)** — when Automations omit `FormSubmissionId`, cross-template near-time matching used to prefer Transfer first. Redeployed build ranks by **newest matching submission** (Receive wins over the Transfer that created it). |

## Correlation (Driver App path)

| Event | Time (UTC) | Evidence |
|-------|------------|----------|
| Transfer `8ee2bca9-…` submitted by driver `61578495` | 21:55:16 | Function `f8ccdceb-…` (`activity`) → created Receive `6653ff32-…` |
| Receive `6653ff32-…` completed by driver `61604643` | 21:58:21 | Function `2049facb-…` (`activity`) → `action=update`, status `completed` → **403** Asset Assignments API |
| Manual re-run after fix | 22:04:52 | Function `b0a39265-…` → set **Active**; logged Assignments API unavailable (soft-fail) |
| Empty-params near-time probe | 22:05:01 | Function `62a0d6b8-…` matched Transfer + Receive, **chose Receive** (`matchCount=2`) |

Asset `281475009229678` (Ryan-Desk): status **Active** after the fixed re-run. Driver ownership assignment still requires Samsara to enable `/fleet/assets/assignments` for this org.

Pinned Function params (live):

- `SUBMIT_AUTOMATION_CONFIGURATION_IDS=053a0af9-e0b0-4ee9-b49d-800f31f7d44c`
- `UPDATE_AUTOMATION_CONFIGURATION_IDS=3f1c8339-ce22-4e98-8816-f95962517de1`

## What the Receive handler requires

`_handle_receive_complete` only assigns + sets **Active** when form status is **`completed`**.

| Automation trigger | Behavior |
|--------------------|----------|
| **Form Completed** (preferred) | Runs when status is completed → assign + Active |
| **Form Submitted** | OK if Driver App submit lands as `completed` (observed). If status is still `notStarted` / `inProgress` / `submitted`, Function **logs and no-ops** until a completed run |
| Form Updated (no status filter) | Too noisy — avoid |

## Ryan dashboard steps (Receive)

Transfer Automation already works. Confirm Receive the same way:

1. **Settings → Developer → Functions → `asset-transfer-assign`**
   - Visibility = customer users
   - Confirm **Deployed** (package updated 2026-11-24T22:04:24Z)
2. **Settings → Organization → Automations** (or classic Alerts)
3. Open / create **Asset Receive → Function**:
   - Trigger: **Form Completed** (best) **or** Form Submitted
   - Template filter: **Asset Receive** (`e6531fce-…`)
   - Action: **Execute Function → `asset-transfer-assign`**
   - Optional: map submission id → event param **`FormSubmissionId`**
   - Save + **Enable** · Apply To = org-wide
4. (Already done for Transfer) Keep Transfer Form Submitted/Completed → same Function.
5. Ask Samsara / internal platform to **enable Asset Assignments API** (`GET/POST /fleet/assets/assignments`) for org `10011491`. Until then, Receive will set **Active** but cannot create driver ownership via API (assign manually in dashboard if needed).

### Verify

1. Driver App: complete Transfer → expect Receive assigned to recipient within ~60s.
2. Recipient driver completes Receive → within ~60s Function log should show `action=update` and status → **Active**.
3. If Assignments API still disabled, log line: `Asset Assignments API unavailable; Receive status set to Active…`.

Manual orphan re-run:

```bash
curl -sS -X POST 'https://api.samsara.com/functions/asset-transfer-assign/runs' \
  -H "Authorization: Bearer $SAMSARA_API_TOKEN" -H 'Content-Type: application/json' \
  -d '{"paramsOverride":{"FormSubmissionId":"<receive-submission-id>"}}'
```

## Code changes redeployed (22:04Z)

1. Cross-template near-time ranking → newest activity (Receive preferred on ties).
2. Receive: set **Active** before assignment; soft-fail 403/404 Assignments API with clear log.
3. Clearer log when Receive fires but status ≠ completed.
4. Pin Transfer/Receive `automationConfigId`s in Function params + code defaults.

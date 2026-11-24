# Automation → `asset-transfer-assign` hook fix (2026-11-24)

## Root cause

1. **Automation Execute Function payloads often omit `formSubmissionId`.** Observed Function runs with `SamsaraFunctionTriggerSource=api` only had configured Function params + `SamsaraFunctionCorrelationId` / `SamsaraFunctionTriggerSource` (sometimes `WORKFLOW_ACTION`). No `_triggerData`, no `alertIncidentTime`, no `automationConfigId`.
2. **Near-time submission lookup ignored `api` triggers.** `_resolve_submission_id` only fell back for `alert` / `activity`, so Automations that invoke via the Functions API failed with `Could not resolve form submission ID`.
3. **Missing `WORKFLOW_ACTION` aborted before recovery.** When Automations also omitted `WORKFLOW_ACTION`, the Function exited with `Could not determine workflow action` before attempting a template/time match.
4. **Some completed Transfers never produced a Function run at all** (example: `01eac0af-68d0-4ec0-9997-6268bad3284f` at 2026-11-24T20:58:38Z; also `feb1e6fe-…` / `bcd1a50c-…` later the same day). That is a dashboard Automation enablement / Form Submitted trigger / Function Visibility issue, not Function code — see [`AUTOMATION_NOT_FIRING.md`](AUTOMATION_NOT_FIRING.md).

Live Driver App Automations now emit `automationConfigId` (Transfer `053a0af9-…`, Receive `3f1c8339-…`); those are pinned on the Function.

## Fix (this branch)

- Deep-parse nested trigger payloads (`_triggerData`, `trigger1.formSubmissionId`, `form.id`, JSON string params).
- Treat `api` / `automation` / `workflow` like alert for near-time submission matching (30‑minute lookback).
- When action is unknown, scan Transfer / Receive / Inventory templates and route by template id.
- Prefer template routing whenever a submission is loaded; `WORKFLOW_ACTION` is optional for live Automations.
- **2026-11-24T22:04Z:** cross-template near-time ranking uses **newest matching submission** (no longer Transfer-first). Receive completed runs set **Active** even when Assignments API returns 403.
- Document exact Automation setup in `DASHBOARD_SETUP.md` and Receive evidence in `RECEIVE_AUTOMATION.md`.

Live Function package was redeployed **2026-11-24T22:04:24Z** with the expanded modules (not the hex loaders).

## Repo packaging note

On the automation-form-hook branch, `form_utils.py` / `entrypoint.py` / `handlers.py` may be short zlib loaders over `*.chunk.*.hex` so GitHub MCP can push without corrupting large blobs. Before editing or bundling for deploy:

```bash
python3 scripts/expand_hex_modules.py
./scripts/bundle.sh
```

`python3 scripts/expand_hex_modules.py --check` verifies chunks decompress without writing.

## Dashboard steps still required

See [`AUTOMATION_NOT_FIRING.md`](AUTOMATION_NOT_FIRING.md), [`RECEIVE_AUTOMATION.md`](RECEIVE_AUTOMATION.md), and [`DASHBOARD_SETUP.md`](DASHBOARD_SETUP.md).

## Verify

```bash
# After deploy — process today's orphaned Transfer (controlled)
curl -sS -X POST 'https://api.samsara.com/functions/asset-transfer-assign/runs' \
  -H "Authorization: Bearer $SAMSARA_API_TOKEN" -H 'Content-Type: application/json' \
  -d '{"paramsOverride":{"FormSubmissionId":"01eac0af-68d0-4ec0-9997-6268bad3284f"}}'

# Simulate Automation Execute Function (no FormSubmissionId) — should near-time match newest template
curl -sS -X POST 'https://api.samsara.com/functions/asset-transfer-assign/runs' \
  -H "Authorization: Bearer $SAMSARA_API_TOKEN" -H 'Content-Type: application/json' \
  -d '{"paramsOverride":{}}'
```

Then check logs and confirm a Receive submission exists for template `e6531fce-…`.

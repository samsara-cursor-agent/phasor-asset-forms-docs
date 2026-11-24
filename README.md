# Phasor asset-forms docs (public)

Public mirror of dashboard/automation docs for **Phasor - Quanta** asset inventory / transfer / receive.

Private Function code lives in [`samsara-cursor-agent/phasor-asset-forms`](https://github.com/samsara-cursor-agent/phasor-asset-forms).

## Start here

| Doc | When to read |
|-----|----------------|
| [docs/DASHBOARD_SETUP.md](docs/DASHBOARD_SETUP.md) | Create forms + Automations (Transfer / **Receive** / Inventory) |
| [docs/RECEIVE_AUTOMATION.md](docs/RECEIVE_AUTOMATION.md) | Receive fired but Active/assign looked broken — evidence + Ryan steps |
| [docs/AUTOMATION_NOT_FIRING.md](docs/AUTOMATION_NOT_FIRING.md) | Transfer submits with **zero** Function runs |
| [docs/AUTOMATION_HOOK_FIX.md](docs/AUTOMATION_HOOK_FIX.md) | Runs fire as `trigger=api` without FormSubmissionId |

## Receive quick steps (Ryan)

1. Confirm Function `asset-transfer-assign` Visibility = customer users + Deployed.
2. Automation: **Form Completed** (preferred) or Form Submitted → template **Asset Receive** → Execute Function `asset-transfer-assign` → Enable.
3. Driver completes Receive → expect asset **Active** within ~60s.
4. If logs show Assignments API unavailable (403): ask Samsara to enable `/fleet/assets/assignments` for org `10011491` (status Active still applies).

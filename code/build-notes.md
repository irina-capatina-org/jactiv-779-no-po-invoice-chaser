# Build Notes — JACTIV-779 No-PO Invoice Chaser

## Plan

1. Read skills (uipath-api-workflow, uipath-platform, uipath-solution) and all design docs
2. Probe CLI surface (`uip solution init --help`)
3. Scaffold solution `no-po-invoice-chaser-779` from `code/`
4. Scaffold API workflow project `no-po-invoice-chaser-api` inside solution
5. Extract reference Workflow.json from `docs/architectural-considerations.md` §4.5 via `awk`
6. Write `bindings_v2.json` for Coupa and Slack connections
7. Write two solution connection-resource JSON files from the skill template
8. Pack proof + run `validate-build.sh`; fix connection resource format (first attempt used wrong schema)

## Summary

Single API Workflow project inside a UiPath Solution, implementing the No-PO Invoice Chaser as the reference workflow from §4.5. No SDD deviations.

## Task Table

| Task | Project | Status | Notes |
|---|---|---|---|
| Solution scaffold | no-po-invoice-chaser-779 | done | |
| API Workflow scaffold + Workflow.json | no-po-invoice-chaser-api | done | extracted from §4.5 |
| bindings_v2.json | no-po-invoice-chaser-api | done | Coupa + Slack |
| Connection resource files | resources/solution_folder | done | used skill template schema |
| uip solution pack proof | — | done | |
| validate-build.sh gate | — | done | 19 activities, passed |

## Deviations from the SDD

None. The reference workflow satisfies every BR and the SDD's output schema. No lines changed.

## Left for a Human

None — no selectors, no credential values, no absolute URLs in the workflow. Connections pre-exist in the `Fusion2026` folder (§4 table).

## How to Test This

```bash
# Static validation
uip api-workflow validate code/no-po-invoice-chaser-779/no-po-invoice-chaser-api/Workflow.json --output json

# Runnability gate
bash .github/scripts/validate-build.sh code docs/architectural-considerations.md

# Pack proof
uip solution pack code/no-po-invoice-chaser-779 /tmp/buildcheck \
  --name no-po-invoice-chaser-779 --version 0.0.1 --output json
```

Live run (requires `uip login` + Coupa/Slack connections enabled in Fusion2026 folder):

```bash
uip api-workflow run code/no-po-invoice-chaser-779/no-po-invoice-chaser-api/Workflow.json --output json
```

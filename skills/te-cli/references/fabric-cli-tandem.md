# `te` + `fab` tandem workflows

`te` (Tabular Editor CLI) and `fab` (Fabric CLI) split cleanly along a single seam. `te` owns the semantic model itself: it parses TMDL/BIM locally, edits objects (`set`, `add`, `rm`, `mv`, `format`), runs local validation and BPA, analyzes VertiPaq from a live XMLA model or imported VPAX, generates TMSL, deploys over XMLA, and runs DAX queries, refreshes, and tests against a live XMLA endpoint. `fab` owns everything around the model in the service: discovering workspaces and items, resolving display names and GUIDs, exporting and importing item definitions over the Fabric REST API, managing permissions and capacities, driving deployment pipelines, and triggering refreshes via the Power BI REST API. Neither tool crosses into the other's half: `fab` has no DAX parser, no BPA engine, and no XMLA/TOM layer, so it cannot validate or BPA-check a model; `te` has no Fabric REST client, so it cannot enumerate workspaces, resolve item GUIDs, or trigger a REST refresh. The two share Azure AD identity but cache credentials independently, so authenticate both at the start of any tandem session.

Before each workflow: `fab auth status` and `te auth status`. If either is unauthenticated, ask the user to run `fab auth login` / `te auth login`. During preview, run `te <command> --help` and `fab <command> --help` the first time a command is composed; both surfaces are still moving.

## 1. Export, edit, deploy back (the common round-trip)

Pull a production model to local TMDL with `fab`, edit and quality-gate with `te`, push it back over XMLA with `te deploy`.

```bash
# 0. Confirm the item path is real before exporting
fab exists "Production.Workspace/Sales.SemanticModel"

# 1. Pre-create the output dir (fab export will not create parents)
mkdir -p ./sales-export

# 2. Download the model definition as a TMDL folder
fab export "Production.Workspace/Sales.SemanticModel" -o ./sales-export -f
#    -> ./sales-export/Sales.SemanticModel/definition/  holds model.tmdl, tables/, etc.

# 3. Confirm te parses the exported TMDL; settle on the path that loads
te load ./sales-export/Sales.SemanticModel/definition

# 4. Baseline gates BEFORE editing (separate pre-existing issues from yours)
te validate -m ./sales-export/Sales.SemanticModel/definition --errors-only
te bpa run --fail-on error -m ./sales-export/Sales.SemanticModel/definition

# 5. Edit (each mutation needs --save to persist)
te set "_Measures/Revenue" -q formatString -i "#,0.00" -m ./sales-export/Sales.SemanticModel/definition --save
te format --save -m ./sales-export/Sales.SemanticModel/definition

# 6. Re-gate: no new errors or violations introduced
te validate -m ./sales-export/Sales.SemanticModel/definition --errors-only
te bpa run --fail-on error -m ./sales-export/Sales.SemanticModel/definition

# 7. Deploy back over XMLA (BPA gate runs by default; --force required non-interactively)
te deploy ./sales-export/Sales.SemanticModel/definition -s "Production" -d "Sales" --force
```

Per-step purpose: `fab exists` fails fast on a wrong name; `mkdir -p` avoids `[InvalidPath]`; `te load` confirms the parse and pins the path; the baseline gates separate pre-existing breakage from edits being introduced; `--save` persists each mutation (staged in memory otherwise); the second gate is the quality bar before deploy; `te deploy` writes the definition back through XMLA, re-running BPA on the way out.

Notes that bite:
- `fab export` nests TMDL under `<Model>.SemanticModel/definition/`. Run `te load` once to confirm whether the binary wants the `definition/` folder or its `.SemanticModel` parent, then use that exact path on every later `te` call.
- `te connect` state does not survive across separate shell calls. Pass `-m` (and `-s`/`-d` for remote) every time, or set `TE_SESSION=<name>` before the first call.
- The BPA gate is ON by default for `te deploy`. If pre-existing violations block the deploy, use `--skip-bpa` once and log it, or `--fix-bpa` to auto-remediate; do not silently bypass.
- `te deploy` overwrites by default. Add `--create-only` to refuse if the model already exists (it errors if it does), to prevent clobbering.

### Variant: import back with `fab` instead of `te deploy`

Use this when XMLA write is blocked (Pro/PPU without XMLA, or tenant policy). `fab import` goes through the Fabric item-definition REST API and works on any SKU. It does NOT run BPA or validation, so the `te` gate before it is the only safety net.

```bash
te validate -m ./sales-export/Sales.SemanticModel/definition --errors-only
te bpa run --fail-on error -m ./sales-export/Sales.SemanticModel/definition
fab import "Production.Workspace/Sales.SemanticModel" -i ./sales-export/Sales.SemanticModel -f
```

`fab import` overwrites the whole item definition (no partial update) and does not refresh data. Pass the `.SemanticModel` folder (the one containing `.platform` and `definition/`), not the inner `definition/`. Trigger a refresh afterward if needed (workflow 5).

### Variant: `te`-only pull (no `fab`)

`te save` can read a remote model and write it to local disk, so the round-trip does not strictly require `fab export` when XMLA is available:

```bash
te save -s "Production" -d "Sales" -o ./sales-export --serialization tmdl
```

Use `fab export` when XMLA is blocked or when the `.platform`/PBIP scaffolding is also wanted; use `te save` when an XMLA connection already exists and only the model source is wanted.

## 2. Discover with `fab`, connect `te` by display name

When the exact workspace or model name is unknown or its casing is uncertain, resolve it with `fab` first, then feed the canonical display name into `te -s`/`-d`.

```bash
fab ls                                              # list visible workspaces
fab find 'sales' -P type=SemanticModel -l           # substring search; -l adds id + workspace_id
fab exists "Sales Analytics.Workspace/Sales Model.SemanticModel"   # confirm the exact path

te load -s "Sales Analytics" -d "Sales Model"       # te takes the bare display name (no .Workspace suffix)
te bpa run --fail-on error -s "Sales Analytics" -d "Sales Model"    # gate the live model, no local export
```

Boundary: `fab` resolves and confirms names against the Fabric REST API and OneLake catalog; `te` connects to the XMLA endpoint using the display name directly. Watch the path-shape mismatch: `fab` uses dot-extension syntax (`"Name.Workspace"`, `"Model.SemanticModel"`), but `te -s` takes the bare workspace display name and `te -d` the bare model name. Strip the `.SemanticModel` / `.Workspace` suffixes when handing off. Quote any name with spaces in both CLIs.

## 3. Extract GUIDs with `fab`, refresh after a `te` change

The Power BI REST API needs raw GUIDs that only `fab` can resolve from display names. `te` validates the model; `fab` triggers the refresh.

```bash
te validate -s "Production" -d "Sales Model" --errors-only      # don't refresh a broken model

WS_ID=$(fab get "Production.Workspace" -q "id" | tr -d '"')
MODEL_ID=$(fab get "Production.Workspace/Sales Model.SemanticModel" -q "id" | tr -d '"')

fab api -A powerbi "groups/$WS_ID/datasets/$MODEL_ID/refreshes" -X post -i '{"type":"Full"}'
fab api -A powerbi "groups/$WS_ID/datasets/$MODEL_ID/refreshes?\$top=1" -q "text.value[0].{status:status,started:startTime}"
```

`fab get -q "id"` returns a quoted JSON string; always pipe through `tr -d '"'` or the GUID carries literal quotes that break URL construction. `te refresh --type full -s ws -d model` also triggers a refresh over XMLA; use `fab api` when the GUIDs are already in scope or when XMLA refresh is not available. The refresh is async; the `?$top=1` check is a sanity probe, not a completion wait.

## 4. Promote dev to test/prod with quality gates

Export from dev, diff against the target, gate, then deploy. The local round-trip is deliberate: `fab cp` can copy workspace-to-workspace faster but skips every `te` gate.

```bash
mkdir -p ./promote/dev ./promote/prod
fab export "Dev.Workspace/Sales.SemanticModel" -o ./promote/dev -f
fab export "Production.Workspace/Sales.SemanticModel" -o ./promote/prod -f

# te diff requires two positional local model paths
te diff \
  ./promote/dev/Sales.SemanticModel/definition \
  ./promote/prod/Sales.SemanticModel/definition
case $? in
  0) echo "Definitions are identical" ;;
  1) echo "Definitions differ; review before promotion" ;;
  2) echo "Diff failed" >&2; exit 1 ;;
esac

te validate -m ./promote/dev/Sales.SemanticModel/definition --errors-only
te bpa run --fail-on error -m ./promote/dev/Sales.SemanticModel/definition

# Deploy, including RLS roles and members
te deploy ./promote/dev/Sales.SemanticModel/definition \
  -s "Production" -d "Sales" \
  --deploy-roles --deploy-role-members --force

fab exists "Production.Workspace/Sales.SemanticModel"           # confirm it landed
```

`te diff` accepts exactly two positional model paths; it cannot compare one local path directly to `-s`/`-d`. Export both definitions first as shown. `--deploy-roles` / `--deploy-role-members` are opt-in; omit them when RLS is managed independently in prod.

### Variant: governed promotion via deployment pipeline

`te` provides the quality gate and an optional TMSL audit artifact; `fab` drives the Fabric deployment pipeline (which preserves item IDs across stages, so thin reports do not need rebinding).

```bash
command -v jq >/dev/null || { echo "jq is required" >&2; exit 1; }
te bpa run --fail-on error --ci azdo --non-interactive -m ./dev-export/Sales.SemanticModel/definition

# Optional: emit the TMSL a direct XMLA deploy WOULD run, as an audit artifact (does not execute)
te deploy ./dev-export/Sales.SemanticModel/definition -s "Dev" -d "Sales" --xmla - > ./audit/deploy.tmsl

PIPELINES_RESPONSE=$(fab api "deploymentPipelines" --output_format json)
[ "$(printf '%s' "$PIPELINES_RESPONSE" | jq -r '.status_code')" = "200" ] ||
  { echo "Pipeline lookup failed" >&2; exit 1; }
PIPELINE_ID=$(printf '%s' "$PIPELINES_RESPONSE" |
  jq -er '[.text.value[] | select(.displayName == "Sales Pipeline") | .id][0]') ||
  { echo "Pipeline not found" >&2; exit 1; }

STAGES_RESPONSE=$(fab api "deploymentPipelines/$PIPELINE_ID/stages" --output_format json)
[ "$(printf '%s' "$STAGES_RESPONSE" | jq -r '.status_code')" = "200" ] ||
  { echo "Stage lookup failed" >&2; exit 1; }
DEV_STAGE=$(printf '%s' "$STAGES_RESPONSE" |
  jq -er '[.text.value[] | select(.order == 0) | .id][0]') ||
  { echo "Dev stage not found" >&2; exit 1; }
TEST_STAGE=$(printf '%s' "$STAGES_RESPONSE" |
  jq -er '[.text.value[] | select(.order == 1) | .id][0]') ||
  { echo "Test stage not found" >&2; exit 1; }

# Promote. fab api returns an envelope containing status_code, headers, and text.
DEPLOY_RESPONSE=$(fab api -X post "deploymentPipelines/$PIPELINE_ID/deploy" \
  -i "{\"sourceStageId\":\"$DEV_STAGE\",\"targetStageId\":\"$TEST_STAGE\",\"note\":\"BPA-gated\"}" \
  --show_headers --output_format json)
DEPLOY_HTTP=$(printf '%s' "$DEPLOY_RESPONSE" | jq -r '.status_code')

# A 202 response is a Fabric long-running operation; capture its explicit ID.
case "$DEPLOY_HTTP" in
  200) ;;
  202)
    OPERATION_ID=$(printf '%s' "$DEPLOY_RESPONSE" |
      jq -er '.headers | to_entries[] | select(.key | ascii_downcase == "x-ms-operation-id") | .value') ||
      { echo "Deployment operation ID missing" >&2; exit 1; }
    while :; do
      OPERATION_RESPONSE=$(fab api "operations/$OPERATION_ID" --output_format json)
      [ "$(printf '%s' "$OPERATION_RESPONSE" | jq -r '.status_code')" = "200" ] ||
        { echo "Deployment status request failed" >&2; exit 1; }
      DEPLOY_STATUS=$(printf '%s' "$OPERATION_RESPONSE" | jq -r '.text.status')
      case "$DEPLOY_STATUS" in
        Succeeded) break ;;
        NotStarted|Running) sleep 30 ;;
        Failed|Cancelled|Error) echo "Deployment $DEPLOY_STATUS" >&2; exit 1 ;;
        *) echo "Unexpected deployment status: $DEPLOY_STATUS" >&2; exit 1 ;;
      esac
    done
    ;;
  *) printf '%s' "$DEPLOY_RESPONSE" | jq -r '.text' >&2; exit 1 ;;
esac
```

The TMSL audit artifact describes a direct XMLA deploy from the local source, not what the pipeline promotion will do; treat it as a reference, not a contract. Fabric returns `x-ms-operation-id` only for the asynchronous (`202`) path; capture it from the same `fab api --show_headers` response. Pipelines copy definitions only; refresh separately (workflow 5).

## 5. Post-deploy: refresh, then DAX regression tests

A pipeline or XMLA deploy moves definitions, not data. Refresh with `fab`, wait, then run the `te` test suite against the live model.

```bash
command -v jq >/dev/null || { echo "jq is required" >&2; exit 1; }
WS_ID=$(fab get "Test.Workspace" -q "id" | tr -d '"')
MODEL_ID=$(fab get "Test.Workspace/Sales.SemanticModel" -q "id" | tr -d '"')
REFRESH_RESPONSE=$(fab api -A powerbi "groups/$WS_ID/datasets/$MODEL_ID/refreshes" \
  -X post -i '{"type":"full","commitMode":"transactional"}' \
  --show_headers --output_format json)
REFRESH_HTTP=$(printf '%s' "$REFRESH_RESPONSE" | jq -r '.status_code')

case "$REFRESH_HTTP" in
  200) ;;
  202)
    REFRESH_ID=$(printf '%s' "$REFRESH_RESPONSE" |
      jq -er '.headers | to_entries[] | select(.key | ascii_downcase == "x-ms-request-id") | .value') ||
      { echo "Refresh request ID missing" >&2; exit 1; }
    while :; do
      REFRESH_STATUS_RESPONSE=$(fab api -A powerbi \
        "groups/$WS_ID/datasets/$MODEL_ID/refreshes/$REFRESH_ID" --output_format json)
      REFRESH_HTTP=$(printf '%s' "$REFRESH_STATUS_RESPONSE" | jq -r '.status_code')
      [ "$REFRESH_HTTP" = "200" ] || [ "$REFRESH_HTTP" = "202" ] ||
        { echo "Refresh status request failed" >&2; exit 1; }
      REFRESH_STATUS=$(printf '%s' "$REFRESH_STATUS_RESPONSE" |
        jq -r '.text.extendedStatus // .text.status // "Unknown"')
      case "$REFRESH_STATUS" in
        Completed) break ;;
        NotStarted|InProgress|Unknown) sleep 30 ;;
        Failed|Cancelled|Error|TimedOut|Disabled) echo "Refresh $REFRESH_STATUS" >&2; exit 1 ;;
        *) echo "Unexpected refresh status: $REFRESH_STATUS" >&2; exit 1 ;;
      esac
    done
    ;;
  *) printf '%s' "$REFRESH_RESPONSE" | jq -r '.text' >&2; exit 1 ;;
esac

te test run --suite ./.te-tests/ --ci azdo --trx test-results.trx --non-interactive -s "Test Workspace" -d "Sales"
```

Do not run `te test` before the refresh finishes; stale or empty data causes false failures. The enhanced refresh request returns `x-ms-request-id` on the asynchronous (`202`) path, and polling that exact ID avoids confusing this run with another refresh in history. `te test run` needs a `.te-tests/` suite; scaffold one first with `te test init --example`. In CI, set `AZURE_CLIENT_ID`/`AZURE_CLIENT_SECRET`/`AZURE_TENANT_ID` and pass `--auth env`.

## 6. Governance: enumerate with `fab`, audit with `te`

Discovery only comes from `fab`; `te` has no workspace or item listing surface. Pair them for tenant-wide BPA audits, VertiPaq profiling, and unused-column sweeps.

```bash
# Enumerate semantic models across all visible workspaces
mkdir -p ./.te-audit
fab find '' -P type=SemanticModel -l --output_format json > ./.te-audit/models.json
jq -r '.[] | "\(.workspace)/\(.name)"' ./.te-audit/models.json

# Per model: export, then run metadata-only checks on local TMDL
fab export "Production.Workspace/Sales.SemanticModel" -o ./.te-audit -f

te bpa run -m ./.te-audit/Sales.SemanticModel/definition --rules ./BPARules.json --fail-on error --ci github
te deps --unused --hidden -m ./.te-audit/Sales.SemanticModel/definition --output-format json

# VertiPaq requires a live model; export VPAX once for later offline analysis
te vertipaq --columns --detail --top 20 --export ./.te-audit/Sales.vpax -s "Production" -d "Sales"
te vertipaq --import ./.te-audit/Sales.vpax --columns --detail --top 20
```

For properties not returned by `fab find`, inspect a known item with `fab get "Production.Workspace/Sales.SemanticModel" -v` and use `fab api` for service metadata such as refresh history; exact response fields depend on the API and caller permissions. A TMDL export is not a VertiPaq snapshot: obtain statistics from a live `te vertipaq -s <workspace> -d <model>` call, optionally `--export` them to VPAX, then use `--import <file.vpax>` offline. `te deps --unused` flags objects with no DAX references, but a relationship key column can show as "unused" despite being load-bearing; inspect with `te get` before removing.

## Boundaries and gotchas

- **Path layout after `fab export`.** TMDL lands in `<Model>.SemanticModel/definition/`. Local `te validate`/`bpa run` and `te deploy` operate on the model source; `fab import` wants the `.SemanticModel` folder (with `.platform`). `te vertipaq` does not analyze local TMDL: use live `-s`/`-d` or `--import <file.vpax>`. Confirm the exact local model target with `te load` first.
- **Return path: XMLA vs REST.** `te deploy` writes via XMLA and runs BPA inline but needs XMLA write (Premium/Fabric capacity, PPU with XMLA, or Trial). `fab import` writes via the Fabric REST API on any SKU but runs no gate. Pick by SKU and by whether the inline BPA gate is wanted.
- **`te diff` exit codes.** `0` means identical, `1` means differences found, and `2` means an error (bad path, load failure, or similar). Branch on all three explicitly in automation.
- **Two-document JSON from `te bpa run --fix`.** With `--output-format json`, a `--fix` run emits two concatenated JSON documents (scan result, then fix summary). Pipe through `jq --slurp` (`jq -s '.[0]'` / `.[1]`) or drop `--output-format json` and read text. A single-document parser fails with trailing-token errors.
- **Quoted GUIDs from `fab get`.** `fab get -q "id"` returns a quoted string; always `| tr -d '"'` before interpolating into an API path.
- **`fab` REST refresh path shape.** `fab api -A powerbi` uses the Power BI `groups/<ws-id>/datasets/<model-id>` shape; the Fabric REST API uses `workspaces/<ws-id>/semanticModels/<model-id>`. Same GUIDs, different URL.
- **No native pipe between the CLIs.** `fab find --output_format json` produces discovery data; a shell intermediary (`jq`, `while read`) feeds model names into a `te` loop. `te` has no multi-model loop and no service-discovery command of its own.
- **Always `mkdir -p` before `fab export`** (it does not create parents), and **always `-f`** on `fab export`/`import` (skips the sensitivity-label / overwrite prompt). If sensitivity labels or DLP policies are in play, confirm with the user before exporting; `-f` strips the label on export.
- **CI flags on `te`.** Pass `--non-interactive` and `--force` (on `te deploy`) or the confirmation prompt defaults to `n` and hangs the pipeline. Use `--auth env` with `AZURE_CLIENT_*`; never put a secret on the command line.
- **Thin-report rebinding is a `fab` job.** After a `fab import` or `fab cp` of a thin report to a new workspace, rebind with `fab set "<ws>/<Report>.Report" -q semanticModelId -i "<target-model-id>"`. Deployment pipelines preserve IDs and skip this; manual import/copy does not.

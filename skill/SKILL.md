---
name: te-cli
version: 0.1.0
description: Expert guidance for the cross-platform Tabular Editor CLI (`te`) — single binary for managing Power BI / Analysis Services semantic models from the terminal on macOS, Linux, and Windows. Use whenever the user mentions "te CLI", "Tabular Editor CLI" (without the "2"), or invokes `te <command>` — e.g. `te init`, `te load`, `te save`, `te deploy`, `te refresh`, `te bpa run`, `te validate`, `te query`, `te script`, `te add`, `te set`, `te rm`, `te mv`, `te ls`, `te get`, `te find`, `te deps`, `te diff`, `te format`, `te vertipaq`, `te test`, `te interactive`, `te connect`, `te auth`, `te profile`, `te session`, `te config`, `te macro`, `te open`, `te incremental-refresh`, `te migrate`. NOT for the legacy Windows-only `TabularEditor.exe` (TE2) — for that, see the TE2 product directly.
---

# Tabular Editor CLI (`te`)

Guidance for the cross-platform `te` CLI: a single self-contained executable that loads, edits, validates, deploys, refreshes, and tests semantic models against TMDL/BIM files, Power BI Desktop, and cloud workspaces (Power BI / Fabric / AS Azure / SSAS).

> [!IMPORTANT]
> **Limited Public Preview.** Preview builds stop functioning after **2026-09-30**. No license required during preview. Production CI/CD adoption is at the user's discretion.
> Issues and feedback: https://github.com/TabularEditor/CLI

> [!IMPORTANT]
> Distinct from the legacy Windows-only `TabularEditor.exe` (TE2). If the user invokes TE2 flag syntax (`-D`, `-S`, `-A`, `-B`, `-TMDL`, `-O`, `-C`, `-V`, `-G`), run the legacy syntax through this CLI's compat layer (see [Migration from TE2](#migration-from-te2)) or invoke `TabularEditor.exe` directly.

## When to use this skill

- User mentions "te CLI", "the new Tabular Editor CLI", or types a `te <command>` from terminal
- User wants to load/save/deploy/refresh/test a semantic model from terminal on macOS, Linux, or Windows
- User wants to convert TMDL ↔ BIM ↔ PBIP, run BPA, format DAX, query a model, or scaffold tests from CLI
- User is migrating CI/CD pipelines from `TabularEditor.exe` (TE2) to `te`
- User asks about `te connect`, `te auth`, `te profile`, `te session`, `te interactive`, `te macro`, `te vertipaq`

## When NOT to use this skill

- User explicitly wants to invoke `TabularEditor.exe` natively (TE2) instead of using this CLI's compat layer → step out and use that product directly
- User asks about Tabular Editor 3 desktop UI features (Preferences.json, MacroActions.json, Layouts.json) → consult the TE3 desktop docs at https://docs.tabulareditor.com/
- User wants in-depth help authoring a C# script body or BPA rule expression itself (rather than running them via this CLI) → use the TOM API docs / a dedicated DAX or BPA reference

## Critical general rules

- **First time using `te` in a session**: run `te --version` and `te auth status`; if not authenticated, ask user to run `te auth login`
- Always use `te --help` and `te <command> --help` the first time you compose a command — flags are still evolving during preview
- **Never put secrets on the command line**: `te auth login -p <secret>` is visible in `ps`/shell history. Use `--auth env` with `AZURE_CLIENT_ID`/`AZURE_CLIENT_SECRET`/`AZURE_TENANT_ID`, stdin (`-`), or `--auth managed-identity` instead
- **In CI**: always pass `--non-interactive` and `--force` (where applicable). `te deploy` defaults to a confirmation prompt with `n` as the safe default — pipelines hang without `--force`
- **BPA gate is ON by default** for `te deploy` and `te save`. Override deliberately: `--skip-bpa` (one command), `--fix-bpa` (auto-apply BPA fix expressions), or `bpa.onDeploy: false` / `bpa.onSave: false` in `~/.config/te/config.json` (BPA config keys are **nested under `bpa.`** — not flat `bpaOnDeploy`)
- **Mutations stage in memory by default.** `te set`, `te add`, `te rm`, `te mv`, `te replace`, `te format`, `te script`, `te macro run`, `te incremental-refresh set/remove` keep changes **in memory** unless `--save` is passed. The default behavior is controlled by `interactiveEditMode` config (`stage` | `save` | `revert`). Inside `te interactive`, the trio `--save` / `--stage` / `--revert` are mutually exclusive per command
- **Suppress preview banner** in scripted output: `te config set hidePreviewNotice true` (banner reappears 14 days before the 2026-09-30 expiry regardless)
- **Avoid destructive ops without explicit user direction**: `te rm`, `te mv`, `te deploy --create-only` (fails if exists — confirm intent), `te save --force`, `te connect --clear`
- If a command is blocked by your permissions, stop and ask the user — never circumvent

## Staging model (`--save` / `--stage` / `--revert`)

Every mutating command runs through a staging dispatcher:

| Command | Mutating commands it applies to |
|---|---|
| Edits | `te set`, `te add`, `te rm`, `te mv`, `te replace` |
| DAX/M | `te format` |
| TOM | `te script`, `te macro run` |
| Refresh | `te incremental-refresh set`, `te incremental-refresh remove` |
| BPA | `te bpa run --fix` |

By default, edits are **staged in memory** and discarded when the process exits. Pass `--save` to persist to the source. The default behavior is configurable via `te config set interactiveEditMode <mode>`:

- `stage` (default) — keep changes in memory only; needs explicit `--save` to persist
- `save` — auto-persist after each successful mutation
- `revert` — auto-roll-back after each mutation (useful for safe `--dry-run`-style audits)

Inside `te interactive`, all three flags are available per-command and are mutually exclusive:

```bash
te> set Sales/Margin -q expression -i "..." --save     # persist now
te> add Sales/Probe -t Measure -i "..." --stage        # keep in memory (default)
te> rm Sales/Old --revert                              # roll back this command only
te> exit                                               # any staged-but-unsaved edits are discarded
```

Outside `te interactive`, only `--save` is exposed (the process can't ask the user to follow up). With `--save-to <path>`, the mutation is written to a different location without overwriting the source.

`--force` on `te script` / `te save` lets a mutation persist even if it introduces *new* DAX validation errors. The default save gate refuses to write a model that's strictly worse than what was loaded.

## Quickstart

0. **Check install + auth**: `te --version` + `te auth status`
1. **Authenticate**: `te auth login` (browser) or `te auth login -u <app-id> -t <tenant>` with secret on stdin
2. **Scaffold a new model**: `te init ./my-model` (defaults to PowerBI mode, TMDL, compat 1702)
3. **Load and inspect**: `te load ./model` then `te ls` and `te ls Sales`
4. **Set an active connection** (avoids passing `-s/-d` every time): `te connect MyWorkspace MyModel`
5. **Find something**: `te find "Revenue" --in names` or `te find "CALCULATE" --in expressions`
6. **Read a measure's DAX**: `te get Sales/Revenue -q expression`
7. **Run BPA**: `te bpa run --fail-on error --ci github`
8. **Format DAX across the model**: `te format --save`
9. **Query**: `te query -q "EVALUATE TOPN(5, 'Sales')" -s ws -d model`
10. **Save / convert**: `te save -o ./out --serialization tmdl` (or `bim`, `pbip`, `te-folder`, `database.json`)
11. **Deploy**: `te deploy ./model -s ws -d model --force --ci github`
12. **Refresh**: `te refresh --type full` or `te refresh --table Sales --type full`
13. **Diff two models**: `te diff ./v1 ./v2` (exit 1 = differs, 2 = error)
14. **Run TE2 scripts unchanged**: rename `te` → `te2`, OR `TE_COMPAT=te2 te <legacy args>`, OR auto-detected from flags
15. **Inspect what shell session "remembers"**: `te session` (active connection, profile, test suite for this terminal)

## Installation

Download from https://tabulareditor.com (signed in with your TE account). Single self-contained binary — no .NET / runtime install needed.

| Platform | Archive | Install location (suggested) |
|---|---|---|
| Windows x64 / ARM64 | `te-win-x64.zip` / `te-win-arm64.zip` | `%LOCALAPPDATA%\Programs\te` |
| macOS Intel / Apple Silicon | `te-osx-x64.zip` / `te-osx-arm64.zip` | `~/.local/bin` |
| Linux x64 / ARM64 | `te-linux-x64.zip` / `te-linux-arm64.zip` | `~/.local/bin` |

Add the install dir to `PATH`. On macOS, allow first-run network access for Gatekeeper notarization check. Update by overwriting the binary; config and credentials persist.

**Shell completion**:
```bash
te completion bash > /etc/bash_completion.d/te
te completion zsh  > "${fpath[1]}/_te"
te completion pwsh | Out-String | Invoke-Expression
```

**Cross-platform limits**: local SSAS connections (TCP) and Power BI Desktop connections (named pipe) are Windows-only. All cloud workflows work on every platform.

## Authentication

Backed by Azure Identity's full credential chain.

| Method | Flag | When to use |
|---|---|---|
| Interactive browser | `--auth interactive` (default) | Local dev |
| Service principal (secret) | `--auth spn -u <appId> -p <secret> -t <tenant>` | Avoid; secret on cmd line |
| Service principal (cert) | `--auth spn -u <appId> -t <tenant> --certificate <path>` | Cert-based CI |
| Environment vars | `--auth env` (reads `AZURE_CLIENT_ID/SECRET/TENANT_ID`) | **Preferred for CI** |
| Managed identity | `--auth managed-identity` | Azure-hosted runners |

```bash
te auth login                            # browser
te auth login --identity                 # managed identity
te auth status                           # exit 0 if authenticated, 1 otherwise
te auth logout                           # clear cached credentials
```

**Credential cache locations** (all file-mode `0600` / DPAPI on Windows):
- Windows: `%USERPROFILE%\.te-cli\` (DPAPI-encrypted)
- Linux: `~/.te-cli/` (libsecret via Azure.Identity)
- macOS: `~/.te-cli/token-cache.bin`

## Connections and profiles

`te connect` sets a per-terminal active connection so subsequent commands don't need `-s/-d/-m` repeated.

```bash
te connect                               # show active connection (or open picker in interactive)
te connect MyWorkspace MyModel           # remote workspace
te connect ./my-model                    # local TMDL/BIM
te connect --local                       # running Power BI Desktop (Windows)
te connect --clear                       # reset
```

**Workspace mirroring** (bidirectional sync between local TMDL folder ↔ remote workspace):

```bash
te connect Finance "Revenue Model" -w ./revenue-model   # remote primary, mirror to local
te connect ./revenue-model -w Finance "Revenue Model"   # local primary, mirror to remote
# --workspace-format <bim|tmdl|te-folder>  # on-disk format for the mirror
# --workspace-auth <method>                # auth for the remote side when primary is local
```

**Profiles** save named connection + behavior overrides:
```bash
te profile set prod -s MyWorkspace -d MyModel --auto-format true
te profile set dev  -s DevWorkspace -d MyModel --bpa-on-deploy false
te profile list
te profile show prod
te profile remove old
te connect --profile prod
```

## Object path syntax

Backed by a formal grammar (`PathParser`) — paths come in two flavors with subtly different rules:

**Object paths** — used by `te get`, `te set`, `te add`, `te rm`, `te mv`. Resolve to **one** object. Wildcards rejected.

**Filter paths** — used by `te ls`, `te find`, `te deps`, `te bpa run --path`. Resolve to a **set** of objects. Wildcards allowed.

### Slash-form (works on both)

- `Sales` — table
- `Sales/Revenue` — measure or column on the Sales table
- `Sales/Measures`, `Sales/Columns`, `Sales/Partitions`, `Sales/Hierarchies` — sub-containers
- `Sales/Geography/Levels` — hierarchy levels
- `Measures/<name>/KPI` — KPI sub-object on a measure (resolves through the KPI wrapper)
- `Roles/<role>/Members`, `Roles/<role>/TablePermissions` — role children
- `Perspectives/<persp>/<table>` — perspective membership (use `te add Perspectives/Default/Sales` to add a table)
- `Tables`, `Measures`, `Relationships`, `Roles`, `Perspectives`, `Cultures`, `Hierarchies`, `Annotations` — model-level containers (pivot via `te ls Measures` for cross-table view)

Container-keyword table names (a table called `Tables`, `Roles`, etc.) resolve correctly via the path parser — the parser disambiguates by position.

### DAX-form (object paths)

DAX-style quoting and bracket-suffix follow DAX conventions — doubled quote char escapes itself (`'Bob''s'` = `Bob's`, `[foo]]bar]` = `foo]bar`):

- `'Sales'[Amount]` — same as `Sales/Amount`
- `"Net Sales"[Sales Amount]` — same as `"Net Sales"/"Sales Amount"`, double-quoted form
- `[Total Sales]` — **model-wide measure-or-column lookup** (no table prefix; resolver searches every table)
- `"Sales[ProdKey]->Product[ProdKey]"` — relationship shorthand (used by `te add` only)
- `"Sales 2024"/Revenue`, `"_Measures/Total Revenue"` — quote any segment that contains a space, `/`, `[`, or `]`

### Wildcards (filter paths only)

Single `*` matches any run of characters within one segment (case-insensitive). Multi-segment globs and `?` are not supported.

```bash
te ls Sa*                       # tables starting with "Sa"
te ls Sales/*Amount             # any child of Sales ending in "Amount"
te ls */Amount                  # an "Amount" column/measure across every table
te ls Roles/Re*/Members         # members of every role matching Re*
te bpa run --path "Sales/*"     # run BPA only on objects under Sales
```

Passing a wildcard to an object-path command (`te get Sa*`, `te set Sa*`) fails fast with a parser error — wildcards on those would resolve to many objects, and the command needs exactly one.

## Global options

Work with every command:

| Option | Description |
|---|---|
| `-m, --model <path>` | TMDL folder, `.bim`, or TE folder |
| `-s, --server <endpoint>` | Workspace name, `powerbi://...`, `asazure://...`, `localhost:PORT` |
| `-d, --database <name>` | Semantic model name on workspace |
| `--local` | Connect to running Power BI Desktop (Windows only) |
| `--auth <method>` | `auto` \| `interactive` \| `spn` \| `env` \| `managed-identity` |
| `--output-format <fmt>` | `auto` \| `text` \| `json` \| `csv` \| `tmsl` (alias `bim`) \| `tmdl` (default `auto`: text on TTY, JSON when piped). Controls how stdout is rendered; distinct from `--serialization` which picks the on-disk model format |
| `--recent [N]` | Use recently-used model (no value = picker, `N` = Nth most recent) |
| `--non-interactive` | Disable prompts; fail if input missing — **set in CI** |
| `--debug` | Debug logs to stderr |

> [!NOTE]
> `--output-format` (how stdout is rendered) and `--serialization` (how models are written to disk on `init`/`save`/etc.) are **two different flags**. Don't conflate them — passing one when you meant the other gives a confusing error or silent wrong output.

## Command reference (10 families)

### Model I/O

| Command | Purpose | Key flags |
|---|---|---|
| `te load <path>` | Load model and show summary | global `-m/-s/-d` |
| `te save` | Save / convert / persist edits | `-o, --output-path <path>`, `--serialization tmdl\|bim\|te-folder\|pbip\|database.json`, `--force`, `--skip-bpa`, `--fix-bpa`, `--bpa-rules <file>` (repeatable, overrides config), `--skip-validation`, `--supporting-files` |
| `te open <path>` | Open in TE3 Desktop (TE3 must be installed) | — |
| `te init [path]` | Create new empty model. Path is optional — falls back to global `--model` when omitted | `--compatibility-mode PowerBI\|AnalysisServices` (default `PowerBI`), `--compatibility-level <int>` (alias `--compat`; defaults to 1702 for PowerBI, 1500 for AnalysisServices), `--name <model-name>`, `--serialization tmdl\|bim\|te-folder\|pbip` (default `tmdl`), `--force` |

```bash
te load ./model                                                  # local TMDL folder
te load model.bim                                                # local BIM file
te load -s MyWorkspace -d MyModel                                # remote

te save                                                          # write back to source
te save ./model.bim -o ./tmdl-out                                # convert BIM → TMDL
te save -o ./project --serialization pbip --supporting-files
te save -o ./out -s ws -d model --skip-validation                # fast passthrough

te init ./my-model                                               # PowerBI mode, TMDL, compat 1702 (default)
te init ./my-model --compatibility-mode AnalysisServices         # AS mode, compat 1500
te init ./my-model --compatibility-level 1604                    # specific compat level
te init ./my-model --serialization bim                           # single-file .bim model
te init ./my-model --serialization pbip                          # full Power BI project structure
te --model ./new.bim init                                        # path via global --model
```

### Model Editing

| Command | Purpose | Key flags |
|---|---|---|
| `te set <obj>` | Set property | `-q <prop>` (e.g. `expression`, `formatString`, `description`, `isHidden`), `-i <value>` (or `-` for stdin), `--save`, `--save-to <path>` |
| `te add <obj>` | Add object | `-t <type>` (`Table`, `Measure`, `Column`, `CalculatedColumn`, `CalculatedTable`, `Hierarchy`, `Role`, `Perspective`, `Culture`, `CalculationGroup`, `CalculationItem`, `MPartition`, `Partition`, `EntityPartition`, `PolicyRangePartition`, `KPI`, `NamedExpression`, ...), `-i <value>`, `--if-not-exists` (idempotent), `--save`. Data-bound tables: `--mode import\|directquery\|directlake`, `--source sql\|lakehouse\|warehouse`, `--endpoint`, `--source-table`, `--source-database`, `--columns "Col1:Type,Col2:Type,..."`, `--partition-expression "<M>"`, `--source-type m\|query\|calculated` |
| `te rm <obj>` | Remove object | `--force`, `--if-exists`, `--dry-run`, `--save` |
| `te mv <src> <dst>` | Move/rename | `--save` |
| `te replace <find> <repl>` | Find+replace text | `--in names\|expressions\|descriptions\|displayFolders\|formatStrings\|annotations\|all`, `--regex`, `--case-sensitive`, `--save` (dry-run by default) |

```bash
te set Sales/Amount -q expression -i "SUM(Sales[Amt])" --save
te set Sales -q isHidden -i true --save
te add Sales/Revenue -t Measure -i "SUM(Sales[Amount])" --save
te add Sales -t Table --save                                              # empty M partition (PowerBI default)
te add "Sales[ProdKey]->Product[ProdKey]" --save                          # relationship shorthand
te add Sales/MarketingFlag -t CalculatedColumn -i "..." --if-not-exists --save
te rm Sales/OldMeasure --if-exists --save
te rm Sales/Revenue --dry-run                                             # preview impact
te mv Sales/Revenue Finance/Revenue --save                                # cross-table move
te replace "OldTable" "NewTable" --in expressions --save
te replace "SUM" "SUMX" --regex --in expressions --save
```

#### Common `-q` properties

Property names are case-insensitive and match TOM. When in doubt, run `te get <obj>` to see what's already on the object, or check `te set <obj> -q` for the settable list. The most-used ones:

| Object | Common properties |
|---|---|
| **Measure** | `expression` (DAX), `formatString`, `displayFolder`, `description`, `isHidden` |
| **DataColumn** | `dataType` (`int64`/`string`/`double`/`decimal`/`dateTime`/`boolean`), `sourceColumn`, `summarizeBy` (`none`/`sum`/`count`/`average`/`max`/`min`/`distinctCount`/`automatic`), `isKey`, `isHidden`, `formatString`, `sortByColumn`, `dataCategory`, `displayFolder`, `description` |
| **CalculatedColumn** | `expression` (DAX) plus most DataColumn properties |
| **MPartition** | **`MExpression`** (NOT `expression` — that's what `te get` displays, but `te set` rejects it), `Mode` (`Import`/`DirectQuery`/`DirectLake`/`Default`), `description` |
| **QueryPartition** | `QueryDefinition` (alias `Query`), `Mode`, `description` |
| **Table** | `isHidden`, `dataCategory` (use `Time` to mark a date table), `description`, `name` (rename) |
| **Hierarchy** | `displayFolder`, `description`, `isHidden`; Levels take `column` (the source column name) |
| **ModelRole** | `modelPermission` (`None`/`Read`/`ReadRefresh`/`Refresh`/`Administrator`); TablePermissions take `filterExpression` (DAX) |
| **KPI** (on a Measure path `Measures/<name>/KPI`) | `statusExpression`, `trendExpression`, `targetExpression`, `statusGraphic`, `trendGraphic` |
| **CalculationItem** | `expression` (DAX), `ordinal` (int), `formatStringDefinition` |
| **Annotations / Translations** | `Annotations[<key>]`, `TranslatedNames[<culture>]`, `TranslatedDescriptions[<culture>]` — bracket-indexed property names |

Properties not in the list are still usable — these are the ones Claude tends to fumble on or need most often. **`te get <obj>` is always the authoritative discovery tool** for what an existing object exposes.

### Inspection

| Command | Purpose | Key flags |
|---|---|---|
| `te ls [filter-path]` | List objects, FS-style (filter-path: wildcards allowed) | `--type <type>`, `--paths-only`, `--no-multiline` (collapse multi-line cells; text output only) |
| `te get <obj>` | Get properties (object-path: no wildcards) | `-q <prop>` (single property), `--output-format tmdl\|tmsl\|bim` (emit object as TMDL/TMSL) |
| `te find <text>` | Search across model | `--in names\|expressions\|descriptions\|displayFolders\|formatStrings\|annotations\|all`, `--regex`, `--case-sensitive`, `--paths-only`, `--no-multiline`. **`--in expressions` walks every `IExpressionObject`** — measure DAX, calculated columns, KPI status/trend/target expressions, measure detail-rows, partition M, table-permission filters, calculation-group selection expressions |
| `te diff <m1> <m2>` | Structural diff | exit 0 identical, 1 differs, 2 error |
| `te deps [obj]` | Dependency analysis | `--unused` (no DAX refs, not in relationships/hierarchies/sort-by/variations/time roles), `--hidden` (narrow to hidden), `--deep`, `--upstream`, `--downstream`, `--max-depth <N>` |

```bash
te ls                                # tables
te ls Sales                          # columns + measures in Sales
te ls Sales/Measures                 # measures only
te ls Measures                       # all measures across model
te ls --type measure --paths-only    # pipeable
te get Sales/Revenue -q expression
te get Model -q description
te find "CALCULATE" --in expressions                # covers DAX, calc-columns, KPI exprs, partition M, role filters, calc-group selection
te find "Revenue" --in names
te find "TODO" --in descriptions --no-multiline     # single-line cells, easy to grep
te find 123 --in expressions --paths-only           # pipeable, e.g. for finding a KPI TargetExpression value
te diff ./model-v1 ./model-v2
te deps "Sales/Revenue"                             # upstream + downstream
te deps --unused                                    # unused everywhere
te deps --unused --hidden                           # hidden + unused
```

### Analysis & Quality

| Command | Purpose | Key flags |
|---|---|---|
| `te validate` | Expressions + schema + TOM errors | `--ci <fmt>` (see below), `--trx <file>`, `--no-multiline`, `--no-warnings`, `--no-antipatterns`, `--errors-only` |
| `te bpa run [model]` | Run BPA (optional positional model path) | `-r/--rules <file-or-url>` (repeatable; URLs supported), `--fix`, `--save`, `--save-to <path>`, `--serialization`, `--fail-on error\|warning`, `--ci`, `--trx`, `--no-defaults`, `--no-model-rules`, `--rule <id>` (repeatable), `--path <filter>` (wildcards OK: `--path "Sales/*"`), `--vpax <file>`, `--vpa-rules`, `--allow-external-rules` (allow URL rules from model annotations), `--no-multiline` |
| `te bpa rules list` | Inspect active rules | `--all` (incl. disabled+ignored), `--ignored`, `--no-multiline` |
| `te vertipaq [path]` | VertiPaq stats (optional positional object path, e.g. `Sales` or `Sales/Amount`) | `--columns`, `--relationships`, `--partitions`, `--all`, `--detail` (encoding/segments breakdown), `--fields <csv>` (custom column set), `--export <vpax>`, `--import <vpax>` (offline), `--obfuscate` (writes `.vpax.dict` sidecar), `--top <N>`, `--stats` (DAX-queried details), `--annotate`, `--save` |
| `te format` | Format DAX or M | `-e <text>` (inline), `-p <obj>` (single), `--lang dax\|m`, `--semicolons` (Euro), `--long` (more line breaks; default is short), `--no-space-after-function`, `-t/--type <kind>` (disambiguate `-p` when path matches multiple), `--save`, `--save-to <path>` |

```bash
te validate ./model --ci github --trx results.trx
te validate ./model --errors-only                   # hide warnings + anti-patterns
te bpa run --fail-on error --ci github
te bpa run --fix --save
te bpa run --rule PERF_UNUSED_HIDDEN_COLUMN
te bpa rules list --all
te vertipaq --all --export stats.vpax
te vertipaq Sales                                    # filter to one table
te vertipaq Sales/Amount                             # filter to one column
te vertipaq --columns --detail                       # encoding/segment breakdown
te vertipaq --fields name,card,size,%tbl,%db,bar     # custom column set
te vertipaq --import stats.vpax                      # offline analysis from VPAX
te format --save                                     # all DAX
te format -p Sales/Amount --save                     # single measure
te format --lang m --save                            # all M
te format -e "SUM ( Sales[Amount] )"                 # inline preview
```

### Execution

| Command | Purpose | Key flags |
|---|---|---|
| `te query` | DAX query | `-q <dax>` or `-f <file.dax>`, `--limit <N>` (default 100), `-o, --output-file <file>` (extension picks format: `.csv\|.tsv\|.json\|.dax`), `--trace`, `--cold`, `--plan`, `--runs <N>` (benchmark), `--no-validate` |
| `te script` | Run C# script (TOM) | `-S <file>` (repeatable, `.cs`/`.csx`), `-e <code>` (inline, `-` = stdin), `--save`, `--save-to`, `--serialization`, `--dry-run`, `--timeout <s>` |
| `te macro <sub>` | TE3 macros | `list`, `run <name-or-id>` (with `--on <obj-paths>`, `--save`), `add`, `set`, `rm`, `sort` |

```bash
te query -q "EVALUATE TOPN(5, 'Sales')" -s ws -d model
te query -f query.dax --output-format json                       # global --output-format controls stdout format
te query -q "EVALUATE Sales" --output-file results.csv           # writes CSV/TSV/JSON/DAX based on extension
te query -q "EVALUATE Sales" --runs 5 --cold --plan
te script --script fix.cs --save
te script -e "Info(Model.Tables.Count)"
echo "Info(Model.Name);" | te script -e -
te macro list
te macro run "Hide all measures"
te macro run "Format DAX" --on "Sales/Revenue,Sales/Margin" --save
```

### Deployment & Refresh

| Command | Purpose | Key flags |
|---|---|---|
| `te deploy` | Deploy model | `-s/-d`, `--deploy-full` (overwrite + connections + partitions + roles + members + shared exprs), `--deploy-connections`, `--deploy-partitions`, `--skip-refresh-policy`, `--deploy-roles`, `--deploy-role-members`, `--deploy-shared-expressions`, `--create-only` (fail if exists), `--xmla <file>` (TMSL only, `-` for stdout), `--skip-bpa`, `--fix-bpa`, `--bpa-rules <file>` (repeatable), `--force` (**required for CI**), `--ci <fmt>`, `-p, --profile <name>` |
| `te refresh` | Trigger refresh | `--type full\|dataonly\|automatic\|calculate\|clearvalues\|defragment\|add` (default `automatic`), `--table <name>` (repeatable), `--partition <Tbl.Part>` (repeatable), `--apply-refresh-policy true\|false` (default true), `--effective-date yyyy-MM-dd`, `--max-parallelism <N>`, `--dry-run` (emit TMSL), `--no-progress`, `--trace [path]` (no value = stderr; with path = log file) |
| `te incremental-refresh <sub> <table>` | Manage IR policies | `show`, `set`, `remove`, `apply` (re-evaluate policy and create/expand partitions) |

```bash
te deploy ./model -s ws -d model --force --ci github
te deploy ./model --xmla script.tmsl                # generate TMSL only
te deploy ./model --xmla -                          # TMSL to stdout
te deploy ./model --profile staging --force
te refresh --type full
te refresh --table Sales --partition "Sales.2024" --type full
te refresh --type full --dry-run > refresh.tmsl
te refresh --type full --trace                      # XMLA trace events to stderr
te refresh --type full --trace refresh.log          # XMLA trace events to log file
te incremental-refresh show Sales
te incremental-refresh apply Sales                  # re-evaluate policy, create/expand partitions
```

### Testing

| Command | Purpose | Key flags |
|---|---|---|
| `te test run` | Run DAX assertion tests | `--suite <path>` (default `.te-tests/`), `--tag <tag>`, `--fail-on error\|warning`, `--ci`, `--trx <file>` |
| `te test init` | Scaffold suite | `--example`, `--from-model --model <path>` |
| `te test spec` | Print assertion format | — |
| `te test use <suite>` | Activate suite (session-scoped) | — |
| `te test list` | List test cases | — |
| `te test snapshot` | Capture model snapshot | — |
| `te test compare` | Compare snapshots | — |

```bash
te test init --example
te test init --from-model --model ./my-model        # generate stubs from model
te test run --ci github --trx results.trx
te test run --tag revenue
te test snapshot
te test compare
```

### Connection & Auth

(Covered above under [Authentication](#authentication) and [Connections and profiles](#connections-and-profiles).) Full subcommands:

```
te connect [<server> <database>] [--local | -w/--workspace <path-or-server-db> | --workspace-format bim|tmdl|te-folder | --workspace-auth <method> | --force | -p/--profile <name> | --clear]
te auth login [-u <appId>] [-p <secret>|-] [-t <tenant>] [--identity|-I] [--certificate <path>] [--certificate-password <pw>] [--save] [--auth interactive|spn|env|managed-identity]
te auth status
te auth logout
te profile {set|show|list|remove} <name> [...]
te session [show | list | clear | prune [--all] [--dry-run]]
```

#### Sessions

Every shell process gets its own session file under `~/.config/te/sessions/<id>.json`, holding the active connection, active profile, active test suite, and timestamps. Default session ID is derived from the parent shell PID; set `TE_SESSION=<name>` to name a session and share it across multiple shells or scripts. Sessions for dead PIDs are auto-cleaned on each invocation; `te session prune` lets you trigger cleanup manually (or with `--all`, drop every session except the current one).

```bash
te session                              # show current session (id, file, active state)
te session list                         # all session files on this machine
te session clear                        # reset active connection / profile / test suite for this shell
te session prune                        # delete sessions whose shell process is dead
te session prune --dry-run              # preview what would be deleted
te session prune --all                  # delete every session except current (incl. named TE_SESSION ones)
TE_SESSION=ci-deploy te connect ws md   # share session under name "ci-deploy"
```

Why it matters: `te connect`, `te test use`, and `--profile` all mutate the session file, not the global config. Two terminals can hold different active connections without stepping on each other.

### Configuration

| Command | Purpose |
|---|---|
| `te config show [--output-format json]` | Show all settings |
| `te config paths` | Resolved file paths (macros, BPA rules, config) |
| `te config init [--force]` | Create default config |
| `te config set <key> <value>` | Update setting |
| `te license …` | **Hidden during preview.** Subcommands (`activate`, `status`, `deactivate`) are parseable so existing scripts don't fail at parse time, but any invocation prints *"`te license` is not available in this preview build"* and exits 1. Don't pipeline this. |
| `te migrate [-A] [--output-format text\|json]` | TE2 → new-CLI flag mapping (interactive lookup or full table) |

**Config file**: `~/.config/te/config.json` (Windows: `%USERPROFILE%\.config\te\config.json`). Resolution order: `$TE_CONFIG` → default path → built-in defaults.

**Configurable keys** (the keys accepted by `te config set`):

| Key | Type | Default | Purpose |
|---|---|---|---|
| `macros` | path | _(none)_ | Override path to a `MacroActions.json` file |
| `queryLog` | path | _(none)_ | Path to the DAX query log file |
| `te3ExePath` | path | _(none)_ | Override path to the TE3 desktop executable (for `te open`) |
| `autoFormat` | bool | `false` | Apply DAX Formatter after mutations |
| `validateOnMutation` | bool | `true` | Verify `Table[Column]` references after edits |
| `vertipaqOnRefresh` | bool | `false` | Capture VertiPaq stats post-refresh |
| `bpa.rules` | string[] | _(none)_ | Path(s)/URL(s) to BPA rule file(s) — repeatable; comma-separated on `te config set` |
| `bpa.onMutation` | bool | `false` | Run BPA after every mutation |
| `bpa.onDeploy` | bool | `true` | **BPA gate before deploy** (bypass: `--skip-bpa`) |
| `bpa.onSave` | bool | `true` | **BPA gate before save** (bypass: `--skip-bpa`) |
| `bpa.builtInRules` | bool | `true` | Include built-in default rules in scans |
| `bpa.disabledBuiltInRuleIds` | string[] | _(none)_ | Suppress specific built-in rule IDs |
| `formatOptions.useSemicolons` | bool | `false` | Use Euro separator (`;`) in DAX output |
| `formatOptions.shortFormat` | bool | `true` | Compact DAX layout (vs `--long`) |
| `formatOptions.skipSpaceAfterFunction` | bool | `false` | `SUM(x)` instead of `SUM (x)` |
| `formatOptions.useSqlBiDaxFormatter` | bool | `false` | Use SQLBI's online formatter instead of the in-house one |
| `interactiveEditMode` | enum | `stage` | Default for mutating commands: `stage` (in-memory only), `save` (auto-persist), `revert` (auto-roll-back). Overridden per-command by `--save`/`--stage`/`--revert` |
| `hidePreviewNotice` | bool | `false` | Suppress yellow preview banner |
| `spinner` | bool | `true` | Animated progress (disable for CI) |
| `debug` | bool | `false` | Debug logs to stderr |
| `disableTelemetry` | bool | `false` | Opt out of anonymous usage telemetry |

> [!NOTE]
> BPA keys are **nested under `bpa.`** — `te config set bpa.onDeploy false`, not `bpaOnDeploy`. Same for `formatOptions.*`.
>
> Active connection / profile / test-suite are **session-scoped** (see `te session`) and explicitly rejected by `te config set` — use `te connect`, `te profile`, `te test use` instead.

**Speed knobs for batch / demo / CI runs**: each `te` invocation has ~1-2 s of process startup + model load. For pipelines that issue many sequential `te` calls (build scripts, live demos, mass-edit loops), set these once before the run:

```bash
te config set bpa.onSave false       # skip the BPA gate on every --save; run BPA once at the end instead
te config set spinner false          # disable the animated progress widget (cleaner CI logs, slightly faster)
te config set hidePreviewNotice true # suppress the yellow preview banner
```

`bpa.onSave: false` is by far the biggest win — without it, BPA runs on every saved mutation, which on a typical model-build script means dozens of redundant passes.

**Project-local BPA gate**: drop a `.te-bpa.json` in repo root (or set via `TE_BPA_CONFIG`) to override gate behavior per project.

### Shell

| Command | Purpose |
|---|---|
| `te interactive [model]` | Model-aware REPL — prompt is `te [MyModel]>` or `te>`. All subcommands work without `te` prefix. Built-ins: `help`/`?`, `status`/`pwd`, `clear`/`cls`, `exit`/`quit`/`q` |
| `te completion <shell>` | Print completion script (`bash`, `zsh`, `pwsh`) |

The REPL's argv splitter is bracket-aware, so DAX-style refs work without escaping the brackets — handy for paste-from-DAX-editor workflows:

```bash
te interactive
te interactive ./model
te interactive -s MyWorkspace -d MyModel
te> ls Sales
te> ls Sa*                              # wildcard filter-paths
te> get "Sales/Revenue" -q expression
te> get [Total Sales]                   # lone-bracket: model-wide measure/column lookup
te> get 'Sales'[Amount]                 # DAX-quoted form
te> ls Roles/Reader/Members             # role members
te> add Perspectives/Default/Sales      # add Sales table to the Default perspective
te> bpa run --fail-on error
te> exit
```

## Common workflows

### Build a new table with an M partition

`te add <Table> -t Table` on a model with no provider data source creates an **MPartition** by default. `--partition-expression "<M>"` delivers the M to that partition. Combine with `--columns` for a one-shot data-bound table:

```bash
te add Sales -t Table \
  --columns "OrderID:Int64,Amount:Decimal,OrderDate:DateTime" \
  --partition-expression "$(cat <<'EOF'
let
    Source = Sql.Database("server", "db"),
    Sales = Source{[Schema="dbo", Item="Sales"]}[Data]
in
    Sales
EOF
)" -m ./model --save
```

Result: one table, one MPartition with the M in place, columns typed. `te get Sales/Partitions/Sales -q sourceType` returns `M`.

**Variants:**

```bash
# Columns + placeholder M (filled later via te set <Table>/Partitions/<Name> -q MExpression …)
te add Sales -t Table --columns "Id:Int64,Amount:Decimal" -m ./model --save

# M only, no explicit column schema (columns auto-discovered at refresh from the M output)
te add Sales -t Table --partition-expression "<M>" -m ./model --save

# Force M explicitly when the model has a legacy provider DS that would otherwise win
te add Sales -t Table --source-type m --columns "..." --partition-expression "<M>" -m ./model --save

# Opt into a legacy Query partition on a modern model
te add LegacyT -t Table --source-type query -q query -i "SELECT * FROM dbo.Foo" -m ./model --save
```

**Inline-data partitions** (demo models, placeholder tables, calc-group hosts): use `#table({...}, {{...}})` inside a single-quoted heredoc. Example for a `_Measures` placeholder table:

```bash
te add _Measures -t Table --columns "_Measures:String" \
  --partition-expression 'let Source = #table({"_Measures"}, {{""}}) in Source' \
  -m ./model --save
```

**Heuristic for `-q expression -i "<value>"`** (when neither `--source-type` nor `--partition-expression` is set): the M.Analyzer lexer tokenises the value and reports M if the first token is `(`, `[`, the `let` keyword, or an identifier starting with `#` (`#table`, `#date`, `#shared`, ...). Everything else (including bare identifiers and SQL-shaped strings) reports as Query, with a stderr hint pointing at `--source-type m` if the user meant M. Leading comments are skipped.

**Pre-validation errors** (fail before mutation):
- `--source-type calculated` paired with `-t Table` → use `-t CalculatedTable`
- `--source-type m` on a model with a provider data source → remove the DS, or use `--source-type query`
- `--source-type m` on Compatibility Level < 1400 → upgrade the model
- `--source-type` combined with `--mode directlake` → DL/Entity partitions are picked automatically

**Updating an existing partition's M** after creation: `te set Sales/Partitions/Sales -q MExpression -i "<M>" --save` (note `MExpression`, not `expression`, despite `te get` displaying the property as `expression`).

### Convert TMDL ↔ BIM ↔ PBIP

```bash
te save ./model.bim -o ./tmdl-out                                       # BIM → TMDL folder
te save ./tmdl-folder -o ./model.bim --serialization bim                # TMDL → BIM
te save ./model.bim -o ./project --serialization pbip --supporting-files # BIM → PBIP (.platform / definition.pbism)
```

### Deploy from local TMDL with BPA gate + CI annotations

```bash
te deploy ./model \
  -s "powerbi://api.powerbi.com/v1.0/myorg/MyWorkspace" -d "MySemanticModel" \
  --force --ci github                  # BPA gate runs by default; pipeline fails on violations
```

For CI without strict BPA gating: add `--skip-bpa` (one-shot) or `te config set bpa.onDeploy false`. To auto-fix instead of failing: `--fix-bpa`.

### Generate TMSL/XMLA without deploying

```bash
te deploy ./model -s ws -d model --xmla deploy.tmsl
te deploy ./model -s ws -d model --xmla - > deploy.tmsl                 # to stdout
```

### Refresh single partition with dry-run safety

```bash
te refresh --table Sales --partition "Sales.2024" --type full --dry-run > refresh.tmsl
# Review TMSL, then drop --dry-run to execute
```

### Find unused measures and remove with preview

```bash
te deps --unused --hidden                                               # discover candidates
te rm Sales/UnusedMeasure --dry-run                                     # confirm impact
te rm Sales/UnusedMeasure --if-exists --save                            # idempotent removal
```

### Mirror remote workspace for local editing

```bash
te connect MyWorkspace MyModel -w ./local-mirror                        # remote → local TMDL
# Edit in your IDE / push commits / experiment with `te set`, `te add`, etc.
te save                                                                 # writes to both source (remote) and mirror (local)
```

### Bulk DAX format and BPA fix as one batch

```bash
te format --save && te bpa run --fix --save
```

### Run a TE3 C# script against a remote model

```bash
te script -S ./scripts/format-all-dax.csx -s ws -d model --save
echo "foreach (var t in Model.Tables) t.Name = t.Name.Replace(\"_\", \" \");" | te script -e - --save
```

### Snapshot + compare for regression testing

```bash
te test snapshot                                                        # capture baseline
# … make changes …
te test compare                                                         # detect drift
```

## Migration from TE2

Activate TE2 compatibility three ways:

```bash
mv te te2 && ./te2 Model.bim -S fix.csx -D server db -O          # 1. binary rename
TE_COMPAT=te2 te Model.bim -S fix.csx -D server db -O            # 2. env var
te Model.bim -S fix.csx -D server db -O                          # 3. auto-detect from flags
```

For the full mapping table (and an interactive single-flag lookup), run:

```bash
te migrate                                # full table
te migrate -A                             # prompt for a TE2 flag, get equivalent
te migrate --output-format json           # machine-readable for codemods
```

**Most-used mappings**:

| TE2 flag | New CLI |
|---|---|
| `<file>` (positional) | `te <command> <path>` or `--model <path>` |
| `-S <file.csx>` / `-S "code"` | `te script -S <file>` / `-e "code"` |
| `-A <rules>` / `-AX <rules>` | `te bpa run --rules <rules>` (`-AX` = no model rules, which is default) |
| `-D <server> <db>` | `te deploy <model> -s <server> -d <db>` |
| `-O` | (default — overwrite) |
| `-C` | `--deploy-connections` |
| `-P` | `--deploy-partitions` |
| `-R` / `-M` / `-SHARED` | `--deploy-roles` / `--deploy-role-members` / `--deploy-shared-expressions` |
| `-FULL` | `--deploy-full` |
| `-X <file>` | `--xmla <file>` (use `-` for stdout) |
| `-V` / `-G` | `--ci vsts` (also `azdo`/`azure-devops`) / `--ci github` (also `gh`) — on `validate`, `bpa run`, `deploy`, `script`, `test run` |
| `-T <file>` | `--trx <file>` |
| `-B <file>` | `te save -o <file> --serialization bim` |
| `-TMDL <dir>` | `te save -o <dir> --serialization tmdl` (default format) |
| `-F <dir>` | `te save -o <dir> --serialization te-folder` (or `--deploy-full` after `-D`) |
| `-Y` | `--deploy-partitions --skip-refresh-policy` |
| `-W` / `-E` | (default) |
| `-L <user> <pass>` (after `-D`) | `te auth login -u <id> -p <secret> -t <tenant>` (prefer env vars) |
| `-SC` | _Not yet implemented_ |

**Behavioral differences from TE2**:
- `te deploy` runs BPA as pre-flight gate by default (TE2 didn't). `--skip-bpa` to disable, `--fix-bpa` to auto-fix.
- `te deploy` prompts for confirmation. CI must pass `--force`.
- All commands support `--output-format json` for machine-readable output.
- No `start /wait` wrapper needed on Windows — it's a normal console binary.

## CI/CD integration

### GitHub Actions

```yaml
- name: Validate model
  env:
    AZURE_CLIENT_ID:     ${{ secrets.AZURE_CLIENT_ID }}
    AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
    AZURE_TENANT_ID:     ${{ secrets.AZURE_TENANT_ID }}
  run: |
    te validate ./model --ci github --trx validate.trx --non-interactive

- name: BPA gate
  run: te bpa run --rules ./rules/BPARules.json --fail-on error --ci github --non-interactive

- name: Deploy
  run: |
    te deploy ./model \
      -s "${{ vars.WORKSPACE }}" \
      -d "${{ vars.SEMANTIC_MODEL }}" \
      --auth env --force --ci github --non-interactive

- name: Run tests
  run: te test run --ci github --trx test.trx --non-interactive

- name: Publish TRX
  if: always()
  uses: dorny/test-reporter@v1
  with:
    name: TE tests
    path: '*.trx'
    reporter: dotnet-trx
```

### Azure DevOps Pipelines

Same commands, swap `--ci github` for `--ci azdo` (or `vsts`/`azure-devops` — all aliases). Pipeline annotations come back as native `##vso[...]` markers; `--trx` integrates with the `PublishTestResults@2` task.

### Patterns

- **Always pass** `--non-interactive` and `--auth env` (with `AZURE_CLIENT_*` env vars) and `--force` (on `te deploy`)
- **Stable annotations**: `--ci azdo` or `--ci github` on `validate`, `bpa run`, `deploy`, `test run`, `script`
- **Test publishing**: `--trx <file>` on `validate`, `bpa run`, `test run` for VSTEST-compatible XML
- **Promotion (dev → test → prod)**: build once, deploy with `--profile dev`, `--profile test`, `--profile prod` against the same TMDL artifact
- **Disable spinner in CI**: `te config set spinner false` in setup step

## Output formats and exit codes

**`--output-format`** (global stdout format):
- `auto` (default): text on TTY, JSON when stdout is piped/redirected
- `text`: forces human-readable
- `json`: always valid JSON to stdout; errors/warnings to stderr (won't contaminate)
- `csv`: tabular results (only `query`, `bpa run`, `vertipaq`)
- `tmsl` (alias `bim`): emit the resolved object(s) as TMSL/BIM JSON — supported on `te get` and `te ls`
- `tmdl`: emit the resolved object as TMDL — supported on `te get` (single named object only) and `te ls`

```bash
te get Sales --output-format tmdl           # Sales table as TMDL
te get "Sales/Revenue" --output-format bim  # Single measure as TMSL fragment
te ls Tables --output-format bim            # All tables as TMSL/BIM
te ls Measures --output-format tmdl         # Every measure across the model, in TMDL
```

**`--ci` formats** (orthogonal to `--output-format` — emits CI-system logging commands to stderr on `validate`, `bpa run`, `deploy`, `test run`, `script`):

| Value | Effect |
|---|---|
| `vsts`, `azdo`, `azure-devops` | Azure DevOps: `##vso[task.logissue type=error/warning;...]message` + `##vso[task.complete result=...]` summary |
| `github`, `gh` | GitHub Actions: `::error file=…,line=…::message` / `::warning::message` |
| anything else | No CI output |

Errors and warnings are accumulated, so a non-zero exit code reflects total error count for the run.

**Exit codes**:
- `0` — success
- `1` — generic failure: invalid args, validation errors, auth failure, BPA gate
- `2` — `te diff` only: models differ

```bash
# JSON-safe pipeline
te ls --type measure --output-format json | jq -r '.[].path'

# Bash conditional on diff
if te diff old.bim new.bim --output-format json > /dev/null; then
  echo "Identical"
elif [ $? -eq 2 ]; then
  echo "Models differ"
fi
```

## Environment variables

| Var | Purpose |
|---|---|
| `TE_CONFIG` | Override config file path (otherwise `~/.config/te/config.json`) |
| `TE_DEBUG` | Set `1` or `true` for debug logging to stderr |
| `TE_COMPAT` | Set `te2` to force legacy compat mode |
| `TE_SESSION` | Name the current session (instead of parent-PID-derived ID). Lets multiple shells share active state; named sessions are never auto-cleaned |
| `TE_MACROS_PATH` | Override path to a `MacroActions.json` (highest priority for `te macro`) |
| `TE_BPA_RULES` | Override path to a BPA rules file (precedence: explicit `--rules` > `TE_BPA_RULES` > `bpa.rules` config > CWD `BPARules.json`) |
| `TE_BPA_CONFIG` | Override path to a `.te-bpa.json` gate-config (for deploy/save BPA gating) |
| `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID` | SPN credentials (used with `--auth env`) |

## Gotchas

### Path & property-name asymmetries

- **MPartition path asymmetry**: `te add` for an MPartition uses `<Table>/<PartitionName>` (no `/Partitions/` segment). Every other partition command — `te rm`, `te get`, `te ls`, `te mv`, `te set` — uses `<Table>/Partitions/<PartitionName>`. Mixing these up errors with "Cannot add a MPartition at path … Check that -t matches the path shape."
- **Partition M property: `MExpression` for `te set`, `expression` for `te get`**: `te get <Table>/Partitions/<P>` displays the M as `expression`, but `te set -q expression …` errors with "Property 'expression' not found on MPartition. Did you mean: MExpression?" — use `-q MExpression -i "<M>" --save` to update.
- **`te mv` cannot rename a partition to its parent table's name**: `te mv <Table>/Partitions/<Table>_m <Table>/Partitions/<Table>` errors with either "Destination '<Table>/<Table>' already exists" or "Partition '<Table>' not found in table '<Table>'" — the destination path `<Table>/<Table>` resolves to the **table object itself**, not a partition slot inside the table. Passing `-t Partition` does **not** disambiguate. Workaround: rename via the `Name` property — `te set <Table>/Partitions/<Table>_m -q Name -i <Table> --save`.
- **Object paths use `/` as separator**, not `\` or `.`. Quote paths with spaces: `"Sales 2024"/Revenue`, `"_Measures/Total Revenue"`, `"Date/Date Hierarchy/Year"`.

### Object types and creation

- **`-t DataColumn` is not accepted by `te add`**: data columns are declared at table creation via `--columns "Name:Type,..."` (auto-creates with `SourceColumn = Name`). Tune additional properties (`IsKey`, `IsHidden`, `FormatString`, `SummarizeBy`, `SortByColumn`, `Description`) afterwards with `te set`. Only `CalculatedColumn` can be added individually with `-t CalculatedColumn -i "<DAX>"`.
- **Workflow ordering — relationships before measures**: measures that use `RELATED()` or cross-table `CALCULATE()` are validated at save time. If the relationship doesn't exist yet, the save gate rejects with `DAX0002: Column '<T>'[<C>] doesn't have a relationship to any table available in the current context`. Add relationships before authoring dependent measures (or pass `--force` to defer the check, then fix up).
- **`te rm` on the last partition fails** with "Cannot remove last partition from table". Always add a replacement partition first, then remove the original.
- **Silent M syntax errors during partition authoring**: a typo in the partition M (unbalanced braces, an unquoted token) doesn't always raise at save time. After creating a data-bound table, sanity-check with `te get <Table>/Partitions/<Table>` to confirm `sourceType: M` and that the expression matches what you passed in.

### Output shapes

- **`te bpa run --output-format json`**: top-level `violations` is the **count** (integer), top-level `results` is the **list** of violation objects with `ruleId`, `severityLabel`, `objectName`, `canFix`, etc. Don't confuse the two when piping into `jq`.
- **`te bpa run --fix [--save] --output-format json` emits TWO concatenated JSON documents** to stdout, not one: first the scan result (`{model, rulesEvaluated, violations, results, errors, ...}`), then a fix summary (`{fixed, fixErrors, skipped, fixedItems, fixErrorItems}`). Strict single-document parsers fail with an "extra data" / "trailing tokens" error on the second document. Prefer in this order:
  1. **`jq --slurp`** (shell-native, no interpreter dependency): `te bpa run --fix --save --output-format json | jq -s '.[0].violations'` (scan count) or `jq -s '.[1].fixed'` (fix count). `--slurp` reads concatenated documents into an array.
  2. **Drop `--output-format json` entirely** and rely on the human-readable text for one-shot summaries — simplest and works without any extra tooling. Pair with `--ci github`/`--ci azdo` when you need machine-actionable annotations.
  3. **Tail-grep for the summary lines** (`grep -E '"violations":|"fixed":' | head -2`) when you only need the counts.

### Behavior

- **`te ls` is filesystem-style, not workspace-style**. `te ls Sales` lists Sales' children (columns + measures), not "find Sales". Use `te find` for full-text search.
- **`--save` is opt-in for editing commands** (when `interactiveEditMode` is the default `stage`). Without it, `te set`, `te add`, `te rm`, `te mv`, `te replace`, `te format`, `te script`, `te macro run` operate in memory only and don't persist. See [Staging model](#staging-model---save----stage----revert) for the full picture and the `save` / `revert` alternatives.
- **`te validate` does not exercise partition M** — it checks structural/DAX validity, not whether `Table.FromRows` literals parse or SQL endpoints respond. A model with broken partitions still passes `te validate` cleanly. Verify partitions explicitly after table creation.
- **`te connect` is session-scoped (per shell PID)**. Each fresh shell (each Claude Code `Bash` tool call spawns one) starts a new session without the active connection. Either set `TE_SESSION=<name>` to share state between shells, or pass `-m <model>` (and `-s`/`-d` where needed) explicitly to every command.
- **BPA config keys are nested under `bpa.`**. `te config set bpaOnDeploy false` will fail with "Unknown key" — the correct form is `te config set bpa.onDeploy false`. Same for `bpa.onSave`, `bpa.onMutation`, `bpa.rules`, `bpa.builtInRules`, `bpa.disabledBuiltInRuleIds`.
- **`--output-format` (stdout) vs `--serialization` (on-disk)** are different flags. The first picks how stdout is rendered (text/json/csv/tmsl/tmdl); the second picks the model file format (tmdl/bim/te-folder/pbip/database.json). Passing one when you meant the other gives a confusing error or silent wrong output.
- **`te deploy --create-only` fails if model exists**. Use without `--create-only` to overwrite (the default), or check with a probe call first.
- **`te deploy` confirmation prompt hangs CI** without `--force`. Default answer is `n` (safe), so non-interactive runs need `--force --non-interactive`.
- **BPA gate on deploy/save can mask sloppy commits**. If you bypass with `--skip-bpa`, log it loudly. Prefer `--fix-bpa` or address violations.
- **TE2 compat mode auto-detects** when args contain TE2-style flags but no `te` subcommand. Don't rely on this in scripts — be explicit with `TE_COMPAT=te2` so behavior is reproducible.
- **Local SSAS / Power BI Desktop are Windows-only**: `te connect --local` and `te connect "localhost:PORT"` won't work on macOS/Linux even though the binary runs there.
- **Preview banner reappears 14 days before the 2026-09-30 cutoff** regardless of `hidePreviewNotice`. Plan for the hard expiry.
- **Secrets on the cmd line leak** into `ps`, shell history, CI logs. Use `--auth env` with env vars, stdin (`-`), or `--auth managed-identity`.

## References

**Authoritative docs**:

- Overview: https://docs.tabulareditor.com/en/features/te-cli/te-cli.html
- Installation: https://docs.tabulareditor.com/en/features/te-cli/te-cli-install.html
- Authentication: https://docs.tabulareditor.com/en/features/te-cli/te-cli-auth.html
- **Command reference**: https://docs.tabulareditor.com/en/features/te-cli/te-cli-commands.html
- Configuration: https://docs.tabulareditor.com/en/features/te-cli/te-cli-config.html
- Interactive mode: https://docs.tabulareditor.com/en/features/te-cli/te-cli-interactive.html
- Automation: https://docs.tabulareditor.com/en/features/te-cli/te-cli-automation.html
- CI/CD: https://docs.tabulareditor.com/en/features/te-cli/te-cli-cicd.html
- Migration from TE2: https://docs.tabulareditor.com/en/features/te-cli/te-cli-migrate.html
- Known limitations: https://docs.tabulareditor.com/en/features/te-cli/te-cli-limitations.html
- GitHub repo (issues, releases): https://github.com/TabularEditor/CLI

**Related references** (external — not bundled with this skill):

- **Tabular Editor 3 desktop**: https://docs.tabulareditor.com/ (UI, Preferences.json, MacroActions.json, Layouts.json)
- **Tabular Editor 2 (legacy)**: https://docs.tabulareditor.com/te2/ (the older Windows-only `TabularEditor.exe`)
- **TOM API**: https://learn.microsoft.com/analysis-services/tom/introduction-to-the-tabular-object-model-tom-in-analysis-services-amo (for authoring C# scripts run by `te script`)
- **BPA rules collection**: https://github.com/microsoft/Analysis-Services/tree/master/BestPracticeRules (community-maintained rule library)

**External references**:

- DAX: https://dax.guide
- Power Query: https://powerquery.guide
- Power BI XMLA endpoints: https://learn.microsoft.com/en-us/power-bi/enterprise/service-premium-connect-tools

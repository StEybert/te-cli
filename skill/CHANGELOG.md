# Changelog

All notable changes to the `te-cli` skill are documented in this file.

## [0.1.0] - 2026-06-04

### Added

Initial public preview release of the `te-cli` skill. Coverage:

- All `te` commands across all families: Model I/O (`load`, `save`, `init`, `open`), Editing (`set`, `add`, `rm`, `mv`, `replace`), Inspection (`ls`, `get`, `find`, `diff`, `deps`), Analysis & Quality (`validate`, `bpa`, `vertipaq`, `format`), Execution (`query`, `script`, `macro`), Deployment & Refresh (`deploy`, `refresh`, `incremental-refresh`), Testing (`test`), Connection & Auth (`connect`, `auth`, `profile`, `session`), Configuration (`config`, `license`, `migrate`), Shell (`interactive`, `completion`).
- Authentication patterns: interactive browser, service principal (secret + certificate), env-var, managed identity.
- Object path grammar: slash-form, DAX-form (quoted + bracket-suffix), wildcard filter paths.
- Staging model (`--save` / `--stage` / `--revert`) and the `interactiveEditMode` config.
- Common workflows: data-bound table creation with one-shot `--columns` + `--partition-expression`, TMDL/BIM/PBIP conversion, deploy with BPA gate, dry-run refresh, find-and-remove-unused, workspace mirroring.
- TE2 migration mapping table and `TE_COMPAT=te2` compat layer.
- CI/CD integration patterns for GitHub Actions and Azure DevOps Pipelines.
- Output formats (text / JSON / CSV / TMSL / TMDL) and CI annotation formats (vsts/azdo/azure-devops, github/gh).
- Environment variables (`TE_CONFIG`, `TE_DEBUG`, `TE_COMPAT`, `TE_SESSION`, `TE_MACROS_PATH`, `TE_BPA_RULES`, `TE_BPA_CONFIG`, `AZURE_CLIENT_*`).
- Configurable keys via `te config set` (BPA gates, format options, interactive edit mode, telemetry, etc.).
- Speed knobs for batch / demo / CI runs.
- Common `-q` properties cheatsheet for the most-used TOM property names per object type.
- Gotchas covering partition path asymmetries, `MExpression` vs `expression`, BPA JSON output shape, `--output-format` vs `--serialization` confusion, `te connect` session non-persistence, and more.

[0.1.0]: https://github.com/TabularEditor/CLI/releases/tag/skill-v0.1.0

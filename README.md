# Tabular Editor CLI (te)

Public home for **bug reports, feature requests, and discussions** about the [Tabular Editor CLI](https://docs.tabulareditor.com/features/te-cli/te-cli.html) (`te`) — a cross-platform command-line tool for Power BI and Analysis Services semantic models.

> [!IMPORTANT]
> The Tabular Editor CLI is in **Limited Public Preview**. It is offered for evaluation with a Tabular Editor account; no license is required during preview. Commands, flags, and outputs may change before general availability. **The preview build stops functioning after 2026-09-30.** We recommend against using the CLI in production CI/CD pipelines during preview.

## What this repository is

- **[Issues](../../issues)** — report bugs, request features, and track known problems.
- **[Discussions](../../discussions)** — ask questions, share feedback, and swap usage tips with other early adopters.

This repository does **not** host the CLI source code. It exists to give the community a public place to reach us during the preview.

## Before you open an issue

Please take a minute to do these two things — it saves everyone time and helps your report get triaged faster.

1. **Make sure you're on the latest version.** Run `te --version` and compare it with the latest preview release on [tabulareditor.com](https://tabulareditor.com). Many reports turn out to be bugs that have already been fixed. Update and try to reproduce before reporting.
2. **Search existing issues and discussions** for anything similar to what you're seeing. If you find a match, please add a comment (with your repro, version, and environment) to the existing thread rather than opening a duplicate. This keeps conversation in one place and bumps priority based on reaction counts.

If you can't find a match, open a new issue and include:

- `te --version`
- OS and architecture (e.g., Windows 11 x64, macOS 14 ARM64, Ubuntu 22.04 x64)
- The exact command you ran and the output (redact any secrets first)
- What you expected to happen vs. what actually happened

## Documentation

Full documentation lives at <https://docs.tabulareditor.com/features/te-cli/te-cli.html>.

Quick links:

- [Installation and Setup](https://docs.tabulareditor.com/features/te-cli/te-cli-install.html)
- [Authentication and Connections](https://docs.tabulareditor.com/features/te-cli/te-cli-auth.html)
- [Command Reference](https://docs.tabulareditor.com/features/te-cli/te-cli-commands.html)
- [CI/CD Integration](https://docs.tabulareditor.com/features/te-cli/te-cli-cicd.html)
- [Migrating from the TE2 CLI](https://docs.tabulareditor.com/features/te-cli/te-cli-migrate.html)

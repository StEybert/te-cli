# Tabular Editor CLI - AI Agent Skill

A drop-in skill that teaches AI coding agents how to use the [Tabular Editor CLI (`te`)](https://github.com/TabularEditor/CLI) - a cross-platform single binary for managing Power BI and Analysis Services semantic models from the terminal on macOS, Linux, and Windows.

> [!IMPORTANT]
> **Preview.** The Tabular Editor CLI itself is in Limited Public Preview, with preview builds stopping after 2026-09-30. This skill tracks the preview surface and will evolve alongside the CLI.

## What this is

A single Markdown file ([`SKILL.md`](./SKILL.md)) packed with the conventions, command reference, common workflows, gotchas, and CI/CD patterns Claude, GitHub CoPilot (and other AI agents) need to drive `te` productively. Once installed, the agent answers "how do I deploy this model?" or "add a measure that calculates margin" with idiomatic `te` invocations instead of guessing or hallucinating flags.

The skill covers:

- All `te` commands across all families (load, save, init, deploy, refresh, bpa, validate, query, script, format, etc.)
- Authentication patterns (interactive, SPN with secret or certificate, env-var, managed identity)
- Object path grammar (slash form, DAX form, wildcards)
- Staging model (`--save` / `--stage` / `--revert`)
- TE2 -> new CLI migration mapping
- CI/CD recipes for GitHub Actions and Azure DevOps
- Output formats, exit codes, environment variables, config keys
- Common `-q` properties cheatsheet
- The gotchas that trip up agents in practice

## What's a "skill"?

In the Anthropic ecosystem (Claude Code / Claude.ai / Claude Desktop), a **skill** is a Markdown file that an agent loads on demand based on the user's prompt. The skill's YAML frontmatter (`name`, `description`, `version`) tells the agent **when** to load it and **what** it covers. The plain-Markdown body teaches the agent **how** to do its job.

Other agents (GitHub Copilot, Cursor, Aider, generic `AGENTS.md` consumers) don't have an identical "skill" concept but can still benefit from the same content as a custom-instructions file. Installation per agent is covered below.

## Downloading the skill file

The skill ships as a single file: [**`SKILL.md`**](./SKILL.md).

To download:

1. Open [`SKILL.md`](./SKILL.md) on GitHub.
2. Click the **Download raw file** button (top-right of the file viewer, next to the **Raw** / **Copy** buttons).
3. Save the file somewhere convenient.

You'll move this file to a tool-specific location in the install steps below. To check what changed between versions before downloading a newer copy, see the [CHANGELOG](./CHANGELOG.md).

## Install

There are two common installation **scopes** for every agent:

- **Project scope** - the skill is available only in a specific project or repository. Use this when not every project needs the `te` skill.
- **User / global scope** - the skill is available in every project on your machine. Use this if you work with semantic models across many repos.

### Claude Code

Claude Code's [skills system](https://docs.claude.com/en/docs/claude-code/skills) auto-loads `SKILL.md` files from `.claude/skills/<name>/`. The skill's `description` field is matched against your prompts, so it only loads when relevant - no token cost when you're working on unrelated code.

**Project scope** (skill loads only inside this project):

1. In your project root, create the folder `.claude/skills/te-cli/`.
2. Place the downloaded `SKILL.md` file inside that folder.
3. Restart Claude Code.

The final path should be: `<your-project>/.claude/skills/te-cli/SKILL.md`.

**User scope** (skill loads in every project for the current user):

1. Create the folder `te-cli` inside your user-level Claude skills directory:
   - **macOS / Linux**: `~/.claude/skills/te-cli/`
   - **Windows**: `%USERPROFILE%\.claude\skills\te-cli\` (typically `C:\Users\<you>\.claude\skills\te-cli\`)
2. Place the downloaded `SKILL.md` file inside that folder.
3. Restart Claude Code.

#### Verifying it loaded

Inside a Claude Code session, type:

```
/skills
```

You should see `te-cli` in the listed skills. If not, double-check the file path and frontmatter (the file should start with `---` and have `name: te-cli` on the second line), then restart Claude Code.

A quick functional smoke test:

```
You> what does `te deploy --xmla` do?
```

Claude should answer with the documented behavior (generates a TMSL/XMLA script to stdout instead of deploying), confirming the skill is loaded and being used.

### Claude.ai / Claude Desktop

Claude.ai and Claude Desktop have a built-in **Skills** feature. You upload `SKILL.md` through the app's UI:

1. Download `SKILL.md` (see [above](#downloading-the-skill-file)).
2. Open Claude.ai (or Claude Desktop) and go to **Settings -> Capabilities -> Skills**.
3. Click **Upload skill** and select the `SKILL.md` file you downloaded.
4. The skill is now available in every conversation and loads automatically when you mention `te` or a related concept.

See Anthropic's [Skills documentation](https://docs.claude.com/en/docs/claude-code/skills) for the most current UI flow if the wording has changed.

### GitHub Copilot

Copilot's custom instructions are **always-on** for the scope where they live - they're injected into every Copilot interaction. Two scopes are available.

> [!WARNING]
> Copilot loads custom instructions into **every** interaction, so the full skill (~900 lines) noticeably raises per-prompt token cost. If your work touches `te` only occasionally, prefer installing at **project** scope on the specific repo, or copy only the sections you actually need (e.g. just the command reference + gotchas).

**Project scope** - one instructions file shared by everyone working in the repo:

1. Download `SKILL.md` (see [above](#downloading-the-skill-file)).
2. Open `SKILL.md` in any text editor and **remove the YAML frontmatter block at the top** - that is, everything between the first `---` line and the second `---` line, including those marker lines. Copilot doesn't use the frontmatter and including it wastes tokens.
3. In your repo, create or open the file `.github/copilot-instructions.md`.
4. Paste the remaining `SKILL.md` content into `.github/copilot-instructions.md`. If the file already has content, append the skill content with a separator line between the sections.
5. Commit and push the file so everyone on the repo gets the instructions automatically.

**User scope** (VS Code Copilot user settings, applies to all your Copilot interactions):

1. Open VS Code Settings (**Ctrl+,** on Windows/Linux, **Cmd+,** on macOS).
2. Search the settings for `github.copilot.chat.codeGeneration.instructions` (or a similarly named key - GitHub's setting names evolve).
3. Add an entry pointing to a local copy of `SKILL.md` (with the frontmatter stripped), or paste the content inline.
4. Save settings.

### Generic agents (`AGENTS.md` convention)

For tools that follow the [`AGENTS.md` convention](https://agents.md) or accept an arbitrary instructions file (Aider, Continue, OpenAI Codex CLI, custom in-house agents):

1. Download `SKILL.md` (see [above](#downloading-the-skill-file)).
2. Open `SKILL.md` in a text editor and remove the YAML frontmatter block at the top (everything between the first `---` line and the second `---` line, including those marker lines).
3. Rename the file to `AGENTS.md` and place it at your project root (or wherever your tool expects its instructions file - check the tool's documentation).
4. The next agent invocation in that project will pick up the instructions.

## Updating

To pull a newer version:

1. Visit [`SKILL.md`](./SKILL.md) on GitHub.
2. Use the **Download raw file** button to grab the latest copy.
3. Replace the `SKILL.md` you previously placed in your skills folder (Claude Code) with the new copy, or re-upload it via the Skills UI (Claude.ai / Desktop), or re-paste the body into `.github/copilot-instructions.md` (Copilot).

See the [CHANGELOG](./CHANGELOG.md) for what changed between versions.

## Issues & feedback

Report skill bugs, gaps, or suggestions in the [Tabular Editor CLI issue tracker](https://github.com/TabularEditor/CLI/issues). Please apply the **skill** label (we add this label to issues that affect the skill rather than the CLI itself) so they're easy to triage.

For CLI behavior questions (not skill-content questions), the same tracker is the right place - just don't apply the `skill` label.

## License

[MIT](./LICENSE). Free to use, modify, and redistribute. Attribution appreciated but not required.

## Files in this folder

- [`SKILL.md`](./SKILL.md) - the skill itself (drop-in for Claude / Copilot / generic agents)
- [`README.md`](./README.md) - this file
- [`CHANGELOG.md`](./CHANGELOG.md) - version history
- [`LICENSE`](./LICENSE) - MIT license

---

For the CLI itself, see https://github.com/TabularEditor/CLI.

# AI Agent Instructions

This repository is a plugin marketplace of agent skills for developers of Home Assistant custom integrations — in any
repository, whatever template or layout it started from. Short form: **haci** (Home Assistant Custom Integration).

## Layout

| Path                                     | Read by                                                            |
| ---------------------------------------- | ------------------------------------------------------------------ |
| `.claude-plugin/marketplace.json`        | Claude Code, Copilot CLI, VS Code                                  |
| `.agents/plugins/marketplace.json`       | Codex                                                              |
| `plugins/<p>/plugin.json`                | Copilot CLI, VS Code, Codex (Agent Plugins 1.0)                    |
| `plugins/<p>/.claude-plugin/plugin.json` | Claude Code — same content as `plugin.json` without `$schema`      |
| `plugins/<p>/skills/<s>/SKILL.md`        | every agent ([Agent Skills](https://agentskills.io/specification)) |

`script/check` verifies that these agree; CI runs it.

## Rules for skills here

- **A skill runs in someone else's repository.** It knows no template, no script name and no directory layout. It links
  to nothing outside its own plugin, refers to the project's commands, files and procedures by what they do, takes them
  from the project's `AGENTS.md` (or `CLAUDE.md`, `CONTRIBUTING.md`), and says what to do where the project names none.
- **The project wins.** A skill states a default and defers wherever the project names its own gates, critical files,
  scratch directory, decision records or rules.
- **Every plugin and every skill name starts with `haci-`.** VS Code and Copilot do not prefix plugin skills and
  silently drop one of two skills with the same name, and `ha-` is taken by the skills projects keep in their own
  repositories.
- **A plugin is a unit someone turns off.** Group skills by what a developer would reject together, not by topic. A
  plugin that needs another names it and says what happens without it.
- Never name anything so it suggests an official Home Assistant or HACS project.

## Releases

One version for every plugin, in both manifests of each, bumped together; the tag is `vX.Y.Z`. Projects pin a tag, so a
release that changes what a skill expects from a project — a new `AGENTS.md` entry it reads — says so in its notes.

Commits follow [Conventional Commits](https://www.conventionalcommits.org/); never commit or push without an explicit
request.

A skill that grants autonomy — merging, posting, acting without asking — sets `disable-model-invocation: true` **and**
says in its description that it loads only when the developer invokes it, because not every agent honours the flag.

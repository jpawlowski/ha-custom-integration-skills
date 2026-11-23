# HA Custom Integration skills (haci)

Agent skills for developers of Home Assistant custom integrations: working habits an agent follows on top of your
project's own rules. They come as plugins, and each plugin can be turned on and off on its own.

This is a community project. It is not affiliated with Home Assistant, HACS or the Open Home Foundation.

## Plugins

| Plugin        | Skills                             | Default   | What it does                                                                        |
| ------------- | ---------------------------------- | --------- | ----------------------------------------------------------------------------------- |
| `haci-plan`   | `haci-grill`, `haci-planning`      | on        | Interviews you until a change's requirements are settled, then plans large changes. |
| `haci-review` | `haci-change-review`               | on        | Simplifies a change, reruns the gates, and has a fresh context review it.           |
| `haci-triage` | `haci-issue-triage`, `haci-issues` | available | Works through a GitHub issue backlog one issue at a time, each fix in its own PR.   |

## What the skills read from your project

The skills assume no template and no layout. They take the specifics from your `AGENTS.md` (or `CLAUDE.md`,
`CONTRIBUTING.md`) and fall back to a sensible default where it says nothing:

| Name it in `AGENTS.md`                             | Used by                                   |
| -------------------------------------------------- | ----------------------------------------- |
| lint, type-check and test commands                 | `haci-change-review`, `haci-issue-triage` |
| files where a miss is expensive (critical files)   | `haci-change-review`                      |
| a command for a second review by another agent CLI | `haci-change-review`                      |
| a gitignored directory for working notes           | `haci-grill`, `haci-planning`             |
| where decision records live                        | `haci-grill`, `haci-planning`             |
| what counts as a breaking change, and how to warn  | `haci-issue-triage`, `haci-grill`         |

[ha-integration-blueprint](https://github.com/jpawlowski/ha-integration-blueprint) is one template that names all of
them and enables the default plugins for you.

## Installing

| Agent       | Add the marketplace                                                        | Turn a plugin off                                                                               |
| ----------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Claude Code | `/plugin marketplace add jpawlowski/ha-custom-integration-skills`          | `/plugin`, or `"<plugin>@ha-custom-integration-skills": false` in `.claude/settings.local.json` |
| Copilot CLI | `copilot plugin marketplace add jpawlowski/ha-custom-integration-skills`   | `copilot plugin disable <plugin>`, or `.github/copilot/settings.local.json`                     |
| VS Code     | `chat.plugins.marketplaces`: `["jpawlowski/ha-custom-integration-skills"]` | the Extensions view, per profile or workspace                                                   |
| Codex       | `codex plugin marketplace add jpawlowski/ha-custom-integration-skills`     | the plugin browser                                                                              |

A repository can register the marketplace and enable plugins for everyone who works in it, pinned to a release tag:
`extraKnownMarketplaces` and `enabledPlugins` in `.claude/settings.json` and `.github/copilot/settings.json`, and
`[plugins."<plugin>@ha-custom-integration-skills"]` in `.codex/config.toml`.

OpenCode has no plugin marketplace that can carry skills, so these skills do not reach it.

In Claude Code a plugin skill is invoked with its plugin's name in front: `/haci-plan:haci-grill`.

## Versions

All plugins share one version and are released together under a `vX.Y.Z` tag.

## License

[MIT](LICENSE)

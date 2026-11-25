# HA Custom Integration skills (haci)

Agent skills for developers of Home Assistant custom integrations: Home Assistant knowledge and working habits an
agent follows on top of your project's own rules. They come as plugins, and each plugin can be turned on and off on its own.

This is a community project. It is not affiliated with Home Assistant, HACS or the Open Home Foundation.

## Plugins

| Plugin         | Skills                                                                                                                                   | Default   | What it does                                                                                                                                                                       |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `haci-plan`    | `haci-grill`, `haci-planning`                                                                                                            | on        | Interviews you until a change's requirements are settled, then plans large changes.                                                                                                |
| `haci-review`  | `haci-change-review`                                                                                                                     | on        | Simplifies a change, reruns the gates, and has a fresh context review it.                                                                                                          |
| `haci-quality` | `haci-testing`, `haci-modern-apis`, `haci-breaking-changes`, `haci-quality-review`                                                       | on        | Tests with `pytest-homeassistant`; current APIs instead of deprecated ones; breaking changes, migrations and repair issues; a Quality Scale audit.                                 |
| `haci-build`   | `haci-entity-platform`, `haci-service-action`, `haci-config-flow`, `haci-translations`, `haci-user-experience`, `haci-coordinator-debug` | on        | Entities and platforms, service actions, triggers and conditions, config flows and the manifest, translations, what users see, the coordinator, diagnostics and runtime debugging. |
| `haci-publish` | `haci-marketplace`                                                                                                                       | on        | Publishing through HACS and the Marketplace: `hacs.json`, release assets, brand images, the catalog pull request.                                                                  |
| `haci-triage`  | `haci-issue-triage`, `haci-issues`                                                                                                       | available | Works through a GitHub issue backlog one issue at a time, each fix in its own PR.                                                                                                  |

## What the skills read from your project

The skills assume no template and no layout. They take the specifics from your `AGENTS.md` (or `CLAUDE.md`,
`CONTRIBUTING.md`) and fall back to a sensible default where it says nothing:

| Name it in `AGENTS.md`                             | Used by                                                    |
| -------------------------------------------------- | ---------------------------------------------------------- |
| lint, type-check and test commands                 | `haci-change-review`, `haci-issue-triage`                  |
| files where a miss is expensive (critical files)   | `haci-change-review`                                       |
| a command for a second review by another agent CLI | `haci-change-review`                                       |
| a gitignored directory for working notes           | `haci-grill`, `haci-planning`                              |
| where decision records live                        | `haci-grill`, `haci-planning`                              |
| what counts as a breaking change, and how to warn  | `haci-breaking-changes`, `haci-issue-triage`, `haci-grill` |
| the test command and the shared test fixtures      | `haci-testing`                                             |
| the gates, the package layout and the target tier  | `haci-quality-review`                                      |
| the virtualenv Home Assistant is installed in      | `haci-modern-apis`                                         |
| the package layout and the class prefix            | the `haci-build` skills                                    |
| the command that starts and reads a dev instance   | `haci-coordinator-debug`, `haci-config-flow`               |
| which language files an agent may change           | `haci-translations`, `haci-user-experience`                |
| the release procedure                              | `haci-marketplace`                                         |

[ha-integration-blueprint](https://github.com/jpawlowski/ha-integration-blueprint) is one template that names all of
them and enables the default plugins for you.

## Rules files

A skill that comes with rules keeps them in `references/rules.md`, or one `references/rules.<topic>.md` per kind of file: frontmatter with a `name` and a `description`, then
plain Markdown with no relative links. A project can copy the file into its own path-scoped instructions — for Copilot
and VS Code `applyTo`, for Claude Code `paths` — so the rules load whenever an agent opens a matching file, even
without the skill. Copy it at the tag you pin, and copy it again when the pin moves.

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

### OpenCode

OpenCode has no plugin marketplace, but it reads a skill catalog over HTTP: each plugin publishes one in
`plugins/<plugin>/skills/index.json`. List the catalogs of the plugins you want in `opencode.json`, pinned to the
release tag the other agents use (catalogs ship from the first release after `v0.4.0`):

```json
{
  "skills": [
    "https://raw.githubusercontent.com/jpawlowski/ha-custom-integration-skills/vX.Y.Z/plugins/haci-plan/skills/",
    "https://raw.githubusercontent.com/jpawlowski/ha-custom-integration-skills/vX.Y.Z/plugins/haci-quality/skills/",
    "https://raw.githubusercontent.com/jpawlowski/ha-custom-integration-skills/vX.Y.Z/plugins/haci-review/skills/"
  ]
}
```

OpenCode downloads every file of each skill, `references/` included, and caches them per URL, so moving the tag
fetches the new release. It fetches each catalog again when it starts and leaves a skill out while the catalog cannot
be reached. Turning a plugin off is removing its line. `script/opencode-index` writes the catalogs and `script/check`
fails when one is out of date.

In Claude Code a plugin skill is invoked with its plugin's name in front: `/haci-plan:haci-grill`.

## Versions

All plugins share one version and are released together under a `vX.Y.Z` tag.

## License

[MIT](LICENSE)

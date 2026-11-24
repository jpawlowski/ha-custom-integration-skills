---
name: haci-modern-apis
description: >-
  Verify that the Home Assistant APIs a custom integration uses are current, and replace deprecated ones. Use when a
  DeprecationWarning, "Detected that custom integration", or "will be removed in HA Core X.Y" appears in logs or
  tests, when asked to "modernize", "update to the latest HA version", "fix deprecations", "is this still the right
  API", or before copying a pattern from an older integration or model memory. Covers checking the
  installed Home Assistant source instead of guessing, what counts as verified, device registry single ownership,
  runtime_data, config flow helpers, entity metadata, async I/O, and "do not use → use instead" tables with
  removal versions. SYMPTOMS — load this if you are about to: write `hass.data[DOMAIN]` instead of `runtime_data`;
  call the unscoped `async_get_device()`; use `FlowResult`, `DEVICE_CLASS_*`, or `async_timeout`; create your own
  aiohttp session; add a warning filter to make a deprecation go away; or adopt an undocumented API or config key on
  an issue's say-so.
---

# Keep the integration on current Home Assistant APIs

Home Assistant changes fast enough that any pattern remembered from training data or copied from a blog post is suspect.
The most common failure is confidently writing a 2023 API.

**The project wins.** Its `AGENTS.md` (or `CLAUDE.md`, `CONTRIBUTING.md`) names the virtualenv, the gates and how the
Home Assistant version is raised; use those. This skill is how to find out what is current.

## The rule: check the source, do not recall

The Home Assistant installed in the project's virtualenv is the version the integration is developed against. It is
the authority. Find it once per session:

```bash
python -c "import homeassistant, pathlib; print(pathlib.Path(homeassistant.__file__).parent)"   # → $HA below
python -c "from homeassistant.const import __version__; print(__version__)"
```

Then grep it:

```bash
# Does this API still exist, and is it deprecated?
rg -n "def async_get_device_by_identifier" "$HA/helpers/device_registry.py"
rg -n "deprecated|breaks_in_ha_version" "$HA/helpers/update_coordinator.py"

# How do current core integrations do it?
rg -n "runtime_data" "$HA/components/<similar_integration>/"
```

`breaks_in_ha_version=` in a `@deprecated_function` decorator tells you the actual removal deadline. Prefer that over
any secondary source.

When the installed source is not enough — a pattern that is new rather than deprecated — check the current
documentation before implementing. Every page of <https://contributors.home-assistant.io/> is available as Markdown
(append `index.md` to its URL), and <https://contributors.home-assistant.io/llms.txt> indexes the site; each area has
its own list, such as <https://contributors.home-assistant.io/core/llms.txt>, and a full-text `llms-full.txt` next to
it. Fetch the area you need, not the whole site. Announcements and removal dates are on the
[developer blog](https://contributors.home-assistant.io/blog/).

**The docs follow Home Assistant's development branch, not the installed version.** A page can describe an API that
arrives only in the next release — check that it exists in the installed source before using it; if it does not, it
is upcoming, and using it means raising the integration's minimum Home Assistant version first.

## The same rule, generalised: what counts as verified

This applies to any API, tool or configuration key you are about to commit to. Only a **primary** source settles a
question — the installed source you import, or the current documentation of the tool you are configuring. A GitHub
issue, a blog post, another tool's convention and your own recall are leads, never answers.

Two traps make a secondary source feel primary:

- **Silent failure.** Most configuration rejects nothing: an unknown key is ignored, and the resulting default often
  looks like success. Before believing "X works", ask what a broken X would look like — if the answer is "the same
  thing", the observation proved nothing. Test the case that can fail: the file that should _not_ match, the entry
  that should _not_ load.
- **A key that belongs to a neighbouring tool.** Copilot, Cursor and Claude Code each name the same idea differently
  (`applyTo`, `globs`, `paths`). That a key works in one tool is no evidence for the next, and the wrong one will not
  complain.

When you deviate from documented behaviour, the bar is: name the primary source you checked, and design an observation
that could have come out the other way. If neither is possible, follow the documentation and say you could not verify
it. When the answer matters enough to encode, put a check in the project's scripts or tests rather than a comment.

## Reference files

| File                                                             | When to read                                                                                                                                                                                                                               |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [`references/deprecations.md`](references/deprecations.md)       | Fixing a specific deprecation. "Do not use → use instead" tables with the removal version — moved imports, config entry setup, config flow, entities and per-platform changes, actions and conditions, async and I/O, recorder statistics. |
| [`references/device-registry.md`](references/device-registry.md) | Anything that registers, looks up, links or removes a device: single ownership, scoped lookups, `via_device_id` versus child devices, helper entities, and the device registry deprecations with their removal versions.                   |

Read one when you have a concrete symbol to replace — do not pull it into context speculatively. The tables were checked
against the Home Assistant release each file names; where the installed source disagrees, the source wins.

## The three that matter most

**1. Device registry single ownership.** A device belongs to exactly one config entry and at most one subentry.
Identifiers are unique only within their owning entry. Always scope lookups:

```python
device = device_registry.async_get_device_by_identifier((DOMAIN, serial), entry.entry_id)
```

Never the unscoped `async_get_device()` — it is deprecated, and with two entries sharing an identifier it resolves
ambiguously. Parts of one product are child devices, not `via_device_id` devices —
[`references/device-registry.md`](references/device-registry.md#hub-or-parts-via_device_id-versus-child-devices).

**2. Typed `runtime_data`.** State belongs on the entry, not in `hass.data`:

```python
type MyIntegrationConfigEntry = ConfigEntry[MyIntegrationData]
entry.runtime_data = MyIntegrationData(client=client, coordinator=coordinator)
```

**3. Tests catch Home Assistant's deprecations through a fixture, not through `warnings`** — Home Assistant logs them
instead of raising, so `filterwarnings = ["error"]` never sees them. The fixture, and the one kind of warning that may be
ignored: [`haci-testing`](../haci-testing/SKILL.md#deprecations-and-warnings). Fix the deprecation; never filter it.

## Fixing a deprecation warning

1. Read the log line — it names the symbol, and usually the replacement and the removal version.
2. Confirm the replacement's real signature in the installed source. Signatures shift between the announcement and the
   release.
3. Replace every occurrence, including tests, diagnostics, repairs, and migrations. A half-migrated codebase is worse
   than an un-migrated one.
4. Run the project's gates and its test suite. For a path no test reaches, restart Home Assistant, exercise it, and
   confirm the line is gone from its log — many are logged only once per run, and only when the code runs:
   `rg -n "Detected that custom integration|The deprecated .* from <domain>" home-assistant.log`.

**Raising the minimum Home Assistant version** to reach a new API is not a breaking change, but users on an older
release silently stop receiving updates, so the changelog has to name the new minimum —
[`haci-breaking-changes`](../haci-breaking-changes/SKILL.md#1-recognise-it).

## Do not

- Do not "fix" a deprecation by suppressing the warning.
- Do not trust an example from an older Home Assistant release, a Stack Overflow answer, or your own recall without
  checking the installed source first.

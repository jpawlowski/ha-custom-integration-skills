---
name: haci-config-flow
description: >-
  Work on the config flow of a Home Assistant custom integration — the setup UI, options flow, reauthentication,
  reconfigure, discovery (zeroconf, dhcp, ssdp, bluetooth, usb, mqtt, homekit), subentries, unique IDs, config entry
  version migration, and the manifest that declares discovery. Use when asked to "add a config option", "add a
  setting", "ask the user for X during setup", "add an options flow", "support reauth", "add discovery", "make the
  host configurable", "migrate the config entry", or when editing the config flow, its schemas and validators,
  `manifest.json`, or its discovery matchers. Covers reserved step names, acceptable unique IDs, abort reasons,
  validation, and the migration contract. SYMPTOMS — load this if you are about to: use a host or IP as the unique ID;
  create a config entry without validating the connection; store a credential in `entry.options`; read
  `entry.data[key]` for a key old entries will not have; or change entry data without bumping VERSION/MINOR_VERSION.
---

# Work on the config flow

The config flow is the only supported way to configure a device or service integration
([ADR-0010](https://github.com/home-assistant/architecture/blob/master/adr/0010-integration-configuration.md)).
Everything the user can change lives here, and mistakes here are expensive: `unique_id` and the shape of `entry.data`
are effectively permanent.

**Read [`references/rules.md`](references/rules.md) first** — it holds the rules for every config flow file:
data-vs-options, reserved step names, acceptable unique IDs and their normalisation, schema hygiene, title
placeholders, the migration contract and the entry lifecycle. [`references/rules.manifest.md`](references/rules.manifest.md)
holds the rules for `manifest.json`: required fields, `integration_type`, `iot_class` and requirements. A project may
already load both as path-scoped instructions files. This skill is the order of operations and the decisions; the
MUST/NEVER list for each individual flow is in the first reference below.

**The project wins.** Its `AGENTS.md` (or `CLAUDE.md`, `CONTRIBUTING.md`) may name where the flow's modules live, its
gates, a way to drive a flow against a running instance, and stricter rules for unique IDs; use those. Where it names
none, the defaults below apply.

| File                                                                   | When to read                                                                    |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| [`references/flow-types.md`](references/flow-types.md)                 | Implementing one flow — user, discovery, reauth, reconfigure, options, subentry |
| [`references/discovery-matchers.md`](references/discovery-matchers.md) | Writing a `manifest.json` discovery matcher, or using a discovery helper API    |
| [`references/oauth2.md`](references/oauth2.md)                         | The service authenticates with OAuth2 rather than an API key or a password      |
| [`references/migration.md`](references/migration.md)                   | A config entry migration can fail, and the user needs a way to retry it         |

## Where the pieces go

The project's layout decides. Core's default is one `config_flow.py` at the integration root holding the config flow,
its options flow and any subentry flow, with schemas beside them. A larger integration may split the same pieces into
a package — the config flow, the options flow, a subentry flow, logic shared between them, schemas per flow, and
validators ("can we actually talk to it" checks and input normalisation). Create a piece when something needs it,
rather than leaving empty scaffolding behind. Whatever the split, `config_flow.py` stays at the integration root,
because that is the module Home Assistant and hassfest look for.

## Adding a config option

1. **Decide data or options** using the table in the rules. This is the one irreversible choice here. A polling
   interval is neither — it is not a field at all. In an integration without options yet, the first real option is
   when the options flow and its schema get created.
2. Add the `CONF_*` key to `const.py`, unless `homeassistant.const` already has one.
3. Add the field to the setup or options schema with a selector and a default.
4. Consume it: the coordinator reads `entry.options`, the client reads `entry.data`.
5. Add the `data` and `data_description` translation keys ([`haci-translations`](../haci-translations/SKILL.md)).
   The description says what the option changes. Once the options no longer fit one screen, or an automation should
   change one, see the options section of [`haci-user-experience`](../haci-user-experience/SKILL.md).
6. **Handle existing entries** that predate the key — a default at read time, or a `MINOR_VERSION` bump plus migration.
   This is the step that gets forgotten and breaks upgrades.

## Adding discovery

1. Add the matcher to `manifest.json` — read [`references/discovery-matchers.md`](references/discovery-matchers.md)
   first, because each protocol has a rule that fails silently when you guess (lowercase-only zeroconf properties,
   DHCP MACs arriving without separators, VID/PID pairs shared by hundreds of devices):

   ```json
   "zeroconf": [{ "type": "_myservice._tcp.local.", "name": "acme*" }]
   ```

2. Implement the matching `async_step_<method>()` following the discovery MUST/NEVER list in
   [`references/flow-types.md`](references/flow-types.md).
3. Decide what the discovery payload gives you as a stable unique ID. If it offers nothing stable, stop and discuss it
   — do not fall back to the IP address.
4. Confirm the flow end-to-end in the UI: the card shows a useful name, a second discovery of the same device aborts,
   and a device that changed IP updates the existing entry rather than creating a second one.

## Config entry migration

Bump `MINOR_VERSION` for additive changes, `VERSION` for restructuring. Read `haci-breaking-changes` (plugin
`haci-quality`) first — changing entry data affects every existing install.

Implement `async_migrate_entry()` in `custom_components/<domain>/__init__.py`:

```python
async def async_migrate_entry(hass: HomeAssistant, entry: MyIntegrationConfigEntry) -> None:
    """Migrate an old config entry."""
    if entry.version == 1 and entry.minor_version < 2:
        data = {**entry.data, CONF_PORT: DEFAULT_PORT}
        hass.config_entries.async_update_entry(entry, data=data, minor_version=2)
```

No downgrade branch and no `return False` — the contract is under _Version and migration_ in
[`references/rules.md`](references/rules.md#version-and-migration); a migration that can fail needs
[`references/migration.md`](references/migration.md).

Migrations must be idempotent and must never delete a key they do not understand. Cover every migration with a test
(`haci-testing`, plugin `haci-quality`).

## Validate

Run the project's gates — lint, type check, hassfest and the tests. Hassfest cross-checks every step, error and abort
key against the translations.

Then restart Home Assistant and **walk every flow you touched** — add, reconfigure, options, and, by invalidating the
credential, reauth. Hassfest proves the translation keys exist; only walking the flow proves it is usable. A flow that
passes CI and dead-ends on the second step is the normal failure here.

Walk it from the terminal first where you can — it is faster, it is repeatable, and it shows the serialised schema Home
Assistant actually built, which is where defaults and selectors go wrong. Use the project's command for driving a
flow if it has one. Without one, the frontend's own endpoints do the same with an administrator's access token:
`POST /api/config/config_entries/flow` with `{"handler": "<domain>"}` starts a flow (add `"entry_id"` for
reconfigure), `POST /api/config/config_entries/flow/<flow_id>` with the field values submits a step, and
`/api/config/config_entries/options/flow` does both for an options flow, with the entry ID as `handler`. Each
response carries the `step_id`, any `errors`, and the serialised `data_schema` with every field's type and default.

Two things only the browser can confirm, so finish there: that the translated labels and `data_description` hints read
correctly, and that the form renders the way the selectors promise.

## Do not

- Do not do blocking I/O or long retries inside a flow step; validate with a short timeout.
- Do not call the flow done before walking it end to end.
- Do not give an optional free-text field `default=None` — probatio injects that default when the field is left empty
  and the selector then rejects it, making the form unsubmittable. Carry the current value in
  `description={"suggested_value": ...}` instead.

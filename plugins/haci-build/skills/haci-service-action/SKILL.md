---
name: haci-service-action
description: >-
  Add, change, or remove a Home Assistant service action (also called a "service" or "action") in a custom
  integration. Use when asked to "add a service", "add an action", "expose X as a service call", "add a service
  that returns data", "make this callable from an automation", "add a trigger", "add a condition", or when editing
  services.yaml, the module that registers actions, service schemas, response data, or a trigger.py / condition.py
  platform. Covers the services.yaml contract, probatio schemas, selectors, entity/device targeting, response data,
  translated errors, triggers and conditions, and the Bronze rule that actions register in async_setup(). SYMPTOMS
  — load this if you are about to: register an action in `async_setup_entry()`; add a `services.yaml` field without a
  selector; add an `entity_id` field where `target:` belongs; raise a plain English `HomeAssistantError`; swallow an
  action failure into a log line; or add a `device_trigger.py`, `device_condition.py` or `device_action.py`.
---

# Add or change a service action

Terminology: Home Assistant renamed "services" to **actions** in the UI, but the Python API, the file name
`services.yaml`, and the translation keys all still say `service`. Every text a user reads says "action" — never
"service" or "service action"; `service` stays in code identifiers.

**Read [`references/rules.md`](references/rules.md) and
[`references/rules.services-yaml.md`](references/rules.services-yaml.md) first** — they hold the rules: where
registration must happen, the probatio schema pattern, how to resolve the target entry or device, which exception type
to raise, permissions, the `target:` field, response-data requirements (including the `.isoformat()` trap), entity
service actions, the `icons.json` shape, and the `services.yaml` structure, selectors and wording. A project may
already load the same text as path-scoped instructions files. This skill is the order of operations and the design
decisions.

**The project wins.** Its `AGENTS.md` (or `CLAUDE.md`, `CONTRIBUTING.md`) may name the module or package that
registers the actions, its handler signature, its config entry type, and its lint, type-check, hassfest and test
commands; use those. Where it names none, Core's layout applies: a `services.py` with `async_setup_services(hass)`
beside `__init__.py`, and the defaults below.

## Decide the shape first

Target the thing the action actually acts on — there are three levels, and picking the wrong one is what makes an
action awkward to call:

| Question                                            | Consequence                                                                      |
| --------------------------------------------------- | -------------------------------------------------------------------------------- |
| Does it act on one entity?                          | Add a `target:` with an `entity:` selector — do not invent an `entity_id` field. |
| Does it act on one device?                          | A required `device_id` field with a `device:` selector.                          |
| Does it act on the whole account/hub?               | A required `config_entry_id` field with a `config_entry:` selector.              |
| Does the caller need data back?                     | `SupportsResponse.ONLY` or `.OPTIONAL` and return a `ServiceResponse` dict.      |
| Could an entity attribute or a button entity do it? | Prefer the entity. Actions are for things that do not map onto entity state.     |

**A target is never optional and never defaulted.** Picking "the only entry" when the caller named none makes every
automation break the day a second entry is added. The one exception is `single_config_entry: true` in the manifest,
where there can only ever be one — the only case for an optional `config_entry_id`, which
`service.async_get_config_entry` then resolves to that one loaded entry when it is `None`.

If an entity would model it better, say so before implementing — a `button` or `number` entity is more discoverable
than an action.

What automations need from the action — the shape of the response data, defaults, which errors `continue_on_error`
can skip — is in [`haci-user-experience`](../haci-user-experience/SKILL.md), in its section on automations.

## Procedure

### 1. Describe it in `services.yaml`

```yaml
set_target_value:
  name: Set target value
  description: Writes a new target value to the device.
  fields:
    config_entry_id:
      name: Integration entry
      description: The configured entry to write to.
      required: true
      selector:
        config_entry:
          integration: <domain>
    value:
      name: Value
      description: The new target value.
      required: true
      example: 21.5
      selector:
        number:
          min: 0
          max: 100
          step: 0.5
          unit_of_measurement: "%"
```

Write this before the Python. The YAML is the contract the user sees, and getting it right forces you to decide what
the action actually takes. Every field needs a selector; `required: true` only when there is no sensible default;
rarely used fields go under `advanced: true`.

### 2. Implement the handler

Put the handler body where the project keeps them — a module per logical group when it has a package for actions,
otherwise beside the registration. A handler that receives the resolved entry keeps the wrapper thin:

```python
async def async_handle_set_target_value(
    hass: HomeAssistant,
    entry: MyIntegrationConfigEntry,
    call: ServiceCall,
) -> None:
    """Handle the set_target_value action."""
    value: float = call.data[ATTR_VALUE]
    client = entry.runtime_data.client
    try:
        await client.async_set_target_value(value)
    except MyIntegrationApiClientAuthenticationError as err:
        entry.async_start_reauth(hass)
        raise HomeAssistantError(
            translation_domain=DOMAIN,
            translation_key="auth_failed",
        ) from err
    except MyIntegrationApiClientError as err:
        raise HomeAssistantError(
            translation_domain=DOMAIN,
            translation_key="set_target_value_failed",
            translation_placeholders={"error": str(err)},
        ) from err
    await entry.runtime_data.coordinator.async_request_refresh()
```

The exception classes are whatever the project's client or library raises. Every raised exception needs a matching
`exceptions.<key>.message` entry in `translations/en.json` ([`haci-translations`](../haci-translations/SKILL.md)) —
that is the step most easily forgotten, and hassfest will catch it.

`ConfigEntryAuthFailed` is deliberately absent — in a handler it would only log (the rules file, under exception
handling, says why).

### 3. Register in `async_setup()` — not `async_setup_entry()`

Actions must exist even when no config entry is loaded, so automations referencing them validate and produce a helpful
error instead of "unknown service". Add the wrapper and registration in the module that registers the actions:

```python
SERVICE_SET_TARGET_VALUE = "set_target_value"

SET_TARGET_VALUE_SCHEMA = probatio.Schema(
    {
        probatio.Required(ATTR_CONFIG_ENTRY_ID): cv.string,
        probatio.Required(ATTR_VALUE): probatio.Coerce(float),
    }
)


@callback
def async_setup_services(hass: HomeAssistant) -> None:
    """Register the integration's actions."""

    async def handle_set_target_value(call: ServiceCall) -> None:
        entry: MyIntegrationConfigEntry = service.async_get_config_entry(
            hass, DOMAIN, call.data[ATTR_CONFIG_ENTRY_ID]
        )
        await async_handle_set_target_value(hass, entry, call)

    if not hass.services.has_service(DOMAIN, SERVICE_SET_TARGET_VALUE):
        hass.services.async_register(
            DOMAIN,
            SERVICE_SET_TARGET_VALUE,
            handle_set_target_value,
            schema=SET_TARGET_VALUE_SCHEMA,
        )
```

Call `async_setup_services(hass)` once, from `async_setup()` in `__init__.py` — check whether it is already called
there, and never add a second call in `async_setup_entry()`.

Resolve the entry in the wrapper, before the handler touches `entry.runtime_data` — the action now exists while no
entry is loaded, so this is the other half of `action-setup`. Which helper, and why it needs no translation keys: the
rules file, under its critical rules.

### 4. Translations and icons

`services.<action>.name` / `.description` plus one pair per field, and the action icon in `icons.json`. The strings in
`services.yaml` are only the fallback — the translation file is what users actually see. See
[`haci-translations`](../haci-translations/SKILL.md).

### 5. Validate

Run the project's lint and type-check commands, then hassfest — it validates `services.yaml` against the translation
keys — and the test suite. Where the project names no commands, run `ruff`, the type checker it configures, `pytest`,
and for hassfest the `home-assistant/actions/hassfest` GitHub Action (or `python -m script.hassfest` in a Home
Assistant Core checkout).

Then restart Home Assistant and call the action from **Settings** > **Tools** > **Actions** — check that every field renders with
its selector, and that an intentional failure surfaces a readable message.

## Triggers and conditions

When the integration should offer its own automation triggers or conditions — "the device reported a fault", "the
charger is charging" — that is `trigger.py` / `condition.py` plus `triggers.yaml` / `conditions.yaml`, never a
`device_trigger.py`, `device_condition.py` or `device_action.py`. Those are frozen upstream and appear all over
integrations from 2021–2023, so the pattern is familiar and wrong.

The class signatures, the `options` / `target` config shape, the entity-state factories, the strings and icons, and
the naming rules are in [`references/triggers-conditions.md`](references/triggers-conditions.md). The API is young:
read the installed source before relying on any of it.

## Actions are what a voice assistant reads

The Assist LLM API exposes service actions to conversation agents using the `description` from the translations. That
makes the description functional, not decorative: write what the action does in a sentence a person would say, not
"Calls the set_target_value endpoint".

## Removing or renaming an action

Renaming an action breaks every automation and script that calls it. Treat it as a breaking change: see
`haci-breaking-changes` (plugin `haci-quality`) and get explicit approval first.

## Do not

- Do not add `probatio.Schema` validation that contradicts the selector in `services.yaml` — two sources of truth for
  the same field is how a form starts accepting values the handler then rejects.
- Do not call it done before invoking it from **Settings** > **Tools** > **Actions**.

The remaining rules — registration location, entry resolution, exception types, permissions, `target:`, response
data — are in [`references/rules.md`](references/rules.md).

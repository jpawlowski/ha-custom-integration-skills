---
name: haci-entity-platform
description: >-
  Add or change a Home Assistant entity platform or an individual entity in a custom integration — sensor,
  binary_sensor, switch, button, number, select, fan, climate, cover, light, lock, and friends. Use when asked to
  "add a sensor", "add a new entity", "add a platform", "expose <value> as an entity", "add an entity to the
  device", "make this entity diagnostic", or when changing EntityDescription metadata, device classes, state
  classes, entity categories, or availability. Covers where a platform lives, the coordinator-backed entity
  contract, unique IDs, device grouping, events, and the translation and validation steps that must follow.
  SYMPTOMS — load this if you are about to: hardcode `name=` or `icon=` in an EntityDescription instead of
  `translation_key`; omit `state_class` on a numeric measurement; call the API client from an entity property; set
  `_attr_unique_id` or `_attr_device_info` in a platform entity; or put two entity classes in one file.
---

# Add or change an entity platform

Entities are the user-visible surface of the integration. Getting the metadata right at creation time matters more than
usual: unique IDs, device classes, and entity IDs are effectively permanent once a user has installed the integration.

**Read [`references/rules.md`](references/rules.md) first** — it holds the rules this procedure assumes: inheriting the
base entity, the required `EntityDescription` fields, the coordinator-only data rule, availability, entity categories,
attributes, device info fields, writes, `PARALLEL_UPDATES`, and device registry ownership. A project may already load
the same text as a path-scoped instructions file. This skill is the procedure and the decisions — it does not restate
those rules.

**The project wins.** Its `AGENTS.md` (or `CLAUDE.md`, `CONTRIBUTING.md`) may name where platforms live, its base
entity class and the helpers on it, its lint, type-check, validation and test commands, and its policy on translation
files; use those. Where it names none, the defaults below apply: Core's layout of one `<platform>.py` module per
platform with the shared base entity in `entity.py`.

Before implementing a platform, read its row — required members and the trap it carries:

| File                                                                           | Platforms                                                                                                                                                     |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`references/sensors.md`](references/sensors.md)                               | `sensor`, `binary_sensor` — state class and unit rules                                                                                                        |
| [`references/controls.md`](references/controls.md)                             | `switch`, `button`, `number`, `select`, `text`, `date`, `datetime`, `time`, `scene`, `siren`, `cover`, `valve`, `lock`, `alarm_control_panel`, `fan`, `light` |
| [`references/climate-and-appliances.md`](references/climate-and-appliances.md) | `climate`, `water_heater`, `humidifier`, `vacuum`, `lawn_mower`, `media_player`                                                                               |
| [`references/other-platforms.md`](references/other-platforms.md)               | `event`, `update`, `image`, `camera`, `device_tracker`, `notify`, `todo`, infrared and radio frequency consumers                                              |

Lifecycle hooks beyond `async_added_to_hass` — any platform: [`references/lifecycle.md`](references/lifecycle.md).

## Before you write code

Clarify — ask the developer if any of this is missing:

- Which platform (`sensor`, `switch`, …) and whether it already exists in the integration. The most specific one
  that fits, not a sensor by habit: the platform-choice section of
  [`haci-user-experience`](../haci-user-experience/SKILL.md).
- Which field of `coordinator.data` backs the entity, and its unit and value range.
- Whether it is a primary entity or a diagnostic/config entity (`EntityCategory`).
- For writable platforms: which API method performs the write.

Then read the closest existing sibling — an entity of the same platform already in the integration. Match it rather
than inventing a second style.

## Layout

Where a platform lives is the project's layout. Without one, Core's is the default: `<platform>.py` holds
`async_setup_entry`, the descriptions and the entity class, and the shared base entity sits in `entity.py`.

Whatever the layout, once a `value_fn` description carries the value lookup, one entity class serves every
description on the platform, and copying it per group of descriptions would be the duplication `value_fn` exists to
remove. Keep that one class in one place and let the group modules hold descriptions only. Split a platform into
several files when a second class would otherwise be a copy of the first, not before.

## Procedure

### 1. Decide the metadata, then write the descriptions

The mechanical rules are in the rules file. What this step actually costs you is judgement:

- **`key`** becomes part of the unique ID. Pick it once — renaming it later is a breaking change
  (`haci-breaking-changes`, plugin `haci-quality`).
- **Primary, diagnostic or config?** The test is in the rules' _Entity categories_ section; disabled by default is a
  separate choice, hidden another. Which is which, and why the choice is final: the rules' _Disabled and hidden by
  default_ section.
- **Is there a matching `device_class`?** Check the platform's enum before inventing units or icons. A device class
  buys unit conversion, correct icons and voice-assistant behaviour for free.
- **Is the value a measurement or a total?** Then it needs a `state_class`, or the user gets no long-term statistics.
- **Who reads it — a dashboard, an automation, or both?** That decides state versus attribute, the icon, and what
  goes beside the state: [`haci-user-experience`](../haci-user-experience/SKILL.md).

### 2. Entity class

Implement only the platform members and the value extraction; everything else comes from the base class.

```python
class MyIntegrationAirQualitySensor(MyIntegrationEntity, SensorEntity):
    """Air quality sensor."""

    entity_description: MyIntegrationSensorEntityDescription

    @property
    def native_value(self) -> StateType:
        """Return the current value."""
        return self.entity_description.value_fn(self.coordinator.data)
```

A value that is simply missing from this poll is `unknown`, not `unavailable` — returning `None` above already does
that, and it keeps the entity usable in templates and automations.

Only override `available` when a single entity can genuinely be **gone** while the rest of the device is fine — a
missing key that means a missing sub-device. The base `CoordinatorEntity` already handles the whole-device case:

```python
@property
def available(self) -> bool:
    """Return True if the backing sub-device is still present."""
    return super().available and self.entity_description.key in self.coordinator.data
```

### 3. Register the platform

For a brand-new platform only:

1. Add the `Platform.<NAME>` member to the `PLATFORMS` list in the integration's `__init__.py` (keep the list
   alphabetical).
2. Check that `async_setup_entry` forwards with `async_forward_entry_setups(entry, PLATFORMS)` and
   `async_unload_entry` tears down with `async_unload_platforms(entry, PLATFORMS)` (or its `_or_raise` variant). If
   both already take `PLATFORMS`, nothing else is needed.

### 4. Device grouping

The base entity places every entity on the config entry's device. Only deviate when the integration genuinely models
more than one device, and then decide whether each one is a part of one product (a child device) or a separate
product behind a hub (`via_device_id`) — the decision table and the single-owner rules are in `device-registry.md` in
`haci-modern-apis` (plugin `haci-quality`).

When devices come and go at runtime (`dynamic-devices`), add entities from a coordinator listener that diffs
`set(coordinator.data)` against a `known_devices` set. If the integration also removes devices that are gone
(`stale-devices`), drop each removed ID from `known_devices` too — otherwise a device that returns is never added
again until the entry reloads. Both rules: `quality-scale-rules.md` in `haci-quality-review` (plugin
`haci-quality`).

### 5. Translations

Add the `translation_key` under `entity.<platform>.<key>.name` in `translations/en.json`, plus `state` keys for enum
sensors. A repository that keeps a Core-style `strings.json` edits that and regenerates or copies `translations/en.json`
from it — a custom integration loads only the latter. Which other language files you may touch is the project's policy; where it names none, leave them to their
translators. See [`haci-translations`](../haci-translations/SKILL.md).

### 6. Validate

Run the project's lint, type check, hassfest validation and test suite. Where it names no commands, run the tools it
configures (`ruff`, `pyright` or `mypy`, `pytest`), and for hassfest the `home-assistant/actions/hassfest` GitHub
Action (or `python -m script.hassfest` in a Home Assistant Core checkout). A check you could not run is reported as
not run, never skipped silently.

Then restart the project's Home Assistant development instance and confirm the entity appears with the expected name,
unit, icon, and category. Without one, say that the entity was verified through the tests only. If it does not appear at all, switch to
[`haci-coordinator-debug`](../haci-coordinator-debug/SKILL.md).

## Something happened, rather than something is

For a button press, a doorbell ring, a fault report — anything that **occurs** instead of **holds** — reach for an
`event` entity, not a sensor.

**The anti-pattern: do not represent an event as entity state.** A binary sensor that is `on` for 30 seconds after a
doorbell ring is wrong in both directions — it invents a duration the device never reported, and it loses a second
ring inside the window. `event` entities exist for exactly this.

Firing on the event bus directly is the fallback for what does not fit an entity at all:

- Event type is `<domain>_event`; include the `device_id` from the device registry in the payload.
- The wiring belongs in `__init__.py`, not in an entity — an entity that is disabled would never subscribe.
- A device that **only** fires events still has to be registered in the device registry by hand; nothing else creates
  it.

Listening is the mirror image: prefer a helper from `homeassistant.helpers.event` over `hass.bus.async_listen`, and
wrap the unsubscribe callable every helper returns in `entry.async_on_unload(...)`.

## Do not

- Do not add an entity for data the coordinator does not already fetch — extend the coordinator first, or you will be
  tempted to call the API from the entity.
- Do not write `significant_change.py` or `reproduce_state.py` speculatively. The first only matters when the
  integration's entities are exported to Google Assistant or Alexa and are noisy enough to need filtering
  (`async_check_significant_change(hass, old_state, old_attrs, new_state, new_attrs, **kwargs) -> bool | None`). The
  second is only for an integration that defines its **own** domain — entities added to an existing platform inherit
  scene support from it for free.
- Do not rename an existing `key`, `translation_key`, or unique ID without reading `haci-breaking-changes`.
- Do not skip the sibling read in _Before you write code_. Most "new" entities are a variation of one that already
  exists.
- Do not raise `ConfigEntryNotReady` from a platform's `async_setup_entry` — the config entry setup has already
  finished by then and nothing catches it.

The remaining hard rules are in [`references/rules.md`](references/rules.md); the per-platform members are in the
`references/` files listed at the top.

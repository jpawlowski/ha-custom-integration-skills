---
name: haci-testing
description: >-
  Write, fix, or extend automated tests for a Home Assistant custom integration using pytest and
  pytest-homeassistant. Use when asked to "add tests", "write a test", "the tests fail", "improve coverage", "add a
  regression test", "test the config flow", "update snapshots", or when a behavioural change needs verification before
  it can be called done. Covers when a test is required and where it sits, the shared fixtures, mocking the API
  client, config flow and entity test patterns, registry assertions, syrupy snapshots, freezing time, and Home
  Assistant's logged deprecations. SYMPTOMS — load this if you are about to: instantiate an entity class directly
  instead of asserting on `hass.states`; redefine a fixture `pytest-homeassistant` provides; use `time.sleep` instead
  of `async_fire_time_changed`; silence a DeprecationWarning to make the suite pass; or commit a snapshot you have not
  read.
---

# Testing

Home Assistant tests are integration tests by nature: you load a real config entry into a real `hass` instance and
assert on `hass.states` — not on Python objects.

**Read [`references/rules.md`](references/rules.md) first** — it holds the rules: the fixtures never to redefine, how a
test is written (types, no branches, no network), the core-interface rule, registry assertions, and what to mock and
what never to mock. A project may already load the same text as a path-scoped instructions file. This skill is the
scaffolding and the judgement calls.

**The project wins.** Its `AGENTS.md` (or `CLAUDE.md`, `CONTRIBUTING.md`) names the test command, the test directory
layout and its own fixtures; use those. Where it names none, run `pytest` and follow the defaults below.

## When a test is required

- Behavioural change, bug fix, or regression → add a proportionate test.
- **The test sits at the seam where the bug occurred** — the coordinator for a parsing bug, `hass.states` for an entity
  bug. A test at a shallower layer passes while the bug lives on; if no seam can reproduce it, say so as a finding.
  An existing test that missed the bug sits at the wrong seam — move it rather than add a second. Name the test after
  the behaviour, not the issue number.
- **A bug fix starts with a failing test.** Write it, watch it fail for the reported reason, then fix — a test written
  after the fix may never have been able to fail.
- Documentation-only, formatting-only, or anything that cannot affect runtime → no test needed.
- If a test is impractical (needs real hardware, non-deterministic timing), say so explicitly and describe the residual
  risk. Do not silently skip it, and never claim coverage that does not exist.

## The shared fixtures

Most integrations need three fixtures in `tests/conftest.py`. Where the project already has them, use them by name
instead of defining a variant per file; where it has none, these names are the common choice:

- `config_entry` — a `MockConfigEntry`, not yet added to `hass`.
- `mock_api` — the `AsyncMock` that stands in for the data source, patched where the integration imports it; set
  `return_value` or `side_effect` per test.
- `init_integration` — adds `config_entry`, sets it up with `mock_api` active, and returns the loaded entry.

Building them, and the variant for a third-party library client:
[`references/recipes.md`](references/recipes.md#the-three-shared-fixtures).

## Layout

`tests/` mirrors the integration package, one test file per module or feature:

```text
tests/
├── conftest.py
├── fixtures/                  # recorded API payloads
├── test_init.py               # setup, unload, reload, migration
├── test_config_flow.py        # user, reauth, reconfigure, options, discovery
├── test_diagnostics.py        # snapshot + redaction
├── sensor/test_air_quality.py
└── snapshots/                 # syrupy .ambr files, committed
```

## Patterns

### Setup and unload lifecycle

```python
async def test_setup_and_unload(hass: HomeAssistant, init_integration: MockConfigEntry) -> None:
    """The entry loads and unloads cleanly."""
    assert init_integration.state is ConfigEntryState.LOADED

    assert await hass.config_entries.async_unload(init_integration.entry_id)
    await hass.async_block_till_done()
    assert init_integration.state is ConfigEntryState.NOT_LOADED
```

### Setup failure

```python
@pytest.mark.parametrize(
    ("error", "expected_state"),
    [
        pytest.param(ApiCommunicationError, ConfigEntryState.SETUP_RETRY, id="unreachable"),
        pytest.param(ApiAuthenticationError, ConfigEntryState.SETUP_ERROR, id="bad-credentials"),
    ],
)
async def test_setup_failures(
    hass: HomeAssistant,
    config_entry: MockConfigEntry,
    mock_api: AsyncMock,
    error: type[Exception],
    expected_state: ConfigEntryState,
) -> None:
    """Client failures map onto the right entry state."""
    mock_api.side_effect = error
    config_entry.add_to_hass(hass)
    await hass.config_entries.async_setup(config_entry.entry_id)
    await hass.async_block_till_done()
    assert config_entry.state is expected_state
```

`ApiCommunicationError` and `ApiAuthenticationError` stand for the client's own exception types.

### Entity state

```python
async def test_sensor_state(hass: HomeAssistant, init_integration: MockConfigEntry) -> None:
    """The sensor exposes the coordinator value."""
    state = hass.states.get("sensor.example_device_pm25")
    assert state is not None
    assert state.state == "12.3"
    assert state.attributes[ATTR_UNIT_OF_MEASUREMENT] == UnitOfDensity.MICROGRAMS_PER_CUBIC_METER
```

Never instantiate an entity class directly — go through `hass.states`. Compare units against the `homeassistant.const`
constant, never a typed literal: `µ` (U+00B5) and the `μ` (U+03BC) Home Assistant uses look identical and compare
unequal.

### Every entity of a platform

`snapshot_platform`, and the `entity_registry_enabled_by_default` fixture it needs, are in
[`references/recipes.md`](references/recipes.md#every-entity-of-a-platform-snapshot_platform).

### Config flow

```python
@pytest.mark.usefixtures("mock_api")
async def test_user_flow(hass: HomeAssistant) -> None:
    """The happy path creates an entry."""
    result = await hass.config_entries.flow.async_init(DOMAIN, context={"source": SOURCE_USER})
    assert result["type"] is FlowResultType.FORM
    assert result["step_id"] == "user"

    result = await hass.config_entries.flow.async_configure(
        result["flow_id"],
        {CONF_USERNAME: "test-user", CONF_PASSWORD: "test-password"},
    )
    assert result["type"] is FlowResultType.CREATE_ENTRY
    assert result["result"].unique_id == "test-unique-id"
```

Parametrise the error branches (`invalid_auth`, `cannot_connect`, `unknown`) and always assert that the flow **recovers**
— show the form again with the error, then succeed on the second attempt. `config-flow-test-coverage` (Bronze) expects
every step and every abort reason to be exercised.

### Time-driven updates

```python
freezer.tick(DEFAULT_SCAN_INTERVAL)
async_fire_time_changed(hass)
await hass.async_block_till_done()
```

Use the `freezer` fixture (`pytest-freezer`, typed `FrozenDateTimeFactory`) and `async_fire_time_changed`.
`time.sleep()` in a test is always wrong.

### Snapshots

```python
async def test_diagnostics(
    hass: HomeAssistant,
    hass_client: ClientSessionGenerator,
    init_integration: MockConfigEntry,
    snapshot: SnapshotAssertion,
) -> None:
    """Diagnostics output is stable and redacted."""
    assert await get_diagnostics_for_config_entry(hass, hass_client, init_integration) == snapshot
```

Snapshots are for large stable structures (diagnostics, entity registry dumps). They complement functional assertions;
they do not replace them. Regenerate with `pytest --snapshot-update` (or the project's wrapper for it) and **read the
diff** before committing the `.ambr` file — an accepted snapshot of a bug is worse than no test.

### Repair issues

Assert through the `issue_registry` fixture: `issue_registry.async_get_issue(DOMAIN, issue_id)` is not `None` after the
trigger, and is `None` again once the condition clears.

## Deprecations and warnings

**Home Assistant's own deprecations are logged, not raised.** Where the pytest configuration sets
`filterwarnings = ["error"]`, a Python `DeprecationWarning` fails the suite, which catches libraries — but a removed helper, a deprecated
constant or a wrong-thread call in Home Assistant produces a log line instead: `Detected that custom integration
'<domain>' …` from `report_usage()`, or `The deprecated … was used from <domain>` from the deprecation helpers. Only an
autouse fixture that fails a test on such a record turns them into failures; if the project has none, add the one in
[`references/recipes.md`](references/recipes.md#failing-on-logged-deprecations). A path no test reaches still needs the
runtime log. Fix the cause; never filter the message.

The one warning to ignore rather than fix is one **Home Assistant Core raises about its own use of a library it
pins**, which no change in the integration can fix and which would otherwise block every test that touches that
subsystem — aiohttp warnings from `homeassistant/helpers/aiohttp_client.py` and `homeassistant/components/http/` are
typical. Before adding one, confirm the warning's origin is a Core file and not this integration; a warning about the
integration's own code stays an error. Then key the ignore to the exact message rather than the category, and say in
a comment where it comes from and why it cannot be fixed here — a bare `DeprecationWarning` ignore also hides the next
real deprecation.

With `asyncio_mode = "auto"`, async tests need no `@pytest.mark.asyncio`.

## Coverage priorities

Order of value, highest first: config flow branches → setup/unload/migration → coordinator error translation → entity
state and availability → service action error paths → diagnostics redaction. Chase behaviour, not a percentage.

## Do not

- Do not mock `aiohttp` at the transport level when mocking the client is enough — you end up testing your mock.
- Do not assert on log output as a substitute for asserting on state.
- Do not commit a snapshot you have not read.
- Do not report coverage you did not measure, or describe a test as passing without running it.

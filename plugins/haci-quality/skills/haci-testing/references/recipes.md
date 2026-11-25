# Test recipes

Fixtures and patterns from [`../SKILL.md`](../SKILL.md) spelled out. `<domain>` is the integration's domain and
`ApiClient` its client class; replace both.

## Imports

```python
from collections.abc import Generator
from unittest.mock import AsyncMock, patch

from freezegun.api import FrozenDateTimeFactory
import pytest
from syrupy.assertion import SnapshotAssertion

from homeassistant.config_entries import SOURCE_USER, ConfigEntryState
from homeassistant.core import HomeAssistant
from homeassistant.data_entry_flow import FlowResultType
from homeassistant.helpers import entity_registry as er
from pytest_homeassistant.common import (
    MockConfigEntry,
    async_fire_time_changed,
    load_json_object_fixture,
    snapshot_platform,
)
from pytest_homeassistant.components.diagnostics import get_diagnostics_for_config_entry
from pytest_homeassistant.typing import ClientSessionGenerator
```

With the older `pytest-homeassistant-custom-component` package, the last three come from
`pytest_homeassistant_custom_component` instead.

## The three shared fixtures

```python
@pytest.fixture
def mock_api() -> Generator[AsyncMock]:
    """Replace the client's HTTP layer, keeping its payload logic under test."""
    with patch(
        "custom_components.<domain>.api.ApiClient._request",
        new_callable=AsyncMock,
        return_value=load_json_object_fixture("device.json"),
    ) as request:
        yield request


@pytest.fixture
def config_entry(request: pytest.FixtureRequest) -> MockConfigEntry:
    """Return a config entry; parametrize it indirectly to set its options."""
    return MockConfigEntry(
        domain=DOMAIN,
        title="Example device",
        unique_id="test-unique-id",
        data={CONF_USERNAME: "test-user", CONF_PASSWORD: "test-password"},
        options=getattr(request, "param", {}),
    )


@pytest.fixture
async def init_integration(hass: HomeAssistant, mock_api: AsyncMock, config_entry: MockConfigEntry) -> MockConfigEntry:
    """Set up the integration from the config entry and return it loaded."""
    config_entry.add_to_hass(hass)
    await hass.config_entries.async_setup(config_entry.entry_id)
    await hass.async_block_till_done()
    return config_entry
```

Patching the client's lowest request method keeps its payload handling under test. The path is the module that
defines the class: `custom_components.<domain>.api.ApiClient._request` for an `api.py`,
`custom_components.<domain>.api.client.ApiClient._request` for an `api/` package. A client without one such method
gets its public methods patched instead, each with `autospec=True`.

## `mock_api` for a third-party library client

When the client is a third-party library instead, keep the fixture name and patch the library class **where the
integration imports it** — every import site, since setup and the config flow's validator each import it separately:

```python
@pytest.fixture
def mock_api() -> Generator[AsyncMock]:
    """Replace the library client at each place the integration imports it."""
    with (
        patch("custom_components.<domain>.ApiClient", autospec=True) as client_class,
        patch("custom_components.<domain>.config_flow.ApiClient", new=client_class),
    ):
        client = client_class.return_value
        client.async_get_data.return_value = load_json_object_fixture("device.json")
        yield client
```

`autospec=True` makes the mock fail when the real signature changes, which is the whole point. With this variant, set
`side_effect` on the method (`mock_api.async_get_data.side_effect`), not on `mock_api` itself.

## Failing on logged deprecations

```python
@pytest.fixture(autouse=True)
def fail_on_deprecation_report(caplog: pytest.LogCaptureFixture) -> Generator[None]:
    """Fail a test when Home Assistant logs a deprecation this integration caused."""
    yield
    reports = [
        message
        for record in [*caplog.get_records("setup"), *caplog.get_records("call")]
        if record.levelno >= logging.WARNING
        and DOMAIN in (message := record.getMessage())
        and message.startswith(("Detected that custom integration", "The deprecated"))
    ]
    assert not reports, "\n".join(reports)
```

Matching on `DOMAIN` keeps a deprecation another integration or Core itself causes from failing this suite.

## Every entity of a platform: `snapshot_platform`

```python
@pytest.mark.usefixtures("entity_registry_enabled_by_default", "mock_api")
async def test_sensors(
    hass: HomeAssistant,
    entity_registry: er.EntityRegistry,
    snapshot: SnapshotAssertion,
    config_entry: MockConfigEntry,
) -> None:
    """Every sensor matches its recorded registry entry and state."""
    with patch("custom_components.<domain>.PLATFORMS", [Platform.SENSOR]):
        config_entry.add_to_hass(hass)
        await hass.config_entries.async_setup(config_entry.entry_id)
        await hass.async_block_till_done()
    await snapshot_platform(hass, entity_registry, snapshot, config_entry.entry_id)
```

`snapshot_platform` accepts exactly one platform and fails on a disabled entity, so request `entity_registry_enabled_by_default`, which `pytest-homeassistant` provides — never define
it again in `conftest.py`.

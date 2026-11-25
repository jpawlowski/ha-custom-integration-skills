---
name: haci-coordinator-debug
description: >-
  Diagnose and fix runtime problems in a Home Assistant custom integration — entities showing "unavailable" or
  "unknown", stale data, the integration failing to set up, "Config entry not ready", repeated reauth prompts,
  coordinator update failures, timeouts, or blocking-call warnings. Use when asked to "debug", "why is my sensor
  unavailable", "the integration won't load", "data isn't updating", "check the logs", "restart Home Assistant", or
  when investigating the coordinator, the API client or the setup path. Covers reading the log, querying a running
  instance, why `.storage/` is not live state, and the coordinator failure contract. SYMPTOMS — load this if you are
  about to: invent your own `hass`/`pytest` command where the project has one; read or write `.storage/` while Home
  Assistant is running; ask the developer to read a state from the UI when the API can answer; return None from
  `_async_update_data` to signal failure; or catch Exception broadly in the coordinator.
---

# Debug the running integration

Work from evidence, not from guesses. The order below is deliberate: reproduce, read the log, localise to a layer, then
fix.

**Read [`references/rules.md`](references/rules.md) first** — it holds the rules for the coordinator and the API
client: the layering, sessions, timeouts and return values, authentication, the exception hierarchy and how
`_async_update_data` maps it, update intervals, push versus pull, the first refresh, and caching. Before editing
`diagnostics.py`, read [`references/rules.diagnostics.md`](references/rules.diagnostics.md) — redaction and what never
goes into the output. A project may already load the same text as path-scoped instructions files. This skill is the
run loop and the four coordinator failures no exception-mapping table can express.

**The project wins.** Its `AGENTS.md` (or `CLAUDE.md`, `CONTRIBUTING.md`) may name how to start and restart the
development instance, a tool that reads it, where its log and configuration directory are, and its lint, type-check
and test commands; use those, and its rules on sharing the instance with the developer. Where it names none, the
defaults below apply.

| File                                                     | When to read                                                                          |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| [`references/async-rules.md`](references/async-rules.md) | A blocking-call warning, a thread-safety error, or a hang with no traceback behind it |

## 1. Reproduce with a clean instance

Start Home Assistant the way the project does. A hand-rolled `hass` or `pip` invocation that misses the virtualenv,
`PYTHONPATH`, port cleanup or debugger setup is the single most common cause of agents getting stuck. Where the project
names nothing, run `hass --config <config directory>` from the virtualenv Home Assistant is installed in.

Restart after **any** change to Python files, `manifest.json`, `services.yaml`, translations, or the config flow.
Reloading the config entry is the shortcut when the change is confined to entry data or options; it does **not**
re-import Python, so it is never a substitute for a restart after a code edit.

### The instance may be shared

A developer often restarts Home Assistant, clicks through the UI and tries things out while you work. **Never carry the
instance's run state from one step to the next. Observe it.**

- **A changed run state is the expected case, not a fault.** Do not go through the process tree or the log looking for
  an explanation — re-check once, adapt, and carry on. Reconciling who started what is never the task.
- **Do not kill an instance someone else started** to get your own, and never restart to tidy up. Restart when a change
  forces it, and say that you are doing it.
- **Announce it before you take exclusive control** — a reset, a token rotation, a loop that restarts repeatedly.
- **The Home Assistant version is the developer's choice.** Switch the environment only when asked, and name the active
  one when you finish.
- **Calling an action, walking a config flow, reloading an entry or changing a log level changes what the developer
  sees.** Cheap and reversible, but a config entry created by a flow stays until someone removes it.
- **State the run state when you finish — after checking it**, never from memory.

### Reading a running instance

Entity states, config entry status, diagnostics and the error log can all be read without asking the developer to look
something up. Use the project's tool when it has one. Otherwise, use the API with a long-lived access token — created
under the user's profile, **Security** > **Long-lived access tokens** — sent as `Authorization: Bearer <token>`. Keep
the token in an environment variable, never on a command line or in output.

| Question                                  | REST (`http://<host>:8123`)                                                      | WebSocket (`/api/websocket`)             |
| ----------------------------------------- | -------------------------------------------------------------------------------- | ---------------------------------------- |
| What is this entity's state right now     | `GET /api/states/<entity_id>`, all: `GET /api/states`                            | `get_states`                             |
| Did the entry load, and why not           | `GET /api/config/config_entries/entry?domain=<domain>` — `state` and `reason`    | `config_entries/get`                     |
| What unique IDs and registry flags it has | —                                                                                | `config/entity_registry/list`            |
| What the coordinator actually hands out   | `GET /api/diagnostics/config_entry/<entry_id>`, when diagnostics are implemented | —                                        |
| Did an error happen                       | `GET /api/error_log` (raw, admin only)                                           | `system_log/list` (deduplicated, counts) |
| Whether a value changes over time         | —                                                                                | `subscribe_events` on `state_changed`    |
| Reload the entry after an options change  | `POST /api/config/config_entries/entry/<entry_id>/reload`                        | —                                        |
| Raise a log level without a restart       | `POST /api/services/logger/set_level`                                            | `call_service`                           |

Ask the developer to use the UI only when there is no API access — **Settings** > **Devices & services** for the entry
and its reason, **Settings** > **System** > **Logs**, and **Download diagnostics** in the integration's menu.

### Ask the instance, or read the files?

There are two sources of truth for runtime state, and which one is correct depends only on whether Home Assistant is
running. Getting this backwards is the most common way to debug a ghost.

**While Home Assistant runs, the API is authoritative and the files under `.storage/` in the configuration directory
are stale by design** — they are `Store.async_delay_save` files, written some time after the change they describe.

| What                              | Written                                                          |
| --------------------------------- | ---------------------------------------------------------------- |
| entity / device / area registry   | 10 s after a change — but **180 s during startup**               |
| config entries                    | 1 s after a change                                               |
| `core.restore_state`              | every 15 minutes, and on shutdown                                |
| `home-assistant_v2.db` (recorder) | every `commit_interval` seconds — 5 s unless the project sets it |

So a registry entry you just created can be missing from `core.entity_registry` for three minutes, and **live state is
in none of these files at all** — `core.restore_state` is a periodic snapshot, and the recorder holds history, not the
present.

Reading the raw files is right when Home Assistant is **stopped**: post-mortem analysis, checking what a config entry
migration actually wrote, or diffing `.storage/` before and after a change. Then they are the only source, and they are
accurate.

**Never write into `.storage/` while Home Assistant is running.** It holds every store in memory and rewrites the whole
file on its next save, so your edit is discarded — silently, minutes later. Anything that must write there takes Home
Assistant's own `.ha_run.lock` first. `.storage/auth` and any stored access token are credentials, not debugging
material: do not read them.

## 2. Read the log

Start with the structured log (`system_log/list`, or the project's tool for it) — it is deduplicated, counts repeats,
and keeps each traceback attached to its record. Widen from errors to warnings. The raw stream is in
`home-assistant.log` in the configuration directory, and the previous run in `home-assistant.log.1`; that is where the
startup order and a traceback's surrounding context are.

```bash
rg -n "<domain>|Traceback|ERROR|WARNING" <config directory>/home-assistant.log | tail -50
```

Raise verbosity without restarting — `logger.set_level` with `custom_components.<domain>: debug` and
`homeassistant.helpers.update_coordinator: debug` takes effect immediately, and does not survive a restart. For a level
that should, put it in `configuration.yaml` and restart:

```yaml
logger:
  default: warning
  logs:
    custom_components.<domain>: debug
```

Read the **first** error in a cascade, not the last. A wall of "Error doing job" usually has one root cause above it.

Two debug modes catch what no log level shows: Home Assistant's own (`homeassistant: debug: true`), and asyncio's. With
both on, calling an `async_*` API from the wrong thread and blocking the event loop are reported the moment they happen,
instead of surfacing later as an unexplained hang or a state that quietly went wrong.

A `debug_console:` line in `configuration.yaml` adds a Python console under **Settings** > **Tools**, with `hass` in
scope — quicker than a log line and a restart for inspecting live objects. It runs in the event loop, so never call
anything blocking there.

**A config flow missing after a restart is usually the frontend cache, not the code.** Hard-refresh the browser before
debugging the flow.

## 3. Localise the failure

Each symptom has one piece of evidence that settles it, so you are reading state rather than guessing at it.

| Symptom                                           | Evidence                                  | Layer to inspect                                                                      |
| ------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- |
| Integration missing from the add-integration list | the log at startup                        | `manifest.json`, import error at module load                                          |
| "Config entry not ready, retrying"                | the entry's `state` and `reason`          | `_async_update_data` / `_async_setup` raising `ConfigEntryNotReady`                   |
| Entry loads, all entities `unavailable`           | the entities' states                      | coordinator's first refresh failed, or `last_update_success` false                    |
| Entities available but values `unknown`/`None`    | one entity's state and attributes         | key mismatch between `coordinator.data` and the entity's read path                    |
| Values never change                               | `state_changed` events over a few minutes | `update_interval`, caching in the API client, or a swallowed error                    |
| Endless reauth prompts                            | the entry's `state` and `reason`          | `ConfigEntryAuthFailed` raised for a non-auth failure                                 |
| Config flow rejects valid input                   | walking the flow                          | see [`haci-config-flow`](../haci-config-flow/SKILL.md)                                |
| "Detected blocking call inside the event loop"    | warnings in the log                       | sync I/O in async code — see [`references/async-rules.md`](references/async-rules.md) |
| Entity duplicated after an update                 | the entity registry                       | `unique_id` changed — see `haci-breaking-changes` (plugin `haci-quality`)             |

The entry's `reason` is the same text the UI shows, which is usually the whole diagnosis for a setup failure.

Watching `state_changed` deserves care: Home Assistant fires it only when the state or an attribute genuinely changes,
so an empty watch is a finding, not a failed measurement. It means the coordinator is handing back identical data, not
that notification is broken.

Cross-check the actual payload rather than assuming its shape:

```python
LOGGER.debug("Coordinator data: %s", self.data)
```

The config entry's diagnostics show the same payload without an edit and a restart, whenever the integration
implements them.

## 4. Check the failure contract

`_async_update_data` communicates through exception type, and getting it wrong is the root of most availability bugs.
Read the coordinator against the mapping table in [`references/rules.md`](references/rules.md) and check for the four
failures that table cannot express:

- **Signalling failure by returning** `None` or an empty dict instead of raising. Entities then show `unknown` forever
  instead of going unavailable, and nothing retries.
- **A broad `except Exception`** that swallows the auth error, so reauth never triggers and the entry just looks broken.
  The OAuth2 helper's token errors are the same trap — see the rules' authentication section.
- **Logging on every failed poll.** The coordinator logs the first failure and then stays quiet by design
  (`log-when-unavailable`, Silver). Manual logging buries the real error in repetition.
- **`ConfigEntryNotReady` raised outside setup**, or `async_config_entry_first_refresh()` called outside setup.

## 5. Common fixes

**Update interval too aggressive** — the Bronze `appropriate-polling` rule. Local devices ~30 s, cloud services
~5–15 min, fixed in code — never a user option (see [`haci-config-flow`](../haci-config-flow/SKILL.md)).

**Timeouts** — every request needs one, with `asyncio.timeout`, never a bare `await`:

```python
async with asyncio.timeout(10):
    response = await self._session.get(url)
```

**Partial data** — when only part of the payload is missing, keep the previous value instead of dropping the whole
update; only raise when the update is genuinely useless.

**Entity reads the wrong key** — compare the `EntityDescription.key` / `value_fn` against the logged payload.

**Setup ordering** — expensive one-off work (fetching device metadata, capabilities) belongs in `_async_setup()`, which
runs once before the first refresh, not in `_async_update_data`.

**A listener or reconnect loop that outlives an unload** — start it with
`entry.async_create_background_task(hass, coro, name)`, which is cancelled when the entry unloads and does not hold up
startup. `hass.async_create_task` survives the unload and keeps running against a dead entry.

## 6. Confirm the fix

Run the project's lint, type-check and test commands. Then restart, reproduce the original scenario, and confirm:

- the entry's state is `loaded`, with no reason;
- the entities show the values you expected;
- the log has nothing new at warning level or above.

If the failure was data-shaped, check the payload directly — the diagnostics `data` should now show the corrected
structure. Diagnostics must run everything through `async_redact_data()`; if you added a field, make sure it is
redacted when sensitive ([`references/rules.diagnostics.md`](references/rules.diagnostics.md)).

## 7. Add a regression test

A runtime bug that reached a user is exactly the case where a test pays for itself. Add one that reproduces the
original failure before the fix. See `haci-testing` (plugin `haci-quality`).

## Stop conditions

After three failed attempts at the same error, stop and report what you tried and what you observed. Do not keep
looping — a wrong mental model does not improve with repetition.

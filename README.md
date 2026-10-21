# Home Assistant Light Control

Control Home Assistant lights and switches from OctoPrint, with brightness, colour,
print status lighting and delayed shutoff. This is the maintained
[idontwantfame fork](https://github.com/idontwantfame/OctoLight_Home-Assistant).

Originally based on [Mark Bloom’s plugin](https://github.com/mark-bloom/OctoLight_Home-Assistant),
[OctoLight by gigibu5](https://github.com/gigibu5/OctoLight) and Home Assistant integration
ideas from [PSUControl](https://github.com/kantlivelong/OctoPrint-PSUControl).

## Installation and upgrade

Requires Python 3.9 or later. Tested with OctoPrint 1.11.8 / Python 3.9 and 3.10, and
OctoPrint **2.0.0rc7** / Python 3.12, including real local server startup and API checks.
OctoPrint 2 is currently verified against this release candidate; repeat the checks
when upgrading to later releases. CI repeats these three environments.

| Component | Requirement / listing compatibility | Validation |
| --- | --- | --- |
| Python | `>=3.9,<4` | 3.9, 3.10 and 3.12 |
| OctoPrint | `>=1.11.8,<3` | 1.11.8 and 2.0.0rc7; later releases require verification |
| Operating system | Linux | Linux CI and local Linux tests; Windows/macOS are not verified |
| Home Assistant | States/services REST API and a long-lived access token | API simulator; no Home Assistant version or physical light is certified |

Supported entity domains are `light`, `switch` and `input_boolean`. Brightness,
RGB colour and short flashes depend on capabilities reported by each light.

In OctoPrint’s Plugin Manager, install this URL, then restart OctoPrint:

```text
https://github.com/idontwantfame/OctoLight_Home-Assistant/archive/refs/heads/master.zip
```

This is a separate plugin with its own identity:

| Identity | Value |
| --- | --- |
| Display name | Home Assistant Light Control |
| OctoPrint identifier | `ha_light_control` |
| Python package | `octoprint_ha_light_control` |
| Distribution | `OctoPrint-HA-Light-Control` |
| HTTP API | `/api/plugin/ha_light_control` |
| Host command | `@HALIGHT` |

The upstream plugin remains `octolightHA` and is not replaced or uninstalled.
Both plugins can be installed, but avoid enabling duplicate automation for the same
light. Each has its own settings, navbar bindings, API and host command.

To migrate an existing OctoLight configuration:

1. Disable **OctoLight Home Assistant** and restart OctoPrint.
2. Open **Home Assistant Light Control** settings before entering new connection fields.
3. Select **Import OctoLight settings**. Known settings are copied into this plugin;
   the original configuration is left intact. Import refuses to overwrite a configured
   destination or run while the original plugin is enabled.
4. Update scripts from `@OCTOLIGHT` to `@HALIGHT` and API callers from
   `/api/plugin/octolightHA` to `/api/plugin/ha_light_control`.

Existing implicit TLS verification preferences are preserved when importing; new
configurations verify certificates by default. Automatic updates track this fork’s
GitHub releases. Local commits and build artifacts are not published releases.

**API migration:** GET requests do not change lights. Integrations using
`GET ?action=toggle`, `turnOn` or `turnOff` must switch to POST commands on the new
endpoint. Plain GET safely reads cached state.

## Setup

1. Open **Settings → Home Assistant Light Control**.
2. Enter the Home Assistant URL, including `http://` or `https://` and its port when
   needed, for example `http://homeassistant.local:8123`. Trailing slashes are accepted.
3. Enter a Home Assistant long-lived access token. The token is masked in the UI,
   restricted to OctoPrint users with Settings permission, and never logged by the plugin.
4. Select **Search Home Assistant**. Search by friendly name or entity ID, then select
   the result. If exactly one supported entity exists and the ID is blank, it is filled
   automatically. Multiple matches are never guessed.
5. Select **Test connection and entity**, then save. Discovery and testing use the
   unsaved connection fields and do not turn anything on or off.

Use complete IDs: `light.printer`, `switch.desk_lamp` or `input_boolean.printer_light`.
Sensors, scenes and arbitrary service domains are not controllable through this plugin.
Home Assistant’s token must grant access to the relevant entities and actions.

Certificate verification applies to HTTPS; it does not need disabling for HTTP.
For a self-signed server, install a trusted certificate or explicitly disable verification.
Redirects are rejected: configure the final Home Assistant URL.

## Controls and state

Each saved entity has a navbar icon. Click the icon to toggle it; open its adjacent
menu for brightness and colour controls supported by that light. Choose a lightbulb
or power icon in settings. Additional lights and switches get independent icons,
print automation options and shutoff timers (up to 20 additional entities).

A single Home Assistant state-list request polls every 10 seconds by default
(configurable from 5 to 3600 seconds). Changes made in Home Assistant or elsewhere
are broadcast to connected OctoPrint browsers. Network calls have connection/read
timeouts and do not run on OctoPrint’s event or serial command dispatch threads.

Unavailable, unknown and failed states appear faded with an explanation; they are
not reported as off. A service call can succeed before a Zigbee/MQTT device updates
its state. The icon shows the state Home Assistant actually reports and catches up
on the next poll. The plugin never invents a successful state or retries a toggle.

Logged-in users need **Status** permission to read state and **Control** permission
to operate lights. Discovery and configuration testing require **Settings** permission.
Browser requests use OctoPrint’s client, including its authentication and CSRF support.

## Brightness, colour and print automation

Enable optional turn-on brightness (1–100%) or colour defaults in settings. These
preferences apply to supported light capabilities, including additional lights.
Switches receive only on/off commands. Lights with only brightness or colour-temperature
support do not receive RGB colour values.

Print automation can turn each entity on at print start, off on completion after a
0–1440 minute delay, and off on failure or cancellation. A new print cancels previous
completion timers even when automatic turn-on is disabled. Manual controls, settings
changes and shutdown also cancel pending shutoff. Timers check that a print is neither
running nor paused before switching off. Turn-on/off automation uses explicit actions,
not cached-state toggles.

**Print status colours** are opt-in and turn colour-capable lights on when the relevant
event occurs. Each colour is configurable:

| Event | Default colour |
| --- | --- |
| Print started / resumed | White |
| Print paused | Amber |
| Print completed | Green |
| Print failed / printer error | Red |
| Print cancelled | Orange |

Optional **short flash on error** uses the light’s native Home Assistant flash capability.
It is skipped when unsupported; there is no repeating software blink loop. Flash duration
and appearance depend on the light’s integration. Failure/cancellation shutoff takes
priority over status colours. Completion colours stay until the configured completion
shutoff. On resume/start, the printing colour replaces the previous status colour.

## OctoDash and G-code

Enable **@HALIGHT host commands** in settings, then use them in OctoDash custom
actions, OctoPrint’s Terminal tab, G-code scripts or streamed G-code files:

```gcode
@HALIGHT on
@HALIGHT off
@HALIGHT toggle
@HALIGHT on light.room
@HALIGHT set brightness=75
@HALIGHT set color=#ff8000
@HALIGHT set light.room brightness=75 color=#ff8000
@HALIGHT set light.printer color=#ff0000 flash=short
@HALIGHT on brightness=100 color=#ffffff
```

| Action | Behaviour |
| --- | --- |
| `on [entity_id] [options]` | Turn on, optionally applying brightness, colour or a flash. |
| `off [entity_id]` | Turn off. |
| `toggle [entity_id]` | Toggle on/off. |
| `set [entity_id] options` | Turn on and apply one or more appearance options. |

Appearance options are `brightness=1` through `brightness=100` (percent),
`color=#RRGGBB` and `flash=short`. Combine them as needed. These use the same
capability checks as the UI/API; unsupported attributes are rejected. Off/toggle
commands do not accept appearance options. An omitted entity means the primary
configured entity; an explicit ID must already be configured. Semicolon comments
are allowed. Invalid commands are logged without sending a light command.

Commands run asynchronously and do not block OctoPrint’s serial thread or wait
for the printer’s motion buffer. They are host commands: direct printer-SD-card
printing does not process them. Custom M-code aliases are unnecessary and are not
intercepted by this plugin.

## HTTP API

The endpoint is `/api/plugin/ha_light_control`. Use a user/application API key with the
required permissions, or an authenticated session with OctoPrint’s CSRF token.
OctoPrint 2 clients should not rely on the removed global API key.

`GET /api/plugin/ha_light_control` returns cached state without sending a light command:

```json
{"state": true, "isLightOn": true, "entities": [{"entity_id": "light.printer", "isLightOn": true, "available": true}]}
```

`state` and `isLightOn` refer to the primary entity and can be `null` when unknown.
Entity records additionally include name, error, attributes and capability flags.
`GET ?action=getState` is accepted; mutation actions return HTTP 405.

Send JSON POST requests:

```json
{"command": "toggle"}
{"command": "turnOn", "entity_id": "switch.room"}
{"command": "turnOff"}
{"command": "setLight", "brightness": 80, "color": "#ff9900"}
```

`setLight` turns the light on. Optional `brightness`, `color` and `flash: "short"`
are capability checked; unsupported manual attributes return a JSON error.
The target defaults to the primary entity and must be configured. Successful controls
return the same state structure as GET. Errors contain an actionable `error` string.

Admin setup commands are `discover` (returns supported entities) and `test` (reads an
entity’s state). Both accept optional unsaved `address`, `api_key` and
`verify_certificate` fields; `test` also accepts `entity_id`. Credentials stay server-side
when contacting Home Assistant and are omitted from plugin messages and command responses.

## Development and validation

Builds use `pyproject.toml` and isolated setuptools; OctoPrint is not a build dependency.
The package name, module and plugin identifier are distinct from the upstream plugin.

```sh
python -m pip install "OctoPrint==2.0.0rc7" ".[develop]"
python -m pytest -q
ruff check .
ruff format --check .
python -m build
npm ci
npx playwright install --with-deps chromium
npm test
```

`npm test` exercises both templates in Chromium with the installed OctoPrint’s bundled
jQuery, Bootstrap and Knockout, covering discovery, autofill, permissions, external state,
live controls and errors. Set `PYTHON` if its interpreter differs from `python`.

To verify installed-package loading, authentication, permissions, CSRF, settings,
colour/brightness payloads and polling against a disposable local OctoPrint server:

```sh
python -m pip install --force-reinstall --no-deps dist/*.whl
HALIGHT_SERVER_TEST=1 python -m pytest tests/test_server.py -q
```

This test uses temporary users, config, localhost ports and a simulated Home Assistant;
it does not access a physical printer or your Home Assistant. The CI workflow repeats
these checks on OctoPrint 1 and 2. Physical device behaviour, especially flash and colour
reproduction, still needs checking with the relevant Home Assistant integration.

## GitHub automation and plugin-repository listing

The test workflow runs on master pushes, pull requests, manual dispatch and weekly.
It checks OctoPrint 1.11.8 with Python 3.9/3.10 and OctoPrint 2.0.0rc7 with Python
3.12, builds isolated distributions, runs installed-package server tests and checks
the UI in Chromium. Actions are pinned and Dependabot checks GitHub Actions, Python
and browser-test dependencies weekly. Workflow changes are checked by actionlint.

Pushing a `v*` tag runs the same tests, verifies that the tag matches the version
in `pyproject.toml`, checks distribution contents and creates a **draft** GitHub
release with notes from the matching `CHANGELOG.md` entry, the wheel, source archive
and SHA256 checksums. A missing or empty entry stops the release. It uses GitHub’s built-in
token; no PyPI token is needed. Review the draft and publish it when ready. For
example, version `0.3.0` requires tag `v0.3.0`. No release is created by ordinary
branch pushes. On a GitHub fork, enable Actions if GitHub has disabled them.

For a separate plugin-repository listing, register `ha_light_control` with the
name **Home Assistant Light Control**, Python compatibility `>=3.9,<4`, and the
tested OctoPrint range `>=1.11.8,<3`. Use this fork’s source and release archive,
enable its GitHub Issues, declare Linux as the verified operating system, and
include current screenshots and the required `ai-developed` attribute. See the
[registration requirements](https://plugins.octoprint.org/help/registering/).
Before submission, publish the tested release, enable this fork’s Issues and
replace the inherited screenshots with current UI images. Create the registration
file in the plugin-repository project, using this README’s identity and support
information; registration drafts are not kept in this working repository.
No listing or release has been published from this working repository.

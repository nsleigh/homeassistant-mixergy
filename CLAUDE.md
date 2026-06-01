# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Home Assistant custom integration (HACS) for Mixergy smart hot water tanks. It polls the Mixergy cloud API (`https://www.mixergy.io/api/v2`) every 30 seconds to expose tank state as HA entities.

## Architecture

The integration is structured as a standard HA custom component under `custom_components/mixergy/`:

- **`tank.py`** — The core `Tank` class that handles all Mixergy API communication. It authenticates via a multi-step REST flow (root → account → login), then discovers tank URLs (measurement, control, settings, schedule) dynamically from the API's HATEOAS `_links` structure. All HTTP calls use HA's `aiohttp_client` session.

- **`__init__.py`** — Integration entry point. Creates a `Tank` instance and wraps it in a `DataUpdateCoordinator` (30s interval). Registers all HA services (`mixergy_set_charge`, `mixergy_set_target_temperature`, `mixergy_set_holiday_dates`, `mixergy_clear_holiday_dates`, `mixergy_set_default_heat_source`).

- **`mixergy_entity.py`** — `MixergyEntityBase(CoordinatorEntity)` base class shared by all entity types. Handles device_info, availability, and callback registration with the `Tank` instance.

- **`sensor.py`** — Sensor and binary sensor entities. `EnergySensor` and `PVEnergySensor` extend HA's `IntegrationSensor` to derive kWh from power readings. PV-related sensors gate their `available` property on `tank.has_pv_diverter`.

- **`switch.py`** — Switch entities for DSR (grid assistance), frost protection, distributed computing (medical research), and PV divert.

- **`number.py`** — Number entities for target temperature (45–55°C), target charge (0–100%), cleansing temperature (51–55°C), and PV-related settings.

- **`config_flow.py`** — UI config flow collecting username, password, and serial number; validates by attempting authentication and tank lookup.

- **`const.py`** — Service and attribute name constants.

## Key Design Patterns

- The `Tank` class stores URLs discovered from the API at runtime (`_latest_measurement_url`, `_control_url`, `_settings_url`, `_schedule_url`). These are only populated after `fetch_tank_information()` runs.
- Authentication token is cached in `self._token`; re-auth only happens when token is empty. Token expiry is not currently handled.
- PV diverter features (`has_pv_diverter`) are conditional on the tank's `configuration.mixergyPvType` field not being `"NO_INVERTER"`.
- The settings API returns `text/plain` content-type despite being JSON — it must be loaded via `resp.text()` + `json.loads()`, not `resp.json()`.
- Holiday mode is set/cleared by fetching the full schedule, modifying the `holiday` key, and PUTting it back.

## CI Validation

Two GitHub Actions workflows run on push and PR:
- **HACS validation** (`hacs.yml`) — validates HACS integration metadata
- **hassfest** (`validate_hassfest.yml`) — validates HA integration structure

There are no automated Python tests. The `testing/mixergy.py` script is a standalone CLI tool for manually exercising the Mixergy API:

```bash
cd testing
pip install -r requirements.txt
python mixergy.py <username> <password>                    # list all tanks
python mixergy.py <username> <password> <serial_number>   # inspect specific tank
```

## Version

Current integration version: `0.9.0` (in `manifest.json`)

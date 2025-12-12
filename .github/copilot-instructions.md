# Copilot Instructions for Pstryk Energy Meter

## Project Overview
This is a **Home Assistant custom component** (integration) that provides sensor entities for Pstryk Energy Meter devices via local HTTP API. It's a "local polling" integration that continuously fetches device state and exposes electrical measurements (voltage, current, power, energy) as Home Assistant sensors.

## Architecture & Data Flow

### Core Components
- **`__init__.py`**: Entry point managing lifecycle (`async_setup`, `async_setup_entry`, `async_unload_entry`) and version migrations
- **`config_flow.py`**: User configuration UI using voluptuous schema; validates device connectivity at `/info` endpoint
- **`sensor.py`**: Entity definitions and coordinator pattern for polling; contains 4 sensor classes (Voltage, Current, Power, Energy)
- **`const.py`**: Domain constants and metadata

### Data Flow Pattern
1. **Config Flow** → User enters device name & host → HTTP GET `/info` endpoint validates device type/product
2. **Setup Entry** → Creates `PstrykEnergyMeterDataUpdateCoordinator` → Registers SENSOR platform
3. **Coordinator** → Polls `http://{host}/state` every 30 seconds → Parses `multiSensor.sensors` array into key-indexed dict
4. **Sensor Entities** → Subscribe to coordinator updates → Extract individual values via `src` key and unit conversions

### Key Design Pattern: Coordinator + Base Class
- **`PstrykEnergyMeterDataUpdateCoordinator`**: Handles polling lifecycle, error tracking (`last_update_success`), device info
- **`PstrykEnergyMeterBaseSensor`**: Abstract base for all sensors; sets up listener subscription, availability tied to coordinator success
- **Specializations**: Voltage/Current/Power/Energy subclasses define unit of measurement and device class

## Device State Structure
The `/state` endpoint returns JSON like:
```json
{
  "multiSensor": {
    "sensors": [
      { "type": "voltage", "id": "1", "value": 2371 },
      { "type": "activePower", "id": "0", "value": 1234 }
    ]
  }
}
```
Keys format: `{type}_{id}` (e.g., `voltage_1`). Values may need unit conversion (voltage/10).

## Entity Creation Pattern
Entities are factory-created in `async_setup_entry`:
- **Per-phase** sensors: 1, 2, 3 (voltage, current, active power, energies)
- **Total** sensors: 0 (e.g., `activePower_0` = total active power)
- Each gets unique ID: `{entry_id}_{sensor_id}`

## Configuration Migration
- **Version 1 → 2**: Renamed config key `"hostname"` → `CONF_HOST` (see `async_migrate_entry` in `__init__.py`)
- Version check prevents forward migrations

## Home Assistant API Usage Conventions
- **Async patterns**: `async_add_executor_job(partial(...))` for blocking I/O (requests, HTTP)
- **Device registry**: `DeviceInfo` with identifiers, model/version from `/info` response
- **State classes**: `MEASUREMENT` for instantaneous values; `TOTAL_INCREASING` for cumulative energy
- **Availability**: Entities automatically unavailable if coordinator polling fails

## Critical Dependencies & Endpoints
- **Python**: `requests` for HTTP (blocking I/O via executor)
- **Home Assistant**: Core, config_entries, sensor components, device registry
- **Device API**:
  - `GET /info`: Returns device metadata (id, product, hv, fv); called once at setup + config flow
  - `GET /state`: Returns live sensor readings; polled every 30 seconds
  - Validation: Checks `device.product == "PstrykEnergyMeter"` and `device.type == "multiSensor"`

## Adding New Sensors
1. Create new sensor class inheriting `PstrykEnergyMeterBaseSensor`
2. Set `_attr_device_class` and `_attr_native_unit_of_measurement` (e.g., `SensorDeviceClass.FREQUENCY`)
3. Implement `native_value` property extracting from `self._coordinator.data[self.src]`
4. Instantiate in `async_setup_entry` entity list with `{type}_{id}` key pattern
5. Add to `MULTISENSOR_TYPES` constant if supporting dynamic discovery

## Common Patterns
- **Unit conversions**: Voltage divides by 10 (`value / 10`); power/current/energy values used as-is
- **Polling interval**: Fixed 30s (see TODO for user-configurable option)
- **Error handling**: `UpdateFailed` exception from coordinator pauses entity updates, triggers unavailable state
- **Logging**: Debug logs at setup phases; `_LOGGER` is module-level instance

## TODOs & Known Limitations
- Separate coordinator for `/info` to poll firmware updates (currently only at setup)
- User-configurable polling interval with lower bound validation
- Dynamic sensor discovery from `MULTISENSOR_TYPES` not yet implemented

# Home Energy & Object Registry — Entity Model

> Domain: Residential energy upgrade (insulation, solar PV, solar thermal, heat pump, battery, IoT controls)
> Design goal: Treat every real-world "thing" as a typed entity with stable IDs, typed relationships, and domain roles.

---

## 1. Entity Type Taxonomy

```
Entity
├── Actor
│   ├── Person
│   └── Animal
├── Asset
│   ├── Vehicle
│   ├── Device
│   │   ├── EnergySource       (solar PV array, solar thermal collector)
│   │   ├── EnergyStorage      (battery, hot-water tank, buffer vessel)
│   │   ├── EnergyConverter    (heat pump, inverter, MPPT controller)
│   │   ├── EnergyDistributor  (consumer unit, distribution board, EV charger)
│   │   ├── Sensor             (temp, humidity, CO2, current, voltage, flow, occupancy)
│   │   ├── Actuator           (relay, valve, thermostat, motorised damper)
│   │   └── Gateway            (MQTT broker host, LoRa gateway, smart meter)
│   └── Structure
│       ├── Zone
│       └── Boundary           (wall, roof, floor — carries insulation data)
├── Service
│   ├── UtilityService         (grid electricity, gas, water supply, sewage)
│   ├── TariffPlan             (day/night rates, export rates, FIT/SEG)
│   └── MaintenanceContract
└── Control
    ├── Schedule               (time-based rule)
    ├── Automation             (condition → action rule)
    └── Scene                  (named multi-device state snapshot)
```

---

## 2. Core Entity Schema

Every entity shares a common base record:

```json
{
  "id":          "uuid-v4",
  "type":        "<EntityType>",
  "subtype":     "<EntitySubtype>",
  "label":       "Human-readable name",
  "description": "Optional free text",
  "tags":        ["tag1", "tag2"],
  "created_at":  "ISO-8601",
  "updated_at":  "ISO-8601",
  "status":      "active | inactive | decommissioned"
}
```

---

## 3. Entity Type Definitions

### 3.1 Person

```json
{
  "type": "Actor",
  "subtype": "Person",
  "label": "<resident-name>",
  "role": ["owner", "admin", "resident"],
  "presence_zone_id": "<zone-id>",
  "devices_owned": ["<device-id>"]
}
```

**Roles:** `owner` | `resident` | `guest` | `contractor` | `occupant`

---

### 3.2 Animal

```json
{
  "type": "Actor",
  "subtype": "Animal",
  "label": "<animal-name>",
  "species": "<species>",
  "zones_permitted": ["<zone-id>"]
}
```

---

### 3.3 Vehicle

```json
{
  "type": "Asset",
  "subtype": "Vehicle",
  "label": "<vehicle-label>",
  "vehicle_type": "BEV",
  "battery_capacity_kwh": 0.0,
  "charge_port": "CCS2",
  "charger_device_id": "<device-id>",
  "home_zone_id": "<zone-id>"
}
```

**Vehicle types:** `BEV` | `PHEV` | `ICE` | `HEV` | `Cargo`

---

### 3.4 Device

All devices extend the base entity with:

```json
{
  "type": "Asset",
  "subtype": "Device",
  "device_class": "EnergySource | EnergyStorage | EnergyConverter | EnergyDistributor | Sensor | Actuator | Gateway",
  "manufacturer": "<manufacturer>",
  "model": "<model>",
  "serial_number": "<serial>",
  "firmware_version": "<version>",
  "install_date": "ISO-8601",
  "warranty_expiry": "ISO-8601",
  "zone_id": "<zone-id>",
  "parent_device_id": "<device-id | null>",
  "protocol": "MQTT | Modbus | SunSpec | KNX | Zigbee | Z-Wave | LoRa | HTTP | Proprietary",
  "topic_prefix": "home/energy/...",
  "rated_power_w": 0,
  "rated_energy_kwh": 0,
  "telemetry_schema": "<schema-id>"
}
```

#### Device subtype examples

| label | device_class | protocol | rated_power_w |
|---|---|---|---|
| Solar PV Array | EnergySource | SunSpec/Modbus | 6000 |
| Solar Thermal Collector | EnergySource | Proprietary | — |
| Battery Storage | EnergyStorage | MQTT/CAN | — |
| Hot Water Tank | EnergyStorage | 1-Wire/MQTT | — |
| Air Source Heat Pump | EnergyConverter | Modbus/MQTT | 3500 |
| Hybrid Inverter | EnergyConverter | SunSpec | 5000 |
| Main Consumer Unit | EnergyDistributor | — | — |
| EV Charger | EnergyDistributor | OCPP/MQTT | 7400 |
| Smart Meter | Gateway | SMETS2/HAN | — |
| LoRa Gateway | Gateway | LoRa/MQTT | — |
| CT Clamp Sensor | Sensor | MQTT | — |
| Room Thermostat | Actuator | MQTT | — |
| Motorised Zone Valve | Actuator | MQTT | — |

---

### 3.5 Zone

Zones form a strict hierarchy. Every physical space is a zone; zones can contain sub-zones.

```json
{
  "type": "Asset",
  "subtype": "Zone",
  "label": "Living Room",
  "zone_class": "room | floor | building | outdoor | virtual",
  "parent_zone_id": "<zone-id | null>",
  "area_m2": 0.0,
  "volume_m3": 0.0,
  "floor_level": 0,
  "heating_circuit_id": "<device-id>",
  "ventilation_circuit_id": "<device-id>",
  "occupancy_sensor_ids": ["<device-id>"],
  "temp_sensor_ids": ["<device-id>"],
  "set_point_heating_c": 20.0,
  "set_point_cooling_c": null
}
```

#### Default zone hierarchy

```
Building (Home)
├── Ground Floor
│   ├── Hall
│   ├── Kitchen / Dining
│   ├── Living Room
│   ├── Utility Room
│   └── WC
├── First Floor
│   ├── Landing
│   ├── Bedroom 1 (Master)
│   ├── Bedroom 2
│   ├── Bedroom 3
│   └── Bathroom
├── Roof Space / Attic
├── External
│   ├── Front Garden
│   └── Rear Garden
└── Virtual
    ├── PV Generation Circuit
    ├── Battery Circuit
    └── Heat Pump Circuit
```

---

### 3.6 Boundary (Thermal Envelope)

```json
{
  "type": "Asset",
  "subtype": "Boundary",
  "label": "Rear Roof — Rafter Insulation",
  "boundary_class": "roof | external_wall | internal_wall | floor | window | door",
  "zone_ids": ["<zone-id-inside>", "<zone-id-outside>"],
  "area_m2": 0.0,
  "u_value_wm2k": 0.18,
  "insulation_layers": [
    { "material": "mineral wool", "thickness_mm": 200, "lambda": 0.035 }
  ],
  "air_permeability_m3hm2": 3.0,
  "upgrade_date": "ISO-8601"
}
```

---

### 3.7 Service

```json
{
  "type": "Service",
  "subtype": "UtilityService | TariffPlan | MaintenanceContract",
  "label": "<tariff-name>",
  "provider": "<provider-name>",
  "start_date": "ISO-8601",
  "end_date": "ISO-8601 | null",
  "rates": [
    { "period": "Day",    "rate_per_kwh": 0.0, "hours": "08:00-23:00" },
    { "period": "Night",  "rate_per_kwh": 0.0, "hours": "23:00-08:00" },
    { "period": "Export", "rate_per_kwh": 0.0, "hours": "all" }
  ],
  "standing_charge_per_day": 0.0,
  "currency": "EUR",
  "meter_device_id": "<device-id>"
}
```

---

### 3.8 Control

#### Schedule

```json
{
  "type": "Control",
  "subtype": "Schedule",
  "label": "Heat Pump Morning Boost",
  "cron": "0 6 * * *",
  "timezone": "<local-timezone>",
  "target_entity_ids": ["<device-id>"],
  "action": { "command": "set_mode", "params": { "mode": "heat", "setpoint_c": 21 } },
  "enabled": true
}
```

#### Automation

```json
{
  "type": "Control",
  "subtype": "Automation",
  "label": "Battery → EV charge when SOC > 80%",
  "trigger": {
    "entity_id": "<battery-device-id>",
    "attribute": "soc_pct",
    "condition": ">",
    "value": 80
  },
  "condition": {
    "entity_id": "<ev-charger-device-id>",
    "attribute": "car_connected",
    "equals": true
  },
  "action": {
    "entity_id": "<ev-charger-device-id>",
    "command": "start_charging",
    "params": { "rate_w": 3700 }
  },
  "enabled": true
}
```

#### Scene

```json
{
  "type": "Control",
  "subtype": "Scene",
  "label": "Away Mode",
  "states": [
    { "entity_id": "<heat-pump-id>",  "attribute": "setpoint_c",          "value": 15 },
    { "entity_id": "<hw-tank-id>",    "attribute": "legionella_schedule",  "value": "weekly" },
    { "entity_id": "<ev-charger-id>", "attribute": "mode",                 "value": "smart" }
  ]
}
```

---

## 4. Relationship Types

| Relationship | From | To | Notes |
|---|---|---|---|
| `owns` | Person | Device, Vehicle | legal ownership |
| `operates` | Person | Device, Control | operational access |
| `inhabits` | Person, Animal | Zone | current or habitual presence |
| `located_in` | Device, Vehicle | Zone | physical location |
| `contains` | Zone | Zone, Device | spatial containment |
| `bounds` | Boundary | Zone × Zone | separates two zones |
| `feeds` | Device | Device | energy/data flow direction |
| `controlled_by` | Device | Control | which automations govern device |
| `metered_by` | Service | Device | utility measured by meter |
| `charged_under` | Device | Service (TariffPlan) | energy cost assignment |
| `maintained_by` | Device | Service (MaintenanceContract) | service contract |

---

## 5. Telemetry Schema (per device_class)

Each device class publishes to MQTT with a canonical payload shape:

```
home/<building_id>/<zone_id>/<device_id>/state   → JSON telemetry
home/<building_id>/<zone_id>/<device_id>/cmd     → JSON command
home/<building_id>/<zone_id>/<device_id>/lwt     → last will (online | offline)
```

### EnergySource (solar PV)
```json
{ "power_w": 0, "energy_today_kwh": 0.0, "voltage_v": 0.0, "current_a": 0.0, "timestamp": "ISO-8601" }
```

### EnergyStorage (battery)
```json
{ "soc_pct": 0.0, "power_w": 0, "voltage_v": 0.0, "temp_c": 0.0, "cycles": 0, "timestamp": "ISO-8601" }
```

### EnergyConverter (heat pump)
```json
{ "mode": "heat", "cop": 0.0, "power_in_w": 0, "heat_out_w": 0, "flow_temp_c": 0.0, "return_temp_c": 0.0, "timestamp": "ISO-8601" }
```

### Sensor (room)
```json
{ "temp_c": 0.0, "humidity_pct": 0, "co2_ppm": 0, "occupancy": false, "timestamp": "ISO-8601" }
```

### Actuator (valve/relay)
```json
{ "state": "open | closed | position_pct", "last_cmd": "ISO-8601", "timestamp": "ISO-8601" }
```

---

## 6. Design Principles

1. **Stable IDs** — UUID v4, never reassigned even after decommission.
2. **Typed relationships** — all links are named and directional.
3. **Zone-first layout** — every physical entity anchors to a zone.
4. **Energy-flow graph** — `feeds` edges form a directed acyclic graph from sources → storage → converters → distributors → loads.
5. **Separation of telemetry and config** — live state lives in MQTT; registry holds static/config data only.
6. **Extensible subtypes** — new device classes add a `device_class` value without schema changes.
7. **Tariff-aware** — every consuming device can be linked to a tariff for cost attribution.

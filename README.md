# Aroha — Design Documents

Architectural and design documentation for the Aroha home energy and network management system.

## Structure

```
docs/
├── energy/
│   ├── home-energy-entity-model.md     — Entity registry model (people, devices, zones, services, controls)
│   └── house-agent-architecture.md     — Local LLM agent architecture for home energy management
└── network/
    └── network-management-agent-architecture.md  — Network management agent (Zabbix, SNMP, WOL)
```

## Companion Repository

Application code, schemas, YAML, and configuration templates live in [aroha-core](https://github.com/ArohaKiwi/aroha-core).

## Design Principles

- **Standards first** — all protocols are open and vendor-neutral (SNMP, MQTT, SSH, IEEE 802.3)
- **Local LLM** — all AI inference runs on-premises via Ollama; no cloud dependency for control decisions
- **Registry-grounded** — the agent never invents device IDs or sensor values; all references resolve to the entity registry
- **Separation of concerns** — design documents (this repo) are kept separate from application artifacts (aroha-core)

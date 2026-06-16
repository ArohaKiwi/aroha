# Network Management Agent Architecture
## Mixed Fleet: Linux · Windows · Raspberry Pi · Arduino

> Extends the house agent architecture to cover IT infrastructure — servers, workstations, and embedded nodes — with Wake-on-LAN control, resource monitoring, and LLM-driven management via local Ollama.

---

## Design Philosophy — Standards First

All monitoring, telemetry, and control protocols in this architecture are based on open, vendor-neutral standards. No proprietary lock-in at any layer.

| Layer | Standard / Protocol | Rationale |
|---|---|---|
| Node monitoring | SNMP v2c / v3 (RFC 3411–3418) | Universal, supported on every OS and most embedded platforms |
| Node agent | Zabbix Agent 2 (open protocol) | Open source (GPL v2), no licensing cost at any scale |
| Metrics transport | SNMP, Zabbix trapper (TCP/JSON), MQTT | All open protocols |
| Time-series storage | PostgreSQL / TimescaleDB | Open source RDBMS + time-series extension |
| Query | Zabbix REST API (JSON over HTTP) | Open, documented, no vendor SDK required |
| Control bus | MQTT (ISO/IEC 20922) | Open pub/sub standard |
| WOL | IEEE 802.3 Magic Packet | Open standard, implemented in every NIC |
| SSH control | OpenSSH (RFC 4251–4254) | Open standard, universally available |
| Embedded telemetry | SNMP (ESP32 via Arduino-SNMP) or MQTT JSON | Both open standards |

Future tool swaps (e.g. replacing Zabbix with OpenNMS or VictoriaMetrics + SNMP exporter) require only changes to the tool adapter layer — the agent loop, entity model, and MQTT bus are protocol-defined and unaffected.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT AGENT (Ollama)                         │
│          qwen2.5:14b  ·  num_ctx 32 000  ·  temp 0.0               │
│                                                                       │
│  System Prompt  │  Node Snapshot  │  Alert Digest  │  History        │
└────────┬────────────────┬────────────────┬──────────────────────────┘
         │ tool call      │ sub-agent      │ structured output
         ▼                ▼               ▼
   Tool Executor     Sub-agents      Response parser
         │
         ├── WOL Controller     (IEEE 802.3 magic packet via MQTT proxy)
         ├── Zabbix API Client  (JSON/HTTP — metrics, hosts, alerts, history)
         ├── SNMP Walker        (net-snmp · SNMPv2c/v3 · RFC 3411)
         ├── SSH Commander      (OpenSSH · RFC 4251 — Linux / Pi only)
         ├── Node Registry      (entity model — extended for compute nodes)
         └── MQTT Bridge        (ISO/IEC 20922 — Arduino/ESP32 telemetry)

                       │ SNMP polls (UDP/161)    │ MQTT (TCP/1883)
         ┌─────────────┴──────────────┐  ┌───────┴────────────────────┐
         │       ZABBIX SERVER         │  │     MQTT BROKER             │
         │  (Docker · PostgreSQL)      │  │     (Mosquitto)             │
         └──┬───────────┬─────────────┘  └──────┬─────────────────────┘
            │           │                        │
   ┌────────▼──────┐ ┌──▼─────────┐   ┌─────────▼──────────┐
   │  Linux / Pi   │ │  Windows   │   │  Arduino / ESP32   │
   │  Zabbix       │ │  Zabbix    │   │  SNMP sketch       │
   │  Agent 2      │ │  Agent 2   │   │  (Arduino-SNMP)    │
   │  + snmpd      │ │  + snmpd   │   │  or MQTT JSON      │
   └───────────────┘ └────────────┘   └────────────────────┘
```

---

## 2. Node Entity Model

Each networked host extends the home registry base entity schema with compute-node fields.

```json
{
  "id": "<uuid>",
  "type": "Asset",
  "subtype": "Device",
  "device_class": "ComputeNode",
  "label": "<node-label>",
  "os_family": "linux | windows",
  "os_distro": "<distro>",
  "arch": "x86_64 | aarch64 | armv7l",
  "hostname": "<hostname>.local",
  "zone_id": "<zone-id>",
  "wol_enabled": true,
  "monitoring": {
    "zabbix_host_id": 0,
    "agent_type": "zabbix-agent2 | snmp-only | mqtt-only",
    "snmp_version": "v2c | v3",
    "snmp_port": 161
  },
  "tags": ["server | workstation | sbc | microcontroller"],
  "status": "active",
  "power_state": "on | off | unknown"
}
```

### Node subtype taxonomy

```
ComputeNode
├── Server          (Linux, headless, always-on)
├── Workstation     (Linux / Windows, desktop)
├── SBC             (Raspberry Pi — any model)
└── Microcontroller (Arduino / ESP32 — SNMP sketch or MQTT only)
```

### Node capabilities matrix

| Subtype | Zabbix Agent 2 | SNMP | SSH | WOL | MQTT |
|---|:---:|:---:|:---:|:---:|:---:|
| Server (Linux) | ✓ | ✓ | ✓ | ✓ | ✓ |
| Workstation (Linux) | ✓ | ✓ | ✓ | ✓ | ✓ |
| Workstation (Windows) | ✓ | ✓ | — | ✓ | ✓ |
| Raspberry Pi | ✓ | ✓ | ✓ | — | ✓ |
| Arduino / ESP32 | — | ✓ | — | — | ✓ (fallback) |

---

## 3. Monitoring Stack — Zabbix

Zabbix is selected as the monitoring backbone on the following criteria:

- **100% open source** (GPL v2) — no feature gating, no licensing cost at any scale
- **Dual collection** — Zabbix Agent 2 (Go binary, ~15 MB RAM idle) for OS/application checks + SNMP for agentless and embedded nodes
- **Built-in templates** — covers Linux, Windows, Raspberry Pi, and custom SNMP devices without manual OID configuration
- **REST API** — JSON-RPC over plain HTTP; no proprietary SDK required
- **Swap path** — if Zabbix is ever replaced, only the `zabbix_api.py` adapter changes; the agent loop and entity model are unaffected

### Built-in templates per node type

| Node type | Template | Key metrics |
|---|---|---|
| Linux (Agent 2) | `Linux by Zabbix agent 2` | CPU, RAM, disk, network, load, systemd units |
| Linux (SNMP) | `Linux SNMP` | Same via SNMPv2c — agentless fallback |
| Windows (Agent 2) | `Windows by Zabbix agent 2` | CPU, RAM, disk, WMI, services, event log |
| Raspberry Pi | Linux template + Pi SNMP extend | + GPU temp, throttle state, voltage |
| Arduino / ESP32 | Custom SNMP template | Heap, RSSI, uptime, sensor OIDs |

---

## 4. Per-Node Agent Stack

### Linux & Raspberry Pi
- **Zabbix Agent 2** — active checks, application discovery, systemd monitoring
- **snmpd (net-snmp)** — SNMPv2c/v3 endpoint for agentless monitoring fallback
- **SNMP extend scripts** — expose /proc/stat (CPU), Raspberry Pi GPU metrics via SNMP extend
- **OpenSSH** — control plane for the management agent

### Windows
- **Zabbix Agent 2** — native Windows service; WMI checks, event log, service state
- **Windows SNMP service** — built-in Windows feature, enables SNMP polling
- **WOL** — enabled via NIC power management settings

### Arduino / ESP32
- **Primary:** Arduino-SNMP library — exposes standard MIB-II OIDs + custom enterprise OIDs (heap, RSSI, uptime, sensor readings). Polled by Zabbix server on the same schedule as any other SNMP device.
- **Fallback (Uno/Nano):** MQTT JSON sketch — retained state payload to `net/node/<id>/state` every 30 seconds.

---

## 5. Wake-on-LAN Architecture

WOL uses the IEEE 802.3 magic packet standard. An always-on host acts as a Layer-2 proxy triggered via MQTT.

```
Management Agent (Ollama)
  → MQTT publish: net/wol/cmd  {node_id, mac}          [ISO/IEC 20922]
    → WOL Proxy daemon (Python, always-on host)
      → UDP magic packet → <subnet-broadcast>:9         [IEEE 802.3]
        → target node powers on
          → Zabbix detects host availability (ICMP ping check)
            → Agent confirms via get_node_status()
```

The WOL proxy daemon subscribes to `net/wol/cmd`, sends the magic packet, and publishes confirmation to `net/wol/status`. It runs as a systemd service on the always-on host.

---

## 6. Tool Taxonomy

### Observation tools

| Tool | Protocol | Description |
|---|---|---|
| `get_node_status(node_id)` | Zabbix REST API | Power state, last seen, active problem count |
| `get_node_metrics(node_id, item_key)` | Zabbix REST API | Latest value for any Zabbix item key |
| `get_node_problems(node_id, severity?)` | Zabbix REST API | Active triggered alerts |
| `snmp_get(target_ip, oid)` | SNMP RFC 3411 | Direct SNMPv2c GET — agentless nodes |
| `snmp_walk(target_ip, oid_prefix)` | SNMP RFC 3411 | Walk OID subtree |
| `get_fleet_summary()` | Zabbix REST API | Fleet-wide online/offline/problem summary |
| `list_nodes(filter_status?, filter_tag?)` | Registry | All nodes from entity registry |

### Control tools

| Tool | Protocol | Description |
|---|---|---|
| `wake_node(node_id)` | MQTT → IEEE 802.3 | WOL magic packet via proxy |
| `shutdown_node(node_id, delay_minutes?)` | SSH RFC 4251 | Graceful shutdown |
| `run_ssh_command(node_id, command)` | SSH RFC 4251 | Arbitrary shell command |
| `restart_service(node_id, service_name)` | SSH RFC 4251 | systemctl restart |
| `zabbix_acknowledge(problem_id, message)` | Zabbix REST API | Acknowledge and annotate problem |

### Analysis tools

| Tool | Protocols | Description |
|---|---|---|
| `diagnose_node(node_id)` | Zabbix + SSH | Structured diagnostic across CPU/RAM/disk/temp/services |
| `compare_nodes(node_ids, item_key)` | Zabbix REST API | Side-by-side metric comparison |
| `get_snmp_inventory(target_ip)` | SNMP RFC 3411 | Walk MIB-II for system inventory |

---

## 7. Common Zabbix Item Keys

| Item key | Metric | Unit |
|---|---|---|
| `system.cpu.util` | CPU utilisation | % |
| `vm.memory.size[available]` | Available RAM | bytes |
| `vm.memory.size[pused]` | RAM used | % |
| `vfs.fs.size[/,pused]` | Root filesystem used | % |
| `system.uptime` | Uptime | seconds |
| `system.load[avg1]` | 1-minute load average | — |
| `net.if.in[eth0]` | Network interface RX | bits/s |
| `net.if.out[eth0]` | Network interface TX | bits/s |
| `sensor.temp.value[coretemp,Core 0]` | CPU core temperature | °C |
| `service.info[<name>,state]` | systemd service state | 0=running |

---

## 8. Agent Context Window

```
BLOCK A  System prompt (network management persona)        ~800 t
BLOCK B  Node registry snapshot (all nodes, compact)      ~600 t
BLOCK C  Fleet summary (online/offline, problem count)    ~200 t
BLOCK D  Active Zabbix problems digest (last 60 min)      ~400 t
BLOCK E  Conversation history (last 3 turns verbatim)    ~2000 t
BLOCK F  Tool schemas (intent-selected subset)           ~2000 t
BLOCK G  Output reserve                                  ~4000 t
─────────────────────────────────────────────────────────────────
Total active:  ~10 000 t  (well within 32 000 limit)
```

---

## 9. Polling & Event Strategy

### Zabbix-managed (primary)
- Agent 2 active checks: 60-second interval
- SNMP polls: 60 seconds (availability), 300 seconds (metrics)
- Triggers fire problems immediately on threshold breach
- Agent fetches active problems via REST API on demand

### MQTT event bus (secondary — embedded nodes and WOL)
- Arduino / ESP32 publish `net/node/<id>/state` every 30 seconds
- LWT fires `net/node/<id>/lwt = "offline"` on disconnect
- `event_watcher.py` daemon bridges MQTT alerts → Zabbix trapper when needed

### On-demand
- SSH commands: only on explicit agent decision
- SNMP direct queries: only when Zabbix agent unavailable
- WOL: only after confirming node is offline

---

## 10. Design Decisions

| Decision | Rationale |
|---|---|
| **Zabbix over Netdata** | 100% GPL, zero cost at any scale. Single platform covers Agent 2, SNMP, WMI, and MQTT trapper. No feature gating. |
| **SNMP on every node** | 30-year open standard (RFC 3411). Any future platform (LibreNMS, OpenNMS, Checkmk) can consume these endpoints without re-deploying agents. |
| **Arduino SNMP over MQTT** | Makes embedded nodes first-class SNMP citizens polled on the same schedule as servers. MQTT JSON retained as fallback for boards without sufficient TCP/IP stack. |
| **Zabbix REST API (no SDK)** | JSON-RPC over HTTP. Replaceable by pointing the adapter at any compatible API. |
| **WOL via MQTT proxy** | Decouples agent from Layer-2 broadcast domain. Proxy rehosted on any always-on node without agent changes. |
| **SSH for Linux/Pi control** | OpenSSH (RFC 4251) — universal open standard. No proprietary agent binary or API key. |
| **Protocol annotation on every tool** | Every tool docstring states the protocol used. Enables clean adapter swaps without changing the agent loop. |
| **Separate network agent loop** | Independent from the house energy agent. Shares MQTT broker and entity registry but uses its own system prompt and tool set. |

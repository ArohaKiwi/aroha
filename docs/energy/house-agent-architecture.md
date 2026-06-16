# House Agent Architecture — Local LLM (Ollama)

> Building a home intelligence agent on a local Ollama instance, grounded in the entity registry defined in `home-energy-entity-model.md`.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      USER / INTERFACE                        │
│          (CLI, MQTT topic, Home Assistant dashboard)          │
└──────────────────────────┬──────────────────────────────────┘
                           │ natural language intent
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR AGENT                        │
│   System prompt (static) + Context Window (assembled)        │
│   Ollama /api/chat  ·  model: qwen2.5:14b or llama3.1:8b    │
│   num_ctx: 32 000 tokens                                     │
│   temperature: 0.0   stop: ["WAIT"]                          │
└──────┬──────────────┬───────────────┬────────────────────────┘
       │ tool call    │ sub-agent     │ structured output
       ▼              ▼               ▼
  Tool Executor   Sub-agents      Response parser
  (Python)        (specialist     (Pydantic schema)
                   Ollama calls)
       │
       ├── Registry Reader     (entity registry JSON files)
       ├── Telemetry Reader    (MQTT last-value store / SQLite)
       ├── Control Actuator    (MQTT publish, schedules)
       ├── Tariff Calculator   (rate lookup + cost attribution)
       └── Alert Publisher     (MQTT / push notification)
```

---

## 2. Context Window Layout

The orchestrator's context window is assembled fresh on every turn from discrete blocks. Total budget: **32 000 tokens** (set via `num_ctx`).

```
┌──────────────────────────────────────────────┐  TIER 3: NEVER compressed
│  BLOCK A — System Prompt            ~1 200 t  │  Persona, rules, tool use protocol
│  BLOCK B — Entity Snapshot           ~800 t   │  Injected from registry at turn start
│  BLOCK C — Active Tariff             ~200 t   │  Current rate + export rate
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐  TIER 2: Compressible
│  BLOCK D — Telemetry Digest          ~600 t   │  Last 15 min of key sensor readings
│  BLOCK E — Automation State          ~400 t   │  Which schedules/automations are live
│  BLOCK F — Conversation History    ~4 000 t   │  Last 3 turn pairs verbatim; older summarised
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐  DYNAMIC
│  BLOCK G — Tool Schemas            ~3 000 t   │  Only tools relevant to this intent
│  BLOCK H — Output Reserve          ~4 000 t   │  Always protected
└──────────────────────────────────────────────┘

Total managed budget: ~14 200 t active + 4 000 reserve = 18 200 t
Remaining headroom for telemetry / memory expansion: ~13 800 t
```

### Token budget rules

| Zone | Threshold | Action |
|---|---|---|
| Green | < 70% of 32 000 | No action |
| Yellow | 70–85% | Flag older history for summarisation |
| Red | > 85% | Compress Tier 2 before LLM call |

---

## 3. System Prompt Design (Block A)

```
You are the house agent for <HOME_LABEL>, a residential energy system.

IDENTITY
- You manage and monitor the home's energy, comfort, and occupancy.
- You have access to real-time telemetry, the entity registry, and control actuators.
- You operate with local grid context (day/night tariff, microgeneration export).

PROTOCOL
1. Reason silently about the intent.
2. If a tool is needed, emit valid JSON then write the word WAIT on its own line.
3. The system will execute the tool and return the result.
4. Continue reasoning with the result until you can give a final answer.
5. Never hallucinate sensor values — always call get_telemetry before stating a reading.
6. Never actuate a device without first calling get_entity to confirm its current state.

TOOL FORMAT
{"name": "<tool_name>", "arguments": {<args>}}
WAIT

CONSTRAINTS
- Do NOT call the same tool twice with the same arguments.
- Only call control tools when the user intent is unambiguous.
- When energy cost matters, always call get_tariff before recommending an action.
- Prefer battery or solar dispatch over grid import when SOC > 30%.

TONE
- Concise. No filler phrases. State readings with units.
- Flag anomalies (e.g. COP < 2.5, SOC drop > 20% in 1 hour) proactively.
```

---

## 4. Entity Snapshot (Block B)

Injected at every turn from the entity registry. Compact summary only — full records stay on disk.

```json
{
  "home": "<home-label>",
  "zones": ["Ground Floor", "First Floor", "Roof Space", "External"],
  "residents": [{"label": "<resident>", "status": "home", "zone": "Ground Floor"}],
  "key_devices": [
    {"id": "pv-array-01",   "label": "Solar PV",    "class": "EnergySource",     "status": "active", "rated_w": 6000},
    {"id": "battery-01",    "label": "Battery",     "class": "EnergyStorage",    "status": "active", "rated_kwh": 10},
    {"id": "ashp-01",       "label": "Heat Pump",   "class": "EnergyConverter",  "status": "active", "rated_w": 3500},
    {"id": "inverter-01",   "label": "Hybrid Inv.", "class": "EnergyConverter",  "status": "active"},
    {"id": "ev-charger-01", "label": "EV Charger",  "class": "EnergyDistributor","status": "active"},
    {"id": "smeter-01",     "label": "Smart Meter", "class": "Gateway",          "status": "active"}
  ],
  "active_tariff": "<tariff-name>",
  "active_automations": ["HP Morning Boost", "Battery → EV when SOC > 80%"]
}
```

---

## 5. Tool Taxonomy

### Observation tools (read-only)
- `get_telemetry(device_id, attribute?)` — latest telemetry from a device
- `get_entity(entity_id)` — full entity record from registry
- `get_zone_state(zone_id)` — all sensors and actuators in a zone
- `get_tariff(period?)` — current or named tariff rates
- `get_energy_flow()` — live directed energy flow graph

### Control tools (write, require confirmation)
- `set_device_state(device_id, command, params)` — MQTT command to actuator
- `set_schedule(schedule_id, enabled, cron?)` — enable/modify a schedule
- `set_automation(automation_id, enabled)` — enable/disable an automation

### Calculation tools
- `calculate_cost(power_w, duration_minutes, period?)` — energy cost estimate
- `forecast_solar(hours_ahead)` — PV generation forecast

### Registry tools
- `search_entities(query, entity_type?)` — full-text search across registry
- `list_alerts(severity?, since_minutes?)` — active alert log

---

## 6. Sub-Agent Pattern

For complex multi-device tasks, the orchestrator spawns focused sub-agents. Each sub-agent:
- Receives a **fresh, minimal context** (its own system prompt + only the data it needs)
- Has access to a **single tool set** relevant to its domain
- Returns a **compact result object**, not its full trace

```
Orchestrator: "Pre-heat the house for 7am and charge the EV overnight cheaply"

→ Sub-agent A: EnergyPlanner
  Output: {"heat_pump_start": "06:30", "ev_charge_window": "23:00–06:00", "cost_eur": X.XX}

→ Sub-agent B: ScheduleWriter
  Output: {"schedules_updated": ["HP Morning Boost → 06:30", "EV Night Charge → 23:00"]}
```

---

## 7. Conversation History Compression

The assembler trims Block F before each LLM call using a three-tier strategy:
- **Tier 1 (protected)** — last 3 turn pairs retained verbatim
- **Tier 2 (compressible)** — older turns summarised by a fast small model (e.g. 3B)
- **Tier 3 (pinned)** — system prompt and critical context never compressed

Compression is triggered when token estimate exceeds 85% of `num_ctx`.

Summarisation prompt preserves: device states changed, commands issued, sensor readings cited, user preferences stated, decisions made. Discards pleasantries and redundant restatements.

---

## 8. Ollama Configuration

| Parameter | Value | Rationale |
|---|---|---|
| `model` | `qwen2.5:14b` (primary) | Strong tool-calling reliability |
| `num_ctx` | 32 000 | Fits all context blocks with headroom |
| `temperature` | 0.0 | Deterministic for control decisions |
| `stop` | `["WAIT"]` | Forces pause after tool call JSON — prevents hallucinated tool output |

### Model selection guide

| Model | VRAM | Role |
|---|---|---|
| `qwen2.5:14b` | ~10 GB | Primary orchestrator |
| `qwen2.5:7b` | ~5 GB | Sub-agents, single-tool tasks |
| `llama3.1:8b` | ~6 GB | Fallback orchestrator |
| `llama3.2:3b` | ~2 GB | Compression / summarisation only |

---

## 9. MQTT Integration

```
Subscriptions (read):
  home/+/+/+/state          → telemetry cache (SQLite)
  home/+/+/+/lwt            → device online/offline

Publications (write, via set_device_state tool):
  home/<bld>/<zone>/<dev>/cmd   → JSON command payload
  home/agent/intent             → last processed intent (audit log)
  home/agent/alert              → agent-raised alerts
```

Topic schema matches the entity model's `topic_prefix` field on each device entity.

---

## 10. Design Principles

1. **Registry-grounded** — the agent never invents device IDs or sensor names.
2. **Telemetry-first** — readings are always fetched live; stale training knowledge is never cited.
3. **Tool isolation** — control tools are gated by a confirmation flag in `agent_config.yaml`.
4. **Small context, big registry** — only a compact snapshot enters the context window; full records fetched by tool on demand.
5. **WAIT stop token** — forces the local model to pause after emitting a tool call JSON.
6. **Sub-agents for complexity** — multi-device planning decomposed; each sub-agent uses a small, focused context.
7. **Compression on demand** — history compressed with a fast 3B model only when budget approaches 85%.

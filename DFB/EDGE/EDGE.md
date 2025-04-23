# EDGE DFB – Status and Alarm Encoder
**Project**: UnityProV11.2  
**DFB**: `EDGE`  
**Platform**: Schneider Electric Unity Pro v11.2  
**Purpose**: Unified status and alarm encoding for integration with Ignition Edge and SCADA systems.

---

## Functional Overview
The `EDGE` Derived Function Block (DFB) is a logic abstraction designed to encode runtime equipment state and active system alarms into structured integer outputs. These can be consumed by SCADA, telemetry systems, or edge platforms like Ignition.

The block is tailored to:
- Encode a prioritized **equipment status code** (`STATUS_CODE`)
- Generate an indexed **list of active alarms** (`ALARM_CODE_ARRAY`)
- Track the **total number of active alarms** (`ALARM_COUNT`)
- Optionally capture raw **engine status word** (`EngineStatus`)

---

## Input Pins

| Pin Name      | Type     | Description                                  |
|---------------|----------|----------------------------------------------|
| `IN_0` to `IN_20` | `BOOL`  | Equipment runtime status flags (e.g. RUNNING, E-STOP, IDLE). Prioritized by order. |
| `IN_10001` to `IN_10010` | `WORD`  | Alarm word inputs. Each WORD maps 16 fault bits to `ALARM_CODE_ARRAY`. |

---

## Output Pins

| Output             | Type              | Description |
|--------------------|-------------------|-------------|
| `STATUS_CODE`      | `INT`             | A single integer representing prioritized system state (e.g., `0 = Stopped`, `1 = Running`, etc.). |
| `ALARM_CODE_ARRAY` | `ARRAY[0..127] OF INT` | List of active alarm integer codes. Cleared and rebuilt every PLC scan. |
| `ALARM_COUNT`      | `INT`             | Number of currently active alarms. |
| `EngineStatus`     | `WORD` (optional) | Raw engine state if passed from another system. |

---

##  Encoding Logic

### Status Code Priority
The first `TRUE` bit in `IN_0` to `IN_20` sets the `STATUS_CODE` from `0` upwards. Later bits are ignored until higher priority clears.

Example:
- `IN_1 = TRUE` → `STATUS_CODE = 1` (Running)
- `IN_0 = TRUE` → `STATUS_CODE = 0` (Stopped overrides)

### Alarm Unwrapping
Each input word (`IN_10001` to `IN_10010`) is bit-unwrapped. Each active bit produces an `INT` code as:

Example:
- `IN_10001` BIT15 (MSB) → `ALARM_CODE = 10025`
- `IN_10002` BIT0 → `ALARM_CODE = 10026`

Only 128 entries are stored; overflow is ignored.

---

##  SCADA Integration Notes

### Ignition Tag Mapping
- `STATUS_CODE` → Single tag for state logic in UDTs
- `ALARM_CODE_ARRAY[i]` → Loop through to evaluate active faults
- `ALARM_COUNT` → Filter logic in perspective / vision views

### Alarm Reset
All data is cleared and re-evaluated every scan. You can build rising-edge trigger events from changes in `ALARM_COUNT`.

---

## Development Notes
- Engineered in Unity Pro v11.2
- Bit priority in `STATUS_CODE` is static (left to right)
- No debounce or delay handling; add externally if needed
- Expandable to support more status flags or alarm registers







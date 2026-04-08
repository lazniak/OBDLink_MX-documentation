# ST Command Reference — STN2120 Exclusive

> **Applies to:** OBDLink MX+ (STN2120) **ONLY** — not available on ELM327 clones  
> **Prefix:** All commands prefixed with `ST`  
> **Advantage:** Extended capabilities unavailable in standard ELM327 firmware

---

## Device Identification

| Command | Description | Example Response |
|---------|-------------|-----------------|
| `STDI` | Device hardware identifier | `STN2120` |
| `STIX` | Extended device info | Full model + revision |
| `STFMR` | Firmware revision | `v4.2.1` |
| `STSN` | Serial number | Unique device serial |
| `STMFR` | Manufacturer string | `OBD Solutions LLC` |

---

## STPX — Protocol Transaction ⚡⭐

**The most important command for ultra-fast polling.** STPX combines header setup and data transmission into a single atomic transaction, eliminating extra Bluetooth round-trips.

### Syntax

```
STPX h:<header>, d:<data> [,t:<timeout>] [,r:<responses>] [,f:<flags>]
```

### Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `h:<header>` | CAN ID / transmit header | `h:7E0`, `h:7DF` |
| `d:<data>` | OBD payload bytes | `d:010C` (RPM) |
| `t:<timeout>` | Response timeout in ms | `t:50` |
| `r:<responses>` | Expected response count | `r:1` |
| `f:<flags>` | Control flags | — |

### Examples

```bash
# Basic — Request RPM from engine ECU
STPX h:7E0, d:010C, r:1

# Fast — RPM with tight timeout
STPX h:7E0, d:010C, t:30, r:1

# Broadcast — Query all ECUs for speed
STPX h:7DF, d:010D, r:1

# Get supported PIDs
STPX h:7E0, d:0100, r:1
```

### Performance Impact

| Method | BT Round-Trips per PID | Relative Speed |
|--------|----------------------|----------------|
| `ATSH` + PID command | 2 | Baseline |
| **`STPX`** | **1** | **~20–30% faster** |

> [!TIP]
> In a polling loop reading 100 PIDs/sec, saving one round-trip per PID means **100 fewer BT transmissions per second** — a significant reduction in latency and radio contention.

---

## CAN Monitoring (STN-Enhanced)

| Command | Description |
|---------|-------------|
| `STMA` | Start Monitor All (enhanced ATMA) |
| `STM` | Start Monitor (with active filters) |
| `STMFR` | Monitor with Filtering and Response |
| `STCMM 0` | CAN Monitor Mode: Raw frames |
| `STCMM 1` | CAN Monitor Mode: Formatted output |
| `STCSWM` | CAN Silent Wakeup Mode |

---

## CAN Pass/Block Filters

STN filters offer more granular control than standard AT filters.

| Command | Description |
|---------|-------------|
| `STFAP <id>, <mask>` | **Add Pass Filter** — only pass matching IDs |
| `STFAB <id>, <mask>` | **Add Block Filter** — block matching IDs |
| `STFAC` | **Clear All Filters** |
| `STFA` | Show Active Filters |

### Filter Examples

```bash
# Pass only engine ECU responses (7E8)
STFAP 7E8, 7FF

# Pass engine + transmission ECU (7E8, 7E9)
STFAP 7E8, 7FE

# Block ABS ECU responses (7EA)
STFAB 7EA, 7FF

# Clear all filters
STFAC
```

### Filter Mask Logic

```
Incoming CAN ID  AND  Mask  ==  Filter ID  AND  Mask  →  PASS/BLOCK

Example:
  Filter: STFAP 7E8, 7FF
  Incoming 7E8: 7E8 & 7FF = 7E8 == 7E8 & 7FF = 7E8 → ✅ PASS
  Incoming 7E9: 7E9 & 7FF = 7E9 ≠ 7E8              → ❌ BLOCK
  
  Filter: STFAP 7E8, 7FE  (last bit masked)
  Incoming 7E8: 7E8 & 7FE = 7E8 == 7E8 & 7FE = 7E8 → ✅ PASS
  Incoming 7E9: 7E9 & 7FE = 7E8 == 7E8              → ✅ PASS
  Incoming 7EA: 7EA & 7FE = 7EA ≠ 7E8               → ❌ BLOCK
```

---

## Power Management

| Command | Description |
|---------|-------------|
| `ATLP` | Enter Low Power mode |
| `STSLEEP` | Enter deep sleep |
| `STSLCS` | Sleep with Low Current State (~2 mA) |
| `STWBR` | Wake on Bluetooth Request |

---

## Baud Rate Control

| Command | Description |
|---------|-------------|
| `STBR <rate>` | Set UART baud rate (e.g., `STBR 921600`) |
| `STBRT <rate>` | Set baud rate with timeout fallback |

> [!NOTE]
> UART baud rate affects USB/serial connections. Bluetooth SPP baud rate is managed automatically by the BT stack and does not need manual configuration.

---

## Protocol Timeout

| Command | Description |
|---------|-------------|
| `STPTO <ms>` | Set protocol timeout in ms |
| `STP <n>` | Select protocol (same as `ATSP`) |

---

## Voltage Monitoring

| Command | Description |
|---------|-------------|
| `STVR` | Read real-time voltage |
| `STVTH <v>` | Set voltage threshold for sleep/wake |

---

## Why STN Commands Matter

```
┌──────────────────────────────────────────────┐
│           ELM327 clone                        │
│  ATSH 7E0   → BT round-trip #1 (~10ms)      │
│  010C\r     → BT round-trip #2 (~10ms)      │
│              + CAN round-trip   (~5ms)       │
│  TOTAL:                          ~25ms       │
├──────────────────────────────────────────────┤
│           OBDLink MX+ (STN2120)              │
│  STPX h:7E0, d:010C, r:1                    │
│              → BT round-trip #1 (~10ms)      │
│              + CAN round-trip   (~5ms)       │
│  TOTAL:                          ~15ms       │
│                                              │
│  SAVINGS: ~40% fewer BT transmissions        │
└──────────────────────────────────────────────┘
```

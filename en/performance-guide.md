# Performance Guide — Real-Time Optimization

> **Goal:** Maximize PID polling rate while maintaining stable, low-latency data delivery  
> **Target:** 80–120 PID samples/second on CAN 500K via Bluetooth Classic

---

## Bottleneck Analysis

Understanding where time is spent per PID request:

```
┌────────────────────────────────────────────────────────────────┐
│                    LATENCY PIPELINE                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────┐  ~0.5ms   TX: App → BT stack                │
│  │ Android App  │──────────►                                   │
│  └──────────────┘           ┌──────────────┐                   │
│                    ~5-10ms  │  Bluetooth    │  RF transmission  │
│                   ─────────►│  Classic SPP  │                   │
│                             └──────┬───────┘                   │
│                                    │ ~1-3ms                    │
│                             ┌──────▼───────┐                   │
│                             │   STN2120    │  Parse + forward  │
│                             └──────┬───────┘                   │
│                                    │ ~0.2ms                    │
│                             ┌──────▼───────┐                   │
│                             │   CAN Bus    │  Physical TX      │
│                             └──────┬───────┘                   │
│                                    │ ~2-5ms                    │
│                             ┌──────▼───────┐                   │
│                             │  Vehicle ECU │  Process request  │
│                             └──────┬───────┘                   │
│                                    │                           │
│                    (reverse path ~8-13ms)                      │
│                                    │                           │
│  TOTAL ROUND-TRIP:          ~8–25 ms per PID                   │
│  BOTTLENECK:                Bluetooth RF (~60% of total)       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Strategy 1: ATAT2 — Aggressive Adaptive Timing

**Impact: ⭐⭐⭐⭐⭐ Critical**

```
ATAT2
```

The adapter dynamically adjusts its internal response timeout. After each successful ECU response, ATAT2 tightens the window more aggressively than ATAT1, approaching the minimum viable wait time.

| Setting | Behavior | Typical Wait After Successful Response |
|---------|----------|---------------------------------------|
| `ATAT0` | Fixed timeout (ATST value) | Full ATST duration |
| `ATAT1` | Gradual adaptation | ~50% of ATST |
| **`ATAT2`** | **Aggressive adaptation** | **~20% of ATST** |

---

## Strategy 2: ATS0 — Eliminate Spaces

**Impact: ⭐⭐⭐ Significant**

```
ATS0
```

| Mode | Response Example | Bytes Over BT |
|------|-----------------|---------------|
| `ATS1` (spaces ON) | `41 0C 1A F8\r\r>` | 17 bytes |
| **`ATS0`** (spaces OFF) | `410C1AF8\r\r>` | **12 bytes** |

**~30% fewer bytes** transmitted over Bluetooth per response. At 100 responses/sec, this saves ~500 bytes/sec of BT bandwidth.

---

## Strategy 3: Physical Addressing + Filtering

**Impact: ⭐⭐⭐⭐ Major**

```
ATSH 7E0       ← Target engine ECU directly
ATCRA 7E8      ← Accept only engine ECU responses
```

### Why This Helps

| Addressing | What Happens |
|-----------|--------------|
| `7DF` (broadcast) | All ECUs respond → adapter waits for all, parses all |
| **`7E0` (physical)** | **Only engine ECU responds → adapter returns immediately** |

Combined with `ATCRA 7E8`, the adapter ignores any stray CAN traffic from other ECUs.

---

## Strategy 4: STPX Batch Transactions

**Impact: ⭐⭐⭐⭐ Major (STN2120 exclusive)**

```
# Instead of:
ATSH 7E0    → BT round-trip #1
010C\r      → BT round-trip #2

# Use:
STPX h:7E0, d:010C, r:1    → Single BT round-trip
```

**Saves 1 Bluetooth round-trip (~10ms) per PID request.**

---

## Strategy 5: Prioritized PID Groups

**Impact: ⭐⭐⭐⭐⭐ Critical for UX**

Not all vehicle parameters need the same update rate. Prioritize:

```
┌─────────────────────────────────────────────────────┐
│              PRIORITY SCHEDULING                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Group A (EVERY cycle):     RPM, Speed, Throttle    │
│  Group B (every 2nd cycle): Load, MAF               │  
│  Group C (every 5th cycle): Temps, Pressure         │
│  Group D (every 10th):      Fuel, Voltage, Oil      │
│                                                     │
│  Cycle pattern:                                     │
│  ┌─────────────────────────────────────────┐        │
│  │ 0: A B C D                              │        │
│  │ 1: A                                    │        │
│  │ 2: A B                                  │        │
│  │ 3: A                                    │        │
│  │ 4: A B                                  │        │
│  │ 5: A B C                                │        │
│  │ 6: A B                                  │        │
│  │ 7: A                                    │        │
│  │ 8: A B                                  │        │
│  │ 9: A                                    │        │
│  │10: A B C D                              │        │
│  │ ...                                      │        │
│  └─────────────────────────────────────────┘        │
│                                                     │
│  Result:                                            │
│    Group A → ~30-50 Hz  (RPM updates 30-50×/sec)    │
│    Group B → ~15-25 Hz                              │
│    Group C → ~6-10 Hz                               │
│    Group D → ~3-5 Hz                                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Strategy 6: Zero-Wait Pipeline

**Impact: ⭐⭐⭐⭐ Major**

```kotlin
// ❌ BAD — Adds unnecessary delay
fun pollLoop() {
    while (running) {
        send("010C\r")
        val response = readUntilPrompt()
        parseAndEmit(response)
        Thread.sleep(10)  // ← NEVER DO THIS
    }
}

// ✅ GOOD — Minimal latency
fun pollLoop() {
    while (running) {
        send("010C\r")
        output.flush()  // ← Force immediate send
        val response = readUntilPrompt()  // Blocking read
        parseAndEmit(response)
        // NO sleep — immediately send next request
    }
}
```

Key rules:
1. **Never** add `Thread.sleep()` in the polling loop
2. **Always** call `flush()` after `write()`
3. Parse **inline** — don't buffer responses
4. Send next command **immediately** after receiving `>`

---

## Strategy 7: Dedicated I/O Thread

**Impact: ⭐⭐⭐⭐⭐ Critical**

```
┌─────────────────────────────────────────────┐
│          THREAD ARCHITECTURE                 │
├─────────────────────────────────────────────┤
│                                             │
│  UI Thread (Main)          Data Thread      │
│  ├── Compose UI            ├── Parse hex    │
│  ├── Animations            ├── Apply formulas│
│  └── Touch events          └── Filter noise │
│        ▲                         ▲          │
│        │   StateFlow             │          │
│        └─────────────────────────┘          │
│                    ▲                         │
│                    │ Raw bytes               │
│              ┌─────┴─────┐                  │
│              │  I/O Thread│ ← Dedicated!    │
│              │ (Blocking) │                  │
│              │ write+read │                  │
│              └────────────┘                  │
│                                             │
└─────────────────────────────────────────────┘
```

**Never perform Bluetooth I/O on the Main thread.** Use a dedicated `Thread` or `Dispatchers.IO` coroutine scope.

---

## Combined Performance Results

| Configuration | PIDs/sec (Single PID) | PIDs/sec (8 PIDs Mixed) |
|--------------|----------------------|------------------------|
| Default ELM327 clone | ~20–30 | ~5–8 |
| OBDLink + ATAT1 | ~50–70 | ~15–25 |
| OBDLink + ATAT2 | ~70–100 | ~25–40 |
| **OBDLink + ATAT2 + ATS0 + ATCRA + STPX** | **~100–130** | **~40–60** |
| **+ Priority scheduling** | *Critical PIDs:* **~50 Hz** | *Combined:* **~80–120/sec** |

---

## Operating Modes

| Mode | PIDs Polled | Target Rate | Use Case |
|------|------------|-------------|----------|
| 🏎️ **Race** | RPM + Speed + Throttle (3) | 40–60 Hz | Track days, drag |
| 📊 **Dashboard** | 8–10 mixed PIDs | 10–20 Hz cycles | Daily driving |
| 🔧 **Diagnostic** | 15–20+ all PIDs | 3–5 Hz cycles | Troubleshooting |
| 📡 **Sniff** | CAN monitoring (passive) | Bus rate | Reverse engineering |

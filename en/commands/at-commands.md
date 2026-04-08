# AT Command Reference — ELM327 Compatible

> **Applies to:** OBDLink MX+ (STN2120) · ELM327 v1.4b compatible firmware  
> **Prefix:** All commands prefixed with `AT`  
> **Terminator:** Send `\r` (0x0D) after each command  
> **Response:** Ends with `>` prompt character

---

## General Commands

| Command | Description | Response |
|---------|-------------|----------|
| `ATZ` | Warm reset (restores defaults) | Device ID string |
| `ATI` | Print firmware version ID | `ELM327 v1.5` |
| `AT@1` | Print device description | `OBDLink MX+` |
| `AT@2` | Print device identifier | Serial/model info |
| `ATE0` | **Echo OFF** — stop echoing sent commands | `OK` |
| `ATE1` | Echo ON | `OK` |
| `ATL0` | **Linefeeds OFF** — omit `\n` in responses | `OK` |
| `ATL1` | Linefeeds ON | `OK` |
| `ATS0` | **Spaces OFF** — compact hex (e.g., `410C1AF8`) | `OK` |
| `ATS1` | Spaces ON (e.g., `41 0C 1A F8`) | `OK` |
| `ATH0` | Headers OFF — hide CAN ID in response | `OK` |
| `ATH1` | **Headers ON** — show CAN ID in response | `OK` |
| `ATD` | Set all to defaults | `OK` |
| `ATWS` | Warm start (like ATZ but faster) | Device ID |
| `ATRV` | Read vehicle battery voltage | e.g., `12.6V` |
| `ATIGN` | Read ignition status | `ON` / `OFF` |

---

## Protocol Selection

| Command | Description |
|---------|-------------|
| `ATSP 0` | **Auto-detect** protocol (recommended for first connection) |
| `ATSP 1` | Force SAE J1850 PWM (41.6 Kbps) |
| `ATSP 2` | Force SAE J1850 VPW (10.4 Kbps) |
| `ATSP 3` | Force ISO 9141-2 (10.4 Kbps) |
| `ATSP 4` | Force ISO 14230-4 KWP, 5-baud init |
| `ATSP 5` | Force ISO 14230-4 KWP, fast init |
| **`ATSP 6`** | **Force CAN 11-bit, 500 Kbps** ⭐ |
| `ATSP 7` | Force CAN 29-bit, 500 Kbps |
| `ATSP 8` | Force CAN 11-bit, 250 Kbps |
| `ATSP 9` | Force CAN 29-bit, 250 Kbps |
| `ATSP A` | Force SAE J1939, 29-bit, 250 Kbps |
| `ATSP B` | User CAN, 11-bit (variable speed) |
| `ATSP C` | User CAN, 29-bit (variable speed) |
| `ATDP` | Describe current protocol (human-readable) |
| `ATDPN` | Describe current protocol number |

> [!TIP]
> For known modern vehicles (2008+), force `ATSP 6` to skip auto-detection and save ~2 seconds at startup.

---

## Timing — Performance Critical ⚡

| Command | Description | Notes |
|---------|-------------|-------|
| `ATAT0` | Adaptive timing OFF | ❌ Avoid |
| `ATAT1` | Adaptive timing ON (normal) | Default behavior |
| **`ATAT2`** | **Adaptive timing ON (aggressive)** | **⭐ USE THIS — fastest polling** |
| `ATST xx` | Set timeout to `xx` × 4ms | Range: `01`–`FF` |
| `ATST 0A` | Timeout = 40ms | Good for CAN |
| `ATST 19` | Timeout = 100ms | Slower protocols |
| `ATST FF` | Timeout = 1020ms (max) | Fallback only |

> [!IMPORTANT]
> **`ATAT2` is the single most impactful command for real-time performance.** It dynamically shortens the response wait window after each successful reply, achieving optimal per-PID latency of **5–10ms on CAN**.

---

## CAN Header & Addressing

| Command | Description |
|---------|-------------|
| `ATSH xxx` | Set transmit header (CAN ID). E.g., `ATSH 7E0` |
| `ATSH 7DF` | Functional broadcast (query all ECUs) |
| **`ATSH 7E0`** | **Physical: Engine ECU only** ⭐ |
| `ATSH 7E1` | Physical: Transmission ECU |
| `ATSH 7E2` | Physical: ABS/ESP ECU |

---

## CAN Filtering & Receive

| Command | Description |
|---------|-------------|
| `ATCF xxx` | Set CAN receive filter ID |
| `ATCM xxx` | Set CAN receive filter mask |
| **`ATCRA xxx`** | **Set CAN Receive Address** (e.g., `ATCRA 7E8`) |
| `ATAR` | Auto Receive — clear receive filter |
| `ATCAF0` | CAN Auto Formatting OFF (raw frames) |
| `ATCAF1` | CAN Auto Formatting ON |

> [!TIP]
> Combine `ATSH 7E0` + `ATCRA 7E8` to create a point-to-point channel with the engine ECU. This eliminates responses from other ECUs and reduces parsing overhead.

---

## Flow Control (ISO-TP Multi-Frame)

| Command | Description |
|---------|-------------|
| `ATFC SH xxx` | Set Flow Control response header |
| `ATFC SD xxxx` | Set Flow Control response data bytes |
| `ATFC SM 0` | FC mode: Auto (adapter handles FC) |
| `ATFC SM 1` | FC mode: User-defined |
| `ATFC SM 2` | FC mode: OFF (no FC sent) |

---

## CAN Monitoring

| Command | Description |
|---------|-------------|
| `ATMA` | **Monitor All** — passive CAN bus listening |
| `ATMT xxx` | Monitor for Transmitter `xxx` |
| `ATMR xxx` | Monitor for Receiver `xxx` |
| `ATBD` | Perform buffer dump |
| `ATBI` | Bypass Initialization |

> [!WARNING]
> `ATMA` generates a continuous high-speed data stream. Ensure your parser can handle 2000–5000 frames/sec. Send any character to stop monitoring.

---

## Misc & Diagnostics

| Command | Description |
|---------|-------------|
| `ATPC` | Protocol Close — close current session |
| `ATDM1` | Monitor DM1 messages (J1939) |
| `ATMA` | Monitor All |
| `ATR0` | Responses OFF |
| `ATR1` | Responses ON |
| `ATAL` | Allow Long messages (>7 bytes) |
| `ATNL` | Normal Length messages only |
| `ATCSM0` | CAN Silent Mode OFF |
| `ATCSM1` | CAN Silent Mode ON |

---

## Quick Reference — Optimal Init Sequence

```
ATZ              Reset
ATE0             Echo OFF
ATL0             Linefeeds OFF
ATS0             Spaces OFF ⭐
ATH1             Headers ON
ATAT2            Aggressive Timing ⭐⭐⭐
ATSP6            CAN 11-bit 500K
ATCRA 7E8        Filter ECU engine only ⭐
ATSH 7E0         Target ECU engine ⭐
```

# OBDLink MX+ Bluetooth × Android — Complete Technical Documentation
## Knowledge Base for Building an Ultra-Fast Real-Time OBD-II Application

---

## Table of Contents

1. [Hardware Architecture — OBDLink MX+](#1-hardware-architecture--obdlink-mx)
2. [Bluetooth Communication Layer](#2-bluetooth-communication-layer)
3. [Android — Bluetooth SPP/RFCOMM Connection](#3-android--bluetooth-sprfcomm-connection)
4. [OBD-II Protocol Stack](#4-obd-ii-protocol-stack)
5. [AT Command Set (ELM327-Compatible)](#5-at-command-set-elm327-compatible)
6. [Extended ST Commands (STN-Exclusive)](#6-extended-st-commands-stn-exclusive)
7. [Device Initialization — Optimal Sequence](#7-device-initialization--optimal-sequence)
8. [Vehicle Protocols — Technical Details](#8-vehicle-protocols--technical-details)
9. [OBD-II Data Modes (SAE J1979)](#9-obd-ii-data-modes-sae-j1979)
10. [Complete PID Table — Mode 01](#10-complete-pid-table--mode-01)
11. [Performance Optimization — Real-Time Polling](#11-performance-optimization--real-time-polling)
12. [CAN Bus Monitoring — Passive Mode](#12-can-bus-monitoring--passive-mode)
13. [ISO-TP — Multi-Frame Communication](#13-iso-tp--multi-frame-communication)
14. [Android Application Architecture](#14-android-application-architecture)
15. [Connection Stability and Reconnect](#15-connection-stability-and-reconnect)
16. [Power Management — BatterySaver™](#16-power-management--batterysaver)
17. [Technical Limits and Bottlenecks](#17-technical-limits-and-bottlenecks)
18. [Summary — Ultra-Fast RT Strategy](#18-summary--ultra-fast-rt-strategy)

---

## 1. Hardware Architecture — OBDLink MX+

### Main Chip: STN2120
OBDLink MX+ is equipped with the **STN2120** — a 32-bit OBD processor from **ScanTool.net (OBD Solutions LLC)**. It is the successor to the STN1170 and STN1110, offering significantly higher protocol processing performance.

| Parameter | Value |
|---|---|
| **Processor** | STN2120 (32-bit ARM) |
| **Bluetooth** | Bluetooth 3.0 Classic, Class 2 |
| **Encryption** | 128-bit data encryption |
| **Range** | ~10m (Class 2) |
| **Firmware Compatibility** | ELM327 v1.4b + ST extensions |
| **Power Supply** | Direct from OBD-II port (12V) |
| **Power Draw (active)** | ~50 mA |
| **Power Draw (sleep)** | ~2 mA (BatterySaver™) |
| **Form Factor** | Compact |
| **Price** | ~$139.95 USD |

### Key Advantages Over ELM327 Clones

| Feature | OBDLink MX+ (STN2120) | Cheap ELM327 Clone |
|---|---|---|
| **Processing Speed** | ~3-5× faster | Baseline |
| **Adaptive Timing** | Advanced (ATAT2) | Basic or none |
| **CAN Monitoring** | Full ATMA/STM support | Limited/unstable |
| **SW-CAN / MS-CAN** | ✅ (Ford/GM networks) | ❌ |
| **Raw CAN** | Full support (ISO 11898) | Limited |
| **Firmware Updates** | ✅ Free | ❌ |
| **BT Security** | 128-bit encryption + physical button | None (open broadcast) |
| **STPX Batch Command** | ✅ | ❌ |
| **Stability** | Production-grade | Unstable |

---

## 2. Bluetooth Communication Layer

### Profile: SPP (Serial Port Profile)

OBDLink MX+ uses **Bluetooth Classic 3.0** with **SPP (Serial Port Profile)**, which means RS-232 serial port emulation over Bluetooth RFCOMM.

| Parameter | Value |
|---|---|
| **Bluetooth Profile** | SPP (Serial Port Profile) |
| **Transport Protocol** | RFCOMM |
| **SPP Service UUID** | `00001101-0000-1000-8000-00805F9B34FB` |
| **Theoretical Throughput (BT 3.0)** | Up to 3 Mbps |
| **Practical Throughput** | ~100-300 Kbps (after overhead) |
| **BT Classic Latency** | ~5-15 ms per round-trip |
| **Pairing** | PIN-based (default: no PIN / or `1234`) |
| **Security** | Physical "Connect" button + 128-bit encryption |

> [!IMPORTANT]
> OBDLink MX+ uses **Bluetooth Classic**, NOT BLE (Bluetooth Low Energy). On Android this requires `BLUETOOTH`, `BLUETOOTH_ADMIN` permissions, and from Android 12+ — `BLUETOOTH_CONNECT`, `BLUETOOTH_SCAN`.

### Bluetooth Classic vs BLE — Why Classic Is Better for OBD?

| Feature | Bluetooth Classic (SPP) | BLE |
|---|---|---|
| **Throughput** | High (~300 Kbps+) | Low (~10-50 Kbps) |
| **Stream Latency** | Low (continuous stream) | Higher (packet-based) |
| **Connection Mode** | Continuous, session-based | Event-driven |
| **Packet Size** | Large (up to 1023B RFCOMM) | Small (20-244B MTU) |
| **For OBD Real-Time** | ✅ Ideal | ⚠️ Compromise |

**Conclusion:** Bluetooth Classic SPP is the optimal choice for continuous, low-latency OBD-II data streaming in real-time mode.

---

## 3. Android — Bluetooth SPP/RFCOMM Connection

### 3.1 Required Permissions (AndroidManifest.xml)

```xml
<!-- Android 11 and below -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<!-- Android 12+ (API 31+) -->
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" 
    android:usesPermissionFlags="neverForLocation" />
```

### 3.2 Connection Sequence

```
┌─────────────────────────────────────────────────────────┐
│              ANDROID BLUETOOTH CONNECTION FLOW           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. BluetoothAdapter.getDefaultAdapter()                │
│     └─> Check if BT is enabled                          │
│                                                         │
│  2. getBondedDevices()                                  │
│     └─> Search for "OBDLink MX+" among paired devices   │
│                                                         │
│  3. device.createRfcommSocketToServiceRecord(SPP_UUID)  │
│     └─> UUID: 00001101-0000-1000-8000-00805F9B34FB      │
│                                                         │
│  4. socket.connect()                                    │
│     └─> Blocking call! Execute on a separate thread     │
│                                                         │
│  5. socket.getInputStream()                             │
│  6. socket.getOutputStream()                            │
│     └─> Ready for AT/OBD communication                  │
│                                                         │
│  7. Device initialization (AT sequence)                 │
│     └─> ATZ → ATE0 → ATL0 → ATS0 → ATH1 → ATAT2      │
│                                                         │
│  8. PID polling loop (Real-Time Loop)                   │
│     └─> Continuous request/response on dedicated thread │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 3.3 Key Connection Constants

```kotlin
companion object {
    // Standard SPP UUID
    val SPP_UUID: UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB")
    
    // OBDLink MX+ device name
    const val DEVICE_NAME = "OBDLink MX+"
    
    // Terminators
    const val COMMAND_TERMINATOR = '\r'    // Carriage Return — end of command
    const val RESPONSE_TERMINATOR = '>'   // Prompt — end of response
    
    // Timeouts
    const val CONNECT_TIMEOUT_MS = 10_000L
    const val COMMAND_TIMEOUT_MS = 2_000L
    const val INIT_TIMEOUT_MS = 5_000L
    
    // Buffer
    const val READ_BUFFER_SIZE = 1024
}
```

### 3.4 Text Communication Protocol

Communication with OBDLink MX+ is **ASCII text-based**:

```
Send:    "<COMMAND>\r"          (terminated with Carriage Return 0x0D)
Receive: "<RESPONSE>\r\r>"     (terminated with '>' prompt)
```

**Example:**
```
TX: "010C\r"            ← Request RPM
RX: "41 0C 1A F8\r\r>"  ← Response with data
```

> [!TIP]
> After every `write()` on `OutputStream`, **immediately** call `flush()` to force the buffer to send without waiting for it to fill.

### 3.5 Fallback — Insecure RFCOMM

If `createRfcommSocketToServiceRecord()` fails (e.g., pairing issues):

```kotlin
// Fallback: Insecure RFCOMM (no pairing required)
val socket = device.createInsecureRfcommSocketToServiceRecord(SPP_UUID)

// Alternative Fallback: Reflection hack (older devices)
val method = device.javaClass.getMethod(
    "createRfcommSocket", Int::class.javaPrimitiveType
)
val socket = method.invoke(device, 1) as BluetoothSocket
```

---

## 4. OBD-II Protocol Stack

### Supported Vehicle Protocols

OBDLink MX+ supports **all** legislated OBD-II protocols plus additional ones:

| # | Protocol | Standard | Speed | Typical Vehicles |
|---|---|---|---|---|
| 0 | Auto-detect | — | — | Automatic selection |
| 1 | SAE J1850 PWM | — | 41.6 Kbps | Ford (older) |
| 2 | SAE J1850 VPW | — | 10.4 Kbps | GM (older) |
| 3 | ISO 9141-2 | — | 10.4 Kbps | European, Asian, Chrysler |
| 4 | ISO 14230-4 KWP (5-baud init) | KWP2000 | 10.4 Kbps | European (older) |
| 5 | ISO 14230-4 KWP (fast init) | KWP2000 | 10.4 Kbps | European |
| **6** | **ISO 15765-4 CAN (11-bit, 500 Kbps)** | **CAN** | **500 Kbps** | **Most 2008+** |
| 7 | ISO 15765-4 CAN (29-bit, 500 Kbps) | CAN | 500 Kbps | Trucks, heavy-duty |
| 8 | ISO 15765-4 CAN (11-bit, 250 Kbps) | CAN | 250 Kbps | Some European |
| 9 | ISO 15765-4 CAN (29-bit, 250 Kbps) | CAN | 250 Kbps | Heavy-duty |
| A | SAE J1939 CAN (29-bit, 250 Kbps) | J1939 | 250 Kbps | Trucks |
| B | User-defined CAN (11-bit) | Raw CAN | Configurable | Custom |
| C | User-defined CAN (29-bit) | Raw CAN | Configurable | Custom |

> [!IMPORTANT]
> **Protocol 6 (ISO 15765-4 CAN 11-bit 500 Kbps)** is the fastest and most common protocol in modern vehicles (2008+). This is the **TARGET** protocol for maximum real-time performance.

### Additional Protocols (MX+ Only)

| Protocol | Description | Vehicles |
|---|---|---|
| **SW-CAN** | Single-Wire CAN (33.3 Kbps) | GM (body control) |
| **MS-CAN** | Medium-Speed CAN (125 Kbps) | Ford (body control) |

---

## 5. AT Command Set (ELM327-Compatible)

OBDLink MX+ is fully compatible with the ELM327 v1.4b command set. Every command is prefixed with "AT".

### 5.1 General Commands

| Command | Description | Response |
|---|---|---|
| `ATZ` | Device reset (warm reset) | `ELM327 v1.5` or `STN2120` |
| `ATI` | Device identification | Firmware version |
| `AT@1` | Manufacturer identifier | `OBDLink MX+` |
| `ATE0` | Echo OFF (disable command echo) | `OK` |
| `ATE1` | Echo ON | `OK` |
| `ATL0` | Linefeeds OFF | `OK` |
| `ATL1` | Linefeeds ON | `OK` |
| `ATS0` | Spaces OFF (no spaces in hex) | `OK` |
| `ATS1` | Spaces ON (spaces in hex) | `OK` |
| `ATH0` | Headers OFF | `OK` |
| `ATH1` | Headers ON (show CAN ID in response) | `OK` |

### 5.2 Protocol Commands

| Command | Description |
|---|---|
| `ATSP0` | Auto-detect protocol |
| `ATSP6` | Force CAN 11-bit 500 Kbps |
| `ATSP7` | Force CAN 29-bit 500 Kbps |
| `ATDP` | Display current protocol |
| `ATDPN` | Display current protocol number |

### 5.3 Timing Commands — Key to Performance

| Command | Description | Notes |
|---|---|---|
| `ATAT0` | Adaptive timing OFF | ❌ Don't use |
| `ATAT1` | Adaptive timing ON (standard) | ⚠️ Default |
| **`ATAT2`** | **Adaptive timing ON (aggressive)** | **✅ ULTRA-FAST — USE THIS!** |
| `ATST FF` | Set timeout (FF = max, 01 = min) | Manual timeout |
| `ATST 0A` | Timeout = 10 × 4ms = 40ms | Typical for CAN |
| `ATST 19` | Timeout = 25 × 4ms = 100ms | For slower protocols |

> [!CAUTION]
> **`ATAT2` is critical for real-time performance!** Aggressive adaptive timing minimizes ECU response wait time. The device dynamically shortens the timeout after each successful response, achieving optimal times of **5-10ms per PID** on CAN.

### 5.4 CAN Filtering Commands

| Command | Description |
|---|---|
| `ATCF xxx` | Set CAN Filter (e.g., `ATCF 7E8`) |
| `ATCM xxx` | Set CAN Mask (e.g., `ATCM 7FF`) |
| `ATCRA xxx` | Set CAN Receive Address (e.g., `ATCRA 7E8`) |
| `ATAR` | Auto Receive (reset filters) |
| `ATCAF0` | CAN Auto Formatting OFF (raw frames) |
| `ATCAF1` | CAN Auto Formatting ON |

### 5.5 Diagnostic Commands

| Command | Description |
|---|---|
| `ATSH xxx` | Set Header (CAN ID for sending, e.g., `ATSH 7E0`) |
| `ATFC SH xxx` | Set Flow Control Header |
| `ATFC SD xxxx` | Set Flow Control Data |
| `ATFC SM x` | Set Flow Control Mode (0=auto, 1=user, 2=off) |

---

## 6. Extended ST Commands (STN-Exclusive)

STN commands are **exclusive** to STN11xx/STN2120 chips and are NOT available on ELM327 clones. This is the main advantage of OBDLink over cheaper adapters.

### 6.1 Identification Commands

| Command | Description |
|---|---|
| `STDI` | Device Identifier — returns chip model |
| `STIX` | Extended Device ID — full information |
| `STFMR` | Firmware revision |
| `STSN` | Device serial number |

### 6.2 STPX — Protocol Transaction (KEY TO PERFORMANCE)

**STPX** is the most important command for ultra-fast polling. It allows sending a complete protocol transaction in a single command, eliminating the overhead of multiple AT commands.

**Syntax:**
```
STPX h:<header>, d:<data> [,t:<timeout>] [,r:<responses>] [,f:<flags>]
```

| Parameter | Description |
|---|---|
| `h:<header>` | CAN ID / Message header (e.g., `h:7E0`) |
| `d:<data>` | Payload data (e.g., `d:010C` = Mode 01, PID 0C) |
| `t:<timeout>` | Timeout in ms (e.g., `t:50`) |
| `r:<responses>` | Expected number of responses (e.g., `r:1`) |
| `f:<flags>` | Control flags |

**Example — Get RPM:**
```
STPX h:7DF, d:010C, r:1
```

**Example — Fast read with minimal timeout:**
```
STPX h:7E0, d:010C, t:30, r:1
```

> [!TIP]
> `STPX` is 20-30% faster than the standard `ATSH` + OBD request sequence, because it combines header setting and data transmission into a single atomic transaction, eliminating Bluetooth round-trip overhead.

### 6.3 STN CAN Monitoring Commands

| Command | Description |
|---|---|
| `STMA` | Start CAN Monitoring All (equivalent to ATMA) |
| `STM` | Start CAN Monitoring (with filters) |
| `STMFR` | Monitor with Filtering and Response |
| `STCMM 1` | CAN Monitoring Mode 1 (formatted) |
| `STCMM 0` | CAN Monitoring Mode 0 (raw) |
| `STCSWM` | CAN Silent Wakeup Mode |

### 6.4 STN Filter Commands

| Command | Description |
|---|---|
| `STFAP xxxx, yyyy` | Add Pass Filter (ID + mask) |
| `STFAB xxxx, yyyy` | Add Block Filter |
| `STFAC` | Clear All Filters |
| `STFA` | Show Active Filters |

### 6.5 Other STN Commands

| Command | Description |
|---|---|
| `STSLCS` | Sleep with Low Current State |
| `STSLEEP` | Enter sleep mode |
| `STWBR` | Wake on Bluetooth Request |
| `STBR xxxxxx` | Set Baud Rate (e.g., `STBR 921600`) |
| `STPTO xxx` | Set Protocol Timeout |

---

## 7. Device Initialization — Optimal Sequence

### 7.1 Minimal Sequence (Quick Start)

```
ATZ          → Reset
ATE0         → Echo OFF (less data over BT)
ATL0         → Linefeeds OFF
ATS0         → Spaces OFF (fewer bytes = faster)
ATH1         → Headers ON (needed for ECU identification)
ATAT2        → Aggressive Adaptive Timing ⭐
ATSP6        → Force CAN 11-bit 500K (if vehicle is known)
```

### 7.2 Optimal Sequence (Ultra-Performance)

```
ATZ              → Device reset
ATE0             → Echo OFF
ATL0             → Linefeeds OFF  
ATS0             → Spaces OFF ⭐ (~30% BT data reduction)
ATH1             → Headers ON
ATAT2            → Aggressive Adaptive Timing ⭐⭐⭐
ATSP6            → CAN 11-bit 500K
ATCAF1           → CAN Auto Formatting ON
ATCRA 7E8        → Filter only engine ECU responses ⭐
ATSH 7E0         → Set header to physical ECU address ⭐
ATST 0A          → Timeout 40ms (sufficient for CAN)
```

### 7.3 Checking Supported PIDs

```
0100             → Supported PIDs [01-20] — bitmask
0120             → Supported PIDs [21-40]
0140             → Supported PIDs [41-60]
0160             → Supported PIDs [61-80]
0180             → Supported PIDs [81-A0]
01A0             → Supported PIDs [A1-C0]
```

> [!NOTE]
> The `0100` response is a 4-byte bitmask (32 bits). Each bit corresponds to a PID. Bit=1 means supported. E.g., response `BE 1F A8 13` means:
> - Bit 31 (PID 01): ✅ Monitor status
> - Bit 30 (PID 02): ❌ 
> - Bit 29 (PID 03): ✅ Fuel system status
> - etc.

---

## 8. Vehicle Protocols — Technical Details

### 8.1 CAN Bus (ISO 15765-4) — Target Protocol

```
┌──────────────────────────────────────────────────────────┐
│                 CAN FRAME STRUCTURE                       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────┐ ┌─────┐ ┌──────────────────┐ ┌─────┐       │
│  │ CAN ID  │ │ DLC │ │   DATA (0-8B)    │ │ CRC │       │
│  │ 11/29b  │ │ 4b  │ │                  │ │ 15b │       │
│  └─────────┘ └─────┘ └──────────────────┘ └─────┘       │
│                                                          │
│  OBD-II on CAN:                                          │
│  ┌─────────┐ ┌──────────────────────────────────────┐    │
│  │ CAN ID  │ │  Byte 0  │ Byte 1 │ Byte 2 │ Data...│    │
│  │ 7DF/7E0 │ │  Length  │ Mode   │  PID   │        │    │
│  └─────────┘ └──────────────────────────────────────┘    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 8.2 CAN Addressing

| Type | Request CAN ID | Response CAN ID | Usage |
|---|---|---|---|
| **Functional (broadcast)** | `7DF` | `7E8`-`7EF` | Query all ECUs |
| **Physical (direct)** | `7E0` | `7E8` | Direct to engine ECU |
| Physical | `7E1` | `7E9` | Transmission ECU |
| Physical | `7E2` | `7EA` | ABS/ESP ECU |

> [!TIP]
> **For maximum speed:** Use **physical** addressing (`7E0`) instead of broadcast (`7DF`). You avoid collecting responses from multiple ECUs and reduce bus noise.

### 8.3 Protocol Speed Comparison

| Protocol | Bus Speed | Typical Response Time per PID | PIDs/second (real) |
|---|---|---|---|
| **CAN 500K** | 500 Kbps | **5-15 ms** | **40-100+** |
| CAN 250K | 250 Kbps | 10-25 ms | 25-60 |
| KWP2000 Fast | 10.4 Kbps | 50-100 ms | 5-10 |
| ISO 9141-2 | 10.4 Kbps | 100-200 ms | 3-5 |
| J1850 VPW | 10.4 Kbps | 50-150 ms | 4-8 |
| J1850 PWM | 41.6 Kbps | 30-80 ms | 8-15 |

---

## 9. OBD-II Data Modes (SAE J1979)

### 9.1 Standard Modes (Mode/Service)

| Mode | Hex | Description | For Real-Time |
|---|---|---|---|
| **01** | `01` | **Current Powertrain Data (Live Data)** | **✅ PRIMARY** |
| 02 | `02` | Freeze Frame Data | ❌ Historical |
| 03 | `03` | Emission-Related DTCs | ❌ Diagnostics |
| 04 | `04` | Clear/Reset DTCs | ❌ Action |
| 05 | `05` | Oxygen Sensor Monitoring | ⚠️ Specific |
| 06 | `06` | On-Board Monitoring Test Results | ⚠️ Specific |
| 07 | `07` | Pending DTCs (current cycle) | ❌ Diagnostics |
| 08 | `08` | Control On-Board System | ❌ Action |
| 09 | `09` | Vehicle Information (VIN etc.) | ❌ One-time |
| 0A | `0A` | Permanent DTCs | ❌ Diagnostics |

### 9.2 Service $22 — Read Data By Identifier (Advanced)

**Mode $22** allows reading **manufacturer-specific** PIDs (2-byte DID). It is the only way to request **multi-PID in a single CAN frame**.

```
Request:  22 F4 0C F4 0D F4 05
Response: 62 F4 0C [data] F4 0D [data] F4 05 [data]
```

> [!WARNING]
> Service $22 PIDs are **manufacturer-specific**. They require knowledge of DID tables for the specific vehicle model. They are not part of the OBD-II standard.

---

## 10. Complete PID Table — Mode 01

### 10.1 Most Important PIDs for Real-Time (High-Priority)

| PID | Hex | Description | Bytes | Formula | Unit | Min | Max |
|---|---|---|---|---|---|---|---|
| **0C** | `0C` | **Engine RPM** | 2 | `(256×A + B) / 4` | rpm | 0 | 16,383.75 |
| **0D** | `0D` | **Vehicle Speed** | 1 | `A` | km/h | 0 | 255 |
| **05** | `05` | **Coolant Temperature** | 1 | `A - 40` | °C | -40 | 215 |
| **04** | `04` | **Calculated Engine Load** | 1 | `A × 100 / 255` | % | 0 | 100 |
| **11** | `11` | **Throttle Position** | 1 | `A × 100 / 255` | % | 0 | 100 |
| **10** | `10` | **MAF Air Flow Rate** | 2 | `(256×A + B) / 100` | g/s | 0 | 655.35 |
| **0E** | `0E` | **Timing Advance** | 1 | `A / 2 - 64` | ° | -64 | 63.5 |
| **0F** | `0F` | **Intake Air Temperature** | 1 | `A - 40` | °C | -40 | 215 |
| **0B** | `0B` | **Intake Manifold Pressure** | 1 | `A` | kPa | 0 | 255 |

### 10.2 Additional PIDs (Medium-Priority)

| PID | Hex | Description | Bytes | Formula | Unit |
|---|---|---|---|---|---|
| **01** | `01` | Monitor Status Since DTCs Cleared | 4 | Bitmask | — |
| **03** | `03` | Fuel System Status | 2 | Bitmask | — |
| **06** | `06` | Short-Term Fuel Trim (Bank 1) | 1 | `A × 100 / 128 - 100` | % |
| **07** | `07` | Long-Term Fuel Trim (Bank 1) | 1 | `A × 100 / 128 - 100` | % |
| **08** | `08` | Short-Term Fuel Trim (Bank 2) | 1 | `A × 100 / 128 - 100` | % |
| **09** | `09` | Long-Term Fuel Trim (Bank 2) | 1 | `A × 100 / 128 - 100` | % |
| **0A** | `0A` | Fuel Pressure | 1 | `A × 3` | kPa |
| **14-1B** | — | O2 Sensor Voltage/Trim (Bank 1-2) | 2 | Varies | V / % |
| **1C** | `1C` | OBD Standard Compliance | 1 | Lookup | — |
| **1F** | `1F` | Runtime Since Engine Start | 2 | `256×A + B` | seconds |
| **21** | `21` | Distance Traveled with MIL ON | 2 | `256×A + B` | km |
| **2F** | `2F` | Fuel Tank Level Input | 1 | `A × 100 / 255` | % |
| **31** | `31` | Distance Since DTCs Cleared | 2 | `256×A + B` | km |
| **33** | `33` | Barometric Pressure | 1 | `A` | kPa |
| **42** | `42` | Control Module Voltage | 2 | `(256×A + B) / 1000` | V |
| **43** | `43` | Absolute Load Value | 2 | `(256×A + B) × 100 / 255` | % |
| **45** | `45` | Relative Throttle Position | 1 | `A × 100 / 255` | % |
| **46** | `46` | Ambient Air Temperature | 1 | `A - 40` | °C |
| **47-4B** | — | Throttle Position B/C/D/E | 1 | `A × 100 / 255` | % |
| **4C** | `4C` | Commanded Throttle Actuator | 1 | `A × 100 / 255` | % |
| **4D** | `4D` | Time Run with MIL ON | 2 | `256×A + B` | min |
| **5C** | `5C` | Engine Oil Temperature | 1 | `A - 40` | °C |
| **5E** | `5E` | Engine Fuel Rate | 2 | `(256×A + B) / 20` | L/h |

### 10.3 Turbo/Boost PIDs (if supported)

| PID | Hex | Description | Bytes | Formula | Unit |
|---|---|---|---|---|---|
| **70** | `70` | Boost Pressure Control | 9 | Complex | kPa |
| **6C** | `6C` | Commanded Throttle Actuator | 5 | Complex | — |

### 10.4 Request and Response Format

```
REQUEST:    01 [PID]
RESPONSE:   41 [PID] [DATA BYTES...]

Example — Engine RPM (PID 0C):
TX: "010C\r"
RX: "41 0C 1A F8"

Decoding: 
  A = 0x1A = 26
  B = 0xF8 = 248
  RPM = (256 × 26 + 248) / 4 = (6656 + 248) / 4 = 6904 / 4 = 1726 RPM
```

---

## 11. Performance Optimization — Real-Time Polling

### 11.1 Strategy 1: Sequential Polling with ATAT2

The simplest and most reliable method:

```
┌──────────────────────────────────────────────────┐
│          SEQUENTIAL POLLING LOOP                  │
├──────────────────────────────────────────────────┤
│                                                  │
│  while (connected) {                             │
│      send("010C\r")   → RPM                      │
│      parse(receive())                            │
│                                                  │
│      send("010D\r")   → Speed                    │
│      parse(receive())                            │
│                                                  │
│      send("0105\r")   → Coolant Temp             │
│      parse(receive())                            │
│                                                  │
│      send("0111\r")   → Throttle                 │
│      parse(receive())                            │
│                                                  │
│      // ~4 PIDs × 10ms = 40ms per cycle          │
│      // = 25 full cycles/second                  │
│      // = ~100 PID reads/second                  │
│  }                                               │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Expected performance on CAN 500K + ATAT2:**
- **1 PID:** ~5-15 ms → up to **100+ PID samples/sec** (single PID)
- **4 PIDs:** ~25-50 ms cycle → **20-40 cycles/sec**
- **8 PIDs:** ~50-100 ms cycle → **10-20 cycles/sec**

### 11.2 Strategy 2: STPX Batch Transactions

Using the STPX command eliminates `ATSH` overhead:

```
// Instead of:
ATSH 7E0        → 1 BT round-trip
010C             → 1 BT round-trip + 1 CAN round-trip
ATSH 7E0        → 1 BT round-trip (redundant!)
010D             → 1 BT round-trip + 1 CAN round-trip

// Use:
STPX h:7E0, d:010C, r:1    → 1 round-trip (BT+CAN integrated)
STPX h:7E0, d:010D, r:1    → 1 round-trip
```

**Gain: ~20-30% fewer BT round-trips.**

### 11.3 Strategy 3: Physical Addressing + Filtering

```
// Set permanently:
ATSH 7E0         → Engine ECU only (no other ECU responds)
ATCRA 7E8        → Filter ONLY engine ECU responses
ATAT2            → Aggressive timeout shortening

// Effect: Adapter doesn't wait for responses from other ECUs
// Wait time reduction: ~5-10ms per PID
```

### 11.4 Strategy 4: ATS0 — Spaces Off

```
// With spaces (ATS1):    "41 0C 1A F8\r\r>"   → 17 bytes
// Without spaces (ATS0): "410C1AF8\r\r>"       → 12 bytes

// ~30% data reduction over Bluetooth!
// At 100 responses/sec = hundreds of saved bytes/sec
```

### 11.5 Strategy 5: Prioritized PID Groups

Create PID groups with different priorities:

```
Group A (CRITICAL — Every Cycle):
  0C = RPM
  0D = Speed
  11 = Throttle

Group B (HIGH — Every 2nd Cycle):
  04 = Engine Load
  10 = MAF

Group C (MEDIUM — Every 5th Cycle):
  05 = Coolant Temp
  0F = Intake Air Temp
  0B = Intake Manifold Pressure

Group D (LOW — Every 10th Cycle):
  2F = Fuel Level
  42 = Control Module Voltage
  5C = Oil Temperature
```

```
Algorithm:
  cycle = 0
  while (connected) {
      poll(GROUP_A)                    // Always
      if (cycle % 2 == 0) poll(GROUP_B)  // Every 2nd cycle
      if (cycle % 5 == 0) poll(GROUP_C)  // Every 5th cycle
      if (cycle % 10 == 0) poll(GROUP_D) // Every 10th cycle
      cycle++
  }
```

**Effect:** Critical data (RPM, Speed, Throttle) refreshed at ~30-50 Hz, slower parameters at lower frequencies without wasting bandwidth.

### 11.6 Strategy 6: Zero-Wait Pipeline

Instead of waiting for the full response, parse byte by byte:

```
┌────────────────────────────────────────────────┐
│         ZERO-WAIT PIPELINE                      │
├────────────────────────────────────────────────┤
│                                                │
│  TX: "010C\r"                                  │
│  ↕ (wait for '>' = end of response)            │
│  RX: "410C1AF8\r\r>"                           │
│  ↕ (immediately parse + send next)             │
│  TX: "010D\r"                                  │
│  ↕ ...                                         │
│                                                │
│  KEY: Never add any Thread.sleep()!            │
│  Parse inline, send immediately after '>'      │
│                                                │
└────────────────────────────────────────────────┘
```

### 11.7 Summary — Optimization Table

| Technique | Performance Impact | Difficulty |
|---|---|---|
| `ATAT2` — Aggressive Timing | ⭐⭐⭐⭐⭐ | Easy |
| `ATS0` — Spaces Off | ⭐⭐⭐ | Easy |
| `ATCRA 7E8` — Filter ECU | ⭐⭐⭐⭐ | Easy |
| Physical addr (`7E0`) | ⭐⭐⭐ | Easy |
| `STPX` batch | ⭐⭐⭐⭐ | Medium |
| Prioritized PID groups | ⭐⭐⭐⭐⭐ | Medium |
| Zero-wait pipeline | ⭐⭐⭐⭐ | Medium |
| `flush()` after write | ⭐⭐ | Easy |
| Dedicated IO thread | ⭐⭐⭐⭐⭐ | Medium |

---

## 12. CAN Bus Monitoring — Passive Mode

### 12.1 What Is CAN Monitoring?

CAN Monitoring is a **passive listening** mode on the CAN bus. The adapter doesn't send any queries — instead it captures **ALL** CAN frames that appear on the bus.

This is the **FASTEST** data capture method because:
- ❌ No round-trip (request → response)
- ✅ Data arrives at the rate the ECU generates it (10-100 Hz per parameter)
- ✅ Multiple parameters simultaneously

### 12.2 Starting CAN Monitoring

```
// Configuration
ATSP6          → CAN 11-bit 500K
ATCAF0         → Raw CAN frames (no formatting)
ATH1           → Show CAN ID

// Filtering (optional but recommended!)
STFAC          → Clear all filters
STFAP 7E8, 7FF → Pass only engine ECU responses

// Start monitoring
ATMA           → Monitor All CAN traffic
// or
STM            → STN Monitor (with filters)
```

### 12.3 Data Format in Monitoring Mode

```
// Raw CAN frames (continuous stream):
7E8 06 41 0C 1A F8 00 00    ← RPM
7E8 04 41 0D 3C 00 00 00    ← Speed (60 km/h)
7E9 05 41 05 6E 00 00 00    ← Coolant Temp from transmission ECU
7E8 04 41 11 2F 00 00 00    ← Throttle
7EA 06 41 04 B2 00 00 00    ← Load from ABS ECU
...
```

### 12.4 Stopping Monitoring

Send **any character** (e.g., `\r`) to interrupt monitoring mode. The adapter will respond with the `>` prompt.

> [!WARNING]
> **CAN Monitoring generates MASSIVE amounts of data!** A modern vehicle's CAN bus can generate 2000-5000 frames/second. Without filters, Bluetooth 3.0 may not keep up with transmission. **ALWAYS** use CAN filters (`STFAP`, `ATCF`, `ATCM`).

### 12.5 Monitoring vs Polling — Comparison

| Feature | Polling (Mode 01) | CAN Monitoring |
|---|---|---|
| **Data Control** | ✅ Full (you choose PIDs) | ⚠️ Whatever ECU broadcasts |
| **Frequency** | ~20-100 PID/sec | Up to 5000 frames/sec |
| **Latency** | 5-15 ms per PID | ~0 ms (real-time!) |
| **BT Overhead** | High (TX+RX) | Low (RX only) |
| **Parsing** | Simple (known PIDs) | Complex (raw CAN IDs) |
| **Required Knowledge** | Standard OBD-II | Vehicle-specific |
| **RT Usefulness** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## 13. ISO-TP — Multi-Frame Communication

### 13.1 When Is Multi-Frame Needed?

A standard CAN frame has **8 bytes** of data. After subtracting the ISO-TP header (1 byte), **7 bytes** are available for OBD data. If the response exceeds 7 bytes (e.g., VIN, DTC list, Mode 06), **ISO-TP segmentation (ISO 15765-2)** is needed.

### 13.2 ISO-TP Frame Types

| Type | Byte 0 | Description |
|---|---|---|
| **Single Frame (SF)** | `0x0n` | Complete message (n = length, 1-7 bytes) |
| **First Frame (FF)** | `0x1n nn` | First frame of multi-frame (nn nn = total length) |
| **Consecutive Frame (CF)** | `0x2n` | Next frame (n = sequence number 0-F) |
| **Flow Control (FC)** | `0x30` | Flow control (BS + STmin) |

### 13.3 Flow Control Parameters

| Parameter | Description | Range |
|---|---|---|
| **FS** | Flow Status (0=CTS, 1=Wait, 2=Overflow) | 0-2 |
| **BS** | Block Size (0 = no limit) | 0-255 |
| **STmin** | Separation Time Minimum | 0-127 ms / 100-900 µs |

> [!NOTE]
> OBDLink MX+ handles ISO-TP flow control **automatically**. You don't need to manually manage FC frames, unless you use `ATFC SM 1` (user-defined flow control).

---

## 14. Android Application Architecture

### 14.1 Threading Model

```
┌──────────────────────────────────────────────────────┐
│                ANDROID APP ARCHITECTURE               │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌─────────────────┐                                 │
│  │   UI THREAD      │ ← Display data                 │
│  │   (Main Thread)  │ ← Gauge animations             │
│  └────────┬─────────┘                                │
│           │ LiveData / StateFlow                     │
│           │                                          │
│  ┌────────▼─────────┐                                │
│  │   DATA LAYER     │ ← Parsing, formulas            │
│  │   (Repository)   │ ← Buffering, interpolation     │
│  └────────┬─────────┘                                │
│           │                                          │
│  ┌────────▼─────────┐                                │
│  │   IO THREAD      │ ← Dedicated polling loop       │
│  │   (BT Worker)    │ ← OutputStream.write + flush   │
│  │                  │ ← InputStream.read blocking    │
│  │   DEDICATED!     │ ← NEVER block the UI Thread!   │
│  └────────┬─────────┘                                │
│           │ BluetoothSocket RFCOMM                   │
│           │                                          │
│  ┌────────▼─────────┐                                │
│  │   OBDLink MX+    │ ← Bluetooth Classic SPP        │
│  │   (Hardware)     │ ← STN2120 → CAN Bus            │
│  └──────────────────┘                                │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 14.2 Key Implementation Rules

1. **Dedicated I/O thread** — ALL BT communication on a separate thread/coroutine
2. **Never block the UI** — Use `LiveData`, `StateFlow`, or `Handler` for UI updates
3. **`flush()` after every write** — Forces immediate transmission
4. **Parse inline** — Don't buffer entire responses, parse byte by byte
5. **Ring buffer** — For real-time data, use ring buffer instead of lists
6. **Atomic state** — Connection state as `AtomicBoolean` / `StateFlow<ConnectionState>`

### 14.3 Pseudocode — Real-Time Polling Engine

```kotlin
class OBDPollingEngine(
    private val socket: BluetoothSocket,
    private val onData: (PIDData) -> Unit
) : Thread("OBD-RT-Polling") {

    private val output = socket.outputStream
    private val input = socket.inputStream
    private val buffer = ByteArray(1024)
    
    @Volatile
    var isRunning = true
    
    // Priority PID groups
    private val criticalPIDs = listOf("010C", "010D", "0111") // RPM, Speed, Throttle
    private val highPIDs = listOf("0104", "0110")             // Load, MAF
    private val mediumPIDs = listOf("0105", "010F", "010B")   // Temps, Pressure
    private val lowPIDs = listOf("012F", "0142", "015C")      // Fuel, Voltage, Oil
    
    private var cycleCounter = 0L
    
    override fun run() {
        while (isRunning && socket.isConnected) {
            try {
                // ALWAYS: Critical PIDs
                for (pid in criticalPIDs) {
                    val response = sendAndReceive(pid)
                    onData(parsePIDResponse(response))
                }
                
                // Every 2nd cycle: High priority
                if (cycleCounter % 2 == 0L) {
                    for (pid in highPIDs) {
                        val response = sendAndReceive(pid)
                        onData(parsePIDResponse(response))
                    }
                }
                
                // Every 5th cycle: Medium priority
                if (cycleCounter % 5 == 0L) {
                    for (pid in mediumPIDs) {
                        val response = sendAndReceive(pid)
                        onData(parsePIDResponse(response))
                    }
                }
                
                // Every 10th cycle: Low priority
                if (cycleCounter % 10 == 0L) {
                    for (pid in lowPIDs) {
                        val response = sendAndReceive(pid)
                        onData(parsePIDResponse(response))
                    }
                }
                
                cycleCounter++
                
            } catch (e: IOException) {
                handleDisconnection(e)
            }
        }
    }
    
    private fun sendAndReceive(command: String): String {
        // Send
        output.write("$command\r".toByteArray(Charsets.US_ASCII))
        output.flush()  // ⭐ CRITICAL!
        
        // Receive until '>' prompt
        val sb = StringBuilder()
        while (true) {
            val b = input.read()
            if (b == -1) throw IOException("Stream closed")
            val c = b.toChar()
            if (c == '>') break
            sb.append(c)
        }
        return sb.toString().trim()
    }
}
```

### 14.4 Foreground Service

For continuous background operation:

```kotlin
class OBDForegroundService : Service() {
    // Use Foreground Service with notification
    // so Android doesn't kill the process while driving
    // START_STICKY ensures restart after OOM kill
}
```

---

## 15. Connection Stability and Reconnect

### 15.1 Common Causes of Connection Loss

| Cause | Solution |
|---|---|
| Out of BT range | Auto-reconnect with backoff |
| Android killed process | Foreground Service + START_STICKY |
| ECU sleep / ignition off | Check `ATIGN` (ignition state) |
| Bluetooth stack crash | `socket.close()` + new socket |
| Adapter buffer overflow | Reduce polling frequency |
| ECU response timeout | Retry with `ATPC` (protocol close + reopen) |

### 15.2 Reconnect Strategy

```
┌────────────────────────────────────┐
│     RECONNECT STRATEGY              │
├────────────────────────────────────┤
│                                    │
│  1. Detect disconnect (IOException)│
│  2. socket.close()                 │
│  3. Wait 500ms                     │
│  4. New socket (createRfcomm...)   │
│  5. socket.connect()              │
│  6. Re-initialization (ATZ...)     │
│  7. Resume polling                 │
│                                    │
│  Retry policy:                     │
│  - Attempt 1: 500ms delay          │
│  - Attempt 2: 1s delay             │
│  - Attempt 3: 2s delay             │
│  - Attempt 4: 5s delay             │
│  - Attempt 5+: 10s delay           │
│  - Max attempts: ∞ (user cancel)   │
│                                    │
└────────────────────────────────────┘
```

### 15.3 Heartbeat / Watchdog

```kotlin
// Watchdog: if no response for 5 seconds, force reconnect
val watchdog = Timer()
watchdog.schedule(object : TimerTask() {
    override fun run() {
        if (System.currentTimeMillis() - lastResponseTime > 5000) {
            forceReconnect()
        }
    }
}, 1000, 1000)
```

---

## 16. Power Management — BatterySaver™

### 16.1 Automatic Sleep

OBDLink MX+ features **BatterySaver™** technology, which automatically switches the adapter to low-power mode (~2 mA) after a defined period of inactivity.

### 16.2 Power Management Commands

| Command | Description |
|---|---|
| `ATLP` | Enter Low Power mode |
| `STSLEEP` | Enter deep sleep (STN exclusive) |
| `STSLCS` | Sleep with Low Current State |
| `STWBR` | Wake on Bluetooth Request |

### 16.3 Ignition Control

```
ATIGN          → Check ignition state ("ON" / "OFF")

// Strategy:
// If ATIGN = OFF → Enter sleep mode
// If ATIGN = ON → Continue polling
```

---

## 17. Technical Limits and Bottlenecks

### 17.1 Bottlenecks — Pipeline

```
┌─────────────────────────────────────────────────────────┐
│              BOTTLENECK ANALYSIS                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. ECU Processing Time         ~2-5 ms per PID         │
│     (ECU response time on CAN)                          │
│                                                         │
│  2. CAN Bus Transmission        ~0.2 ms per frame       │
│     (physical CAN 500K transmission)                    │
│                                                         │
│  3. STN2120 Processing          ~1-3 ms                 │
│     (CAN → ASCII decoding)                              │
│                                                         │
│  4. Bluetooth Transmission      ~5-15 ms round-trip     │
│     (BIGGEST BOTTLENECK!)                               │
│                                                         │
│  5. Android Processing          ~0.5-2 ms               │
│     (parsing, UI update)                                │
│                                                         │
│  TOTAL per PID:                 ~8-25 ms                │
│  Real throughput:               40-120 PIDs/sec         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 17.2 Absolute Limits

| Parameter | Limit |
|---|---|
| **Max PIDs/sec (1 PID continuous)** | ~100-130 |
| **Max PIDs/sec (mixed polling)** | ~40-80 |
| **Max CAN frames/sec (monitoring)** | ~2000-3000 (with BT bottleneck) |
| **Min latency per PID** | ~5-8 ms (CAN + STN + BT) |
| **Bluetooth MTU** | 1023 bytes (RFCOMM) |
| **Concurrent ECU connections** | No limit (but sequential) |

### 17.3 OBD-II vs Manufacturer Limitations

| Feature | Standard OBD-II (Mode 01) | Manufacturer-Specific (Mode 22) |
|---|---|---|
| **Available PIDs** | ~100 standardized | 1000+ per model |
| **Multi-PID request** | ❌ No | ✅ Yes |
| **ECU update rate** | 10-50 Hz | Up to 100+ Hz |
| **Turbo/Boost data** | ⚠️ Limited | ✅ Full |
| **Transmission data** | ⚠️ Basic | ✅ Full |
| **ABS/ESP data** | ❌ No (different ECU) | ✅ Yes |

---

## 18. Summary — Ultra-Fast RT Strategy

### 18.1 Optimal Configuration

```
┌──────────────────────────────────────────────────────────┐
│         GOLDEN CONFIGURATION FOR ULTRA-FAST RT            │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  HARDWARE:                                               │
│    ✅ OBDLink MX+ (STN2120)                              │
│    ✅ Bluetooth 3.0 Classic (SPP/RFCOMM)                 │
│                                                          │
│  INIT SEQUENCE:                                          │
│    ATZ → ATE0 → ATL0 → ATS0 → ATH1                     │
│    ATAT2 → ATSP6 → ATCRA 7E8 → ATSH 7E0                │
│                                                          │
│  PROTOCOL:                                               │
│    ✅ CAN 11-bit 500 Kbps (Protocol 6)                   │
│    ✅ Physical addressing (7E0 → 7E8)                    │
│    ✅ Response filtering (ATCRA 7E8)                     │
│                                                          │
│  POLLING:                                                │
│    ✅ Prioritized PID groups (A/B/C/D)                   │
│    ✅ Zero-wait pipeline                                 │
│    ✅ STPX batch commands                                │
│    ✅ flush() after every write                          │
│                                                          │
│  ANDROID:                                                │
│    ✅ Dedicated IO thread (not Coroutine Main!)           │
│    ✅ Foreground Service (continuous operation)           │
│    ✅ Ring buffer (not ArrayList)                         │
│    ✅ StateFlow/LiveData → UI                            │
│    ✅ Auto-reconnect with exponential backoff            │
│                                                          │
│  EXPECTED PERFORMANCE:                                   │
│    📊 3-4 critical PIDs @ 30-50 Hz                       │
│    📊 2-3 high PIDs @ 15-25 Hz                           │
│    📊 3 medium PIDs @ 6-10 Hz                            │
│    📊 3 low PIDs @ 3-5 Hz                                │
│    📊 TOTAL: ~80-120 PID samples/sec                     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 18.2 Application Operating Modes

| Mode | Description | PIDs | Refresh Rate |
|---|---|---|---|
| **🏎️ Race Mode** | RPM + Speed + Throttle | 3 | 40-60 Hz |
| **📊 Dashboard Mode** | 8-10 basic PIDs | 8-10 | 10-20 Hz |
| **🔧 Diagnostic Mode** | All available + DTCs | 15-20+ | 3-5 Hz |
| **📡 Sniff Mode** | Passive CAN Monitoring | ∞ | Bus rate |

### 18.3 Key Documentation Sources

| Source | Description |
|---|---|
| **SAE J1979** | OBD-II standard service modes and PIDs |
| **ISO 15765-4** | OBD-II on CAN |
| **ISO 15765-2** | ISO-TP (multi-frame CAN) |
| **ISO 14230-4** | KWP2000 |
| **ISO 9141-2** | K-Line protocol |
| **SAE J1850** | PWM/VPW protocols |
| **SAE J1939** | Heavy-duty CAN |
| **ELM327 Datasheet** | AT command reference |
| **STN11xx/STN2120 FRPM** | Family Reference & Programming Manual (ScanTool.net) |
| **Android BT Developer Guide** | developer.android.com/guide/topics/connectivity/bluetooth |
| **OBDLink Support** | obdlink.com/support |

---

> [!IMPORTANT]
> **This documentation is a complete technical foundation** for building an Android application based on OBDLink MX+ in real-time mode. Key findings:
> 1. **CAN 500K + ATAT2 + Physical Addressing** = foundation of speed
> 2. **STPX** = unique advantage of STN2120 over ELM327
> 3. **Bluetooth Classic SPP** is optimal (not BLE)
> 4. **PID prioritization** is critical for UX
> 5. **CAN Monitoring** offers the lowest latency but requires vehicle-specific CAN ID knowledge
> 6. **The bottleneck is Bluetooth**, not the CAN bus or ECU

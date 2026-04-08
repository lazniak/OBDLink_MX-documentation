# Protocol Stack — OBD-II Deep Dive

> **Scope:** Complete protocol reference for all vehicle communication layers supported by OBDLink MX+

---

## Protocol Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                       APPLICATION LAYER                          │
│              SAE J1979 (OBD-II Services/Modes)                   │
├──────────────────────────────────────────────────────────────────┤
│                       TRANSPORT LAYER                            │
│              ISO 15765-2 (ISO-TP / CAN Transport)                │
├──────────────────────────────────────────────────────────────────┤
│                       NETWORK LAYER                              │
│              ISO 15765-4 (OBD on CAN)                            │
├──────────────────────────────────────────────────────────────────┤
│                       DATA LINK LAYER                            │
│              ISO 11898 (CAN 2.0A / 2.0B)                         │
├──────────────────────────────────────────────────────────────────┤
│                       PHYSICAL LAYER                             │
│              CAN High / CAN Low (differential pair)              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. CAN Bus — ISO 15765-4

### Supported CAN Variants

| Variant | CAN ID Length | Bus Speed | Standard | ATSP Code |
|---------|--------------|-----------|----------|-----------|
| **CAN 11-bit 500K** | 11-bit | 500 Kbps | ISO 15765-4 | **6** ⭐ |
| CAN 29-bit 500K | 29-bit | 500 Kbps | ISO 15765-4 | 7 |
| CAN 11-bit 250K | 11-bit | 250 Kbps | ISO 15765-4 | 8 |
| CAN 29-bit 250K | 29-bit | 250 Kbps | ISO 15765-4 | 9 |
| J1939 CAN | 29-bit | 250 Kbps | SAE J1939 | A |
| User CAN 11-bit | 11-bit | Custom | ISO 11898 | B |
| User CAN 29-bit | 29-bit | Custom | ISO 11898 | C |

### CAN Frame Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    CAN 2.0A FRAME (11-bit)                   │
├─────┬────┬─────┬─────────────────────────┬─────┬─────┬─────┤
│ SOF │ ID │ RTR │        DATA FIELD       │ CRC │ ACK │ EOF │
│ 1b  │11b │ 1b  │  DLC(4b) + Data(0-8B)  │ 15b │ 2b  │ 7b  │
└─────┴────┴─────┴─────────────────────────┴─────┴─────┴─────┘
```

### OBD-II CAN Addressing

| Type | Request ID | Response ID | Usage |
|------|-----------|-------------|-------|
| **Functional** (broadcast) | `0x7DF` | `0x7E8`–`0x7EF` | Query all ECUs |
| **Physical** Engine ECU | `0x7E0` | `0x7E8` | Direct to engine |
| Physical Transmission | `0x7E1` | `0x7E9` | Direct to transmission |
| Physical ABS/ESP | `0x7E2` | `0x7EA` | Direct to brakes/stability |
| Physical Airbags | `0x7E3` | `0x7EB` | Direct to SRS |
| Physical Body | `0x7E4` | `0x7EC` | Direct to BCM |

### OBD-II Request Frame on CAN

```
CAN ID: 7DF (functional) or 7E0 (physical)
┌────────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ Byte 0 │  B1  │  B2  │  B3  │  B4  │  B5  │  B6  │  B7  │
│ Length │ Mode │ PID  │ Pad  │ Pad  │ Pad  │ Pad  │ Pad  │
│  0x02  │ 0x01 │ 0x0C │ 0x00 │ 0x00 │ 0x00 │ 0x00 │ 0x00 │
└────────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
         ↑ Request: Mode 01, PID 0C (RPM)
```

### OBD-II Response Frame on CAN

```
CAN ID: 7E8 (from engine ECU)
┌────────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ Byte 0 │  B1  │  B2  │  B3  │  B4  │  B5  │  B6  │  B7  │
│ Length │ Mode │ PID  │  A   │  B   │ Pad  │ Pad  │ Pad  │
│  0x04  │ 0x41 │ 0x0C │ 0x1A │ 0xF8 │ 0x00 │ 0x00 │ 0x00 │
└────────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
         ↑ Response: Mode 41 (01+40), PID 0C, Data: 1AF8 → 1726 RPM
```

---

## 2. ISO-TP — ISO 15765-2 (CAN Transport Protocol)

When OBD-II responses exceed 7 data bytes (single CAN frame), ISO-TP segments the message across multiple frames.

### Frame Types

| Type | Nibble | PCI Byte(s) | Purpose |
|------|--------|------------|---------|
| **Single Frame (SF)** | `0` | `0N` (N=length) | Complete message ≤7 bytes |
| **First Frame (FF)** | `1` | `1N NN` (NNN=total length) | Start of multi-frame |
| **Consecutive Frame (CF)** | `2` | `2N` (N=sequence 0–F) | Continuation data |
| **Flow Control (FC)** | `3` | `30 BS STmin` | Receiver → Sender control |

### Single Frame Example (Normal OBD Response)

```
7E8  04  41 0C 1A F8  00 00 00
     ↑SF  ↑ 4 bytes of data
```

### Multi-Frame Example (VIN Request)

```
Request:   7DF  02  09 02  00 00 00 00 00
                    ↑ Mode 09, PID 02 (VIN)

Response:
Frame 1 (FF):  7E8  10 14  49 02 01  57 42 41   ← First Frame, 20 bytes total
Frame 2 (FC):  7E0  30 00 00  00 00 00 00 00     ← Flow Control (from adapter)
Frame 3 (CF):  7E8  21  47 47 33 34 38 5A        ← Consecutive Frame #1
Frame 4 (CF):  7E8  22  4C 53 31 32 33 34        ← Consecutive Frame #2
Frame 5 (CF):  7E8  23  35 36 00 00 00 00        ← Consecutive Frame #3
```

### Flow Control Parameters

| Param | Field | Description | Values |
|-------|-------|-------------|--------|
| **FS** | Flow Status | 0 = Continue, 1 = Wait, 2 = Overflow | 0–2 |
| **BS** | Block Size | Frames before next FC (0 = unlimited) | 0–255 |
| **STmin** | Min Separation Time | Delay between consecutive frames | See below |

### STmin Encoding

| Hex Value | Meaning |
|-----------|---------|
| `0x00`–`0x7F` | 0–127 milliseconds |
| `0x80`–`0xF0` | Reserved |
| `0xF1`–`0xF9` | 100–900 microseconds |

> [!NOTE]
> OBDLink MX+ handles ISO-TP flow control **automatically**. The adapter generates FC frames and reassembles multi-frame responses before sending the complete result over Bluetooth.

---

## 3. Legacy Protocols

### ISO 9141-2 (K-Line)

| Characteristic | Value |
|---------------|-------|
| Speed | 10.4 Kbps |
| Initialization | 5-baud |
| Physical Layer | Single wire (K-line) |
| Common Vehicles | European, Asian (pre-2008) |
| ATSP Code | 3 |

### ISO 14230-4 (KWP2000)

| Variant | Init Method | ATSP Code |
|---------|-------------|-----------|
| KWP2000 Slow | 5-baud init | 4 |
| KWP2000 Fast | Fast init | 5 |

| Characteristic | Value |
|---------------|-------|
| Speed | 10.4 Kbps |
| Physical Layer | K-line (+ optional L-line) |
| Common Vehicles | European (2000–2010) |

### SAE J1850 PWM (Ford)

| Characteristic | Value |
|---------------|-------|
| Speed | 41.6 Kbps |
| Encoding | Pulse Width Modulation |
| Physical Layer | 2-wire |
| Common Vehicles | Ford (pre-2008) |
| ATSP Code | 1 |

### SAE J1850 VPW (GM)

| Characteristic | Value |
|---------------|-------|
| Speed | 10.4 Kbps |
| Encoding | Variable Pulse Width |
| Physical Layer | 1-wire |
| Common Vehicles | GM (pre-2008) |
| ATSP Code | 2 |

---

## 4. Extended Protocols (MX+ Only)

### SW-CAN (Single Wire CAN)

| Characteristic | Value |
|---------------|-------|
| Speed | 33.3 Kbps |
| Physical Layer | Single wire |
| Common Vehicles | GM body control, interior modules |
| Usage | Window, Mirror, Seat, HVAC control |

### MS-CAN (Medium Speed CAN)

| Characteristic | Value |
|---------------|-------|
| Speed | 125 Kbps |
| Physical Layer | 2-wire differential |
| Common Vehicles | Ford body control modules |
| Usage | Instrument cluster, HVAC, BCM |

---

## 5. Protocol Auto-Detection

When using `ATSP 0`, the OBDLink MX+ attempts protocols in this order:

```
1. ISO 15765-4 CAN 11-bit 500K  (most common modern vehicles)
2. ISO 15765-4 CAN 29-bit 500K
3. ISO 15765-4 CAN 11-bit 250K
4. ISO 15765-4 CAN 29-bit 250K
5. ISO 14230-4 KWP Fast Init
6. ISO 14230-4 KWP 5-Baud Init
7. ISO 9141-2
8. SAE J1850 VPW
9. SAE J1850 PWM
```

Auto-detection typically takes **2–5 seconds**. For production apps targeting known vehicles, skip this by forcing `ATSP 6`.

---

## 6. Protocol Selection Strategy

```
┌─────────────────────────────────────────────┐
│         PROTOCOL DECISION TREE               │
├─────────────────────────────────────────────┤
│                                             │
│  Vehicle Year ≥ 2008?                       │
│  ├── YES → ATSP 6 (CAN 11-bit 500K)        │
│  │         99% of modern vehicles           │
│  │                                          │
│  └── NO → Vehicle Model Known?              │
│       ├── Ford (US) → Try ATSP 1 (PWM)     │
│       ├── GM (US) → Try ATSP 2 (VPW)       │
│       ├── European → Try ATSP 5 (KWP Fast) │
│       ├── Asian → Try ATSP 3 (ISO 9141)    │
│       └── Unknown → ATSP 0 (Auto-detect)   │
│                                             │
└─────────────────────────────────────────────┘
```

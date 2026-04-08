# PID Reference — OBD-II Mode 01

> **Standard:** SAE J1979 / ISO 15031-5  
> **Mode:** `01` — Request Current Powertrain Diagnostic Data  
> **Request format:** `01 <PID>`  
> **Response format:** `41 <PID> <A> [B] [C] [D]`

---

## PID Support Discovery

Before polling PIDs, discover which ones your vehicle supports:

| PID | Request | Returns | Covers PIDs |
|-----|---------|---------|-------------|
| `00` | `0100` | 4-byte bitmask | `01` – `20` |
| `20` | `0120` | 4-byte bitmask | `21` – `40` |
| `40` | `0140` | 4-byte bitmask | `41` – `60` |
| `60` | `0160` | 4-byte bitmask | `61` – `80` |
| `80` | `0180` | 4-byte bitmask | `81` – `A0` |
| `A0` | `01A0` | 4-byte bitmask | `A1` – `C0` |
| `C0` | `01C0` | 4-byte bitmask | `C1` – `E0` |

### Bitmask Decoding

```
Response: 41 00 BE 1F A8 13

Byte A = 0xBE = 1011 1110  →  PIDs 01,03,04,05,06,07
Byte B = 0x1F = 0001 1111  →  PIDs 0C,0D,0E,0F,10
Byte C = 0xA8 = 1010 1000  →  PIDs 11,13,15
Byte D = 0x13 = 0001 0011  →  PIDs 1C,1F,20
```

---

## ⚡ Priority Group A — Critical (Poll Every Cycle)

These PIDs update fastest and are essential for real-time gauges.

| PID | Hex | Description | Bytes | Formula | Unit | Min | Max |
|-----|-----|-------------|-------|---------|------|-----|-----|
| 12 | `0C` | **Engine RPM** | 2 | `(256A + B) / 4` | rpm | 0 | 16,383.75 |
| 13 | `0D` | **Vehicle Speed** | 1 | `A` | km/h | 0 | 255 |
| 17 | `11` | **Throttle Position** | 1 | `A × 100 / 255` | % | 0 | 100 |

### Decoding Examples

```
RPM:      41 0C 1A F8  →  (0x1A×256 + 0xF8) / 4 = (26×256 + 248) / 4 = 1726 rpm
Speed:    41 0D 3C     →  0x3C = 60 km/h
Throttle: 41 11 80     →  0x80 × 100 / 255 = 128 × 100 / 255 = 50.2%
```

---

## ⚡ Priority Group B — High (Poll Every 2nd Cycle)

| PID | Hex | Description | Bytes | Formula | Unit | Min | Max |
|-----|-----|-------------|-------|---------|------|-----|-----|
| 4 | `04` | Calculated Engine Load | 1 | `A × 100 / 255` | % | 0 | 100 |
| 16 | `10` | MAF Air Flow Rate | 2 | `(256A + B) / 100` | g/s | 0 | 655.35 |
| 14 | `0E` | Timing Advance | 1 | `A / 2 - 64` | ° bTDC | -64 | 63.5 |

---

## 📊 Priority Group C — Medium (Poll Every 5th Cycle)

| PID | Hex | Description | Bytes | Formula | Unit | Min | Max |
|-----|-----|-------------|-------|---------|------|-----|-----|
| 5 | `05` | Engine Coolant Temp | 1 | `A - 40` | °C | -40 | 215 |
| 15 | `0F` | Intake Air Temperature | 1 | `A - 40` | °C | -40 | 215 |
| 11 | `0B` | Intake Manifold Abs. Pressure | 1 | `A` | kPa | 0 | 255 |
| 10 | `0A` | Fuel Pressure (gauge) | 1 | `A × 3` | kPa | 0 | 765 |

---

## 📦 Priority Group D — Low (Poll Every 10th Cycle)

| PID | Hex | Description | Bytes | Formula | Unit | Min | Max |
|-----|-----|-------------|-------|---------|------|-----|-----|
| 47 | `2F` | Fuel Tank Level Input | 1 | `A × 100 / 255` | % | 0 | 100 |
| 66 | `42` | Control Module Voltage | 2 | `(256A + B) / 1000` | V | 0 | 65.535 |
| 92 | `5C` | Engine Oil Temperature | 1 | `A - 40` | °C | -40 | 215 |
| 31 | `1F` | Run Time Since Engine Start | 2 | `256A + B` | sec | 0 | 65,535 |
| 94 | `5E` | Engine Fuel Rate | 2 | `(256A + B) / 20` | L/h | 0 | 3,276.75 |

---

## Complete Mode 01 PID Table

### PIDs `01` – `20`

| PID | Description | Bytes | Formula | Unit |
|-----|-------------|-------|---------|------|
| `01` | Monitor status since DTCs cleared | 4 | Bit-encoded | — |
| `02` | DTC that caused freeze frame | 2 | Encoded DTC | — |
| `03` | Fuel system status | 2 | Bit-encoded | — |
| `04` | Calculated engine load | 1 | `A × 100 / 255` | % |
| `05` | Engine coolant temperature | 1 | `A - 40` | °C |
| `06` | Short-term fuel trim — Bank 1 | 1 | `(A - 128) × 100 / 128` | % |
| `07` | Long-term fuel trim — Bank 1 | 1 | `(A - 128) × 100 / 128` | % |
| `08` | Short-term fuel trim — Bank 2 | 1 | `(A - 128) × 100 / 128` | % |
| `09` | Long-term fuel trim — Bank 2 | 1 | `(A - 128) × 100 / 128` | % |
| `0A` | Fuel pressure (gauge) | 1 | `A × 3` | kPa |
| `0B` | Intake manifold absolute pressure | 1 | `A` | kPa |
| `0C` | Engine RPM | 2 | `(256A + B) / 4` | rpm |
| `0D` | Vehicle speed | 1 | `A` | km/h |
| `0E` | Timing advance | 1 | `A / 2 - 64` | ° bTDC |
| `0F` | Intake air temperature | 1 | `A - 40` | °C |
| `10` | MAF air flow rate | 2 | `(256A + B) / 100` | g/s |
| `11` | Throttle position | 1 | `A × 100 / 255` | % |
| `12` | Commanded secondary air status | 1 | Bit-encoded | — |
| `13` | O2 sensors present (2 banks) | 1 | Bit-encoded | — |
| `14` | O2 Sensor 1 — Voltage + Trim | 2 | `A / 200`, `(B - 128) × 100 / 128` | V, % |
| `15` | O2 Sensor 2 — Voltage + Trim | 2 | Same as `14` | V, % |
| `16` | O2 Sensor 3 — Voltage + Trim | 2 | Same as `14` | V, % |
| `17` | O2 Sensor 4 — Voltage + Trim | 2 | Same as `14` | V, % |
| `18` | O2 Sensor 5 — Voltage + Trim | 2 | Same as `14` | V, % |
| `19` | O2 Sensor 6 — Voltage + Trim | 2 | Same as `14` | V, % |
| `1A` | O2 Sensor 7 — Voltage + Trim | 2 | Same as `14` | V, % |
| `1B` | O2 Sensor 8 — Voltage + Trim | 2 | Same as `14` | V, % |
| `1C` | OBD standards compliance | 1 | Lookup table | — |
| `1D` | O2 sensors present (4 banks) | 1 | Bit-encoded | — |
| `1E` | AUX input status | 1 | Bit-encoded | — |
| `1F` | Run time since engine start | 2 | `256A + B` | sec |

### PIDs `21` – `40`

| PID | Description | Bytes | Formula | Unit |
|-----|-------------|-------|---------|------|
| `21` | Distance traveled with MIL on | 2 | `256A + B` | km |
| `22` | Fuel rail pressure (manifold vac) | 2 | `(256A + B) × 0.079` | kPa |
| `23` | Fuel rail gauge pressure (diesel) | 2 | `(256A + B) × 10` | kPa |
| `24` | O2 Sensor 1 — λ + Voltage | 4 | `(256A+B) × 2/65536`, `(256C+D) × 8/65536` | ratio, V |
| `25`–`2B` | O2 Sensors 2–8 — λ + Voltage | 4 | Same as `24` | ratio, V |
| `2C` | Commanded EGR | 1 | `A × 100 / 255` | % |
| `2D` | EGR error | 1 | `(A - 128) × 100 / 128` | % |
| `2E` | Commanded evaporative purge | 1 | `A × 100 / 255` | % |
| `2F` | Fuel tank level input | 1 | `A × 100 / 255` | % |
| `30` | Warm-ups since codes cleared | 1 | `A` | count |
| `31` | Distance traveled since codes cleared | 2 | `256A + B` | km |
| `32` | Evap. system vapor pressure | 2 | `((256A + B) - 32768) / 4` | Pa |
| `33` | Absolute barometric pressure | 1 | `A` | kPa |
| `34`–`3B` | O2 Sensors 1–8 — λ + Current | 4 | Complex | ratio, mA |
| `3C` | Catalyst temp — Bank 1, Sensor 1 | 2 | `(256A + B) / 10 - 40` | °C |
| `3D` | Catalyst temp — Bank 2, Sensor 1 | 2 | Same as `3C` | °C |
| `3E` | Catalyst temp — Bank 1, Sensor 2 | 2 | Same as `3C` | °C |
| `3F` | Catalyst temp — Bank 2, Sensor 2 | 2 | Same as `3C` | °C |

### PIDs `41` – `60`

| PID | Description | Bytes | Formula | Unit |
|-----|-------------|-------|---------|------|
| `41` | Monitor status (current cycle) | 4 | Bit-encoded | — |
| `42` | Control module voltage | 2 | `(256A + B) / 1000` | V |
| `43` | Absolute load value | 2 | `(256A + B) × 100 / 255` | % |
| `44` | Commanded air-fuel equiv. ratio | 2 | `(256A + B) × 2 / 65536` | ratio |
| `45` | Relative throttle position | 1 | `A × 100 / 255` | % |
| `46` | Ambient air temperature | 1 | `A - 40` | °C |
| `47` | Abs. throttle position B | 1 | `A × 100 / 255` | % |
| `48` | Abs. throttle position C | 1 | `A × 100 / 255` | % |
| `49` | Accel. pedal position D | 1 | `A × 100 / 255` | % |
| `4A` | Accel. pedal position E | 1 | `A × 100 / 255` | % |
| `4B` | Accel. pedal position F | 1 | `A × 100 / 255` | % |
| `4C` | Commanded throttle actuator | 1 | `A × 100 / 255` | % |
| `4D` | Time run with MIL on | 2 | `256A + B` | min |
| `4E` | Time since trouble codes cleared | 2 | `256A + B` | min |
| `51` | Fuel type | 1 | Lookup | — |
| `52` | Ethanol fuel percentage | 1 | `A × 100 / 255` | % |
| `5A` | Relative accel. pedal position | 1 | `A × 100 / 255` | % |
| `5B` | Hybrid battery pack remaining life | 1 | `A × 100 / 255` | % |
| `5C` | Engine oil temperature | 1 | `A - 40` | °C |
| `5D` | Fuel injection timing | 2 | `(256A + B - 26880) / 128` | ° |
| `5E` | Engine fuel rate | 2 | `(256A + B) / 20` | L/h |

---

## Mode 09 — Vehicle Information (One-Time Reads)

| PID | Description | Request |
|-----|-------------|---------|
| `02` | Vehicle Identification Number (VIN) | `0902` |
| `04` | Calibration ID | `0904` |
| `06` | Calibration Verification Numbers | `0906` |
| `0A` | ECU Name | `090A` |

---

## Byte-to-Decimal Quick Reference

| Hex | Dec | Hex | Dec | Hex | Dec | Hex | Dec |
|-----|-----|-----|-----|-----|-----|-----|-----|
| `00` | 0 | `40` | 64 | `80` | 128 | `C0` | 192 |
| `10` | 16 | `50` | 80 | `90` | 144 | `D0` | 208 |
| `20` | 32 | `60` | 96 | `A0` | 160 | `E0` | 224 |
| `30` | 48 | `70` | 112 | `B0` | 176 | `FF` | 255 |

<p align="center">
  <img src="docs/assets/logo.svg" width="120" alt="car-RT logo" />
</p>

<h1 align="center">car-RT</h1>

<p align="center">
  <strong>Ultra-Fast Real-Time OBD-II Telemetry for Android</strong><br/>
  <sub>Powered by OBDLink MX+ &bull; STN2120 &bull; Bluetooth Classic SPP</sub>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/platform-Android-3DDC84?style=flat-square&logo=android&logoColor=white" alt="Android" />
  <img src="https://img.shields.io/badge/hardware-OBDLink_MX+-0078D4?style=flat-square" alt="OBDLink MX+" />
  <img src="https://img.shields.io/badge/protocol-CAN_500K-FF6B35?style=flat-square" alt="CAN 500K" />
  <img src="https://img.shields.io/badge/chip-STN2120-8B5CF6?style=flat-square" alt="STN2120" />
  <img src="https://img.shields.io/badge/bluetooth-3.0_SPP-0082FC?style=flat-square&logo=bluetooth&logoColor=white" alt="Bluetooth 3.0" />
  <img src="https://img.shields.io/badge/status-R&D-EAB308?style=flat-square" alt="Status" />
</p>

---

## 🏁 Overview

**car-RT** is an Android application engineered for **ultra-low-latency, real-time vehicle telemetry** through the **OBDLink MX+** Bluetooth OBD-II adapter. The project targets the absolute performance ceiling of the OBD-II standard, achieving **80–120 PID samples/second** with sub-15ms latency per parameter.

```
  ┌──────────┐     Bluetooth 3.0     ┌──────────┐     CAN 500K      ┌──────────┐
  │ Android  │ ◄──── SPP/RFCOMM ────► │ OBDLink  │ ◄──── ISO 15765 ──► │ Vehicle  │
  │   App    │     ~5-15ms RTT       │   MX+    │     ~2-5ms RTT    │   ECU    │
  └──────────┘                       └──────────┘                    └──────────┘
```

## ⚡ Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Critical PIDs (RPM, Speed, Throttle) | **30–50 Hz** | Priority Group A |
| High PIDs (Load, MAF) | **15–25 Hz** | Priority Group B |
| Medium PIDs (Temps, Pressure) | **6–10 Hz** | Priority Group C |
| Low PIDs (Fuel, Voltage, Oil) | **3–5 Hz** | Priority Group D |
| Total PID throughput | **80–120 samples/sec** | Combined all groups |
| End-to-end latency | **< 25 ms** | BT + CAN + parse |

## 🧬 Architecture

```
┌───────────────────────────────────────────────────┐
│                    car-RT App                      │
├───────────────────────────────────────────────────┤
│                                                   │
│   ┌─────────────┐    StateFlow    ┌────────────┐  │
│   │  UI Layer   │ ◄────────────── │   Data     │  │
│   │  (Compose)  │                 │   Layer    │  │
│   └─────────────┘                 └─────┬──────┘  │
│                                         │         │
│                                   ┌─────▼──────┐  │
│                                   │ BT I/O     │  │
│                                   │ Thread     │  │
│                                   │ (Polling)  │  │
│                                   └─────┬──────┘  │
│                                         │         │
├─────────────────────────────────────────┼─────────┤
│              Bluetooth SPP/RFCOMM       │         │
├─────────────────────────────────────────┼─────────┤
│              OBDLink MX+ (STN2120)      │         │
├─────────────────────────────────────────┼─────────┤
│              CAN Bus (ISO 15765-4)      │         │
└─────────────────────────────────────────┴─────────┘
```

## 📖 Documentation

All technical documentation lives in [`docs/`](docs/):

| Document | Description |
|----------|-------------|
| [**Technical Reference**](docs/technical-reference.md) | Complete hardware, protocol & implementation guide |
| [**AT Command Reference**](docs/commands/at-commands.md) | ELM327-compatible AT command set |
| [**ST Command Reference**](docs/commands/st-commands.md) | STN2120-exclusive extended commands |
| [**PID Reference**](docs/pid-reference.md) | Complete Mode 01 PID table with formulas |
| [**Performance Guide**](docs/performance-guide.md) | Optimization strategies for real-time polling |
| [**Protocol Stack**](docs/protocol-stack.md) | OBD-II protocol deep-dive (CAN, ISO-TP, KWP) |
| [**Android Integration**](docs/android-integration.md) | Bluetooth SPP connection & app architecture |

## 🗂️ Repository Structure

```
car-RT/
├── docs/                          # 📖 Technical documentation
│   ├── technical-reference.md     #    Master reference document
│   ├── pid-reference.md           #    Complete PID table
│   ├── performance-guide.md       #    RT optimization strategies
│   ├── protocol-stack.md          #    Protocol deep-dive
│   ├── android-integration.md     #    Android BT architecture
│   ├── commands/                  #    Command references
│   │   ├── at-commands.md         #    ELM327 AT commands
│   │   └── st-commands.md         #    STN2120 ST commands
│   └── assets/                    #    Diagrams & images
│       └── logo.svg               #    Project logo
├── app/                           # 📱 Android application (future)
├── .gitignore                     #    Git ignore rules
├── LICENSE                        #    Project license
└── README.md                      #    This file
```

## 🔑 Key Technical Decisions

- **Bluetooth Classic SPP** over BLE — higher throughput (~300 Kbps vs ~50 Kbps), lower stream latency
- **Physical CAN addressing** (`7E0→7E8`) over broadcast (`7DF`) — eliminates multi-ECU noise
- **ATAT2 aggressive timing** — dynamically minimizes ECU response wait time
- **STPX batch transactions** — 20–30% fewer Bluetooth round-trips vs sequential AT+PID
- **Prioritized PID groups** — critical data at 50 Hz, secondary at lower frequencies
- **Dedicated I/O thread** — never blocks UI, zero `Thread.sleep()` in polling loop

## 🛠️ Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Kotlin |
| UI Framework | Jetpack Compose |
| Async | Kotlin Coroutines + Flows |
| Bluetooth | Android Bluetooth Classic API (SPP/RFCOMM) |
| Architecture | MVVM + Repository pattern |
| State | StateFlow / SharedFlow |
| Background | Foreground Service (START_STICKY) |
| Hardware | OBDLink MX+ (STN2120, BT 3.0 Class 2) |
| Vehicle Protocol | CAN 500K (ISO 15765-4) |

---

<p align="center">
  <sub>Built for speed. Engineered for precision.</sub>
</p>

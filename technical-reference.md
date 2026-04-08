# OBDLink MX+ Bluetooth × Android — Kompletna Dokumentacja Techniczna
## Baza wiedzy do budowy ultra-szybkiej aplikacji Real-Time OBD-II

---

## Spis Treści

1. [Architektura Sprzętowa OBDLink MX+](#1-architektura-sprzętowa)
2. [Warstwa Komunikacji Bluetooth](#2-warstwa-bluetooth)
3. [Android — Połączenie Bluetooth SPP/RFCOMM](#3-android-bluetooth-connection)
4. [Stos Protokołów OBD-II](#4-stos-protokołów-obd-ii)
5. [Zestaw Komend AT (ELM327-Compatible)](#5-komendy-at)
6. [Rozszerzone Komendy ST (STN-Exclusive)](#6-komendy-st)
7. [Inicjalizacja Urządzenia — Sekwencja Optymalna](#7-inicjalizacja)
8. [Protokoły Pojazdowe — Szczegóły Techniczne](#8-protokoły-pojazdowe)
9. [Tryby Danych OBD-II (SAE J1979)](#9-tryby-danych)
10. [Kompletna Tabela PID Mode 01](#10-tabela-pid)
11. [Optymalizacja Performance — Real-Time Polling](#11-optymalizacja-performance)
12. [CAN Bus Monitoring — Tryb Pasywny](#12-can-monitoring)
13. [ISO-TP — Multi-Frame Communication](#13-iso-tp)
14. [Architektura Aplikacji Android](#14-architektura-android)
15. [Stabilność Połączenia i Reconnect](#15-stabilność)
16. [Zarządzenie Energią — BatterySaver](#16-zarządzanie-energią)
17. [Limity Techniczne i Wąskie Gardła](#17-limity)
18. [Podsumowanie — Strategia Ultra-Fast RT](#18-podsumowanie)

---

## 1. Architektura Sprzętowa OBDLink MX+ {#1-architektura-sprzętowa}

### Chip Główny: STN2120
OBDLink MX+ jest wyposażony w **STN2120** — 32-bitowy procesor OBD od firmy **ScanTool.net (OBD Solutions LLC)**. Jest to następca STN1170 i STN1110, oferujący znacznie wyższą wydajność przetwarzania protokołów.

| Parametr | Wartość |
|---|---|
| **Procesor** | STN2120 (32-bit ARM) |
| **Bluetooth** | Bluetooth 3.0 Classic, Class 2 |
| **Szyfrowanie** | 128-bit data encryption |
| **Zasięg** | ~10m (Class 2) |
| **Kompatybilność firmware** | ELM327 v1.4b + rozszerzenia ST |
| **Zasilanie** | Bezpośrednio z portu OBD-II (12V) |
| **Pobór prądu (aktywny)** | ~50 mA |
| **Pobór prądu (sleep)** | ~2 mA (BatterySaver™) |
| **Wymiary** | Kompaktowy form factor |
| **Cena** | ~$139.95 USD |

### Kluczowe Przewagi nad Klonami ELM327

| Cecha | OBDLink MX+ (STN2120) | Tani klon ELM327 |
|---|---|---|
| **Szybkość przetwarzania** | ~3-5× szybszy | Baseline |
| **Adaptive Timing** | Zaawansowany (ATAT2) | Podstawowy lub brak |
| **CAN Monitoring** | Pełne wsparcie ATMA/STM | Ograniczone/niestabilne |
| **SW-CAN / MS-CAN** | ✅ (Ford/GM networks) | ❌ |
| **Raw CAN** | Pełne wsparcie (ISO 11898) | Ograniczone |
| **Firmware updates** | ✅ Bezpłatne | ❌ |
| **Bezpieczeństwo BT** | 128-bit encryption + przycisk fizyczny | Brak (otwarty broadcast) |
| **STPX batch command** | ✅ | ❌ |
| **Stabilność** | Produkcyjna | Niestabilna |

---

## 2. Warstwa Komunikacji Bluetooth {#2-warstwa-bluetooth}

### Profil: SPP (Serial Port Profile)

OBDLink MX+ wykorzystuje **Bluetooth Classic 3.0** z profilem **SPP (Serial Port Profile)**, co oznacza emulację portu szeregowego RS-232 przez Bluetooth RFCOMM.

| Parametr | Wartość |
|---|---|
| **Profil Bluetooth** | SPP (Serial Port Profile) |
| **Protokół transportowy** | RFCOMM |
| **UUID usługi SPP** | `00001101-0000-1000-8000-00805F9B34FB` |
| **Przepustowość teoretyczna (BT 3.0)** | Do 3 Mbps |
| **Przepustowość praktyczna** | ~100-300 Kbps (po overhead) |
| **Latencja BT Classic** | ~5-15 ms per round-trip |
| **Parowanie** | PIN-based (domyślnie brak PIN / lub `1234`) |
| **Bezpieczeństwo** | Fizyczny przycisk "Connect" + szyfrowanie 128-bit |

> [!IMPORTANT]
> OBDLink MX+ używa **Bluetooth Classic**, NIE BLE (Bluetooth Low Energy). Na Androidzie wymaga to uprawnień `BLUETOOTH`, `BLUETOOTH_ADMIN`, oraz od Android 12+ — `BLUETOOTH_CONNECT`, `BLUETOOTH_SCAN`.

### Bluetooth Classic vs BLE — Dlaczego Classic jest lepszy dla OBD?

| Cecha | Bluetooth Classic (SPP) | BLE |
|---|---|---|
| **Przepustowość** | Wysoka (~300 Kbps+) | Niska (~10-50 Kbps) |
| **Latencja strumienia** | Niska (ciągły stream) | Wyższa (pakietowy) |
| **Tryb połączenia** | Ciągłe, sesyjne | Event-driven |
| **Rozmiar pakietu** | Duży (do 1023B RFCOMM) | Mały (20-244B MTU) |
| **Dla OBD Real-Time** | ✅ Idealny | ⚠️ Kompromis |

**Wniosek:** Bluetooth Classic SPP jest optymalnym wyborem dla ciągłego, niskolatencyjnego strumienia danych OBD-II w trybie real-time.

---

## 3. Android — Połączenie Bluetooth SPP/RFCOMM {#3-android-bluetooth-connection}

### 3.1 Wymagane Uprawnienia (AndroidManifest.xml)

```xml
<!-- Android 11 i niżej -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<!-- Android 12+ (API 31+) -->
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" 
    android:usesPermissionFlags="neverForLocation" />
```

### 3.2 Sekwencja Połączenia

```
┌─────────────────────────────────────────────────────────┐
│              ANDROID BLUETOOTH CONNECTION FLOW           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. BluetoothAdapter.getDefaultAdapter()                │
│     └─> Sprawdź czy BT jest włączony                    │
│                                                         │
│  2. getBondedDevices()                                  │
│     └─> Szukaj "OBDLink MX+" wśród sparowanych          │
│                                                         │
│  3. device.createRfcommSocketToServiceRecord(SPP_UUID)  │
│     └─> UUID: 00001101-0000-1000-8000-00805F9B34FB      │
│                                                         │
│  4. socket.connect()                                    │
│     └─> Blokujące! Wykonaj w osobnym wątku              │
│                                                         │
│  5. socket.getInputStream()                             │
│  6. socket.getOutputStream()                            │
│     └─> Gotowe do komunikacji AT/OBD                    │
│                                                         │
│  7. Inicjalizacja urządzenia (sekwencja AT)             │
│     └─> ATZ → ATE0 → ATL0 → ATS0 → ATH1 → ATAT2      │
│                                                         │
│  8. Pętla pollingu PID (Real-Time Loop)                 │
│     └─> Ciągłe request/response na dedykowanym wątku    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 3.3 Kluczowe Stałe Połączenia

```kotlin
companion object {
    // Standard SPP UUID
    val SPP_UUID: UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB")
    
    // Nazwa urządzenia OBDLink MX+
    const val DEVICE_NAME = "OBDLink MX+"
    
    // Terminatory
    const val COMMAND_TERMINATOR = '\r'    // Carriage Return — koniec komendy
    const val RESPONSE_TERMINATOR = '>'   // Prompt — koniec odpowiedzi
    
    // Timeouty
    const val CONNECT_TIMEOUT_MS = 10_000L
    const val COMMAND_TIMEOUT_MS = 2_000L
    const val INIT_TIMEOUT_MS = 5_000L
    
    // Buffer
    const val READ_BUFFER_SIZE = 1024
}
```

### 3.4 Protokół Komunikacji Tekstowej

Komunikacja z OBDLink MX+ odbywa się w formie **tekstowej ASCII**:

```
Wysyłanie:  "<KOMENDA>\r"          (zakończona Carriage Return 0x0D)
Odbiór:     "<ODPOWIEDŹ>\r\r>"     (zakończona promptem '>')
```

**Przykład:**
```
TX: "010C\r"            ← Zapytanie o RPM
RX: "41 0C 1A F8\r\r>"  ← Odpowiedź z danymi
```

> [!TIP]
> Po każdym `write()` na `OutputStream`, **natychmiast** wywołaj `flush()` aby wymusić wysłanie bufora bez czekania na jego zapełnienie.

### 3.5 Fallback — Insecure RFCOMM

Jeśli `createRfcommSocketToServiceRecord()` zawodzi (np. problemy z parowaniem):

```kotlin
// Fallback: Insecure RFCOMM (bez wymagania parowania)
val socket = device.createInsecureRfcommSocketToServiceRecord(SPP_UUID)

// Alternatywny Fallback: Reflection hack (starsze urządzenia)
val method = device.javaClass.getMethod(
    "createRfcommSocket", Int::class.javaPrimitiveType
)
val socket = method.invoke(device, 1) as BluetoothSocket
```

---

## 4. Stos Protokołów OBD-II {#4-stos-protokołów-obd-ii}

### Obsługiwane Protokoły Pojazdowe

OBDLink MX+ obsługuje **wszystkie** legislowane protokoły OBD-II oraz dodatkowe:

| # | Protokół | Standard | Szybkość | Typowe Pojazdy |
|---|---|---|---|---|
| 0 | Auto-detect | — | — | Automatyczny wybór |
| 1 | SAE J1850 PWM | — | 41.6 Kbps | Ford (starsze) |
| 2 | SAE J1850 VPW | — | 10.4 Kbps | GM (starsze) |
| 3 | ISO 9141-2 | — | 10.4 Kbps | Europejskie, Azjatyckie, Chrysler |
| 4 | ISO 14230-4 KWP (5-baud init) | KWP2000 | 10.4 Kbps | Europejskie (starsze) |
| 5 | ISO 14230-4 KWP (fast init) | KWP2000 | 10.4 Kbps | Europejskie |
| **6** | **ISO 15765-4 CAN (11-bit, 500 Kbps)** | **CAN** | **500 Kbps** | **Większość 2008+** |
| 7 | ISO 15765-4 CAN (29-bit, 500 Kbps) | CAN | 500 Kbps | Trucks, heavy-duty |
| 8 | ISO 15765-4 CAN (11-bit, 250 Kbps) | CAN | 250 Kbps | Niektóre europejskie |
| 9 | ISO 15765-4 CAN (29-bit, 250 Kbps) | CAN | 250 Kbps | Heavy-duty |
| A | SAE J1939 CAN (29-bit, 250 Kbps) | J1939 | 250 Kbps | Ciężarówki |
| B | User-defined CAN (11-bit) | Raw CAN | Konfig. | Custom |
| C | User-defined CAN (29-bit) | Raw CAN | Konfig. | Custom |

> [!IMPORTANT]
> **Protokół 6 (ISO 15765-4 CAN 11-bit 500 Kbps)** to najszybszy i najczęstszy protokół w nowoczesnych pojazdach (2008+). Jest to **DOCELOWY** protokół dla maksymalnej wydajności real-time.

### Dodatkowe Protokoły (tylko MX+)

| Protokół | Opis | Pojazdy |
|---|---|---|
| **SW-CAN** | Single-Wire CAN (33.3 Kbps) | GM (body control) |
| **MS-CAN** | Medium-Speed CAN (125 Kbps) | Ford (body control) |

---

## 5. Zestaw Komend AT (ELM327-Compatible) {#5-komendy-at}

OBDLink MX+ jest w pełni kompatybilny z zestawem komend ELM327 v1.4b. Każda komenda jest poprzedzona prefixem "AT".

### 5.1 Komendy Ogólne

| Komenda | Opis | Odpowiedź |
|---|---|---|
| `ATZ` | Reset urządzenia (warm reset) | `ELM327 v1.5` lub `STN2120` |
| `ATI` | Identyfikacja urządzenia | Wersja firmware |
| `AT@1` | Identyfikator producenta | `OBDLink MX+` |
| `ATE0` | Echo OFF (wyłącz echo komend) | `OK` |
| `ATE1` | Echo ON | `OK` |
| `ATL0` | Linefeeds OFF | `OK` |
| `ATL1` | Linefeeds ON | `OK` |
| `ATS0` | Spaces OFF (bez spacji w hex) | `OK` |
| `ATS1` | Spaces ON (spacje w hex) | `OK` |
| `ATH0` | Headers OFF | `OK` |
| `ATH1` | Headers ON (pokaż CAN ID w odpowiedzi) | `OK` |

### 5.2 Komendy Protokołu

| Komenda | Opis |
|---|---|
| `ATSP0` | Auto-detect protokołu |
| `ATSP6` | Wymuś CAN 11-bit 500 Kbps |
| `ATSP7` | Wymuś CAN 29-bit 500 Kbps |
| `ATDP` | Wyświetl aktualny protokół |
| `ATDPN` | Wyświetl numer aktualnego protokołu |

### 5.3 Komendy Timingowe — Kluczowe dla Performance

| Komenda | Opis | Uwagi |
|---|---|---|
| `ATAT0` | Adaptive timing OFF | ❌ Nie używaj |
| `ATAT1` | Adaptive timing ON (standard) | ⚠️ Domyślne |
| **`ATAT2`** | **Adaptive timing ON (aggressive)** | **✅ ULTRA-FAST — UŻYWAJ!** |
| `ATST FF` | Set timeout (FF = max, 01 = min) | Ręczny timeout |
| `ATST 0A` | Timeout = 10 × 4ms = 40ms | Typowy dla CAN |
| `ATST 19` | Timeout = 25 × 4ms = 100ms | Dla wolniejszych protokołów |

> [!CAUTION]
> **`ATAT2` jest kluczowy dla real-time performance!** Agresywne adaptive timing minimalizuje czas oczekiwania na odpowiedź ECU. Urządzenie dynamicznie skraca timeout po każdej udanej odpowiedzi, osiągając optymalne czasy rzędu **5-10ms per PID** na CAN.

### 5.4 Komendy Filtrowania CAN

| Komenda | Opis |
|---|---|
| `ATCF xxx` | Set CAN Filter (np. `ATCF 7E8`) |
| `ATCM xxx` | Set CAN Mask (np. `ATCM 7FF`) |
| `ATCRA xxx` | Set CAN Receive Address (np. `ATCRA 7E8`) |
| `ATAR` | Auto Receive (reset filtrów) |
| `ATCAF0` | CAN Auto Formatting OFF (raw frames) |
| `ATCAF1` | CAN Auto Formatting ON |

### 5.5 Komendy Diagnostyczne

| Komenda | Opis |
|---|---|
| `ATSH xxx` | Set Header (CAN ID do wysyłki, np. `ATSH 7E0`) |
| `ATFC SH xxx` | Set Flow Control Header |
| `ATFC SD xxxx` | Set Flow Control Data |
| `ATFC SM x` | Set Flow Control Mode (0=auto, 1=user, 2=off) |

---

## 6. Rozszerzone Komendy ST (STN-Exclusive) {#6-komendy-st}

Komendy STN są **ekskluzywne** dla chipów STN11xx/STN2120 i NIE są dostępne na klonach ELM327. To główna przewaga OBDLink nad tańszymi adapterami.

### 6.1 Komendy Identyfikacyjne

| Komenda | Opis |
|---|---|
| `STDI` | Device Identifier — zwraca model chipu |
| `STIX` | Extended Device ID — pełna informacja |
| `STFMR` | Firmware revision |
| `STSN` | Serial number urządzenia |

### 6.2 STPX — Protocol Transaction (KLUCZOWY DLA PERFORMANCE)

**STPX** jest najważniejszą komendą dla ultra-szybkiego pollingu. Pozwala na wysłanie kompletnej transakcji protokołu w jednym poleceniu, eliminując overhead wielu komend AT.

**Składnia:**
```
STPX h:<header>, d:<data> [,t:<timeout>] [,r:<responses>] [,f:<flags>]
```

| Parametr | Opis |
|---|---|
| `h:<header>` | CAN ID / Header wiadomości (np. `h:7E0`) |
| `d:<data>` | Dane payload (np. `d:010C` = Mode 01, PID 0C) |
| `t:<timeout>` | Timeout w ms (np. `t:50`) |
| `r:<responses>` | Oczekiwana liczba odpowiedzi (np. `r:1`) |
| `f:<flags>` | Flagi kontrolne |

**Przykład — Pobranie RPM:**
```
STPX h:7DF, d:010C, r:1
```

**Przykład — Szybkie pobranie z minimalnym timeoutem:**
```
STPX h:7E0, d:010C, t:30, r:1
```

> [!TIP]
> `STPX` jest 20-30% szybszy niż standardowa sekwencja `ATSH` + OBD request, ponieważ łączy ustawienie headera i wysyłkę danych w jedną atomową transakcję, eliminując round-trip przez Bluetooth.

### 6.3 Komendy CAN Monitoring STN

| Komenda | Opis |
|---|---|
| `STMA` | Start CAN Monitoring All (odpowiednik ATMA) |
| `STM` | Start CAN Monitoring (z filtrami) |
| `STMFR` | Monitor with Filtering and Response |
| `STCMM 1` | CAN Monitoring Mode 1 (formatted) |
| `STCMM 0` | CAN Monitoring Mode 0 (raw) |
| `STCSWM` | CAN Silent Wakeup Mode |

### 6.4 Komendy Filtrów STN

| Komenda | Opis |
|---|---|
| `STFAP xxxx, yyyy` | Add Pass Filter (ID + mask) |
| `STFAB xxxx, yyyy` | Add Block Filter |
| `STFAC` | Clear All Filters |
| `STFA` | Show Active Filters |

### 6.5 Inne Komendy STN

| Komenda | Opis |
|---|---|
| `STSLCS` | Sleep with Low Current State |
| `STSLEEP` | Enter sleep mode |
| `STWBR` | Wake on Bluetooth Request |
| `STBR xxxxxx` | Set Baud Rate (np. `STBR 921600`) |
| `STPTO xxx` | Set Protocol Timeout |

---

## 7. Inicjalizacja Urządzenia — Sekwencja Optymalna {#7-inicjalizacja}

### 7.1 Sekwencja Minimalna (Szybki Start)

```
ATZ          → Reset
ATE0         → Echo OFF (mniej danych przez BT)
ATL0         → Linefeeds OFF
ATS0         → Spaces OFF (mniej bajtów = szybciej)
ATH1         → Headers ON (potrzebne do identyfikacji ECU)
ATAT2        → Aggressive Adaptive Timing ⭐
ATSP6        → Wymuś CAN 11-bit 500K (jeśli znany pojazd)
```

### 7.2 Sekwencja Optymalna (Ultra-Performance)

```
ATZ              → Reset urządzenia
ATE0             → Echo OFF
ATL0             → Linefeeds OFF  
ATS0             → Spaces OFF ⭐ (redukcja ~30% danych BT)
ATH1             → Headers ON
ATAT2            → Aggressive Adaptive Timing ⭐⭐⭐
ATSP6            → CAN 11-bit 500K
ATCAF1           → CAN Auto Formatting ON
ATCRA 7E8        → Filtruj tylko odpowiedzi z ECU silnika ⭐
ATSH 7E0         → Ustaw header na fizyczny adres ECU ⭐
ATST 0A          → Timeout 40ms (wystarczający dla CAN)
```

### 7.3 Sprawdzenie Obsługiwanych PID

```
0100             → Supported PIDs [01-20] — bitmask
0120             → Supported PIDs [21-40]
0140             → Supported PIDs [41-60]
0160             → Supported PIDs [61-80]
0180             → Supported PIDs [81-A0]
01A0             → Supported PIDs [A1-C0]
```

> [!NOTE]
> Odpowiedź `0100` to bitmaska 4 bajtów (32 bity). Każdy bit odpowiada kolejnemu PID. Bit=1 oznacza wsparcie. Np. odpowiedź `BE 1F A8 13` oznacza:
> - Bit 31 (PID 01): ✅ Monitor status
> - Bit 30 (PID 02): ❌ 
> - Bit 29 (PID 03): ✅ Fuel system status
> - itd.

---

## 8. Protokoły Pojazdowe — Szczegóły Techniczne {#8-protokoły-pojazdowe}

### 8.1 CAN Bus (ISO 15765-4) — Protokół Docelowy

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
│  OBD-II na CAN:                                          │
│  ┌─────────┐ ┌──────────────────────────────────────┐    │
│  │ CAN ID  │ │  Byte 0  │ Byte 1 │ Byte 2 │ Data...│    │
│  │ 7DF/7E0 │ │  Length  │ Mode   │  PID   │        │    │
│  └─────────┘ └──────────────────────────────────────┘    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 8.2 Adresowanie CAN

| Typ | Request CAN ID | Response CAN ID | Użycie |
|---|---|---|---|
| **Functional (broadcast)** | `7DF` | `7E8`-`7EF` | Zapytanie do wszystkich ECU |
| **Physical (direct)** | `7E0` | `7E8` | Bezpośrednio do ECU silnika |
| Physical | `7E1` | `7E9` | ECU transmisji |
| Physical | `7E2` | `7EA` | ECU ABS/ESP |

> [!TIP]
> **Dla maksymalnej prędkości:** Używaj adresowania **fizycznego** (`7E0`) zamiast broadcast (`7DF`). Unikasz zbierania odpowiedzi z wielu ECU i redukujesz szum na magistrali.

### 8.3 Porównanie Szybkości Protokołów

| Protokół | Bus Speed | Typowy czas odpowiedzi per PID | PIDs/sekundę (realny) |
|---|---|---|---|
| **CAN 500K** | 500 Kbps | **5-15 ms** | **40-100+** |
| CAN 250K | 250 Kbps | 10-25 ms | 25-60 |
| KWP2000 Fast | 10.4 Kbps | 50-100 ms | 5-10 |
| ISO 9141-2 | 10.4 Kbps | 100-200 ms | 3-5 |
| J1850 VPW | 10.4 Kbps | 50-150 ms | 4-8 |
| J1850 PWM | 41.6 Kbps | 30-80 ms | 8-15 |

---

## 9. Tryby Danych OBD-II (SAE J1979) {#9-tryby-danych}

### 9.1 Standardowe Tryby (Mode/Service)

| Mode | Hex | Opis | Dla Real-Time |
|---|---|---|---|
| **01** | `01` | **Current Powertrain Data (Live Data)** | **✅ GŁÓWNY** |
| 02 | `02` | Freeze Frame Data | ❌ Historyczne |
| 03 | `03` | Emission-Related DTCs | ❌ Diagnostyka |
| 04 | `04` | Clear/Reset DTCs | ❌ Akcja |
| 05 | `05` | Oxygen Sensor Monitoring | ⚠️ Specyficzne |
| 06 | `06` | On-Board Monitoring Test Results | ⚠️ Specyficzne |
| 07 | `07` | Pending DTCs (current cycle) | ❌ Diagnostyka |
| 08 | `08` | Control On-Board System | ❌ Akcja |
| 09 | `09` | Vehicle Information (VIN itp.) | ❌ Jednorazowe |
| 0A | `0A` | Permanent DTCs | ❌ Diagnostyka |

### 9.2 Service $22 — Read Data By Identifier (Zaawansowany)

**Mode $22** pozwala na odczyt **manufacturer-specific** PIDs (2-bajtowe DID). Jest to jedyny sposób na **multi-PID w jednej ramce CAN**.

```
Request:  22 F4 0C F4 0D F4 05
Response: 62 F4 0C [data] F4 0D [data] F4 05 [data]
```

> [!WARNING]
> Service $22 PIDs są **specyficzne dla producenta**. Wymagają znajomości tabel DID dla konkretnego modelu pojazdu. Nie są częścią standardu OBD-II.

---

## 10. Kompletna Tabela PID Mode 01 {#10-tabela-pid}

### 10.1 PIDs Najważniejsze dla Real-Time (High-Priority)

| PID | Hex | Opis | Bajty | Formuła | Jednostka | Min | Max |
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

### 10.2 PIDs Dodatkowe (Medium-Priority)

| PID | Hex | Opis | Bajty | Formuła | Jednostka |
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

### 10.3 PIDs Turbodoładowania (jeśli obsługiwane)

| PID | Hex | Opis | Bajty | Formuła | Jednostka |
|---|---|---|---|---|---|
| **70** | `70` | Boost Pressure Control | 9 | Complex | kPa |
| **6C** | `6C` | Commanded Throttle Actuator | 5 | Complex | — |

### 10.4 Format Zapytania i Odpowiedzi

```
ZAPYTANIE:   01 [PID]
ODPOWIEDŹ:   41 [PID] [DATA BYTES...]

Przykład — Engine RPM (PID 0C):
TX: "010C\r"
RX: "41 0C 1A F8"

Dekodowanie: 
  A = 0x1A = 26
  B = 0xF8 = 248
  RPM = (256 × 26 + 248) / 4 = (6656 + 248) / 4 = 6904 / 4 = 1726 RPM
```

---

## 11. Optymalizacja Performance — Real-Time Polling {#11-optymalizacja-performance}

### 11.1 Strategia 1: Sekwencyjny Polling z ATAT2

Najprostsza i najbardziej niezawodna metoda:

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
│      // = 25 pełnych cykli/sekundę               │
│      // = ~100 PID reads/sekundę                 │
│  }                                               │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Spodziewana wydajność na CAN 500K + ATAT2:**
- **1 PID:** ~5-15 ms → do **100+ PID samples/sec** (dla jednego PID)
- **4 PIDs:** ~25-50 ms cykl → **20-40 cykli/sec**
- **8 PIDs:** ~50-100 ms cykl → **10-20 cykli/sec**

### 11.2 Strategia 2: STPX Batch Transactions

Używając komendy STPX, eliminujesz overhead `ATSH`:

```
// Zamiast:
ATSH 7E0        → 1 round-trip BT
010C             → 1 round-trip BT + 1 round-trip CAN
ATSH 7E0        → 1 round-trip BT (powtórzenie!)
010D             → 1 round-trip BT + 1 round-trip CAN

// Użyj:
STPX h:7E0, d:010C, r:1    → 1 round-trip (BT+CAN zintegrowane)
STPX h:7E0, d:010D, r:1    → 1 round-trip
```

**Zysk: ~20-30% mniej round-tripów BT.**

### 11.3 Strategia 3: Physical Addressing + Filtering

```
// Ustaw na stałe:
ATSH 7E0         → Tylko ECU silnika (żaden inny ECU nie odpowiada)
ATCRA 7E8        → Filtruj TYLKO odpowiedzi z ECU silnika
ATAT2            → Agresywne skracanie timeoutu

// Efekt: Adapter nie czeka na odpowiedzi z innych ECU
// Redukcja czasu oczekiwania: ~5-10ms per PID
```

### 11.4 Strategia 4: ATS0 — Spaces Off

```
// Z spacjami (ATS1):  "41 0C 1A F8\r\r>"   → 17 bajtów
// Bez spacji (ATS0):  "410C1AF8\r\r>"       → 12 bajtów

// Redukcja ~30% danych przez Bluetooth!
// Przy 100 odpowiedzi/sec = setki zaoszczędzonych bajtów/sec
```

### 11.5 Strategia 5: Prioritized PID Groups

Stwórz grupy PID o różnych priorytetach:

```
Grupa A (CRITICAL — Every Cycle):
  0C = RPM
  0D = Speed
  11 = Throttle

Grupa B (HIGH — Every 2nd Cycle):
  04 = Engine Load
  10 = MAF

Grupa C (MEDIUM — Every 5th Cycle):
  05 = Coolant Temp
  0F = Intake Air Temp
  0B = Intake Manifold Pressure

Grupa D (LOW — Every 10th Cycle):
  2F = Fuel Level
  42 = Control Module Voltage
  5C = Oil Temperature
```

```
Algorytm:
  cycle = 0
  while (connected) {
      poll(GROUP_A)                    // Zawsze
      if (cycle % 2 == 0) poll(GROUP_B)  // Co 2 cykl
      if (cycle % 5 == 0) poll(GROUP_C)  // Co 5 cykl
      if (cycle % 10 == 0) poll(GROUP_D) // Co 10 cykl
      cycle++
  }
```

**Efekt:** Krytyczne dane (RPM, Speed, Throttle) odświeżane z ~30-50 Hz, wolniejsze parametry z niższą częstotliwością bez marnowania bandwidth.

### 11.6 Strategia 6: Zero-Wait Pipeline

Zamiast czekać na pełną odpowiedź, parsuj bajt po bajcie:

```
┌────────────────────────────────────────────────┐
│         ZERO-WAIT PIPELINE                      │
├────────────────────────────────────────────────┤
│                                                │
│  TX: "010C\r"                                  │
│  ↕ (czekaj na '>' = koniec odpowiedzi)         │
│  RX: "410C1AF8\r\r>"                           │
│  ↕ (natychmiast parsuj + wyślij następne)      │
│  TX: "010D\r"                                  │
│  ↕ ...                                         │
│                                                │
│  KLUCZOWE: Nie dodawaj żadnych Thread.sleep()!  │
│  Parsuj inline, wysyłaj natychmiast po '>'     │
│                                                │
└────────────────────────────────────────────────┘
```

### 11.7 Podsumowanie — Tabela Optymalizacji

| Technika | Wpływ na Performance | Trudność |
|---|---|---|
| `ATAT2` — Aggressive Timing | ⭐⭐⭐⭐⭐ | Łatwa |
| `ATS0` — Spaces Off | ⭐⭐⭐ | Łatwa |
| `ATCRA 7E8` — Filter ECU | ⭐⭐⭐⭐ | Łatwa |
| Physical addr (`7E0`) | ⭐⭐⭐ | Łatwa |
| `STPX` batch | ⭐⭐⭐⭐ | Średnia |
| Prioritized PID groups | ⭐⭐⭐⭐⭐ | Średnia |
| Zero-wait pipeline | ⭐⭐⭐⭐ | Średnia |
| `flush()` after write | ⭐⭐ | Łatwa |
| Dedicated IO thread | ⭐⭐⭐⭐⭐ | Średnia |

---

## 12. CAN Bus Monitoring — Tryb Pasywny {#12-can-monitoring}

### 12.1 Czym jest CAN Monitoring?

CAN Monitoring to tryb **pasywnego nasłuchu** na magistrali CAN. Adapter nie wysyła żadnych zapytań — zamiast tego przechwytuje **WSZYSTKIE** ramki CAN, które pojawiają się na magistrali.

Jest to metoda **NAJSZYBSZEGO** przechwytywania danych, ponieważ:
- ❌ Nie ma round-trip (request → response)
- ✅ Dane napływają z częstotliwością, z jaką ECU je generują (10-100 Hz per parametr)
- ✅ Wiele parametrów jednocześnie

### 12.2 Uruchomienie CAN Monitoring

```
// Konfiguracja
ATSP6          → CAN 11-bit 500K
ATCAF0         → Raw CAN frames (no formatting)
ATH1           → Pokaż CAN ID

// Filtrowanie (opcjonalne, ale zalecane!)
STFAC          → Clear all filters
STFAP 7E8, 7FF → Pass only ECU engine responses

// Start monitorowania
ATMA           → Monitor All CAN traffic
// lub
STM            → STN Monitor (z filtrami)
```

### 12.3 Format Danych w Monitoring Mode

```
// Raw CAN frames (ciągły strumień):
7E8 06 41 0C 1A F8 00 00    ← RPM
7E8 04 41 0D 3C 00 00 00    ← Speed (60 km/h)
7E9 05 41 05 6E 00 00 00    ← Coolant Temp z transmisji ECU
7E8 04 41 11 2F 00 00 00    ← Throttle
7EA 06 41 04 B2 00 00 00    ← Load z ABS ECU
...
```

### 12.4 Zatrzymanie Monitoring

Wyślij **dowolny znak** (np. `\r`) aby przerwać tryb monitorowania. Adapter odpowie promptem `>`.

> [!WARNING]
> **CAN Monitoring generuje OGROMNE ilości danych!** Na nowoczesnym pojeździe magistrala CAN może generować 2000-5000 ramek/sekundę. Bez filtrów, Bluetooth 3.0 może nie nadążyć z transmisją. **ZAWSZE** stosuj filtry CAN (`STFAP`, `ATCF`, `ATCM`).

### 12.5 Monitoring vs Polling — Porównanie

| Cecha | Polling (Mode 01) | CAN Monitoring |
|---|---|---|
| **Kontrola nad danymi** | ✅ Pełna (wybierasz PIDs) | ⚠️ Co ECU nadaje |
| **Częstotliwość** | ~20-100 PID/sec | Do 5000 frames/sec |
| **Latencja** | 5-15 ms per PID | ~0 ms (real-time!) |
| **Overhead BT** | Wysoki (TX+RX) | Niski (tylko RX) |
| **Parsowanie** | Proste (znane PIDs) | Złożone (raw CAN IDs) |
| **Wymagana wiedza** | Standardowa OBD-II | Specyficzna dla pojazdu |
| **Przydatność RT** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## 13. ISO-TP — Multi-Frame Communication {#13-iso-tp}

### 13.1 Kiedy Potrzebna Multi-Frame?

Standardowa ramka CAN ma **8 bajtów** danych. Po odjęciu nagłówka ISO-TP (1 bajt), dostępnych jest **7 bajtów** na dane OBD. Jeśli odpowiedź przekracza 7 bajtów (np. VIN, DTC lista, Mode 06), potrzebna jest **segmentacja ISO-TP (ISO 15765-2)**.

### 13.2 Typy Ramek ISO-TP

| Typ | Byte 0 | Opis |
|---|---|---|
| **Single Frame (SF)** | `0x0n` | Kompletna wiadomość (n = długość, 1-7 bajtów) |
| **First Frame (FF)** | `0x1n nn` | Pierwsza ramka multi-frame (nn nn = total length) |
| **Consecutive Frame (CF)** | `0x2n` | Kolejna ramka (n = numer sekwencji 0-F) |
| **Flow Control (FC)** | `0x30` | Kontrola przepływu (BS + STmin) |

### 13.3 Parametry Flow Control

| Parametr | Opis | Zakres |
|---|---|---|
| **FS** | Flow Status (0=CTS, 1=Wait, 2=Overflow) | 0-2 |
| **BS** | Block Size (0 = bez limitu) | 0-255 |
| **STmin** | Separation Time Minimum | 0-127 ms / 100-900 µs |

> [!NOTE]
> OBDLink MX+ obsługuje **automatycznie** ISO-TP flow control. Nie musisz ręcznie zarządzać FC framami, chyba że używasz `ATFC SM 1` (user-defined flow control).

---

## 14. Architektura Aplikacji Android {#14-architektura-android}

### 14.1 Model Wątkowy

```
┌──────────────────────────────────────────────────────┐
│                ANDROID APP ARCHITECTURE               │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌─────────────────┐                                 │
│  │   UI THREAD      │ ← Wyświetlanie danych          │
│  │   (Main Thread)  │ ← Animacje gauges              │
│  └────────┬─────────┘                                │
│           │ LiveData / StateFlow                     │
│           │                                          │
│  ┌────────▼─────────┐                                │
│  │   DATA LAYER     │ ← Parsowanie, formulas         │
│  │   (Repository)   │ ← Buforowanie, interpolacja    │
│  └────────┬─────────┘                                │
│           │                                          │
│  ┌────────▼─────────┐                                │
│  │   IO THREAD      │ ← Dedicated polling loop       │
│  │   (BT Worker)    │ ← OutputStream.write + flush   │
│  │                  │ ← InputStream.read blocking    │
│  │   WYDZIELONY!    │ ← NIGDY nie blokuj UI Thread!  │
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

### 14.2 Kluczowe Zasady Implementacji

1. **Dedykowany wątek I/O** — CAŁA komunikacja BT na osobnym wątku/koroutynie
2. **Nigdy nie blokuj UI** — Użyj `LiveData`, `StateFlow`, lub `Handler` do aktualizacji UI
3. **`flush()` po każdym write** — Wymusza natychmiastowe wysłanie
4. **Parsuj inline** — Nie buforuj całych odpowiedzi, parsuj bajt po bajcie
5. **Ring buffer** — Dla danych real-time, używaj ring buffer zamiast list
6. **Atomic state** — Stan połączenia jako `AtomicBoolean` / `StateFlow<ConnectionState>`

### 14.3 Pseudokod — Real-Time Polling Engine

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
    
    // Priorytetowe grupy PID
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
        // Wyślij
        output.write("$command\r".toByteArray(Charsets.US_ASCII))
        output.flush()  // ⭐ KRYTYCZNE!
        
        // Odbierz do promptu '>'
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

Dla ciągłego działania w tle:

```kotlin
class OBDForegroundService : Service() {
    // Użyj Foreground Service z notyfikacją
    // aby Android nie zabił procesu podczas jazdy
    // START_STICKY zapewnia restart po OOM kill
}
```

---

## 15. Stabilność Połączenia i Reconnect {#15-stabilność}

### 15.1 Typowe Przyczyny Zerwania Połączenia

| Przyczyna | Rozwiązanie |
|---|---|
| Wyjście z zasięgu BT | Auto-reconnect z backoff |
| Android zabił proces | Foreground Service + START_STICKY |
| ECU sleep / ignition off | Sprawdzanie `ATIGN` (stan zapłonu) |
| Bluetooth stack crash | `socket.close()` + nowy socket |
| Buffer overflow adaptera | Zmniejsz częstotliwość pollingu |
| Timeout odpowiedzi ECU | Retry z `ATPC` (protocol close + reopen) |

### 15.2 Strategia Reconnect

```
┌────────────────────────────────────┐
│     RECONNECT STRATEGY              │
├────────────────────────────────────┤
│                                    │
│  1. Wykryj zerwanie (IOException)  │
│  2. socket.close()                 │
│  3. Czekaj 500ms                   │
│  4. Nowy socket (createRfcomm...)  │
│  5. socket.connect()              │
│  6. Re-inicjalizacja (ATZ...)      │
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
// Watchdog: jeśli brak odpowiedzi przez 5 sekund, wymuś reconnect
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

## 16. Zarządzanie Energią — BatterySaver™ {#16-zarządzanie-energią}

### 16.1 Automatyczny Sleep

OBDLink MX+ posiada technologię **BatterySaver™**, która automatycznie przełącza adapter w tryb niskiego poboru prądu (~2 mA) po określonym czasie nieaktywności.

### 16.2 Komendy Power Management

| Komenda | Opis |
|---|---|
| `ATLP` | Enter Low Power mode |
| `STSLEEP` | Enter deep sleep (STN exclusive) |
| `STSLCS` | Sleep with Low Current State |
| `STWBR` | Wake on Bluetooth Request |

### 16.3 Kontrola Zapłonu

```
ATIGN          → Sprawdź stan zapłonu ("ON" / "OFF")

// Strategia:
// Jeśli ATIGN = OFF → Przejdź w sleep mode
// Jeśli ATIGN = ON → Kontynuuj polling
```

---

## 17. Limity Techniczne i Wąskie Gardła {#17-limity}

### 17.1 Wąskie Gardła — Pipeline

```
┌─────────────────────────────────────────────────────────┐
│              BOTTLENECK ANALYSIS                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. ECU Processing Time         ~2-5 ms per PID         │
│     (czas odpowiedzi ECU na CAN)                        │
│                                                         │
│  2. CAN Bus Transmission        ~0.2 ms per frame       │
│     (fizyczna transmisja CAN 500K)                      │
│                                                         │
│  3. STN2120 Processing          ~1-3 ms                 │
│     (dekodowanie CAN → ASCII)                           │
│                                                         │
│  4. Bluetooth Transmission      ~5-15 ms round-trip     │
│     (NAJWIĘKSZE WĄSKIE GARDŁO!)                         │
│                                                         │
│  5. Android Processing          ~0.5-2 ms               │
│     (parsowanie, UI update)                             │
│                                                         │
│  TOTAL per PID:                 ~8-25 ms                │
│  Realny throughput:             40-120 PIDs/sec         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 17.2 Limity Absolutne

| Parametr | Limit |
|---|---|
| **Max PIDs/sec (1 PID ciągłe)** | ~100-130 |
| **Max PIDs/sec (mixed polling)** | ~40-80 |
| **Max CAN frames/sec (monitoring)** | ~2000-3000 (z BT bottleneck) |
| **Min latencja per PID** | ~5-8 ms (CAN + STN + BT) |
| **Bluetooth MTU** | 1023 bytes (RFCOMM) |
| **Concurrent ECU connections** | Brak limitu (ale sekwencyjne) |

### 17.3 Ograniczenia OBD-II vs Producenta

| Feature | Standardowy OBD-II (Mode 01) | Manufacturer-Specific (Mode 22) |
|---|---|---|
| **Dostępne PIDs** | ~100 standardowych | 1000+ per model |
| **Multi-PID request** | ❌ Nie | ✅ Tak |
| **Update rate ECU** | 10-50 Hz | Do 100+ Hz |
| **Turbo/Boost data** | ⚠️ Ograniczone | ✅ Pełne |
| **Transmission data** | ⚠️ Podstawowe | ✅ Pełne |
| **ABS/ESP danych** | ❌ Nie (inny ECU) | ✅ Tak |

---

## 18. Podsumowanie — Strategia Ultra-Fast RT {#18-podsumowanie}

### 18.1 Optymalna Konfiguracja

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
│    ✅ Dedicated IO thread (nie Coroutine Main!)           │
│    ✅ Foreground Service (ciągłe działanie)              │
│    ✅ Ring buffer (nie ArrayList)                        │
│    ✅ StateFlow/LiveData → UI                            │
│    ✅ Auto-reconnect z exponential backoff               │
│                                                          │
│  SPODZIEWANA WYDAJNOŚĆ:                                  │
│    📊 3-4 PIDs krytycznych @ 30-50 Hz                    │
│    📊 2-3 PIDs wysokich @ 15-25 Hz                       │
│    📊 3 PIDs średnich @ 6-10 Hz                          │
│    📊 3 PIDs niskich @ 3-5 Hz                            │
│    📊 TOTAL: ~80-120 PID samples/sec                     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 18.2 Tryby Działania Aplikacji

| Tryb | Opis | PIDs | Refresh Rate |
|---|---|---|---|
| **🏎️ Race Mode** | RPM + Speed + Throttle | 3 | 40-60 Hz |
| **📊 Dashboard Mode** | 8-10 podstawowych PIDs | 8-10 | 10-20 Hz |
| **🔧 Diagnostic Mode** | Wszystkie dostępne + DTCs | 15-20+ | 3-5 Hz |
| **📡 Sniff Mode** | CAN Monitoring pasywny | ∞ | Bus rate |

### 18.3 Kluczowe Źródła Dokumentacji

| Źródło | Opis |
|---|---|
| **SAE J1979** | Standard OBD-II service modes i PIDs |
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
> **Ta dokumentacja stanowi kompletną bazę techniczną** do budowy aplikacji Android opartej o OBDLink MX+ w trybie real-time. Najważniejsze odkrycia:
> 1. **CAN 500K + ATAT2 + Physical Addressing** = fundament prędkości
> 2. **STPX** = unikalna przewaga STN2120 nad ELM327
> 3. **Bluetooth Classic SPP** jest optymalny (nie BLE)
> 4. **Priorytetyzacja PID** jest kluczowa dla UX
> 5. **CAN Monitoring** oferuje najniższą latencję, ale wymaga wiedzy o CAN IDs specyficznych dla pojazdu
> 6. **Wąskim gardłem jest Bluetooth**, nie CAN bus ani ECU

# Android Integration — Bluetooth SPP & App Architecture

> **Target:** Android 8.0+ (API 26+)  
> **Connection:** Bluetooth Classic 3.0, SPP/RFCOMM  
> **Language:** Kotlin  
> **UI:** Jetpack Compose  

---

## 1. Required Permissions

### AndroidManifest.xml

```xml
<!-- Bluetooth — API ≤ 30 -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<!-- Bluetooth — API ≥ 31 (Android 12+) -->
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />

<!-- Foreground Service (for continuous background operation) -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />

<!-- Wake Lock (prevent CPU sleep during polling) -->
<uses-permission android:name="android.permission.WAKE_LOCK" />
```

### Runtime Permission Flow (Android 12+)

```kotlin
val requiredPermissions = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    arrayOf(
        Manifest.permission.BLUETOOTH_CONNECT,
        Manifest.permission.BLUETOOTH_SCAN
    )
} else {
    arrayOf(
        Manifest.permission.BLUETOOTH,
        Manifest.permission.BLUETOOTH_ADMIN,
        Manifest.permission.ACCESS_FINE_LOCATION
    )
}
```

---

## 2. Device Discovery & Pairing

### Finding OBDLink MX+

```kotlin
object OBDLinkDiscovery {
    
    const val DEVICE_NAME = "OBDLink MX+"
    const val DEVICE_NAME_ALT = "OBDLink"
    val SPP_UUID: UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB")
    
    /**
     * Find OBDLink MX+ among bonded (paired) devices.
     * The device must be paired via Android Settings first.
     */
    fun findBondedDevice(adapter: BluetoothAdapter): BluetoothDevice? {
        return adapter.bondedDevices?.firstOrNull { device ->
            device.name?.contains("OBDLink", ignoreCase = true) == true
        }
    }
}
```

### Pairing Notes

| Property | Value |
|----------|-------|
| Default PIN | No PIN required (button-based pairing) |
| Fallback PIN | `1234` |
| Pairing trigger | Physical "Connect" button on device |
| Security | 128-bit encryption after pairing |

> [!IMPORTANT]
> OBDLink MX+ requires pressing the physical **Connect button** on the device to enable Bluetooth pairing. This is a security feature — the device does NOT broadcast continuously like cheap ELM327 clones.

---

## 3. Bluetooth Socket Connection

### Primary Connection Method

```kotlin
class OBDConnection(private val device: BluetoothDevice) {
    
    private var socket: BluetoothSocket? = null
    private var input: InputStream? = null
    private var output: OutputStream? = null
    
    /**
     * Connect to OBDLink MX+ via Bluetooth SPP.
     * MUST be called from a background thread.
     */
    suspend fun connect(): Result<Unit> = withContext(Dispatchers.IO) {
        try {
            // Cancel any ongoing discovery (reduces interference)
            BluetoothAdapter.getDefaultAdapter()?.cancelDiscovery()
            
            // Create RFCOMM socket
            socket = device.createRfcommSocketToServiceRecord(
                OBDLinkDiscovery.SPP_UUID
            )
            
            // Connect (blocking call)
            socket!!.connect()
            
            // Get streams
            input = socket!!.inputStream
            output = socket!!.outputStream
            
            Result.success(Unit)
        } catch (e: IOException) {
            // Fallback: insecure RFCOMM
            tryInsecureFallback()
        }
    }
    
    private fun tryInsecureFallback(): Result<Unit> {
        return try {
            socket = device.createInsecureRfcommSocketToServiceRecord(
                OBDLinkDiscovery.SPP_UUID
            )
            socket!!.connect()
            input = socket!!.inputStream
            output = socket!!.outputStream
            Result.success(Unit)
        } catch (e: IOException) {
            // Last resort: reflection hack for older devices
            tryReflectionFallback()
        }
    }
    
    private fun tryReflectionFallback(): Result<Unit> {
        return try {
            val method = device.javaClass.getMethod(
                "createRfcommSocket",
                Int::class.javaPrimitiveType
            )
            socket = method.invoke(device, 1) as BluetoothSocket
            socket!!.connect()
            input = socket!!.inputStream
            output = socket!!.outputStream
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    fun disconnect() {
        try {
            input?.close()
            output?.close()
            socket?.close()
        } catch (_: IOException) {}
    }
}
```

---

## 4. Command Protocol Layer

### Sending Commands & Reading Responses

```kotlin
class OBDCommandLayer(
    private val input: InputStream,
    private val output: OutputStream
) {
    companion object {
        const val PROMPT = '>'
        const val CR = '\r'
        const val TIMEOUT_MS = 2000L
    }
    
    /**
     * Send a command and wait for the response.
     * Response is complete when '>' prompt is received.
     */
    fun sendCommand(command: String): String {
        // Write command + CR
        val bytes = "$command$CR".toByteArray(Charsets.US_ASCII)
        output.write(bytes)
        output.flush()  // ⭐ CRITICAL — force immediate transmission
        
        // Read until '>' prompt
        val response = StringBuilder()
        val deadline = System.currentTimeMillis() + TIMEOUT_MS
        
        while (System.currentTimeMillis() < deadline) {
            if (input.available() > 0) {
                val byte = input.read()
                if (byte == -1) throw IOException("Stream closed")
                
                val char = byte.toChar()
                if (char == PROMPT) break
                response.append(char)
            }
        }
        
        return response.toString()
            .replace("\r", "")
            .replace("\n", "")
            .trim()
    }
    
    /**
     * Check if response indicates an error.
     */
    fun isError(response: String): Boolean {
        return response.contains("NO DATA") ||
               response.contains("UNABLE TO CONNECT") ||
               response.contains("BUS INIT") ||
               response.contains("ERROR") ||
               response.contains("?")
    }
}
```

---

## 5. Initialization Sequence

```kotlin
class OBDInitializer(private val cmd: OBDCommandLayer) {
    
    data class InitResult(
        val success: Boolean,
        val deviceId: String,
        val protocol: String,
        val supportedPIDs: Set<Int>,
        val voltage: String
    )
    
    /**
     * Initialize OBDLink MX+ with optimal performance settings.
     * Returns supported PIDs and device info.
     */
    fun initialize(): InitResult {
        // 1. Reset device
        val deviceId = cmd.sendCommand("ATZ")
        Thread.sleep(500) // Allow reset to complete
        
        // 2. Core settings
        cmd.sendCommand("ATE0")    // Echo OFF
        cmd.sendCommand("ATL0")    // Linefeeds OFF
        cmd.sendCommand("ATS0")    // Spaces OFF ⭐
        cmd.sendCommand("ATH1")    // Headers ON
        
        // 3. Performance settings
        cmd.sendCommand("ATAT2")   // Aggressive Adaptive Timing ⭐⭐⭐
        cmd.sendCommand("ATSP6")   // CAN 11-bit 500K
        
        // 4. Verify protocol
        val protocol = cmd.sendCommand("ATDP")
        
        // 5. Read voltage
        val voltage = cmd.sendCommand("ATRV")
        
        // 6. Set up CAN filtering
        cmd.sendCommand("ATSH 7E0")    // Target engine ECU
        cmd.sendCommand("ATCRA 7E8")   // Filter engine responses only
        
        // 7. Discover supported PIDs
        val supportedPIDs = discoverPIDs()
        
        return InitResult(
            success = true,
            deviceId = deviceId,
            protocol = protocol,
            supportedPIDs = supportedPIDs,
            voltage = voltage
        )
    }
    
    private fun discoverPIDs(): Set<Int> {
        val pids = mutableSetOf<Int>()
        val ranges = listOf(0x00, 0x20, 0x40, 0x60, 0x80, 0xA0)
        
        for (base in ranges) {
            val response = cmd.sendCommand("01${base.toString(16).padStart(2, '0')}")
            if (cmd.isError(response)) break
            
            // Parse 4-byte bitmask
            val dataBytes = extractDataBytes(response)
            if (dataBytes.size >= 4) {
                for (i in 0 until 32) {
                    val byteIndex = i / 8
                    val bitIndex = 7 - (i % 8)
                    if (dataBytes[byteIndex].toInt() and (1 shl bitIndex) != 0) {
                        pids.add(base + i + 1)
                    }
                }
            }
        }
        
        return pids
    }
    
    private fun extractDataBytes(response: String): ByteArray {
        // Remove header (first 4 chars for CAN ID + length + mode + PID)
        // With ATS0 + ATH1: "7E80641000XXXXXXXX"
        val hex = response.replace(" ", "")
        // Skip CAN ID (3 chars) + length (2) + mode (2) + PID (2) = 9 chars
        val dataHex = if (hex.length > 9) hex.substring(9) else return byteArrayOf()
        
        return dataHex.chunked(2).map { it.toInt(16).toByte() }.toByteArray()
    }
}
```

---

## 6. Real-Time Polling Engine

```kotlin
class RealtimePollingEngine(
    private val cmd: OBDCommandLayer,
    private val supportedPIDs: Set<Int>
) {
    // State
    private val _vehicleData = MutableStateFlow(VehicleData())
    val vehicleData: StateFlow<VehicleData> = _vehicleData.asStateFlow()
    
    @Volatile
    var isRunning = false
        private set
    
    private var pollingThread: Thread? = null
    private var cycleCount = 0L
    
    // PID Groups (only include supported PIDs)
    private val groupA = listOf(0x0C, 0x0D, 0x11).filter { it in supportedPIDs }
    private val groupB = listOf(0x04, 0x10).filter { it in supportedPIDs }
    private val groupC = listOf(0x05, 0x0F, 0x0B).filter { it in supportedPIDs }
    private val groupD = listOf(0x2F, 0x42, 0x5C, 0x1F).filter { it in supportedPIDs }
    
    fun start() {
        isRunning = true
        pollingThread = Thread({
            pollingLoop()
        }, "OBD-RT-Poll").apply {
            priority = Thread.MAX_PRIORITY
            start()
        }
    }
    
    fun stop() {
        isRunning = false
        pollingThread?.join(2000)
    }
    
    private fun pollingLoop() {
        while (isRunning) {
            try {
                // Group A — EVERY cycle
                pollGroup(groupA)
                
                // Group B — every 2nd cycle
                if (cycleCount % 2 == 0L) pollGroup(groupB)
                
                // Group C — every 5th cycle
                if (cycleCount % 5 == 0L) pollGroup(groupC)
                
                // Group D — every 10th cycle
                if (cycleCount % 10 == 0L) pollGroup(groupD)
                
                cycleCount++
                
            } catch (e: IOException) {
                isRunning = false
                _vehicleData.update { it.copy(connectionLost = true) }
            }
        }
    }
    
    private fun pollGroup(pids: List<Int>) {
        for (pid in pids) {
            if (!isRunning) return
            
            val pidHex = pid.toString(16).padStart(2, '0').uppercase()
            val response = cmd.sendCommand("01$pidHex")
            
            if (!cmd.isError(response)) {
                val value = PIDDecoder.decode(pid, response)
                _vehicleData.update { data ->
                    data.withPID(pid, value, System.currentTimeMillis())
                }
            }
        }
    }
}
```

---

## 7. PID Decoder

```kotlin
object PIDDecoder {
    
    fun decode(pid: Int, rawResponse: String): Double {
        val bytes = extractPayloadBytes(rawResponse)
        if (bytes.isEmpty()) return Double.NaN
        
        val a = bytes.getOrElse(0) { 0 }.toInt() and 0xFF
        val b = bytes.getOrElse(1) { 0 }.toInt() and 0xFF
        
        return when (pid) {
            0x04 -> a * 100.0 / 255.0                    // Engine Load (%)
            0x05 -> a - 40.0                              // Coolant Temp (°C)
            0x06, 0x07, 0x08, 0x09 ->
                   (a - 128.0) * 100.0 / 128.0            // Fuel Trim (%)
            0x0A -> a * 3.0                               // Fuel Pressure (kPa)
            0x0B -> a.toDouble()                          // Manifold Pressure (kPa)
            0x0C -> (256.0 * a + b) / 4.0                 // RPM
            0x0D -> a.toDouble()                          // Speed (km/h)
            0x0E -> a / 2.0 - 64.0                        // Timing Advance (°)
            0x0F -> a - 40.0                              // Intake Air Temp (°C)
            0x10 -> (256.0 * a + b) / 100.0               // MAF (g/s)
            0x11 -> a * 100.0 / 255.0                     // Throttle (%)
            0x1F -> 256.0 * a + b                         // Runtime (sec)
            0x21 -> 256.0 * a + b                         // Distance MIL (km)
            0x2F -> a * 100.0 / 255.0                     // Fuel Level (%)
            0x31 -> 256.0 * a + b                         // Distance since clear (km)
            0x33 -> a.toDouble()                          // Barometric (kPa)
            0x42 -> (256.0 * a + b) / 1000.0              // Module Voltage (V)
            0x43 -> (256.0 * a + b) * 100.0 / 255.0      // Absolute Load (%)
            0x45 -> a * 100.0 / 255.0                     // Rel Throttle (%)
            0x46 -> a - 40.0                              // Ambient Temp (°C)
            0x5C -> a - 40.0                              // Oil Temp (°C)
            0x5E -> (256.0 * a + b) / 20.0                // Fuel Rate (L/h)
            else -> a.toDouble()
        }
    }
    
    private fun extractPayloadBytes(response: String): ByteArray {
        val hex = response.replace(" ", "").replace("\r", "").replace("\n", "")
        // With ATH1 + ATS0: "7E80441XX..."  
        // CAN ID (3) + PCI length (2) + Mode (2) + PID (2) = 9 hex chars to skip
        val payload = if (hex.length > 9) hex.substring(9) else return byteArrayOf()
        return payload.chunked(2)
            .filter { it.length == 2 }
            .map { it.toInt(16).toByte() }
            .toByteArray()
    }
}
```

---

## 8. Foreground Service

```kotlin
class OBDForegroundService : Service() {
    
    companion object {
        const val CHANNEL_ID = "obd_rt_channel"
        const val NOTIFICATION_ID = 1
    }
    
    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
        
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("car-RT")
            .setContentText("Real-time vehicle telemetry active")
            .setSmallIcon(R.drawable.ic_gauge)
            .setOngoing(true)
            .build()
        
        startForeground(NOTIFICATION_ID, notification)
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        return START_STICKY  // Restart if killed
    }
    
    override fun onBind(intent: Intent?): IBinder? = null
    
    private fun createNotificationChannel() {
        val channel = NotificationChannel(
            CHANNEL_ID,
            "OBD Real-Time",
            NotificationManager.IMPORTANCE_LOW
        )
        getSystemService(NotificationManager::class.java)
            .createNotificationChannel(channel)
    }
}
```

---

## 9. Connection Stability

### Reconnect Strategy

```kotlin
class ConnectionManager {
    
    private val backoffMs = listOf(500L, 1000L, 2000L, 5000L, 10_000L)
    
    suspend fun connectWithRetry(device: BluetoothDevice): OBDConnection {
        var attempt = 0
        
        while (true) {
            val connection = OBDConnection(device)
            val result = connection.connect()
            
            if (result.isSuccess) {
                return connection
            }
            
            val delay = backoffMs.getOrElse(attempt) { backoffMs.last() }
            delay(delay)
            attempt++
        }
    }
}
```

### Watchdog Timer

```kotlin
class ConnectionWatchdog(
    private val onTimeout: () -> Unit,
    private val timeoutMs: Long = 5000L
) {
    @Volatile
    var lastResponseTime = System.currentTimeMillis()
    
    private val timer = Timer("OBD-Watchdog", true)
    
    fun start() {
        timer.schedule(object : TimerTask() {
            override fun run() {
                if (System.currentTimeMillis() - lastResponseTime > timeoutMs) {
                    onTimeout()
                }
            }
        }, 1000, 1000)
    }
    
    fun heartbeat() {
        lastResponseTime = System.currentTimeMillis()
    }
    
    fun stop() = timer.cancel()
}
```

---

## 10. Vehicle Data Model

```kotlin
data class VehicleData(
    val rpm: Double = 0.0,
    val speed: Double = 0.0,
    val throttle: Double = 0.0,
    val engineLoad: Double = 0.0,
    val coolantTemp: Double = 0.0,
    val intakeTemp: Double = 0.0,
    val maf: Double = 0.0,
    val fuelLevel: Double = 0.0,
    val voltage: Double = 0.0,
    val oilTemp: Double = 0.0,
    val timingAdvance: Double = 0.0,
    val manifoldPressure: Double = 0.0,
    val fuelRate: Double = 0.0,
    val runtime: Double = 0.0,
    val connectionLost: Boolean = false,
    val lastUpdateMs: Long = 0L
) {
    fun withPID(pid: Int, value: Double, timestamp: Long): VehicleData {
        return when (pid) {
            0x0C -> copy(rpm = value, lastUpdateMs = timestamp)
            0x0D -> copy(speed = value, lastUpdateMs = timestamp)
            0x11 -> copy(throttle = value, lastUpdateMs = timestamp)
            0x04 -> copy(engineLoad = value, lastUpdateMs = timestamp)
            0x05 -> copy(coolantTemp = value, lastUpdateMs = timestamp)
            0x0F -> copy(intakeTemp = value, lastUpdateMs = timestamp)
            0x10 -> copy(maf = value, lastUpdateMs = timestamp)
            0x2F -> copy(fuelLevel = value, lastUpdateMs = timestamp)
            0x42 -> copy(voltage = value, lastUpdateMs = timestamp)
            0x5C -> copy(oilTemp = value, lastUpdateMs = timestamp)
            0x0E -> copy(timingAdvance = value, lastUpdateMs = timestamp)
            0x0B -> copy(manifoldPressure = value, lastUpdateMs = timestamp)
            0x5E -> copy(fuelRate = value, lastUpdateMs = timestamp)
            0x1F -> copy(runtime = value, lastUpdateMs = timestamp)
            else -> copy(lastUpdateMs = timestamp)
        }
    }
}
```

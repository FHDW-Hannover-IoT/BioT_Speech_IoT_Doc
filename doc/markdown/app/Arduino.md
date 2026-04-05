# How the Arduino Firmware Works

> This document explains how the ESP8266 NodeMCU firmware works in the BioT Speech IoT project. No prior hardware or embedded programming knowledge required.

---

## Table of Contents

1. [What is the ESP8266 and what does it do?](#1-what-is-the-esp8266-and-what-does-it-do)
2. [What is Arduino IDE?](#2-what-is-arduino-ide)
3. [Pin Layout and Sensor Connections](#3-pin-layout-and-sensor-connections)
4. [How the Code is Structured](#4-how-the-code-is-structured)
5. [Connecting to Wi-Fi and MQTT](#5-connecting-to-wi-fi-and-mqtt)
6. [Reading the Sensors](#6-reading-the-sensors)
7. [The Three Transmission Modes](#7-the-three-transmission-modes)
8. [Timing: How the Loop Works](#8-timing--how-the-loop-works)
9. [What Happens on Startup](#9-what-happens-on-startup)
10. [Adding New Sensors When They Arrive](#10-adding-new-sensors-when-they-arrive)

---

## 1. What is the ESP8266 and what does it do?

The **ESP8266** is a tiny microcontroller, essentially a miniature computer on a chip. The development board used in this project is the **NodeMCU ESP-12E**, which adds a USB port and pin headers to make the ESP8266 easy to use.

It has two main abilities that make it perfect for this project:

- It can run code, like reading a sensor value every 20 milliseconds
- It has built-in Wi-Fi, so it can send those readings over your network

Think of it as a small, always-on device that reads sensors and reports back over Wi-Fi. It has no screen, no keyboard. The only way to interact with it is through the Serial Monitor in Arduino IDE (for debugging) and through MQTT messages sent over Wi-Fi.

The ESP8266 in this project does three things:
1. Reads sensor values (microphone, accelerometer, gyroscope, hall effect)
2. Connects to the Mosquitto broker over Wi-Fi
3. Publishes those values to MQTT topics so the Android app can receive them

---

## 2. What is Arduino IDE?

**Arduino IDE** is the application you use to write code for the ESP8266 and upload it. The language is C++ with a simplified structure.

Every Arduino sketch (the word used for an Arduino program) has exactly two required functions:

```cpp
void setup() {
    // runs once when the board powers on
}

void loop() {
    // runs over and over forever, as fast as possible
}
```

That's the entire structure. `setup()` initialises everything; connects to Wi-Fi, sets up MQTT, configures pins. Then `loop()` runs in an infinite cycle doing the actual work; reading sensors, publishing data, checking for incoming commands.

The ESP8266 runs `loop()` thousands of times per second. In this project it is deliberately slowed down using timers so sensors are only sampled at the right intervals.

---

## 3. Pin Layout and Sensor Connections

The NodeMCU has pins on both sides. Each pin has a name printed on the board. Here is a reference table for all connections used in this project:

### KY-037 (Microphone)

| KY-037 pin | NodeMCU pin | Wire colour | Purpose |
|------------|-------------|------------|---------|
| `A` | `A0` | Yellow | Analog sound level (0–1023) |
| `G` | `GND` | Black | Ground |
| `+` | `3V3` | Red | 3.3V power |
| `DO` | Not connected | — | Digital threshold (unused for now) |

### MPU-6050 (Accelerometer + Gyroscope)

| MPU-6050 pin | NodeMCU pin | Purpose |
|--------------|-------------|---------|
| `VCC` | `3V3` | 3.3V power |
| `GND` | `GND` | Ground |
| `SCL` | `D1` (GPIO5) | I2C clock signal |
| `SDA` | `D2` (GPIO4) | I2C data signal |

> **What is I2C?** I2C is a communication protocol that lets two devices talk using just two wires, one for the clock (timing), one for the data. The MPU-6050 uses I2C to send accelerometer and gyroscope readings to the ESP8266. You do not need to understand the details, the library handles it.

### A3144 (Hall Effect / Magnet sensor)

| A3144 pin | NodeMCU pin | Purpose |
|-----------|-------------|---------|
| `VCC` | `3V3` | 3.3V power |
| `GND` | `GND` | Ground |
| `OUT` | `D5` (GPIO14) | Digital output, HIGH or LOW depending on magnet |

> **What is a Hall effect sensor?** It detects magnetic fields. When a magnet is close, the output pin goes LOW (0). When there is no magnet, it stays HIGH (1). This is used to detect if something magnetic is present or moving past the sensor.

### Important notes on power

- Always use `3V3`, NOT `5V` or `VIN`. The ESP8266 operates at 3.3V and so do all three sensors. Using 5V would damage them.
- `GND` is shared, all sensors connect their ground to the same `GND` pin on the NodeMCU.
- The NodeMCU only has **one analog input** (`A0`). This is why only the KY-037 (microphone) uses analog, the other two sensors use digital communication (I2C for MPU-6050, digital pin for A3144).

---

## 4. How the Code is Structured

The firmware is one `.ino` file split into logical sections:

```
Configuration (constants at the top)
    ↓
Mode state (STREAM / BURST / AVERAGE)
    ↓
Buffers and accumulators (storage for burst and average modes)
    ↓
WiFiClient + PubSubClient (MQTT library objects)
    ↓
setMode()     — switches transmission mode, resets all buffers
onMessage()   — handles incoming MQTT commands from the app
connectWiFi() — connects to the router
connectMQTT() — connects to Mosquitto broker
    ↓
readMic()     — reads KY-037
readAccel()   — reads MPU-6050 accelerometer (stubbed)
readGyro()    — reads MPU-6050 gyroscope (stubbed)
readHall()    — reads A3144 (stubbed)
    ↓
publishMic()    — formats and sends mic data
publishAccel()  — formats and sends accel data
publishGyro()   — formats and sends gyro data
publishHall()   — formats and sends hall data
    ↓
sendBurst()   — sends collected burst data
sendAverage() — sends computed averages
    ↓
setup()       — runs once on boot
loop()        — runs forever
```

Each section has one clear job. The `read` functions only read. The `publish` functions only send. The `loop()` only decides when to call them.

---

## 5. Connecting to Wi-Fi and MQTT

### Wi-Fi connection

```cpp
void connectWiFi() {
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
    }
}
```

`WiFi.begin()` tells the ESP8266 to start connecting. The `while` loop just waits, it keeps checking until the connection succeeds. Once connected, the ESP8266 gets an IP address from your router just like your phone or laptop would.

### MQTT connection

```cpp
void connectMQTT() {
    mqtt.setKeepAlive(120);
    while (!mqtt.connected()) {
        if (mqtt.connect(CLIENT_ID)) {
            mqtt.subscribe(TOPIC_MODE, 1);
        }
    }
}
```

`mqtt.connect()` opens a connection to Mosquitto on your PC. Once connected it subscribes to `Control/Mode` so it can receive mode change commands from the Android app. `setKeepAlive(120)` tells the broker the ESP8266 will ping it at least every 120 seconds, this prevents the broker from dropping the connection during the 5-second silences in burst and average modes.

### Automatic reconnection

Both `connectWiFi()` and `connectMQTT()` are called at the start of every `loop()` cycle:

```cpp
void loop() {
    if (!mqtt.connected()) connectMQTT();
    mqtt.loop();
    ...
}
```

This means if the connection drops for any reason, router restart, PC sleep, anything, the ESP8266 will automatically reconnect without you having to do anything.

---

## 6. Reading the Sensors

### KY-037 microphone (analog)

```cpp
void readMic(int &val) {
    val = analogRead(MIC_PIN);
}
```

`analogRead()` reads the voltage on pin `A0` and converts it to a number between 0 and 1023. 0 means no voltage (silence), 1023 means maximum voltage (very loud). In a quiet room the KY-037 typically reads around 80-90. Talking or clapping pushes it higher.

The `int &val` means the result is written directly into the variable you pass in, this avoids making a copy of the value and is a common C++ pattern for functions that need to return values.

### MPU-6050 (I2C)

```cpp
void readAccel(float &x, float &y, float &z) {
    int16_t ax, ay, az;
    mpu.getAcceleration(&ax, &ay, &az);
    x = ax / 16384.0;  // converts raw to g (±2g range)
    y = ay / 16384.0;
    z = az / 16384.0;
}
```

When the sensor arrives, uncommenting those lines is all that is needed. `mpu.getAcceleration()` talks to the MPU-6050 over I2C and returns raw 16-bit integers. Dividing by 16384 converts them to g-force units (where 1.0 = Earth gravity). The gyroscope works the same way, divide by 131 to get degrees per second.

### A3144 hall effect (digital)

```cpp
void readHall(int &val) {
    val = digitalRead(HALL_PIN);
}
```

`digitalRead()` returns either `1` (no magnet) or `0` (magnet detected). Simple as that.

---

## 7. The Three Transmission Modes

The mode controls how data is sent to the MQTT broker. The Android app can switch modes by publishing to `Control/Mode`.

### Stream (default)

Every reading is published immediately as it is taken. The mic publishes every 20ms, the other sensors every 100ms.

```
Time:   0ms        20ms   40ms   60ms   80ms   100ms
Mic:    [88]       [89]   [88]   [91]   [88]   [90]
Accel:  [0,0,9.81]                             [0,0,9.81]
```

Good for: real-time monitoring, voice detection.

### Burst

Readings are collected in memory for 5 seconds, then sent all at once in a single large payload.

```
Time:   0 → 5000ms - collecting silently
        5000ms - publishes "88,89,88,91,88,90,..." (all mic readings)
        5000ms - publishes last accel/gyro/hall reading
```

The mic buffer holds up to 250 readings (5s × 50Hz). The other sensors hold up to 50 readings (5s × 10Hz).

Good for: reducing network traffic while preserving raw data.

### Average

Like burst, but instead of storing every reading it just keeps a running sum and count. At the 5-second mark it divides to get the average and sends that single number.

```cpp
averageSum += micValue;
averageCount++;

// At 5 seconds:
int avg = averageSum / averageCount;  // e.g. 88
```

This uses almost no memory, just two integers per sensor regardless of how many readings were taken.

Good for: minimal data transmission, trend monitoring.

### Switching modes

When the Android app publishes `BURST` to `Control/Mode`, the ESP8266's `onMessage()` callback fires:

```cpp
void onMessage(char* topic, byte* payload, unsigned int length) {
    String msg = "";
    for (unsigned int i = 0; i < length; i++) msg += (char)payload[i];
    msg.trim();

    if (String(topic) == TOPIC_MODE) {
        if      (msg == "STREAM")  setMode(MODE_STREAM);
        else if (msg == "BURST")   setMode(MODE_BURST);
        else if (msg == "AVERAGE") setMode(MODE_AVERAGE);
    }
}
```

`setMode()` resets all buffers and timers immediately so there is no stale data carried over from the previous mode.

---

## 8. Timing: How the Loop Works

The `loop()` function runs as fast as the ESP8266 can execute it, potentially thousands of times per second. Sampling the sensor on every single iteration would flood the MQTT broker. Instead the code uses **elapsed time checks** to create independent timers:

```cpp
unsigned long lastSampleMic  = 0;  // tracks when mic was last read
unsigned long lastSampleSlow = 0;  // tracks when other sensors were last read

void loop() {
    mqtt.loop();  // always first, keeps connection alive

    unsigned long now = millis();  // milliseconds since boot

    // Mic: sample every 20ms
    if (now - lastSampleMic >= 20) {
        lastSampleMic = now;
        // read and handle mic
    }

    // Accel/Gyro/Hall: sample every 100ms
    if (now - lastSampleSlow >= 100) {
        lastSampleSlow = now;
        // read and handle other sensors
    }

    // Burst/Average: send every 5000ms
    if (currentMode != MODE_STREAM && now - lastWindow >= 5000) {
        lastWindow = now;
        // send collected data
    }
}
```

`millis()` returns the number of milliseconds since the board powered on. By storing the last time something happened and comparing it to `now`, you get independent timers that never block the loop.

This is important: using `delay(100)` would freeze the entire program for 100ms, preventing MQTT from receiving messages. The timer approach lets everything run concurrently.

---

## 9. What Happens on Startup

When you power on or reset the NodeMCU, this is the exact sequence:

```
1. setup() runs
   ├── Serial.begin(115200)     — opens Serial Monitor connection
   ├── pinMode(D5, INPUT)       — configures Hall pin as input
   ├── connectWiFi()            — connects to your router
   ├── mqtt.setServer(...)      — points MQTT library at Mosquitto
   ├── mqtt.setCallback(...)    — registers onMessage() for incoming commands
   ├── mqtt.setBufferSize(512)  — enough buffer for burst payloads
   └── connectMQTT()            — connects to broker, subscribes to Control/Mode

2. loop() starts running forever
   ├── Every call: mqtt.loop(), keep-alive pings + receive commands
   ├── Every 20ms:  read mic, publish/buffer/accumulate based on mode
   ├── Every 100ms: read accel/gyro/hall, publish/buffer/accumulate
   └── Every 5000ms (burst/average only): send collected data
```

In Serial Monitor you should see:

```
WiFi...... OK 192.168.178.xxx
MQTT...connected
Mode → STREAM
STREAM: 88
STREAM: 89
STREAM: 88
```

---
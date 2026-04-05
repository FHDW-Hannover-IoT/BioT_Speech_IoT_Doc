# How MQTT Works in BioT Speech IoT

> This document explains how the BioT Speech IoT app uses MQTT to move sensor data from the ESP8266 hardware to the Android app. No prior knowledge of message brokers required.

---

## Table of Contents

1. [What is MQTT?](#1-what-is-mqtt)
2. [The Three Players](#2-the-three-players)
3. [How Messages Travel](#3-how-messages-travel)
4. [Topics — The Address System](#4-topics--the-address-system)
5. [How This Project Uses MQTT](#5-how-this-project-uses-mqtt)
6. [Retained Messages](#6-retained-messages)
7. [QoS — Delivery Guarantees](#7-qos--delivery-guarantees)
8. [Keep-Alive — Staying Connected](#8-keep-alive--staying-connected)
9. [The Full Picture](#9-the-full-picture)
10. [What Happens When Something Goes Wrong](#10-what-happens-when-something-goes-wrong)

---

## 1. What is MQTT?

MQTT stands for **Message Queuing Telemetry Transport**. Here is what it actually means in plain terms:

Imagine a walkie-talkie radio system. One person talks on a channel, and everyone who is tuned to that channel hears it. Nobody calls anyone directly, you just broadcast on a channel and whoever is listening will receive it.

MQTT works exactly like that, except over Wi-Fi and the internet instead of radio waves. It was originally designed for IoT devices because it is extremely lightweight, it uses very little data and works even on tiny microcontrollers like the ESP8266.

---

## 2. The Three Players

Every MQTT system has exactly three types of participant:

```
┌─────────────┐        ┌─────────────┐        ┌─────────────┐
│  Publisher  │──────▶│   Broker    │──────▶ │ Subscriber  │
│ (ESP8266)   │        │ (Mosquitto) │        │ (App)       │
└─────────────┘        └─────────────┘        └─────────────┘
```

### The Publisher
The device that **sends** messages. In this project that is the **ESP8266 NodeMCU**. It reads sensor values (microphone, accelerometer, gyroscope, hall sensor) and sends them out.

### The Broker
The **middleman** that sits in between. It receives every message and forwards it to whoever is interested. In this project that is **Mosquitto**, running on your PC. The broker does not care what the messages contain, it just routes them. 

### The Subscriber
The device that **receives** messages. In this project that is the **Android app**. It tells the broker which topics it is interested in, and the broker sends it everything published on those topics.

> **Remember:** The ESP8266 and the Android app never talk to each other directly. They both only talk to Mosquitto. Mosquitto is the bridge between them.

---

## 3. How Messages Travel

Here is the step-by-step journey of a single microphone reading:

```
1. ESP8266 reads the mic → gets value 88

2. ESP8266 sends to Mosquitto:
   "Hey broker, I am publishing value 88 on topic Sensor/Mic"

3. Mosquitto receives it and checks:
   "Who has subscribed to Sensor/Mic?" → The Android app has

4. Mosquitto forwards to the Android app:
   "Here is a message on Sensor/Mic: 88"

5. Android app receives it and updates the UI:
   Wert: 88
```

The whole journey takes a few milliseconds over a local Wi-Fi network.

---

## 4. Topics: The Address System

A **topic** is like a channel name or a postal address for a message. It tells the broker where a message belongs and who should receive it.

Topics in MQTT use forward slashes to create a hierarchy, similar to folders on a computer:

```
Sensor/Mic
Sensor/Bewegung
Sensor/Gyro
Sensor/Magnet
Control/Mode
Control/Mode/Ack
```

The ESP8266 **publishes** to sensor topics. The Android app **subscribes** to those same topics so it receives every message. The app also **publishes** to `Control/Mode` to send commands back to the ESP8266, and the ESP8266 **subscribes** to that topic so it receives mode changes.

This means the communication goes both ways:

```
ESP8266  ──publishes──▶  Sensor/Mic         ──▶  Android app
ESP8266  ──publishes──▶  Sensor/Bewegung    ──▶  Android app
ESP8266  ──publishes──▶  Sensor/Gyro        ──▶  Android app
ESP8266  ──publishes──▶  Sensor/Magnet      ──▶  Android app

Android  ──publishes──▶  Control/Mode       ──▶  ESP8266
```

The Android app subscribes to all sensor topics. The ESP8266 subscribes only to `Control/Mode`.

---

## 5. How This Project Uses MQTT

### Sensor data flow (ESP8266 → Android)

The ESP8266 reads its sensors and publishes the values to specific topics. The payload (the actual data in the message) is a simple text string:

| Topic | What it contains | Example payload |
|-------|-----------------|-----------------|
| `Sensor/Mic` | Microphone sound level | `88` |
| `Sensor/Bewegung` | Accelerometer X, Y, Z | `1.23,-0.45,9.81` |
| `Sensor/Gyro` | Gyroscope X, Y, Z | `-0.94,4.50,-4.06` |
| `Sensor/Magnet` | Hall effect sensor | `1,0,0` |

The Android app receives these, parses the numbers out of the text, updates the screen, and writes the data to the local Room database.

### Control flow (Android → ESP8266)

When you tap a mode button on the app, the app publishes a command:

| Topic | What it contains | Example payload |
|-------|-----------------|-----------------|
| `Control/Mode` | Which transmission mode to use | `STREAM`, `BURST`, or `AVERAGE` |

The ESP8266 receives this, switches its internal mode.

### The three transmission modes

The mode system exists because sending every mic reading individually at 50Hz (50 times per second) uses a lot of network capacity. The three modes let you trade off between data resolution and network usage:

| Mode | What ESP8266 does | What arrives at the app |
|------|-------------------|------------------------|
| **Stream** | Sends every reading immediately | Continuous flow of individual values |
| **Burst** | Collects 5 seconds of readings, then sends them all at once | One large batch every 5 seconds |
| **Average** | Computes the mean over 5 seconds, sends one number | One averaged value every 5 seconds |

The mic samples at 50Hz (every 20ms). The other sensors sample at 10Hz (every 100ms). The mic needs the higher rate because voice patterns require more detail.

---

## 6. Retained Messages

A normal MQTT message disappears once it has been delivered. If you subscribe to a topic after a message was sent, you missed it, you get nothing until the next message arrives.

A **retained message** works differently. The broker keeps a copy of the last retained message on a topic. When a new subscriber connects, the broker immediately sends them the last retained message, even if it was published a long time ago.

This project uses retained messages for `Control/Mode`. Here is why:

Imagine you set the mode to **Burst**, then restart the ESP8266. Without retained messages, the ESP8266 would boot up in Stream mode (its default) and wait for a new command from the app. With retained messages, as soon as the ESP8266 reconnects and subscribes to `Control/Mode`, Mosquitto immediately sends it the last known mode (**Burst**) and the ESP8266 picks up where it left off.

The same applies to the Android app. When it reconnects it receives the current mode via `Control/Mode` and immediately highlights the correct button.

---

## 7. QoS: Delivery Guarantees

QoS stands for **Quality of Service**. It controls how hard MQTT tries to guarantee that a message is delivered. There are three levels:

| Level | Name | What it means |
|-------|------|--------------|
| QoS 0 | At most once | Fire and forget, message may be lost |
| **QoS 1** | **At least once** | **Message is guaranteed to arrive, but may arrive twice** |
| QoS 2 | Exactly once | Guaranteed to arrive exactly once (slowest) |

This project uses **QoS 1** for everything. This means:

- Every message is guaranteed to reach the broker at least once
- The broker guarantees to deliver it to every subscriber at least once
- Duplicate messages can occasionally happen but the app handles this gracefully since sensor values updating twice is harmless

QoS 0 would risk losing sensor readings, and QoS 2 adds unnecessary overhead for sensor data.

---

## 8. Keep-Alive: Staying Connected

MQTT connections run over TCP, which is a persistent connection that stays open. But if no messages are sent for a long time, the TCP connection can silently drop without either side noticing. This is especially a problem in burst and average modes where the ESP8266 is silent for up to 5 seconds.

The solution is the **keep-alive** mechanism. The ESP8266 tells Mosquitto on connection: "I will send you a ping at least every 120 seconds to prove I am still alive." If Mosquitto does not hear from the ESP8266 within that window, it drops the connection.

In this project the keep-alive is set to 120 seconds:

```cpp
mqtt.setKeepAlive(120);
```

> **Note:** This is handled automatically by the MQTT library, you do not need to write any ping logic yourself. The library sends a tiny PINGREQ packet in the background and Mosquitto responds with PINGRESP.

---

## 9. The Full Picture

Here is the complete flow from sensor to screen, step by step:

```
            ┌───────────────────────────────────────────────┐
            │                 ESP8266 NodeMCU               │
            │                                               │
            │  KY-037 mic ──▶ analogRead(A0) every 20ms    │
            │  MPU-6050   ──▶ I2C read every 100ms         │
            │  A3144      ──▶ digitalRead(D5) every 100ms  │
            │                        │                      │
            │              checks current mode              │
            │         STREAM / BURST / AVERAGE              │
            │                        │                      │
            │              mqtt.publish(topic, value)       │
            └────────────────────────┼──────────────────────┘
                                     │ Wi-Fi (TCP port 1883)
                                     ▼
            ┌───────────────────────────────────────────────┐
            │              Mosquitto Broker (PC)            │
            │                                               │
            │  Receives message on topic e.g. "Sensor/Mic"  │
            │  Looks up: who subscribed to "Sensor/Mic"?    │
            │  → Android app                                │
            │  Forwards message to Android app              │
            └────────────────────────┼──────────────────────┘
                                     │ Wi-Fi or localhost (emulator)
                                     ▼
            ┌─────────────────────────────────────────────────┐
            │                   Android App                   │
            │                                                 │
            │  MqttHandler.messageArrived(topic, payload)     │
            │                        │                        │
            │              switch(topic)                      │
            │         ┌──────────────┼──────────────────┐     │
            │         ▼              ▼                  ▼     │
            │  Sensor/Mic    Sensor/Bewegung    Control/Mode  │
            │  update UI     update UI +        update button │
            │  (no DB)       write to Room DB   highlighting  │
            └─────────────────────────────────────────────────┘
```

---

## 10. What Happens When Something Goes Wrong

MQTT is designed to handle disconnections gracefully.

**If the ESP8266 loses Wi-Fi:**
The ESP8266's `connectMQTT()` function runs in a loop, it keeps retrying every 2 seconds until it reconnects. Once back online it subscribes to `Control/Mode` again, and Mosquitto immediately delivers the retained mode message so it knows which mode to use.

**If the Android app loses connection:**
The Paho MQTT library has `automaticReconnect = true` set. It silently reconnects in the background. Once reconnected it re-subscribes to all topics and receives the retained `Control/Mode` message so the UI buttons update to the correct state.

**If Mosquitto is stopped:**
Both the ESP8266 and the Android app will report a connection failure and keep retrying. Once Mosquitto is restarted, both automatically reconnect within a few seconds.

**If a message is lost:**
QoS 1 ensures the broker will retry delivery. For sensor data this rarely matters, a missed reading is just replaced by the next one a few milliseconds later.

---
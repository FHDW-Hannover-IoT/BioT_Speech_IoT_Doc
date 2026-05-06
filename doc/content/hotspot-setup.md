# BioT — Hotspot Demo Setup

**4-device deployment: Broker · Sensor Mockup · Backend · Phone**

---

## System Topology

```
              Phone Hotspot (192.168.43.x)
              ─────────────────────────────────────────────────────
              │                                                   │
   ┌──────────┴──────────┐                          ┌────────────┴──────────────┐
   │  Laptop A — Broker  │                          │  Laptop C — Backend System│
   │                     │◄── Sensor/* (data) ──────│  FastAPI  :8001           │
   │  Mosquitto :1883    │──── Sensor/* (data) ─────►│  MCP      :8002 (intern)  │
   └──────────┬──────────┘                          │  SQLite   data/sensor.db  │
              │                                     │  MQTT Subscriber          │
              │ Sensor/* (data, red)                └───────────────────────────┘
              │ Control/* (config, green)                         ▲
              │                                                   │ POST /chat
   ┌──────────┴──────────┐                                       │
   │  Laptop B — Sensor  │                                       │
   │  Mockup             │──── Sensor/* ──────────►  Broker      │
   │  Python MQTT pub    │                                        │
   └─────────────────────┘                          ┌────────────┴──────────────┐
                                                    │  Phone — Android App      │
                                                    │  Room DB (live charts)    │
                                                    │  Voice → POST /chat       │
                                                    │  MQTT sub/pub → Broker    │
                                                    └───────────────────────────┘
```

### Data flow (matching diagram arrows)

| Arrow colour | Direction | What flows |
|---|---|---|
| Red | Sensor Mockup → Broker → App & Backend | Live sensor readings (`Sensor/Bewegung`, `Sensor/Gyro`, `Sensor/Magnet`, `Sensor/Mic`) |
| Green | App → Broker → App | Configuration and control messages (`Control/Mode`, `Control/OperatingMode`) |
| — | Phone → Backend `:8001` | Voice query via `POST /chat`, structured JSON action back |

---

## Before You Start — Hotspot

1. One device creates the hotspot (recommend the phone — keeps one laptop free).
2. **Disable client isolation** (AP isolation) in the hotspot settings. Without this, the two laptops and the phone can each reach the internet but not each other — all MQTT and HTTP traffic silently drops.
3. Connect all three laptops to the hotspot.
4. Run `ipconfig` (Windows) on each laptop and note the IP in the hotspot subnet (e.g. `192.168.43.x`). You will need all three IPs in the steps below.

```
Laptop A IP  = 192.168.43.___   ← fill in
Laptop B IP  = 192.168.43.___   ← fill in
Laptop C IP  = 192.168.43.___   ← fill in
```

---

## Laptop A — Broker (Mosquitto)

### 1. Configure Mosquitto to listen on all interfaces

Open `C:\Program Files\mosquitto\mosquitto.conf` and add/update:

```conf
listener 1883 0.0.0.0
allow_anonymous true
```

Restart the service:

```powershell
net stop mosquitto
net start mosquitto
```

### 2. Open port 1883 in Windows Firewall

```powershell
New-NetFirewallRule -DisplayName "Mosquitto MQTT" `
  -Direction Inbound -Protocol TCP -LocalPort 1883 -Action Allow
```

### 3. Verify

From any other device on the hotspot:

```powershell
# Windows
Test-NetConnection -ComputerName 192.168.43.A -Port 1883
```

---

## Laptop B — Sensor Mockup

This laptop simulates the ESP8266 by publishing realistic sensor data to the broker.

### 1. Install Python dependencies

```powershell
pip install paho-mqtt
```

### 2. Create `sensor_mockup.py`

```python
"""
sensor_mockup.py — BioT Sensor Mockup
Publishes fake MPU-6050 + A3144 readings to the Mosquitto broker.
"""

import math
import random
import time
import paho.mqtt.client as mqtt

BROKER_HOST = "192.168.43.A"   # ← Laptop A IP
BROKER_PORT = 1883
PUBLISH_INTERVAL = 0.5         # seconds between publishes

client = mqtt.Client(client_id="biot-sensor-mockup", protocol=mqtt.MQTTv5)
client.connect(BROKER_HOST, BROKER_PORT, keepalive=60)
client.loop_start()

print(f"Sensor mockup running → {BROKER_HOST}:{BROKER_PORT}")
print("Ctrl+C to stop\n")

t = 0.0
try:
    while True:
        t += PUBLISH_INTERVAL

        # Accelerometer: gentle sine wave + noise (units: g)
        ax = round(0.02 * math.sin(t) + random.gauss(0, 0.005), 4)
        ay = round(0.01 * math.cos(t) + random.gauss(0, 0.005), 4)
        az = round(9.81 + random.gauss(0, 0.01), 4)
        client.publish("Sensor/Bewegung", f"{ax},{ay},{az}", qos=1)

        # Gyroscope: slow drift + noise (units: deg/s)
        gx = round(random.gauss(0, 0.5), 4)
        gy = round(random.gauss(0, 0.5), 4)
        gz = round(random.gauss(0, 0.5), 4)
        client.publish("Sensor/Gyro", f"{gx},{gy},{gz}", qos=1)

        # Magnetometer (arbitrary units)
        mx = round(random.gauss(20.0, 1.0), 4)
        my = round(random.gauss(-5.0, 1.0), 4)
        mz = round(random.gauss(40.0, 1.0), 4)
        client.publish("Sensor/Magnet", f"{mx},{my},{mz}", qos=1)

        # Microphone (display-only, not persisted by backend)
        mic = random.randint(100, 400)
        client.publish("Sensor/Mic", str(mic), qos=0)

        print(f"  Accel: {ax},{ay},{az}  Gyro: {gx},{gy},{gz}  Mic: {mic}")
        time.sleep(PUBLISH_INTERVAL)

except KeyboardInterrupt:
    print("\nStopping mockup.")
    client.loop_stop()
    client.disconnect()
```

### 3. Run

```powershell
python sensor_mockup.py
```

You should see a line printed for every publish cycle. If you see a connection error, check that Laptop A's firewall rule is active and the IP is correct.

---

## Laptop C — Backend System (LLM_App)

### 1. Prerequisites

- Python 3.11+
- `uv` package manager:
  ```powershell
  powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
  ```
- Copy the `BioT_Refactor/LLM_App/` folder to this machine (USB or git clone)

### 2. Create `.env`

Copy `.env.example` to `.env` and set these values:

```env
LLM_PROVIDER=anthropic
LLM_API_KEY=<your actual API key>
LLM_MODEL=claude-sonnet-4-6

# Leave empty → defaults to data/sensor_database.db inside the project folder
SQLITE_DB_PATH=
MCP_FS_ROOTS=

# FastAPI — bind to all interfaces so the phone can reach it
SERVER_HOST=0.0.0.0
SERVER_PORT=8001

# MCP server is internal-only (used by the FastAPI agent on the same machine)
MCP_SERVER_PORT=8002

# CRITICAL: must be Laptop A's hotspot IP, NOT 127.0.0.1
MQTT_BROKER_HOST=192.168.43.A
MQTT_BROKER_PORT=1883
```

> Port 8002 (MCP) is only used internally between FastAPI and its subprocess
> on the same machine. No firewall rule needed for it.

### 3. Open port 8001 in Windows Firewall

```powershell
New-NetFirewallRule -DisplayName "BioT Backend API" `
  -Direction Inbound -Protocol TCP -LocalPort 8001 -Action Allow
```

### 4. Install dependencies and start

```powershell
uv sync
uv run python -m app.main
```

Expected startup output:

```
INFO | Starting MCP sensor server on port 8002
INFO | MCP server ready at http://localhost:8002/mcp
INFO | Agent ready (provider=anthropic, model=claude-sonnet-4-6, ...)
INFO | MQTT subscriber running (broker=192.168.43.A:1883)
INFO | Application startup complete.
```

### 5. Smoke test from Laptop A or B

```powershell
curl.exe -X POST http://192.168.43.C:8001/chat `
  -H "Content-Type: application/json" `
  -d '{"message": "How many gyro readings are in the database?"}'
```

Expected: a JSON response with `"reply"` containing a JSON action with a `"tts"` field.

---

## Phone — Android App

### 1. Update gradle.properties before building

In `BioT_Refactor/App/` (next to `settings.gradle`), create or edit `gradle.properties`:

```properties
# Broker is on Laptop A
biot.phoneBroker=tcp://192.168.43.A:1883

# LLM backend is on Laptop C
biot.llmHostPhone=192.168.43.C
biot.llmPort=8001
```

These feed into `BuildConfig` via `app/build.gradle` and are read at runtime by `AppConfig.java`.

### 2. Rebuild and install

Build the debug APK from Android Studio and sideload it, or:

```powershell
cd BioT_Refactor\App
gradlew.bat installDebug
```

### 3. Runtime check

On the phone's main screen, tap connect — the MQTT status should turn green once it reaches Laptop A on port 1883. Try a voice command ("Tell me the gyro X value") — the app posts to Laptop C port 8001 and speaks the TTS reply.

---

## Test Sequence (in order)

| Step | Command / Action | Proves |
|---|---|---|
| 1 | Ping Laptop C from Laptop B: `ping 192.168.43.C` | Hotspot routing works between laptops |
| 2 | `Test-NetConnection 192.168.43.A 1883` from Laptop C | MQTT port reachable across hotspot |
| 3 | `mosquitto_sub -h 192.168.43.A -t "Sensor/#" -v` on Laptop C | Sensor Mockup data flowing through Broker |
| 4 | GET `http://192.168.43.C:8001/health` from phone browser | Backend reachable from phone |
| 5 | `curl.exe` smoke test from Laptop A (see above) | Full LLM + MCP + SQLite stack works |
| 6 | Open Android app → MQTT connects | Phone reaches Broker on Laptop A |
| 7 | Speak a voice command in the app | Full end-to-end round-trip |

---

## Known Code Issues

These issues exist in the current codebase and should be addressed before the demo.

### Critical — verify before demo

**`LLM_App/app/agent.py` line 313 — MCP tool dispatch URL**

The agent calls:
```python
httpx.post(f"{self._mcp_url}/tools/call", json={"name": name, "arguments": inputs})
```
This resolves to `POST http://localhost:8002/mcp/tools/call`.
The standard FastMCP Streamable HTTP transport only listens at `/mcp` (root) using JSON-RPC format — there is no `/tools/call` sub-path in the MCP spec.

If tool calls return HTTP 404 or "Tool dispatch error" during the smoke test, replace the dispatch with the correct JSON-RPC format:

```python
response = httpx.post(
    self._mcp_url,          # http://localhost:8002/mcp
    json={
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {"name": name, "arguments": inputs},
    },
    timeout=30.0,
)
```

### Minor — can fix after demo

| File | Line | Issue |
|---|---|---|
| `app/agent.py` | 277 | `db_path: Path` parameter is accepted but never stored or used — dead parameter, remove it |
| `mcp_server/mqtt_subscriber.py` | 293 | `_db_insert` opens and closes a new SQLite connection per INSERT — under burst mode this is slow; use a persistent WAL-mode connection |
| `app/database.py` | — | Entire file is dead code, not imported anywhere; duplicates logic in the MCP server inline |
| `.env` | 24–27 | `SQLITE_DB_PATH` and `MCP_FS_ROOTS` still point to old `BioT_Speech_IoT_LLM_App` path — clear both |
| `app/main.py` | 83 | Unconditional `time.sleep(3.0)` before probe loop — reduce to 1.0 or remove |

---

## MQTT Topics Reference

| Topic | Direction | Format | Persisted |
|---|---|---|---|
| `Sensor/Bewegung` | Mockup/ESP → Broker → App & Backend | `"x,y,z"` floats (g-force) | Yes → `accel_data` |
| `Sensor/Gyro` | Mockup/ESP → Broker → App & Backend | `"x,y,z"` floats (deg/s) | Yes → `gyro_data` |
| `Sensor/Magnet` | Mockup/ESP → Broker → App & Backend | `"x,y,z"` floats | Yes → `magnet_data` |
| `Sensor/Mic` | Mockup/ESP → Broker → App | integer 0–1023 | No (display-only) |
| `Control/Mode` | App/LLM → Broker → ESP | `STREAM` \| `BURST` \| `AVERAGE` | Retained |
| `Control/OperatingMode` | App/LLM → Broker → ESP | `AUTARK` \| `SUPERVISION` \| `EVENT` \| `IDENTIFICATION` | Retained |

---

## Port Summary

| Port | Machine | Service | Who connects |
|---|---|---|---|
| 1883 | Laptop A | Mosquitto MQTT | Laptop B (pub), Laptop C (sub), Phone (sub/pub) |
| 8001 | Laptop C | FastAPI `/chat`, `/health` | Phone (voice queries) |
| 8002 | Laptop C | MCP Server (internal only) | FastAPI subprocess on same machine |

---

*BioT Speech IoT — Demo Setup Guide · FHDW Hannover 2025-26*

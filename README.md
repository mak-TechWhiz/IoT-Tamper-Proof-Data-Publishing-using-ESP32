# ESP32 → Node-RED → InfluxDB with HMAC Authentication

This repository demonstrates how to securely publish sensor data from an ESP32 to InfluxDB, using HMAC for end-to-end message integrity. Node-RED sits in the middle to validate the HMAC and forward only valid data into InfluxDB.

### Table of Contents

1. [Overview](#overview)  
2. [Prerequisites](#prerequisites)  
3. [ESP32 Side](#esp32-side)  
   - [Folder Structure](#folder-structure)  
   - [Configuring `secrets.h`](#configuring-secretsh)  
   - [Flashing the ESP32](#flashing-the-esp32)  
4. [Node-RED Side](#node-red-side)  
   - [Installing Dependencies](#installing-dependencies)  
   - [Importing the Flow](#importing-the-flow)  
   - [Configuring the HMAC Secret & InfluxDB Node](#configuring-the-hmac-secret--influxdb-node)  
5. [InfluxDB Side](#influxdb-side)  
   - [Using Docker Compose](#using-docker-compose)  
   - [Running InfluxDB & Creating a Bucket/User](#running-influxdb--creating-a-bucketuser)  
6. [Putting It All Together](#putting-it-all-together)  
7. [Troubleshooting](#troubleshooting)  
8. [License](#license)

---

## Overview

1. **ESP32**  
   - Reads or simulates a sensor value (e.g. temperature).  
   - Packages the data into a JSON string.  
   - Computes an HMAC-SHA256 over that JSON, using a shared secret.  
   - Publishes to an MQTT topic:  
     ```
     topic: sensors/esp32/temperature
     payload: {
       "ts": 1685870400,
       "value": 26.3,
       "hmac": "<hex-encoded-hmac>"
     }
     ```

2. **Node-RED**  
   - Subscribes to `sensors/esp32/temperature` via an MQTT In node.  
   - A Function node recomputes HMAC (using the same secret); if valid, it passes the data to an InfluxDB Out node.  
   - Optionally, if the HMAC fails, it can drop/alert.

3. **InfluxDB**  
   - Receives valid measurements via its HTTP API (through Node-RED).  
   - Stores them in a bucket called `esp32_sensors`.  
   - Allows you to visualize or query the data later.

---

## Prerequisites

- **Hardware:**  
  - An ESP32 development board (e.g. Wemos D1 R32, ESP32-DevKitC).  
  - USB cable to program your ESP32.  
- **Software on your PC:**  
  - [Arduino IDE (≥ 1.8.13) or PlatformIO](https://www.arduino.cc/en/software).  
  - [Node-RED v2.x](https://nodered.org/docs/getting-started/).  
  - [InfluxDB 2.x](https://docs.influxdata.com/influxdb/v2.0/) (we’ll show a Docker Compose setup).  
  - `openssl` (optional, to compute/verify HMAC offline during testing).  
- **Libraries/Node-RED Nodes:**  
  - **Arduino side (Arduino IDE):**  
    - `WiFi.h` (bundled)  
    - `PubSubClient.h` (for MQTT)  
    - `mbedtls/md.h` (bundled in ESP32 core, for HMAC)  
  - **Node-RED side:**  
    - `node-red-contrib-crypto-js` (for HMAC verification)  
    - `node-red-contrib-influxdb` (v1 or v2 depending on your InfluxDB version)  

---

## ESP32 Side

### Folder Structure

```text
esp32/
├── HMAC_Publisher.ino
└── secrets.h     ← *Do not share this file publicly!*

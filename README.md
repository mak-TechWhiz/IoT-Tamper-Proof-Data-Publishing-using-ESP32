# Secure ESP32 → Node-RED → InfluxDB Pipeline with HMAC-SHA256

This repository contains the ESP32 firmware (`HMAC_Publisher.ino`) that reads heart-rate (MAX30105) and temperature (DS18B20), computes an HMAC-SHA256 over each JSON payload, and publishes it via MQTT. On the server side, Node-RED verifies the HMAC before forwarding valid data into InfluxDB. No TLS/mTLS—only HMAC.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [ESP32 (Client) Side](#esp32-client-side)

   * [Library Installation](#library-installation)
   * [Sketch Overview](#sketch-overview)
   * [Configuration](#configuration)
   * [Flashing & Running](#flashing--running)
4. [Node-RED + InfluxDB (Server) Side](#node-red--influxdb-server-side)

   * [MQTT Broker Setup](#mqtt-broker-setup)
   * [Node-RED HMAC Verification](#node-red-hmac-verification)
   * [InfluxDB Configuration](#influxdb-configuration)
5. [Putting It All Together](#putting-it-all-together)
6. [Troubleshooting](#troubleshooting)
7. [License](#license)

---

## Prerequisites

* **Hardware**

  * ESP32 development board (e.g. “ESP32 Dev Module”)
  * MAX30105 heart-rate sensor module
  * DS18B20 one-wire temperature sensor (4.7 kΩ pull-up to 3.3 V)
  * USB cable to flash and power the ESP32

* **Software**

  * [Arduino IDE (≥ 1.8.13)](https://www.arduino.cc/en/software) or [PlatformIO](https://platformio.org/)
  * [Node-RED v2.x](https://nodered.org/docs/getting-started/)
  * [InfluxDB 2.x](https://docs.influxdata.com/influxdb/v2.0/) (standalone or Docker)
  * An MQTT broker (e.g. Mosquitto) running on your LAN (no TLS)

* **Arduino Libraries** (install via Sketch → Include Library → Manage Libraries…)

  * `MAX30105` (SparkFun)
  * `heartRate` (included in MAX30105 examples)
  * `OneWire` (by Paul Stoffregen)
  * `DallasTemperature` (by Miles Burton)
  * `PubSubClient` (by Nick O’Leary)
  * `mbedtls/md.h` is built into the ESP32 core

* **Node-RED Nodes** (install via the Node-RED palette manager)

  * `node-red-contrib-crypto-js` (HMAC-SHA256)
  * `node-red-contrib-influxdb` (InfluxDB v2 integration)

---

## Project Structure

```
your-project/
├── README.md
└── esp32/
    └── HMAC_Publisher.ino
```

* **README.md**: This file—explains how to set up ESP32 firmware and configure the server side (Node-RED + InfluxDB).
* **esp32/HMAC\_Publisher.ino**: Arduino sketch for the ESP32. It reads sensor data, builds a JSON payload, computes HMAC-SHA256, and publishes via MQTT.

---

## ESP32 (Client) Side

### Library Installation

1. Open Arduino IDE.
2. Go to **Sketch → Include Library → Manage Libraries…**
3. Search for and install:

   * **MAX30105** (SparkFun)
   * **OneWire** (by Paul Stoffregen)
   * **DallasTemperature** (by Miles Burton)
   * **PubSubClient** (by Nick O’Leary)
4. Verify your ESP32 board package is installed:

   * **File → Preferences → Additional Boards Manager URLs** → add
     `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
   * **Tools → Board → Boards Manager…** → search “esp32” → install “esp32 by Espressif Systems”

### Sketch Overview

```cpp
#include <Wire.h>
#include "MAX30105.h"
#include "heartRate.h"
#include <OneWire.h>
#include <DallasTemperature.h>
#include <WiFi.h>
#include <PubSubClient.h>
#include "mbedtls/md.h"

// ===== Sensor Config =====
MAX30105 particleSensor;
const byte RATE_SIZE = 8;
byte rates[RATE_SIZE];
byte rateSpot = 0;
long lastBeat = 0;
float beatsPerMinute;
int beatAvg;

static const int ONE_WIRE_PIN = 4;
OneWire oneWire(ONE_WIRE_PIN);
DallasTemperature sensors(&oneWire);

// ===== Network Config =====
const char* ssid       = "YOUR_WIFI_SSID";
const char* password   = "YOUR_WIFI_PASSWORD";
const char* mqtt_server = "192.168.1.100";  // Replace with your broker’s IP
const int mqtt_port    = 1883;
const char* mqtt_topic = "sensor/data";

// ===== HMAC-SHA256 Key (32 ASCII chars) =====
const char* HMAC_KEY = "0123456789ABCDEF0123456789ABCDEF";

WiFiClient netClient;
PubSubClient client(netClient);

void setup_wifi() {
  Serial.printf("Connecting to Wi-Fi: %s\n", ssid);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }
  Serial.printf("\nWi-Fi connected. IP: %s\n", WiFi.localIP().toString().c_str());
}

void reconnect_mqtt() {
  while (!client.connected()) {
    Serial.print("Connecting to MQTT broker... ");
    if (client.connect("ESP32_HMAC_Client")) {
      Serial.println("Connected");
    } else {
      Serial.printf("Failed [rc=%d], retrying...\n", client.state());
      delay(2000);
    }
  }
}

String computeHMAC(const String& data, const char* key) {
  const uint8_t* keyBytes = (const uint8_t*) key;
  size_t keyLen = strlen(key);

  const uint8_t* dataBytes = (const uint8_t*) data.c_str();
  size_t dataLen = data.length();

  unsigned char mac[32];
  const mbedtls_md_info_t* info = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);
  mbedtls_md_context_t ctx;
  mbedtls_md_init(&ctx);
  mbedtls_md_setup(&ctx, info, 1);
  mbedtls_md_hmac_starts(&ctx, keyBytes, keyLen);
  mbedtls_md_hmac_update(&ctx, dataBytes, dataLen);
  mbedtls_md_hmac_finish(&ctx, mac);
  mbedtls_md_free(&ctx);

  char hexBuf[65];
  for (int i = 0; i < 32; i++) {
    sprintf(hexBuf + (i*2), "%02x", mac[i]);
  }
  hexBuf[64] = '\0';
  return String(hexBuf);
}

void setup() {
  Serial.begin(115200);

  // Initialize MAX30105 (I2C on SDA=27, SCL=26)
  Wire.begin(27, 26);
  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30105 not found. Stopping.");
    while (1);
  }
  particleSensor.setup();  
  particleSensor.setPulseAmplitudeRed(0x0A);

  // Initialize DS18B20
  sensors.begin();

  // Network
  setup_wifi();
  client.setServer(mqtt_server, mqtt_port);
}

void loop() {
  if (!client.connected()) {
    reconnect_mqtt();
  }
  client.loop();

  // Measure heart rate
  long irValue = particleSensor.getIR();
  if (checkForBeat(irValue)) {
    long delta = millis() - lastBeat;
    lastBeat = millis();
    beatsPerMinute = 60.0 / (delta / 1000.0);
    if (beatsPerMinute > 20 && beatsPerMinute < 255) {
      rates[rateSpot++] = (byte)beatsPerMinute;
      rateSpot %= RATE_SIZE;
      beatAvg = 0;
      for (byte x = 0; x < RATE_SIZE; x++) {
        beatAvg += rates[x];
      }
      beatAvg /= RATE_SIZE;
    }
  }
  if (irValue < 50000) {
    beatsPerMinute = 0;
    beatAvg = 0;
  }

  // Every 2 seconds, read temperature and publish
  static unsigned long lastSend = 0;
  if (millis() - lastSend < 2000) return;
  lastSend = millis();

  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);

  // Build JSON without HMAC
  char dataBuf[128];
  int dataLen = snprintf(dataBuf, sizeof(dataBuf),
    "{\"tempC\":%.2f,\"bpm\":%.1f,\"avg_bpm\":%d}",
    tempC, beatsPerMinute, beatAvg);

  // Compute HMAC-SHA256
  String payloadData = String(dataBuf);
  String hmac = computeHMAC(payloadData, HMAC_KEY);

  // Final payload
  char payload[256];
  snprintf(payload, sizeof(payload),
    "{\"data\":%s,\"hmac\":\"%s\"}",
    payloadData.c_str(), hmac.c_str());

  if (client.publish(mqtt_topic, payload)) {
    Serial.printf("Published: %s\n", payload);
  } else {
    Serial.println("Publish failed!");
  }
}
```

### Configuration

1. In the sketch, set:

   ```cpp
   const char* ssid       = "YOUR_WIFI_SSID";
   const char* password   = "YOUR_WIFI_PASSWORD";
   const char* mqtt_server = "192.168.1.100";  // Your MQTT broker’s IP
   const int mqtt_port    = 1883;
   const char* mqtt_topic = "sensor/data";
   const char* HMAC_KEY   = "0123456789ABCDEF0123456789ABCDEF"; // 32 ASCII chars
   ```
2. Ensure your MQTT broker is running without TLS and listening on port 1883.
3. The HMAC key must match exactly (string and casing) in Node-RED.

### Flashing & Running

1. Select **Tools → Board → ESP32 Dev Module** (or your specific board).
2. Select the correct COM port under **Tools → Port**.
3. Click **Sketch → Upload**.
4. Open **Serial Monitor** at 115200 baud to see Wi-Fi, MQTT, and publishing status.

---

## Node-RED + InfluxDB (Server) Side

### MQTT Broker Setup

* Install and run an MQTT broker (e.g. Mosquitto) on the same LAN.

  ```bash
  sudo apt update
  sudo apt install mosquitto mosquitto-clients
  sudo systemctl enable mosquitto
  sudo systemctl start mosquitto
  ```
* Ensure it listens on port 1883 (default). No authentication or TLS required.

### Node-RED HMAC Verification

1. Install Node-RED (if you haven’t already):

   ```bash
   npm install -g --unsafe-perm node-red
   ```

2. Start Node-RED:

   ```bash
   node-red
   ```

3. In the Node-RED editor ([http://localhost:1880](http://localhost:1880)):

   * Open **Manage palette → Install**, then install:

     * `node-red-contrib-crypto-js`
     * `node-red-contrib-influxdb`
   * Build a flow with:

     1. **MQTT In** node:

        * Broker: `localhost:1883`
        * Topic: `sensor/data`
        * Output: parsed JSON
     2. **Function** node (“Verify HMAC”):

        ```js
        // Must match HMAC_KEY in ESP32 sketch
        const SECRET = "0123456789ABCDEF0123456789ABCDEF";

        try {
          let obj = msg.payload;   // parsed JSON
          let dataObj = obj.data;  // { tempC:…, bpm:…, avg_bpm:… }
          let recvHmac = obj.hmac;

          // Reconstruct JSON string exactly
          let jsonData = JSON.stringify({
            tempC: dataObj.tempC,
            bpm: dataObj.bpm,
            avg_bpm: dataObj.avg_bpm
          });

          let CryptoJS = global.get('CryptoJS');
          let key      = CryptoJS.enc.Utf8.parse(SECRET);
          let msgUtf8  = CryptoJS.enc.Utf8.parse(jsonData);
          let hmacCalc = CryptoJS.HmacSHA256(msgUtf8, key).toString(CryptoJS.enc.Hex);

          if (hmacCalc === recvHmac) {
            // Valid → prepare for InfluxDB
            msg.payload = {
              measurement: 'esp32_sensor',
              tags: { device: 'esp32_01' },
              fields: {
                tempC: dataObj.tempC,
                bpm: dataObj.bpm,
                avg_bpm: dataObj.avg_bpm
              },
              timestamp: Date.now()
            };
            return [ msg, null ];  // output[0]=valid, output[1]=invalid
          } else {
            node.warn(`HMAC mismatch: recv=${recvHmac} calc=${hmacCalc}`);
            return [ null, msg ];
          }
        } catch (e) {
          node.error("HMAC verify error: " + e);
          return [ null, msg ];
        }
        ```

        * Set **Outputs** to 2. Wire output 1 → InfluxDB Out, output 2 → Debug.
     3. **InfluxDB Out** node:

        * URL: `http://localhost:8086`
        * Organization: your InfluxDB org (e.g. `my-org`)
        * Bucket: `esp32_sensors`
        * Token: your InfluxDB write token (see next section)
     4. **Debug** node on the second output of “Verify HMAC” to log invalid messages.

4. Deploy the flow. Incoming messages with correct HMAC proceed to InfluxDB; invalid ones get logged.

### InfluxDB Configuration

1. Install InfluxDB 2.x (or use Docker):

   ```bash
   # On Linux:
   wget https://dl.influxdata.com/influxdb/releases/influxdb2-2.8.0-linux-amd64.tar.gz
   tar xvf influxdb2-2.8.0-linux-amd64.tar.gz
   sudo cp influxdb2-2.8.0-linux-amd64/influxd /usr/local/bin/
   influxd
   ```
2. Open [http://localhost:8086](http://localhost:8086) in your browser.
3. On first run, create:

   * **Username** (admin) / **Password** (adminpassword)
   * **Organization** (e.g. `my-org`)
   * **Bucket** (e.g. `esp32_sensors`)
   * **Token** (copy this, e.g. `my-influxdb-token`)
4. In Node-RED’s InfluxDB Out node, paste the **Token**, set Organization to `my-org` and Bucket to `esp32_sensors`.

---

## Putting It All Together

1. **Start MQTT Broker**

   ```bash
   sudo systemctl start mosquitto
   ```

2. **Start InfluxDB** (if using standalone)

   ```bash
   influxd
   ```

   or if using Docker:

   ```bash
   docker run -d --name influxdb \
     -p 8086:8086 \
     -e DOCKER_INFLUXDB_INIT_MODE=setup \
     -e DOCKER_INFLUXDB_INIT_USERNAME=admin \
     -e DOCKER_INFLUXDB_INIT_PASSWORD=adminpassword \
     -e DOCKER_INFLUXDB_INIT_ORG=my-org \
     -e DOCKER_INFLUXDB_INIT_BUCKET=esp32_sensors \
     -e DOCKER_INFLUXDB_INIT_AUTH_TOKEN=my-influxdb-token \
     influxdb:2.8
   ```

3. **Launch Node-RED**

   ```bash
   node-red
   ```

   * Install required nodes, import/configure the HMAC verification flow, and deploy.

4. **Flash & Run ESP32**

   * Open `esp32/HMAC_Publisher.ino` in Arduino IDE.
   * Update Wi-Fi SSID/PASSWORD, MQTT broker IP, HMAC key (must match Node-RED).
   * Upload to ESP32.
   * Open Serial Monitor at 115200 baud to observe publishing.

5. **Verify Data in InfluxDB**

   * In Node-RED’s Debug sidebar, invalid HMAC messages (if any) will appear.
   * In InfluxDB UI → **Explore** → select bucket `esp32_sensors` → run:

     ```flux
     from(bucket: "esp32_sensors")
       |> range(start: -1h)
       |> filter(fn: (r) => r._measurement == "esp32_sensor")
       |> sort(columns: ["_time"], desc: false)
     ```
   * You should see time‐series points with `tempC`, `bpm`, and `avg_bpm` every \~2 seconds.

---

## Troubleshooting

* **ESP32 won’t connect to Wi-Fi**

  * Verify SSID/password.
  * Ensure ESP32 is in Wi-Fi range.
  * Check Serial Monitor for status messages.

* **MQTT publish fails**

  * Confirm your broker’s IP/port in `HMAC_Publisher.ino`.
  * Ensure Mosquitto is running: `sudo systemctl status mosquitto`.
  * Look for `Publish failed!` messages in Serial Monitor.

* **HMAC never validates in Node-RED**

  * Confirm that the JSON string used for HMAC matches exactly in both client and server:

    * Client: `{"tempC":36.52,"bpm":72.3,"avg_bpm":70}`
    * Server: `JSON.stringify({ tempC:…, bpm:…, avg_bpm:… })`
    * Even an extra space or different key order breaks HMAC.
  * Ensure `HMAC_KEY` in the sketch is identical (case‐sensitive) to `SECRET` in the Function node.

* **InfluxDB Out node errors (401/403)**

  * Verify the token in Node-RED matches the write token generated for `esp32_sensors`.
  * Check InfluxDB logs (`influxd` output) for authentication errors.

* **Time synchronization**

  * ESP32 uses `millis()/1000` for timestamp approximation.
  * If accurate Unix time is required, integrate an NTP client (e.g. `WiFiUDP` + `NTPClient`) and replace `Date.now()` or `millis()/1000` accordingly.

---

## License

This project is released under the MIT License. Feel free to use, modify, and redistribute for private or commercial purposes.

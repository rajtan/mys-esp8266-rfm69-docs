# MySensors ESP8266 RFM69 System Documentation

Complete documentation for the multi-repository MySensors IoT system using ESP8266 and RFM69 radio. 

---

## 📦 **System Repositories**

| Repository | Purpose | Link |
|------------|---------|------|
| 🌐 **Gateway** | RFM69 ↔ MQTT bridge | [mys-esp8266-rfm69-mqtt-gw](https://github.com/rajtan/mys-esp8266-rfm69-mqtt-gw) |
| 📡 **Nodes** | Sensor node firmware | [mys-esp8266-rfm69-nodes](https://github.com/rajtan/mys-esp8266-rfm69-nodes) |
| 📚 **Documentation** | This repository | [mys-esp8266-rfm69-docs](https://github.com/rajtan/mys-esp8266-rfm69-docs) |

---

## 🏗️ **System Architecture**

```
┌─────────────────┐   RFM69 Radio    ┌─────────────┐   MQTT/WiFi   ┌──────────────────┐
│  Sensor Nodes   │ ◄─────────────► │   Gateway   │ ◄────────────► │ Home Automation  │
│   (ESP8266)     │   868 MHz        │  (ESP8266)  │                │  (HA/Node-RED)   │
│  - Contact      │                  │  Node ID:  0 │                └──────────────────┘
│  - Temp/RH      │                  │             │
│  - Motion       │                  └─────────────┘
│  - (Custom)     │
└─────────────────┘
```

**Data Flow:**
1. Sensor node reads data (temperature, motion, etc.)
2. Node sends MySensors packet via RFM69 radio
3. Gateway receives, parses packet
4. Gateway publishes to MQTT broker
5. Home automation system subscribes to MQTT topics

---

## 📡 **Protocol Overview**

### **MySensors Packet Structure**

8-byte header + up to 61 bytes payload: 

```cpp
struct MySensorsPacket {
    uint8_t sender;      // Node ID (1-254, gateway=0)
    uint8_t last;        // Last hop (for routing/mesh)
    uint8_t next;        // Destination node ID
    uint8_t length;      // Payload length (0-61 bytes)
    uint8_t command;     // Command type (2 bits, packed in byte 4)
    uint8_t ack;         // ACK request flag (1 bit, packed in byte 4)
    uint8_t msgType;     // Message type (5 bits, packed in byte 4)
    uint8_t childId;     // Child sensor ID on the node
    uint16_t sensorId;   // Sensor type identifier
    uint8_t payload[61]; // Actual data (text or binary)
};
```

**See:** [`protocol/packet-format.md`](protocol/packet-format.md) for detailed breakdown. 

### **MQTT Topic Format**

**Outgoing (Nodes → MQTT):**
```
mys-out/<nodeId>/<childId>/<command>/<ack>/<type>
```

**Incoming (MQTT → Nodes):**
```
mys-in/<nodeId>/<childId>/<command>/<ack>/<type>
```

**Examples:**
- `mys-out/2/0/1/0/16` → `"1"` (Contact sensor node 2, open)
- `mys-out/3/0/1/0/0` → `"22. 5"` (Temp sensor node 3, 22.5°C)
- `mys-out/4/0/1/0/2` → `"1"` (Motion sensor node 4, detected)

**See:** [`protocol/mqtt-topics.md`](protocol/mqtt-topics.md) for complete reference.

---

## 🔧 **Development Guide**

<!-- SECTION: GATEWAY -->
### 🌐 **Gateway Development**

**Repository:** [mys-esp8266-rfm69-mqtt-gw](https://github.com/rajtan/mys-esp8266-rfm69-mqtt-gw)

**Purpose:** Bridge between RFM69 radio and MQTT broker. 

#### **Key Responsibilities:**
- Listen for RFM69 packets from nodes
- Parse MySensors protocol
- Publish sensor data to MQTT
- Forward MQTT commands to nodes via radio
- Provide web configuration interface

#### **Important Files:**
- `src/radio.cpp` - RFM69 communication, packet parsing
- `src/mqtt.cpp` - MQTT bridge logic, topic handling
- `include/settings.h` - Protocol constants, MySensors definitions
- `src/web_config.cpp` - Web-based configuration UI

#### **Building:**
```bash
cd mys-esp8266-rfm69-mqtt-gw
platformio run -e esp12e          # Build
platformio run -e esp12e -t upload # Upload
```

#### **Configuration:**
- **Node ID:** Always `0` (gateway identifier)
- **Network ID:** Default `100` (must match all nodes)
- **Frequency:** Default `868 MHz` (RF69_868MHZ)
- **MQTT:** Configured via web interface on first boot

#### **Critical Design Decisions:**
- ✅ Uses RFM69 **hardware ACK** (`radio.sendWithRetry()` with 3 retries, 100ms timeout)
- ❌ **No application-level ACK tracking** (hardware layer is sufficient)
- ❌ **No message deduplication** (duplicates are extremely rare)
- 📝 May add sequence tracking in future if needed

#### **Key Functions:**
```cpp
// Receive from nodes
void radioReceive() {
    if (! radio.receiveDone()) return;
    // Parse MySensorsPacket
    // Forward to publishRadioMessage()
}

// Send to nodes
void radioSend(uint8_t toNodeId, uint8_t command, uint8_t ack, 
               uint8_t type, const char *payload) {
    // Build MySensorsPacket
    // Call radio.sendWithRetry(toNodeId, buf, len, 3, 100)
}
```

<!-- END SECTION:  GATEWAY -->

---

<!-- SECTION: NODES -->
### 📡 **Node Development**

**Repository:** [mys-esp8266-rfm69-nodes](https://github.com/rajtan/mys-esp8266-rfm69-nodes)

**Purpose:** Battery-powered sensor nodes communicating via RFM69.

#### **Existing Node Types:**

| Node Type | Node ID | Sensors | Description |
|-----------|---------|---------|-------------|
| Contact Sensor | 2 | Door/window switch | Reports open/closed status |
| Temp/Humidity | 3 | DHT22 | Temperature + humidity readings |
| Motion Sensor | 4 | PIR (HC-SR501) | Detects motion |

#### **Building Specific Node:**
```bash
cd mys-esp8266-rfm69-nodes
platformio run -e node-contact     # Contact sensor
platformio run -e node-temp-rh     # Temperature/humidity
platformio run -e node-motion      # Motion sensor
```

#### **Adding New Node Type:**

1. **Create directory:**
   ```bash
   mkdir -p my-sensor/src
   ```

2. **Create Arduino sketch:**  `my-sensor/src/my-sensor.ino`
   ```cpp
   #define MY_RADIO_RF69
   #define MY_RF69_FREQUENCY RFM69_868MHZ
   #define MY_IS_RFM69HW
   #include <MySensors.h>
   
   #define CHILD_ID 0
   MyMessage msg(CHILD_ID, V_STATUS);
   
   void presentation() {
       sendSketchInfo("My Sensor", "1.0");
       present(CHILD_ID, S_BINARY);
   }
   
   void loop() {
       // Read sensor, send data
       send(msg. set(value));
       sleep(60000); // Sleep 60 seconds
   }
   ```

3. **Add to `platformio.ini`:**
   ```ini
   [env:node-mysensor]
   src_dir = my-sensor
   build_flags =
       -D MY_NODE_ID=5                    # Unique ID! 
       -D MY_RF69_NETWORKID=100           # Must match gateway
       # ... (copy other flags from existing nodes)
   ```

#### **Configuration Requirements:**
- ✅ **Node ID:** Must be unique (1-254, avoid 0)
- ✅ **Network ID:** Must match gateway (default:  100)
- ✅ **Frequency:** Must match gateway (868 MHz)
- ✅ **Encryption Key:** If gateway uses encryption, must match exactly

#### **MySensors Library Usage:**
```cpp
// Presenting sensors (in presentation())
present(childId, sensorType);

// Sending data
send(message. set(value));

// Requesting data from gateway
request(childId, valueType);

// Sleeping (power saving)
sleep(milliseconds);
```

<!-- END SECTION: NODES -->

---

## 🔒 **Design Decisions & Rationale**

### **1. No Application-Level ACK (Current)**

**Decision:** Rely solely on RFM69 hardware ACK mechanism.

**Rationale:**
- RFM69 chip has built-in ACK at hardware level
- `radio.sendWithRetry()` provides 3 retry attempts with 100ms timeout
- This is sufficient for reliable delivery in normal RF environments
- Simpler codebase, less overhead
- Application-level ACK would be redundant

**Future Consideration:** Add only if message loss observed in production.

---

### **2. No Message Deduplication (Current)**

**Decision:** Process all received messages without deduplication.

**Rationale:**
- With hardware ACK, duplicate messages are extremely rare
- Only occurs if gateway receives message but its ACK is lost in RF noise
- Node would then retry, causing duplicate
- In practice, this happens < 0.1% of the time
- Added complexity not justified

**Future Consideration:** Add sequence number tracking if duplicates become problematic.

---

### **3. No Sequence Number Tracking (Current)**

**Decision:** No message sequence numbers or gap detection.

**Rationale:**
- MySensors packet has `last` and `next` fields but we don't use them for sequencing
- Hardware retry mechanism makes message loss very rare
- Sequence tracking adds state management complexity
- Current use case doesn't require guaranteed delivery audit trail

**Future Consideration:** Implement if: 
- Critical sensors require message delivery proof
- Compliance/audit requirements arise
- Debugging shows message gaps in production

---

## 🧪 **Testing & Validation**

### **Integration Test Procedure:**

1. **Start MQTT Broker:**
   ```bash
   # Using Mosquitto
   mosquitto -v
   ```

2. **Configure Gateway:**
   - Power on gateway (enters config mode on first boot)
   - Connect to WiFi AP: `MPSHUB-GW`
   - Navigate to: `192.168.4.1`
   - Configure WiFi, MQTT broker, Network ID

3. **Deploy Node:**
   - Flash node firmware with matching Network ID
   - Power on node
   - Check serial output for connection messages

4. **Verify MQTT Messages:**
   ```bash
   # Subscribe to all topics
   mosquitto_sub -h localhost -t "mys-out/#" -v
   ```

5. **Expected Output:**
   ```
   mys-out/2/0/1/0/16 1        # Contact open
   mys-out/3/0/1/0/0 22.5      # Temp reading
   mys-out/3/1/1/0/1 65        # Humidity reading
   mys-out/4/0/1/0/2 1         # Motion detected
   ```

### **Common Issues:**

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| No MQTT messages | Network ID mismatch | Verify all nodes use Network ID 100 |
| Intermittent communication | Weak signal | Check antenna, reduce distance |
| No radio communication | Frequency mismatch | Verify all use 868 MHz |
| Garbled data | Encryption mismatch | Ensure key matches or is disabled |

---

## 📊 **Version Compatibility Matrix**

| Gateway Version | Nodes Version | Protocol Version | Breaking Changes |
|-----------------|---------------|------------------|------------------|
| v1.0.x | v1.0.x | 1.0 | Initial release |
| v1.1.x | v1.0.x+ | 1.0 | None (backward compatible) |

**Protocol v1.0 (Current):**
- MySensors-compatible packet structure
- RFM69 hardware ACK only
- No sequence tracking
- No deduplication

**Protocol v2.0 (Future):**
- May add sequence numbers
- May add deduplication logic
- Will maintain backward compatibility with v1.0 packets

---

## 📚 **Additional Resources**

### **Detailed Documentation:**
- [Packet Format Specification](protocol/packet-format. md) - Byte-by-byte breakdown
- [MQTT Topics Reference](protocol/mqtt-topics.md) - Complete topic structure
- [Message Examples](examples/message-samples.md) - Sample packets & payloads

### **External References:**
- [MySensors Protocol Documentation](https://www.mysensors.org/download/serial_api_20)
- [RFM69 Library (LowPowerLab)](https://github.com/LowPowerLab/RFM69)
- [ESP8266 Arduino Core](https://github.com/esp8266/Arduino)

---

## 🤝 **Contributing**

### **Making Protocol Changes:**

1. **Propose changes** via GitHub Issue in this docs repo
2. **Update documentation** in this repo first
3. **Increment protocol version** if breaking
4. **Implement in gateway** repo
5. **Implement in nodes** repo
6. **Tag releases** in both repos:  `v2.0.0`
7. **Update compatibility matrix** above

### **Adding New Node Types:**

1. Create new directory in nodes repo
2. Document in nodes repo README
3. Add example to `examples/message-samples.md`
4. Assign unique Node ID
5. Test integration with gateway

---

## 📝 **Changelog**

### **v1.0.0** (2026-01-06)
- Initial documentation repository
- Documented protocol v1.0
- Created composite README and Copilot instructions
- Established multi-repo structure

---

## 📞 **Support**

For issues or questions: 
1. Check [troubleshooting section](#-testing--validation) above
2. Review detailed protocol docs in `protocol/` directory
3. Check gateway and nodes repo READMEs
4. Open GitHub Issue in relevant repository

---

## 📄 **License**

This documentation is provided as-is for the MySensors ESP8266 RFM69 system.

---

**Last Updated:** 2026-01-06  
**Documentation Version:** 1.0.0  
**Protocol Version:** 1.0
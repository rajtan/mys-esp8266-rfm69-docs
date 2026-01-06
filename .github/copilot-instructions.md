# GitHub Copilot Instructions - MySensors ESP8266 RFM69 System

This file provides AI coding assistant context for **all repositories** in the MySensors ESP8266 RFM69 system.

---

## 📦 **Multi-Repository Context**

This system spans three repositories:

| Repository | Purpose | When to Focus Here |
|------------|---------|-------------------|
| `mys-esp8266-rfm69-mqtt-gw` | Gateway (Radio ↔ MQTT) | Working on bridging, MQTT, web config |
| `mys-esp8266-rfm69-nodes` | Sensor Nodes | Working on sensor reading, node firmware |
| `mys-esp8266-rfm69-docs` | Documentation | Writing docs, updating protocol specs |

**Always consider:** Changes in one repo may affect others (protocol compatibility).

---

## 🔗 **Related Repositories**

When generating code or suggestions, be aware of: 
- **Gateway:** https://github.com/rajtan/mys-esp8266-rfm69-mqtt-gw
- **Nodes:** https://github.com/rajtan/mys-esp8266-rfm69-nodes
- **Docs:** https://github.com/rajtan/mys-esp8266-rfm69-docs

---

## 📡 **Core Protocol (APPLIES TO ALL REPOS)**

### **MySensors Packet Structure**

This is the **shared data format** between gateway and nodes:

```cpp
struct MySensorsPacket {
    uint8_t sender;      // Byte 0: Node ID (1-254, gateway=0)
    uint8_t last;        // Byte 1: Last hop (routing, currently unused)
    uint8_t next;        // Byte 2: Destination node ID
    uint8_t length;      // Byte 3: Payload length (0-61 bytes)
    uint8_t command;     // Byte 4 bits 6-7: Command type (0-3)
    uint8_t ack;         // Byte 4 bit 5: ACK request (0 or 1)
    uint8_t msgType;     // Byte 4 bits 0-4: Message type (0-31)
    uint8_t childId;     // Byte 5:  Child sensor ID
    uint16_t sensorId;   // Bytes 6-7: Sensor type (big-endian)
    uint8_t payload[61]; // Bytes 8+: Data (text or binary)
};
```

**Byte 4 Encoding:**
```
Bits:  7 6 | 5 | 4 3 2 1 0
      cmd  ack   msgType
```

Example: 
```cpp
uint8_t byte4 = ((command & 0x3) << 6) | ((ack & 0x1) << 5) | (msgType & 0x1F);
```

---

### **Protocol Constants (SHARED ACROSS ALL REPOS)**

```cpp
// Network Configuration
#define DEFAULT_NETWORK_ID 100
#define DEFAULT_FREQUENCY RF69_868MHZ  // 868 MHz
#define GATEWAY_NODE_ID 0

// Command Types
#define MYSENSORS_CMD_PRESENTATION 0  // Node/sensor introduction
#define MYSENSORS_CMD_SET 1           // Set value (sensor → gateway)
#define MYSENSORS_CMD_REQ 2           // Request value
#define MYSENSORS_CMD_INTERNAL 3      // Internal messages
#define MYSENSORS_CMD_STREAM 4        // Streaming data

// Sensor Types (Examples - not exhaustive)
#define MYSENSORS_S_DOOR 0            // Door/window sensor
#define MYSENSORS_S_MOTION 1          // Motion sensor
#define MYSENSORS_S_SMOKE 2           // Smoke sensor
#define MYSENSORS_S_TEMP 6            // Temperature sensor
#define MYSENSORS_S_HUM 7             // Humidity sensor

// Value Types
#define MYSENSORS_V_STATUS 2          // Binary status (0/1)
#define MYSENSORS_V_TEMP 0            // Temperature value
#define MYSENSORS_V_HUM 1             // Humidity value
```

**Note:** Full constant definitions are in `include/settings.h` (gateway repo).

---

### **MQTT Topic Structure (SHARED)**

**Outgoing (Nodes → MQTT via Gateway):**
```
mys-out/<nodeId>/<childId>/<command>/<ack>/<type>
```

**Incoming (MQTT → Nodes via Gateway):**
```
mys-in/<nodeId>/<childId>/<command>/<ack>/<type>
```

**Example Topics:**
```
mys-out/2/0/1/0/16       # Contact sensor (node 2, child 0, SET cmd, type 16=V_STATUS)
mys-out/3/0/1/0/0        # Temperature (node 3, child 0, SET cmd, type 0=V_TEMP)
mys-out/3/1/1/0/1        # Humidity (node 3, child 1, SET cmd, type 1=V_HUM)
mys-out/4/0/1/0/2        # Motion (node 4, child 0, SET cmd, type 2=V_STATUS)
```

**Payload:** Sensor value as string (e.g., `"22.5"`, `"1"`, `"65"`)

---

## 🌐 **GATEWAY-SPECIFIC CONTEXT**

**Active when working in:** `mys-esp8266-rfm69-mqtt-gw` repository

---

### **Gateway Role & Responsibilities:**
- Receive RFM69 packets from nodes
- Parse MySensors protocol
- Publish sensor data to MQTT broker
- Subscribe to MQTT commands
- Forward MQTT commands to nodes via radio
- Provide web-based configuration interface

---

### **Key Design Principles (GATEWAY):**

1. **Gateway is ALWAYS Node ID 0**
   - Never change this
   - Nodes send to destination ID 0

2. **Use Hardware ACK Only**
   - Call `radio.sendWithRetry(nodeId, buf, len, 3, 100)` for sending
   - Parameters: 3 retries, 100ms ACK timeout
   - RFM69 chip handles ACK automatically
   - **DO NOT** implement application-level ACK

3. **No Message Deduplication**
   - Currently not implemented
   - Duplicates are extremely rare with hardware ACK
   - May add in future (protocol v2.0) if needed

4. **No Sequence Tracking**
   - `last` and `next` fields are parsed but not used for gap detection
   - May add in future for critical applications

---

### **Gateway Code Locations:**

| File | Purpose |
|------|---------|
| `src/radio.cpp` | RFM69 communication, packet parsing |
| `src/mqtt.cpp` | MQTT bridge, topic handling |
| `include/settings.h` | Protocol constants, MySensors definitions |
| `src/web_config.cpp` | Web configuration UI |
| `src/config. cpp` | EEPROM configuration persistence |
| `src/main.cpp` | Main loop, initialization |

---

### **Critical Gateway Functions:**

```cpp
// Radio Receive (called in loop)
void radioReceive() {
    if (!radio.receiveDone()) return;  // Check for packet
    
    // Parse buffer into MySensorsPacket struct
    MySensorsPacket pkt;
    pkt.sender = radio.DATA[0];
    pkt.last = radio.DATA[1];
    // ... (extract remaining fields)
    
    // Forward to MQTT
    publishRadioMessage(pkt. sender, pkt.childId, pkt.command, 
                       pkt.ack, pkt.msgType, payloadStr);
}

// Radio Send (called from MQTT callback)
void radioSend(uint8_t toNodeId, uint8_t command, uint8_t ack, 
               uint8_t type, const char *payload) {
    // Build MySensors packet
    uint8_t buf[66];
    buf[0] = activeConfig.nodeId;  // Gateway = 0
    buf[1] = activeConfig.nodeId;  // Last = 0
    buf[2] = toNodeId;             // Destination
    buf[3] = payloadLen;
    buf[4] = ((command & 0x3) << 6) | ((ack & 0x1) << 5) | (type & 0x1F);
    // ... (fill remaining fields)
    
    // Send with hardware ACK retry
    if (radio.sendWithRetry(toNodeId, buf, totalLen, 3, 100)) {
        debugLog("Sent OK");
    } else {
        debugLog("Send FAILED after 3 retries");
    }
}

// MQTT → Radio Bridge
void onMqttMessage(char *topic, byte *payload, unsigned int length) {
    // Parse topic:  mys-in/<nodeId>/<childId>/<cmd>/<ack>/<type>
    uint8_t nodeId, childId, command, ack, type;
    parseMqttTopic(topic, nodeId, childId, command, ack, type);
    
    // Forward to radio
    radioSend(nodeId, command, ack, type, payloadStr);
}

// Radio → MQTT Bridge
void publishRadioMessage(uint8_t fromNode, uint8_t childId, uint8_t command,
                        uint8_t ack, uint8_t type, const char *payload) {
    // Build topic: mys-out/<nodeId>/<childId>/<cmd>/<ack>/<type>
    String topic = mqttPrefixOut + String(fromNode) + "/" + 
                   String(childId) + "/" + String(command) + "/" + 
                   String(ack) + "/" + String(type);
    
    mqttClient.publish(topic.c_str(), payload);
}
```

---

### **When Generating Gateway Code:**

✅ **DO:**
- Include:  `#include "../include/settings.h"`
- Use existing functions: `radioSend()`, `publishRadioMessage()`
- Add debug logging: `debugLog("message")`
- Use `activeConfig.nodeId` (always 0)
- Call `radio.sendWithRetry()` for sending
- Follow camelCase naming convention

❌ **DO NOT:**
- Change gateway Node ID (must be 0)
- Implement application-level ACK (hardware ACK is sufficient)
- Add message deduplication (not in current design)
- Change packet structure
- Modify MQTT topic format

---

## 📡 **NODE-SPECIFIC CONTEXT**

**Active when working in:** `mys-esp8266-rfm69-nodes` repository

---

### **Node Role & Responsibilities:**
- Read sensor data (temperature, motion, contact, etc.)
- Build MySensors packets
- Send data to gateway (Node ID 0) via RFM69
- Receive commands from gateway
- Sleep between readings (power saving)

---

### **Key Design Principles (NODES):**

1. **Each Node Has Unique ID**
   - Must be 1-254 (0 is gateway)
   - Defined in `platformio.ini`: `-D MY_NODE_ID=2`
   - Never duplicate IDs on same network

2. **Network ID Must Match Gateway**
   - Default: 100
   - Defined in `platformio.ini`: `-D MY_RF69_NETWORKID=100`
   - All nodes + gateway must use same Network ID

3. **Frequency Must Match Gateway**
   - Default: 868 MHz
   - Defined in `platformio.ini`: `-D MY_RF69_FREQUENCY=RFM69_868MHZ`

4. **Use MySensors Library**
   - Don't manually build packets (library handles it)
   - Use `send()`, `present()`, `request()` functions

---

### **Existing Node Types:**

| Node Type | Node ID | Sensor | Child IDs |
|-----------|---------|--------|-----------|
| Contact Sensor | 2 | Door/window switch | 0 (status) |
| Temp/Humidity | 3 | DHT22 | 0 (temp), 1 (humidity) |
| Motion Sensor | 4 | PIR (HC-SR501) | 0 (motion status) |

---

### **Node Code Structure:**

```cpp
#define MY_RADIO_RF69                           // Use RFM69 radio
#define MY_RF69_FREQUENCY RFM69_868MHZ          // 868 MHz
#define MY_IS_RFM69HW                           // High-power variant
#include <MySensors.h>

// Define child sensor IDs
#define CHILD_ID_SENSOR 0

// Create message objects
MyMessage msgStatus(CHILD_ID_SENSOR, V_STATUS);

// Presentation (called once at startup)
void presentation() {
    sendSketchInfo("Contact Sensor", "1.0");
    present(CHILD_ID_SENSOR, S_DOOR);  // Present as door sensor
}

// Setup (called once)
void setup() {
    pinMode(SENSOR_PIN, INPUT_PULLUP);
}

// Main loop
void loop() {
    int value = digitalRead(SENSOR_PIN);
    send(msgStatus.set(value));  // Send to gateway
    sleep(60000);  // Sleep 60 seconds
}

// Receive commands from gateway (optional)
void receive(const MyMessage &message) {
    if (message.type == V_STATUS) {
        // Handle incoming command
    }
}
```

---

### **platformio.ini Configuration:**

```ini
[env:node-contact]
platform = espressif8266@2.6.2
board = esp12e
framework = arduino
src_dir = contact-sensor
lib_deps = 
    mysensors/MySensors@^2.3.2
    adafruit/DHT sensor library@^1.4.3  # If using DHT
build_flags = 
    -D MY_RADIO_RF69                    # Use RFM69
    -D MY_NODE_ID=2                     # UNIQUE NODE ID
    -D MY_RF69_NETWORKID=100            # MUST MATCH GATEWAY
    -D MY_RF69_FREQUENCY=RFM69_868MHZ   # MUST MATCH GATEWAY
    -D MY_IS_RFM69HW                    # High-power variant
    -D MY_RFM69_TX_POWER_DBM=20         # 20 dBm (max)
```

---

### **When Generating Node Code:**

✅ **DO:**
- Define `MY_RADIO_RF69` before including MySensors
- Use unique Node ID (check existing nodes)
- Use same Network ID as gateway (100)
- Call `present()` in `presentation()`
- Call `send()` to transmit data
- Use `sleep()` for power saving
- Include sensor libraries (DHT, etc.)

❌ **DO NOT:**
- Use Node ID 0 (reserved for gateway)
- Duplicate Node IDs
- Change Network ID without updating gateway
- Manually build MySensors packets (library does this)
- Use `delay()` instead of `sleep()` (wastes power)

---

### **Adding New Node Type:**

1. **Create directory:** `new-sensor/src/`
2. **Create sketch:** `new-sensor/src/new-sensor.ino`
3. **Add environment** to root `platformio.ini`:
   ```ini
   [env:node-newsensor]
   src_dir = new-sensor
   build_flags = 
       -D MY_NODE_ID=5  # ASSIGN NEW UNIQUE ID
       # ... (copy other flags from existing nodes)
   ```
4. **Test and document**

---

## 🚫 **BREAKING CHANGES - DO NOT MODIFY (ALL REPOS)**

These are **system-wide constants** that MUST remain unchanged unless coordinating a major protocol version upgrade: 

❌ **Never Change:**
- MySensorsPacket structure (field order, sizes)
- MQTT topic format (`mys-out/...` and `mys-in/...`)
- Command type values (0-4)
- Sensor type constants
- Value type constants
- Default Network ID (100)
- Default Frequency (868 MHz)
- Gateway Node ID (0)

---

## ✅ **SAFE TO MODIFY (CONTEXT-DEPENDENT)**

### **Gateway - Safe Changes:**
- ✅ MQTT connection/reconnection logic
- ✅ Web configuration UI
- ✅ WiFi reconnection handling
- ✅ Debug logging verbosity
- ✅ Radio power levels
- ✅ EEPROM storage format (with migration)

### **Nodes - Safe Changes:**
- ✅ Sensor reading logic
- ✅ Sleep durations
- ✅ Sensor thresholds (when to send updates)
- ✅ Debounce timing
- ✅ Child sensor IDs (per node)
- ✅ Adding new node types (with unique IDs)

### **Documentation - Safe Changes:**
- ✅ Examples and tutorials
- ✅ Troubleshooting guides
- ✅ Architecture diagrams
- ✅ Code comments and explanations

---

## 🧪 **Testing Considerations (ALL REPOS)**

When suggesting test code or validation:

1. **Network ID Matching:**
   - Verify all nodes use Network ID 100
   - Verify gateway uses Network ID 100

2. **Frequency Matching:**
   - Verify all devices use 868 MHz (or same frequency)

3. **MQTT Topics:**
   - Check format: `mys-out/<nodeId>/<childId>/<cmd>/<ack>/<type>`
   - Validate all fields are numeric

4. **Packet Validation:**
   - Sender ID should be 1-254 (not 0, that's gateway)
   - Destination (next) should be 0 (gateway)
   - Command should be 0-4
   - Payload length should be ≤ 61 bytes

5. **Real-World Testing:**
   - Test with actual MQTT broker (Mosquitto recommended)
   - Test RF range in deployment environment
   - Monitor for message loss or duplicates

---

## 💡 **Code Generation Guidelines**

### **When Writing Gateway Code:**

**Include headers:**
```cpp
#include "../include/settings.h"
#include <RFM69.h>
#include <PubSubClient.h>
```

**Use existing globals:**
```cpp
extern GatewayConfig activeConfig;  // Configuration
extern RFM69 radio;                 // Radio instance
extern PubSubClient mqttClient;     // MQTT client
```

**Function style:**
```cpp
void myFunction() {
    debugLog("Starting function");  // Use debug logging
    
    if (radio.sendWithRetry(nodeId, buf, len, 3, 100)) {
        debugLog("Success");
    } else {
        debugLog("Failed");
    }
}
```

---

### **When Writing Node Code:**

**Standard header:**
```cpp
#define MY_RADIO_RF69
#define MY_RF69_FREQUENCY RFM69_868MHZ
#define MY_IS_RFM69HW
#include <MySensors.h>

// Sensor-specific includes
#include <DHT.h>  // If using DHT sensor
```

**Message declaration:**
```cpp
#define CHILD_ID 0
MyMessage msg(CHILD_ID, V_TEMP);  // Temperature message
```

**Standard functions:**
```cpp
void presentation() {
    sendSketchInfo("My Sensor", "1.0");
    present(CHILD_ID, S_TEMP);
}

void loop() {
    float temp = readSensor();
    send(msg.set(temp, 1));  // Send with 1 decimal place
    sleep(60000);            // Sleep 60 seconds
}
```

---

## 📚 **Documentation References**

When unsure about protocol details, refer to: 
- **Packet format:** `protocol/packet-format.md` (docs repo)
- **MQTT topics:** `protocol/mqtt-topics.md` (docs repo)
- **System README:** `README.md` (docs repo)
- **Gateway constants:** `include/settings.h` (gateway repo)

---

## 🔄 **Protocol Version Compatibility**

**Current:** Protocol v1.0

**Future Changes:**
- Protocol v2.0 may add sequence number tracking
- Protocol v2.0 may add message deduplication
- Will maintain backward compatibility where possible

When suggesting features requiring protocol changes:
1. Note that it requires protocol version bump
2. Consider backward compatibility
3. Document impact on both gateway and nodes
4. Suggest migration path

---

## 📝 **Example Scenarios**

### **Scenario:  User asks to add temperature sensor support**

**Context:** This is a NODE task. 

**Response should:**
- ✅ Suggest creating new node type in nodes repo
- ✅ Use MySensors library (`S_TEMP`, `V_TEMP`)
- ✅ Assign unique Node ID (check existing:  2, 3, 4 are taken)
- ✅ Include DHT library for DHT22 sensor
- ✅ Show `present()` and `send()` usage
- ❌ Don't modify gateway code (already supports all sensor types)

---

### **Scenario: User reports duplicate MQTT messages**

**Context:** This affects GATEWAY. 

**Response should:**
- ✅ Acknowledge this is rare with current design
- ✅ Explain hardware ACK mechanism
- ✅ Suggest monitoring to confirm frequency
- ✅ Propose sequence number tracking as solution (protocol v2.0)
- ✅ Note this requires changes to both gateway and nodes
- ❌ Don't immediately implement (verify problem first)

---

### **Scenario: User wants to add encryption**

**Context:** Affects BOTH gateway and nodes.

**Response should:**
- ✅ Note gateway already supports encryption (web config)
- ✅ Explain nodes need matching 16-byte key
- ✅ Show how to add `-D MY_RF69_ENCRYPTION_KEY="..."` to node build flags
- ✅ Warn that all devices must use same key
- ❌ Don't change packet structure (encryption is at radio layer)

---

## 🎯 **Summary for Copilot**

When assisting with this codebase:

1. **Identify repository context** (gateway, nodes, or docs)
2. **Respect protocol** (don't break MySensors packet format)
3. **Maintain compatibility** (gateway and nodes must stay in sync)
4. **Follow design decisions** (hardware ACK only, no deduplication yet)
5. **Use existing patterns** (don't reinvent MySensors protocol)
6. **Consider both repos** (changes may affect gateway AND nodes)

**When in doubt:** Suggest reviewing `README.md` in docs repo for system-wide context.

---

**Last Updated:** 2026-01-06  
**Instructions Version:** 1.0.0  
**Protocol Version:** 1.0
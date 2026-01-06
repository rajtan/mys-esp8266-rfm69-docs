# MySensors Packet Format Specification

Detailed byte-by-byte breakdown of the MySensors protocol used in this system.

---

## Packet Structure

**Total Size:** 8 bytes (header) + 0-61 bytes (payload) = 8-69 bytes

```
Offset  | Size | Field      | Description
--------|------|------------|------------------------------------------
0       | 1    | sender     | Source node ID (1-254, 0=gateway)
1       | 1    | last       | Last hop node ID (routing, currently unused)
2       | 1    | next       | Destination node ID (0=gateway for sensors)
3       | 1    | length     | Payload length in bytes (0-61)
4       | 1    | flags      | Combined:  command(2) + ack(1) + msgType(5)
5       | 1    | childId    | Child sensor ID on node (0-255)
6-7     | 2    | sensorId   | Sensor type identifier (big-endian uint16)
8+      | 0-61 | payload    | Sensor data (text or binary)
```

---

## Byte 4: Flags Field Encoding

Byte 4 packs three fields into a single byte:

```
Bit:      7   6   5   4   3   2   1   0
Field:   |  cmd  | ack |    msgType      |
```

**Extraction:**
```cpp
uint8_t command = (byte4 >> 6) & 0x3;   // Bits 6-7
uint8_t ack     = (byte4 >> 5) & 0x1;   // Bit 5
uint8_t msgType = byte4 & 0x1F;         // Bits 0-4
```

**Construction:**
```cpp
uint8_t byte4 = ((command & 0x3) << 6) | 
                ((ack & 0x1) << 5) | 
                (msgType & 0x1F);
```

---

## Field Definitions

### sender (Byte 0)
- **Range:** 0-254
- **Values:**
  - `0`: Gateway (reserved)
  - `1-254`: Sensor nodes
  - `255`: Broadcast (not commonly used)

### last (Byte 1)
- **Purpose:** Mesh routing (last hop)
- **Current Use:** Not used for routing in this system
- **Typical Value:** Same as `sender`

### next (Byte 2)
- **Purpose:** Destination node
- **For sensor data:** Always `0` (gateway)
- **For gateway commands:** Node ID to target

### length (Byte 3)
- **Range:** 0-61
- **Purpose:** Number of bytes in payload
- **Validation:** Must be ≤ 61 (RFM69 max payload after header)

### command (Bits 6-7 of Byte 4)
- **Range:** 0-3 (2 bits)
- **Values:**
  - `0`: Presentation (node/sensor introduction)
  - `1`: Set (sensor value update)
  - `2`: Req (request value)
  - `3`: Internal (system messages)

### ack (Bit 5 of Byte 4)
- **Range:** 0-1 (1 bit)
- **Values:**
  - `0`: No application-level ACK requested
  - `1`: Request application-level ACK (not implemented in v1.0)
- **Note:** Hardware ACK is always used at RFM69 layer

### msgType (Bits 0-4 of Byte 4)
- **Range:** 0-31 (5 bits)
- **Purpose:** Sensor value type or internal message type
- **Examples:**
  - `0`: V_TEMP (temperature)
  - `1`: V_HUM (humidity)
  - `2`: V_STATUS (binary status)
  - See `include/settings.h` for complete list

### childId (Byte 5)
- **Range:** 0-255
- **Purpose:** Identifies which sensor on a multi-sensor node
- **Examples:**
  - Node with temp+humidity: childId 0=temp, childId 1=humidity
  - Node with single sensor: childId 0

### sensorId (Bytes 6-7)
- **Format:** Big-endian uint16
- **Purpose:** Sensor type identifier
- **Examples:**
  - `0`: S_DOOR (door/window sensor)
  - `1`: S_MOTION (motion sensor)
  - `6`: S_TEMP (temperature sensor)
  - `7`: S_HUM (humidity sensor)

### payload (Bytes 8+)
- **Size:** 0-61 bytes
- **Format:** Typically ASCII string representation of value
- **Examples:**
  - Temperature: `"22.5"`
  - Humidity: `"65"`
  - Status: `"1"` or `"0"`
  - Text: `"Motion detected"`

---

## Example Packets

### Contact Sensor (Door Open)

```
Hex:   02 02 00 01 41 00 00 00 31
      |  |  |  |  |  |  |  |  |
      |  |  |  |  |  |  |  |  +-- Payload: "1" (open)
      |  |  |  |  |  |  +-----+-- SensorId: 0x0000 (S_DOOR)
      |  |  |  |  |  +----------- ChildId: 0
      |  |  |  |  +-------------- Flags: 0x41 = cmd: 1, ack:0, type:1
      |  |  |  +----------------- Length: 1 byte
      |  |  +-------------------- Next:  0 (gateway)
      |  +----------------------- Last: 2
      +-------------------------- Sender: 2 (contact sensor node)
```

**Breakdown:**
- **Sender:** Node 2
- **Destination:** Gateway (0)
- **Command:** 1 (SET)
- **ACK:** 0 (no app-level ACK)
- **MsgType:** 1 (V_STATUS)
- **ChildId:** 0
- **SensorId:** 0 (S_DOOR)
- **Payload:** "1" (door is open)

### Temperature Reading

```
Hex:   03 03 00 04 40 00 00 00 32 32 2E 35
      |  |  |  |  |  |  |  |  |
      |  |  |  |  |  |  |  |  +-- Payload: "22.5"
      |  |  |  |  |  |  +-----+-- SensorId: 0x0000 (S_TEMP)
      |  |  |  |  |  +----------- ChildId: 0
      |  |  |  |  +-------------- Flags: 0x40 = cmd:1, ack:0, type: 0
      |  |  |  +----------------- Length: 4 bytes
      |  |  +-------------------- Next: 0 (gateway)
      |  +----------------------- Last: 3
      +-------------------------- Sender:  3 (temp/humidity node)
```

**Breakdown:**
- **Sender:** Node 3
- **Command:** 1 (SET)
- **MsgType:** 0 (V_TEMP)
- **ChildId:** 0 (temperature sensor)
- **Payload:** "22.5" (22.5°C)

---

## Validation Rules

When parsing packets: 

1. **Sender ID:** Must be 1-254 (0 is gateway, 255 is broadcast)
2. **Destination (next):** For sensor data, should be 0 (gateway)
3. **Length:** Must be 0-61 bytes
4. **Payload size:** Must match `length` field
5. **Command:** Must be 0-3
6. **Total packet size:** 8 + length ≤ 69 bytes (RFM69 limit)

---

## Protocol Version

**Current:** v1.0

**Future Considerations:**
- May use upper byte of `sensorId` for sequence numbers (v2.0)
- `last` field could be used for message tracking
- Payload format may support binary encoding

---

**Reference:** Based on MySensors Serial API 2.0  
**Last Updated:** 2026-01-06
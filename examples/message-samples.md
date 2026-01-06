# Message Examples

Real-world message samples for testing and reference.

## Contact Sensor (Node 2)

### Door Opens

MQTT Topic: mys-out/2/0/1/0/16
MQTT Payload: 1

Radio Packet (hex):
02 02 00 01 41 00 00 00 31

Radio Packet (decoded):
- Sender: 2
- Destination: 0 (gateway)
- Command: SET (1)
- Type: V_STATUS (16)
- Payload: "1"

### Door Closes

MQTT Topic: mys-out/2/0/1/0/16
MQTT Payload: 0

## Temperature/Humidity Sensor (Node 3)

### Temperature Reading

MQTT Topic: mys-out/3/0/1/0/0
MQTT Payload:  22.5

Radio Packet (hex):
03 03 00 04 40 00 00 00 32 32 2E 35

### Humidity Reading

MQTT Topic:  mys-out/3/1/1/0/1
MQTT Payload: 65

## Motion Sensor (Node 4)

### Motion Detected

MQTT Topic: mys-out/4/0/1/0/2
MQTT Payload: 1

### Motion Cleared

MQTT Topic:  mys-out/4/0/1/0/2
MQTT Payload: 0

---

Last Updated: 2026-01-06

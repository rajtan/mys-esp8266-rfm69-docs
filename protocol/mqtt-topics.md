# MQTT Topic Structure

Complete reference for MQTT topic format used by the gateway.

## Topic Format

### Outgoing (Nodes to MQTT)

Pattern: 
mys-out/<nodeId>/<childId>/<command>/<ack>/<type>

Fields:
- nodeId:  Sensor node ID (1-254)
- childId: Child sensor ID (0-255)
- command: Command type (0-3)
- ack: ACK flag (0 or 1)
- type: Message/value type (0-31)

Payload:  Sensor value as string

### Incoming (MQTT to Nodes)

Pattern:
mys-in/<nodeId>/<childId>/<command>/<ack>/<type>

Fields:  Same as outgoing

Payload: Command value as string

## Topic Examples

### Contact Sensor (Node 2)

Topic: mys-out/2/0/1/0/16

Breakdown:
- Node:  2
- Child: 0
- Command: 1 (SET)
- ACK: 0
- Type: 16 (V_STATUS)

Payloads:
- "0" = closed
- "1" = open

### Temperature/Humidity Sensor (Node 3)

Temperature: 
- Topic: mys-out/3/0/1/0/0
- Payload: "22.5" (22.5°C)

Humidity:
- Topic: mys-out/3/1/1/0/1
- Payload:  "65" (65%)

### Motion Sensor (Node 4)

Topic: mys-out/4/0/1/0/2

Payloads:
- "0" = no motion
- "1" = motion detected

## Subscribing

### Subscribe to All Sensor Data

mosquitto_sub -h broker.example.com -t "mys-out/#"

### Subscribe to Specific Node

mosquitto_sub -h broker.example.com -t "mys-out/3/#"

### Subscribe to Specific Sensor

mosquitto_sub -h broker.example.com -t "mys-out/3/0/#"

## Publishing Commands

### Send Command to Node

mosquitto_pub -h broker.example.com -t "mys-in/5/0/1/0/2" -m "1"

This would send "1" to node 5, child sensor 0.

## Configuration

### Default Topic Prefixes

Configurable in gateway web interface: 
- Outgoing prefix: mys-out/ (default)
- Incoming prefix: mys-in/ (default)

### Wildcard Subscription

Gateway subscribes to:
mys-in/+/+/+/+/+

This matches all incoming command topics.

---

Last Updated: 2026-01-06

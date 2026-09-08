# Events — IoT Service

> The IoT service emits hardware lifecycle and classroom assignment events used by attendance and monitoring workflows.

---

## Events this service PUBLISHES

| Event | Topic/Exchange | When it is emitted | Payload (key fields) |
|-------|---------------|--------------------|---------------------|
| DeviceRegistered | iot.device.events | A new device is registered | `{deviceId, schoolId, deviceName}` |
| DeviceAssigned | iot.assignment.events | A device is mapped to a classroom | `{deviceId, classroomId, assignedFrom}` |
| DeviceStatusChanged | iot.device.events | Device state changes | `{deviceId, status, lastConnection}` |
| DeviceOnline | iot.device.events | Device comes online and is ready for use | `{deviceId, schoolId, classroomId, timestamp}` |
| DeviceHeartbeat | iot.telemetry.events | Telemetry or health signal is received | `{deviceId, status, timestamp}` |

### Event schema: DeviceStatusChanged

```json
{
  "eventId": "uuid",
  "eventType": "DeviceStatusChanged",
  "timestamp": "2026-09-05T12:00:00Z",
  "version": "1.0",
  "source": "iot-service",
  "payload": {
    "deviceId": "uuid",
    "status": "Maintenance",
    "schoolId": "uuid",
    "lastConnection": "2026-09-05T12:00:00Z"
  }
}
```

---

## Events this service CONSUMES

| Event | Published by | Topic/Exchange | What this service does when it receives it |
|-------|-------------|----------------|-------------------------------------------|
| ClassSessionOpened | attendance-service | attendance.session.events | Correlates session opening with the classroom and relevant device state |
| ConfigurationUpdated | configuration-service | configuration.system.events | Applies operational settings used by hardware monitoring |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Device status updates should be processed idempotently |
| At-most-once | No | Some telemetry can be replayed during reconnects |
| Exactly-once | No | Event ordering is handled by device and time stamps |

---

## Error handling

- **Dead Letter Queue:** `iot.dlq`
- **Retries:** 5 retries with exponential backoff
- **Alerts:** Notify operations when a device remains in `Maintenance` or fails to report heartbeats for a sustained period

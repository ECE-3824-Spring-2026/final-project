# API Specification

Base URL: `/api/v1`

---

## POST /readings

Submit a new sensor reading.

**Request:**
```http
POST /api/v1/readings
Content-Type: application/json

{
    "sensor_id": "sensor-001",
    "temperature": 22.5,
    "humidity": 45.0,
    "location": "living-room"
}
```

**Success Response (201 Created):**
```json
{
    "id": "507f1f77bcf86cd799439011",
    "sensor_id": "sensor-001",
    "temperature": 22.5,
    "humidity": 45.0,
    "timestamp": "2024-03-15T14:30:00Z",
    "location": "living-room"
}
```

**Error Responses:**

| Status | When | Example |
|--------|------|---------|
| 400 | Business rule violation | Temperature out of range |
| 422 | Schema validation failed | Missing required field |

**Side Effects:**
- If `sensor_id` is new, create entry in `sensors` collection
- Update `last_seen` timestamp for the sensor

---

## GET /readings

Query historical readings.

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `sensor_id` | string | — | Filter by sensor |
| `start` | ISO datetime | — | Readings after this time |
| `end` | ISO datetime | — | Readings before this time |
| `limit` | int | 100 | Max results (1-1000) |
| `offset` | int | 0 | Skip N results |

**Request:**
```http
GET /api/v1/readings?sensor_id=sensor-001&limit=10
```

**Success Response (200 OK):**
```json
{
    "count": 2,
    "total": 150,
    "readings": [
        {
            "id": "507f1f77bcf86cd799439011",
            "sensor_id": "sensor-001",
            "temperature": 22.5,
            "humidity": 45.0,
            "timestamp": "2024-03-15T14:30:00Z",
            "location": "living-room"
        }
    ]
}
```

**Notes:**
- Results sorted by timestamp descending (newest first)
- `count` = items in this response, `total` = total matching

---

## GET /readings/{sensor_id}/latest

Get the most recent reading for a sensor.

**Request:**
```http
GET /api/v1/readings/sensor-001/latest
```

**Success Response (200 OK):**
```json
{
    "id": "507f1f77bcf86cd799439011",
    "sensor_id": "sensor-001",
    "temperature": 22.5,
    "humidity": 45.0,
    "timestamp": "2024-03-15T14:30:00Z",
    "location": "living-room"
}
```

**Error Response (404 Not Found):**
```json
{
    "error": "NOT_FOUND",
    "message": "No readings found for sensor 'sensor-001'"
}
```

---

## GET /sensors

List all registered sensors.

**Request:**
```http
GET /api/v1/sensors
```

**Success Response (200 OK):**
```json
{
    "count": 2,
    "sensors": [
        {
            "sensor_id": "sensor-001",
            "name": null,
            "location": "living-room",
            "registered_at": "2024-03-01T10:00:00Z",
            "last_seen": "2024-03-15T14:30:00Z"
        },
        {
            "sensor_id": "sensor-002",
            "name": null,
            "location": "bedroom",
            "registered_at": "2024-03-02T12:00:00Z",
            "last_seen": "2024-03-15T14:28:00Z"
        }
    ]
}
```

---

## GET /sensors/{sensor_id}/stats

Get statistics for a sensor over a time period.

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `hours` | int | 24 | Hours to look back (1-720) |

**Request:**
```http
GET /api/v1/sensors/sensor-001/stats?hours=24
```

**Success Response (200 OK):**
```json
{
    "sensor_id": "sensor-001",
    "period_hours": 24,
    "temperature": {
        "min": 20.1,
        "max": 24.5,
        "avg": 22.3
    },
    "humidity": {
        "min": 40.0,
        "max": 55.0,
        "avg": 47.5
    },
    "reading_count": 1440
}
```

**Error Response (404 Not Found):**
```json
{
    "error": "NOT_FOUND",
    "message": "No readings found for sensor 'sensor-001' in the last 24 hours"
}
```

---

## Health Check

## GET /health

**Response (200 OK):**
```json
{
    "status": "healthy",
    "database": "connected",
    "timestamp": "2024-03-15T14:30:00Z"
}
```

**Response (503 Service Unavailable):**
```json
{
    "status": "unhealthy",
    "database": "disconnected",
    "error": "Connection refused"
}
```

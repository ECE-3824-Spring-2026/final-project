# Sensor API - Complete Specification

## Overview

A REST API that receives temperature/humidity readings from IoT sensors and stores them in MongoDB. Includes a simple web dashboard.

**Tech Stack:**
- Python 3.11+
- FastAPI
- MongoDB (pymongo)
- Pydantic for validation

---

## Data Models

### SensorReading

```python
{
    "sensor_id": str,        # e.g., "sensor-001"
    "temperature": float,    # Celsius, range -40 to 85
    "humidity": float,       # Percentage, range 0 to 100
    "timestamp": datetime,   # ISO 8601 format, UTC
    "location": str          # Optional, e.g., "living-room"
}
```

**Validation Rules:**
- `sensor_id`: Required, 1-50 characters, alphanumeric with hyphens
- `temperature`: Required, must be between -40 and 85
- `humidity`: Required, must be between 0 and 100
- `timestamp`: Optional, defaults to current UTC time if not provided
- `location`: Optional, max 100 characters

### Sensor

```python
{
    "sensor_id": str,        # Primary key
    "name": str,             # Human-readable name
    "location": str,
    "registered_at": datetime,
    "last_seen": datetime    # Updated on each reading
}
```

---

## API Endpoints

### POST /readings

Receives a new sensor reading.

**Request Body:**
```json
{
    "sensor_id": "sensor-001",
    "temperature": 22.5,
    "humidity": 45.0,
    "location": "living-room"
}
```

**Response (201 Created):**
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
- 400: Invalid data (temperature out of range, missing fields)
- 422: Validation error (Pydantic)

---

### GET /readings

Returns recent readings with optional filters.

**Query Parameters:**
- `sensor_id` (optional): Filter by sensor
- `start` (optional): ISO datetime, readings after this time
- `end` (optional): ISO datetime, readings before this time
- `limit` (optional): Max results, default 100, max 1000

**Response (200 OK):**
```json
{
    "count": 2,
    "readings": [
        {
            "id": "507f1f77bcf86cd799439011",
            "sensor_id": "sensor-001",
            "temperature": 22.5,
            "humidity": 45.0,
            "timestamp": "2024-03-15T14:30:00Z",
            "location": "living-room"
        },
        {
            "id": "507f1f77bcf86cd799439012",
            "sensor_id": "sensor-001",
            "temperature": 22.8,
            "humidity": 44.0,
            "timestamp": "2024-03-15T14:25:00Z",
            "location": "living-room"
        }
    ]
}
```

---

### GET /readings/{sensor_id}/latest

Returns the most recent reading for a specific sensor.

**Response (200 OK):**
```json
{
    "sensor_id": "sensor-001",
    "temperature": 22.5,
    "humidity": 45.0,
    "timestamp": "2024-03-15T14:30:00Z",
    "location": "living-room"
}
```

**Error Responses:**
- 404: Sensor not found or no readings exist

---

### GET /sensors

List all known sensors.

**Response (200 OK):**
```json
{
    "count": 2,
    "sensors": [
        {
            "sensor_id": "sensor-001",
            "name": "Living Room Sensor",
            "location": "living-room",
            "last_seen": "2024-03-15T14:30:00Z"
        },
        {
            "sensor_id": "sensor-002",
            "name": "Bedroom Sensor",
            "location": "bedroom",
            "last_seen": "2024-03-15T14:28:00Z"
        }
    ]
}
```

---

### GET /stats/{sensor_id}

Returns min/max/avg statistics for a sensor.

**Query Parameters:**
- `hours` (optional): How many hours back to calculate, default 24

**Response (200 OK):**
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

---

## Database Schema

**Collection: `readings`**
```javascript
{
    _id: ObjectId,
    sensor_id: String (indexed),
    temperature: Number,
    humidity: Number,
    timestamp: Date (indexed, descending),
    location: String
}
```

**Collection: `sensors`**
```javascript
{
    _id: String (sensor_id),
    name: String,
    location: String,
    registered_at: Date,
    last_seen: Date
}
```

**Indexes:**
- `readings`: compound index on (sensor_id, timestamp desc)
- `readings`: TTL index on timestamp (optional, for auto-cleanup after 30 days)

---

## Environment Variables

```
MONGODB_URL=mongodb://localhost:27017
DATABASE_NAME=sensor_data
DEBUG=false
```

---

## Unit Tests

### Validation Tests

| Test Case | Input | Expected |
|-----------|-------|----------|
| Valid reading | `{"sensor_id": "s1", "temperature": 22.5, "humidity": 45}` | 201 Created |
| Missing sensor_id | `{"temperature": 22.5, "humidity": 45}` | 422 Error |
| Temperature too high | `{"sensor_id": "s1", "temperature": 100, "humidity": 45}` | 400 Error |
| Temperature too low | `{"sensor_id": "s1", "temperature": -50, "humidity": 45}` | 400 Error |
| Humidity over 100 | `{"sensor_id": "s1", "temperature": 22, "humidity": 150}` | 400 Error |
| Humidity negative | `{"sensor_id": "s1", "temperature": 22, "humidity": -5}` | 400 Error |
| Empty sensor_id | `{"sensor_id": "", "temperature": 22, "humidity": 45}` | 422 Error |

### Query Tests

| Test Case | Setup | Query | Expected |
|-----------|-------|-------|----------|
| Filter by sensor | 3 readings from s1, 2 from s2 | `GET /readings?sensor_id=s1` | 3 readings |
| Limit results | 10 readings | `GET /readings?limit=5` | 5 readings |
| Time range | Readings at 1:00, 2:00, 3:00 | `GET /readings?start=1:30&end=2:30` | 1 reading (2:00) |
| Latest reading | 3 readings for s1 | `GET /readings/s1/latest` | Most recent only |
| No readings | Empty DB | `GET /readings/s1/latest` | 404 |

### Statistics Tests

| Test Case | Readings | Expected Stats |
|-----------|----------|----------------|
| Single reading | temp=20 | min=20, max=20, avg=20 |
| Three readings | temp=20,22,24 | min=20, max=24, avg=22 |
| Empty period | No readings in range | 404 or empty stats |

---

## Error Response Format

All errors should return:
```json
{
    "error": "Short error code",
    "message": "Human readable description",
    "details": {}  // Optional, validation errors etc.
}
```

---

## File Structure

```
sensor-api/
├── app/
│   ├── __init__.py
│   ├── main.py           # FastAPI app, routes
│   ├── models.py         # Pydantic models
│   ├── database.py       # MongoDB connection
│   └── config.py         # Environment config
├── tests/
│   ├── __init__.py
│   ├── test_api.py       # Endpoint tests
│   ├── test_models.py    # Validation tests
│   └── conftest.py       # Pytest fixtures
├── requirements.txt
├── .env.example
└── README.md
```

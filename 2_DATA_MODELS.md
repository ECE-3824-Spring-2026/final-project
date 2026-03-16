# Data Models Specification

## Pydantic Models

### SensorReadingCreate (Input)

Used when POSTing a new reading.

```python
class SensorReadingCreate(BaseModel):
    sensor_id: str
    temperature: float
    humidity: float
    timestamp: datetime | None = None  # Defaults to now
    location: str | None = None
```

**Validation Rules:**

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `sensor_id` | str | Yes | 1-50 chars, pattern: `^[a-zA-Z0-9-]+$` |
| `temperature` | float | Yes | Range: -40.0 to 85.0 (DHT22 spec) |
| `humidity` | float | Yes | Range: 0.0 to 100.0 |
| `timestamp` | datetime | No | ISO 8601 format, defaults to UTC now |
| `location` | str | No | Max 100 chars |

### SensorReadingResponse (Output)

Returned from API queries.

```python
class SensorReadingResponse(BaseModel):
    id: str                    # MongoDB ObjectId as string
    sensor_id: str
    temperature: float
    humidity: float
    timestamp: datetime
    location: str | None
```

### SensorResponse

```python
class SensorResponse(BaseModel):
    sensor_id: str
    name: str | None
    location: str | None
    registered_at: datetime
    last_seen: datetime
```

### StatsResponse

```python
class StatValues(BaseModel):
    min: float
    max: float
    avg: float

class StatsResponse(BaseModel):
    sensor_id: str
    period_hours: int
    temperature: StatValues
    humidity: StatValues
    reading_count: int
```

### ErrorResponse

```python
class ErrorResponse(BaseModel):
    error: str          # Short code: "VALIDATION_ERROR", "NOT_FOUND"
    message: str        # Human readable
    details: dict = {}  # Optional field-level errors
```

---

## MongoDB Collections

### Collection: `readings`

```javascript
{
    _id: ObjectId,
    sensor_id: String,
    temperature: Number,
    humidity: Number,
    timestamp: Date,
    location: String | null
}
```

**Indexes:**
```javascript
// Fast queries by sensor + time range
{ sensor_id: 1, timestamp: -1 }

// Optional: Auto-delete after 30 days
{ timestamp: 1 }, { expireAfterSeconds: 2592000 }
```

### Collection: `sensors`

Auto-populated when new sensor_id first appears.

```javascript
{
    _id: String,          // sensor_id is the primary key
    name: String | null,
    location: String | null,
    registered_at: Date,
    last_seen: Date
}
```

**Indexes:**
```javascript
// Fast lookup by last activity
{ last_seen: -1 }
```

---

## Validation Error Examples

**Temperature out of range:**
```json
{
    "error": "VALIDATION_ERROR",
    "message": "Temperature must be between -40 and 85",
    "details": {
        "field": "temperature",
        "value": 100,
        "constraint": "range(-40, 85)"
    }
}
```

**Missing required field:**
```json
{
    "error": "VALIDATION_ERROR", 
    "message": "sensor_id is required",
    "details": {
        "field": "sensor_id",
        "type": "missing"
    }
}
```

**Invalid sensor_id format:**
```json
{
    "error": "VALIDATION_ERROR",
    "message": "sensor_id must contain only letters, numbers, and hyphens",
    "details": {
        "field": "sensor_id",
        "value": "sensor@123!",
        "pattern": "^[a-zA-Z0-9-]+$"
    }
}
```

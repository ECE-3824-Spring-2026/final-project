# Test Specification

## Test Structure

```
tests/
├── conftest.py          # Fixtures: test client, mock DB
├── test_models.py       # Pydantic validation tests
├── test_api_readings.py # /readings endpoint tests
├── test_api_sensors.py  # /sensors endpoint tests
└── test_api_stats.py    # /stats endpoint tests
```

---

## Fixtures (conftest.py)

```python
@pytest.fixture
def test_client():
    """FastAPI TestClient with test database"""
    
@pytest.fixture
def mock_db():
    """In-memory MongoDB or mongomock"""

@pytest.fixture
def sample_readings():
    """Pre-populated readings for query tests"""
    return [
        {"sensor_id": "s1", "temperature": 20.0, "humidity": 40.0, "timestamp": "2024-03-15T10:00:00Z"},
        {"sensor_id": "s1", "temperature": 22.0, "humidity": 45.0, "timestamp": "2024-03-15T11:00:00Z"},
        {"sensor_id": "s1", "temperature": 24.0, "humidity": 50.0, "timestamp": "2024-03-15T12:00:00Z"},
        {"sensor_id": "s2", "temperature": 18.0, "humidity": 55.0, "timestamp": "2024-03-15T10:30:00Z"},
        {"sensor_id": "s2", "temperature": 19.0, "humidity": 60.0, "timestamp": "2024-03-15T11:30:00Z"},
    ]
```

---

## Model Validation Tests

### Valid Input Tests

| Test Name | Input | Expected |
|-----------|-------|----------|
| `test_valid_reading_minimal` | `{"sensor_id": "s1", "temperature": 22.5, "humidity": 45}` | Model creates successfully |
| `test_valid_reading_full` | `{"sensor_id": "s1", "temperature": 22.5, "humidity": 45, "timestamp": "2024-03-15T10:00:00Z", "location": "room1"}` | Model creates successfully |
| `test_valid_sensor_id_with_hyphens` | `{"sensor_id": "my-sensor-01", ...}` | Valid |
| `test_valid_temperature_boundary_low` | `{"temperature": -40.0, ...}` | Valid |
| `test_valid_temperature_boundary_high` | `{"temperature": 85.0, ...}` | Valid |
| `test_valid_humidity_zero` | `{"humidity": 0.0, ...}` | Valid |
| `test_valid_humidity_hundred` | `{"humidity": 100.0, ...}` | Valid |

### Invalid Input Tests

| Test Name | Input | Expected Error |
|-----------|-------|----------------|
| `test_missing_sensor_id` | `{"temperature": 22, "humidity": 45}` | ValidationError: sensor_id required |
| `test_missing_temperature` | `{"sensor_id": "s1", "humidity": 45}` | ValidationError: temperature required |
| `test_missing_humidity` | `{"sensor_id": "s1", "temperature": 22}` | ValidationError: humidity required |
| `test_empty_sensor_id` | `{"sensor_id": "", ...}` | ValidationError: min length 1 |
| `test_sensor_id_too_long` | `{"sensor_id": "x" * 51, ...}` | ValidationError: max length 50 |
| `test_sensor_id_invalid_chars` | `{"sensor_id": "sensor@123!", ...}` | ValidationError: pattern mismatch |
| `test_temperature_too_low` | `{"temperature": -41, ...}` | ValidationError: >= -40 |
| `test_temperature_too_high` | `{"temperature": 86, ...}` | ValidationError: <= 85 |
| `test_humidity_negative` | `{"humidity": -1, ...}` | ValidationError: >= 0 |
| `test_humidity_over_100` | `{"humidity": 101, ...}` | ValidationError: <= 100 |
| `test_invalid_timestamp_format` | `{"timestamp": "not-a-date", ...}` | ValidationError: datetime format |

---

## API Endpoint Tests

### POST /readings

| Test Name | Setup | Request | Expected |
|-----------|-------|---------|----------|
| `test_create_reading_success` | Empty DB | Valid reading | 201, reading returned with id |
| `test_create_reading_auto_timestamp` | — | Reading without timestamp | 201, timestamp auto-populated |
| `test_create_reading_new_sensor` | Empty sensors | New sensor_id | 201, sensor auto-registered |
| `test_create_reading_updates_last_seen` | Existing sensor | New reading | sensor.last_seen updated |
| `test_create_reading_validation_error` | — | Invalid temp | 422, error details |
| `test_create_reading_missing_field` | — | Missing sensor_id | 422, field error |

### GET /readings

| Test Name | Setup | Query | Expected |
|-----------|-------|-------|----------|
| `test_get_readings_empty` | Empty DB | No params | 200, empty list |
| `test_get_readings_all` | 5 readings | No params | 200, 5 readings |
| `test_get_readings_by_sensor` | 3 from s1, 2 from s2 | `?sensor_id=s1` | 200, 3 readings |
| `test_get_readings_limit` | 10 readings | `?limit=5` | 200, 5 readings |
| `test_get_readings_offset` | 10 readings | `?limit=5&offset=5` | 200, readings 6-10 |
| `test_get_readings_time_range` | Readings at 10:00, 11:00, 12:00 | `?start=10:30&end=11:30` | 200, 1 reading (11:00) |
| `test_get_readings_sorted_desc` | Multiple readings | No params | Newest first |
| `test_get_readings_invalid_limit` | — | `?limit=5000` | 422, max is 1000 |

### GET /readings/{sensor_id}/latest

| Test Name | Setup | Request | Expected |
|-----------|-------|---------|----------|
| `test_get_latest_success` | 3 readings for s1 | `/readings/s1/latest` | 200, newest reading |
| `test_get_latest_not_found` | Empty DB | `/readings/s1/latest` | 404 |
| `test_get_latest_wrong_sensor` | Readings for s1 only | `/readings/s2/latest` | 404 |

### GET /sensors

| Test Name | Setup | Request | Expected |
|-----------|-------|---------|----------|
| `test_get_sensors_empty` | Empty DB | `/sensors` | 200, empty list |
| `test_get_sensors_multiple` | 3 sensors | `/sensors` | 200, 3 sensors |
| `test_get_sensors_includes_last_seen` | Sensor with readings | `/sensors` | last_seen populated |

### GET /sensors/{sensor_id}/stats

| Test Name | Setup | Request | Expected |
|-----------|-------|---------|----------|
| `test_stats_single_reading` | 1 reading, temp=20 | `/sensors/s1/stats` | min=20, max=20, avg=20 |
| `test_stats_multiple_readings` | temps: 20, 22, 24 | `/sensors/s1/stats` | min=20, max=24, avg=22 |
| `test_stats_custom_hours` | Readings over 48h | `?hours=12` | Only last 12h counted |
| `test_stats_not_found` | Empty DB | `/sensors/s1/stats` | 404 |
| `test_stats_no_recent` | Old readings only | `?hours=1` | 404 or empty |

---

## Edge Cases & Error Handling

| Test Name | Scenario | Expected |
|-----------|----------|----------|
| `test_db_connection_error` | DB unreachable | 503, error message |
| `test_concurrent_writes` | Simultaneous POSTs | All succeed, no duplicates |
| `test_unicode_location` | `location: "客厅"` | Stored and returned correctly |
| `test_very_long_query` | 1000 readings requested | Completes in < 1s |
| `test_malformed_json` | `{invalid json}` | 422 |
| `test_wrong_content_type` | POST as form data | 422 |

---

## Integration Tests

| Test Name | Scenario | Expected |
|-----------|----------|----------|
| `test_full_workflow` | POST reading → GET latest → GET stats | All operations succeed |
| `test_sensor_auto_registration` | POST with new sensor_id | Sensor appears in /sensors |
| `test_stats_update_with_new_reading` | POST reading → GET stats | New reading included |

---

## Test Data Generators

```python
def random_reading(sensor_id: str = "test-sensor") -> dict:
    """Generate a valid random reading for testing"""
    return {
        "sensor_id": sensor_id,
        "temperature": random.uniform(-40, 85),
        "humidity": random.uniform(0, 100),
        "location": random.choice(["room1", "room2", None])
    }

def readings_over_time(sensor_id: str, count: int, hours: int) -> list[dict]:
    """Generate readings spread over N hours"""
    ...
```

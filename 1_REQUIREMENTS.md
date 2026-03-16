# Sensor API - Project Requirements

## Overview

Build a REST API that receives temperature/humidity readings from IoT sensors, stores them in MongoDB, and provides a simple web dashboard.

## Tech Stack

- **Language:** Python 3.11+
- **Framework:** FastAPI
- **Database:** MongoDB (pymongo driver)
- **Validation:** Pydantic v2
- **Testing:** pytest + pytest-asyncio

## User Stories

### As a sensor device, I need to:
- POST readings (temperature, humidity) to the API
- Include my sensor ID with each reading
- Receive confirmation that my reading was stored

### As a dashboard user, I need to:
- View the latest reading from each sensor
- View historical readings with time filters
- See min/max/avg statistics over configurable time periods
- See a list of all registered sensors

### As a system administrator, I need to:
- Configure the database connection via environment variables
- Have readings auto-expire after 30 days (optional)
- See meaningful error messages when things fail

## Non-Functional Requirements

- API response time < 100ms for single-reading queries
- Handle at least 10 readings/second sustained
- All timestamps in UTC
- Input validation with clear error messages

## Related Specs

- [Data Models](2_DATA_MODELS.md) — Pydantic schemas and database collections
- [API Specification](3_API_SPEC.md) — Endpoints, requests, responses
- [Test Cases](4_TESTS.md) — Unit and integration tests

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
├── docker-compose.yml
├── .env.example
└── README.md
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `MONGODB_URL` | Yes | — | MongoDB connection string |
| `DATABASE_NAME` | No | `sensor_data` | Database name |
| `DEBUG` | No | `false` | Enable debug logging |

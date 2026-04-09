# beacon-observability

Real-time operational data microservice for the Beacon emergency-response platform. beacon-observability is the data supply layer for the platform — it aggregates live feeds from eleven categories of city infrastructure (traffic, public transport, weather, hospitals, power grid, water levels, and more) and serves them to Emergency Response Teams through a consistent REST and Server-Sent Events API.

---

## Overview

During a city-scale emergency, responders need more than incident records — they need a live picture of the city's operational state. Is the nearest hospital at capacity? Which roads are blocked? Is there flooding in the affected area? Are any power substations offline? beacon-observability is the service that answers those questions in real time.

The service does not collect data itself. That is the job of beacon-etl-flows, which ingests data from Irish public APIs (NTA, Met Éireann, ESB Networks, TomTom, Irish Rail, and others) and stores it in ClickHouse. beacon-observability sits in front of that data store and exposes it through a queryable, streaming API that is optimized for the ERT dashboard and mobile app.

beacon-core depends on beacon-observability at dispatch creation time: when a dispatcher assigns a ground-staff team to an incident, beacon-core calls the observability service to find the nearest emergency station to the incident location and pre-populate the dispatch with station coordinates.

---

## Architecture

```
beacon-etl-flows
(Apache Airflow DAGs)
        │
        │  ETL: Irish public APIs → ClickHouse
        ▼
┌───────────────────────────────┐
│  ClickHouse (external_data DB)│  High-volume time-series store
└──────────────┬────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────┐
│  beacon-observability                            │
│                                                  │
│  ClickHouse reader ──► Domain services           │
│  PostgreSQL reader  ──► (traffic, weather,       │
│  Redis cache        ──►  transport, hospitals,   │
│                          ambulances, power, etc.)│
│                                                  │
│  REST API                                        │
│  SSE streaming endpoints                         │
└──────────────────────────────────────────────────┘
        │
        ├── beacon-core       (station lookup for dispatch creation)
        ├── beacon-dashboard  (ERT situational awareness panels)
        └── beacon-mobile-app (citizen safety info + ground-staff map)
```

---

## Data Domains

beacon-observability serves eleven categories of operational data. Each domain has its own set of REST endpoints and, for time-critical categories, a Server-Sent Events endpoint that pushes updates to connected clients.

### Geographic

Reference data for countries and cities. Used for filtering and display in the ERT dashboard. Updated infrequently; served from Redis cache.

### Traffic

Real-time traffic flows and incidents sourced from the TomTom Traffic API. Traffic flows report congestion levels on major road segments. Traffic incidents report accidents, roadworks, and closures with geometry and estimated clearance times.

The traffic incidents endpoint is used by the ERT dashboard to overlay road hazards on the incident map. SSE streaming keeps the overlay current without polling.

### Public Transport

Vehicle tracking for buses (NTA GTFS-R), Luas trams, and Irish Rail trains. The service stores the current position, route, delay, and next-stop information for each active vehicle. Ground staff use this data to understand transport disruptions near an incident.

### Camera Feeds

Metadata and thumbnail images for city CCTV cameras near an incident. The service stores camera locations, operational status, and the most recent frame. Full video streams are accessed directly from the camera provider — beacon-observability provides only the index and metadata.

### Power Grid

Power outage data sourced from ESB Networks. Stores active outages with affected area, estimated restoration time, and number of impacted customers. Critical for responders to know if an area has lost power during an incident (no lifts, no traffic lights, no refrigerated medication).

### Weather

Weather observations and forecasts for six Dublin monitoring stations from Met Éireann. The service stores current conditions (temperature, wind, precipitation, pressure) and short-term forecasts. Also serves air quality measurements. Updated hourly.

### Water Levels

River gauge and coastal sensor readings from waterlevel.ie. Each sensor reports current level, alert threshold, and flood risk status. When a sensor exceeds its alert threshold, beacon-etl-flows fires an incident to beacon-core via the alert DAG. The SSE endpoint streams live level updates to the ERT dashboard during flood events.

### Hospitals

Hospital operational status including bed availability by department, emergency department wait times, and diversion status (whether ambulances are being redirected to a different hospital). Updated every 15 minutes. Critical for dispatch coordination — a responder needs to know if the nearest ED is on diversion before routing an ambulance.

### Ambulances

Real-time ambulance location and availability. Stores current position, status (available / responding / at-scene / returning), and the incident they are assigned to if responding. The SSE endpoint streams position updates for the live map view. Also supports querying for the closest available ambulance to a given coordinate pair.

### Emergency Services

Police and fire station locations, unit counts, and current operational status. Used by beacon-core's dispatch creation flow to identify the nearest station to an incident and pre-populate the dispatch.

### Social Media

Aggregated social media alerts related to city emergencies. Stores posts mentioning disaster-related keywords, filtered and deduplicated. Used by ERT members to gain situational awareness beyond the official incident record — citizens often report conditions on social media before calling emergency services.

---

## API Reference

All endpoints are prefixed with `/api/observability/v1`.

### Traffic

#### `GET /traffic/flows`
Returns current traffic flow readings for all monitored road segments. Supports filtering by bounding box (lat/lng rectangle) for map viewport queries. Each record includes road segment ID, speed, free-flow speed, congestion ratio, and last-updated timestamp.

**Query parameters:** `bbox` (e.g. `53.2,-6.4,53.5,-6.0`), `min_congestion` (float 0–1)

#### `GET /traffic/incidents`
Returns active traffic incidents (accidents, closures, roadworks). Each incident includes location, type, severity, description, affected roads, and estimated clearance time.

**Query parameters:** `bbox`, `type` (accident|closure|roadwork), `severity`

#### `GET /traffic/incidents/stream`
SSE endpoint. Pushes new and updated traffic incidents to connected clients as events are ingested from TomTom. Each SSE event is a JSON-serialized incident record.

### Public Transport

#### `GET /transport/vehicles`
Returns the current position and status of all tracked public transport vehicles. Each record includes vehicle ID, route, line, operator (Bus Éireann, Dublin Bus, Luas, Irish Rail), current position, next stop, and delay in seconds.

**Query parameters:** `operator` (bus|luas|rail), `route`, `bbox`

#### `GET /transport/vehicles/:id`
Returns the full tracking record for a specific vehicle including its full trip history for the current service day.

### Weather

#### `GET /weather/current`
Returns current weather observations for all monitored Dublin stations. Each observation includes temperature, feels-like, wind speed and direction, precipitation, pressure, humidity, and visibility.

**Query parameters:** `station_id` (filter to a specific station)

#### `GET /weather/forecast`
Returns the short-term weather forecast for a given location (lat/lng). Calls the stored Met Éireann forecast data; does not make an external API call in the request path.

#### `GET /weather/warnings`
Returns active weather warnings issued by Met Éireann. Each warning includes severity (advisory/watch/warning), phenomenon type (wind/rain/snow/ice/fog), affected counties, and valid time range.

### Water Levels

#### `GET /water/sensors`
Returns all water level sensors with current readings, alert thresholds, and flood risk classification (normal / advisory / watch / warning / emergency).

#### `GET /water/sensors/:id`
Returns a time series of readings for a specific sensor for the last 24 hours.

#### `GET /water/sensors/stream`
SSE endpoint. Pushes updated sensor readings as new data is ingested. Used by the ERT dashboard during flood events to track rising levels without polling.

### Hospitals

#### `GET /hospitals`
Returns operational status for all hospitals in the network. Each record includes the hospital name, location, total bed count, available beds, ED wait time, and diversion status.

#### `GET /hospitals/:id`
Returns detailed status for a specific hospital including bed availability broken down by ward, ED occupancy history for the last 6 hours, and contact information.

### Ambulances

#### `GET /ambulances`
Returns all ambulances with current position, status, and assignment.

**Query parameters:** `status` (available|responding|at_scene|returning), `bbox`

#### `GET /ambulances/closest`
Returns the closest available ambulance to a given coordinate. Used by dispatch operators to identify the nearest unit.

**Query parameters:** `lat`, `lng` (required)

#### `GET /ambulances/stream`
SSE endpoint. Streams live ambulance position updates for the map view.

### Emergency Services

#### `GET /emergency-stations`
Returns all emergency stations (police, fire, ambulance) with location, type, unit count, and operational status. beacon-core calls this endpoint during dispatch creation to find the nearest station to an incident.

**Query parameters:** `type` (police|fire|ambulance|all), `lat`, `lng`, `radius_km`

### Power Grid

#### `GET /power/outages`
Returns all active power outages. Each record includes the affected area geometry, outage type (planned/unplanned), number of customers affected, time of occurrence, and estimated restoration time.

#### `GET /power/usage`
Returns current power grid usage statistics including total demand, renewable generation percentage, and regional load distribution.

---

## Data Store Architecture

beacon-observability uses three storage backends, each chosen for its access pattern:

### ClickHouse

Time-series and high-volume operational data lives in the `external_data` database in ClickHouse. This includes all traffic data, transport vehicle positions, water level readings, weather observations, and social media alerts. ClickHouse handles the high ingestion rate from beacon-etl-flows (which writes to it directly) and makes wide time-range analytical queries fast.

### PostgreSQL

Reference and relational data — hospital records, emergency station directories, camera feed metadata, and geographic data — lives in PostgreSQL. This data is lower volume and benefits from relational queries and transactions.

### Redis

Frequently queried, slowly changing data (weather current conditions, hospital status summaries, emergency station directories) is cached in Redis with short TTLs. This absorbs the read load from the dashboard and mobile app, which poll these endpoints every few minutes per active user.

---

## Local Development

```bash
# Install dependencies
go mod download

# Start ClickHouse, PostgreSQL, and Redis
docker-compose up -d

# Seed reference data (emergency stations, hospital directory, etc.)
# The first_input.sql file seeds the initial static dataset
psql -U user -d observability -f migrations/seed/first_input.sql

# Run migrations
make migrate-up

# Start the service
make run
```

---

## Project Structure

```
beacon-observability/
├── cmd/server/main.go           # Composition root
├── internal/
│   ├── domain/
│   │   ├── geographic/          # Countries, cities
│   │   ├── traffic/             # Flows, incidents, SSE broker
│   │   ├── transport/           # Vehicle tracking
│   │   ├── camera/              # CCTV feed metadata
│   │   ├── power/               # Outages, usage
│   │   ├── weather/             # Observations, forecasts, warnings, air quality
│   │   ├── water/               # Sensors, flood risk, SSE broker
│   │   ├── hospital/            # Status, bed availability
│   │   ├── ambulance/           # Location, availability, SSE broker
│   │   ├── emergency/           # Police, fire stations
│   │   └── social/              # Alert aggregation, keyword monitoring
│   ├── http/
│   │   ├── router.go
│   │   └── handlers/
│   ├── registry/
│   │   ├── commands/
│   │   └── queries/
│   └── adapters/
│       ├── clickhouse/          # ClickHouse readers
│       ├── database/            # PostgreSQL repositories
│       └── cache/               # Redis adapter
├── migrations/
└── pkg/
```

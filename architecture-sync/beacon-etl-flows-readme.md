# beacon-etl-flows

Apache Airflow DAG collection for the Beacon emergency-response platform. This repository contains the data pipelines that feed beacon-observability's real-time city-state picture: 13 ETL DAGs ingest data from Irish public APIs into ClickHouse, and 5 alert DAGs continuously monitor that data and fire incidents to beacon-core when thresholds are crossed.

---

## Overview

The Beacon platform's operational awareness depends on a continuous stream of live city data: bus positions, traffic incidents, river levels, power outages, weather warnings. None of this data originates in the Beacon system — it comes from Irish public APIs operated by government agencies and third-party providers. beacon-etl-flows is the pipeline layer that bridges those external data sources with the platform's internal data warehouse.

The architecture is deliberately separated from the rest of the platform: ETL pipelines run on their own Airflow cluster, write to their own ClickHouse database (`external_data`), and interact with beacon-core only through the public incident API. This separation means a pipeline failure does not take down the core emergency management service, and adding a new data source only requires writing a new DAG, not modifying any service.

---

## Architecture

```
Irish Public APIs
┌─────────────────────────────────────────────────────────┐
│  NTA GTFS-R  │  Met Éireann  │  ESB Networks            │
│  Irish Rail  │  TomTom       │  Luas                    │
│  waterlevel.ie │  Dublin Live RSS  │  Waterlevel sensors │
└─────────────────────────────────────────────────────────┘
                          │
                          │  HTTP / REST / GTFS Protobuf
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Apache Airflow (Kubernetes, CeleryExecutor)            │
│                                                         │
│  ETL DAGs (13)           Alert DAGs (5)                 │
│  ┌───────────────────┐   ┌────────────────────────┐     │
│  │ Extract from API  │   │ Query ClickHouse        │     │
│  │ Transform + clean │   │ Evaluate thresholds     │     │
│  │ Load → ClickHouse │   │ Deduplicate via PG      │     │
│  └───────────────────┘   │ POST → beacon-core API  │     │
│                          └────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
                   ClickHouse (external_data DB)
                          │
                          ▼
                   beacon-observability (reads and serves to platform)
```

---

## ETL DAGs

Each ETL DAG is responsible for a single data source. They run on fixed schedules using Airflow's native cron scheduling. All DAGs follow the same three-task pattern: `extract` → `transform` → `load`.

### Real-Time Transport

#### `gtfs_realtime_vehicle_positions_dag`
**Schedule:** Every 5 minutes

Fetches the NTA GTFS-R vehicle positions feed, decodes the Protobuf binary format, and loads current bus positions (route, direction, lat/lng, bearing, occupancy, delay) into the `vehicle_positions` ClickHouse table. This feed covers all Dublin Bus, Bus Éireann, and Go-Ahead services operating on NTA routes.

#### `gtfs_realtime_trip_updates_dag`
**Schedule:** Every 5 minutes

Fetches the NTA GTFS-R trip updates feed, which provides arrival and departure time predictions for every upcoming stop on active trips. Loads into the `trip_updates` table. Used by beacon-observability to display real-time public transport delay information.

#### `luas_forecast_dag`
**Schedule:** Every 5 minutes

Fetches the Luas Cross-City and Red Line vehicle forecast APIs, which provide the next expected tram arrivals at each stop. Loads into the `luas_forecasts` table. The Luas API is REST JSON-based rather than GTFS, requiring a separate DAG from the bus feed.

#### `irish_rail_dag`
**Schedule:** Every 5 minutes

Fetches Irish Rail's real-time train movement API for all Dart, Commuter, and Intercity services. Loads current train positions, punctuality status, and next-stop information into the `irish_rail_movements` table.

#### `gtfs_static_dag`
**Schedule:** Daily at 03:00

Fetches the full NTA GTFS static dataset (zip archive containing routes, stops, shapes, trips, and calendars). Unpacks and loads the static schedule tables used for route lookups and stop name resolution. Runs at 03:00 to avoid overlapping with peak real-time ingestion.

### Weather and Environment

#### `met_eireann_dag`
**Schedule:** Hourly

Fetches current weather observations and short-term forecasts from Met Éireann's open data API for six monitoring stations in the Greater Dublin area. Stores temperature, wind speed and direction, precipitation, pressure, humidity, and visibility. Also fetches the ultraviolet index and marine forecast for coastal stations.

#### `weather_warnings_dag`
**Schedule:** Every 30 minutes

Polls Met Éireann's weather warning API for active warnings. Stores each warning with its severity level (advisory/watch/warning), phenomenon type, affected counties, and validity period. Checks for duplicate warnings before inserting to avoid storing the same warning multiple times across polling intervals.

#### `water_levels_dag`
**Schedule:** Every 15 minutes

Fetches water level readings from waterlevel.ie's sensor API for rivers and coastal gauges in the Dublin region. Each reading includes the current level in centimetres, the sensor's alert threshold, the percentage of threshold reached, and the trend direction. Loads into the `water_levels` table.

### Infrastructure

#### `esb_power_outages_dag`
**Schedule:** Every 10 minutes

Fetches the ESB Networks outage map API, which reports active and planned power outages across Ireland. Parses outage records with affected area geometry, type (planned maintenance or unplanned fault), number of affected customers, and estimated restoration time. Loads into the `power_outages` ClickHouse table. This is one of the most operationally critical feeds — power outages during an emergency compound rescue and hospital coordination challenges significantly.

### Traffic and Location

#### `tomtom_traffic_incidents_dag`
**Schedule:** Hourly

Fetches traffic incident data from the TomTom Traffic Incidents API for ten major Irish cities (Dublin, Cork, Limerick, Galway, Waterford, Kilkenny, Wexford, Drogheda, Dundalk, Sligo). Incidents include accidents, road closures, and planned roadworks with geometry, severity, and estimated clearance time. Loads into the `traffic_incidents` table.

### News and Social

#### `dublin_live_news_dag`
**Schedule:** Every 15 minutes

Fetches the Dublin Live RSS news feed and extracts articles related to city incidents, traffic, and emergency events. Stores article titles, summaries, publication times, and URLs. These articles are surfaced in the beacon-communication-platform's social feed to give ERT members access to verified news alongside citizen reports.

#### `poi_locations_dag`
**Schedule:** Monthly

Fetches Points of Interest data from TomTom for emergency-relevant locations: hospitals, police stations, fire stations, shelters, pharmacies, and petrol stations. This is static reference data that changes infrequently; monthly refresh is sufficient.

---

## Alert DAGs

Alert DAGs are the second layer of the pipeline. They do not ingest data — they watch the data already loaded by the ETL DAGs and raise incidents in beacon-core when conditions warrant.

Each alert DAG follows the same pattern:
1. Query ClickHouse for records that exceed an alert threshold
2. For each matching record, check the `incident_alerts` PostgreSQL table to see if an incident has already been raised for this event (deduplication)
3. If no existing incident is found, call the beacon-core incident API to create a new incident
4. Record the mapping in `incident_alerts` so future runs do not create duplicates

### `alert_power_outages_dag`
**Schedule:** Every 10 minutes

Monitors the `power_outages` table for large-scale outages (above a configurable customer-count threshold). Raises a beacon-core incident with type `power_outage` for each qualifying event. The incident description includes the affected area and estimated restoration time.

### `alert_severe_weather_dag`
**Schedule:** Every 30 minutes

Monitors the `weather_warnings` table for active severe weather warnings of watch or warning severity. Raises incidents for Status Yellow, Orange, and Red weather events. Status Advisory events are not escalated to incidents but are noted in the observability API.

### `alert_traffic_incidents_dag`
**Schedule:** Hourly

Monitors the `traffic_incidents` table for major road closures and serious accidents on primary routes. Raises incidents for closures that affect key arterial routes or emergency vehicle access corridors.

### `alert_water_levels_dag`
**Schedule:** Every 15 minutes

Monitors the `water_levels` table for sensors exceeding their alert threshold. Raises flood-risk incidents for sensors above 80% of threshold (watch) and above 100% (active flood). The incident severity is scaled based on the threshold percentage.

### `alert_weather_warnings_dag`
**Schedule:** Every 30 minutes

Raised as a separate DAG from `alert_severe_weather_dag` to handle specific meteorological events (thunderstorms, freezing fog, black ice) that require different response protocols from general severe weather.

---

## Utilities

### `config.py`

Reads all configuration from Airflow Variables. This includes API keys, service URLs, ClickHouse connection details, and threshold values for the alert DAGs. All configuration is centralized here — DAGs import from `config.py` rather than reading environment variables directly.

### `clickhouse_client.py`

A thin wrapper around the ClickHouse HTTP interface. Handles connection, query execution, and batch inserts. All DAG tasks that write to ClickHouse use this client to ensure consistent connection settings and error handling.

### `http_client.py`

HTTP client with built-in retry logic (3 attempts with exponential backoff). All DAG tasks that call external APIs use this client. This ensures that transient API failures do not cause DAG task failures — only persistent failures propagate.

### `incident_client.py`

Client for the beacon-core incident creation API. Called by alert DAGs to raise incidents. Handles authentication, deduplication checking against the `incident_alerts` PostgreSQL table, and response parsing.

---

## API Keys Required

The following external API credentials must be configured as Airflow Variables before the DAGs can run:

| Variable Name | Provider | Used By |
|---|---|---|
| `NTA_API_KEY` | NTA (National Transport Authority) | GTFS-R and static DAGs |
| `ESB_API_KEY` | ESB Networks | Power outage DAG |
| `TOMTOM_API_KEY` | TomTom | Traffic incidents + POI DAGs |
| `BEACON_CORE_API_KEY` | Internal | Alert DAGs (incident creation) |

Met Éireann, Irish Rail, Luas, waterlevel.ie, and Dublin Live provide open data that does not require authentication.

---

## Deployment

beacon-etl-flows is deployed to Kubernetes using the official Apache Airflow Helm chart with a CeleryExecutor configuration. This means:

- **Scheduler**: One Airflow scheduler pod manages DAG parsing and task scheduling
- **Webserver**: One Airflow webserver pod serves the management UI
- **Workers**: Celery workers scale out to handle concurrent task execution
- **Metadata DB**: Cloud SQL PostgreSQL stores Airflow's internal state (task history, DAG runs, variables, connections)
- **Result Backend**: External Redis stores Celery task results
- **DAG Sync**: A `git-sync` sidecar container on each scheduler and worker pod pulls the latest DAGs from this repository every 60 seconds. No redeployment is required to update a DAG — just push to the main branch.

### Deploy / Update

```bash
# Deploy or upgrade the Helm chart
helm upgrade --install beacon-airflow apache-airflow/airflow \
  -f deploy/values.yaml \
  --namespace airflow

# Check pod status
kubectl get pods -n airflow

# Trigger a DAG run manually (for testing)
kubectl exec -n airflow deployment/beacon-airflow-scheduler -- \
  airflow dags trigger gtfs_realtime_vehicle_positions_dag

# View DAG run logs
kubectl exec -n airflow deployment/beacon-airflow-scheduler -- \
  airflow tasks logs gtfs_realtime_vehicle_positions_dag extract 2026-04-09
```

### Airflow Variables (set via UI or CLI)

```bash
airflow variables set NTA_API_KEY "your-nta-api-key"
airflow variables set ESB_API_KEY "your-esb-api-key"
airflow variables set TOMTOM_API_KEY "your-tomtom-api-key"
airflow variables set CLICKHOUSE_HOST "clickhouse.internal"
airflow variables set CLICKHOUSE_PORT "8443"
airflow variables set BEACON_CORE_BASE_URL "https://beacon-tcd.tech"
```

---

## ClickHouse Schema

All tables are in the `external_data` database. SQL DDL files are in `sql/`:

| Table | Source DAG | Description |
|---|---|---|
| `vehicle_positions` | gtfs_realtime_vehicle_positions | NTA bus positions |
| `trip_updates` | gtfs_realtime_trip_updates | NTA arrival predictions |
| `luas_forecasts` | luas_forecast | Luas arrival times |
| `irish_rail_movements` | irish_rail | Train positions and punctuality |
| `weather_observations` | met_eireann | Dublin weather station readings |
| `weather_warnings` | weather_warnings | Met Éireann active warnings |
| `water_levels` | water_levels | River and coastal sensor readings |
| `power_outages` | esb_power_outages | Active ESB outages |
| `traffic_incidents` | tomtom_traffic_incidents | Road incidents across 10 cities |
| `news_articles` | dublin_live_news | Dublin Live news feed |
| `poi_locations` | poi_locations | Emergency POI reference data |

---

## Project Structure

```
beacon-etl-flows/
├── dags/
│   ├── etl/                     # 13 ETL ingestion DAGs
│   │   ├── gtfs_realtime_vehicle_positions_dag.py
│   │   ├── gtfs_realtime_trip_updates_dag.py
│   │   ├── luas_forecast_dag.py
│   │   ├── irish_rail_dag.py
│   │   ├── gtfs_static_dag.py
│   │   ├── met_eireann_dag.py
│   │   ├── weather_warnings_dag.py
│   │   ├── water_levels_dag.py
│   │   ├── esb_power_outages_dag.py
│   │   ├── tomtom_traffic_incidents_dag.py
│   │   ├── dublin_live_news_dag.py
│   │   └── poi_locations_dag.py
│   ├── alerts/                  # 5 alert/trigger DAGs
│   │   ├── alert_power_outages_dag.py
│   │   ├── alert_severe_weather_dag.py
│   │   ├── alert_traffic_incidents_dag.py
│   │   ├── alert_water_levels_dag.py
│   │   └── alert_weather_warnings_dag.py
│   ├── mock_reset/              # 4 mock data reset DAGs (dev/staging)
│   └── utils/
│       ├── config.py            # Airflow Variables reader
│       ├── clickhouse_client.py # ClickHouse write/query client
│       ├── http_client.py       # External API HTTP client with retry
│       └── incident_client.py   # beacon-core incident API client
├── sql/                         # ClickHouse DDL for all tables
├── deploy/
│   └── values.yaml              # Helm chart values for Kubernetes deployment
└── tests/                       # DAG unit tests
```

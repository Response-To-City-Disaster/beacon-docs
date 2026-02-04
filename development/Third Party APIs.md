# Third Party APIs

This docs will help you understand what third party APIs we will be using in Beacon Project.

1. Countries Cities and State Data

https://www.kaggle.com/datasets/tanweerulhaque/countries-states-cities-dataset

2. Maps SDK

https://docs.tomtom.com/maps-sdk-js/examples

3. Maps Routing API

https://docs.tomtom.com/routing-api/documentation/tomtom-maps/routing-service

4. Traffic Data

https://docs.tomtom.com/traffic-api/documentation/tomtom-maps/product-information/introduction

5. Public Transport Data

**Key Endpoints:**
```
GET https://api.nationaltransport.ie/gtfsr/v2/gtfsr?format=json
Header: x-api-key: YOUR_API_KEY
```

**Covers:** Dublin Bus, Bus Éireann, Go-Ahead Ireland (real-time)
**Irish Rail (Iarnród Éireann):**
```
http://api.irishrail.ie/realtime/realtime.asmx/getCurrentTrainsXML
```

Returns: Train latitude, longitude, status, delays, direction

6. City Traffic Camera Feed

https://www.earthcam.com/world/ireland/dublin/?cam=templebar go to inspect and check for dublin stream

7. Water Level Data

**Key Endpoints (Office of Public Works):**
```
http://waterlevel.ie/data/month/{station_id}_0001.csv
http://waterlevel.ie/data/month/{station_id}_OD.csv
http://waterlevel.ie/group/list/ (station groups)
```

**Update Frequency:** Every 15 minutes (don't poll more frequently)

**EPA HydroNet:**
```
https://www.epa.ie/hydronet/
- 1,000+ hydrometric stations
- Water levels, flow rates
- Open data for EPA-operated stations
```

9. Power Outage Data

**Key Finding:**
ESB Networks has a **PowerCheck API** used by their website, but it's not officially documented. A Home Assistant integration exists that reveals the API structure:

```
API Key: Found via browser inspection (Api-Subscription-Key)
Endpoint: ESB PowerCheck service
Returns: outageType, location, numCustAffected, estRestoreTime, statusMessage
```

**GitHub Resource:** `jasonmadigan/ha-esb-faults`

**Note:** Using this requires monitoring browser requests to extract the API key - not officially sanctioned for external use.

10. Meterological Data

**Met Éireann (Irish Meteorological Service):**
```
Forecast API: https://www.met.ie/Open_Data/Notes-on-API-XML-file_V8.odt
License: Met Éireann Open Data Custom License (CC BY 4.0-like)
```

**Open-Meteo (Recommended for global):**
```
https://api.open-meteo.com/v1/forecast
- No API key required
- Free for non-commercial use
- 1-11km resolution
```

**EPA Sonitus API (Air Quality + Noise):**
```
https://data.smartdublin.ie/dataset/sonitus
- Real-time air quality (PM2.5, PM10, NO2, O3)
- Noise level monitoring
- Requires registration
```

11. Hospital Data

> No API found we need to mock this

12. Police Data

> No API found we need to mock this

13. Fire Brigade Data

> No API found we need to mock this

14. Social Network Data

> APIs are expensive


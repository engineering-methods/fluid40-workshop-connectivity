# Hochfahren der Services
Reihenfolge einhalten wird empfohlen

## AAS Infrastructure (based on BaSyx Java)

Vorarbeit: Docker Network erzeugen

`docker network create fluid40-workshop-connectivity`

Info: Keine vorgeschaltete Security Layer -> Keine Authentifizierung notwendig

| Service | URL |
| -- | -- |
| Web UI | http://aasgui.basyx.localhost |
| AAS Env (Repositories) | http://aasenv.basyx.localhost |
| AAS Registry | http://aasreg.basyx.localhost |
| SM Registry | http://smreg.basyx.localhost |
| Discovery | http://discovery.basyx.localhost |

## Asset Connector

Swagger-API: http://localhost:8000/docs

`get-value`:
```json
{
  "Reference": {
    "type": "ModelReference",
    "keys": [
      {
        "type": "Submodel",
        "value": "https://fluid40.de/ids/sm/9792_8316_5239_9554"
      },
      {
        "type": "SubmodelElementCollection",
        "value": "Interface_MQTT_Cube"
      },
      {
        "type": "SubmodelElementCollection",
        "value": "InteractionMetadata"
      },
      {
        "type": "SubmodelElementCollection",
        "value": "properties"
      },
      {
        "type": "SubmodelElementCollection",
        "value": "LiveData"
      }
    ]
  }
}
```

Response:
```json
{
  "StatusCode": 200,
  "Message": "Successfully invoked `/get-value` with raw JSON in payload",
  "Payload": {
    "axis_1": 0.161,
    "axis_2": 0.951,
    "axis_3": 0.007,
    "threshold": 0.98,
    "reset": 0,
    "result": 0,
    "emission": 3.46,
    "timestamp": "2026-06-23T16:55:43.577726Z"
  },
  "Value": ""
}
```

## Data Mapping Processor

Swagger-API: http://localhost:3088/docs

Game starten, damit die Daten generiert werden und der Asset-Connector sich verbindet

## Influx V2 DB

Docker Volume erzeugen: `docker volume create fluid40-workshop-connectivity-influxv2-data` (sonst ist der Port nicht sichtbar und Influx auch nicht über localhost erreichbar)

Zugang unter http://localhost:8031/
Standard-Zugang aus compose.yml:

DOCKER_INFLUXDB_INIT_USERNAME: admin
DOCKER_INFLUXDB_INIT_PASSWORD: fluid40secure!

### API-Token erzeugen 
Alternative: Admin-Token aus compose-File benutzen

Name überlegen (nur für Lesbarkeit in Influx UI)
Option "Custom API Token" wählen mit folgenden Optionen:
- read + write für den angegebenen Bucket aus Compose.yml (Default: fluid40-bucket)
- read für "all other resources"

**wichtig: API Token kopieren und zwischenspeichern!**

## Database Connector
`docker load -i image.tar` aus dem database_connector Ordner ausführen

**Wichtig: Influx DB muss über den Port verfügbar sein, da der Database Connector sonst beim Start einen Fehler wirft**

# Zusammenspiel

Influx-Abfrage mit Alias-Mapping
```flux
from(bucket: "fluid40-bucket")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "play_4_in_a_row")
  |> filter(fn: (r) => r["_field"] == "https://fluid40.de/ids/sm/1704_4135_2769_8983/submodel-elements/Emission.CurrentEmission")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> map(fn: (r) => ({
     r with
    _field:
    if r["_field"] == "https://fluid40.de/ids/sm/1704_4135_2769_8983/submodel-elements/Emission.CurrentEmission" then "CurrentEmission"
    else r["_field"]    
    }))
```
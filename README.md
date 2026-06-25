# Hochfahren der Services
Reihenfolge einhalten wird empfohlen

## AAS Infrastructure (based on BaSyx Java)

Vorarbeit: Docker Network erzeugen

`docker network create fluid40-workshop-connectivity`

Info: Keine vorgeschaltete Security Layer -> Keine Authentifizierung notwendig

| Service | URL | Container-Adresse |
| -- | -- | -- |
| Web UI | http://aasgui.basyx.localhost | http://fluid40-aas-web-ui:3000 |
| AAS Env (Repositories) | http://aasenv.basyx.localhost | http://fluid40-aas-server:8081 |
| AAS Registry | http://aasreg.basyx.localhost | http://fluid40-aas-registry:8080 |
| SM Registry | http://smreg.basyx.localhost | http://fluid40-sm-registry:8080 |
| Discovery | http://discovery.basyx.localhost | http://fluid40-aas-discovery:8081 |

## Asset Connector

Swagger-API: http://localhost:8000/docs
Container-Adresse: http://fluid40-asset-connector:8000

Verbindung zu Asset Herstellen mit `add-config`:
```json
{
  "Aid": "BaSyx Web UI -> AAS Editor Modus -> AID Sorting Machine Optionen -> Copy Submodel as JSON"
}
```

Test Datenabfrage mit `get-value` (Beispiel JSON hier übernehmen):
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

Erwartete Antwort:
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
1. release.tar.gz herunterladen: https://github.com/fluid40/ms-database-connector/releases/tag/v0.1.1
2. entpacken (ggf. mehrfach) und image.tar in Ordner database_connector ablegen
3. `docker load -i image.tar` aus dem database_connector Ordner ausführen

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

# Dashboard in Grafana

Datenquelle hinzufügen:
- http://fluid40-influx-v2:8086 (hier wird der Container-Name und der originale Port benötigt)
- Auth: keine Option anhaken
- InfluxDB Details
  - Organization: fluid40-org
  - Token: API Token oder Admin-Token nutzen
  - Default Bucket: fluid40-bucket

Neues Dashboard erstellen
- Angelegte Datenquelle einstellen
- Flux-Query eingeben
- Query-Inspektor -> Apply



# Hochfahren der Services
- Reihenfolge einhalten wird empfohlen
- für das Hochfahren mit der Konsole in den entsprechenden Unterordner navigieren (z.B. `cd aas_infrastructure`) und `docker compose up -d` nutzen
- Überprüfung der Container in Docker Desktop möglich (Status, Logs, Ports)

## 1. AAS Infrastructure (mit BaSyx Java)

Vorarbeit: Docker Network erzeugen

`docker network create fluid40-workshop-connectivity`

Info: Keine vorgeschaltete Security Layer -> Keine Authentifizierung notwendig

| Service | URL | Container-Adresse |
| -- | -- | -- |
| Web UI | http://localhost:8078 | http://fluid40-aas-web-ui:3000 |
| AAS Env (Repositories) | http://localhost:8081 | http://fluid40-aas-server:8081 |
| AAS Registry | http://localhost:8076 | http://fluid40-aas-registry:8080 |
| SM Registry | http://localhost:8077 | http://fluid40-sm-registry:8080 |
| Discovery | http://localhost:8084 | http://fluid40-aas-discovery:8081 |

In der AAS WebUI im **Submodel Viewer** gerne nochmal das AID/AIMC Modell anschauen
Hinweis: Aufgrund des deaktivierten Proxies + lokalem Setup auf dem eigenen Computer wird der AAS-Viewer mit der Schalen-Auflistung nicht ganz korrekt funktionieren -> deswegen am besten den Submodel-Viewer nutzen

## 2. Asset Connector

- Swagger-API: http://localhost:8000/docs
- Container-Adresse: http://fluid40-asset-connector:8000
- Keine Konfigurationsdatei wie bei den anderen Microservices erforderlich (hier läuft alles über die REST API)
- Payload zur Konfiguration & Datenabfrage muss übereinstimmen ("Aid"/ "Reference"-Wrapper siehe Bsp. nicht vergessen)

Verbindung zu Asset Herstellen mit `add-config`:
```json
{
  "Aid": "BaSyx Web UI -> SM Editor Modus -> AID Sorting Machine Optionen -> Copy Submodel as JSON"
}
```

Game unter https://fluidon.com/en/Demo_Game_Fluid4.0_in_a_row_with_Cube starten, damit die Daten generiert werden und der Asset-Connector sich auf das Topic erfolgreich subscriben kann
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

## 3. Data Mapping Processor

- Swagger-API: http://localhost:3088/docs
- Container-Adresse: http://fluid40-data-mapping-processor:3088
- AAS-ID der Sortiermaschine (notwendig für Konfiguration): `https://fluid40.de/ids/aas/9911_6092_2508_3450`
- Game unter https://fluidon.com/en/Demo_Game_Fluid4.0_in_a_row_with_Cube starten, damit die Daten generiert werden und der Asset-Connector sich auf das Topic erfolgreich subscriben kann

## 4. Influx V2 DB

Docker Volume erzeugen: `docker volume create fluid40-workshop-connectivity-influxv2-data` (sonst ist der Port nicht sichtbar und Influx auch nicht über localhost erreichbar)

Zugang unter http://localhost:8031/

Standard-Zugangsdaten aus compose.yml:
- DOCKER_INFLUXDB_INIT_USERNAME: admin
- DOCKER_INFLUXDB_INIT_PASSWORD: fluid40secure!

Container-Name: http://fluid40-influx-v2:8086

### API-Token erzeugen 
**Alternative: Admin-Token aus compose-File benutzen**

Name überlegen (nur für Lesbarkeit in Influx UI)
Option "Custom API Token" wählen mit folgenden Optionen:
- read + write für den angegebenen Bucket aus Compose.yml (Default: fluid40-bucket)
- read für "all other resources"

**Wichtig: API Token kopieren und zwischenspeichern!**

## 4. Database Connector

1. release.tar.gz herunterladen und lokal speichern: https://github.com/fluid40/ms-database-connector/releases/tag/v0.1.1
2. In Kommandozeile (Windows, wenn es im normalen Dateisystem gespeichert wurde!) in Ordner navigieren, in dem die .tar.gz abelegt wurde
3. `tar -xzf release.tar.gz` -> Extrahiert die Datei image.tar
4. Lokales Image mit Docker laden: `docker load -i image.tar`
5. Image-Tag ist bereits im Docker-Compose fertig eingefügt (theoretisch hier das hinterlegte Image überprüfen & ggf. aktualisieren)

**Wichtig: Influx DB muss über den Port verfügbar sein, da der Database Connector sonst beim Start einen Fehler wirft**


Influx-Abfrage mit **Alias-Mapping** (da die Namen der Felder durch die Submodel-Element Pfade recht lang sind):
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

# 5. Dashboard in Grafana

Zugriff über http://localhost:9000

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



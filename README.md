# AAS Infrastructure (based on BaSyx Java)

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

# Asset Connector

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

# Data Mapping Processor

Swagger-API: http://localhost:3088/docs

Game starten, damit die Daten generiert werden und der Asset-Connector sich verbindet
flowchart LR
  HA[Home Assistant<br/>(MQTT Integrace)]
  MQ[MQTT broker]
  subgraph NR[Node-RED flow]
    P1[Poll<br/>GET /businessmodule/.../admins/{installationId}]
    P2[Per-device GET<br/>/iotmanagement/.../{id}/{id}/v1/content]
    PARSE[Parse JSON → °C<br/>hvac_mode, floor (bo/df)]
    PUB[Publish MQTT<br/>state/*, state/all, info/*, attributes]
    SUB[Subscribe MQTT<br/>fenix/+/set, set/mode, set/temperature]
    PUT[PUT config/replace<br/>(Dm/Cm/Ma/Sp)]
    VER[Verify GET<br/>.../content/{Dm|Cm|Ma|Sp} → ack/*]
  end

  HA <-->|state/all, state/*, availability, discovery| MQ
  MQ <-->|set/* commands| HA

  MQ --> SUB
  SUB --> PUT --> VER
  VER --> PUB
  PUB --> MQ

  NR -->|HTTP| API[(FENIX/WATTS API)]
  P1 --> API
  P2 --> API
  PUT --> API
  VER --> API

  API --> P2 --> PARSE --> PUB
  P1 --> P2

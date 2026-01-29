flowchart LR
  subgraph UI[Frontends]
    K[Kiosk UI\n(Auto-recepción)]
    P[Panel UI\n(Recepción/Supervisor)]
    D[Display UI\n(Pantalla pública)]
  end

  subgraph CORE[Backend / Core]
    API[API Core\n(REST + WS)]
    QS[Queue Service\n(Reglas + estados)]
    AUD[Audit Log\n(Trazabilidad)]
  end

  subgraph INTEG[Integraciones]
    HIS[HIS Adapter\n(HL7 v2 ADT)]
    PACS[PACS Adapter\n(Mock DICOM)]
    N8N[n8n\n(Automatización)]
  end

  DB[(DB\nSQLite/Postgres)]

  K -->|REST /checkin| API
  P -->|REST acciones| API
  D -->|GET snapshot| API

  API <--> |WebSocket eventos| K
  API <--> |WebSocket eventos| P
  API <--> |WebSocket eventos| D

  API --> QS
  QS --> DB
  QS --> AUD
  AUD --> DB

  API --> HIS
  API --> PACS

  HIS -->|ADT A04/A08 + ACK simulado| DB
  PACS -->|Study UID mock| DB

  API -->|webhook/event| N8N
  N8N -->|alertas/métricas/log| DB


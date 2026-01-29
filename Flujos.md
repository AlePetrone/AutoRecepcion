sequenceDiagram
  autonumber
  participant Patient as Paciente (Kiosk)
  participant Kiosk as Kiosk UI
  participant API as API Core
  participant QS as Queue Service
  participant DB as DB
  participant Display as Display UI
  participant Panel as Panel UI
  participant HIS as HIS Adapter (HL7)

  Patient->>Kiosk: Ingresa DNI + Servicio
  Kiosk->>API: POST /checkin
  API->>QS: createTicket(visit)
  QS->>DB: INSERT Visit + Ticket + AuditEvent
  QS-->>API: ticket(public_code=A023, state=WAITING)

  API-->>Kiosk: 200 OK (A023)
  API-->>Display: WS event ticket.created
  API-->>Panel: WS event ticket.created

  Panel->>API: POST /queue/{id}/call (station=Box2)
  API->>QS: transition(WAITING->CALLED)
  QS->>DB: UPDATE ticket + AuditEvent
  QS-->>API: ok(ticket CALLED)

  API-->>Display: WS event ticket.updated (A023 -> Box2)
  API-->>Panel: WS event ticket.updated
  API->>HIS: build HL7 ADT^A08 (state update)
  HIS->>DB: store HL7 msg + ACK(simulado)

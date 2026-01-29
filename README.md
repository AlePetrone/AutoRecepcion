# 🏥 Sistema de Auto-Recepción y Gestión de Cola

**Proyecto demostrativo de interoperabilidad en salud (HIS / PACS)**

## 1. Descripción general

Este proyecto implementa un **sistema de auto-recepción de pacientes y gestión de cola operativa** para centros de salud (clínicas, hospitales, centros de diagnóstico).

El foco **no** está en la agenda de turnos, sino en el **flujo real de llegada del paciente**, su registro operativo y la coordinación entre recepción, puestos de atención y pantallas públicas, con integración simulada a sistemas clínicos (HIS / PACS).

El objetivo del proyecto es ser:

* Realista desde el punto de vista operativo
* Defendible técnicamente en entrevistas
* Mostrable como pieza de arquitectura y diseño de sistemas

---

## 2. Alcance funcional (MVP)

### 2.1 Auto-recepción (Kiosk)

* Registro de llegada mediante identificador (ej. DNI)
* Selección de servicio / especialidad
* Generación de un **código público de atención** (ej. A023)
* El paciente ingresa automáticamente a la cola de espera

> No se exponen datos personales en pantallas públicas.

---

### 2.2 Gestión de cola (Panel interno)

Acciones operativas:

* Llamar paciente
* Marcar ausente
* Reintentar llamado
* Iniciar atención
* Finalizar atención

Control por roles:

* Recepcionista
* Supervisor

---

### 2.3 Pantalla pública

* Visualización de:

  * Código de turno
  * Puesto / consultorio asignado
* Actualización en tiempo real
* Sin datos identificatorios del paciente

---

## 3. Integración en salud (simulada)

### 3.1 HIS – HL7 v2

Se simula la integración con un HIS mediante la generación de mensajes HL7 v2:

* **ADT^A04** – Registro de llegada / visita
* **ADT^A08** – Actualización de estado de la visita

Los mensajes se generan correctamente a nivel estructural y se registra un ACK simulado.

---

### 3.2 PACS / DICOM (Mock)

Se simula un PACS mediante:

* Generación de un `StudyInstanceUID`
* Asociación del estudio a la visita clínica

No se implementa un PACS real: el objetivo es demostrar comprensión del **flujo clínico y de imágenes**, no montar infraestructura pesada.

---

## 4. Arquitectura (visión general)

Componentes principales:

* **Kiosk UI**: interfaz de auto-recepción
* **Panel UI**: gestión operativa de la cola
* **Display UI**: pantalla pública
* **API Core / Orquestador**

  * Gestión de cola
  * Validación de estados
  * Emisión de eventos
* **Queue Service**
* **HIS Adapter (HL7)**
* **PACS Adapter (mock)**
* **Event Bus (WebSockets)**

La arquitectura prioriza:

* Separación de responsabilidades
* Contratos explícitos
* Trazabilidad de eventos

---

## 5. Modelo de datos mínimo

### Patient (interno)

```text
patient_id (UUID)
doc_type
doc_number
full_name (opcional)
dob (opcional)
created_at
```

---

### Visit (episodio de llegada)

```text
visit_id (UUID)
patient_id
service
arrival_ts
external_his_ref (opcional)
notes
```

---

### QueueTicket (entidad operativa)

```text
ticket_id (UUID)
visit_id
public_code (ej. A023)
priority (NORMAL | URGENT | VIP)
state
assigned_station
attempts
called_ts
start_ts
end_ts
last_action_by
created_at
updated_at
```

---

### AuditEvent (trazabilidad)

```text
event_id (UUID)
timestamp
actor_type (SYSTEM | USER | KIOSK)
actor_id
entity_type
entity_id
action
payload (JSON)
correlation_id
```

---

## 6. Máquina de estados de la cola

Estados definidos:

* CREATED
* WAITING
* CALLED
* ABSENT
* IN_PROGRESS
* DONE
* CANCELLED

Transiciones principales:

* CREATED → WAITING
* WAITING → CALLED
* CALLED → IN_PROGRESS
* CALLED → ABSENT
* ABSENT → CALLED
* IN_PROGRESS → DONE

Restricciones:

* No se puede finalizar sin estar en atención
* Cada llamado incrementa el contador de intentos
* El puesto se asigna al llamar

---

## 7. Contratos de eventos (WebSocket)

Todos los eventos usan un envelope común:

```json
{
  "type": "queue.ticket.updated",
  "ts": "2026-01-29T19:12:33-03:00",
  "correlation_id": "uuid",
  "data": {}
}
```

Eventos principales:

* queue.ticket.created
* queue.ticket.updated
* queue.display.snapshot
* queue.audit.appended

---

## 8. API REST (mínimo)

### Kiosk

* `POST /checkin`

### Panel

* `GET /queue`
* `POST /queue/{id}/call`
* `POST /queue/{id}/absent`
* `POST /queue/{id}/start`
* `POST /queue/{id}/finish`
* `POST /queue/{id}/recall`

### Display

* `GET /display`
* WebSocket para actualizaciones

---

## 9. Seguridad y privacidad

* Separación entre identidad del paciente y código público
* No se exponen nombres en pantallas públicas
* Auditoría completa de acciones
* Control por roles

---

## 10. Automatización (n8n)

Se integra **n8n** para automatizar flujos operativos, por ejemplo:

* Paciente ausente → alerta
* Métricas de tiempos de espera
* Logs operativos
* Notificaciones internas

---

## 11. Objetivo del proyecto

Este proyecto busca demostrar competencias en:

* Diseño de sistemas distribuidos
* Interoperabilidad en salud (HL7 / DICOM)
* Gestión de estados y colas operativas
* Arquitectura orientada a eventos
* Trazabilidad y criterios de privacidad

No es un producto comercial, sino un **proyecto técnico demostrativo**.

---

## 12. Estado actual

* ✔ Diseño funcional
* ✔ Modelo de datos
* ✔ Máquina de estados
* ✔ Contratos definidos
* ⏳ Implementación del Core
* ⏳ Interfaces
* ⏳ Integraciones simuladas

---

## Arquitectura

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




## Flujo operativo principal

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

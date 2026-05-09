# TunnelSafetyDB - Diccionario de datos

Generado: 2026-05-09 17:42:42 UTC

Este documento describe la estructura definitiva de `TunnelSafetyDB`, sus tablas y sus atributos. El modelo está pensado para un sistema de decisión y gestión operativa de protocolos de seguridad en túneles.

## Resumen de tablas

| Tabla | Descripción |
|---|---|
| `tunnel.ActionDefinition` | Catálogo de acciones atómicas ejecutables o registrables: señalización, semáforos, cierre, ventilación, iluminación, megafonía, orden de trabajo, aviso externo, etc. |
| `tunnel.ActionExecution` | Acción concreta solicitada, enviada, ejecutada o fallida durante una ejecución de protocolo. |
| `tunnel.ActionTarget` | Equipos o activos afectados por una acción concreta. Permite una acción sobre múltiples objetivos. |
| `tunnel.Agency` | Organismo o entidad interna/externa que puede ser avisada o participar en la respuesta: emergencias, tráfico, mantenimiento, centro de control, etc. |
| `tunnel.Asset` | Activo físico o lógico instalado o supervisado: cámara, sensor, ventilador, luminaria, PMV, semáforo, sistema SOS, etc. |
| `tunnel.AssetInstallation` | Instalación de un activo en una localización concreta, con cobertura, orientación y relación con tubo si aplica. |
| `tunnel.AssetType` | Catálogo de tipos de equipamiento independientes de fabricante: CCTV, DAI, SCADA, ventilación, iluminación, PMV, semáforos, SOS, sensores, etc. |
| `tunnel.AuditLog` | Registro de auditoría legal/operativa: cambios, ejecuciones, overrides y acciones relevantes del usuario o del sistema. |
| `tunnel.CodeScheme` | Esquema de codificación de incidentes. Permite soportar códigos locales como 100-TRA o esquemas de otros países/operadores. |
| `tunnel.ContactPoint` | Punto de contacto de un organismo: teléfono, radio, email, SMS, endpoint API u otro canal. |
| `tunnel.ControlSystem` | Sistema de control o supervisión que gobierna o monitoriza activos: SCADA, CCTV/VMS, DAI, ATMS, BMS u otros. |
| `tunnel.DatabaseDocumentation` | Diccionario interno de la base de datos. Permite consultar desde SQL Server para qué sirve cada tabla y cada columna sin depender únicamente del script. |
| `tunnel.DetectionEvent` | Evidencia o evento de detección asociado a un incidente: sensor, CCTV/DAI, alarma SCADA, llamada SOS, aviso externo u observación del operador. |
| `tunnel.EmergencyLevel` | Nivel de gravedad o activación: prealerta, alerta, emergencia u otros niveles configurables. |
| `tunnel.Facility` | Ámbito operativo gestionado por una organización: red urbana, concesión, centro de control, autopista o instalación equivalente. |
| `tunnel.IncidentEvent` | Incidente real registrado en explotación. Une túnel, tipo de incidente, estado, tiempos, notas y datos operativos. |
| `tunnel.IncidentFamily` | Familia funcional de incidentes: tráfico, avería, incendio, ambiental, iluminación u otras. |
| `tunnel.IncidentLocation` | Relación N:M entre incidente y localización. Permite indicar varias ubicaciones, ubicación principal y grado de confianza. |
| `tunnel.IncidentType` | Tipo de incidente catalogado. Es la clasificación que permite seleccionar el protocolo adecuado. |
| `tunnel.Location` | Localización operativa dentro de un túnel: boca, tramo, punto kilométrico, sala técnica, salida de emergencia, poste SOS, cámara u otro punto relevante. |
| `tunnel.Notification` | Notificación real enviada o pendiente dentro de una ejecución de protocolo. |
| `tunnel.NotificationRule` | Regla o plantilla de aviso a organismos, centros de control, mantenimiento u otros contactos. |
| `tunnel.Organization` | Organización propietaria, gestora o concesionaria responsable de una o varias instalaciones o redes de túneles. |
| `tunnel.ParameterDefinition` | Definición global de un parámetro configurable: umbrales, límites, tiempos, modos o reglas reutilizables. |
| `tunnel.ParameterSet` | Colección de valores de parámetros aplicable a un túnel o a una versión de plan. Sirve para overrides y personalización. |
| `tunnel.ParameterValue` | Valor concreto de un parámetro dentro de un conjunto. Usa columnas tipadas para mantener compatibilidad y facilitar validación. |
| `tunnel.Plan` | Plan documental u operativo: PAU, Plan de Emergencia o conjunto de protocolos de explotación. |
| `tunnel.PlanTunnelScope` | Relación entre una versión de plan y los túneles cubiertos por ella. Permite que un plan cubra varios túneles y viceversa. |
| `tunnel.PlanVersion` | Versión concreta de un plan, con vigencia, aprobación y estado. Permite gestionar revisiones y trazabilidad normativa. |
| `tunnel.Protocol` | Plantilla de actuación asociada a una versión de plan y a un tipo de incidente. |
| `tunnel.ProtocolParameter` | Relación documental entre protocolo y parámetros que utiliza. Ayuda a validar configuración antes de ejecutar protocolos. |
| `tunnel.ProtocolRun` | Ejecución concreta de un protocolo para un incidente real. Guarda estado, usuario iniciador, paso actual y snapshot de contexto. |
| `tunnel.ProtocolStep` | Nodo del flujo de trabajo de un protocolo: decisión, acción, espera, información o checklist. |
| `tunnel.Role` | Rol operativo o administrativo de un usuario: operador, jefe de turno, mantenimiento, supervisor, etc. |
| `tunnel.StepAction` | Relación entre un paso de protocolo y una acción definida. Permite ejecutar varias acciones ordenadas por cada paso. |
| `tunnel.StepNotification` | Relación entre un paso de protocolo y una regla de notificación. Define cuándo y bajo qué condición se envía un aviso. |
| `tunnel.StepTransition` | Transición dirigida entre pasos de un protocolo. Permite modelar decisiones condicionales, ramas y rutas alternativas. |
| `tunnel.Tube` | Tubo, sentido o calzada interna de un túnel. Permite modelar túneles de uno o varios tubos y sentidos de circulación. |
| `tunnel.Tunnel` | Túnel físico individual. Representa la infraestructura principal y permite vincular tubos, zonas, localizaciones, planes e incidencias. |
| `tunnel.UserAccount` | Usuario de la aplicación o consola operativa. La autenticación puede estar en la aplicación, pero aquí queda la identidad operativa. |
| `tunnel.WorkOrder` | Orden de trabajo o mantenimiento asociada a un activo y opcionalmente generada desde un incidente. |
| `tunnel.Zone` | Sectorización interna de un tubo: zona operativa, compartimento de incendio, zona de evacuación, zona vulnerable o tramo de riesgo. |

## Detalle por tabla

### `tunnel.ActionDefinition`

Catálogo de acciones atómicas ejecutables o registrables: señalización, semáforos, cierre, ventilación, iluminación, megafonía, orden de trabajo, aviso externo, etc.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `action_definition_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la acción definida. |
| `action_type` | `NVARCHAR(30)` | Sí | No | `` | Tipo de acción: SET_SIGNAGE, SET_SEMAPHORE, CLOSE_TUBE, VENTILATION_MODE, LIGHTING_MODE, PA_ANNOUNCEMENT, CREATE_WORK_ORDER, REQUEST_EXTERNAL, LOG_ONLY u OTHER. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre operativo de la acción. |
| `description` | `NVARCHAR(400)` | No | No | `` | Descripción de qué hace la acción. |
| `payload_schema_json` | `NVARCHAR(MAX)` | No | No | `` | Esquema JSON de los parámetros necesarios para ejecutar la acción. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.ActionExecution`

Acción concreta solicitada, enviada, ejecutada o fallida durante una ejecución de protocolo.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `action_execution_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la ejecución de acción. |
| `protocol_run_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Ejecución de protocolo a la que pertenece. |
| `action_definition_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Acción definida que se ejecuta. |
| `step_id` | `UNIQUEIDENTIFIER` | No | No | `` | Paso que disparó la acción. |
| `requested_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha/hora UTC en que se solicitó la acción. |
| `executed_at_utc` | `DATETIME2(3)` | No | No | `` | Fecha/hora UTC en que se ejecutó o se confirmó. |
| `exec_status` | `NVARCHAR(12)` | Sí | No | `('REQUESTED')` | Estado: REQUESTED, SENT, SUCCESS, FAILED o CANCELLED. |
| `payload_json` | `NVARCHAR(MAX)` | Sí | No | `` | Payload JSON real enviado o registrado para ejecutar la acción. |
| `result_json` | `NVARCHAR(MAX)` | No | No | `` | Resultado JSON devuelto por el sistema o integración. |
| `error_message` | `NVARCHAR(400)` | No | No | `` | Mensaje de error si la acción falla. |

### `tunnel.ActionTarget`

Equipos o activos afectados por una acción concreta. Permite una acción sobre múltiples objetivos.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `action_target_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del objetivo de acción. |
| `action_execution_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Ejecución de acción que afecta al activo. |
| `asset_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Activo objetivo de la acción. |
| `target_role` | `NVARCHAR(200)` | No | No | `` | Papel del activo en la acción, por ejemplo PMV entrada o ventilador sector 2. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Agency`

Organismo o entidad interna/externa que puede ser avisada o participar en la respuesta: emergencias, tráfico, mantenimiento, centro de control, etc.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `agency_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del organismo. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre del organismo o entidad. |
| `agency_type` | `NVARCHAR(30)` | Sí | No | `` | Tipo: EMERGENCY_SERVICES, TRAFFIC_AUTHORITY, CONTROL_CENTER, MAINTENANCE u OTHER. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Asset`

Activo físico o lógico instalado o supervisado: cámara, sensor, ventilador, luminaria, PMV, semáforo, sistema SOS, etc.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `asset_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del activo. |
| `asset_type_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Tipo de activo al que pertenece. |
| `control_system_id` | `UNIQUEIDENTIFIER` | No | No | `` | Sistema de control que supervisa o controla el activo. |
| `asset_tag` | `NVARCHAR(120)` | Sí | No | `` | Etiqueta de inventario única del activo. |
| `manufacturer` | `NVARCHAR(120)` | No | No | `` | Fabricante del activo. |
| `model` | `NVARCHAR(120)` | No | No | `` | Modelo del activo. |
| `serial_number` | `NVARCHAR(120)` | No | No | `` | Número de serie. |
| `criticality` | `NVARCHAR(20)` | Sí | No | `('MEDIUM')` | Criticidad operativa: LOW, MEDIUM, HIGH o SAFETY_CRITICAL. |
| `asset_status` | `NVARCHAR(20)` | Sí | No | `('OK')` | Estado del activo: OK, DEGRADED, FAILED o MAINTENANCE. |
| `last_healthcheck_at_utc` | `DATETIME2(3)` | No | No | `` | Última fecha/hora UTC de comprobación, telemetría o heartbeat. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.AssetInstallation`

Instalación de un activo en una localización concreta, con cobertura, orientación y relación con tubo si aplica.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `asset_installation_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la instalación del activo. |
| `asset_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Activo instalado. |
| `location_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Localización donde está instalado el activo. |
| `tube_id` | `UNIQUEIDENTIFIER` | No | No | `` | Tubo asociado a la instalación si aplica. |
| `installed_at` | `DATE` | No | No | `` | Fecha de instalación. |
| `coverage_start_chainage_m` | `DECIMAL(12,3)` | No | No | `` | Inicio de cobertura del activo en metros. |
| `coverage_end_chainage_m` | `DECIMAL(12,3)` | No | No | `` | Fin de cobertura del activo en metros. |
| `orientation` | `NVARCHAR(80)` | No | No | `` | Orientación física o lógica del activo. |
| `is_primary` | `BIT` | Sí | No | `(0)` | Indica si es la instalación principal del activo. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.AssetType`

Catálogo de tipos de equipamiento independientes de fabricante: CCTV, DAI, SCADA, ventilación, iluminación, PMV, semáforos, SOS, sensores, etc.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `asset_type_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del tipo de activo. |
| `category` | `NVARCHAR(30)` | Sí | No | `` | Categoría funcional normalizada del activo. |
| `name` | `NVARCHAR(120)` | Sí | No | `` | Nombre concreto del tipo de activo. |
| `vendor_independent` | `BIT` | Sí | No | `(1)` | Indica si el tipo es independiente de fabricante. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.AuditLog`

Registro de auditoría legal/operativa: cambios, ejecuciones, overrides y acciones relevantes del usuario o del sistema.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `audit_log_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del evento de auditoría. |
| `user_id` | `UNIQUEIDENTIFIER` | No | No | `` | Usuario que realiza la acción, si aplica. |
| `incident_event_id` | `UNIQUEIDENTIFIER` | No | No | `` | Incidente relacionado con la acción auditada, si aplica. |
| `entity_type` | `NVARCHAR(60)` | Sí | No | `` | Tipo de entidad afectada, por ejemplo ProtocolRun, Asset o IncidentEvent. |
| `entity_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Identificador de la entidad afectada. |
| `action` | `NVARCHAR(40)` | Sí | No | `` | Acción realizada: CREATE, UPDATE, EXECUTE, OVERRIDE, etc. |
| `timestamp_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha/hora UTC del evento auditado. |
| `details_json` | `NVARCHAR(MAX)` | No | No | `` | Detalles de auditoría en JSON. |

### `tunnel.CodeScheme`

Esquema de codificación de incidentes. Permite soportar códigos locales como 100-TRA o esquemas de otros países/operadores.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `code_scheme_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del esquema de códigos. |
| `name` | `NVARCHAR(120)` | Sí | No | `` | Nombre del esquema de codificación. |
| `description` | `NVARCHAR(400)` | No | No | `` | Descripción funcional del esquema. |
| `pattern_hint` | `NVARCHAR(200)` | No | No | `` | Pista de patrón o formato, por ejemplo una regex o estructura esperada. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.ContactPoint`

Punto de contacto de un organismo: teléfono, radio, email, SMS, endpoint API u otro canal.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `contact_point_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del punto de contacto. |
| `agency_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Organismo al que pertenece el contacto. |
| `name` | `NVARCHAR(200)` | No | No | `` | Nombre o etiqueta del contacto. |
| `channel` | `NVARCHAR(12)` | Sí | No | `` | Canal: PHONE, EMAIL, RADIO, API, SMS u OTHER. |
| `address` | `NVARCHAR(400)` | Sí | No | `` | Valor del contacto: teléfono, email, endpoint, canal de radio, etc. |
| `availability` | `NVARCHAR(200)` | No | No | `` | Disponibilidad horaria o condición de uso. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.ControlSystem`

Sistema de control o supervisión que gobierna o monitoriza activos: SCADA, CCTV/VMS, DAI, ATMS, BMS u otros.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `control_system_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del sistema de control. |
| `facility_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Ámbito operativo al que pertenece el sistema. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre operativo del sistema. |
| `system_type` | `NVARCHAR(20)` | Sí | No | `` | Tipo de sistema: SCADA, VMS_CCTV, DAI, ATMS, BMS u OTHER. |
| `primary_site` | `NVARCHAR(200)` | No | No | `` | Ubicación principal del sistema si aplica. |
| `has_backup` | `BIT` | Sí | No | `(0)` | Indica si dispone de sistema o sala de respaldo. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.DatabaseDocumentation`

Diccionario interno de la base de datos. Permite consultar desde SQL Server para qué sirve cada tabla y cada columna sin depender únicamente del script.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `documentation_id` | `INT IDENTITY(1,1)` | Sí | Sí | `` | Identificador incremental de cada entrada de documentación. |
| `object_type` | `NVARCHAR(20)` | Sí | No | `` | Tipo de objeto documentado. Valores previstos: TABLE o COLUMN. |
| `table_name` | `NVARCHAR(128)` | Sí | No | `` | Nombre de la tabla documentada. |
| `column_name` | `NVARCHAR(128)` | No | No | `` | Nombre de la columna documentada. Es NULL cuando se documenta una tabla completa. |
| `description` | `NVARCHAR(MAX)` | Sí | No | `` | Texto descriptivo del objeto documentado. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha y hora UTC en la que se creó la entrada documental. |

### `tunnel.DetectionEvent`

Evidencia o evento de detección asociado a un incidente: sensor, CCTV/DAI, alarma SCADA, llamada SOS, aviso externo u observación del operador.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `detection_event_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del evento de detección. |
| `incident_event_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Incidente al que pertenece la evidencia. |
| `source_type` | `NVARCHAR(20)` | Sí | No | `` | Origen: SENSOR, CCTV_DAI, SCADA_ALARM, SOS_CALL, EXTERNAL_CALL u OPERATOR_OBS. |
| `source_asset_id` | `UNIQUEIDENTIFIER` | No | No | `` | Activo que generó la detección, si aplica. |
| `reported_by` | `NVARCHAR(200)` | No | No | `` | Persona, servicio, centro o sistema que reporta la detección. |
| `payload_json` | `NVARCHAR(MAX)` | No | No | `` | Datos de detección en JSON: medidas, alarmas, valores, texto, etc. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha/hora UTC de registro de la detección. |

### `tunnel.EmergencyLevel`

Nivel de gravedad o activación: prealerta, alerta, emergencia u otros niveles configurables.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `emergency_level_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del nivel de emergencia. |
| `name` | `NVARCHAR(30)` | Sí | No | `` | Nombre del nivel. |
| `rank` | `INT` | Sí | No | `` | Orden de gravedad o prioridad. A mayor valor, mayor nivel si así se configura. |
| `description` | `NVARCHAR(400)` | No | No | `` | Descripción operativa del nivel. |

### `tunnel.Facility`

Ámbito operativo gestionado por una organización: red urbana, concesión, centro de control, autopista o instalación equivalente.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `facility_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del ámbito o instalación. |
| `organization_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Organización propietaria o gestora de la instalación. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre operativo de la instalación, red o centro de control. |
| `facility_type` | `NVARCHAR(40)` | Sí | No | `` | Tipo de instalación: CITY_NETWORK, HIGHWAY_CONCESSION, SINGLE_TUNNEL o CONTROL_CENTER_SCOPE. |
| `address` | `NVARCHAR(400)` | No | No | `` | Dirección física o descripción de ubicación si procede. |
| `timezone` | `NVARCHAR(64)` | Sí | No | `` | Zona horaria propia del ámbito operativo. |
| `contact_phone` | `NVARCHAR(50)` | No | No | `` | Teléfono general de contacto. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### `tunnel.IncidentEvent`

Incidente real registrado en explotación. Une túnel, tipo de incidente, estado, tiempos, notas y datos operativos.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `incident_event_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del incidente real. |
| `incident_type_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Tipo de incidente catalogado. |
| `tunnel_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Túnel donde ocurre el incidente. |
| `incident_status` | `NVARCHAR(12)` | Sí | No | `('OPEN')` | Estado del incidente: OPEN, MITIGATING, RESOLVED o CLOSED. |
| `started_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha/hora UTC de inicio o apertura del incidente. |
| `detected_at_utc` | `DATETIME2(3)` | No | No | `` | Fecha/hora UTC de detección si difiere de la apertura. |
| `resolved_at_utc` | `DATETIME2(3)` | No | No | `` | Fecha/hora UTC de resolución. |
| `severity_override` | `NVARCHAR(30)` | No | No | `` | Reclasificación manual de severidad si el operador la aplica. |
| `summary` | `NVARCHAR(400)` | No | No | `` | Resumen corto del incidente. |
| `operator_notes` | `NVARCHAR(MAX)` | No | No | `` | Notas libres del operador. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.IncidentFamily`

Familia funcional de incidentes: tráfico, avería, incendio, ambiental, iluminación u otras.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `incident_family_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la familia. |
| `code` | `NVARCHAR(20)` | Sí | No | `` | Código corto de familia, por ejemplo TRA, AVA, FOC, AMB o ILI. |
| `name` | `NVARCHAR(120)` | Sí | No | `` | Nombre de la familia. |
| `description` | `NVARCHAR(400)` | No | No | `` | Descripción de los incidentes incluidos en la familia. |

### `tunnel.IncidentLocation`

Relación N:M entre incidente y localización. Permite indicar varias ubicaciones, ubicación principal y grado de confianza.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `incident_location_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la relación incidente-localización. |
| `incident_event_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Incidente localizado. |
| `location_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Localización asociada al incidente. |
| `confidence` | `DECIMAL(4,3)` | Sí | No | `(1.000)` | Confianza de la localización entre 0 y 1. |
| `is_primary` | `BIT` | Sí | No | `(0)` | Indica si es la localización principal del incidente. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.IncidentType`

Tipo de incidente catalogado. Es la clasificación que permite seleccionar el protocolo adecuado.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `incident_type_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del tipo de incidente. |
| `code_scheme_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Esquema de codificación al que pertenece el código. |
| `emergency_level_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Nivel de emergencia asociado al tipo. |
| `incident_family_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Familia funcional del incidente. |
| `code_raw` | `NVARCHAR(40)` | Sí | No | `` | Código textual del incidente, por ejemplo 260-AVA. |
| `title` | `NVARCHAR(200)` | Sí | No | `` | Título operativo del incidente. |
| `description` | `NVARCHAR(MAX)` | Sí | No | `` | Descripción completa del tipo de incidente. |
| `detection_notes` | `NVARCHAR(MAX)` | No | No | `` | Notas sobre cómo detectar, verificar o confirmar el incidente. |
| `operational_context` | `NVARCHAR(20)` | Sí | No | `('NORMAL')` | Contexto: NORMAL, WORKS, EVENT u OTHER. |
| `info_to_collect_json` | `NVARCHAR(MAX)` | No | No | `` | Campos o checklist que el operador debe recopilar, en JSON. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Location`

Localización operativa dentro de un túnel: boca, tramo, punto kilométrico, sala técnica, salida de emergencia, poste SOS, cámara u otro punto relevante.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `location_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la localización. |
| `tunnel_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Túnel al que pertenece la localización. |
| `zone_id` | `UNIQUEIDENTIFIER` | No | No | `` | Zona a la que pertenece la localización si aplica. |
| `location_type` | `NVARCHAR(30)` | Sí | No | `` | Tipo: PORTAL, SEGMENT, LANE_POINT, TECH_ROOM, CROSS_PASSAGE, EMERGENCY_EXIT, SOS_POST, CAMERA_POLE u OTHER. |
| `name` | `NVARCHAR(200)` | No | No | `` | Nombre o etiqueta de la localización. |
| `chainage_m` | `DECIMAL(12,3)` | No | No | `` | Progresiva o referencia lineal en metros. |
| `geom` | `GEOGRAPHY` | No | No | `` | Coordenada geográfica opcional. Normalmente SRID 4326. |
| `access_description` | `NVARCHAR(500)` | No | No | `` | Descripción de acceso para operadores o ayuda externa. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Notification`

Notificación real enviada o pendiente dentro de una ejecución de protocolo.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `notification_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la notificación. |
| `protocol_run_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Ejecución de protocolo que origina la notificación. |
| `contact_point_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Punto de contacto destinatario. |
| `notification_rule_id` | `UNIQUEIDENTIFIER` | No | No | `` | Regla de notificación aplicada, si procede. |
| `sent_at_utc` | `DATETIME2(3)` | No | No | `` | Fecha/hora UTC de envío. |
| `notif_status` | `NVARCHAR(10)` | Sí | No | `('PENDING')` | Estado: PENDING, SENT, FAILED o ACKED. |
| `message` | `NVARCHAR(MAX)` | Sí | No | `` | Mensaje final enviado o preparado. |
| `ack_at_utc` | `DATETIME2(3)` | No | No | `` | Fecha/hora UTC de acuse o confirmación. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.NotificationRule`

Regla o plantilla de aviso a organismos, centros de control, mantenimiento u otros contactos.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `notification_rule_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la regla de notificación. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre de la regla. |
| `purpose` | `NVARCHAR(400)` | No | No | `` | Finalidad del aviso. |
| `default_channel` | `NVARCHAR(12)` | Sí | No | `` | Canal por defecto: PHONE, EMAIL, RADIO, API, SMS u OTHER. |
| `message_template` | `NVARCHAR(MAX)` | Sí | No | `` | Plantilla del mensaje con variables sustituibles por la aplicación. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Organization`

Organización propietaria, gestora o concesionaria responsable de una o varias instalaciones o redes de túneles.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `organization_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la organización. Se usa como clave primaria y no debe cambiar. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre oficial o operativo de la organización. |
| `legal_id` | `NVARCHAR(100)` | No | No | `` | Identificador legal/fiscal si aplica, por ejemplo CIF, NIF o identificador administrativo. |
| `country_code` | `NVARCHAR(3)` | Sí | No | `` | Código de país ISO-3166 alfa-3, por ejemplo ESP. |
| `timezone_default` | `NVARCHAR(64)` | Sí | No | `` | Zona horaria por defecto de la organización o ámbito principal. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Campo JSON extensible para datos adicionales no normalizados. |

### `tunnel.ParameterDefinition`

Definición global de un parámetro configurable: umbrales, límites, tiempos, modos o reglas reutilizables.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `parameter_definition_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la definición de parámetro. |
| `key` | `NVARCHAR(120)` | Sí | No | `` | Clave única del parámetro, estable para uso en código y configuración. |
| `data_type` | `NVARCHAR(10)` | Sí | No | `` | Tipo de dato esperado: INT, DECIMAL, BOOLEAN, TEXT o JSON. |
| `unit` | `NVARCHAR(32)` | No | No | `` | Unidad del parámetro si aplica, por ejemplo segundos, ppm, km/h o porcentaje. |
| `description` | `NVARCHAR(400)` | Sí | No | `` | Descripción funcional del parámetro. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.ParameterSet`

Colección de valores de parámetros aplicable a un túnel o a una versión de plan. Sirve para overrides y personalización.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `parameter_set_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del conjunto de parámetros. |
| `tunnel_id` | `UNIQUEIDENTIFIER` | No | No | `` | Túnel al que aplica el conjunto, si es un set por túnel. |
| `plan_version_id` | `UNIQUEIDENTIFIER` | No | No | `` | Versión de plan a la que aplica el conjunto, si es un set por plan. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre del conjunto de parámetros. |
| `priority` | `INT` | Sí | No | `(0)` | Prioridad para resolver overrides cuando hay varios conjuntos aplicables. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### `tunnel.ParameterValue`

Valor concreto de un parámetro dentro de un conjunto. Usa columnas tipadas para mantener compatibilidad y facilitar validación.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `parameter_value_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del valor de parámetro. |
| `parameter_set_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Conjunto de parámetros al que pertenece el valor. |
| `parameter_definition_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Definición de parámetro que se está valorando. |
| `value_int` | `INT` | No | No | `` | Valor entero si el parámetro es de tipo INT. |
| `value_decimal` | `DECIMAL(18,6)` | No | No | `` | Valor decimal si el parámetro es de tipo DECIMAL. |
| `value_bool` | `BIT` | No | No | `` | Valor booleano si el parámetro es de tipo BOOLEAN. |
| `value_text` | `NVARCHAR(MAX)` | No | No | `` | Valor textual si el parámetro es de tipo TEXT. |
| `value_json` | `NVARCHAR(MAX)` | No | No | `` | Valor JSON si el parámetro es de tipo JSON. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Plan`

Plan documental u operativo: PAU, Plan de Emergencia o conjunto de protocolos de explotación.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `plan_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del plan. |
| `facility_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Ámbito operativo al que pertenece el plan. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre del plan. |
| `plan_type` | `NVARCHAR(30)` | Sí | No | `` | Tipo de plan: PAU, EMERGENCY_PLAN u OPERATIONS_PROTOCOLS. |
| `authority` | `NVARCHAR(200)` | No | No | `` | Autoridad, organismo o área responsable del plan. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### `tunnel.PlanTunnelScope`

Relación entre una versión de plan y los túneles cubiertos por ella. Permite que un plan cubra varios túneles y viceversa.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `plan_tunnel_scope_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la relación de alcance. |
| `plan_version_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Versión de plan aplicable. |
| `tunnel_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Túnel cubierto por la versión del plan. |
| `scope_note` | `NVARCHAR(400)` | No | No | `` | Nota de alcance, por ejemplo fase de obras, tramo parcial o condición especial. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.PlanVersion`

Versión concreta de un plan, con vigencia, aprobación y estado. Permite gestionar revisiones y trazabilidad normativa.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `plan_version_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la versión del plan. |
| `plan_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Plan al que pertenece la versión. |
| `version_label` | `NVARCHAR(50)` | Sí | No | `` | Etiqueta de versión, por ejemplo v1.0, v1.1 o 2025.03. |
| `effective_from` | `DATE` | No | No | `` | Fecha de inicio de vigencia. |
| `effective_to` | `DATE` | No | No | `` | Fecha de fin de vigencia. |
| `version_status` | `NVARCHAR(12)` | Sí | No | `('DRAFT')` | Estado de versión: DRAFT, ACTIVE o RETIRED. |
| `approved_by` | `NVARCHAR(200)` | No | No | `` | Persona, área u organismo que aprueba la versión. |
| `approval_date` | `DATE` | No | No | `` | Fecha de aprobación. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### `tunnel.Protocol`

Plantilla de actuación asociada a una versión de plan y a un tipo de incidente.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `protocol_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del protocolo. |
| `plan_version_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Versión del plan que define el protocolo. |
| `incident_type_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Tipo de incidente gestionado por el protocolo. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre operativo del protocolo. |
| `objective` | `NVARCHAR(400)` | No | No | `` | Objetivo resumido del protocolo. |
| `is_remote_executable` | `BIT` | Sí | No | `(1)` | Indica si puede ejecutarse desde centro de control o consola remota. |
| `protocol_status` | `NVARCHAR(12)` | Sí | No | `('ACTIVE')` | Estado del protocolo: ACTIVE, RETIRED o DRAFT. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.ProtocolParameter`

Relación documental entre protocolo y parámetros que utiliza. Ayuda a validar configuración antes de ejecutar protocolos.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `protocol_parameter_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la relación protocolo-parámetro. |
| `protocol_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Protocolo que utiliza el parámetro. |
| `parameter_definition_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Parámetro requerido o utilizado por el protocolo. |
| `usage_note` | `NVARCHAR(400)` | No | No | `` | Nota que explica cómo usa el protocolo este parámetro. |

### `tunnel.ProtocolRun`

Ejecución concreta de un protocolo para un incidente real. Guarda estado, usuario iniciador, paso actual y snapshot de contexto.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `protocol_run_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la ejecución del protocolo. |
| `incident_event_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Incidente gestionado por esta ejecución. |
| `protocol_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Protocolo que se está ejecutando. |
| `started_by_user_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Usuario que inició la ejecución. |
| `started_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha/hora UTC de inicio de la ejecución. |
| `ended_at_utc` | `DATETIME2(3)` | No | No | `` | Fecha/hora UTC de finalización. |
| `run_status` | `NVARCHAR(12)` | Sí | No | `('RUNNING')` | Estado de ejecución: RUNNING, PAUSED, COMPLETED o ABORTED. |
| `current_step_id` | `UNIQUEIDENTIFIER` | No | No | `` | Paso actual dentro del protocolo. |
| `context_snapshot_json` | `NVARCHAR(MAX)` | No | No | `` | Snapshot JSON de contexto: parámetros resueltos, estado de activos, datos del incidente, etc. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.ProtocolStep`

Nodo del flujo de trabajo de un protocolo: decisión, acción, espera, información o checklist.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `protocol_step_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del paso. |
| `protocol_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Protocolo al que pertenece el paso. |
| `step_key` | `NVARCHAR(80)` | Sí | No | `` | Clave estable del paso para referenciarlo desde configuración o interfaz. |
| `step_type` | `NVARCHAR(12)` | Sí | No | `` | Tipo de paso: DECISION, ACTION, INFO, WAIT o CHECKLIST. |
| `title` | `NVARCHAR(200)` | Sí | No | `` | Título visible del paso. |
| `instructions` | `NVARCHAR(MAX)` | Sí | No | `` | Instrucciones operativas que debe seguir el operador o el sistema. |
| `requires_ack` | `BIT` | Sí | No | `(1)` | Indica si el operador debe confirmar el paso. |
| `timeout_seconds` | `INT` | No | No | `` | Tiempo máximo recomendado antes de escalar o disparar otra transición. |
| `ui_form_schema_json` | `NVARCHAR(MAX)` | No | No | `` | Esquema JSON para pintar formularios dinámicos en la interfaz. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Role`

Rol operativo o administrativo de un usuario: operador, jefe de turno, mantenimiento, supervisor, etc.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `role_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del rol. |
| `name` | `NVARCHAR(80)` | Sí | No | `` | Nombre único del rol. |
| `description` | `NVARCHAR(400)` | No | No | `` | Descripción de permisos o responsabilidades del rol. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.StepAction`

Relación entre un paso de protocolo y una acción definida. Permite ejecutar varias acciones ordenadas por cada paso.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `step_action_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la relación paso-acción. |
| `protocol_step_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Paso de protocolo que dispara la acción. |
| `action_definition_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Acción definida que se debe ejecutar o registrar. |
| `execution_order` | `INT` | Sí | No | `(0)` | Orden de ejecución dentro del paso. |
| `is_mandatory` | `BIT` | Sí | No | `(1)` | Indica si la acción es obligatoria en el paso. |
| `parameter_binding_json` | `NVARCHAR(MAX)` | No | No | `` | Mapeo JSON entre parámetros del protocolo/incidente y payload de acción. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.StepNotification`

Relación entre un paso de protocolo y una regla de notificación. Define cuándo y bajo qué condición se envía un aviso.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `step_notification_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la relación paso-notificación. |
| `protocol_step_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Paso de protocolo que dispara la notificación. |
| `notification_rule_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Regla de notificación a aplicar. |
| `when` | `NVARCHAR(12)` | Sí | No | `` | Momento de disparo: ON_ENTER, ON_EXIT, ON_TIMEOUT u ON_CONDITION. |
| `condition_expr` | `NVARCHAR(MAX)` | No | No | `` | Condición adicional para disparar el aviso. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.StepTransition`

Transición dirigida entre pasos de un protocolo. Permite modelar decisiones condicionales, ramas y rutas alternativas.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `step_transition_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la transición. |
| `protocol_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Protocolo al que pertenece la transición. |
| `from_step_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Paso origen. |
| `to_step_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Paso destino. |
| `condition_expr` | `NVARCHAR(MAX)` | No | No | `` | Expresión de condición o regla de negocio. Si es NULL puede interpretarse como transición por defecto. |
| `priority` | `INT` | Sí | No | `(0)` | Prioridad de evaluación cuando hay varias transiciones posibles. |
| `label` | `NVARCHAR(200)` | No | No | `` | Texto visible para la transición, por ejemplo Sí, No, Confirmado o Escalar. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Tube`

Tubo, sentido o calzada interna de un túnel. Permite modelar túneles de uno o varios tubos y sentidos de circulación.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `tube_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del tubo o sentido. |
| `tunnel_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Túnel al que pertenece el tubo. |
| `name` | `NVARCHAR(120)` | Sí | No | `` | Nombre del tubo o sentido, por ejemplo Besòs, Llobregat, Ascendente o Descendente. |
| `direction` | `NVARCHAR(16)` | Sí | No | `` | Dirección normalizada: N, S, E, W, A_TO_B, B_TO_A o BIDIR. |
| `lanes_count` | `INT` | Sí | No | `` | Número de carriles del tubo. |
| `speed_limit_kmh` | `INT` | No | No | `` | Velocidad máxima autorizada en km/h. |
| `gradient_percent` | `DECIMAL(6,3)` | No | No | `` | Pendiente media o relevante expresada en porcentaje. |
| `cross_section_type` | `NVARCHAR(80)` | No | No | `` | Tipo de sección o configuración geométrica. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Tunnel`

Túnel físico individual. Representa la infraestructura principal y permite vincular tubos, zonas, localizaciones, planes e incidencias.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `tunnel_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del túnel. |
| `facility_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Ámbito operativo al que pertenece el túnel. |
| `name` | `NVARCHAR(200)` | Sí | No | `` | Nombre oficial u operativo del túnel. |
| `local_code` | `NVARCHAR(80)` | No | No | `` | Código interno/local del túnel. |
| `road_name` | `NVARCHAR(120)` | No | No | `` | Carretera, ronda, vía o eje viario asociado. |
| `country_code` | `NVARCHAR(3)` | Sí | No | `` | Código de país ISO-3166 alfa-3. |
| `city` | `NVARCHAR(120)` | No | No | `` | Ciudad o área territorial. |
| `length_m` | `DECIMAL(12,3)` | No | No | `` | Longitud aproximada del túnel en metros. |
| `has_bidirectional_tubes` | `BIT` | Sí | No | `(0)` | Indica si el túnel tiene tubos o sentidos bidireccionales. |
| `commissioning_date` | `DATE` | No | No | `` | Fecha de puesta en servicio. |
| `tunnel_status` | `NVARCHAR(20)` | Sí | No | `('ACTIVE')` | Estado operativo del túnel: ACTIVE, WORKS o DECOMMISSIONED. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales como normativa, restricciones, notas geométricas o configuración local. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### `tunnel.UserAccount`

Usuario de la aplicación o consola operativa. La autenticación puede estar en la aplicación, pero aquí queda la identidad operativa.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `user_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único del usuario. |
| `facility_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Ámbito operativo al que pertenece el usuario. |
| `role_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Rol asignado al usuario. |
| `username` | `NVARCHAR(120)` | Sí | No | `` | Nombre de usuario o login. |
| `display_name` | `NVARCHAR(200)` | Sí | No | `` | Nombre visible del usuario. |
| `phone` | `NVARCHAR(50)` | No | No | `` | Teléfono de contacto. |
| `email` | `NVARCHAR(200)` | No | No | `` | Correo electrónico. |
| `is_active` | `BIT` | Sí | No | `(1)` | Indica si el usuario está activo. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### `tunnel.WorkOrder`

Orden de trabajo o mantenimiento asociada a un activo y opcionalmente generada desde un incidente.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `work_order_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la orden de trabajo. |
| `asset_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Activo afectado por la orden. |
| `created_from_incident_id` | `UNIQUEIDENTIFIER` | No | No | `` | Incidente que originó la orden si aplica. |
| `priority` | `NVARCHAR(10)` | Sí | No | `('MEDIUM')` | Prioridad: LOW, MEDIUM, HIGH o URGENT. |
| `work_status` | `NVARCHAR(12)` | Sí | No | `('OPEN')` | Estado: OPEN, IN_PROGRESS, DONE o CANCELLED. |
| `description` | `NVARCHAR(MAX)` | Sí | No | `` | Descripción del trabajo o incidencia técnica. |
| `created_at_utc` | `DATETIME2(3)` | Sí | No | `SYSUTCDATETIME()` | Fecha/hora UTC de creación de la orden. |
| `closed_at_utc` | `DATETIME2(3)` | No | No | `` | Fecha/hora UTC de cierre. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

### `tunnel.Zone`

Sectorización interna de un tubo: zona operativa, compartimento de incendio, zona de evacuación, zona vulnerable o tramo de riesgo.

| Atributo | Tipo | Obligatorio | PK | Valor por defecto | Descripción |
|---|---:|:---:|:---:|---|---|
| `zone_id` | `UNIQUEIDENTIFIER` | Sí | Sí | `NEWSEQUENTIALID()` | Identificador técnico único de la zona. |
| `tube_id` | `UNIQUEIDENTIFIER` | Sí | No | `` | Tubo al que pertenece la zona. |
| `name` | `NVARCHAR(120)` | Sí | No | `` | Nombre de la zona. |
| `zone_type` | `NVARCHAR(24)` | Sí | No | `` | Tipo de zona: OPERATIONAL, FIRE_COMPARTMENT, EVACUATION, RISK o VULNERABLE. |
| `start_chainage_m` | `DECIMAL(12,3)` | No | No | `` | Inicio de la zona en metros de progresiva o referencia lineal. |
| `end_chainage_m` | `DECIMAL(12,3)` | No | No | `` | Fin de la zona en metros de progresiva o referencia lineal. |
| `meta_json` | `NVARCHAR(MAX)` | No | No | `` | Datos adicionales flexibles en JSON. |

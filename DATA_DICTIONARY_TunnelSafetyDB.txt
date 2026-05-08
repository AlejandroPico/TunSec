# Diccionario de datos - TunnelSafetyDB

Este documento describe la base de datos `TunnelSafetyDB`, diseñada para un sistema de soporte a decisiones en protocolos de seguridad de túneles.

La documentación también queda integrada dentro de SQL Server mediante `MS_Description` y puede consultarse con:

```sql
SELECT * FROM tunnel.v_TableDocumentation ORDER BY table_name;
SELECT * FROM tunnel.v_ColumnDocumentation ORDER BY table_name, column_id;
SELECT * FROM tunnel.v_DatabaseDictionary ORDER BY table_name, item_type, column_id;
```

## Estructura general

- **Topología:** organización, ámbito operativo, túnel, tubo, zona y localización.
- **Activos:** tipos de equipos, sistemas de control, activos e instalaciones.
- **Planes:** planes, versiones y alcance por túnel.
- **Catálogo de incidentes:** esquemas de código, niveles, familias y tipos.
- **Protocolos:** protocolos, pasos, transiciones, acciones y notificaciones.
- **Parámetros:** definiciones, sets, valores y uso por protocolo.
- **Operación:** incidencias reales, detecciones, ejecuciones, acciones, avisos, mantenimiento y auditoría.

## Organization

Organización propietaria, gestora o concesionaria responsable de una o varias instalaciones o redes de túneles.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `organization_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Organization PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la organización. Se usa como clave primaria y no debe cambiar. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre oficial o operativo de la organización. |
| `legal_id` | `NVARCHAR(100) NULL` | Identificador legal/fiscal si aplica, por ejemplo CIF, NIF o identificador administrativo. |
| `country_code` | `NVARCHAR(3) NOT NULL` | Código de país ISO-3166 alfa-3, por ejemplo ESP. |
| `timezone_default` | `NVARCHAR(64) NOT NULL` | Zona horaria por defecto de la organización o ámbito principal. |
| `created_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_Organization_created DEFAULT SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Campo JSON extensible para datos adicionales no normalizados. |

### Restricciones principales

- `CONSTRAINT CK_Organization_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Organization_name ON tunnel.Organization(name);`

## Facility

Ámbito operativo gestionado por una organización: red urbana, concesión, centro de control, autopista o instalación equivalente.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `facility_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Facility PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del ámbito o instalación. |
| `organization_id` | `UNIQUEIDENTIFIER NOT NULL` | Organización propietaria o gestora de la instalación. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre operativo de la instalación, red o centro de control. |
| `facility_type` | `NVARCHAR(40) NOT NULL` | Tipo de instalación: CITY_NETWORK, HIGHWAY_CONCESSION, SINGLE_TUNNEL o CONTROL_CENTER_SCOPE. |
| `address` | `NVARCHAR(400) NULL` | Dirección física o descripción de ubicación si procede. |
| `timezone` | `NVARCHAR(64) NOT NULL` | Zona horaria propia del ámbito operativo. |
| `contact_phone` | `NVARCHAR(50) NULL` | Teléfono general de contacto. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_Facility_created DEFAULT SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### Restricciones principales

- `CONSTRAINT FK_Facility_Organization FOREIGN KEY (organization_id) REFERENCES tunnel.Organization(organization_id)`
- `CONSTRAINT CK_Facility_type CHECK (facility_type IN ('CITY_NETWORK','HIGHWAY_CONCESSION','SINGLE_TUNNEL','CONTROL_CENTER_SCOPE'))`
- `CONSTRAINT CK_Facility_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Facility_org_name ON tunnel.Facility(organization_id, name);`
- `CREATE INDEX IX_Facility_organization ON tunnel.Facility(organization_id);`

## Tunnel

Túnel físico individual. Representa la infraestructura principal y permite vincular tubos, zonas, localizaciones, planes e incidencias.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `tunnel_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Tunnel PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del túnel. |
| `facility_id` | `UNIQUEIDENTIFIER NOT NULL` | Ámbito operativo al que pertenece el túnel. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre oficial u operativo del túnel. |
| `local_code` | `NVARCHAR(80) NULL` | Código interno/local del túnel. |
| `road_name` | `NVARCHAR(120) NULL` | Carretera, ronda, vía o eje viario asociado. |
| `country_code` | `NVARCHAR(3) NOT NULL` | Código de país ISO-3166 alfa-3. |
| `city` | `NVARCHAR(120) NULL` | Ciudad o área territorial. |
| `length_m` | `DECIMAL(12,3) NULL` | Longitud aproximada del túnel en metros. |
| `has_bidirectional_tubes` | `BIT NOT NULL CONSTRAINT DF_Tunnel_bidir DEFAULT (0)` | Indica si el túnel tiene tubos o sentidos bidireccionales. |
| `commissioning_date` | `DATE NULL` | Fecha de puesta en servicio. |
| `tunnel_status` | `NVARCHAR(20) NOT NULL CONSTRAINT DF_Tunnel_status DEFAULT ('ACTIVE')` | Estado operativo del túnel: ACTIVE, WORKS o DECOMMISSIONED. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales como normativa, restricciones, notas geométricas o configuración local. |
| `created_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_Tunnel_created DEFAULT SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### Restricciones principales

- `CONSTRAINT FK_Tunnel_Facility FOREIGN KEY (facility_id) REFERENCES tunnel.Facility(facility_id)`
- `CONSTRAINT CK_Tunnel_status CHECK (tunnel_status IN ('ACTIVE','WORKS','DECOMMISSIONED'))`
- `CONSTRAINT CK_Tunnel_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Tunnel_facility_name ON tunnel.Tunnel(facility_id, name);`
- `CREATE INDEX IX_Tunnel_facility ON tunnel.Tunnel(facility_id);`

## Tube

Tubo, sentido o calzada interna de un túnel. Permite modelar túneles de uno o varios tubos y sentidos de circulación.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `tube_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Tube PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del tubo o sentido. |
| `tunnel_id` | `UNIQUEIDENTIFIER NOT NULL` | Túnel al que pertenece el tubo. |
| `name` | `NVARCHAR(120) NOT NULL` | Nombre del tubo o sentido, por ejemplo Besòs, Llobregat, Ascendente o Descendente. |
| `direction` | `NVARCHAR(16) NOT NULL` | Dirección normalizada: N, S, E, W, A_TO_B, B_TO_A o BIDIR. |
| `lanes_count` | `INT NOT NULL` | Número de carriles del tubo. |
| `speed_limit_kmh` | `INT NULL` | Velocidad máxima autorizada en km/h. |
| `gradient_percent` | `DECIMAL(6,3) NULL` | Pendiente media o relevante expresada en porcentaje. |
| `cross_section_type` | `NVARCHAR(80) NULL` | Tipo de sección o configuración geométrica. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_Tube_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id)`
- `CONSTRAINT CK_Tube_direction CHECK (direction IN ('N','S','E','W','A_TO_B','B_TO_A','BIDIR'))`
- `CONSTRAINT CK_Tube_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Tube_tunnel_name ON tunnel.Tube(tunnel_id, name);`
- `CREATE INDEX IX_Tube_tunnel ON tunnel.Tube(tunnel_id);`

## Zone

Sectorización interna de un tubo: zona operativa, compartimento de incendio, zona de evacuación, zona vulnerable o tramo de riesgo.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `zone_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Zone PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la zona. |
| `tube_id` | `UNIQUEIDENTIFIER NOT NULL` | Tubo al que pertenece la zona. |
| `name` | `NVARCHAR(120) NOT NULL` | Nombre de la zona. |
| `zone_type` | `NVARCHAR(24) NOT NULL` | Tipo de zona: OPERATIONAL, FIRE_COMPARTMENT, EVACUATION, RISK o VULNERABLE. |
| `start_chainage_m` | `DECIMAL(12,3) NULL` | Inicio de la zona en metros de progresiva o referencia lineal. |
| `end_chainage_m` | `DECIMAL(12,3) NULL` | Fin de la zona en metros de progresiva o referencia lineal. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_Zone_Tube FOREIGN KEY (tube_id) REFERENCES tunnel.Tube(tube_id)`
- `CONSTRAINT CK_Zone_type CHECK (zone_type IN ('OPERATIONAL','FIRE_COMPARTMENT','EVACUATION','RISK','VULNERABLE'))`
- `CONSTRAINT CK_Zone_chainage CHECK (start_chainage_m IS NULL OR end_chainage_m IS NULL OR start_chainage_m <= end_chainage_m)`
- `CONSTRAINT CK_Zone_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Zone_tube_name ON tunnel.Zone(tube_id, name);`
- `CREATE INDEX IX_Zone_tube ON tunnel.Zone(tube_id);`

## Location

Localización operativa dentro de un túnel: boca, tramo, punto kilométrico, sala técnica, salida de emergencia, poste SOS, cámara u otro punto relevante.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `location_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Location PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la localización. |
| `tunnel_id` | `UNIQUEIDENTIFIER NOT NULL` | Túnel al que pertenece la localización. |
| `zone_id` | `UNIQUEIDENTIFIER NULL` | Zona a la que pertenece la localización si aplica. |
| `location_type` | `NVARCHAR(30) NOT NULL` | Tipo: PORTAL, SEGMENT, LANE_POINT, TECH_ROOM, CROSS_PASSAGE, EMERGENCY_EXIT, SOS_POST, CAMERA_POLE u OTHER. |
| `name` | `NVARCHAR(200) NULL` | Nombre o etiqueta de la localización. |
| `chainage_m` | `DECIMAL(12,3) NULL` | Progresiva o referencia lineal en metros. |
| `geom` | `GEOGRAPHY NULL` | Coordenada geográfica opcional. Normalmente SRID 4326. |
| `access_description` | `NVARCHAR(500) NULL` | Descripción de acceso para operadores o ayuda externa. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_Location_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id)`
- `CONSTRAINT FK_Location_Zone FOREIGN KEY (zone_id) REFERENCES tunnel.Zone(zone_id)`
- `CONSTRAINT CK_Location_type CHECK (location_type IN ('PORTAL','SEGMENT','LANE_POINT','TECH_ROOM','CROSS_PASSAGE','EMERGENCY_EXIT','SOS_POST','CAMERA_POLE','OTHER'))`
- `CONSTRAINT CK_Location_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE INDEX IX_Location_tunnel ON tunnel.Location(tunnel_id);`
- `CREATE INDEX IX_Location_zone ON tunnel.Location(zone_id);`

## AssetType

Catálogo de tipos de equipamiento independientes de fabricante: CCTV, DAI, SCADA, ventilación, iluminación, PMV, semáforos, SOS, sensores, etc.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `asset_type_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_AssetType PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del tipo de activo. |
| `category` | `NVARCHAR(30) NOT NULL` | Categoría funcional normalizada del activo. |
| `name` | `NVARCHAR(120) NOT NULL` | Nombre concreto del tipo de activo. |
| `vendor_independent` | `BIT NOT NULL CONSTRAINT DF_AssetType_vendorind DEFAULT (1)` | Indica si el tipo es independiente de fabricante. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT CK_AssetType_category CHECK (category IN ('CCTV','DAI','SCADA_IO','VENTILATION','LIGHTING','PMV','SEMAPHORE','SOS','FIRE_DETECTION','CO_NOX','OPACITY','PA_SYSTEM','RADIO_REBROADCAST','POWER_SUPPLY','DRAINAGE','STRUCTURAL_SENSOR','OTHER'))`
- `CONSTRAINT CK_AssetType_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_AssetType_cat_name ON tunnel.AssetType(category, name);`

## ControlSystem

Sistema de control o supervisión que gobierna o monitoriza activos: SCADA, CCTV/VMS, DAI, ATMS, BMS u otros.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `control_system_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ControlSystem PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del sistema de control. |
| `facility_id` | `UNIQUEIDENTIFIER NOT NULL` | Ámbito operativo al que pertenece el sistema. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre operativo del sistema. |
| `system_type` | `NVARCHAR(20) NOT NULL` | Tipo de sistema: SCADA, VMS_CCTV, DAI, ATMS, BMS u OTHER. |
| `primary_site` | `NVARCHAR(200) NULL` | Ubicación principal del sistema si aplica. |
| `has_backup` | `BIT NOT NULL CONSTRAINT DF_ControlSystem_hasbackup DEFAULT (0)` | Indica si dispone de sistema o sala de respaldo. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_ControlSystem_Facility FOREIGN KEY (facility_id) REFERENCES tunnel.Facility(facility_id)`
- `CONSTRAINT CK_ControlSystem_type CHECK (system_type IN ('SCADA','VMS_CCTV','DAI','ATMS','BMS','OTHER'))`
- `CONSTRAINT CK_ControlSystem_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_ControlSystem_facility_name ON tunnel.ControlSystem(facility_id, name);`
- `CREATE INDEX IX_ControlSystem_facility ON tunnel.ControlSystem(facility_id);`

## Asset

Activo físico o lógico instalado o supervisado: cámara, sensor, ventilador, luminaria, PMV, semáforo, sistema SOS, etc.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `asset_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Asset PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del activo. |
| `asset_type_id` | `UNIQUEIDENTIFIER NOT NULL` | Tipo de activo al que pertenece. |
| `control_system_id` | `UNIQUEIDENTIFIER NULL` | Sistema de control que supervisa o controla el activo. |
| `asset_tag` | `NVARCHAR(120) NOT NULL` | Etiqueta de inventario única del activo. |
| `manufacturer` | `NVARCHAR(120) NULL` | Fabricante del activo. |
| `model` | `NVARCHAR(120) NULL` | Modelo del activo. |
| `serial_number` | `NVARCHAR(120) NULL` | Número de serie. |
| `criticality` | `NVARCHAR(20) NOT NULL CONSTRAINT DF_Asset_criticality DEFAULT ('MEDIUM')` | Criticidad operativa: LOW, MEDIUM, HIGH o SAFETY_CRITICAL. |
| `asset_status` | `NVARCHAR(20) NOT NULL CONSTRAINT DF_Asset_status DEFAULT ('OK')` | Estado del activo: OK, DEGRADED, FAILED o MAINTENANCE. |
| `last_healthcheck_at_utc` | `DATETIME2(3) NULL` | Última fecha/hora UTC de comprobación, telemetría o heartbeat. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_Asset_AssetType FOREIGN KEY (asset_type_id) REFERENCES tunnel.AssetType(asset_type_id)`
- `CONSTRAINT FK_Asset_ControlSystem FOREIGN KEY (control_system_id) REFERENCES tunnel.ControlSystem(control_system_id)`
- `CONSTRAINT CK_Asset_criticality CHECK (criticality IN ('LOW','MEDIUM','HIGH','SAFETY_CRITICAL'))`
- `CONSTRAINT CK_Asset_status CHECK (asset_status IN ('OK','DEGRADED','FAILED','MAINTENANCE'))`
- `CONSTRAINT CK_Asset_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Asset_asset_tag ON tunnel.Asset(asset_tag);`
- `CREATE INDEX IX_Asset_type ON tunnel.Asset(asset_type_id);`
- `CREATE INDEX IX_Asset_controlsystem ON tunnel.Asset(control_system_id);`

## AssetInstallation

Instalación de un activo en una localización concreta, con cobertura, orientación y relación con tubo si aplica.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `asset_installation_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_AssetInstallation PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la instalación del activo. |
| `asset_id` | `UNIQUEIDENTIFIER NOT NULL` | Activo instalado. |
| `location_id` | `UNIQUEIDENTIFIER NOT NULL` | Localización donde está instalado el activo. |
| `tube_id` | `UNIQUEIDENTIFIER NULL` | Tubo asociado a la instalación si aplica. |
| `installed_at` | `DATE NULL` | Fecha de instalación. |
| `coverage_start_chainage_m` | `DECIMAL(12,3) NULL` | Inicio de cobertura del activo en metros. |
| `coverage_end_chainage_m` | `DECIMAL(12,3) NULL` | Fin de cobertura del activo en metros. |
| `orientation` | `NVARCHAR(80) NULL` | Orientación física o lógica del activo. |
| `is_primary` | `BIT NOT NULL CONSTRAINT DF_AssetInstallation_primary DEFAULT (0)` | Indica si es la instalación principal del activo. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_AssetInstallation_Asset FOREIGN KEY (asset_id) REFERENCES tunnel.Asset(asset_id)`
- `CONSTRAINT FK_AssetInstallation_Location FOREIGN KEY (location_id) REFERENCES tunnel.Location(location_id)`
- `CONSTRAINT FK_AssetInstallation_Tube FOREIGN KEY (tube_id) REFERENCES tunnel.Tube(tube_id)`
- `CONSTRAINT CK_AssetInstallation_coverage CHECK (coverage_start_chainage_m IS NULL OR coverage_end_chainage_m IS NULL OR coverage_start_chainage_m <= coverage_end_chainage_m)`
- `CONSTRAINT CK_AssetInstallation_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE INDEX IX_AssetInstallation_asset ON tunnel.AssetInstallation(asset_id);`
- `CREATE INDEX IX_AssetInstallation_location ON tunnel.AssetInstallation(location_id);`
- `CREATE INDEX IX_AssetInstallation_tube ON tunnel.AssetInstallation(tube_id);`

## Plan

Plan documental u operativo: PAU, Plan de Emergencia o conjunto de protocolos de explotación.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `plan_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Plan PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del plan. |
| `facility_id` | `UNIQUEIDENTIFIER NOT NULL` | Ámbito operativo al que pertenece el plan. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre del plan. |
| `plan_type` | `NVARCHAR(30) NOT NULL` | Tipo de plan: PAU, EMERGENCY_PLAN u OPERATIONS_PROTOCOLS. |
| `authority` | `NVARCHAR(200) NULL` | Autoridad, organismo o área responsable del plan. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_Plan_created DEFAULT SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### Restricciones principales

- `CONSTRAINT FK_Plan_Facility FOREIGN KEY (facility_id) REFERENCES tunnel.Facility(facility_id)`
- `CONSTRAINT CK_Plan_type CHECK (plan_type IN ('PAU','EMERGENCY_PLAN','OPERATIONS_PROTOCOLS'))`
- `CONSTRAINT CK_Plan_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Plan_facility_name ON tunnel.[Plan](facility_id, name);`
- `CREATE INDEX IX_Plan_facility ON tunnel.[Plan](facility_id);`

## PlanVersion

Versión concreta de un plan, con vigencia, aprobación y estado. Permite gestionar revisiones y trazabilidad normativa.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `plan_version_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_PlanVersion PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la versión del plan. |
| `plan_id` | `UNIQUEIDENTIFIER NOT NULL` | Plan al que pertenece la versión. |
| `version_label` | `NVARCHAR(50) NOT NULL` | Etiqueta de versión, por ejemplo v1.0, v1.1 o 2025.03. |
| `effective_from` | `DATE NULL` | Fecha de inicio de vigencia. |
| `effective_to` | `DATE NULL` | Fecha de fin de vigencia. |
| `version_status` | `NVARCHAR(12) NOT NULL CONSTRAINT DF_PlanVersion_status DEFAULT ('DRAFT')` | Estado de versión: DRAFT, ACTIVE o RETIRED. |
| `approved_by` | `NVARCHAR(200) NULL` | Persona, área u organismo que aprueba la versión. |
| `approval_date` | `DATE NULL` | Fecha de aprobación. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_PlanVersion_created DEFAULT SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### Restricciones principales

- `CONSTRAINT FK_PlanVersion_Plan FOREIGN KEY (plan_id) REFERENCES tunnel.[Plan](plan_id)`
- `CONSTRAINT CK_PlanVersion_status CHECK (version_status IN ('DRAFT','ACTIVE','RETIRED'))`
- `CONSTRAINT CK_PlanVersion_dates CHECK (effective_from IS NULL OR effective_to IS NULL OR effective_from <= effective_to)`
- `CONSTRAINT CK_PlanVersion_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_PlanVersion_plan_version ON tunnel.PlanVersion(plan_id, version_label);`
- `CREATE INDEX IX_PlanVersion_plan ON tunnel.PlanVersion(plan_id);`

## PlanTunnelScope

Relación entre una versión de plan y los túneles cubiertos por ella. Permite que un plan cubra varios túneles y viceversa.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `plan_tunnel_scope_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_PlanTunnelScope PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la relación de alcance. |
| `plan_version_id` | `UNIQUEIDENTIFIER NOT NULL` | Versión de plan aplicable. |
| `tunnel_id` | `UNIQUEIDENTIFIER NOT NULL` | Túnel cubierto por la versión del plan. |
| `scope_note` | `NVARCHAR(400) NULL` | Nota de alcance, por ejemplo fase de obras, tramo parcial o condición especial. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_PlanTunnelScope_PlanVersion FOREIGN KEY (plan_version_id) REFERENCES tunnel.PlanVersion(plan_version_id)`
- `CONSTRAINT FK_PlanTunnelScope_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id)`
- `CONSTRAINT CK_PlanTunnelScope_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_PlanTunnelScope_unique ON tunnel.PlanTunnelScope(plan_version_id, tunnel_id);`
- `CREATE INDEX IX_PlanTunnelScope_tunnel ON tunnel.PlanTunnelScope(tunnel_id);`

## CodeScheme

Esquema de codificación de incidentes. Permite soportar códigos locales como 100-TRA o esquemas de otros países/operadores.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `code_scheme_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_CodeScheme PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del esquema de códigos. |
| `name` | `NVARCHAR(120) NOT NULL` | Nombre del esquema de codificación. |
| `description` | `NVARCHAR(400) NULL` | Descripción funcional del esquema. |
| `pattern_hint` | `NVARCHAR(200) NULL` | Pista de patrón o formato, por ejemplo una regex o estructura esperada. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT CK_CodeScheme_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_CodeScheme_name ON tunnel.CodeScheme(name);`

## EmergencyLevel

Nivel de gravedad o activación: prealerta, alerta, emergencia u otros niveles configurables.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `emergency_level_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_EmergencyLevel PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del nivel de emergencia. |
| `name` | `NVARCHAR(30) NOT NULL` | Nombre del nivel. |
| `rank` | `INT NOT NULL` | Orden de gravedad o prioridad. A mayor valor, mayor nivel si así se configura. |
| `description` | `NVARCHAR(400) NULL` | Descripción operativa del nivel. |

### Índices

- `CREATE UNIQUE INDEX UX_EmergencyLevel_rank ON tunnel.EmergencyLevel(rank);`
- `CREATE UNIQUE INDEX UX_EmergencyLevel_name ON tunnel.EmergencyLevel(name);`

## IncidentFamily

Familia funcional de incidentes: tráfico, avería, incendio, ambiental, iluminación u otras.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `incident_family_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_IncidentFamily PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la familia. |
| `code` | `NVARCHAR(20) NOT NULL` | Código corto de familia, por ejemplo TRA, AVA, FOC, AMB o ILI. |
| `name` | `NVARCHAR(120) NOT NULL` | Nombre de la familia. |
| `description` | `NVARCHAR(400) NULL` | Descripción de los incidentes incluidos en la familia. |

### Índices

- `CREATE UNIQUE INDEX UX_IncidentFamily_code ON tunnel.IncidentFamily(code);`

## IncidentType

Tipo de incidente catalogado. Es la clasificación que permite seleccionar el protocolo adecuado.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `incident_type_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_IncidentType PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del tipo de incidente. |
| `code_scheme_id` | `UNIQUEIDENTIFIER NOT NULL` | Esquema de codificación al que pertenece el código. |
| `emergency_level_id` | `UNIQUEIDENTIFIER NOT NULL` | Nivel de emergencia asociado al tipo. |
| `incident_family_id` | `UNIQUEIDENTIFIER NOT NULL` | Familia funcional del incidente. |
| `code_raw` | `NVARCHAR(40) NOT NULL` | Código textual del incidente, por ejemplo 260-AVA. |
| `title` | `NVARCHAR(200) NOT NULL` | Título operativo del incidente. |
| `description` | `NVARCHAR(MAX) NOT NULL` | Descripción completa del tipo de incidente. |
| `detection_notes` | `NVARCHAR(MAX) NULL` | Notas sobre cómo detectar, verificar o confirmar el incidente. |
| `operational_context` | `NVARCHAR(20) NOT NULL CONSTRAINT DF_IncidentType_context DEFAULT ('NORMAL')` | Contexto: NORMAL, WORKS, EVENT u OTHER. |
| `info_to_collect_json` | `NVARCHAR(MAX) NULL` | Campos o checklist que el operador debe recopilar, en JSON. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_IncidentType_CodeScheme FOREIGN KEY (code_scheme_id) REFERENCES tunnel.CodeScheme(code_scheme_id)`
- `CONSTRAINT FK_IncidentType_EmergencyLevel FOREIGN KEY (emergency_level_id) REFERENCES tunnel.EmergencyLevel(emergency_level_id)`
- `CONSTRAINT FK_IncidentType_IncidentFamily FOREIGN KEY (incident_family_id) REFERENCES tunnel.IncidentFamily(incident_family_id)`
- `CONSTRAINT CK_IncidentType_context CHECK (operational_context IN ('NORMAL','WORKS','EVENT','OTHER'))`
- `CONSTRAINT CK_IncidentType_info_json CHECK (info_to_collect_json IS NULL OR ISJSON(info_to_collect_json) = 1)`
- `CONSTRAINT CK_IncidentType_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_IncidentType_scheme_code ON tunnel.IncidentType(code_scheme_id, code_raw);`
- `CREATE INDEX IX_IncidentType_level ON tunnel.IncidentType(emergency_level_id);`
- `CREATE INDEX IX_IncidentType_family ON tunnel.IncidentType(incident_family_id);`

## Protocol

Plantilla de actuación asociada a una versión de plan y a un tipo de incidente.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `protocol_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Protocol PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del protocolo. |
| `plan_version_id` | `UNIQUEIDENTIFIER NOT NULL` | Versión del plan que define el protocolo. |
| `incident_type_id` | `UNIQUEIDENTIFIER NOT NULL` | Tipo de incidente gestionado por el protocolo. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre operativo del protocolo. |
| `objective` | `NVARCHAR(400) NULL` | Objetivo resumido del protocolo. |
| `is_remote_executable` | `BIT NOT NULL CONSTRAINT DF_Protocol_remote DEFAULT (1)` | Indica si puede ejecutarse desde centro de control o consola remota. |
| `protocol_status` | `NVARCHAR(12) NOT NULL CONSTRAINT DF_Protocol_status DEFAULT ('ACTIVE')` | Estado del protocolo: ACTIVE, RETIRED o DRAFT. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_Protocol_PlanVersion FOREIGN KEY (plan_version_id) REFERENCES tunnel.PlanVersion(plan_version_id)`
- `CONSTRAINT FK_Protocol_IncidentType FOREIGN KEY (incident_type_id) REFERENCES tunnel.IncidentType(incident_type_id)`
- `CONSTRAINT CK_Protocol_status CHECK (protocol_status IN ('ACTIVE','RETIRED','DRAFT'))`
- `CONSTRAINT CK_Protocol_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Protocol_unique ON tunnel.Protocol(plan_version_id, incident_type_id);`
- `CREATE INDEX IX_Protocol_planversion ON tunnel.Protocol(plan_version_id);`
- `CREATE INDEX IX_Protocol_incidenttype ON tunnel.Protocol(incident_type_id);`

## ProtocolStep

Nodo del flujo de trabajo de un protocolo: decisión, acción, espera, información o checklist.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `protocol_step_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ProtocolStep PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del paso. |
| `protocol_id` | `UNIQUEIDENTIFIER NOT NULL` | Protocolo al que pertenece el paso. |
| `step_key` | `NVARCHAR(80) NOT NULL` | Clave estable del paso para referenciarlo desde configuración o interfaz. |
| `step_type` | `NVARCHAR(12) NOT NULL` | Tipo de paso: DECISION, ACTION, INFO, WAIT o CHECKLIST. |
| `title` | `NVARCHAR(200) NOT NULL` | Título visible del paso. |
| `instructions` | `NVARCHAR(MAX) NOT NULL` | Instrucciones operativas que debe seguir el operador o el sistema. |
| `requires_ack` | `BIT NOT NULL CONSTRAINT DF_ProtocolStep_ack DEFAULT (1)` | Indica si el operador debe confirmar el paso. |
| `timeout_seconds` | `INT NULL` | Tiempo máximo recomendado antes de escalar o disparar otra transición. |
| `ui_form_schema_json` | `NVARCHAR(MAX) NULL` | Esquema JSON para pintar formularios dinámicos en la interfaz. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_ProtocolStep_Protocol FOREIGN KEY (protocol_id) REFERENCES tunnel.Protocol(protocol_id)`
- `CONSTRAINT CK_ProtocolStep_type CHECK (step_type IN ('DECISION','ACTION','INFO','WAIT','CHECKLIST'))`
- `CONSTRAINT CK_ProtocolStep_ui_json CHECK (ui_form_schema_json IS NULL OR ISJSON(ui_form_schema_json) = 1)`
- `CONSTRAINT CK_ProtocolStep_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`
- `CONSTRAINT UQ_ProtocolStep_id_protocol UNIQUE (protocol_step_id, protocol_id)`

### Índices

- `CREATE UNIQUE INDEX UX_ProtocolStep_protocol_stepkey ON tunnel.ProtocolStep(protocol_id, step_key);`
- `CREATE INDEX IX_ProtocolStep_protocol ON tunnel.ProtocolStep(protocol_id);`

## StepTransition

Transición dirigida entre pasos de un protocolo. Permite modelar decisiones condicionales, ramas y rutas alternativas.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `step_transition_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_StepTransition PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la transición. |
| `protocol_id` | `UNIQUEIDENTIFIER NOT NULL` | Protocolo al que pertenece la transición. |
| `from_step_id` | `UNIQUEIDENTIFIER NOT NULL` | Paso origen. |
| `to_step_id` | `UNIQUEIDENTIFIER NOT NULL` | Paso destino. |
| `condition_expr` | `NVARCHAR(MAX) NULL` | Expresión de condición o regla de negocio. Si es NULL puede interpretarse como transición por defecto. |
| `priority` | `INT NOT NULL CONSTRAINT DF_StepTransition_priority DEFAULT (0)` | Prioridad de evaluación cuando hay varias transiciones posibles. |
| `label` | `NVARCHAR(200) NULL` | Texto visible para la transición, por ejemplo Sí, No, Confirmado o Escalar. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_StepTransition_Protocol FOREIGN KEY (protocol_id) REFERENCES tunnel.Protocol(protocol_id)`
- `CONSTRAINT FK_StepTransition_FromStep FOREIGN KEY (from_step_id, protocol_id) REFERENCES tunnel.ProtocolStep(protocol_step_id, protocol_id)`
- `CONSTRAINT FK_StepTransition_ToStep FOREIGN KEY (to_step_id, protocol_id) REFERENCES tunnel.ProtocolStep(protocol_step_id, protocol_id)`
- `CONSTRAINT CK_StepTransition_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`
- `CONSTRAINT CK_StepTransition_not_self CHECK (from_step_id <> to_step_id)`

### Índices

- `CREATE INDEX IX_StepTransition_protocol_from ON tunnel.StepTransition(protocol_id, from_step_id);`

## ActionDefinition

Catálogo de acciones atómicas ejecutables o registrables: señalización, semáforos, cierre, ventilación, iluminación, megafonía, orden de trabajo, aviso externo, etc.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `action_definition_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ActionDefinition PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la acción definida. |
| `action_type` | `NVARCHAR(30) NOT NULL` | Tipo de acción: SET_SIGNAGE, SET_SEMAPHORE, CLOSE_TUBE, VENTILATION_MODE, LIGHTING_MODE, PA_ANNOUNCEMENT, CREATE_WORK_ORDER, REQUEST_EXTERNAL, LOG_ONLY u OTHER. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre operativo de la acción. |
| `description` | `NVARCHAR(400) NULL` | Descripción de qué hace la acción. |
| `payload_schema_json` | `NVARCHAR(MAX) NULL` | Esquema JSON de los parámetros necesarios para ejecutar la acción. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT CK_ActionDefinition_type CHECK (action_type IN ('SET_SIGNAGE','SET_SEMAPHORE','CLOSE_TUBE','VENTILATION_MODE','LIGHTING_MODE','PA_ANNOUNCEMENT','CREATE_WORK_ORDER','REQUEST_EXTERNAL','LOG_ONLY','OTHER'))`
- `CONSTRAINT CK_ActionDefinition_payload_json CHECK (payload_schema_json IS NULL OR ISJSON(payload_schema_json) = 1)`
- `CONSTRAINT CK_ActionDefinition_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_ActionDefinition_name ON tunnel.ActionDefinition(name);`

## StepAction

Relación entre un paso de protocolo y una acción definida. Permite ejecutar varias acciones ordenadas por cada paso.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `step_action_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_StepAction PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la relación paso-acción. |
| `protocol_step_id` | `UNIQUEIDENTIFIER NOT NULL` | Paso de protocolo que dispara la acción. |
| `action_definition_id` | `UNIQUEIDENTIFIER NOT NULL` | Acción definida que se debe ejecutar o registrar. |
| `execution_order` | `INT NOT NULL CONSTRAINT DF_StepAction_order DEFAULT (0)` | Orden de ejecución dentro del paso. |
| `is_mandatory` | `BIT NOT NULL CONSTRAINT DF_StepAction_mandatory DEFAULT (1)` | Indica si la acción es obligatoria en el paso. |
| `parameter_binding_json` | `NVARCHAR(MAX) NULL` | Mapeo JSON entre parámetros del protocolo/incidente y payload de acción. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_StepAction_ProtocolStep FOREIGN KEY (protocol_step_id) REFERENCES tunnel.ProtocolStep(protocol_step_id)`
- `CONSTRAINT FK_StepAction_ActionDefinition FOREIGN KEY (action_definition_id) REFERENCES tunnel.ActionDefinition(action_definition_id)`
- `CONSTRAINT CK_StepAction_binding_json CHECK (parameter_binding_json IS NULL OR ISJSON(parameter_binding_json) = 1)`
- `CONSTRAINT CK_StepAction_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_StepAction_unique ON tunnel.StepAction(protocol_step_id, action_definition_id);`
- `CREATE INDEX IX_StepAction_step ON tunnel.StepAction(protocol_step_id);`

## NotificationRule

Regla o plantilla de aviso a organismos, centros de control, mantenimiento u otros contactos.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `notification_rule_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_NotificationRule PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la regla de notificación. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre de la regla. |
| `purpose` | `NVARCHAR(400) NULL` | Finalidad del aviso. |
| `default_channel` | `NVARCHAR(12) NOT NULL` | Canal por defecto: PHONE, EMAIL, RADIO, API, SMS u OTHER. |
| `message_template` | `NVARCHAR(MAX) NOT NULL` | Plantilla del mensaje con variables sustituibles por la aplicación. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT CK_NotificationRule_channel CHECK (default_channel IN ('PHONE','EMAIL','RADIO','API','SMS','OTHER'))`
- `CONSTRAINT CK_NotificationRule_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_NotificationRule_name ON tunnel.NotificationRule(name);`

## StepNotification

Relación entre un paso de protocolo y una regla de notificación. Define cuándo y bajo qué condición se envía un aviso.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `step_notification_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_StepNotification PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la relación paso-notificación. |
| `protocol_step_id` | `UNIQUEIDENTIFIER NOT NULL` | Paso de protocolo que dispara la notificación. |
| `notification_rule_id` | `UNIQUEIDENTIFIER NOT NULL` | Regla de notificación a aplicar. |
| `when` | `NVARCHAR(12) NOT NULL` | Momento de disparo: ON_ENTER, ON_EXIT, ON_TIMEOUT u ON_CONDITION. |
| `condition_expr` | `NVARCHAR(MAX) NULL` | Condición adicional para disparar el aviso. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_StepNotification_ProtocolStep FOREIGN KEY (protocol_step_id) REFERENCES tunnel.ProtocolStep(protocol_step_id)`
- `CONSTRAINT FK_StepNotification_NotificationRule FOREIGN KEY (notification_rule_id) REFERENCES tunnel.NotificationRule(notification_rule_id)`
- `CONSTRAINT CK_StepNotification_when CHECK ([when] IN ('ON_ENTER','ON_EXIT','ON_TIMEOUT','ON_CONDITION'))`
- `CONSTRAINT CK_StepNotification_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_StepNotification_unique ON tunnel.StepNotification(protocol_step_id, notification_rule_id, [when]);`

## ParameterDefinition

Definición global de un parámetro configurable: umbrales, límites, tiempos, modos o reglas reutilizables.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `parameter_definition_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ParameterDefinition PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la definición de parámetro. |
| `key` | `NVARCHAR(120) NOT NULL` | Clave única del parámetro, estable para uso en código y configuración. |
| `data_type` | `NVARCHAR(10) NOT NULL` | Tipo de dato esperado: INT, DECIMAL, BOOLEAN, TEXT o JSON. |
| `unit` | `NVARCHAR(32) NULL` | Unidad del parámetro si aplica, por ejemplo segundos, ppm, km/h o porcentaje. |
| `description` | `NVARCHAR(400) NOT NULL` | Descripción funcional del parámetro. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT CK_ParameterDefinition_type CHECK (data_type IN ('INT','DECIMAL','BOOLEAN','TEXT','JSON'))`
- `CONSTRAINT CK_ParameterDefinition_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_ParameterDefinition_key ON tunnel.ParameterDefinition([key]);`

## ParameterSet

Colección de valores de parámetros aplicable a un túnel o a una versión de plan. Sirve para overrides y personalización.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `parameter_set_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ParameterSet PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del conjunto de parámetros. |
| `tunnel_id` | `UNIQUEIDENTIFIER NULL` | Túnel al que aplica el conjunto, si es un set por túnel. |
| `plan_version_id` | `UNIQUEIDENTIFIER NULL` | Versión de plan a la que aplica el conjunto, si es un set por plan. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre del conjunto de parámetros. |
| `priority` | `INT NOT NULL CONSTRAINT DF_ParameterSet_priority DEFAULT (0)` | Prioridad para resolver overrides cuando hay varios conjuntos aplicables. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_ParameterSet_created DEFAULT SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### Restricciones principales

- `CONSTRAINT FK_ParameterSet_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id)`
- `CONSTRAINT FK_ParameterSet_PlanVersion FOREIGN KEY (plan_version_id) REFERENCES tunnel.PlanVersion(plan_version_id)`
- `CONSTRAINT CK_ParameterSet_owner CHECK ((CASE WHEN tunnel_id IS NULL THEN 0 ELSE 1 END) + (CASE WHEN plan_version_id IS NULL THEN 0 ELSE 1 END) = 1)`
- `CONSTRAINT CK_ParameterSet_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE INDEX IX_ParameterSet_tunnel ON tunnel.ParameterSet(tunnel_id);`
- `CREATE INDEX IX_ParameterSet_planversion ON tunnel.ParameterSet(plan_version_id);`

## ParameterValue

Valor concreto de un parámetro dentro de un conjunto. Usa columnas tipadas para mantener compatibilidad y facilitar validación.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `parameter_value_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ParameterValue PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del valor de parámetro. |
| `parameter_set_id` | `UNIQUEIDENTIFIER NOT NULL` | Conjunto de parámetros al que pertenece el valor. |
| `parameter_definition_id` | `UNIQUEIDENTIFIER NOT NULL` | Definición de parámetro que se está valorando. |
| `value_int` | `INT NULL` | Valor entero si el parámetro es de tipo INT. |
| `value_decimal` | `DECIMAL(18,6) NULL` | Valor decimal si el parámetro es de tipo DECIMAL. |
| `value_bool` | `BIT NULL` | Valor booleano si el parámetro es de tipo BOOLEAN. |
| `value_text` | `NVARCHAR(MAX) NULL` | Valor textual si el parámetro es de tipo TEXT. |
| `value_json` | `NVARCHAR(MAX) NULL` | Valor JSON si el parámetro es de tipo JSON. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_ParameterValue_ParameterSet FOREIGN KEY (parameter_set_id) REFERENCES tunnel.ParameterSet(parameter_set_id)`
- `CONSTRAINT FK_ParameterValue_ParameterDefinition FOREIGN KEY (parameter_definition_id) REFERENCES tunnel.ParameterDefinition(parameter_definition_id)`
- `CONSTRAINT CK_ParameterValue_value_json CHECK (value_json IS NULL OR ISJSON(value_json) = 1)`
- `CONSTRAINT CK_ParameterValue_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_ParameterValue_unique ON tunnel.ParameterValue(parameter_set_id, parameter_definition_id);`

## ProtocolParameter

Relación documental entre protocolo y parámetros que utiliza. Ayuda a validar configuración antes de ejecutar protocolos.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `protocol_parameter_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ProtocolParameter PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la relación protocolo-parámetro. |
| `protocol_id` | `UNIQUEIDENTIFIER NOT NULL` | Protocolo que utiliza el parámetro. |
| `parameter_definition_id` | `UNIQUEIDENTIFIER NOT NULL` | Parámetro requerido o utilizado por el protocolo. |
| `usage_note` | `NVARCHAR(400) NULL` | Nota que explica cómo usa el protocolo este parámetro. |

### Restricciones principales

- `CONSTRAINT FK_ProtocolParameter_Protocol FOREIGN KEY (protocol_id) REFERENCES tunnel.Protocol(protocol_id)`
- `CONSTRAINT FK_ProtocolParameter_ParameterDefinition FOREIGN KEY (parameter_definition_id) REFERENCES tunnel.ParameterDefinition(parameter_definition_id)`

### Índices

- `CREATE UNIQUE INDEX UX_ProtocolParameter_unique ON tunnel.ProtocolParameter(protocol_id, parameter_definition_id);`

## Role

Rol operativo o administrativo de un usuario: operador, jefe de turno, mantenimiento, supervisor, etc.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `role_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Role PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del rol. |
| `name` | `NVARCHAR(80) NOT NULL` | Nombre único del rol. |
| `description` | `NVARCHAR(400) NULL` | Descripción de permisos o responsabilidades del rol. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT CK_Role_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Role_name ON tunnel.[Role](name);`

## UserAccount

Usuario de la aplicación o consola operativa. La autenticación puede estar en la aplicación, pero aquí queda la identidad operativa.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `user_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_UserAccount PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del usuario. |
| `facility_id` | `UNIQUEIDENTIFIER NOT NULL` | Ámbito operativo al que pertenece el usuario. |
| `role_id` | `UNIQUEIDENTIFIER NOT NULL` | Rol asignado al usuario. |
| `username` | `NVARCHAR(120) NOT NULL` | Nombre de usuario o login. |
| `display_name` | `NVARCHAR(200) NOT NULL` | Nombre visible del usuario. |
| `phone` | `NVARCHAR(50) NULL` | Teléfono de contacto. |
| `email` | `NVARCHAR(200) NULL` | Correo electrónico. |
| `is_active` | `BIT NOT NULL CONSTRAINT DF_UserAccount_active DEFAULT (1)` | Indica si el usuario está activo. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |
| `created_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_UserAccount_created DEFAULT SYSUTCDATETIME()` | Fecha y hora UTC de creación del registro. |

### Restricciones principales

- `CONSTRAINT FK_UserAccount_Facility FOREIGN KEY (facility_id) REFERENCES tunnel.Facility(facility_id)`
- `CONSTRAINT FK_UserAccount_Role FOREIGN KEY (role_id) REFERENCES tunnel.[Role](role_id)`
- `CONSTRAINT CK_UserAccount_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_UserAccount_facility_username ON tunnel.UserAccount(facility_id, username);`
- `CREATE INDEX IX_UserAccount_role ON tunnel.UserAccount(role_id);`

## Agency

Organismo o entidad interna/externa que puede ser avisada o participar en la respuesta: emergencias, tráfico, mantenimiento, centro de control, etc.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `agency_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Agency PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del organismo. |
| `name` | `NVARCHAR(200) NOT NULL` | Nombre del organismo o entidad. |
| `agency_type` | `NVARCHAR(30) NOT NULL` | Tipo: EMERGENCY_SERVICES, TRAFFIC_AUTHORITY, CONTROL_CENTER, MAINTENANCE u OTHER. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT CK_Agency_type CHECK (agency_type IN ('EMERGENCY_SERVICES','TRAFFIC_AUTHORITY','CONTROL_CENTER','MAINTENANCE','OTHER'))`
- `CONSTRAINT CK_Agency_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_Agency_name ON tunnel.Agency(name);`

## ContactPoint

Punto de contacto de un organismo: teléfono, radio, email, SMS, endpoint API u otro canal.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `contact_point_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ContactPoint PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del punto de contacto. |
| `agency_id` | `UNIQUEIDENTIFIER NOT NULL` | Organismo al que pertenece el contacto. |
| `name` | `NVARCHAR(200) NULL` | Nombre o etiqueta del contacto. |
| `channel` | `NVARCHAR(12) NOT NULL` | Canal: PHONE, EMAIL, RADIO, API, SMS u OTHER. |
| `address` | `NVARCHAR(400) NOT NULL` | Valor del contacto: teléfono, email, endpoint, canal de radio, etc. |
| `availability` | `NVARCHAR(200) NULL` | Disponibilidad horaria o condición de uso. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_ContactPoint_Agency FOREIGN KEY (agency_id) REFERENCES tunnel.Agency(agency_id)`
- `CONSTRAINT CK_ContactPoint_channel CHECK (channel IN ('PHONE','EMAIL','RADIO','API','SMS','OTHER'))`
- `CONSTRAINT CK_ContactPoint_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_ContactPoint_unique ON tunnel.ContactPoint(agency_id, channel, [address]);`
- `CREATE INDEX IX_ContactPoint_agency ON tunnel.ContactPoint(agency_id);`

## IncidentEvent

Incidente real registrado en explotación. Une túnel, tipo de incidente, estado, tiempos, notas y datos operativos.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `incident_event_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_IncidentEvent PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del incidente real. |
| `incident_type_id` | `UNIQUEIDENTIFIER NOT NULL` | Tipo de incidente catalogado. |
| `tunnel_id` | `UNIQUEIDENTIFIER NOT NULL` | Túnel donde ocurre el incidente. |
| `incident_status` | `NVARCHAR(12) NOT NULL CONSTRAINT DF_IncidentEvent_status DEFAULT ('OPEN')` | Estado del incidente: OPEN, MITIGATING, RESOLVED o CLOSED. |
| `started_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_IncidentEvent_started DEFAULT SYSUTCDATETIME()` | Fecha/hora UTC de inicio o apertura del incidente. |
| `detected_at_utc` | `DATETIME2(3) NULL` | Fecha/hora UTC de detección si difiere de la apertura. |
| `resolved_at_utc` | `DATETIME2(3) NULL` | Fecha/hora UTC de resolución. |
| `severity_override` | `NVARCHAR(30) NULL` | Reclasificación manual de severidad si el operador la aplica. |
| `summary` | `NVARCHAR(400) NULL` | Resumen corto del incidente. |
| `operator_notes` | `NVARCHAR(MAX) NULL` | Notas libres del operador. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_IncidentEvent_IncidentType FOREIGN KEY (incident_type_id) REFERENCES tunnel.IncidentType(incident_type_id)`
- `CONSTRAINT FK_IncidentEvent_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id)`
- `CONSTRAINT CK_IncidentEvent_status CHECK (incident_status IN ('OPEN','MITIGATING','RESOLVED','CLOSED'))`
- `CONSTRAINT CK_IncidentEvent_times CHECK ((detected_at_utc IS NULL OR detected_at_utc >= started_at_utc) AND (resolved_at_utc IS NULL OR resolved_at_utc >= started_at_utc))`
- `CONSTRAINT CK_IncidentEvent_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE INDEX IX_IncidentEvent_tunnel ON tunnel.IncidentEvent(tunnel_id, started_at_utc);`
- `CREATE INDEX IX_IncidentEvent_type ON tunnel.IncidentEvent(incident_type_id);`

## DetectionEvent

Evidencia o evento de detección asociado a un incidente: sensor, CCTV/DAI, alarma SCADA, llamada SOS, aviso externo u observación del operador.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `detection_event_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_DetectionEvent PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del evento de detección. |
| `incident_event_id` | `UNIQUEIDENTIFIER NOT NULL` | Incidente al que pertenece la evidencia. |
| `source_type` | `NVARCHAR(20) NOT NULL` | Origen: SENSOR, CCTV_DAI, SCADA_ALARM, SOS_CALL, EXTERNAL_CALL u OPERATOR_OBS. |
| `source_asset_id` | `UNIQUEIDENTIFIER NULL` | Activo que generó la detección, si aplica. |
| `reported_by` | `NVARCHAR(200) NULL` | Persona, servicio, centro o sistema que reporta la detección. |
| `payload_json` | `NVARCHAR(MAX) NULL` | Datos de detección en JSON: medidas, alarmas, valores, texto, etc. |
| `created_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_DetectionEvent_created DEFAULT SYSUTCDATETIME()` | Fecha/hora UTC de registro de la detección. |

### Restricciones principales

- `CONSTRAINT FK_DetectionEvent_IncidentEvent FOREIGN KEY (incident_event_id) REFERENCES tunnel.IncidentEvent(incident_event_id)`
- `CONSTRAINT FK_DetectionEvent_Asset FOREIGN KEY (source_asset_id) REFERENCES tunnel.Asset(asset_id)`
- `CONSTRAINT CK_DetectionEvent_source CHECK (source_type IN ('SENSOR','CCTV_DAI','SCADA_ALARM','SOS_CALL','EXTERNAL_CALL','OPERATOR_OBS'))`
- `CONSTRAINT CK_DetectionEvent_payload_json CHECK (payload_json IS NULL OR ISJSON(payload_json) = 1)`

### Índices

- `CREATE INDEX IX_DetectionEvent_incident ON tunnel.DetectionEvent(incident_event_id, created_at_utc);`
- `CREATE INDEX IX_DetectionEvent_asset ON tunnel.DetectionEvent(source_asset_id);`

## IncidentLocation

Relación N:M entre incidente y localización. Permite indicar varias ubicaciones, ubicación principal y grado de confianza.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `incident_location_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_IncidentLocation PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la relación incidente-localización. |
| `incident_event_id` | `UNIQUEIDENTIFIER NOT NULL` | Incidente localizado. |
| `location_id` | `UNIQUEIDENTIFIER NOT NULL` | Localización asociada al incidente. |
| `confidence` | `DECIMAL(4,3) NOT NULL CONSTRAINT DF_IncidentLocation_conf DEFAULT (1.000)` | Confianza de la localización entre 0 y 1. |
| `is_primary` | `BIT NOT NULL CONSTRAINT DF_IncidentLocation_primary DEFAULT (0)` | Indica si es la localización principal del incidente. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_IncidentLocation_IncidentEvent FOREIGN KEY (incident_event_id) REFERENCES tunnel.IncidentEvent(incident_event_id)`
- `CONSTRAINT FK_IncidentLocation_Location FOREIGN KEY (location_id) REFERENCES tunnel.Location(location_id)`
- `CONSTRAINT CK_IncidentLocation_conf CHECK (confidence >= 0 AND confidence <= 1)`
- `CONSTRAINT CK_IncidentLocation_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_IncidentLocation_unique ON tunnel.IncidentLocation(incident_event_id, location_id);`
- `CREATE INDEX IX_IncidentLocation_incident ON tunnel.IncidentLocation(incident_event_id);`

## ProtocolRun

Ejecución concreta de un protocolo para un incidente real. Guarda estado, usuario iniciador, paso actual y snapshot de contexto.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `protocol_run_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ProtocolRun PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la ejecución del protocolo. |
| `incident_event_id` | `UNIQUEIDENTIFIER NOT NULL` | Incidente gestionado por esta ejecución. |
| `protocol_id` | `UNIQUEIDENTIFIER NOT NULL` | Protocolo que se está ejecutando. |
| `started_by_user_id` | `UNIQUEIDENTIFIER NOT NULL` | Usuario que inició la ejecución. |
| `started_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_ProtocolRun_started DEFAULT SYSUTCDATETIME()` | Fecha/hora UTC de inicio de la ejecución. |
| `ended_at_utc` | `DATETIME2(3) NULL` | Fecha/hora UTC de finalización. |
| `run_status` | `NVARCHAR(12) NOT NULL CONSTRAINT DF_ProtocolRun_status DEFAULT ('RUNNING')` | Estado de ejecución: RUNNING, PAUSED, COMPLETED o ABORTED. |
| `current_step_id` | `UNIQUEIDENTIFIER NULL` | Paso actual dentro del protocolo. |
| `context_snapshot_json` | `NVARCHAR(MAX) NULL` | Snapshot JSON de contexto: parámetros resueltos, estado de activos, datos del incidente, etc. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_ProtocolRun_IncidentEvent FOREIGN KEY (incident_event_id) REFERENCES tunnel.IncidentEvent(incident_event_id)`
- `CONSTRAINT FK_ProtocolRun_Protocol FOREIGN KEY (protocol_id) REFERENCES tunnel.Protocol(protocol_id)`
- `CONSTRAINT FK_ProtocolRun_User FOREIGN KEY (started_by_user_id) REFERENCES tunnel.UserAccount(user_id)`
- `CONSTRAINT CK_ProtocolRun_status CHECK (run_status IN ('RUNNING','PAUSED','COMPLETED','ABORTED'))`
- `CONSTRAINT CK_ProtocolRun_context_json CHECK (context_snapshot_json IS NULL OR ISJSON(context_snapshot_json) = 1)`
- `CONSTRAINT CK_ProtocolRun_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE INDEX IX_ProtocolRun_incident ON tunnel.ProtocolRun(incident_event_id, started_at_utc);`
- `CREATE INDEX IX_ProtocolRun_protocol ON tunnel.ProtocolRun(protocol_id);`

## ActionExecution

Acción concreta solicitada, enviada, ejecutada o fallida durante una ejecución de protocolo.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `action_execution_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ActionExecution PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la ejecución de acción. |
| `protocol_run_id` | `UNIQUEIDENTIFIER NOT NULL` | Ejecución de protocolo a la que pertenece. |
| `action_definition_id` | `UNIQUEIDENTIFIER NOT NULL` | Acción definida que se ejecuta. |
| `step_id` | `UNIQUEIDENTIFIER NULL` | Paso que disparó la acción. |
| `requested_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_ActionExecution_req DEFAULT SYSUTCDATETIME()` | Fecha/hora UTC en que se solicitó la acción. |
| `executed_at_utc` | `DATETIME2(3) NULL` | Fecha/hora UTC en que se ejecutó o se confirmó. |
| `exec_status` | `NVARCHAR(12) NOT NULL CONSTRAINT DF_ActionExecution_status DEFAULT ('REQUESTED')` | Estado: REQUESTED, SENT, SUCCESS, FAILED o CANCELLED. |
| `payload_json` | `NVARCHAR(MAX) NOT NULL` | Payload JSON real enviado o registrado para ejecutar la acción. |
| `result_json` | `NVARCHAR(MAX) NULL` | Resultado JSON devuelto por el sistema o integración. |
| `error_message` | `NVARCHAR(400) NULL` | Mensaje de error si la acción falla. |

### Restricciones principales

- `CONSTRAINT FK_ActionExecution_Run FOREIGN KEY (protocol_run_id) REFERENCES tunnel.ProtocolRun(protocol_run_id)`
- `CONSTRAINT FK_ActionExecution_Def FOREIGN KEY (action_definition_id) REFERENCES tunnel.ActionDefinition(action_definition_id)`
- `CONSTRAINT FK_ActionExecution_Step FOREIGN KEY (step_id) REFERENCES tunnel.ProtocolStep(protocol_step_id)`
- `CONSTRAINT CK_ActionExecution_status CHECK (exec_status IN ('REQUESTED','SENT','SUCCESS','FAILED','CANCELLED'))`
- `CONSTRAINT CK_ActionExecution_payload_json CHECK (ISJSON(payload_json) = 1)`
- `CONSTRAINT CK_ActionExecution_result_json CHECK (result_json IS NULL OR ISJSON(result_json) = 1)`

### Índices

- `CREATE INDEX IX_ActionExecution_run ON tunnel.ActionExecution(protocol_run_id, requested_at_utc);`
- `CREATE INDEX IX_ActionExecution_step ON tunnel.ActionExecution(step_id);`

## ActionTarget

Equipos o activos afectados por una acción concreta. Permite una acción sobre múltiples objetivos.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `action_target_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ActionTarget PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del objetivo de acción. |
| `action_execution_id` | `UNIQUEIDENTIFIER NOT NULL` | Ejecución de acción que afecta al activo. |
| `asset_id` | `UNIQUEIDENTIFIER NOT NULL` | Activo objetivo de la acción. |
| `target_role` | `NVARCHAR(200) NULL` | Papel del activo en la acción, por ejemplo PMV entrada o ventilador sector 2. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_ActionTarget_ActionExecution FOREIGN KEY (action_execution_id) REFERENCES tunnel.ActionExecution(action_execution_id)`
- `CONSTRAINT FK_ActionTarget_Asset FOREIGN KEY (asset_id) REFERENCES tunnel.Asset(asset_id)`
- `CONSTRAINT CK_ActionTarget_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE UNIQUE INDEX UX_ActionTarget_unique ON tunnel.ActionTarget(action_execution_id, asset_id);`

## Notification

Notificación real enviada o pendiente dentro de una ejecución de protocolo.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `notification_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Notification PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la notificación. |
| `protocol_run_id` | `UNIQUEIDENTIFIER NOT NULL` | Ejecución de protocolo que origina la notificación. |
| `contact_point_id` | `UNIQUEIDENTIFIER NOT NULL` | Punto de contacto destinatario. |
| `notification_rule_id` | `UNIQUEIDENTIFIER NULL` | Regla de notificación aplicada, si procede. |
| `sent_at_utc` | `DATETIME2(3) NULL` | Fecha/hora UTC de envío. |
| `notif_status` | `NVARCHAR(10) NOT NULL CONSTRAINT DF_Notification_status DEFAULT ('PENDING')` | Estado: PENDING, SENT, FAILED o ACKED. |
| `message` | `NVARCHAR(MAX) NOT NULL` | Mensaje final enviado o preparado. |
| `ack_at_utc` | `DATETIME2(3) NULL` | Fecha/hora UTC de acuse o confirmación. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_Notification_Run FOREIGN KEY (protocol_run_id) REFERENCES tunnel.ProtocolRun(protocol_run_id)`
- `CONSTRAINT FK_Notification_ContactPoint FOREIGN KEY (contact_point_id) REFERENCES tunnel.ContactPoint(contact_point_id)`
- `CONSTRAINT FK_Notification_Rule FOREIGN KEY (notification_rule_id) REFERENCES tunnel.NotificationRule(notification_rule_id)`
- `CONSTRAINT CK_Notification_status CHECK (notif_status IN ('PENDING','SENT','FAILED','ACKED'))`
- `CONSTRAINT CK_Notification_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE INDEX IX_Notification_run ON tunnel.Notification(protocol_run_id);`
- `CREATE INDEX IX_Notification_contact ON tunnel.Notification(contact_point_id);`

## WorkOrder

Orden de trabajo o mantenimiento asociada a un activo y opcionalmente generada desde un incidente.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `work_order_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_WorkOrder PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único de la orden de trabajo. |
| `asset_id` | `UNIQUEIDENTIFIER NOT NULL` | Activo afectado por la orden. |
| `created_from_incident_id` | `UNIQUEIDENTIFIER NULL` | Incidente que originó la orden si aplica. |
| `priority` | `NVARCHAR(10) NOT NULL CONSTRAINT DF_WorkOrder_priority DEFAULT ('MEDIUM')` | Prioridad: LOW, MEDIUM, HIGH o URGENT. |
| `work_status` | `NVARCHAR(12) NOT NULL CONSTRAINT DF_WorkOrder_status DEFAULT ('OPEN')` | Estado: OPEN, IN_PROGRESS, DONE o CANCELLED. |
| `description` | `NVARCHAR(MAX) NOT NULL` | Descripción del trabajo o incidencia técnica. |
| `created_at_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_WorkOrder_created DEFAULT SYSUTCDATETIME()` | Fecha/hora UTC de creación de la orden. |
| `closed_at_utc` | `DATETIME2(3) NULL` | Fecha/hora UTC de cierre. |
| `meta_json` | `NVARCHAR(MAX) NULL` | Datos adicionales flexibles en JSON. |

### Restricciones principales

- `CONSTRAINT FK_WorkOrder_Asset FOREIGN KEY (asset_id) REFERENCES tunnel.Asset(asset_id)`
- `CONSTRAINT FK_WorkOrder_Incident FOREIGN KEY (created_from_incident_id) REFERENCES tunnel.IncidentEvent(incident_event_id)`
- `CONSTRAINT CK_WorkOrder_priority CHECK (priority IN ('LOW','MEDIUM','HIGH','URGENT'))`
- `CONSTRAINT CK_WorkOrder_status CHECK (work_status IN ('OPEN','IN_PROGRESS','DONE','CANCELLED'))`
- `CONSTRAINT CK_WorkOrder_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)`

### Índices

- `CREATE INDEX IX_WorkOrder_asset ON tunnel.WorkOrder(asset_id, created_at_utc);`
- `CREATE INDEX IX_WorkOrder_incident ON tunnel.WorkOrder(created_from_incident_id);`

## AuditLog

Registro de auditoría legal/operativa: cambios, ejecuciones, overrides y acciones relevantes del usuario o del sistema.

### Atributos

| Atributo | Definición SQL | Descripción |
|---|---|---|
| `audit_log_id` | `UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_AuditLog PRIMARY KEY DEFAULT NEWSEQUENTIALID()` | Identificador técnico único del evento de auditoría. |
| `user_id` | `UNIQUEIDENTIFIER NULL` | Usuario que realiza la acción, si aplica. |
| `incident_event_id` | `UNIQUEIDENTIFIER NULL` | Incidente relacionado con la acción auditada, si aplica. |
| `entity_type` | `NVARCHAR(60) NOT NULL` | Tipo de entidad afectada, por ejemplo ProtocolRun, Asset o IncidentEvent. |
| `entity_id` | `UNIQUEIDENTIFIER NOT NULL` | Identificador de la entidad afectada. |
| `action` | `NVARCHAR(40) NOT NULL` | Acción realizada: CREATE, UPDATE, EXECUTE, OVERRIDE, etc. |
| `timestamp_utc` | `DATETIME2(3) NOT NULL CONSTRAINT DF_AuditLog_ts DEFAULT SYSUTCDATETIME()` | Fecha/hora UTC del evento auditado. |
| `details_json` | `NVARCHAR(MAX) NULL` | Detalles de auditoría en JSON. |

### Restricciones principales

- `CONSTRAINT FK_AuditLog_User FOREIGN KEY (user_id) REFERENCES tunnel.UserAccount(user_id)`
- `CONSTRAINT FK_AuditLog_Incident FOREIGN KEY (incident_event_id) REFERENCES tunnel.IncidentEvent(incident_event_id)`
- `CONSTRAINT CK_AuditLog_details_json CHECK (details_json IS NULL OR ISJSON(details_json) = 1)`

### Índices

- `CREATE INDEX IX_AuditLog_incident_ts ON tunnel.AuditLog(incident_event_id, [timestamp_utc]);`
- `CREATE INDEX IX_AuditLog_entity ON tunnel.AuditLog(entity_type, entity_id);`


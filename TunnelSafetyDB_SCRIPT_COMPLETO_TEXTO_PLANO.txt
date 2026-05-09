/* =============================================================================
   Tunnel Safety Decision Support DB - SQL Server 2022
   Archivo: 001_create_tunnel_safety_db.sql

   OBJETIVO
   -------
   Crear una base de datos documental y operativa para un sistema informático que
   agilice la toma de decisiones en protocolos de seguridad de túneles.

   El modelo está pensado para ser versátil:
   - varias organizaciones y redes de túneles;
   - túneles con múltiples tubos, zonas y localizaciones;
   - planes de emergencia/PAU versionados;
   - catálogo de incidentes por nivel, familia y esquema de código;
   - protocolos como workflows con pasos, decisiones, transiciones y acciones;
   - notificaciones a organismos/contactos;
   - incidencias reales, ejecuciones de protocolos, acciones y auditoría.

   COMPATIBILIDAD
   --------------
   - SQL Server 2022.
   - COMPATIBILITY_LEVEL = 160.
   - ISJSON() está soportado.
   - NVARCHAR(MAX) está soportado. Si DBeaver lo marca en rojo, normalmente es
     un falso positivo del parser del editor, no del motor de SQL Server.

   DOCUMENTACIÓN INTERNA
   ---------------------
   Además de comentarios en este script, las tablas y columnas quedan documentadas
   con extended properties estándar de SQL Server: MS_Description.

   Puedes consultar la documentación desde la propia base de datos con:

       SELECT * FROM tunnel.v_TableDocumentation ORDER BY table_name;

       SELECT * FROM tunnel.v_ColumnDocumentation
       ORDER BY table_name, column_id;

       SELECT * FROM tunnel.v_DatabaseDictionary
       ORDER BY table_name, item_type, column_id;

   EJECUCIÓN EN DBEAVER
   --------------------
   Ejecuta el script completo contra tu servidor SQL Server. El script crea la
   base [TunnelSafetyDB] si no existe y cambia de contexto con USE [TunnelSafetyDB]
   de forma real, no mediante EXEC('USE ...').

   ============================================================================= */

SET NOCOUNT ON;
SET XACT_ABORT ON;

-------------------------------------------------------------------------------
-- 1) CREACIÓN DE BASE DE DATOS DESTINO
-------------------------------------------------------------------------------
IF DB_ID(N'TunnelSafetyDB') IS NULL
BEGIN
    PRINT N'Creando base de datos [TunnelSafetyDB]...';
    EXEC(N'CREATE DATABASE [TunnelSafetyDB];');
END
ELSE
BEGIN
    PRINT N'La base de datos [TunnelSafetyDB] ya existe. Se validará el esquema.';
END;

ALTER DATABASE [TunnelSafetyDB] SET COMPATIBILITY_LEVEL = 160;
ALTER DATABASE [TunnelSafetyDB] SET RECOVERY SIMPLE;

-------------------------------------------------------------------------------
-- 2) CAMBIO REAL DE CONTEXTO
--    IMPORTANTE: no usar EXEC('USE ...') porque no cambia el contexto externo.
-------------------------------------------------------------------------------
USE [TunnelSafetyDB];

-------------------------------------------------------------------------------
-- 3) CREACIÓN DEL SCHEMA LÓGICO
-------------------------------------------------------------------------------
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'tunnel')
BEGIN
    EXEC(N'CREATE SCHEMA [tunnel] AUTHORIZATION [dbo];');
END;

-------------------------------------------------------------------------------
-- 4) CREACIÓN DE TABLAS, CLAVES, CHECKS E ÍNDICES
-------------------------------------------------------------------------------
BEGIN TRY
BEGIN TRAN;

/* ORGANIZATION
   Organización propietaria, gestora o concesionaria responsable de una o varias instalaciones o
   redes de túneles.
*/
IF OBJECT_ID(N'tunnel.Organization', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Organization (
        -- organization_id: Identificador técnico único de la organización. Se usa como clave
        -- primaria y no debe cambiar.
        organization_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Organization PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- name: Nombre oficial o operativo de la organización.
        name NVARCHAR(200) NOT NULL,
        -- legal_id: Identificador legal/fiscal si aplica, por ejemplo CIF, NIF o identificador
        -- administrativo.
        legal_id NVARCHAR(100) NULL,
        -- country_code: Código de país ISO-3166 alfa-3, por ejemplo ESP.
        country_code NVARCHAR(3) NOT NULL,
        -- timezone_default: Zona horaria por defecto de la organización o ámbito principal.
        timezone_default NVARCHAR(64) NOT NULL,
        -- created_at_utc: Fecha y hora UTC de creación del registro.
        created_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_Organization_created DEFAULT SYSUTCDATETIME(),
        -- meta_json: Campo JSON extensible para datos adicionales no normalizados.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT CK_Organization_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Organization_name' AND object_id = OBJECT_ID(N'tunnel.Organization'))
BEGIN
    CREATE UNIQUE INDEX UX_Organization_name ON tunnel.Organization(name);
END;


/* FACILITY
   Ámbito operativo gestionado por una organización: red urbana, concesión, centro de control,
   autopista o instalación equivalente.
*/
IF OBJECT_ID(N'tunnel.Facility', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Facility (
        -- facility_id: Identificador técnico único del ámbito o instalación.
        facility_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Facility PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- organization_id: Organización propietaria o gestora de la instalación.
        organization_id UNIQUEIDENTIFIER NOT NULL,
        -- name: Nombre operativo de la instalación, red o centro de control.
        name NVARCHAR(200) NOT NULL,
        -- facility_type: Tipo de instalación: CITY_NETWORK, HIGHWAY_CONCESSION, SINGLE_TUNNEL o
        -- CONTROL_CENTER_SCOPE.
        facility_type NVARCHAR(40) NOT NULL,
        -- address: Dirección física o descripción de ubicación si procede.
        [address] NVARCHAR(400) NULL,
        -- timezone: Zona horaria propia del ámbito operativo.
        timezone NVARCHAR(64) NOT NULL,
        -- contact_phone: Teléfono general de contacto.
        contact_phone NVARCHAR(50) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        -- created_at_utc: Fecha y hora UTC de creación del registro.
        created_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_Facility_created DEFAULT SYSUTCDATETIME(),
        CONSTRAINT FK_Facility_Organization FOREIGN KEY (organization_id) REFERENCES tunnel.Organization(organization_id),
        CONSTRAINT CK_Facility_type CHECK (facility_type IN ('CITY_NETWORK','HIGHWAY_CONCESSION','SINGLE_TUNNEL','CONTROL_CENTER_SCOPE')),
        CONSTRAINT CK_Facility_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Facility_org_name' AND object_id = OBJECT_ID(N'tunnel.Facility'))
BEGIN
    CREATE UNIQUE INDEX UX_Facility_org_name ON tunnel.Facility(organization_id, name);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Facility_organization' AND object_id = OBJECT_ID(N'tunnel.Facility'))
BEGIN
    CREATE INDEX IX_Facility_organization ON tunnel.Facility(organization_id);
END;


/* TUNNEL
   Túnel físico individual. Representa la infraestructura principal y permite vincular tubos,
   zonas, localizaciones, planes e incidencias.
*/
IF OBJECT_ID(N'tunnel.Tunnel', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Tunnel (
        -- tunnel_id: Identificador técnico único del túnel.
        tunnel_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Tunnel PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- facility_id: Ámbito operativo al que pertenece el túnel.
        facility_id UNIQUEIDENTIFIER NOT NULL,
        -- name: Nombre oficial u operativo del túnel.
        name NVARCHAR(200) NOT NULL,
        -- local_code: Código interno/local del túnel.
        local_code NVARCHAR(80) NULL,
        -- road_name: Carretera, ronda, vía o eje viario asociado.
        road_name NVARCHAR(120) NULL,
        -- country_code: Código de país ISO-3166 alfa-3.
        country_code NVARCHAR(3) NOT NULL,
        -- city: Ciudad o área territorial.
        city NVARCHAR(120) NULL,
        -- length_m: Longitud aproximada del túnel en metros.
        length_m DECIMAL(12,3) NULL,
        -- has_bidirectional_tubes: Indica si el túnel tiene tubos o sentidos bidireccionales.
        has_bidirectional_tubes BIT NOT NULL CONSTRAINT DF_Tunnel_bidir DEFAULT (0),
        -- commissioning_date: Fecha de puesta en servicio.
        commissioning_date DATE NULL,
        -- tunnel_status: Estado operativo del túnel: ACTIVE, WORKS o DECOMMISSIONED.
        tunnel_status NVARCHAR(20) NOT NULL CONSTRAINT DF_Tunnel_status DEFAULT ('ACTIVE'),
        -- meta_json: Datos adicionales como normativa, restricciones, notas geométricas o
        -- configuración local.
        meta_json NVARCHAR(MAX) NULL,
        -- created_at_utc: Fecha y hora UTC de creación del registro.
        created_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_Tunnel_created DEFAULT SYSUTCDATETIME(),
        CONSTRAINT FK_Tunnel_Facility FOREIGN KEY (facility_id) REFERENCES tunnel.Facility(facility_id),
        CONSTRAINT CK_Tunnel_status CHECK (tunnel_status IN ('ACTIVE','WORKS','DECOMMISSIONED')),
        CONSTRAINT CK_Tunnel_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Tunnel_facility_name' AND object_id = OBJECT_ID(N'tunnel.Tunnel'))
BEGIN
    CREATE UNIQUE INDEX UX_Tunnel_facility_name ON tunnel.Tunnel(facility_id, name);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Tunnel_facility' AND object_id = OBJECT_ID(N'tunnel.Tunnel'))
BEGIN
    CREATE INDEX IX_Tunnel_facility ON tunnel.Tunnel(facility_id);
END;


/* TUBE
   Tubo, sentido o calzada interna de un túnel. Permite modelar túneles de uno o varios tubos y
   sentidos de circulación.
*/
IF OBJECT_ID(N'tunnel.Tube', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Tube (
        -- tube_id: Identificador técnico único del tubo o sentido.
        tube_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Tube PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- tunnel_id: Túnel al que pertenece el tubo.
        tunnel_id UNIQUEIDENTIFIER NOT NULL,
        -- name: Nombre del tubo o sentido, por ejemplo Besòs, Llobregat, Ascendente o Descendente.
        name NVARCHAR(120) NOT NULL,
        -- direction: Dirección normalizada: N, S, E, W, A_TO_B, B_TO_A o BIDIR.
        direction NVARCHAR(16) NOT NULL,
        -- lanes_count: Número de carriles del tubo.
        lanes_count INT NOT NULL,
        -- speed_limit_kmh: Velocidad máxima autorizada en km/h.
        speed_limit_kmh INT NULL,
        -- gradient_percent: Pendiente media o relevante expresada en porcentaje.
        gradient_percent DECIMAL(6,3) NULL,
        -- cross_section_type: Tipo de sección o configuración geométrica.
        cross_section_type NVARCHAR(80) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_Tube_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id),
        CONSTRAINT CK_Tube_direction CHECK (direction IN ('N','S','E','W','A_TO_B','B_TO_A','BIDIR')),
        CONSTRAINT CK_Tube_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Tube_tunnel_name' AND object_id = OBJECT_ID(N'tunnel.Tube'))
BEGIN
    CREATE UNIQUE INDEX UX_Tube_tunnel_name ON tunnel.Tube(tunnel_id, name);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Tube_tunnel' AND object_id = OBJECT_ID(N'tunnel.Tube'))
BEGIN
    CREATE INDEX IX_Tube_tunnel ON tunnel.Tube(tunnel_id);
END;


/* ZONE
   Sectorización interna de un tubo: zona operativa, compartimento de incendio, zona de
   evacuación, zona vulnerable o tramo de riesgo.
*/
IF OBJECT_ID(N'tunnel.Zone', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Zone (
        -- zone_id: Identificador técnico único de la zona.
        zone_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Zone PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- tube_id: Tubo al que pertenece la zona.
        tube_id UNIQUEIDENTIFIER NOT NULL,
        -- name: Nombre de la zona.
        name NVARCHAR(120) NOT NULL,
        -- zone_type: Tipo de zona: OPERATIONAL, FIRE_COMPARTMENT, EVACUATION, RISK o VULNERABLE.
        zone_type NVARCHAR(24) NOT NULL,
        -- start_chainage_m: Inicio de la zona en metros de progresiva o referencia lineal.
        start_chainage_m DECIMAL(12,3) NULL,
        -- end_chainage_m: Fin de la zona en metros de progresiva o referencia lineal.
        end_chainage_m DECIMAL(12,3) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_Zone_Tube FOREIGN KEY (tube_id) REFERENCES tunnel.Tube(tube_id),
        CONSTRAINT CK_Zone_type CHECK (zone_type IN ('OPERATIONAL','FIRE_COMPARTMENT','EVACUATION','RISK','VULNERABLE')),
        CONSTRAINT CK_Zone_chainage CHECK (start_chainage_m IS NULL OR end_chainage_m IS NULL OR start_chainage_m <= end_chainage_m),
        CONSTRAINT CK_Zone_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Zone_tube_name' AND object_id = OBJECT_ID(N'tunnel.Zone'))
BEGIN
    CREATE UNIQUE INDEX UX_Zone_tube_name ON tunnel.Zone(tube_id, name);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Zone_tube' AND object_id = OBJECT_ID(N'tunnel.Zone'))
BEGIN
    CREATE INDEX IX_Zone_tube ON tunnel.Zone(tube_id);
END;


/* LOCATION
   Localización operativa dentro de un túnel: boca, tramo, punto kilométrico, sala técnica, salida
   de emergencia, poste SOS, cámara u otro punto relevante.
*/
IF OBJECT_ID(N'tunnel.Location', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Location (
        -- location_id: Identificador técnico único de la localización.
        location_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Location PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- tunnel_id: Túnel al que pertenece la localización.
        tunnel_id UNIQUEIDENTIFIER NOT NULL,
        -- zone_id: Zona a la que pertenece la localización si aplica.
        zone_id UNIQUEIDENTIFIER NULL,
        -- location_type: Tipo: PORTAL, SEGMENT, LANE_POINT, TECH_ROOM, CROSS_PASSAGE,
        -- EMERGENCY_EXIT, SOS_POST, CAMERA_POLE u OTHER.
        location_type NVARCHAR(30) NOT NULL,
        -- name: Nombre o etiqueta de la localización.
        name NVARCHAR(200) NULL,
        -- chainage_m: Progresiva o referencia lineal en metros.
        chainage_m DECIMAL(12,3) NULL,
        -- geom: Coordenada geográfica opcional. Normalmente SRID 4326.
        geom GEOGRAPHY NULL,
        -- access_description: Descripción de acceso para operadores o ayuda externa.
        access_description NVARCHAR(500) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_Location_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id),
        CONSTRAINT FK_Location_Zone FOREIGN KEY (zone_id) REFERENCES tunnel.Zone(zone_id),
        CONSTRAINT CK_Location_type CHECK (location_type IN ('PORTAL','SEGMENT','LANE_POINT','TECH_ROOM','CROSS_PASSAGE','EMERGENCY_EXIT','SOS_POST','CAMERA_POLE','OTHER')),
        CONSTRAINT CK_Location_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Location_tunnel' AND object_id = OBJECT_ID(N'tunnel.Location'))
BEGIN
    CREATE INDEX IX_Location_tunnel ON tunnel.Location(tunnel_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Location_zone' AND object_id = OBJECT_ID(N'tunnel.Location'))
BEGIN
    CREATE INDEX IX_Location_zone ON tunnel.Location(zone_id);
END;


/* ASSETTYPE
   Catálogo de tipos de equipamiento independientes de fabricante: CCTV, DAI, SCADA, ventilación,
   iluminación, PMV, semáforos, SOS, sensores, etc.
*/
IF OBJECT_ID(N'tunnel.AssetType', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.AssetType (
        -- asset_type_id: Identificador técnico único del tipo de activo.
        asset_type_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_AssetType PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- category: Categoría funcional normalizada del activo.
        category NVARCHAR(30) NOT NULL,
        -- name: Nombre concreto del tipo de activo.
        name NVARCHAR(120) NOT NULL,
        -- vendor_independent: Indica si el tipo es independiente de fabricante.
        vendor_independent BIT NOT NULL CONSTRAINT DF_AssetType_vendorind DEFAULT (1),
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT CK_AssetType_category CHECK (category IN ('CCTV','DAI','SCADA_IO','VENTILATION','LIGHTING','PMV','SEMAPHORE','SOS','FIRE_DETECTION','CO_NOX','OPACITY','PA_SYSTEM','RADIO_REBROADCAST','POWER_SUPPLY','DRAINAGE','STRUCTURAL_SENSOR','OTHER')),
        CONSTRAINT CK_AssetType_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_AssetType_cat_name' AND object_id = OBJECT_ID(N'tunnel.AssetType'))
BEGIN
    CREATE UNIQUE INDEX UX_AssetType_cat_name ON tunnel.AssetType(category, name);
END;


/* CONTROLSYSTEM
   Sistema de control o supervisión que gobierna o monitoriza activos: SCADA, CCTV/VMS, DAI, ATMS,
   BMS u otros.
*/
IF OBJECT_ID(N'tunnel.ControlSystem', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ControlSystem (
        -- control_system_id: Identificador técnico único del sistema de control.
        control_system_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ControlSystem PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- facility_id: Ámbito operativo al que pertenece el sistema.
        facility_id UNIQUEIDENTIFIER NOT NULL,
        -- name: Nombre operativo del sistema.
        name NVARCHAR(200) NOT NULL,
        -- system_type: Tipo de sistema: SCADA, VMS_CCTV, DAI, ATMS, BMS u OTHER.
        system_type NVARCHAR(20) NOT NULL,
        -- primary_site: Ubicación principal del sistema si aplica.
        primary_site NVARCHAR(200) NULL,
        -- has_backup: Indica si dispone de sistema o sala de respaldo.
        has_backup BIT NOT NULL CONSTRAINT DF_ControlSystem_hasbackup DEFAULT (0),
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_ControlSystem_Facility FOREIGN KEY (facility_id) REFERENCES tunnel.Facility(facility_id),
        CONSTRAINT CK_ControlSystem_type CHECK (system_type IN ('SCADA','VMS_CCTV','DAI','ATMS','BMS','OTHER')),
        CONSTRAINT CK_ControlSystem_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_ControlSystem_facility_name' AND object_id = OBJECT_ID(N'tunnel.ControlSystem'))
BEGIN
    CREATE UNIQUE INDEX UX_ControlSystem_facility_name ON tunnel.ControlSystem(facility_id, name);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_ControlSystem_facility' AND object_id = OBJECT_ID(N'tunnel.ControlSystem'))
BEGIN
    CREATE INDEX IX_ControlSystem_facility ON tunnel.ControlSystem(facility_id);
END;


/* ASSET
   Activo físico o lógico instalado o supervisado: cámara, sensor, ventilador, luminaria, PMV,
   semáforo, sistema SOS, etc.
*/
IF OBJECT_ID(N'tunnel.Asset', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Asset (
        -- asset_id: Identificador técnico único del activo.
        asset_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Asset PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- asset_type_id: Tipo de activo al que pertenece.
        asset_type_id UNIQUEIDENTIFIER NOT NULL,
        -- control_system_id: Sistema de control que supervisa o controla el activo.
        control_system_id UNIQUEIDENTIFIER NULL,
        -- asset_tag: Etiqueta de inventario única del activo.
        asset_tag NVARCHAR(120) NOT NULL,
        -- manufacturer: Fabricante del activo.
        manufacturer NVARCHAR(120) NULL,
        -- model: Modelo del activo.
        model NVARCHAR(120) NULL,
        -- serial_number: Número de serie.
        serial_number NVARCHAR(120) NULL,
        -- criticality: Criticidad operativa: LOW, MEDIUM, HIGH o SAFETY_CRITICAL.
        criticality NVARCHAR(20) NOT NULL CONSTRAINT DF_Asset_criticality DEFAULT ('MEDIUM'),
        -- asset_status: Estado del activo: OK, DEGRADED, FAILED o MAINTENANCE.
        asset_status NVARCHAR(20) NOT NULL CONSTRAINT DF_Asset_status DEFAULT ('OK'),
        -- last_healthcheck_at_utc: Última fecha/hora UTC de comprobación, telemetría o heartbeat.
        last_healthcheck_at_utc DATETIME2(3) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_Asset_AssetType FOREIGN KEY (asset_type_id) REFERENCES tunnel.AssetType(asset_type_id),
        CONSTRAINT FK_Asset_ControlSystem FOREIGN KEY (control_system_id) REFERENCES tunnel.ControlSystem(control_system_id),
        CONSTRAINT CK_Asset_criticality CHECK (criticality IN ('LOW','MEDIUM','HIGH','SAFETY_CRITICAL')),
        CONSTRAINT CK_Asset_status CHECK (asset_status IN ('OK','DEGRADED','FAILED','MAINTENANCE')),
        CONSTRAINT CK_Asset_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Asset_asset_tag' AND object_id = OBJECT_ID(N'tunnel.Asset'))
BEGIN
    CREATE UNIQUE INDEX UX_Asset_asset_tag ON tunnel.Asset(asset_tag);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Asset_type' AND object_id = OBJECT_ID(N'tunnel.Asset'))
BEGIN
    CREATE INDEX IX_Asset_type ON tunnel.Asset(asset_type_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Asset_controlsystem' AND object_id = OBJECT_ID(N'tunnel.Asset'))
BEGIN
    CREATE INDEX IX_Asset_controlsystem ON tunnel.Asset(control_system_id);
END;


/* ASSETINSTALLATION
   Instalación de un activo en una localización concreta, con cobertura, orientación y relación
   con tubo si aplica.
*/
IF OBJECT_ID(N'tunnel.AssetInstallation', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.AssetInstallation (
        -- asset_installation_id: Identificador técnico único de la instalación del activo.
        asset_installation_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_AssetInstallation PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- asset_id: Activo instalado.
        asset_id UNIQUEIDENTIFIER NOT NULL,
        -- location_id: Localización donde está instalado el activo.
        location_id UNIQUEIDENTIFIER NOT NULL,
        -- tube_id: Tubo asociado a la instalación si aplica.
        tube_id UNIQUEIDENTIFIER NULL,
        -- installed_at: Fecha de instalación.
        installed_at DATE NULL,
        -- coverage_start_chainage_m: Inicio de cobertura del activo en metros.
        coverage_start_chainage_m DECIMAL(12,3) NULL,
        -- coverage_end_chainage_m: Fin de cobertura del activo en metros.
        coverage_end_chainage_m DECIMAL(12,3) NULL,
        -- orientation: Orientación física o lógica del activo.
        orientation NVARCHAR(80) NULL,
        -- is_primary: Indica si es la instalación principal del activo.
        is_primary BIT NOT NULL CONSTRAINT DF_AssetInstallation_primary DEFAULT (0),
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_AssetInstallation_Asset FOREIGN KEY (asset_id) REFERENCES tunnel.Asset(asset_id),
        CONSTRAINT FK_AssetInstallation_Location FOREIGN KEY (location_id) REFERENCES tunnel.Location(location_id),
        CONSTRAINT FK_AssetInstallation_Tube FOREIGN KEY (tube_id) REFERENCES tunnel.Tube(tube_id),
        CONSTRAINT CK_AssetInstallation_coverage CHECK (coverage_start_chainage_m IS NULL OR coverage_end_chainage_m IS NULL OR coverage_start_chainage_m <= coverage_end_chainage_m),
        CONSTRAINT CK_AssetInstallation_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_AssetInstallation_asset' AND object_id = OBJECT_ID(N'tunnel.AssetInstallation'))
BEGIN
    CREATE INDEX IX_AssetInstallation_asset ON tunnel.AssetInstallation(asset_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_AssetInstallation_location' AND object_id = OBJECT_ID(N'tunnel.AssetInstallation'))
BEGIN
    CREATE INDEX IX_AssetInstallation_location ON tunnel.AssetInstallation(location_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_AssetInstallation_tube' AND object_id = OBJECT_ID(N'tunnel.AssetInstallation'))
BEGIN
    CREATE INDEX IX_AssetInstallation_tube ON tunnel.AssetInstallation(tube_id);
END;


/* PLAN
   Plan documental u operativo: PAU, Plan de Emergencia o conjunto de protocolos de explotación.
*/
IF OBJECT_ID(N'tunnel.Plan', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.[Plan] (
        -- plan_id: Identificador técnico único del plan.
        plan_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Plan PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- facility_id: Ámbito operativo al que pertenece el plan.
        facility_id UNIQUEIDENTIFIER NOT NULL,
        -- name: Nombre del plan.
        name NVARCHAR(200) NOT NULL,
        -- plan_type: Tipo de plan: PAU, EMERGENCY_PLAN u OPERATIONS_PROTOCOLS.
        plan_type NVARCHAR(30) NOT NULL,
        -- authority: Autoridad, organismo o área responsable del plan.
        authority NVARCHAR(200) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        -- created_at_utc: Fecha y hora UTC de creación del registro.
        created_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_Plan_created DEFAULT SYSUTCDATETIME(),
        CONSTRAINT FK_Plan_Facility FOREIGN KEY (facility_id) REFERENCES tunnel.Facility(facility_id),
        CONSTRAINT CK_Plan_type CHECK (plan_type IN ('PAU','EMERGENCY_PLAN','OPERATIONS_PROTOCOLS')),
        CONSTRAINT CK_Plan_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Plan_facility_name' AND object_id = OBJECT_ID(N'tunnel.Plan'))
BEGIN
    CREATE UNIQUE INDEX UX_Plan_facility_name ON tunnel.[Plan](facility_id, name);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Plan_facility' AND object_id = OBJECT_ID(N'tunnel.Plan'))
BEGIN
    CREATE INDEX IX_Plan_facility ON tunnel.[Plan](facility_id);
END;


/* PLANVERSION
   Versión concreta de un plan, con vigencia, aprobación y estado. Permite gestionar revisiones y
   trazabilidad normativa.
*/
IF OBJECT_ID(N'tunnel.PlanVersion', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.PlanVersion (
        -- plan_version_id: Identificador técnico único de la versión del plan.
        plan_version_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_PlanVersion PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- plan_id: Plan al que pertenece la versión.
        plan_id UNIQUEIDENTIFIER NOT NULL,
        -- version_label: Etiqueta de versión, por ejemplo v1.0, v1.1 o 2025.03.
        version_label NVARCHAR(50) NOT NULL,
        -- effective_from: Fecha de inicio de vigencia.
        effective_from DATE NULL,
        -- effective_to: Fecha de fin de vigencia.
        effective_to DATE NULL,
        -- version_status: Estado de versión: DRAFT, ACTIVE o RETIRED.
        version_status NVARCHAR(12) NOT NULL CONSTRAINT DF_PlanVersion_status DEFAULT ('DRAFT'),
        -- approved_by: Persona, área u organismo que aprueba la versión.
        approved_by NVARCHAR(200) NULL,
        -- approval_date: Fecha de aprobación.
        approval_date DATE NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        -- created_at_utc: Fecha y hora UTC de creación del registro.
        created_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_PlanVersion_created DEFAULT SYSUTCDATETIME(),
        CONSTRAINT FK_PlanVersion_Plan FOREIGN KEY (plan_id) REFERENCES tunnel.[Plan](plan_id),
        CONSTRAINT CK_PlanVersion_status CHECK (version_status IN ('DRAFT','ACTIVE','RETIRED')),
        CONSTRAINT CK_PlanVersion_dates CHECK (effective_from IS NULL OR effective_to IS NULL OR effective_from <= effective_to),
        CONSTRAINT CK_PlanVersion_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_PlanVersion_plan_version' AND object_id = OBJECT_ID(N'tunnel.PlanVersion'))
BEGIN
    CREATE UNIQUE INDEX UX_PlanVersion_plan_version ON tunnel.PlanVersion(plan_id, version_label);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_PlanVersion_plan' AND object_id = OBJECT_ID(N'tunnel.PlanVersion'))
BEGIN
    CREATE INDEX IX_PlanVersion_plan ON tunnel.PlanVersion(plan_id);
END;


/* PLANTUNNELSCOPE
   Relación entre una versión de plan y los túneles cubiertos por ella. Permite que un plan cubra
   varios túneles y viceversa.
*/
IF OBJECT_ID(N'tunnel.PlanTunnelScope', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.PlanTunnelScope (
        -- plan_tunnel_scope_id: Identificador técnico único de la relación de alcance.
        plan_tunnel_scope_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_PlanTunnelScope PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- plan_version_id: Versión de plan aplicable.
        plan_version_id UNIQUEIDENTIFIER NOT NULL,
        -- tunnel_id: Túnel cubierto por la versión del plan.
        tunnel_id UNIQUEIDENTIFIER NOT NULL,
        -- scope_note: Nota de alcance, por ejemplo fase de obras, tramo parcial o condición
        -- especial.
        scope_note NVARCHAR(400) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_PlanTunnelScope_PlanVersion FOREIGN KEY (plan_version_id) REFERENCES tunnel.PlanVersion(plan_version_id),
        CONSTRAINT FK_PlanTunnelScope_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id),
        CONSTRAINT CK_PlanTunnelScope_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_PlanTunnelScope_unique' AND object_id = OBJECT_ID(N'tunnel.PlanTunnelScope'))
BEGIN
    CREATE UNIQUE INDEX UX_PlanTunnelScope_unique ON tunnel.PlanTunnelScope(plan_version_id, tunnel_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_PlanTunnelScope_tunnel' AND object_id = OBJECT_ID(N'tunnel.PlanTunnelScope'))
BEGIN
    CREATE INDEX IX_PlanTunnelScope_tunnel ON tunnel.PlanTunnelScope(tunnel_id);
END;


/* CODESCHEME
   Esquema de codificación de incidentes. Permite soportar códigos locales como 100-TRA o esquemas
   de otros países/operadores.
*/
IF OBJECT_ID(N'tunnel.CodeScheme', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.CodeScheme (
        -- code_scheme_id: Identificador técnico único del esquema de códigos.
        code_scheme_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_CodeScheme PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- name: Nombre del esquema de codificación.
        name NVARCHAR(120) NOT NULL,
        -- description: Descripción funcional del esquema.
        description NVARCHAR(400) NULL,
        -- pattern_hint: Pista de patrón o formato, por ejemplo una regex o estructura esperada.
        pattern_hint NVARCHAR(200) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT CK_CodeScheme_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_CodeScheme_name' AND object_id = OBJECT_ID(N'tunnel.CodeScheme'))
BEGIN
    CREATE UNIQUE INDEX UX_CodeScheme_name ON tunnel.CodeScheme(name);
END;


/* EMERGENCYLEVEL
   Nivel de gravedad o activación: prealerta, alerta, emergencia u otros niveles configurables.
*/
IF OBJECT_ID(N'tunnel.EmergencyLevel', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.EmergencyLevel (
        -- emergency_level_id: Identificador técnico único del nivel de emergencia.
        emergency_level_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_EmergencyLevel PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- name: Nombre del nivel.
        name NVARCHAR(30) NOT NULL,
        -- rank: Orden de gravedad o prioridad. A mayor valor, mayor nivel si así se configura.
        rank INT NOT NULL,
        -- description: Descripción operativa del nivel.
        description NVARCHAR(400) NULL
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_EmergencyLevel_rank' AND object_id = OBJECT_ID(N'tunnel.EmergencyLevel'))
BEGIN
    CREATE UNIQUE INDEX UX_EmergencyLevel_rank ON tunnel.EmergencyLevel(rank);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_EmergencyLevel_name' AND object_id = OBJECT_ID(N'tunnel.EmergencyLevel'))
BEGIN
    CREATE UNIQUE INDEX UX_EmergencyLevel_name ON tunnel.EmergencyLevel(name);
END;


/* INCIDENTFAMILY
   Familia funcional de incidentes: tráfico, avería, incendio, ambiental, iluminación u otras.
*/
IF OBJECT_ID(N'tunnel.IncidentFamily', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.IncidentFamily (
        -- incident_family_id: Identificador técnico único de la familia.
        incident_family_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_IncidentFamily PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- code: Código corto de familia, por ejemplo TRA, AVA, FOC, AMB o ILI.
        code NVARCHAR(20) NOT NULL,
        -- name: Nombre de la familia.
        name NVARCHAR(120) NOT NULL,
        -- description: Descripción de los incidentes incluidos en la familia.
        description NVARCHAR(400) NULL
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_IncidentFamily_code' AND object_id = OBJECT_ID(N'tunnel.IncidentFamily'))
BEGIN
    CREATE UNIQUE INDEX UX_IncidentFamily_code ON tunnel.IncidentFamily(code);
END;


/* INCIDENTTYPE
   Tipo de incidente catalogado. Es la clasificación que permite seleccionar el protocolo
   adecuado.
*/
IF OBJECT_ID(N'tunnel.IncidentType', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.IncidentType (
        -- incident_type_id: Identificador técnico único del tipo de incidente.
        incident_type_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_IncidentType PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- code_scheme_id: Esquema de codificación al que pertenece el código.
        code_scheme_id UNIQUEIDENTIFIER NOT NULL,
        -- emergency_level_id: Nivel de emergencia asociado al tipo.
        emergency_level_id UNIQUEIDENTIFIER NOT NULL,
        -- incident_family_id: Familia funcional del incidente.
        incident_family_id UNIQUEIDENTIFIER NOT NULL,
        -- code_raw: Código textual del incidente, por ejemplo 260-AVA.
        code_raw NVARCHAR(40) NOT NULL,
        -- title: Título operativo del incidente.
        title NVARCHAR(200) NOT NULL,
        -- description: Descripción completa del tipo de incidente.
        description NVARCHAR(MAX) NOT NULL,
        -- detection_notes: Notas sobre cómo detectar, verificar o confirmar el incidente.
        detection_notes NVARCHAR(MAX) NULL,
        -- operational_context: Contexto: NORMAL, WORKS, EVENT u OTHER.
        operational_context NVARCHAR(20) NOT NULL CONSTRAINT DF_IncidentType_context DEFAULT ('NORMAL'),
        -- info_to_collect_json: Campos o checklist que el operador debe recopilar, en JSON.
        info_to_collect_json NVARCHAR(MAX) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_IncidentType_CodeScheme FOREIGN KEY (code_scheme_id) REFERENCES tunnel.CodeScheme(code_scheme_id),
        CONSTRAINT FK_IncidentType_EmergencyLevel FOREIGN KEY (emergency_level_id) REFERENCES tunnel.EmergencyLevel(emergency_level_id),
        CONSTRAINT FK_IncidentType_IncidentFamily FOREIGN KEY (incident_family_id) REFERENCES tunnel.IncidentFamily(incident_family_id),
        CONSTRAINT CK_IncidentType_context CHECK (operational_context IN ('NORMAL','WORKS','EVENT','OTHER')),
        CONSTRAINT CK_IncidentType_info_json CHECK (info_to_collect_json IS NULL OR ISJSON(info_to_collect_json) = 1),
        CONSTRAINT CK_IncidentType_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_IncidentType_scheme_code' AND object_id = OBJECT_ID(N'tunnel.IncidentType'))
BEGIN
    CREATE UNIQUE INDEX UX_IncidentType_scheme_code ON tunnel.IncidentType(code_scheme_id, code_raw);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_IncidentType_level' AND object_id = OBJECT_ID(N'tunnel.IncidentType'))
BEGIN
    CREATE INDEX IX_IncidentType_level ON tunnel.IncidentType(emergency_level_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_IncidentType_family' AND object_id = OBJECT_ID(N'tunnel.IncidentType'))
BEGIN
    CREATE INDEX IX_IncidentType_family ON tunnel.IncidentType(incident_family_id);
END;


/* PROTOCOL
   Plantilla de actuación asociada a una versión de plan y a un tipo de incidente.
*/
IF OBJECT_ID(N'tunnel.Protocol', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Protocol (
        -- protocol_id: Identificador técnico único del protocolo.
        protocol_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Protocol PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- plan_version_id: Versión del plan que define el protocolo.
        plan_version_id UNIQUEIDENTIFIER NOT NULL,
        -- incident_type_id: Tipo de incidente gestionado por el protocolo.
        incident_type_id UNIQUEIDENTIFIER NOT NULL,
        -- name: Nombre operativo del protocolo.
        name NVARCHAR(200) NOT NULL,
        -- objective: Objetivo resumido del protocolo.
        objective NVARCHAR(400) NULL,
        -- is_remote_executable: Indica si puede ejecutarse desde centro de control o consola
        -- remota.
        is_remote_executable BIT NOT NULL CONSTRAINT DF_Protocol_remote DEFAULT (1),
        -- protocol_status: Estado del protocolo: ACTIVE, RETIRED o DRAFT.
        protocol_status NVARCHAR(12) NOT NULL CONSTRAINT DF_Protocol_status DEFAULT ('ACTIVE'),
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_Protocol_PlanVersion FOREIGN KEY (plan_version_id) REFERENCES tunnel.PlanVersion(plan_version_id),
        CONSTRAINT FK_Protocol_IncidentType FOREIGN KEY (incident_type_id) REFERENCES tunnel.IncidentType(incident_type_id),
        CONSTRAINT CK_Protocol_status CHECK (protocol_status IN ('ACTIVE','RETIRED','DRAFT')),
        CONSTRAINT CK_Protocol_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Protocol_unique' AND object_id = OBJECT_ID(N'tunnel.Protocol'))
BEGIN
    CREATE UNIQUE INDEX UX_Protocol_unique ON tunnel.Protocol(plan_version_id, incident_type_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Protocol_planversion' AND object_id = OBJECT_ID(N'tunnel.Protocol'))
BEGIN
    CREATE INDEX IX_Protocol_planversion ON tunnel.Protocol(plan_version_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Protocol_incidenttype' AND object_id = OBJECT_ID(N'tunnel.Protocol'))
BEGIN
    CREATE INDEX IX_Protocol_incidenttype ON tunnel.Protocol(incident_type_id);
END;


/* PROTOCOLSTEP
   Nodo del flujo de trabajo de un protocolo: decisión, acción, espera, información o checklist.
*/
IF OBJECT_ID(N'tunnel.ProtocolStep', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ProtocolStep (
        -- protocol_step_id: Identificador técnico único del paso.
        protocol_step_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ProtocolStep PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- protocol_id: Protocolo al que pertenece el paso.
        protocol_id UNIQUEIDENTIFIER NOT NULL,
        -- step_key: Clave estable del paso para referenciarlo desde configuración o interfaz.
        step_key NVARCHAR(80) NOT NULL,
        -- step_type: Tipo de paso: DECISION, ACTION, INFO, WAIT o CHECKLIST.
        step_type NVARCHAR(12) NOT NULL,
        -- title: Título visible del paso.
        title NVARCHAR(200) NOT NULL,
        -- instructions: Instrucciones operativas que debe seguir el operador o el sistema.
        instructions NVARCHAR(MAX) NOT NULL,
        -- requires_ack: Indica si el operador debe confirmar el paso.
        requires_ack BIT NOT NULL CONSTRAINT DF_ProtocolStep_ack DEFAULT (1),
        -- timeout_seconds: Tiempo máximo recomendado antes de escalar o disparar otra transición.
        timeout_seconds INT NULL,
        -- ui_form_schema_json: Esquema JSON para pintar formularios dinámicos en la interfaz.
        ui_form_schema_json NVARCHAR(MAX) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_ProtocolStep_Protocol FOREIGN KEY (protocol_id) REFERENCES tunnel.Protocol(protocol_id),
        CONSTRAINT CK_ProtocolStep_type CHECK (step_type IN ('DECISION','ACTION','INFO','WAIT','CHECKLIST')),
        CONSTRAINT CK_ProtocolStep_ui_json CHECK (ui_form_schema_json IS NULL OR ISJSON(ui_form_schema_json) = 1),
        CONSTRAINT CK_ProtocolStep_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1),
        CONSTRAINT UQ_ProtocolStep_id_protocol UNIQUE (protocol_step_id, protocol_id)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_ProtocolStep_protocol_stepkey' AND object_id = OBJECT_ID(N'tunnel.ProtocolStep'))
BEGIN
    CREATE UNIQUE INDEX UX_ProtocolStep_protocol_stepkey ON tunnel.ProtocolStep(protocol_id, step_key);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_ProtocolStep_protocol' AND object_id = OBJECT_ID(N'tunnel.ProtocolStep'))
BEGIN
    CREATE INDEX IX_ProtocolStep_protocol ON tunnel.ProtocolStep(protocol_id);
END;


/* STEPTRANSITION
   Transición dirigida entre pasos de un protocolo. Permite modelar decisiones condicionales,
   ramas y rutas alternativas.
*/
IF OBJECT_ID(N'tunnel.StepTransition', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.StepTransition (
        -- step_transition_id: Identificador técnico único de la transición.
        step_transition_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_StepTransition PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- protocol_id: Protocolo al que pertenece la transición.
        protocol_id UNIQUEIDENTIFIER NOT NULL,
        -- from_step_id: Paso origen.
        from_step_id UNIQUEIDENTIFIER NOT NULL,
        -- to_step_id: Paso destino.
        to_step_id UNIQUEIDENTIFIER NOT NULL,
        -- condition_expr: Expresión de condición o regla de negocio. Si es NULL puede
        -- interpretarse como transición por defecto.
        condition_expr NVARCHAR(MAX) NULL,
        -- priority: Prioridad de evaluación cuando hay varias transiciones posibles.
        priority INT NOT NULL CONSTRAINT DF_StepTransition_priority DEFAULT (0),
        -- label: Texto visible para la transición, por ejemplo Sí, No, Confirmado o Escalar.
        label NVARCHAR(200) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_StepTransition_Protocol FOREIGN KEY (protocol_id) REFERENCES tunnel.Protocol(protocol_id),
        CONSTRAINT FK_StepTransition_FromStep FOREIGN KEY (from_step_id, protocol_id) REFERENCES tunnel.ProtocolStep(protocol_step_id, protocol_id),
        CONSTRAINT FK_StepTransition_ToStep FOREIGN KEY (to_step_id, protocol_id) REFERENCES tunnel.ProtocolStep(protocol_step_id, protocol_id),
        CONSTRAINT CK_StepTransition_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1),
        CONSTRAINT CK_StepTransition_not_self CHECK (from_step_id <> to_step_id)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_StepTransition_protocol_from' AND object_id = OBJECT_ID(N'tunnel.StepTransition'))
BEGIN
    CREATE INDEX IX_StepTransition_protocol_from ON tunnel.StepTransition(protocol_id, from_step_id);
END;


/* ACTIONDEFINITION
   Catálogo de acciones atómicas ejecutables o registrables: señalización, semáforos, cierre,
   ventilación, iluminación, megafonía, orden de trabajo, aviso externo, etc.
*/
IF OBJECT_ID(N'tunnel.ActionDefinition', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ActionDefinition (
        -- action_definition_id: Identificador técnico único de la acción definida.
        action_definition_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ActionDefinition PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- action_type: Tipo de acción: SET_SIGNAGE, SET_SEMAPHORE, CLOSE_TUBE, VENTILATION_MODE,
        -- LIGHTING_MODE, PA_ANNOUNCEMENT, CREATE_WORK_ORDER, REQUEST_EXTERNAL, LOG_ONLY u OTHER.
        action_type NVARCHAR(30) NOT NULL,
        -- name: Nombre operativo de la acción.
        name NVARCHAR(200) NOT NULL,
        -- description: Descripción de qué hace la acción.
        description NVARCHAR(400) NULL,
        -- payload_schema_json: Esquema JSON de los parámetros necesarios para ejecutar la acción.
        payload_schema_json NVARCHAR(MAX) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT CK_ActionDefinition_type CHECK (action_type IN ('SET_SIGNAGE','SET_SEMAPHORE','CLOSE_TUBE','VENTILATION_MODE','LIGHTING_MODE','PA_ANNOUNCEMENT','CREATE_WORK_ORDER','REQUEST_EXTERNAL','LOG_ONLY','OTHER')),
        CONSTRAINT CK_ActionDefinition_payload_json CHECK (payload_schema_json IS NULL OR ISJSON(payload_schema_json) = 1),
        CONSTRAINT CK_ActionDefinition_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_ActionDefinition_name' AND object_id = OBJECT_ID(N'tunnel.ActionDefinition'))
BEGIN
    CREATE UNIQUE INDEX UX_ActionDefinition_name ON tunnel.ActionDefinition(name);
END;


/* STEPACTION
   Relación entre un paso de protocolo y una acción definida. Permite ejecutar varias acciones
   ordenadas por cada paso.
*/
IF OBJECT_ID(N'tunnel.StepAction', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.StepAction (
        -- step_action_id: Identificador técnico único de la relación paso-acción.
        step_action_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_StepAction PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- protocol_step_id: Paso de protocolo que dispara la acción.
        protocol_step_id UNIQUEIDENTIFIER NOT NULL,
        -- action_definition_id: Acción definida que se debe ejecutar o registrar.
        action_definition_id UNIQUEIDENTIFIER NOT NULL,
        -- execution_order: Orden de ejecución dentro del paso.
        execution_order INT NOT NULL CONSTRAINT DF_StepAction_order DEFAULT (0),
        -- is_mandatory: Indica si la acción es obligatoria en el paso.
        is_mandatory BIT NOT NULL CONSTRAINT DF_StepAction_mandatory DEFAULT (1),
        -- parameter_binding_json: Mapeo JSON entre parámetros del protocolo/incidente y payload de
        -- acción.
        parameter_binding_json NVARCHAR(MAX) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_StepAction_ProtocolStep FOREIGN KEY (protocol_step_id) REFERENCES tunnel.ProtocolStep(protocol_step_id),
        CONSTRAINT FK_StepAction_ActionDefinition FOREIGN KEY (action_definition_id) REFERENCES tunnel.ActionDefinition(action_definition_id),
        CONSTRAINT CK_StepAction_binding_json CHECK (parameter_binding_json IS NULL OR ISJSON(parameter_binding_json) = 1),
        CONSTRAINT CK_StepAction_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_StepAction_unique' AND object_id = OBJECT_ID(N'tunnel.StepAction'))
BEGIN
    CREATE UNIQUE INDEX UX_StepAction_unique ON tunnel.StepAction(protocol_step_id, action_definition_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_StepAction_step' AND object_id = OBJECT_ID(N'tunnel.StepAction'))
BEGIN
    CREATE INDEX IX_StepAction_step ON tunnel.StepAction(protocol_step_id);
END;


/* NOTIFICATIONRULE
   Regla o plantilla de aviso a organismos, centros de control, mantenimiento u otros contactos.
*/
IF OBJECT_ID(N'tunnel.NotificationRule', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.NotificationRule (
        -- notification_rule_id: Identificador técnico único de la regla de notificación.
        notification_rule_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_NotificationRule PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- name: Nombre de la regla.
        name NVARCHAR(200) NOT NULL,
        -- purpose: Finalidad del aviso.
        purpose NVARCHAR(400) NULL,
        -- default_channel: Canal por defecto: PHONE, EMAIL, RADIO, API, SMS u OTHER.
        default_channel NVARCHAR(12) NOT NULL,
        -- message_template: Plantilla del mensaje con variables sustituibles por la aplicación.
        message_template NVARCHAR(MAX) NOT NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT CK_NotificationRule_channel CHECK (default_channel IN ('PHONE','EMAIL','RADIO','API','SMS','OTHER')),
        CONSTRAINT CK_NotificationRule_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_NotificationRule_name' AND object_id = OBJECT_ID(N'tunnel.NotificationRule'))
BEGIN
    CREATE UNIQUE INDEX UX_NotificationRule_name ON tunnel.NotificationRule(name);
END;


/* STEPNOTIFICATION
   Relación entre un paso de protocolo y una regla de notificación. Define cuándo y bajo qué
   condición se envía un aviso.
*/
IF OBJECT_ID(N'tunnel.StepNotification', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.StepNotification (
        -- step_notification_id: Identificador técnico único de la relación paso-notificación.
        step_notification_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_StepNotification PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- protocol_step_id: Paso de protocolo que dispara la notificación.
        protocol_step_id UNIQUEIDENTIFIER NOT NULL,
        -- notification_rule_id: Regla de notificación a aplicar.
        notification_rule_id UNIQUEIDENTIFIER NOT NULL,
        -- when: Momento de disparo: ON_ENTER, ON_EXIT, ON_TIMEOUT u ON_CONDITION.
        [when] NVARCHAR(12) NOT NULL,
        -- condition_expr: Condición adicional para disparar el aviso.
        condition_expr NVARCHAR(MAX) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_StepNotification_ProtocolStep FOREIGN KEY (protocol_step_id) REFERENCES tunnel.ProtocolStep(protocol_step_id),
        CONSTRAINT FK_StepNotification_NotificationRule FOREIGN KEY (notification_rule_id) REFERENCES tunnel.NotificationRule(notification_rule_id),
        CONSTRAINT CK_StepNotification_when CHECK ([when] IN ('ON_ENTER','ON_EXIT','ON_TIMEOUT','ON_CONDITION')),
        CONSTRAINT CK_StepNotification_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_StepNotification_unique' AND object_id = OBJECT_ID(N'tunnel.StepNotification'))
BEGIN
    CREATE UNIQUE INDEX UX_StepNotification_unique ON tunnel.StepNotification(protocol_step_id, notification_rule_id, [when]);
END;


/* PARAMETERDEFINITION
   Definición global de un parámetro configurable: umbrales, límites, tiempos, modos o reglas
   reutilizables.
*/
IF OBJECT_ID(N'tunnel.ParameterDefinition', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ParameterDefinition (
        -- parameter_definition_id: Identificador técnico único de la definición de parámetro.
        parameter_definition_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ParameterDefinition PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- key: Clave única del parámetro, estable para uso en código y configuración.
        [key] NVARCHAR(120) NOT NULL,
        -- data_type: Tipo de dato esperado: INT, DECIMAL, BOOLEAN, TEXT o JSON.
        data_type NVARCHAR(10) NOT NULL,
        -- unit: Unidad del parámetro si aplica, por ejemplo segundos, ppm, km/h o porcentaje.
        unit NVARCHAR(32) NULL,
        -- description: Descripción funcional del parámetro.
        description NVARCHAR(400) NOT NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT CK_ParameterDefinition_type CHECK (data_type IN ('INT','DECIMAL','BOOLEAN','TEXT','JSON')),
        CONSTRAINT CK_ParameterDefinition_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_ParameterDefinition_key' AND object_id = OBJECT_ID(N'tunnel.ParameterDefinition'))
BEGIN
    CREATE UNIQUE INDEX UX_ParameterDefinition_key ON tunnel.ParameterDefinition([key]);
END;


/* PARAMETERSET
   Colección de valores de parámetros aplicable a un túnel o a una versión de plan. Sirve para
   overrides y personalización.
*/
IF OBJECT_ID(N'tunnel.ParameterSet', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ParameterSet (
        -- parameter_set_id: Identificador técnico único del conjunto de parámetros.
        parameter_set_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ParameterSet PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- tunnel_id: Túnel al que aplica el conjunto, si es un set por túnel.
        tunnel_id UNIQUEIDENTIFIER NULL,
        -- plan_version_id: Versión de plan a la que aplica el conjunto, si es un set por plan.
        plan_version_id UNIQUEIDENTIFIER NULL,
        -- name: Nombre del conjunto de parámetros.
        name NVARCHAR(200) NOT NULL,
        -- priority: Prioridad para resolver overrides cuando hay varios conjuntos aplicables.
        priority INT NOT NULL CONSTRAINT DF_ParameterSet_priority DEFAULT (0),
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        -- created_at_utc: Fecha y hora UTC de creación del registro.
        created_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_ParameterSet_created DEFAULT SYSUTCDATETIME(),
        CONSTRAINT FK_ParameterSet_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id),
        CONSTRAINT FK_ParameterSet_PlanVersion FOREIGN KEY (plan_version_id) REFERENCES tunnel.PlanVersion(plan_version_id),
        CONSTRAINT CK_ParameterSet_owner CHECK ((CASE WHEN tunnel_id IS NULL THEN 0 ELSE 1 END) + (CASE WHEN plan_version_id IS NULL THEN 0 ELSE 1 END) = 1),
        CONSTRAINT CK_ParameterSet_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_ParameterSet_tunnel' AND object_id = OBJECT_ID(N'tunnel.ParameterSet'))
BEGIN
    CREATE INDEX IX_ParameterSet_tunnel ON tunnel.ParameterSet(tunnel_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_ParameterSet_planversion' AND object_id = OBJECT_ID(N'tunnel.ParameterSet'))
BEGIN
    CREATE INDEX IX_ParameterSet_planversion ON tunnel.ParameterSet(plan_version_id);
END;


/* PARAMETERVALUE
   Valor concreto de un parámetro dentro de un conjunto. Usa columnas tipadas para mantener
   compatibilidad y facilitar validación.
*/
IF OBJECT_ID(N'tunnel.ParameterValue', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ParameterValue (
        -- parameter_value_id: Identificador técnico único del valor de parámetro.
        parameter_value_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ParameterValue PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- parameter_set_id: Conjunto de parámetros al que pertenece el valor.
        parameter_set_id UNIQUEIDENTIFIER NOT NULL,
        -- parameter_definition_id: Definición de parámetro que se está valorando.
        parameter_definition_id UNIQUEIDENTIFIER NOT NULL,
        -- value_int: Valor entero si el parámetro es de tipo INT.
        value_int INT NULL,
        -- value_decimal: Valor decimal si el parámetro es de tipo DECIMAL.
        value_decimal DECIMAL(18,6) NULL,
        -- value_bool: Valor booleano si el parámetro es de tipo BOOLEAN.
        value_bool BIT NULL,
        -- value_text: Valor textual si el parámetro es de tipo TEXT.
        value_text NVARCHAR(MAX) NULL,
        -- value_json: Valor JSON si el parámetro es de tipo JSON.
        value_json NVARCHAR(MAX) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_ParameterValue_ParameterSet FOREIGN KEY (parameter_set_id) REFERENCES tunnel.ParameterSet(parameter_set_id),
        CONSTRAINT FK_ParameterValue_ParameterDefinition FOREIGN KEY (parameter_definition_id) REFERENCES tunnel.ParameterDefinition(parameter_definition_id),
        CONSTRAINT CK_ParameterValue_value_json CHECK (value_json IS NULL OR ISJSON(value_json) = 1),
        CONSTRAINT CK_ParameterValue_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_ParameterValue_unique' AND object_id = OBJECT_ID(N'tunnel.ParameterValue'))
BEGIN
    CREATE UNIQUE INDEX UX_ParameterValue_unique ON tunnel.ParameterValue(parameter_set_id, parameter_definition_id);
END;


/* PROTOCOLPARAMETER
   Relación documental entre protocolo y parámetros que utiliza. Ayuda a validar configuración
   antes de ejecutar protocolos.
*/
IF OBJECT_ID(N'tunnel.ProtocolParameter', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ProtocolParameter (
        -- protocol_parameter_id: Identificador técnico único de la relación protocolo-parámetro.
        protocol_parameter_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ProtocolParameter PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- protocol_id: Protocolo que utiliza el parámetro.
        protocol_id UNIQUEIDENTIFIER NOT NULL,
        -- parameter_definition_id: Parámetro requerido o utilizado por el protocolo.
        parameter_definition_id UNIQUEIDENTIFIER NOT NULL,
        -- usage_note: Nota que explica cómo usa el protocolo este parámetro.
        usage_note NVARCHAR(400) NULL,
        CONSTRAINT FK_ProtocolParameter_Protocol FOREIGN KEY (protocol_id) REFERENCES tunnel.Protocol(protocol_id),
        CONSTRAINT FK_ProtocolParameter_ParameterDefinition FOREIGN KEY (parameter_definition_id) REFERENCES tunnel.ParameterDefinition(parameter_definition_id)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_ProtocolParameter_unique' AND object_id = OBJECT_ID(N'tunnel.ProtocolParameter'))
BEGIN
    CREATE UNIQUE INDEX UX_ProtocolParameter_unique ON tunnel.ProtocolParameter(protocol_id, parameter_definition_id);
END;


/* ROLE
   Rol operativo o administrativo de un usuario: operador, jefe de turno, mantenimiento,
   supervisor, etc.
*/
IF OBJECT_ID(N'tunnel.Role', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.[Role] (
        -- role_id: Identificador técnico único del rol.
        role_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Role PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- name: Nombre único del rol.
        name NVARCHAR(80) NOT NULL,
        -- description: Descripción de permisos o responsabilidades del rol.
        description NVARCHAR(400) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT CK_Role_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Role_name' AND object_id = OBJECT_ID(N'tunnel.Role'))
BEGIN
    CREATE UNIQUE INDEX UX_Role_name ON tunnel.[Role](name);
END;


/* USERACCOUNT
   Usuario de la aplicación o consola operativa. La autenticación puede estar en la aplicación,
   pero aquí queda la identidad operativa.
*/
IF OBJECT_ID(N'tunnel.UserAccount', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.UserAccount (
        -- user_id: Identificador técnico único del usuario.
        user_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_UserAccount PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- facility_id: Ámbito operativo al que pertenece el usuario.
        facility_id UNIQUEIDENTIFIER NOT NULL,
        -- role_id: Rol asignado al usuario.
        role_id UNIQUEIDENTIFIER NOT NULL,
        -- username: Nombre de usuario o login.
        username NVARCHAR(120) NOT NULL,
        -- display_name: Nombre visible del usuario.
        display_name NVARCHAR(200) NOT NULL,
        -- phone: Teléfono de contacto.
        phone NVARCHAR(50) NULL,
        -- email: Correo electrónico.
        email NVARCHAR(200) NULL,
        -- is_active: Indica si el usuario está activo.
        is_active BIT NOT NULL CONSTRAINT DF_UserAccount_active DEFAULT (1),
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        -- created_at_utc: Fecha y hora UTC de creación del registro.
        created_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_UserAccount_created DEFAULT SYSUTCDATETIME(),
        CONSTRAINT FK_UserAccount_Facility FOREIGN KEY (facility_id) REFERENCES tunnel.Facility(facility_id),
        CONSTRAINT FK_UserAccount_Role FOREIGN KEY (role_id) REFERENCES tunnel.[Role](role_id),
        CONSTRAINT CK_UserAccount_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_UserAccount_facility_username' AND object_id = OBJECT_ID(N'tunnel.UserAccount'))
BEGIN
    CREATE UNIQUE INDEX UX_UserAccount_facility_username ON tunnel.UserAccount(facility_id, username);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_UserAccount_role' AND object_id = OBJECT_ID(N'tunnel.UserAccount'))
BEGIN
    CREATE INDEX IX_UserAccount_role ON tunnel.UserAccount(role_id);
END;


/* AGENCY
   Organismo o entidad interna/externa que puede ser avisada o participar en la respuesta:
   emergencias, tráfico, mantenimiento, centro de control, etc.
*/
IF OBJECT_ID(N'tunnel.Agency', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Agency (
        -- agency_id: Identificador técnico único del organismo.
        agency_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Agency PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- name: Nombre del organismo o entidad.
        name NVARCHAR(200) NOT NULL,
        -- agency_type: Tipo: EMERGENCY_SERVICES, TRAFFIC_AUTHORITY, CONTROL_CENTER, MAINTENANCE u
        -- OTHER.
        agency_type NVARCHAR(30) NOT NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT CK_Agency_type CHECK (agency_type IN ('EMERGENCY_SERVICES','TRAFFIC_AUTHORITY','CONTROL_CENTER','MAINTENANCE','OTHER')),
        CONSTRAINT CK_Agency_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_Agency_name' AND object_id = OBJECT_ID(N'tunnel.Agency'))
BEGIN
    CREATE UNIQUE INDEX UX_Agency_name ON tunnel.Agency(name);
END;


/* CONTACTPOINT
   Punto de contacto de un organismo: teléfono, radio, email, SMS, endpoint API u otro canal.
*/
IF OBJECT_ID(N'tunnel.ContactPoint', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ContactPoint (
        -- contact_point_id: Identificador técnico único del punto de contacto.
        contact_point_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ContactPoint PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- agency_id: Organismo al que pertenece el contacto.
        agency_id UNIQUEIDENTIFIER NOT NULL,
        -- name: Nombre o etiqueta del contacto.
        name NVARCHAR(200) NULL,
        -- channel: Canal: PHONE, EMAIL, RADIO, API, SMS u OTHER.
        channel NVARCHAR(12) NOT NULL,
        -- address: Valor del contacto: teléfono, email, endpoint, canal de radio, etc.
        [address] NVARCHAR(400) NOT NULL,
        -- availability: Disponibilidad horaria o condición de uso.
        availability NVARCHAR(200) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_ContactPoint_Agency FOREIGN KEY (agency_id) REFERENCES tunnel.Agency(agency_id),
        CONSTRAINT CK_ContactPoint_channel CHECK (channel IN ('PHONE','EMAIL','RADIO','API','SMS','OTHER')),
        CONSTRAINT CK_ContactPoint_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_ContactPoint_unique' AND object_id = OBJECT_ID(N'tunnel.ContactPoint'))
BEGIN
    CREATE UNIQUE INDEX UX_ContactPoint_unique ON tunnel.ContactPoint(agency_id, channel, [address]);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_ContactPoint_agency' AND object_id = OBJECT_ID(N'tunnel.ContactPoint'))
BEGIN
    CREATE INDEX IX_ContactPoint_agency ON tunnel.ContactPoint(agency_id);
END;


/* INCIDENTEVENT
   Incidente real registrado en explotación. Une túnel, tipo de incidente, estado, tiempos, notas
   y datos operativos.
*/
IF OBJECT_ID(N'tunnel.IncidentEvent', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.IncidentEvent (
        -- incident_event_id: Identificador técnico único del incidente real.
        incident_event_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_IncidentEvent PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- incident_type_id: Tipo de incidente catalogado.
        incident_type_id UNIQUEIDENTIFIER NOT NULL,
        -- tunnel_id: Túnel donde ocurre el incidente.
        tunnel_id UNIQUEIDENTIFIER NOT NULL,
        -- incident_status: Estado del incidente: OPEN, MITIGATING, RESOLVED o CLOSED.
        incident_status NVARCHAR(12) NOT NULL CONSTRAINT DF_IncidentEvent_status DEFAULT ('OPEN'),
        -- started_at_utc: Fecha/hora UTC de inicio o apertura del incidente.
        started_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_IncidentEvent_started DEFAULT SYSUTCDATETIME(),
        -- detected_at_utc: Fecha/hora UTC de detección si difiere de la apertura.
        detected_at_utc DATETIME2(3) NULL,
        -- resolved_at_utc: Fecha/hora UTC de resolución.
        resolved_at_utc DATETIME2(3) NULL,
        -- severity_override: Reclasificación manual de severidad si el operador la aplica.
        severity_override NVARCHAR(30) NULL,
        -- summary: Resumen corto del incidente.
        summary NVARCHAR(400) NULL,
        -- operator_notes: Notas libres del operador.
        operator_notes NVARCHAR(MAX) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_IncidentEvent_IncidentType FOREIGN KEY (incident_type_id) REFERENCES tunnel.IncidentType(incident_type_id),
        CONSTRAINT FK_IncidentEvent_Tunnel FOREIGN KEY (tunnel_id) REFERENCES tunnel.Tunnel(tunnel_id),
        CONSTRAINT CK_IncidentEvent_status CHECK (incident_status IN ('OPEN','MITIGATING','RESOLVED','CLOSED')),
        CONSTRAINT CK_IncidentEvent_times CHECK ((detected_at_utc IS NULL OR detected_at_utc >= started_at_utc) AND (resolved_at_utc IS NULL OR resolved_at_utc >= started_at_utc)),
        CONSTRAINT CK_IncidentEvent_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_IncidentEvent_tunnel' AND object_id = OBJECT_ID(N'tunnel.IncidentEvent'))
BEGIN
    CREATE INDEX IX_IncidentEvent_tunnel ON tunnel.IncidentEvent(tunnel_id, started_at_utc);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_IncidentEvent_type' AND object_id = OBJECT_ID(N'tunnel.IncidentEvent'))
BEGIN
    CREATE INDEX IX_IncidentEvent_type ON tunnel.IncidentEvent(incident_type_id);
END;


/* DETECTIONEVENT
   Evidencia o evento de detección asociado a un incidente: sensor, CCTV/DAI, alarma SCADA,
   llamada SOS, aviso externo u observación del operador.
*/
IF OBJECT_ID(N'tunnel.DetectionEvent', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.DetectionEvent (
        -- detection_event_id: Identificador técnico único del evento de detección.
        detection_event_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_DetectionEvent PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- incident_event_id: Incidente al que pertenece la evidencia.
        incident_event_id UNIQUEIDENTIFIER NOT NULL,
        -- source_type: Origen: SENSOR, CCTV_DAI, SCADA_ALARM, SOS_CALL, EXTERNAL_CALL u
        -- OPERATOR_OBS.
        source_type NVARCHAR(20) NOT NULL,
        -- source_asset_id: Activo que generó la detección, si aplica.
        source_asset_id UNIQUEIDENTIFIER NULL,
        -- reported_by: Persona, servicio, centro o sistema que reporta la detección.
        reported_by NVARCHAR(200) NULL,
        -- payload_json: Datos de detección en JSON: medidas, alarmas, valores, texto, etc.
        payload_json NVARCHAR(MAX) NULL,
        -- created_at_utc: Fecha/hora UTC de registro de la detección.
        created_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_DetectionEvent_created DEFAULT SYSUTCDATETIME(),
        CONSTRAINT FK_DetectionEvent_IncidentEvent FOREIGN KEY (incident_event_id) REFERENCES tunnel.IncidentEvent(incident_event_id),
        CONSTRAINT FK_DetectionEvent_Asset FOREIGN KEY (source_asset_id) REFERENCES tunnel.Asset(asset_id),
        CONSTRAINT CK_DetectionEvent_source CHECK (source_type IN ('SENSOR','CCTV_DAI','SCADA_ALARM','SOS_CALL','EXTERNAL_CALL','OPERATOR_OBS')),
        CONSTRAINT CK_DetectionEvent_payload_json CHECK (payload_json IS NULL OR ISJSON(payload_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_DetectionEvent_incident' AND object_id = OBJECT_ID(N'tunnel.DetectionEvent'))
BEGIN
    CREATE INDEX IX_DetectionEvent_incident ON tunnel.DetectionEvent(incident_event_id, created_at_utc);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_DetectionEvent_asset' AND object_id = OBJECT_ID(N'tunnel.DetectionEvent'))
BEGIN
    CREATE INDEX IX_DetectionEvent_asset ON tunnel.DetectionEvent(source_asset_id);
END;


/* INCIDENTLOCATION
   Relación N:M entre incidente y localización. Permite indicar varias ubicaciones, ubicación
   principal y grado de confianza.
*/
IF OBJECT_ID(N'tunnel.IncidentLocation', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.IncidentLocation (
        -- incident_location_id: Identificador técnico único de la relación incidente-localización.
        incident_location_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_IncidentLocation PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- incident_event_id: Incidente localizado.
        incident_event_id UNIQUEIDENTIFIER NOT NULL,
        -- location_id: Localización asociada al incidente.
        location_id UNIQUEIDENTIFIER NOT NULL,
        -- confidence: Confianza de la localización entre 0 y 1.
        confidence DECIMAL(4,3) NOT NULL CONSTRAINT DF_IncidentLocation_conf DEFAULT (1.000),
        -- is_primary: Indica si es la localización principal del incidente.
        is_primary BIT NOT NULL CONSTRAINT DF_IncidentLocation_primary DEFAULT (0),
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_IncidentLocation_IncidentEvent FOREIGN KEY (incident_event_id) REFERENCES tunnel.IncidentEvent(incident_event_id),
        CONSTRAINT FK_IncidentLocation_Location FOREIGN KEY (location_id) REFERENCES tunnel.Location(location_id),
        CONSTRAINT CK_IncidentLocation_conf CHECK (confidence >= 0 AND confidence <= 1),
        CONSTRAINT CK_IncidentLocation_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_IncidentLocation_unique' AND object_id = OBJECT_ID(N'tunnel.IncidentLocation'))
BEGIN
    CREATE UNIQUE INDEX UX_IncidentLocation_unique ON tunnel.IncidentLocation(incident_event_id, location_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_IncidentLocation_incident' AND object_id = OBJECT_ID(N'tunnel.IncidentLocation'))
BEGIN
    CREATE INDEX IX_IncidentLocation_incident ON tunnel.IncidentLocation(incident_event_id);
END;


/* PROTOCOLRUN
   Ejecución concreta de un protocolo para un incidente real. Guarda estado, usuario iniciador,
   paso actual y snapshot de contexto.
*/
IF OBJECT_ID(N'tunnel.ProtocolRun', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ProtocolRun (
        -- protocol_run_id: Identificador técnico único de la ejecución del protocolo.
        protocol_run_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ProtocolRun PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- incident_event_id: Incidente gestionado por esta ejecución.
        incident_event_id UNIQUEIDENTIFIER NOT NULL,
        -- protocol_id: Protocolo que se está ejecutando.
        protocol_id UNIQUEIDENTIFIER NOT NULL,
        -- started_by_user_id: Usuario que inició la ejecución.
        started_by_user_id UNIQUEIDENTIFIER NOT NULL,
        -- started_at_utc: Fecha/hora UTC de inicio de la ejecución.
        started_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_ProtocolRun_started DEFAULT SYSUTCDATETIME(),
        -- ended_at_utc: Fecha/hora UTC de finalización.
        ended_at_utc DATETIME2(3) NULL,
        -- run_status: Estado de ejecución: RUNNING, PAUSED, COMPLETED o ABORTED.
        run_status NVARCHAR(12) NOT NULL CONSTRAINT DF_ProtocolRun_status DEFAULT ('RUNNING'),
        -- current_step_id: Paso actual dentro del protocolo.
        current_step_id UNIQUEIDENTIFIER NULL,
        -- context_snapshot_json: Snapshot JSON de contexto: parámetros resueltos, estado de
        -- activos, datos del incidente, etc.
        context_snapshot_json NVARCHAR(MAX) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_ProtocolRun_IncidentEvent FOREIGN KEY (incident_event_id) REFERENCES tunnel.IncidentEvent(incident_event_id),
        CONSTRAINT FK_ProtocolRun_Protocol FOREIGN KEY (protocol_id) REFERENCES tunnel.Protocol(protocol_id),
        CONSTRAINT FK_ProtocolRun_User FOREIGN KEY (started_by_user_id) REFERENCES tunnel.UserAccount(user_id),
        CONSTRAINT CK_ProtocolRun_status CHECK (run_status IN ('RUNNING','PAUSED','COMPLETED','ABORTED')),
        CONSTRAINT CK_ProtocolRun_context_json CHECK (context_snapshot_json IS NULL OR ISJSON(context_snapshot_json) = 1),
        CONSTRAINT CK_ProtocolRun_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_ProtocolRun_incident' AND object_id = OBJECT_ID(N'tunnel.ProtocolRun'))
BEGIN
    CREATE INDEX IX_ProtocolRun_incident ON tunnel.ProtocolRun(incident_event_id, started_at_utc);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_ProtocolRun_protocol' AND object_id = OBJECT_ID(N'tunnel.ProtocolRun'))
BEGIN
    CREATE INDEX IX_ProtocolRun_protocol ON tunnel.ProtocolRun(protocol_id);
END;


/* ACTIONEXECUTION
   Acción concreta solicitada, enviada, ejecutada o fallida durante una ejecución de protocolo.
*/
IF OBJECT_ID(N'tunnel.ActionExecution', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ActionExecution (
        -- action_execution_id: Identificador técnico único de la ejecución de acción.
        action_execution_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ActionExecution PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- protocol_run_id: Ejecución de protocolo a la que pertenece.
        protocol_run_id UNIQUEIDENTIFIER NOT NULL,
        -- action_definition_id: Acción definida que se ejecuta.
        action_definition_id UNIQUEIDENTIFIER NOT NULL,
        -- step_id: Paso que disparó la acción.
        step_id UNIQUEIDENTIFIER NULL,
        -- requested_at_utc: Fecha/hora UTC en que se solicitó la acción.
        requested_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_ActionExecution_req DEFAULT SYSUTCDATETIME(),
        -- executed_at_utc: Fecha/hora UTC en que se ejecutó o se confirmó.
        executed_at_utc DATETIME2(3) NULL,
        -- exec_status: Estado: REQUESTED, SENT, SUCCESS, FAILED o CANCELLED.
        exec_status NVARCHAR(12) NOT NULL CONSTRAINT DF_ActionExecution_status DEFAULT ('REQUESTED'),
        -- payload_json: Payload JSON real enviado o registrado para ejecutar la acción.
        payload_json NVARCHAR(MAX) NOT NULL,
        -- result_json: Resultado JSON devuelto por el sistema o integración.
        result_json NVARCHAR(MAX) NULL,
        -- error_message: Mensaje de error si la acción falla.
        error_message NVARCHAR(400) NULL,
        CONSTRAINT FK_ActionExecution_Run FOREIGN KEY (protocol_run_id) REFERENCES tunnel.ProtocolRun(protocol_run_id),
        CONSTRAINT FK_ActionExecution_Def FOREIGN KEY (action_definition_id) REFERENCES tunnel.ActionDefinition(action_definition_id),
        CONSTRAINT FK_ActionExecution_Step FOREIGN KEY (step_id) REFERENCES tunnel.ProtocolStep(protocol_step_id),
        CONSTRAINT CK_ActionExecution_status CHECK (exec_status IN ('REQUESTED','SENT','SUCCESS','FAILED','CANCELLED')),
        CONSTRAINT CK_ActionExecution_payload_json CHECK (ISJSON(payload_json) = 1),
        CONSTRAINT CK_ActionExecution_result_json CHECK (result_json IS NULL OR ISJSON(result_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_ActionExecution_run' AND object_id = OBJECT_ID(N'tunnel.ActionExecution'))
BEGIN
    CREATE INDEX IX_ActionExecution_run ON tunnel.ActionExecution(protocol_run_id, requested_at_utc);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_ActionExecution_step' AND object_id = OBJECT_ID(N'tunnel.ActionExecution'))
BEGIN
    CREATE INDEX IX_ActionExecution_step ON tunnel.ActionExecution(step_id);
END;


/* ACTIONTARGET
   Equipos o activos afectados por una acción concreta. Permite una acción sobre múltiples
   objetivos.
*/
IF OBJECT_ID(N'tunnel.ActionTarget', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.ActionTarget (
        -- action_target_id: Identificador técnico único del objetivo de acción.
        action_target_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_ActionTarget PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- action_execution_id: Ejecución de acción que afecta al activo.
        action_execution_id UNIQUEIDENTIFIER NOT NULL,
        -- asset_id: Activo objetivo de la acción.
        asset_id UNIQUEIDENTIFIER NOT NULL,
        -- target_role: Papel del activo en la acción, por ejemplo PMV entrada o ventilador sector
        -- 2.
        target_role NVARCHAR(200) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_ActionTarget_ActionExecution FOREIGN KEY (action_execution_id) REFERENCES tunnel.ActionExecution(action_execution_id),
        CONSTRAINT FK_ActionTarget_Asset FOREIGN KEY (asset_id) REFERENCES tunnel.Asset(asset_id),
        CONSTRAINT CK_ActionTarget_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'UX_ActionTarget_unique' AND object_id = OBJECT_ID(N'tunnel.ActionTarget'))
BEGIN
    CREATE UNIQUE INDEX UX_ActionTarget_unique ON tunnel.ActionTarget(action_execution_id, asset_id);
END;


/* NOTIFICATION
   Notificación real enviada o pendiente dentro de una ejecución de protocolo.
*/
IF OBJECT_ID(N'tunnel.Notification', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.Notification (
        -- notification_id: Identificador técnico único de la notificación.
        notification_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_Notification PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- protocol_run_id: Ejecución de protocolo que origina la notificación.
        protocol_run_id UNIQUEIDENTIFIER NOT NULL,
        -- contact_point_id: Punto de contacto destinatario.
        contact_point_id UNIQUEIDENTIFIER NOT NULL,
        -- notification_rule_id: Regla de notificación aplicada, si procede.
        notification_rule_id UNIQUEIDENTIFIER NULL,
        -- sent_at_utc: Fecha/hora UTC de envío.
        sent_at_utc DATETIME2(3) NULL,
        -- notif_status: Estado: PENDING, SENT, FAILED o ACKED.
        notif_status NVARCHAR(10) NOT NULL CONSTRAINT DF_Notification_status DEFAULT ('PENDING'),
        -- message: Mensaje final enviado o preparado.
        message NVARCHAR(MAX) NOT NULL,
        -- ack_at_utc: Fecha/hora UTC de acuse o confirmación.
        ack_at_utc DATETIME2(3) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_Notification_Run FOREIGN KEY (protocol_run_id) REFERENCES tunnel.ProtocolRun(protocol_run_id),
        CONSTRAINT FK_Notification_ContactPoint FOREIGN KEY (contact_point_id) REFERENCES tunnel.ContactPoint(contact_point_id),
        CONSTRAINT FK_Notification_Rule FOREIGN KEY (notification_rule_id) REFERENCES tunnel.NotificationRule(notification_rule_id),
        CONSTRAINT CK_Notification_status CHECK (notif_status IN ('PENDING','SENT','FAILED','ACKED')),
        CONSTRAINT CK_Notification_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Notification_run' AND object_id = OBJECT_ID(N'tunnel.Notification'))
BEGIN
    CREATE INDEX IX_Notification_run ON tunnel.Notification(protocol_run_id);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_Notification_contact' AND object_id = OBJECT_ID(N'tunnel.Notification'))
BEGIN
    CREATE INDEX IX_Notification_contact ON tunnel.Notification(contact_point_id);
END;


/* WORKORDER
   Orden de trabajo o mantenimiento asociada a un activo y opcionalmente generada desde un
   incidente.
*/
IF OBJECT_ID(N'tunnel.WorkOrder', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.WorkOrder (
        -- work_order_id: Identificador técnico único de la orden de trabajo.
        work_order_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_WorkOrder PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- asset_id: Activo afectado por la orden.
        asset_id UNIQUEIDENTIFIER NOT NULL,
        -- created_from_incident_id: Incidente que originó la orden si aplica.
        created_from_incident_id UNIQUEIDENTIFIER NULL,
        -- priority: Prioridad: LOW, MEDIUM, HIGH o URGENT.
        priority NVARCHAR(10) NOT NULL CONSTRAINT DF_WorkOrder_priority DEFAULT ('MEDIUM'),
        -- work_status: Estado: OPEN, IN_PROGRESS, DONE o CANCELLED.
        work_status NVARCHAR(12) NOT NULL CONSTRAINT DF_WorkOrder_status DEFAULT ('OPEN'),
        -- description: Descripción del trabajo o incidencia técnica.
        description NVARCHAR(MAX) NOT NULL,
        -- created_at_utc: Fecha/hora UTC de creación de la orden.
        created_at_utc DATETIME2(3) NOT NULL CONSTRAINT DF_WorkOrder_created DEFAULT SYSUTCDATETIME(),
        -- closed_at_utc: Fecha/hora UTC de cierre.
        closed_at_utc DATETIME2(3) NULL,
        -- meta_json: Datos adicionales flexibles en JSON.
        meta_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_WorkOrder_Asset FOREIGN KEY (asset_id) REFERENCES tunnel.Asset(asset_id),
        CONSTRAINT FK_WorkOrder_Incident FOREIGN KEY (created_from_incident_id) REFERENCES tunnel.IncidentEvent(incident_event_id),
        CONSTRAINT CK_WorkOrder_priority CHECK (priority IN ('LOW','MEDIUM','HIGH','URGENT')),
        CONSTRAINT CK_WorkOrder_status CHECK (work_status IN ('OPEN','IN_PROGRESS','DONE','CANCELLED')),
        CONSTRAINT CK_WorkOrder_meta_json CHECK (meta_json IS NULL OR ISJSON(meta_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_WorkOrder_asset' AND object_id = OBJECT_ID(N'tunnel.WorkOrder'))
BEGIN
    CREATE INDEX IX_WorkOrder_asset ON tunnel.WorkOrder(asset_id, created_at_utc);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_WorkOrder_incident' AND object_id = OBJECT_ID(N'tunnel.WorkOrder'))
BEGIN
    CREATE INDEX IX_WorkOrder_incident ON tunnel.WorkOrder(created_from_incident_id);
END;


/* AUDITLOG
   Registro de auditoría legal/operativa: cambios, ejecuciones, overrides y acciones relevantes
   del usuario o del sistema.
*/
IF OBJECT_ID(N'tunnel.AuditLog', N'U') IS NULL
BEGIN
    CREATE TABLE tunnel.AuditLog (
        -- audit_log_id: Identificador técnico único del evento de auditoría.
        audit_log_id UNIQUEIDENTIFIER NOT NULL CONSTRAINT PK_AuditLog PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
        -- user_id: Usuario que realiza la acción, si aplica.
        user_id UNIQUEIDENTIFIER NULL,
        -- incident_event_id: Incidente relacionado con la acción auditada, si aplica.
        incident_event_id UNIQUEIDENTIFIER NULL,
        -- entity_type: Tipo de entidad afectada, por ejemplo ProtocolRun, Asset o IncidentEvent.
        entity_type NVARCHAR(60) NOT NULL,
        -- entity_id: Identificador de la entidad afectada.
        entity_id UNIQUEIDENTIFIER NOT NULL,
        -- action: Acción realizada: CREATE, UPDATE, EXECUTE, OVERRIDE, etc.
        [action] NVARCHAR(40) NOT NULL,
        -- timestamp_utc: Fecha/hora UTC del evento auditado.
        [timestamp_utc] DATETIME2(3) NOT NULL CONSTRAINT DF_AuditLog_ts DEFAULT SYSUTCDATETIME(),
        -- details_json: Detalles de auditoría en JSON.
        details_json NVARCHAR(MAX) NULL,
        CONSTRAINT FK_AuditLog_User FOREIGN KEY (user_id) REFERENCES tunnel.UserAccount(user_id),
        CONSTRAINT FK_AuditLog_Incident FOREIGN KEY (incident_event_id) REFERENCES tunnel.IncidentEvent(incident_event_id),
        CONSTRAINT CK_AuditLog_details_json CHECK (details_json IS NULL OR ISJSON(details_json) = 1)
    );
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_AuditLog_incident_ts' AND object_id = OBJECT_ID(N'tunnel.AuditLog'))
BEGIN
    CREATE INDEX IX_AuditLog_incident_ts ON tunnel.AuditLog(incident_event_id, [timestamp_utc]);
END;

IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = N'IX_AuditLog_entity' AND object_id = OBJECT_ID(N'tunnel.AuditLog'))
BEGIN
    CREATE INDEX IX_AuditLog_entity ON tunnel.AuditLog(entity_type, entity_id);
END;



-------------------------------------------------------------------------------
-- FK COMPUESTA POSTERIOR
-- Asegura que current_step_id de ProtocolRun pertenece al mismo protocol_id.
-------------------------------------------------------------------------------
IF NOT EXISTS (
    SELECT 1
    FROM sys.foreign_keys
    WHERE name = N'FK_ProtocolRun_CurrentStep'
      AND parent_object_id = OBJECT_ID(N'tunnel.ProtocolRun')
)
BEGIN
    ALTER TABLE tunnel.ProtocolRun WITH CHECK
    ADD CONSTRAINT FK_ProtocolRun_CurrentStep
    FOREIGN KEY (current_step_id, protocol_id)
    REFERENCES tunnel.ProtocolStep(protocol_step_id, protocol_id);
END;



-------------------------------------------------------------------------------
-- 6) PROCEDIMIENTO AUXILIAR PARA DOCUMENTAR TABLAS Y COLUMNAS
--    Usa extended properties estándar de SQL Server (MS_Description).
--    Se pueden consultar desde las vistas:
--      tunnel.v_TableDocumentation
--      tunnel.v_ColumnDocumentation
-------------------------------------------------------------------------------
IF OBJECT_ID(N'tunnel.usp_SetDescription', N'P') IS NULL
BEGIN
    EXEC(N'CREATE PROCEDURE tunnel.usp_SetDescription AS RETURN 0;');
END;

EXEC(N'
ALTER PROCEDURE tunnel.usp_SetDescription
    @SchemaName SYSNAME = N''tunnel'',
    @ObjectName SYSNAME,
    @ColumnName SYSNAME = NULL,
    @Description NVARCHAR(4000)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @ObjectId INT = OBJECT_ID(QUOTENAME(@SchemaName) + N''.'' + QUOTENAME(@ObjectName), N''U'');

    IF @ObjectId IS NULL
        RETURN;

    IF @ColumnName IS NULL
    BEGIN
        IF EXISTS (
            SELECT 1
            FROM sys.extended_properties
            WHERE class = 1
              AND major_id = @ObjectId
              AND minor_id = 0
              AND name = N''MS_Description''
        )
        BEGIN
            EXEC sys.sp_updateextendedproperty
                @name = N''MS_Description'',
                @value = @Description,
                @level0type = N''SCHEMA'', @level0name = @SchemaName,
                @level1type = N''TABLE'',  @level1name = @ObjectName;
        END
        ELSE
        BEGIN
            EXEC sys.sp_addextendedproperty
                @name = N''MS_Description'',
                @value = @Description,
                @level0type = N''SCHEMA'', @level0name = @SchemaName,
                @level1type = N''TABLE'',  @level1name = @ObjectName;
        END
    END
    ELSE
    BEGIN
        DECLARE @ColumnId INT = (
            SELECT column_id
            FROM sys.columns
            WHERE object_id = @ObjectId
              AND name = @ColumnName
        );

        IF @ColumnId IS NULL
            RETURN;

        IF EXISTS (
            SELECT 1
            FROM sys.extended_properties
            WHERE class = 1
              AND major_id = @ObjectId
              AND minor_id = @ColumnId
              AND name = N''MS_Description''
        )
        BEGIN
            EXEC sys.sp_updateextendedproperty
                @name = N''MS_Description'',
                @value = @Description,
                @level0type = N''SCHEMA'', @level0name = @SchemaName,
                @level1type = N''TABLE'',  @level1name = @ObjectName,
                @level2type = N''COLUMN'', @level2name = @ColumnName;
        END
        ELSE
        BEGIN
            EXEC sys.sp_addextendedproperty
                @name = N''MS_Description'',
                @value = @Description,
                @level0type = N''SCHEMA'', @level0name = @SchemaName,
                @level1type = N''TABLE'',  @level1name = @ObjectName,
                @level2type = N''COLUMN'', @level2name = @ColumnName;
        END
    END
END;
');


-------------------------------------------------------------------------------
-- 7) CARGA DE DESCRIPCIONES INTERNAS (MS_Description)
-------------------------------------------------------------------------------
EXEC tunnel.usp_SetDescription @ObjectName=N'Organization', @Description=N'Organización propietaria, gestora o concesionaria responsable de una o varias instalaciones o redes de túneles.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Organization', @ColumnName=N'organization_id', @Description=N'Identificador técnico único de la organización. Se usa como clave primaria y no debe cambiar.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Organization', @ColumnName=N'name', @Description=N'Nombre oficial o operativo de la organización.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Organization', @ColumnName=N'legal_id', @Description=N'Identificador legal/fiscal si aplica, por ejemplo CIF, NIF o identificador administrativo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Organization', @ColumnName=N'country_code', @Description=N'Código de país ISO-3166 alfa-3, por ejemplo ESP.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Organization', @ColumnName=N'timezone_default', @Description=N'Zona horaria por defecto de la organización o ámbito principal.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Organization', @ColumnName=N'created_at_utc', @Description=N'Fecha y hora UTC de creación del registro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Organization', @ColumnName=N'meta_json', @Description=N'Campo JSON extensible para datos adicionales no normalizados.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @Description=N'Ámbito operativo gestionado por una organización: red urbana, concesión, centro de control, autopista o instalación equivalente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @ColumnName=N'facility_id', @Description=N'Identificador técnico único del ámbito o instalación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @ColumnName=N'organization_id', @Description=N'Organización propietaria o gestora de la instalación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @ColumnName=N'name', @Description=N'Nombre operativo de la instalación, red o centro de control.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @ColumnName=N'facility_type', @Description=N'Tipo de instalación: CITY_NETWORK, HIGHWAY_CONCESSION, SINGLE_TUNNEL o CONTROL_CENTER_SCOPE.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @ColumnName=N'address', @Description=N'Dirección física o descripción de ubicación si procede.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @ColumnName=N'timezone', @Description=N'Zona horaria propia del ámbito operativo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @ColumnName=N'contact_phone', @Description=N'Teléfono general de contacto.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Facility', @ColumnName=N'created_at_utc', @Description=N'Fecha y hora UTC de creación del registro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @Description=N'Túnel físico individual. Representa la infraestructura principal y permite vincular tubos, zonas, localizaciones, planes e incidencias.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'tunnel_id', @Description=N'Identificador técnico único del túnel.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'facility_id', @Description=N'Ámbito operativo al que pertenece el túnel.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'name', @Description=N'Nombre oficial u operativo del túnel.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'local_code', @Description=N'Código interno/local del túnel.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'road_name', @Description=N'Carretera, ronda, vía o eje viario asociado.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'country_code', @Description=N'Código de país ISO-3166 alfa-3.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'city', @Description=N'Ciudad o área territorial.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'length_m', @Description=N'Longitud aproximada del túnel en metros.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'has_bidirectional_tubes', @Description=N'Indica si el túnel tiene tubos o sentidos bidireccionales.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'commissioning_date', @Description=N'Fecha de puesta en servicio.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'tunnel_status', @Description=N'Estado operativo del túnel: ACTIVE, WORKS o DECOMMISSIONED.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'meta_json', @Description=N'Datos adicionales como normativa, restricciones, notas geométricas o configuración local.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tunnel', @ColumnName=N'created_at_utc', @Description=N'Fecha y hora UTC de creación del registro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @Description=N'Tubo, sentido o calzada interna de un túnel. Permite modelar túneles de uno o varios tubos y sentidos de circulación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @ColumnName=N'tube_id', @Description=N'Identificador técnico único del tubo o sentido.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @ColumnName=N'tunnel_id', @Description=N'Túnel al que pertenece el tubo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @ColumnName=N'name', @Description=N'Nombre del tubo o sentido, por ejemplo Besòs, Llobregat, Ascendente o Descendente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @ColumnName=N'direction', @Description=N'Dirección normalizada: N, S, E, W, A_TO_B, B_TO_A o BIDIR.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @ColumnName=N'lanes_count', @Description=N'Número de carriles del tubo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @ColumnName=N'speed_limit_kmh', @Description=N'Velocidad máxima autorizada en km/h.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @ColumnName=N'gradient_percent', @Description=N'Pendiente media o relevante expresada en porcentaje.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @ColumnName=N'cross_section_type', @Description=N'Tipo de sección o configuración geométrica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Tube', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Zone', @Description=N'Sectorización interna de un tubo: zona operativa, compartimento de incendio, zona de evacuación, zona vulnerable o tramo de riesgo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Zone', @ColumnName=N'zone_id', @Description=N'Identificador técnico único de la zona.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Zone', @ColumnName=N'tube_id', @Description=N'Tubo al que pertenece la zona.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Zone', @ColumnName=N'name', @Description=N'Nombre de la zona.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Zone', @ColumnName=N'zone_type', @Description=N'Tipo de zona: OPERATIONAL, FIRE_COMPARTMENT, EVACUATION, RISK o VULNERABLE.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Zone', @ColumnName=N'start_chainage_m', @Description=N'Inicio de la zona en metros de progresiva o referencia lineal.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Zone', @ColumnName=N'end_chainage_m', @Description=N'Fin de la zona en metros de progresiva o referencia lineal.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Zone', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @Description=N'Localización operativa dentro de un túnel: boca, tramo, punto kilométrico, sala técnica, salida de emergencia, poste SOS, cámara u otro punto relevante.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @ColumnName=N'location_id', @Description=N'Identificador técnico único de la localización.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @ColumnName=N'tunnel_id', @Description=N'Túnel al que pertenece la localización.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @ColumnName=N'zone_id', @Description=N'Zona a la que pertenece la localización si aplica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @ColumnName=N'location_type', @Description=N'Tipo: PORTAL, SEGMENT, LANE_POINT, TECH_ROOM, CROSS_PASSAGE, EMERGENCY_EXIT, SOS_POST, CAMERA_POLE u OTHER.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @ColumnName=N'name', @Description=N'Nombre o etiqueta de la localización.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @ColumnName=N'chainage_m', @Description=N'Progresiva o referencia lineal en metros.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @ColumnName=N'geom', @Description=N'Coordenada geográfica opcional. Normalmente SRID 4326.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @ColumnName=N'access_description', @Description=N'Descripción de acceso para operadores o ayuda externa.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Location', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetType', @Description=N'Catálogo de tipos de equipamiento independientes de fabricante: CCTV, DAI, SCADA, ventilación, iluminación, PMV, semáforos, SOS, sensores, etc.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetType', @ColumnName=N'asset_type_id', @Description=N'Identificador técnico único del tipo de activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetType', @ColumnName=N'category', @Description=N'Categoría funcional normalizada del activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetType', @ColumnName=N'name', @Description=N'Nombre concreto del tipo de activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetType', @ColumnName=N'vendor_independent', @Description=N'Indica si el tipo es independiente de fabricante.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetType', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ControlSystem', @Description=N'Sistema de control o supervisión que gobierna o monitoriza activos: SCADA, CCTV/VMS, DAI, ATMS, BMS u otros.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ControlSystem', @ColumnName=N'control_system_id', @Description=N'Identificador técnico único del sistema de control.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ControlSystem', @ColumnName=N'facility_id', @Description=N'Ámbito operativo al que pertenece el sistema.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ControlSystem', @ColumnName=N'name', @Description=N'Nombre operativo del sistema.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ControlSystem', @ColumnName=N'system_type', @Description=N'Tipo de sistema: SCADA, VMS_CCTV, DAI, ATMS, BMS u OTHER.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ControlSystem', @ColumnName=N'primary_site', @Description=N'Ubicación principal del sistema si aplica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ControlSystem', @ColumnName=N'has_backup', @Description=N'Indica si dispone de sistema o sala de respaldo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ControlSystem', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @Description=N'Activo físico o lógico instalado o supervisado: cámara, sensor, ventilador, luminaria, PMV, semáforo, sistema SOS, etc.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'asset_id', @Description=N'Identificador técnico único del activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'asset_type_id', @Description=N'Tipo de activo al que pertenece.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'control_system_id', @Description=N'Sistema de control que supervisa o controla el activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'asset_tag', @Description=N'Etiqueta de inventario única del activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'manufacturer', @Description=N'Fabricante del activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'model', @Description=N'Modelo del activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'serial_number', @Description=N'Número de serie.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'criticality', @Description=N'Criticidad operativa: LOW, MEDIUM, HIGH o SAFETY_CRITICAL.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'asset_status', @Description=N'Estado del activo: OK, DEGRADED, FAILED o MAINTENANCE.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'last_healthcheck_at_utc', @Description=N'Última fecha/hora UTC de comprobación, telemetría o heartbeat.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Asset', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @Description=N'Instalación de un activo en una localización concreta, con cobertura, orientación y relación con tubo si aplica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'asset_installation_id', @Description=N'Identificador técnico único de la instalación del activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'asset_id', @Description=N'Activo instalado.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'location_id', @Description=N'Localización donde está instalado el activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'tube_id', @Description=N'Tubo asociado a la instalación si aplica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'installed_at', @Description=N'Fecha de instalación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'coverage_start_chainage_m', @Description=N'Inicio de cobertura del activo en metros.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'coverage_end_chainage_m', @Description=N'Fin de cobertura del activo en metros.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'orientation', @Description=N'Orientación física o lógica del activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'is_primary', @Description=N'Indica si es la instalación principal del activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AssetInstallation', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Plan', @Description=N'Plan documental u operativo: PAU, Plan de Emergencia o conjunto de protocolos de explotación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Plan', @ColumnName=N'plan_id', @Description=N'Identificador técnico único del plan.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Plan', @ColumnName=N'facility_id', @Description=N'Ámbito operativo al que pertenece el plan.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Plan', @ColumnName=N'name', @Description=N'Nombre del plan.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Plan', @ColumnName=N'plan_type', @Description=N'Tipo de plan: PAU, EMERGENCY_PLAN u OPERATIONS_PROTOCOLS.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Plan', @ColumnName=N'authority', @Description=N'Autoridad, organismo o área responsable del plan.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Plan', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Plan', @ColumnName=N'created_at_utc', @Description=N'Fecha y hora UTC de creación del registro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @Description=N'Versión concreta de un plan, con vigencia, aprobación y estado. Permite gestionar revisiones y trazabilidad normativa.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'plan_version_id', @Description=N'Identificador técnico único de la versión del plan.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'plan_id', @Description=N'Plan al que pertenece la versión.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'version_label', @Description=N'Etiqueta de versión, por ejemplo v1.0, v1.1 o 2025.03.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'effective_from', @Description=N'Fecha de inicio de vigencia.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'effective_to', @Description=N'Fecha de fin de vigencia.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'version_status', @Description=N'Estado de versión: DRAFT, ACTIVE o RETIRED.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'approved_by', @Description=N'Persona, área u organismo que aprueba la versión.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'approval_date', @Description=N'Fecha de aprobación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanVersion', @ColumnName=N'created_at_utc', @Description=N'Fecha y hora UTC de creación del registro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanTunnelScope', @Description=N'Relación entre una versión de plan y los túneles cubiertos por ella. Permite que un plan cubra varios túneles y viceversa.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanTunnelScope', @ColumnName=N'plan_tunnel_scope_id', @Description=N'Identificador técnico único de la relación de alcance.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanTunnelScope', @ColumnName=N'plan_version_id', @Description=N'Versión de plan aplicable.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanTunnelScope', @ColumnName=N'tunnel_id', @Description=N'Túnel cubierto por la versión del plan.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanTunnelScope', @ColumnName=N'scope_note', @Description=N'Nota de alcance, por ejemplo fase de obras, tramo parcial o condición especial.';
EXEC tunnel.usp_SetDescription @ObjectName=N'PlanTunnelScope', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'CodeScheme', @Description=N'Esquema de codificación de incidentes. Permite soportar códigos locales como 100-TRA o esquemas de otros países/operadores.';
EXEC tunnel.usp_SetDescription @ObjectName=N'CodeScheme', @ColumnName=N'code_scheme_id', @Description=N'Identificador técnico único del esquema de códigos.';
EXEC tunnel.usp_SetDescription @ObjectName=N'CodeScheme', @ColumnName=N'name', @Description=N'Nombre del esquema de codificación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'CodeScheme', @ColumnName=N'description', @Description=N'Descripción funcional del esquema.';
EXEC tunnel.usp_SetDescription @ObjectName=N'CodeScheme', @ColumnName=N'pattern_hint', @Description=N'Pista de patrón o formato, por ejemplo una regex o estructura esperada.';
EXEC tunnel.usp_SetDescription @ObjectName=N'CodeScheme', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'EmergencyLevel', @Description=N'Nivel de gravedad o activación: prealerta, alerta, emergencia u otros niveles configurables.';
EXEC tunnel.usp_SetDescription @ObjectName=N'EmergencyLevel', @ColumnName=N'emergency_level_id', @Description=N'Identificador técnico único del nivel de emergencia.';
EXEC tunnel.usp_SetDescription @ObjectName=N'EmergencyLevel', @ColumnName=N'name', @Description=N'Nombre del nivel.';
EXEC tunnel.usp_SetDescription @ObjectName=N'EmergencyLevel', @ColumnName=N'rank', @Description=N'Orden de gravedad o prioridad. A mayor valor, mayor nivel si así se configura.';
EXEC tunnel.usp_SetDescription @ObjectName=N'EmergencyLevel', @ColumnName=N'description', @Description=N'Descripción operativa del nivel.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentFamily', @Description=N'Familia funcional de incidentes: tráfico, avería, incendio, ambiental, iluminación u otras.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentFamily', @ColumnName=N'incident_family_id', @Description=N'Identificador técnico único de la familia.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentFamily', @ColumnName=N'code', @Description=N'Código corto de familia, por ejemplo TRA, AVA, FOC, AMB o ILI.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentFamily', @ColumnName=N'name', @Description=N'Nombre de la familia.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentFamily', @ColumnName=N'description', @Description=N'Descripción de los incidentes incluidos en la familia.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @Description=N'Tipo de incidente catalogado. Es la clasificación que permite seleccionar el protocolo adecuado.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'incident_type_id', @Description=N'Identificador técnico único del tipo de incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'code_scheme_id', @Description=N'Esquema de codificación al que pertenece el código.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'emergency_level_id', @Description=N'Nivel de emergencia asociado al tipo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'incident_family_id', @Description=N'Familia funcional del incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'code_raw', @Description=N'Código textual del incidente, por ejemplo 260-AVA.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'title', @Description=N'Título operativo del incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'description', @Description=N'Descripción completa del tipo de incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'detection_notes', @Description=N'Notas sobre cómo detectar, verificar o confirmar el incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'operational_context', @Description=N'Contexto: NORMAL, WORKS, EVENT u OTHER.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'info_to_collect_json', @Description=N'Campos o checklist que el operador debe recopilar, en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentType', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Protocol', @Description=N'Plantilla de actuación asociada a una versión de plan y a un tipo de incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Protocol', @ColumnName=N'protocol_id', @Description=N'Identificador técnico único del protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Protocol', @ColumnName=N'plan_version_id', @Description=N'Versión del plan que define el protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Protocol', @ColumnName=N'incident_type_id', @Description=N'Tipo de incidente gestionado por el protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Protocol', @ColumnName=N'name', @Description=N'Nombre operativo del protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Protocol', @ColumnName=N'objective', @Description=N'Objetivo resumido del protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Protocol', @ColumnName=N'is_remote_executable', @Description=N'Indica si puede ejecutarse desde centro de control o consola remota.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Protocol', @ColumnName=N'protocol_status', @Description=N'Estado del protocolo: ACTIVE, RETIRED o DRAFT.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Protocol', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @Description=N'Nodo del flujo de trabajo de un protocolo: decisión, acción, espera, información o checklist.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'protocol_step_id', @Description=N'Identificador técnico único del paso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'protocol_id', @Description=N'Protocolo al que pertenece el paso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'step_key', @Description=N'Clave estable del paso para referenciarlo desde configuración o interfaz.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'step_type', @Description=N'Tipo de paso: DECISION, ACTION, INFO, WAIT o CHECKLIST.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'title', @Description=N'Título visible del paso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'instructions', @Description=N'Instrucciones operativas que debe seguir el operador o el sistema.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'requires_ack', @Description=N'Indica si el operador debe confirmar el paso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'timeout_seconds', @Description=N'Tiempo máximo recomendado antes de escalar o disparar otra transición.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'ui_form_schema_json', @Description=N'Esquema JSON para pintar formularios dinámicos en la interfaz.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolStep', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepTransition', @Description=N'Transición dirigida entre pasos de un protocolo. Permite modelar decisiones condicionales, ramas y rutas alternativas.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepTransition', @ColumnName=N'step_transition_id', @Description=N'Identificador técnico único de la transición.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepTransition', @ColumnName=N'protocol_id', @Description=N'Protocolo al que pertenece la transición.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepTransition', @ColumnName=N'from_step_id', @Description=N'Paso origen.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepTransition', @ColumnName=N'to_step_id', @Description=N'Paso destino.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepTransition', @ColumnName=N'condition_expr', @Description=N'Expresión de condición o regla de negocio. Si es NULL puede interpretarse como transición por defecto.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepTransition', @ColumnName=N'priority', @Description=N'Prioridad de evaluación cuando hay varias transiciones posibles.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepTransition', @ColumnName=N'label', @Description=N'Texto visible para la transición, por ejemplo Sí, No, Confirmado o Escalar.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepTransition', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionDefinition', @Description=N'Catálogo de acciones atómicas ejecutables o registrables: señalización, semáforos, cierre, ventilación, iluminación, megafonía, orden de trabajo, aviso externo, etc.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionDefinition', @ColumnName=N'action_definition_id', @Description=N'Identificador técnico único de la acción definida.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionDefinition', @ColumnName=N'action_type', @Description=N'Tipo de acción: SET_SIGNAGE, SET_SEMAPHORE, CLOSE_TUBE, VENTILATION_MODE, LIGHTING_MODE, PA_ANNOUNCEMENT, CREATE_WORK_ORDER, REQUEST_EXTERNAL, LOG_ONLY u OTHER.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionDefinition', @ColumnName=N'name', @Description=N'Nombre operativo de la acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionDefinition', @ColumnName=N'description', @Description=N'Descripción de qué hace la acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionDefinition', @ColumnName=N'payload_schema_json', @Description=N'Esquema JSON de los parámetros necesarios para ejecutar la acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionDefinition', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepAction', @Description=N'Relación entre un paso de protocolo y una acción definida. Permite ejecutar varias acciones ordenadas por cada paso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepAction', @ColumnName=N'step_action_id', @Description=N'Identificador técnico único de la relación paso-acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepAction', @ColumnName=N'protocol_step_id', @Description=N'Paso de protocolo que dispara la acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepAction', @ColumnName=N'action_definition_id', @Description=N'Acción definida que se debe ejecutar o registrar.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepAction', @ColumnName=N'execution_order', @Description=N'Orden de ejecución dentro del paso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepAction', @ColumnName=N'is_mandatory', @Description=N'Indica si la acción es obligatoria en el paso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepAction', @ColumnName=N'parameter_binding_json', @Description=N'Mapeo JSON entre parámetros del protocolo/incidente y payload de acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepAction', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'NotificationRule', @Description=N'Regla o plantilla de aviso a organismos, centros de control, mantenimiento u otros contactos.';
EXEC tunnel.usp_SetDescription @ObjectName=N'NotificationRule', @ColumnName=N'notification_rule_id', @Description=N'Identificador técnico único de la regla de notificación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'NotificationRule', @ColumnName=N'name', @Description=N'Nombre de la regla.';
EXEC tunnel.usp_SetDescription @ObjectName=N'NotificationRule', @ColumnName=N'purpose', @Description=N'Finalidad del aviso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'NotificationRule', @ColumnName=N'default_channel', @Description=N'Canal por defecto: PHONE, EMAIL, RADIO, API, SMS u OTHER.';
EXEC tunnel.usp_SetDescription @ObjectName=N'NotificationRule', @ColumnName=N'message_template', @Description=N'Plantilla del mensaje con variables sustituibles por la aplicación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'NotificationRule', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepNotification', @Description=N'Relación entre un paso de protocolo y una regla de notificación. Define cuándo y bajo qué condición se envía un aviso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepNotification', @ColumnName=N'step_notification_id', @Description=N'Identificador técnico único de la relación paso-notificación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepNotification', @ColumnName=N'protocol_step_id', @Description=N'Paso de protocolo que dispara la notificación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepNotification', @ColumnName=N'notification_rule_id', @Description=N'Regla de notificación a aplicar.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepNotification', @ColumnName=N'when', @Description=N'Momento de disparo: ON_ENTER, ON_EXIT, ON_TIMEOUT u ON_CONDITION.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepNotification', @ColumnName=N'condition_expr', @Description=N'Condición adicional para disparar el aviso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'StepNotification', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterDefinition', @Description=N'Definición global de un parámetro configurable: umbrales, límites, tiempos, modos o reglas reutilizables.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterDefinition', @ColumnName=N'parameter_definition_id', @Description=N'Identificador técnico único de la definición de parámetro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterDefinition', @ColumnName=N'key', @Description=N'Clave única del parámetro, estable para uso en código y configuración.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterDefinition', @ColumnName=N'data_type', @Description=N'Tipo de dato esperado: INT, DECIMAL, BOOLEAN, TEXT o JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterDefinition', @ColumnName=N'unit', @Description=N'Unidad del parámetro si aplica, por ejemplo segundos, ppm, km/h o porcentaje.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterDefinition', @ColumnName=N'description', @Description=N'Descripción funcional del parámetro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterDefinition', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterSet', @Description=N'Colección de valores de parámetros aplicable a un túnel o a una versión de plan. Sirve para overrides y personalización.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterSet', @ColumnName=N'parameter_set_id', @Description=N'Identificador técnico único del conjunto de parámetros.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterSet', @ColumnName=N'tunnel_id', @Description=N'Túnel al que aplica el conjunto, si es un set por túnel.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterSet', @ColumnName=N'plan_version_id', @Description=N'Versión de plan a la que aplica el conjunto, si es un set por plan.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterSet', @ColumnName=N'name', @Description=N'Nombre del conjunto de parámetros.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterSet', @ColumnName=N'priority', @Description=N'Prioridad para resolver overrides cuando hay varios conjuntos aplicables.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterSet', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterSet', @ColumnName=N'created_at_utc', @Description=N'Fecha y hora UTC de creación del registro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @Description=N'Valor concreto de un parámetro dentro de un conjunto. Usa columnas tipadas para mantener compatibilidad y facilitar validación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @ColumnName=N'parameter_value_id', @Description=N'Identificador técnico único del valor de parámetro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @ColumnName=N'parameter_set_id', @Description=N'Conjunto de parámetros al que pertenece el valor.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @ColumnName=N'parameter_definition_id', @Description=N'Definición de parámetro que se está valorando.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @ColumnName=N'value_int', @Description=N'Valor entero si el parámetro es de tipo INT.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @ColumnName=N'value_decimal', @Description=N'Valor decimal si el parámetro es de tipo DECIMAL.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @ColumnName=N'value_bool', @Description=N'Valor booleano si el parámetro es de tipo BOOLEAN.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @ColumnName=N'value_text', @Description=N'Valor textual si el parámetro es de tipo TEXT.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @ColumnName=N'value_json', @Description=N'Valor JSON si el parámetro es de tipo JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ParameterValue', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolParameter', @Description=N'Relación documental entre protocolo y parámetros que utiliza. Ayuda a validar configuración antes de ejecutar protocolos.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolParameter', @ColumnName=N'protocol_parameter_id', @Description=N'Identificador técnico único de la relación protocolo-parámetro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolParameter', @ColumnName=N'protocol_id', @Description=N'Protocolo que utiliza el parámetro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolParameter', @ColumnName=N'parameter_definition_id', @Description=N'Parámetro requerido o utilizado por el protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolParameter', @ColumnName=N'usage_note', @Description=N'Nota que explica cómo usa el protocolo este parámetro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Role', @Description=N'Rol operativo o administrativo de un usuario: operador, jefe de turno, mantenimiento, supervisor, etc.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Role', @ColumnName=N'role_id', @Description=N'Identificador técnico único del rol.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Role', @ColumnName=N'name', @Description=N'Nombre único del rol.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Role', @ColumnName=N'description', @Description=N'Descripción de permisos o responsabilidades del rol.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Role', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @Description=N'Usuario de la aplicación o consola operativa. La autenticación puede estar en la aplicación, pero aquí queda la identidad operativa.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'user_id', @Description=N'Identificador técnico único del usuario.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'facility_id', @Description=N'Ámbito operativo al que pertenece el usuario.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'role_id', @Description=N'Rol asignado al usuario.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'username', @Description=N'Nombre de usuario o login.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'display_name', @Description=N'Nombre visible del usuario.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'phone', @Description=N'Teléfono de contacto.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'email', @Description=N'Correo electrónico.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'is_active', @Description=N'Indica si el usuario está activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'UserAccount', @ColumnName=N'created_at_utc', @Description=N'Fecha y hora UTC de creación del registro.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Agency', @Description=N'Organismo o entidad interna/externa que puede ser avisada o participar en la respuesta: emergencias, tráfico, mantenimiento, centro de control, etc.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Agency', @ColumnName=N'agency_id', @Description=N'Identificador técnico único del organismo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Agency', @ColumnName=N'name', @Description=N'Nombre del organismo o entidad.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Agency', @ColumnName=N'agency_type', @Description=N'Tipo: EMERGENCY_SERVICES, TRAFFIC_AUTHORITY, CONTROL_CENTER, MAINTENANCE u OTHER.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Agency', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ContactPoint', @Description=N'Punto de contacto de un organismo: teléfono, radio, email, SMS, endpoint API u otro canal.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ContactPoint', @ColumnName=N'contact_point_id', @Description=N'Identificador técnico único del punto de contacto.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ContactPoint', @ColumnName=N'agency_id', @Description=N'Organismo al que pertenece el contacto.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ContactPoint', @ColumnName=N'name', @Description=N'Nombre o etiqueta del contacto.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ContactPoint', @ColumnName=N'channel', @Description=N'Canal: PHONE, EMAIL, RADIO, API, SMS u OTHER.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ContactPoint', @ColumnName=N'address', @Description=N'Valor del contacto: teléfono, email, endpoint, canal de radio, etc.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ContactPoint', @ColumnName=N'availability', @Description=N'Disponibilidad horaria o condición de uso.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ContactPoint', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @Description=N'Incidente real registrado en explotación. Une túnel, tipo de incidente, estado, tiempos, notas y datos operativos.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'incident_event_id', @Description=N'Identificador técnico único del incidente real.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'incident_type_id', @Description=N'Tipo de incidente catalogado.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'tunnel_id', @Description=N'Túnel donde ocurre el incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'incident_status', @Description=N'Estado del incidente: OPEN, MITIGATING, RESOLVED o CLOSED.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'started_at_utc', @Description=N'Fecha/hora UTC de inicio o apertura del incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'detected_at_utc', @Description=N'Fecha/hora UTC de detección si difiere de la apertura.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'resolved_at_utc', @Description=N'Fecha/hora UTC de resolución.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'severity_override', @Description=N'Reclasificación manual de severidad si el operador la aplica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'summary', @Description=N'Resumen corto del incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'operator_notes', @Description=N'Notas libres del operador.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentEvent', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'DetectionEvent', @Description=N'Evidencia o evento de detección asociado a un incidente: sensor, CCTV/DAI, alarma SCADA, llamada SOS, aviso externo u observación del operador.';
EXEC tunnel.usp_SetDescription @ObjectName=N'DetectionEvent', @ColumnName=N'detection_event_id', @Description=N'Identificador técnico único del evento de detección.';
EXEC tunnel.usp_SetDescription @ObjectName=N'DetectionEvent', @ColumnName=N'incident_event_id', @Description=N'Incidente al que pertenece la evidencia.';
EXEC tunnel.usp_SetDescription @ObjectName=N'DetectionEvent', @ColumnName=N'source_type', @Description=N'Origen: SENSOR, CCTV_DAI, SCADA_ALARM, SOS_CALL, EXTERNAL_CALL u OPERATOR_OBS.';
EXEC tunnel.usp_SetDescription @ObjectName=N'DetectionEvent', @ColumnName=N'source_asset_id', @Description=N'Activo que generó la detección, si aplica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'DetectionEvent', @ColumnName=N'reported_by', @Description=N'Persona, servicio, centro o sistema que reporta la detección.';
EXEC tunnel.usp_SetDescription @ObjectName=N'DetectionEvent', @ColumnName=N'payload_json', @Description=N'Datos de detección en JSON: medidas, alarmas, valores, texto, etc.';
EXEC tunnel.usp_SetDescription @ObjectName=N'DetectionEvent', @ColumnName=N'created_at_utc', @Description=N'Fecha/hora UTC de registro de la detección.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentLocation', @Description=N'Relación N:M entre incidente y localización. Permite indicar varias ubicaciones, ubicación principal y grado de confianza.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentLocation', @ColumnName=N'incident_location_id', @Description=N'Identificador técnico único de la relación incidente-localización.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentLocation', @ColumnName=N'incident_event_id', @Description=N'Incidente localizado.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentLocation', @ColumnName=N'location_id', @Description=N'Localización asociada al incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentLocation', @ColumnName=N'confidence', @Description=N'Confianza de la localización entre 0 y 1.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentLocation', @ColumnName=N'is_primary', @Description=N'Indica si es la localización principal del incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'IncidentLocation', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @Description=N'Ejecución concreta de un protocolo para un incidente real. Guarda estado, usuario iniciador, paso actual y snapshot de contexto.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'protocol_run_id', @Description=N'Identificador técnico único de la ejecución del protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'incident_event_id', @Description=N'Incidente gestionado por esta ejecución.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'protocol_id', @Description=N'Protocolo que se está ejecutando.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'started_by_user_id', @Description=N'Usuario que inició la ejecución.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'started_at_utc', @Description=N'Fecha/hora UTC de inicio de la ejecución.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'ended_at_utc', @Description=N'Fecha/hora UTC de finalización.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'run_status', @Description=N'Estado de ejecución: RUNNING, PAUSED, COMPLETED o ABORTED.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'current_step_id', @Description=N'Paso actual dentro del protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'context_snapshot_json', @Description=N'Snapshot JSON de contexto: parámetros resueltos, estado de activos, datos del incidente, etc.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ProtocolRun', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @Description=N'Acción concreta solicitada, enviada, ejecutada o fallida durante una ejecución de protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'action_execution_id', @Description=N'Identificador técnico único de la ejecución de acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'protocol_run_id', @Description=N'Ejecución de protocolo a la que pertenece.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'action_definition_id', @Description=N'Acción definida que se ejecuta.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'step_id', @Description=N'Paso que disparó la acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'requested_at_utc', @Description=N'Fecha/hora UTC en que se solicitó la acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'executed_at_utc', @Description=N'Fecha/hora UTC en que se ejecutó o se confirmó.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'exec_status', @Description=N'Estado: REQUESTED, SENT, SUCCESS, FAILED o CANCELLED.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'payload_json', @Description=N'Payload JSON real enviado o registrado para ejecutar la acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'result_json', @Description=N'Resultado JSON devuelto por el sistema o integración.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionExecution', @ColumnName=N'error_message', @Description=N'Mensaje de error si la acción falla.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionTarget', @Description=N'Equipos o activos afectados por una acción concreta. Permite una acción sobre múltiples objetivos.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionTarget', @ColumnName=N'action_target_id', @Description=N'Identificador técnico único del objetivo de acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionTarget', @ColumnName=N'action_execution_id', @Description=N'Ejecución de acción que afecta al activo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionTarget', @ColumnName=N'asset_id', @Description=N'Activo objetivo de la acción.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionTarget', @ColumnName=N'target_role', @Description=N'Papel del activo en la acción, por ejemplo PMV entrada o ventilador sector 2.';
EXEC tunnel.usp_SetDescription @ObjectName=N'ActionTarget', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @Description=N'Notificación real enviada o pendiente dentro de una ejecución de protocolo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @ColumnName=N'notification_id', @Description=N'Identificador técnico único de la notificación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @ColumnName=N'protocol_run_id', @Description=N'Ejecución de protocolo que origina la notificación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @ColumnName=N'contact_point_id', @Description=N'Punto de contacto destinatario.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @ColumnName=N'notification_rule_id', @Description=N'Regla de notificación aplicada, si procede.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @ColumnName=N'sent_at_utc', @Description=N'Fecha/hora UTC de envío.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @ColumnName=N'notif_status', @Description=N'Estado: PENDING, SENT, FAILED o ACKED.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @ColumnName=N'message', @Description=N'Mensaje final enviado o preparado.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @ColumnName=N'ack_at_utc', @Description=N'Fecha/hora UTC de acuse o confirmación.';
EXEC tunnel.usp_SetDescription @ObjectName=N'Notification', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @Description=N'Orden de trabajo o mantenimiento asociada a un activo y opcionalmente generada desde un incidente.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @ColumnName=N'work_order_id', @Description=N'Identificador técnico único de la orden de trabajo.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @ColumnName=N'asset_id', @Description=N'Activo afectado por la orden.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @ColumnName=N'created_from_incident_id', @Description=N'Incidente que originó la orden si aplica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @ColumnName=N'priority', @Description=N'Prioridad: LOW, MEDIUM, HIGH o URGENT.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @ColumnName=N'work_status', @Description=N'Estado: OPEN, IN_PROGRESS, DONE o CANCELLED.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @ColumnName=N'description', @Description=N'Descripción del trabajo o incidencia técnica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @ColumnName=N'created_at_utc', @Description=N'Fecha/hora UTC de creación de la orden.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @ColumnName=N'closed_at_utc', @Description=N'Fecha/hora UTC de cierre.';
EXEC tunnel.usp_SetDescription @ObjectName=N'WorkOrder', @ColumnName=N'meta_json', @Description=N'Datos adicionales flexibles en JSON.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AuditLog', @Description=N'Registro de auditoría legal/operativa: cambios, ejecuciones, overrides y acciones relevantes del usuario o del sistema.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AuditLog', @ColumnName=N'audit_log_id', @Description=N'Identificador técnico único del evento de auditoría.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AuditLog', @ColumnName=N'user_id', @Description=N'Usuario que realiza la acción, si aplica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AuditLog', @ColumnName=N'incident_event_id', @Description=N'Incidente relacionado con la acción auditada, si aplica.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AuditLog', @ColumnName=N'entity_type', @Description=N'Tipo de entidad afectada, por ejemplo ProtocolRun, Asset o IncidentEvent.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AuditLog', @ColumnName=N'entity_id', @Description=N'Identificador de la entidad afectada.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AuditLog', @ColumnName=N'action', @Description=N'Acción realizada: CREATE, UPDATE, EXECUTE, OVERRIDE, etc.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AuditLog', @ColumnName=N'timestamp_utc', @Description=N'Fecha/hora UTC del evento auditado.';
EXEC tunnel.usp_SetDescription @ObjectName=N'AuditLog', @ColumnName=N'details_json', @Description=N'Detalles de auditoría en JSON.';


-------------------------------------------------------------------------------
-- 8) VISTAS INTERNAS DE DOCUMENTACIÓN
--    Estas vistas permiten consultar la documentación desde la propia BBDD.
-------------------------------------------------------------------------------
EXEC(N'
CREATE OR ALTER VIEW tunnel.v_TableDocumentation
AS
SELECT
    DB_NAME() AS database_name,
    s.name AS schema_name,
    t.name AS table_name,
    CAST(ep.value AS NVARCHAR(4000)) AS table_description
FROM sys.tables t
JOIN sys.schemas s
    ON s.schema_id = t.schema_id
LEFT JOIN sys.extended_properties ep
    ON ep.class = 1
   AND ep.major_id = t.object_id
   AND ep.minor_id = 0
   AND ep.name = N''MS_Description''
WHERE s.name = N''tunnel'';
');

EXEC(N'
CREATE OR ALTER VIEW tunnel.v_ColumnDocumentation
AS
SELECT
    DB_NAME() AS database_name,
    s.name AS schema_name,
    t.name AS table_name,
    c.column_id,
    c.name AS column_name,
    CASE
        WHEN ty.name IN (N''nvarchar'', N''nchar'')
            THEN ty.name + N''('' + CASE WHEN c.max_length = -1 THEN N''MAX'' ELSE CONVERT(NVARCHAR(20), c.max_length / 2) END + N'')''
        WHEN ty.name IN (N''varchar'', N''char'', N''varbinary'', N''binary'')
            THEN ty.name + N''('' + CASE WHEN c.max_length = -1 THEN N''MAX'' ELSE CONVERT(NVARCHAR(20), c.max_length) END + N'')''
        WHEN ty.name IN (N''decimal'', N''numeric'')
            THEN ty.name + N''('' + CONVERT(NVARCHAR(20), c.precision) + N'','' + CONVERT(NVARCHAR(20), c.scale) + N'')''
        WHEN ty.name IN (N''datetime2'', N''datetimeoffset'', N''time'')
            THEN ty.name + N''('' + CONVERT(NVARCHAR(20), c.scale) + N'')''
        ELSE ty.name
    END AS data_type,
    c.is_nullable,
    dc.definition AS default_definition,
    CAST(ep.value AS NVARCHAR(4000)) AS column_description
FROM sys.tables t
JOIN sys.schemas s
    ON s.schema_id = t.schema_id
JOIN sys.columns c
    ON c.object_id = t.object_id
JOIN sys.types ty
    ON ty.user_type_id = c.user_type_id
LEFT JOIN sys.default_constraints dc
    ON dc.parent_object_id = t.object_id
   AND dc.parent_column_id = c.column_id
LEFT JOIN sys.extended_properties ep
    ON ep.class = 1
   AND ep.major_id = t.object_id
   AND ep.minor_id = c.column_id
   AND ep.name = N''MS_Description''
WHERE s.name = N''tunnel'';
');

EXEC(N'
CREATE OR ALTER VIEW tunnel.v_DatabaseDictionary
AS
SELECT
    N''TABLE'' AS item_type,
    schema_name,
    table_name,
    CAST(NULL AS INT) AS column_id,
    CAST(NULL AS SYSNAME) AS column_name,
    CAST(NULL AS NVARCHAR(128)) AS data_type,
    table_description AS description
FROM tunnel.v_TableDocumentation
UNION ALL
SELECT
    N''COLUMN'' AS item_type,
    schema_name,
    table_name,
    column_id,
    column_name,
    data_type,
    column_description AS description
FROM tunnel.v_ColumnDocumentation;
');



-------------------------------------------------------------------------------
-- 9) DATOS BASE MÍNIMOS
--    Son valores genéricos de partida. Puedes ampliarlos desde la aplicación.
-------------------------------------------------------------------------------

IF NOT EXISTS (SELECT 1 FROM tunnel.[Role] WHERE name = N'OPERADOR')
BEGIN
    INSERT INTO tunnel.[Role] (name, description)
    VALUES (N'OPERADOR', N'Usuario operativo de consola o centro de control.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.[Role] WHERE name = N'JEFE_TURNO')
BEGIN
    INSERT INTO tunnel.[Role] (name, description)
    VALUES (N'JEFE_TURNO', N'Responsable operativo del turno.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.[Role] WHERE name = N'MANTENIMIENTO')
BEGIN
    INSERT INTO tunnel.[Role] (name, description)
    VALUES (N'MANTENIMIENTO', N'Personal o equipo responsable de mantenimiento.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.EmergencyLevel WHERE name = N'PREALERTA')
BEGIN
    INSERT INTO tunnel.EmergencyLevel (name, rank, description)
    VALUES (N'PREALERTA', 1, N'Situación inicial, anomalía o incidente de menor gravedad.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.EmergencyLevel WHERE name = N'ALERTA')
BEGIN
    INSERT INTO tunnel.EmergencyLevel (name, rank, description)
    VALUES (N'ALERTA', 2, N'Situación que requiere seguimiento operativo y posible activación de recursos.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.EmergencyLevel WHERE name = N'EMERGENCIA')
BEGIN
    INSERT INTO tunnel.EmergencyLevel (name, rank, description)
    VALUES (N'EMERGENCIA', 3, N'Situación grave que requiere actuación coordinada y/o servicios externos.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.IncidentFamily WHERE code = N'TRA')
BEGIN
    INSERT INTO tunnel.IncidentFamily (code, name, description)
    VALUES (N'TRA', N'Tráfico', N'Incidentes de circulación, retenciones, accidentes y afectaciones viarias.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.IncidentFamily WHERE code = N'AVA')
BEGIN
    INSERT INTO tunnel.IncidentFamily (code, name, description)
    VALUES (N'AVA', N'Avería', N'Averías técnicas, pérdida de sistemas, fallos de control o equipamiento.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.IncidentFamily WHERE code = N'FOC')
BEGIN
    INSERT INTO tunnel.IncidentFamily (code, name, description)
    VALUES (N'FOC', N'Incendio', N'Fuego, humo, incendio de vehículo o incendio en instalaciones.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.IncidentFamily WHERE code = N'AMB')
BEGIN
    INSERT INTO tunnel.IncidentFamily (code, name, description)
    VALUES (N'AMB', N'Ambiental', N'Condiciones ambientales: CO, NOx, opacidad, ventilación o calidad del aire.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.IncidentFamily WHERE code = N'ILI')
BEGIN
    INSERT INTO tunnel.IncidentFamily (code, name, description)
    VALUES (N'ILI', N'Iluminación', N'Incidencias relacionadas con iluminación normal, emergencia o túnel.');
END;

IF NOT EXISTS (SELECT 1 FROM tunnel.CodeScheme WHERE name = N'GENERIC_TUNNEL_SAFETY')
BEGIN
    INSERT INTO tunnel.CodeScheme (name, description, pattern_hint)
    VALUES (
        N'GENERIC_TUNNEL_SAFETY',
        N'Esquema genérico adaptable a protocolos de seguridad de túneles.',
        N'[nivel]-[familia]-[variante opcional]'
    );
END;

COMMIT;
PRINT N'Creación/validación de TunnelSafetyDB finalizada correctamente.';
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK;

    DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE();
    DECLARE @ErrNum INT = ERROR_NUMBER();
    DECLARE @ErrLine INT = ERROR_LINE();

    PRINT N'ERROR creando TunnelSafetyDB.';
    PRINT N'Número: ' + CAST(@ErrNum AS NVARCHAR(20));
    PRINT N'Línea: ' + CAST(@ErrLine AS NVARCHAR(20));
    PRINT N'Mensaje: ' + @ErrMsg;

    THROW;
END CATCH;

-------------------------------------------------------------------------------
-- 10) COMPROBACIÓN FINAL
-------------------------------------------------------------------------------
PRINT N'Comprobación final de tablas creadas en [TunnelSafetyDB].[tunnel]';

SELECT
    DB_NAME() AS current_database,
    s.name AS schema_name,
    t.name AS table_name
FROM sys.tables t
JOIN sys.schemas s ON s.schema_id = t.schema_id
WHERE s.name = N'tunnel'
ORDER BY t.name;

SELECT
    COUNT(*) AS tunnel_schema_table_count
FROM sys.tables t
JOIN sys.schemas s ON s.schema_id = t.schema_id
WHERE s.name = N'tunnel';

PRINT N'Consulta de documentación disponible:';
PRINT N'SELECT * FROM tunnel.v_TableDocumentation ORDER BY table_name;';
PRINT N'SELECT * FROM tunnel.v_ColumnDocumentation ORDER BY table_name, column_id;';
PRINT N'SELECT * FROM tunnel.v_DatabaseDictionary ORDER BY table_name, item_type, column_id;';

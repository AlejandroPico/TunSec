# Tunnel Safety Decision Support DB

Repositorio de base de datos SQL Server para un sistema de soporte a decisiones en protocolos de seguridad de túneles.

## Archivos

- `001_create_tunnel_safety_db.sql`: script completo de creación de la base de datos `TunnelSafetyDB`.
- `DATA_DICTIONARY_TunnelSafetyDB.md`: desglose documental de tablas y atributos.
- `DATA_DICTIONARY_TunnelSafetyDB.txt`: misma documentación en texto plano.

## Requisitos

- SQL Server 2022.
- DBeaver, SSMS, Azure Data Studio u otro cliente SQL compatible.
- Permisos para crear bases de datos.

## Ejecución en DBeaver

1. Abre una conexión a SQL Server.
2. Abre un editor SQL.
3. Ejecuta el archivo `001_create_tunnel_safety_db.sql` completo.
4. Refresca la conexión o el nodo `Databases`.
5. Verifica:

```sql
SELECT name
FROM sys.databases
WHERE name = N'TunnelSafetyDB';

USE TunnelSafetyDB;

SELECT s.name AS schema_name, t.name AS table_name
FROM sys.tables t
JOIN sys.schemas s ON s.schema_id = t.schema_id
WHERE s.name = N'tunnel'
ORDER BY t.name;
```

## Documentación interna en la base de datos

El script crea extended properties `MS_Description` para tablas y columnas.

Puedes consultar la documentación interna con:

```sql
USE TunnelSafetyDB;

SELECT * FROM tunnel.v_TableDocumentation
ORDER BY table_name;

SELECT * FROM tunnel.v_ColumnDocumentation
ORDER BY table_name, column_id;

SELECT * FROM tunnel.v_DatabaseDictionary
ORDER BY table_name, item_type, column_id;
```

## Notas importantes

- `NVARCHAR(MAX)` es compatible con SQL Server 2022. Si DBeaver lo marca en rojo, normalmente es un falso positivo del parser.
- `ISJSON()` es compatible y el script fuerza `COMPATIBILITY_LEVEL = 160`.
- El script no contiene ninguna referencia a `Cocherama`; está pensado para un sistema nuevo.

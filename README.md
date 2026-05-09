# TunnelSafetyDB - Paquete final con un único SQL

Generado: 2026-05-09 22:14:44 UTC

Este paquete contiene la versión final preparada para subir a Git.

## Estructura

```text
TunnelSafetyDB_Final_Unico_SQL/
├── README.md
├── sql/
│   └── TunnelSafetyDB_FINAL_RESET_CREATE_VERIFY.sql
├── txt/
│   └── TunnelSafetyDB_FINAL_RESET_CREATE_VERIFY.txt
└── docs/
    ├── DATA_DICTIONARY_TunnelSafetyDB.md
    ├── DATA_DICTIONARY_TunnelSafetyDB.txt
    └── EXECUTION_GUIDE_DBEAVER.md
```

## Importante

Solo hay **un archivo SQL** en la carpeta `sql`.

Ese archivo único:

- Borra `TunnelSafetyDB` si existe.
- Crea `TunnelSafetyDB` desde cero.
- Crea todas las tablas en el orden adecuado.
- Crea claves primarias, foráneas, checks e índices.
- Inserta datos base.
- Crea documentación interna consultable.
- Ejecuta comprobaciones finales.

## Archivo SQL principal

```text
sql/TunnelSafetyDB_FINAL_RESET_CREATE_VERIFY.sql
```

## Copia en texto plano

El mismo contenido está duplicado como TXT:

```text
txt/TunnelSafetyDB_FINAL_RESET_CREATE_VERIFY.txt
```

## Ejecución en DBeaver

Ejecuta el SQL completo con:

```text
Execute SQL Script / Alt + X
```

No ejecutes el archivo por selección parcial.

## Documentación interna en la base de datos

Después de ejecutar el script, puedes consultar:

```sql
USE TunnelSafetyDB

SELECT *
FROM tunnel.DatabaseDocumentation
ORDER BY table_name, object_type, column_name
```

## Documentación externa

La carpeta `docs` contiene el diccionario de datos en Markdown y TXT.

# Guía de ejecución en DBeaver

## Archivo único que debes ejecutar

```text
sql/TunnelSafetyDB_FINAL_RESET_CREATE_VERIFY.sql
```

## Qué hace

Este es el único archivo SQL del paquete. Hace todo de principio a fin:

1. Entra en `master`.
2. Si existe `TunnelSafetyDB`, la pone en modo `SINGLE_USER` y la borra.
3. Crea `TunnelSafetyDB` desde cero.
4. Configura compatibilidad SQL Server 2022.
5. Entra en `TunnelSafetyDB`.
6. Crea el schema `tunnel`.
7. Crea todas las tablas en orden correcto.
8. Crea claves primarias, foráneas, checks e índices.
9. Inserta datos base mínimos.
10. Crea documentación interna en `tunnel.DatabaseDocumentation`.
11. Ejecuta comprobaciones finales dentro del mismo SQL.

## Cómo ejecutarlo

En DBeaver usa:

```text
Execute SQL Script
```

Atajo habitual:

```text
Alt + X
```

No uses `Ctrl + Enter` ni ejecutes por selección, porque podrías lanzar solo una parte del script.

## Por qué no lleva punto y coma

La versión funcional en tu DBeaver ha sido la versión sin `;`. Algunas configuraciones de DBeaver parten los scripts por punto y coma y pueden cortar bloques T-SQL. SQL Server acepta este script sin puntos y coma.

## Resultado esperado

Al final del script verás:

- Lista de tablas del schema `tunnel`.
- Conteo total de tablas.
- Listado de documentación interna.

También deberías ver en el panel izquierdo:

```text
TunnelSafetyDB
└── Schemas
    └── tunnel
        └── Tables
```

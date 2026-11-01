# Evidencia · Búsqueda de profesionales sobre el catálogo real (Isla 1, issue #31)

**Fecha:** 2026-11-01 · **Operación:** la aplicación cargó el snapshot real de la cátedra en una PostgreSQL descartable (solo lectura contra la cátedra) y la búsqueda se consultó después solo con la copia local

Host, puerto, token y credenciales omitidos a propósito (ver ADR 0001). Los datos del catálogo son ficticios (enunciado 1).

## Resultado

| Dato | Valor |
|---|---|
| Carga inicial | `sync_state`: versión 1, `IDLE`, resultado `SNAPSHOT` |
| Catálogo cargado | 8 categorías, 24 profesionales y 44 horarios (igual que en la [evidencia del cliente REST](isla-1-cliente-rest-snapshot.md)) |
| Sin filtros, `size=3` | 24 resultados en 8 páginas; primeros: Acosta Romina (218), Alvarez Diego (207) y Benitez Rocio (223), por apellido y nombre |
| `categoryId=101` | 3 profesionales |
| `enabled=true` / `enabled=false` | 21 / 3 (suman 24) |
| `available=true` / `available=false` | 21 / 3 (suman 24, sin solaparse) |
| `available=true&dayOfWeek=MONDAY` | 9 |
| `available=true&dayOfWeek=SUNDAY` | 0 |
| `name=ACOSTA` (en mayúsculas, apellido real) | 1 |
| Detalle del profesional 218 | Ginecologia; martes 13:00 a 16:00 y jueves 11:00 a 14:00, turnos de 30 min, ambos habilitados |
| Categorías | 8, por nombre, todas habilitadas |

## Errores

| Pedido | Respuesta |
|---|---|
| `size=101` | `400` `BUSQUEDA_INVALIDA`: "size debe estar entre 1 y 100" |
| `dayOfWeek=MONDAY` sin `available` | `400` `BUSQUEDA_INVALIDA`: "dayOfWeek requiere available" |
| `dayOfWeek=LUNES` | `400` `BUSQUEDA_INVALIDA` (sin repetir el valor) |
| `/api/professionals/99999` | `404` `PROFESIONAL_NO_ENCONTRADO` |
| `/api/professionals/abc` | `400` `BUSQUEDA_INVALIDA` |

## Cómo se ejecutó

1. Una PostgreSQL 17 en un contenedor descartable, en un puerto local distinto al de la base de desarrollo, que no se tocó.
2. La aplicación con `ATLAS_SEARCH_ENDPOINT=true`, la verificación periódica y el consumidor de Kafka apagados, y la sincronización al arrancar encendida: con la base vacía pidió el snapshot a la cátedra una sola vez.
3. Consultas con `curl` a `localhost`, por ejemplo `curl 'localhost:8080/api/professionals?available=true&dayOfWeek=MONDAY&size=100'`.
4. Al terminar se detuvo la aplicación y se borró el contenedor. El log de la aplicación no contenía el token ni la contraseña de la base (0 apariciones).

## Observaciones

- **Lo que se comprobó:** que la búsqueda funciona sobre el catálogo real y que los números son coherentes entre sí (habilitados más no habilitados, y disponibles más no disponibles, suman el total).
- **Lo que no se comprobó:** el rechazo de accesos no autenticados, porque la seguridad base (ítem 5) todavía no existe; hasta entonces la búsqueda está apagada por defecto. Tampoco el rendimiento, que con 24 profesionales no dice nada, ni el trato de las tildes con datos reales: no se buscó ningún nombre con tilde.
- La cátedra sigue publicando solo la versión 1, así que la búsqueda no se probó con un catálogo que cambia entre versiones. Esa parte está cubierta por las pruebas de sincronización, no por esta evidencia.

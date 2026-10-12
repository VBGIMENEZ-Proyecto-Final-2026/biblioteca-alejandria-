# Evidencia · El README de atlas-catalogo, ejecutado desde cero (Isla 1, ítem 13)

**Fecha:** 2026-10-12 · **Operación:** se siguieron, tal cual están escritos, los pasos de "Inicio rápido desde cero" del README sobre un clon limpio, contra la cátedra real (solo lectura) y una base PostgreSQL nueva

Host, puerto, usuario, contraseña, token y `groupId` omitidos a propósito (ver ADR 0001).

## Resultado

| Paso del README | Resultado |
|---|---|
| `git clone` y `git checkout dev` | Clon limpio en `dev`, sin `.env` ni carpeta `build` |
| `cp .env.example .env` y completar los valores | Sin placeholders sin completar |
| `docker compose up -d` | La base queda `healthy` |
| `./gradlew build` | `BUILD SUCCESSFUL`: 161 tests, 0 fallas, con el `.env` real presente (los tests siguen usando valores ficticios) |
| `./gradlew bootRun` y `curl localhost:8080/actuator/health` | `{"groups":["liveness","readiness"],"status":"UP"}` |
| `SELECT ... FROM sync_state` (el que sugiere el README) | `local_version = 1`, `status = IDLE`, `last_result = SNAPSHOT`, `last_snapshot_reason = BASE_VACIA`, sin error |
| Conteo de la copia local | 8 categorías, 24 profesionales, 44 horarios (los mismos totales que el snapshot REST y que Redis) |
| Consumidor de Kafka | Unido al grupo con la única partición asignada (aparece en el log) |
| Arranque de la aplicación | Unos 6 s |

## Controles de seguridad comprobados en el log de esa ejecución

El log del arranque (unas 200 líneas, `INFO`) se revisó buscando los valores reales del `.env`:

| Valor | Apariciones |
|---|---|
| Token JWT de la cátedra | 0 |
| Contraseña de Redis | 0 |
| Usuario de Redis | 0 |
| Contraseña de la base | 0 |
| Servidores, topic y consumer group de Kafka | Sí: los registra el propio cliente de Kafka en `INFO` al volcar su configuración y al asignar la partición |

Conclusión: ningún secreto llega al log; sí aparecen datos de la integración que no son secretos pero no conviene publicar, así que los logs no se pegan en issues ni evidencias sin revisarlos (ver [seguridad](../arquitectura/seguridad-atlas-catalogo.md)).

## Cómo se ejecutó

1. Clon de la rama `dev` en una carpeta temporal, fuera del repositorio.
2. Para no tocar una base local anterior, se usó otro nombre de proyecto de Compose, otro nombre de contenedor (mediante un archivo `compose.override.yaml` descartable) y otro puerto (`ATLAS_DB_PORT` en el `.env` de la prueba). El resto de los pasos es el del README.
3. Con Podman rootless se exportó `DOCKER_HOST` al socket de Podman, como indica el README.
4. Al terminar se detuvo la aplicación con una señal de interrupción (cierre ordenado) y se borró la base de la prueba con su volumen.

## Observaciones

- El único ajuste al README que salió de la prueba fue la salida de `curl`: el endpoint devuelve también `groups`, no solo `status`. Se corrigió.
- El README ahora aclara que **no hay integración continua**: `./gradlew build` se corre a mano antes de cada PR.
- La verificación no cubre Windows ni macOS: solo Linux con Podman rootless.
- La prueba no recorre la búsqueda ni la seguridad de la API porque todavía no existen (ítems 11 y 5).

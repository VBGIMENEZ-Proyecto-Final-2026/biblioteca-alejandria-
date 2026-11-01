# Evidencia · Seguridad base con la aplicación real (Isla 1, issue #33)

**Fecha:** 2026-11-01 · **Operación:** la aplicación corrió contra una PostgreSQL descartable, con la sincronización con la cátedra apagada (no se llamó a la cátedra), y se la atacó con `curl` usando tokens firmados con una clave de **desarrollo**

Esta evidencia cubre "rechazo de accesos no autenticados o no autorizados" (enunciado 11). Los tokens son de prueba, firmados con `scripts/dev-token.sh`: **no se probó con un token real de `cronos-turnos`**, que todavía no existe. Ningún token ni clave figura acá.

## Arranque sin clave pública

| Prueba | Resultado |
|---|---|
| Arrancar sin `ATLAS_JWT_PUBLIC_KEY_LOCATION` ni `ATLAS_JWT_PUBLIC_KEY` | La aplicación **no arranca** (código de salida 1) y dice qué falta: "Falta ATLAS_JWT_PUBLIC_KEY_LOCATION (archivo) o ATLAS_JWT_PUBLIC_KEY (PEM) con la clave pública con la que cronos-turnos firma los JWT" |

## Matriz de accesos

| Pedido | Respuesta |
|---|---|
| Sin token → `GET /api/categories` | `401` `NO_AUTENTICADO` |
| Sin token → `GET /api/professionals` | `401` `NO_AUTENTICADO` |
| Sin token → `GET /api/sync/status` | `401` `NO_AUTENTICADO` |
| Sin token → `GET /actuator/health` | `200` (público) |
| Sin token → `GET /actuator/env` | `401` `NO_AUTENTICADO` |
| `ROLE_USER` → `GET /api/categories` | `200` |
| `ROLE_USER` → `GET /api/professionals?available=true&size=5` | `200` |
| `ROLE_USER` → `GET /api/professionals/99999` | `404` `PROFESIONAL_NO_ENCONTRADO` (pasó la autorización) |
| `ROLE_USER` → `GET /api/sync/status` | `403` `SIN_PERMISO` |
| `ROLE_ADMIN` → `GET /api/sync/status` | `200` |
| `ROLE_ADMIN` → `GET /api/professionals` | `200` |
| `ROLE_GUEST` (rol que no existe en atlas) → `GET /api/categories` | `403` `SIN_PERMISO` |
| `ROLE_ADMIN` → `POST /api/professionals` | `403` `SIN_PERMISO` (ninguna escritura está permitida) |
| `ROLE_ADMIN` → `GET /api/lo-que-sea` | `403` `SIN_PERMISO` (se deniega en vez de revelar si la ruta existe) |
| Token con `exp` una hora en el pasado → `GET /api/categories` | `401` `NO_AUTENTICADO` (**ver la observación sobre este caso**) |
| Token firmado con **otra clave** (con `ROLE_USER` y `ROLE_ADMIN`) → `GET /api/sync/status` | `401` `NO_AUTENTICADO` |
| Texto que no es un JWT → `GET /api/categories` | `401` `NO_AUTENTICADO` |

**Cabeceras de un `401`:** `HTTP/1.1 401`, `WWW-Authenticate: Bearer` y `Content-Type: application/problem+json`. **Cuerpo de un `403`:** `{"title":"Forbidden","status":403,"message":"error.http.403","path":"/api/sync/status","code":"SIN_PERMISO"}`: no dice qué rol faltaba.

## CORS

Con `ATLAS_CORS_ALLOWED_ORIGINS` vacío, una consulta previa (`OPTIONS`) de un origen cualquiera no recibe ningún encabezado `Access-Control-*` (0 encabezados), así que un navegador la bloquea. Con orígenes configurados, solo esos orígenes reciben los encabezados, solo para `GET` y `OPTIONS` y sin credenciales (cubierto por `CorsApiTest`).

## Trazabilidad en el log

La ejecución dejó **13 rechazos** registrados, uno por cada petición rechazada, con el formato `Acceso rechazado: <código> <método> <ruta>`, y el usuario en los `403`. Del resumen del log se vieron 11, agrupados así:

| Registro | Veces |
|---|---|
| `403 GET /api/sync/status usuario=ana` | 2 |
| `401 GET /api/categories`, sin token | 2 |
| `401 GET /api/categories`, token inválido (vencido y texto que no es un JWT) | 2 |
| `401 GET /api/sync/status`, sin token | 1 |
| `401 GET /api/sync/status`, token inválido (firmado con otra clave) | 1 |
| `403 POST /api/professionals usuario=root` | 1 |
| `403 GET /api/lo-que-sea usuario=root` | 1 |
| `403 GET /api/categories usuario=visita` | 1 |

Los otros 2 no se listaron en el resumen que se revisó: por el conteo corresponden a `GET /api/professionals` y a `GET /actuator/env` sin token, que también se pidieron en la matriz.

En el log completo hubo **0 apariciones de un token** (cadenas `eyJ`) y 0 de la contraseña de la base.

## Cómo se ejecutó

1. Una PostgreSQL 17 en un contenedor descartable, en un puerto local distinto del de la base de desarrollo, que no se tocó.
2. Un par de claves RSA de 2048 bits generado en un directorio temporal, de desarrollo y fuera del repositorio. La aplicación leyó solo la **pública** (`ATLAS_JWT_PUBLIC_KEY_LOCATION`).
3. Tokens con `scripts/dev-token.sh <clave-privada> <usuario> <roles>`. La firma de uno se verificó de forma independiente con `openssl dgst -sha256 -verify` (`Verified OK`), y el script rechaza usuarios o roles con caracteres que romperían el JSON.
4. La aplicación con la sincronización al arrancar, la verificación periódica y el consumidor de Kafka apagados, para no llamar a la cátedra.
5. Al terminar se detuvo la aplicación, se borró el contenedor y se borraron las claves de desarrollo.

## Verificaciones adicionales

| Verificación | Resultado |
|---|---|
| `./gradlew build` | 320 tests, 0 fallos, 0 omitidos (antes 248) |
| Clon limpio de la rama | Los tests de contexto, de seguridad y de carga de clave pasan, así que la clave pública de prueba versionada alcanza |
| Nueve mutaciones manuales | Las nueve detectadas por los tests: no validar el vencimiento, abrir el estado a `ROLE_USER`, autenticar en vez de denegar el resto, no exigir `sub` y `exp`, dejar público todo Actuator, CORS con credenciales, CORS con escritura, quitar `WWW-Authenticate` y revelar el rol en el `403` |

## Observaciones

- **El caso del token vencido, con una salvedad que descubrí después:** el script de desarrollo emitía ese token con `iat` (ahora) **posterior** a `exp` (una hora atrás), es decir, un token incoherente, que Spring rechaza por esa razón y no por estar vencido. El `401` fue correcto, pero esta ejecución no prueba el vencimiento en sí. **El vencimiento y la tolerancia de 60 s sí están probados** en `SeguridadApiTest` con tokens coherentes (emitidos una hora antes de vencer), y verificados por mutación: al quitar la validación de vigencia, esos tests fallan. El script se corrigió después (rama `fix/dev-token-iat`, aún sin publicar).
- **Lo que se comprobó:** que sin credenciales o con un token inválido, vencido o mal firmado no se accede a nada; que un token válido accede solo a lo que su rol permite; que lo no listado se deniega; y que los rechazos quedan registrados sin el token.
- **Lo que no se comprobó:** el flujo con un token emitido por `cronos-turnos` (los claims `sub` y `auth` son una propuesta), el canal cifrado entre la app y atlas (atlas escucha en HTTP), ni el comportamiento con un reloj muy desfasado más allá de la tolerancia de 60 s, que sí cubre un test.
- La búsqueda y el estado quedaron encendidos por defecto; con esta evidencia se cierra el pendiente de "no encenderlos sin seguridad" de las secciones de los ítems 11 y 12.

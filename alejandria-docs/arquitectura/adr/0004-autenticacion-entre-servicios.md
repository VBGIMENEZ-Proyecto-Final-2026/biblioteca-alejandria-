# ADR 0004 · Autenticación entre servicios: JWT del usuario propagado, firmado con RS256

**Estado:** aceptado por el alumno el 2026-11-01 y implementado en `atlas-catalogo` (ítem 5); a confirmar con el docente (consulta 1) · **Componentes:** atlas-catalogo y cronos-turnos · **Contexto:** Isla 1, issue #33 · **Relacionado:** [ADR 0002](0002-stack.md) (D12) y [seguridad de atlas-catalogo](../seguridad-atlas-catalogo.md)

## Contexto

El enunciado (sección 9) pide que los dos servicios se comuniquen con contratos protegidos con JWT, y que el alumno **defina si se propaga el JWT del usuario o se usa un JWT técnico, y justifique cómo preserva la identidad, la autorización y la trazabilidad**. En todos los casos hay que validar firma, vigencia y permisos del token recibido. El docente pidió, además, elegir una opción y explicar por qué esa y por qué no las otras.

Quién llama a quién:

| Llamada | Para qué |
|---|---|
| app → cronos-turnos | Registro, inicio de sesión, disponibilidad, bloqueos y reservas. Cronos emite el JWT del usuario ([decisión del 2026-11-01](#decisiones-que-condicionan-esta)). |
| **app → atlas-catalogo** | Búsqueda y filtros sobre el catálogo local, directo y con el mismo JWT. |
| cronos-turnos → atlas-catalogo | Horario semanal vigente de un profesional, solo antes de iniciar una operación nueva, y esa operación siempre nace de una acción de un usuario. |
| cronos-turnos → cátedra | Disponibilidad real, bloqueos y reservas, con la cuenta técnica de la cátedra (no pasa por atlas). |

Dos hechos acotan la decisión:

1. **La app le pega a atlas directamente con el JWT que emite cronos.** Por eso atlas tiene que validar tokens de usuario que no emite, elija lo que elija para la llamada entre servicios. Cualquier opción se suma a eso; no lo reemplaza.
2. **La recuperación tras un reinicio de cronos se reconcilia contra la cátedra, no contra atlas.** Entonces ningún proceso de fondo de cronos necesita llamar a atlas sin un usuario de por medio.

## Opciones evaluadas

| Opción | Pros | Contras |
|---|---|---|
| **A. JWT del usuario propagado, RS256** | Un solo tipo de token y un solo camino de validación en atlas. Conserva la identidad del usuario en los logs de atlas. Atlas solo tiene la clave pública: no puede forjar tokens. | Cualquier usuario autenticado podría llamar al endpoint interno de horarios (es el mismo dato que ya ve en la búsqueda: riesgo bajo). Cronos debe reenviar el encabezado `Authorization` en cada llamada. |
| **B. JWT técnico entre servicios, RS256** | Separa el contrato interno del público y permite restringir por *scope*. Sirve cuando no hay un usuario de por medio. | No reemplaza a A, porque la app igual le pega directo a atlas: atlas valida **dos** tipos de token. Se pierde la identidad del usuario. Un encabezado `X-On-Behalf-Of` sin firma se falsifica, y firmarlo ya es un intercambio de tokens (*token exchange*): más código, documentación y pruebas. |
| **C. Secreto compartido (HS256/HS512)** | Es el valor por defecto de JHipster y no requiere código extra. | Atlas guarda la clave de firma, así que podría forjar el token de cualquier usuario: si se compromete atlas, se compromete la autenticación de cronos. El secreto vive en dos lugares y se rota en dos. Solo es defendible documentando la limitación (consulta 7). |

## Decisión

**Opción A: se propaga el JWT del usuario, firmado con RS256.**

El argumento central es que **atlas no debe poder forjar tokens de usuario**. Con A, además, la consulta 7 (si alcanza con documentar la limitación del secreto compartido) queda sin objeto, y D12 del ADR 0002 deja de ser una propuesta aislada: es parte de esta decisión.

- **Por qué no B:** su única ventaja real, contar con un token sin usuario, no se necesita: no hay ningún proceso de fondo que llame a atlas. A cambio duplica la validación y esconde al usuario detrás de un token de servicio.
- **Por qué no C:** es la más simple, pero es la única de las tres en la que un servicio puede falsificar la identidad de los usuarios del otro. Con un solo servicio emisor y otro validador, no hay motivo para aceptar ese riesgo.

### Cómo se cumplen los tres requisitos del enunciado

| Requisito | Cómo se cumple con A |
|---|---|
| **Identidad** | El `sub` del token viaja de la app a cronos y de cronos a atlas, sin intermediarios que la reescriban. |
| **Autorización** | Atlas autoriza por las autoridades del propio token (`ROLE_USER` para buscar, `ROLE_ADMIN` para el estado de la sincronización). Nada depende de un identificador enviado por el cliente. |
| **Trazabilidad** | Cada rechazo en atlas queda en el log con método, ruta y usuario; los accesos quedan atribuidos al mismo usuario en los dos servicios. |

### Detalle de implementación

- **atlas (hecho, ítem 5):** servidor de recursos con la **clave pública** RSA. Valida firma RS256 (rechaza `HS256`, `none` y otros algoritmos), vigencia con tolerancia de reloj y la presencia de `sub` y `exp`. No tiene clave privada ni usuarios.
- **cronos (pendiente):** emite con la clave privada (RS256), valida sus propios endpoints y reenvía el `Authorization` del usuario a atlas con un interceptor del cliente HTTP.
- **app:** guarda un solo token y lo manda a los dos servicios.
- **Claims** (propuesta, a fijar con cronos): `sub` con el usuario y `auth` con las autoridades, como el token de JHipster. Así sirve igual si cronos usa JHipster o no, y el enunciado pide un modelo de usuario compatible con el de JHipster. Atlas acepta las autoridades como texto con espacios o como lista, y el nombre del claim es configurable.
- **La clave pública se carga de un archivo o de una variable, no por JWKS.** Con JWKS atlas dependería de que cronos esté levantado para validar tokens, y eso contradice la independencia de los servicios. Costo: rotar la clave exige reiniciar atlas, aceptable acá.
- **Sin clave, atlas no arranca.** Fallar rápido es más seguro que arrancar con la API abierta o rechazándolo todo sin explicación.

## Consecuencias

- **Atlas depende de la clave pública de cronos.** Mientras cronos no exista, se prueba con un par de claves de desarrollo (`scripts/dev-token.sh`) y con claves generadas en los tests.
- **Cualquier usuario autenticado puede llamar a lo que atlas expone con `ROLE_USER`**, incluido un futuro endpoint de horarios para cronos. Es el mismo dato que ya devuelve la búsqueda. Si eso dejara de ser cierto, se agrega una autoridad específica en el claim y se exige en esa ruta.
- **Límites aceptados:** no se valida el emisor (`iss`) ni la audiencia (`aud`); no hay revocación de tokens, que dependen de su vencimiento; la comunicación entre la app y los servicios no está cifrada por atlas (ver [seguridad](../seguridad-atlas-catalogo.md)).
- **Cronos debe emitir RS256.** Si se usa JHipster y su firma por defecto fuera simétrica, habría que reemplazar el codificador y el decodificador (a verificar, ver más abajo).

## Cuándo revisar

- Si el docente (consulta 1) pide un JWT técnico: pasa a B para la llamada entre servicios, y atlas conserva la validación de usuario por la llamada directa de la app.
- Si algún proceso de cronos necesitara llamar a atlas sin un usuario de por medio (por ejemplo una conciliación periódica): ahí aparece la necesidad real de B.
- Si el docente no acepta RS256, o cronos no puede emitirlo: se reemplaza por un ADR con secreto compartido (opción C), documentando que atlas puede forjar tokens. La validación de atlas está detrás de propiedades, así que el cambio sería de configuración y no de arquitectura.
- Si aparece más de un emisor o hay que rotar claves sin reiniciar: se evalúa JWKS.

## Decisiones que condicionan esta

- **Los usuarios, el registro, el inicio de sesión y la emisión del JWT viven en cronos-turnos** (2026-11-01). Atlas solo valida. Es lo natural: cronos es dueño de las reservas por usuario (enunciado 4.2) y mantiene la identidad fuera del catálogo.
- **JHipster en cronos sigue abierto.** No afecta a esta decisión, pero sí a su costo. Según las notas de la versión 9.3.0, JHipster genera sobre Spring Boot 4.1, igual que atlas. Lo que **no está verificado** y habría que comprobar con un proyecto descartable es si firma con secreto compartido por defecto (lo que obligaría a pasar a RS256), si deja inactivo al usuario recién registrado (el enunciado lo exige activo) y qué herramienta de migraciones usa. Se resolverá en un ADR propio.

## Referencias

Enunciado 3.2, 4.2, 9 y 10; [ADR 0002](0002-stack.md) (D12); consultas 1, 5, 6 y 7 al docente.

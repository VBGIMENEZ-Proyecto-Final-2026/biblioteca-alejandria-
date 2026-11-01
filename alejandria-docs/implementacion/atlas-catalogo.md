# Implementación de atlas-catalogo · evolución

**Componente:** atlas-catalogo · **Isla:** 1 · Eolia · **Actualizado:** 2026-10-21

Este documento registra cómo se fue construyendo el código, en el orden en que se hizo, y qué se decidió o se cambió respecto del diseño original (ADR [0002](../arquitectura/adr/0002-stack.md) y [0003](../arquitectura/adr/0003-arquitectura.md), [modelo de datos](../modelo-datos/atlas-catalogo.md)). Las decisiones de diseño siguen viviendo en los ADR; acá van las que aparecieron **al implementar**, con su porqué, el costo aceptado y cuándo conviene revisarlas. Se agrega una sección por cada issue de código que se cierra.

## Línea de tiempo

| Fecha | Issue / PR | Qué se hizo | Estado |
|---|---|---|---|
| 2026-10-07 | #5, #6 | Stack (ADR 0002) y arquitectura interna (ADR 0003) | Cerrados |
| 2026-10-08 | #7 | Modelo de datos (decisiones A a L) | Cerrado |
| 2026-10-08 | #9 · PR #10 | Esqueleto Spring Boot 4.1, Java 25, Gradle; Flyway sin tablas; ArchUnit con reglas vacías | Cerrado |
| 2026-10-11 | #11 · PR #13 | Migración `V1`, entidades JPA, repositorios con *upsert* y candado, tests contra PostgreSQL real | Cerrado |
| 2026-10-11 | #12 · PR #14 | Cliente REST del snapshot, con reintentos y errores traducidos; verificado contra la cátedra real | Cerrado |
| 2026-10-12 | #15 · PR #16 | Sincronización completa: aplicar el snapshot y arranque con base vacía | Cerrado |
| 2026-10-12 | #17 · PR #18 | Cliente Redis de la cátedra: metadata, cambios y entidades; verificado contra la cátedra real | Cerrado |
| 2026-10-12 | #19 · PR #20 | Sincronización incremental: planificador, aplicación de cambios por versión, arranque y verificación periódica | Cerrado |
| 2026-10-12 | #21 · PR #22 | Consumidor de Kafka del aviso `CatalogUpdated` como disparador de la sincronización | Cerrado |
| 2026-10-12 | #30 · PR #26 | Documentación: README ejecutable desde cero, estrategia de sincronización y seguridad (ítem 13) | Cerrado (primera entrega; se completa con 5 y 11) |
| 2026-10-14 | Sin issue | Visor de la base (DBeaver) y resumen de la implementación contrastado con el código; sin cambios de código | Hecho |
| 2026-10-21 | Sin issue | Decisiones locales del grupo 1 (regla de sincronización, Flyway, Kafka configurable, disponibilidad) contrastadas con el código; sin cambios de código | Hecho |
| 2026-10-21 | #27 · PR #23 | Seguridad de Kafka configurable: `PLAINTEXT` por defecto y perfil `kafka-sasl` con `SASL_SSL` | Cerrado |
| 2026-10-21 | #28 · PR #24 | Filtro de disponibilidad detrás de una interfaz, sobre la copia local | Cerrado |
| 2026-10-21 | #29 · PR #25 | Estado de la sincronización (`GET /api/sync/status`, apagado por defecto hasta el ítem 5) y pruebas de falla y recuperación | Cerrado |
| 2026-11-01 | #31 · PR #32 | Búsqueda de profesionales sobre la copia local: categoría, nombre, habilitado y disponibilidad, detalle y categorías | Hecho; protegida por el ítem 5 (PR #34) |
| 2026-11-01 | #33 · PR #34 | Seguridad base: JWT RS256 de `cronos-turnos` validado con su clave pública, roles por ruta, `401` y `403` como `problem+json` y CORS | Mergeado a `dev` el 2026-11-01; el issue sigue abierto hasta fijar los claims con `cronos-turnos` |

Tests automatizados tras cada paso: 5 (esqueleto) → 19 (persistencia) → 40 (cliente REST) → 58 (sincronización completa) → 86 (cliente Redis) → 133 (sincronización incremental) → 161 (consumidor de Kafka) → 199 (Kafka con seguridad configurable, disponibilidad y estado, ya en `dev`) → 248 (búsqueda) → 320 (seguridad base).

## Mapa del código frente al ADR 0003

| Paquete (`ar.edu.um.atlas`) | Contenido hoy | Issue |
|---|---|---|
| `domain` | `ProfessionalCategory`, `Professional`, `WeeklySchedule`, `SyncState` y los enums `SyncStatus`, `SyncResult`, `SnapshotReason` | #11 |
| `application.repository` | Un repositorio de Spring Data por entidad; con `disableAllExcept` para las bajas | #11, #15 |
| `application.port.out` | `CatalogSnapshotSource`, `CatalogSnapshot` y las excepciones `CatedraUnavailableException`, `CatedraCredentialRejectedException`, `CatedraInvalidResponseException` | #12 |
| `application.port.out` | `CatalogChangeSource`, `CatalogMetadata` y `VersionChanges` (las excepciones son las mismas) | #17 |
| `adapters.out.rest` | Cliente declarativo `CatedraSnapshotApi`, DTO del JSON, conversión al modelo y `CatedraRestSnapshotSource` | #12 |
| `application.sync` | Casos de uso `SyncCatalog` (orquesta), `ApplySnapshot` y `ApplyIncremental`; `SyncStateRecorder`, `SyncErrorCode`, `SyncException`, `SyncTrigger` y `SyncLimits` | #15, #19 |
| `application.sync.plan` | `SyncPlanner` (función pura) y `SyncPlan` | #19 |
| `adapters.in.startup` | `SyncOnStartup`: corre la sincronización al terminar de arrancar | #15, #19 |
| `adapters.in.schedule` | `PeriodicSyncCheck`: verificación periódica de la versión | #19 |
| `adapters.in.kafka` | `CatalogUpdatedListener`, `CatalogUpdatedParser` y `ResilientConsumerLifecycle`: el aviso `CatalogUpdated` como disparador | #21 |
| `config` | `CatedraProperties`, `CatedraRestConfig`, `SyncConfig` y `SyncProperties` (reloj, cotas y disparadores), `CatedraRedisProperties`, `CatedraRedisConfig`, `CatedraKafkaProperties` y `CatedraKafkaConfig` | #12, #15, #17, #19, #21 |
| `adapters.out.redis` | `CatedraRedisChangeSource`, `RedisJson` y `RedisEntityMapper` | #17 |
| `adapters.in` (web) | Todavía no existe | issues por abrir |

## #11 · Migraciones y persistencia

**Qué quedó:** `V1__esquema_catalogo.sql` con las cuatro tablas del modelo y la fila inicial de `sync_state`; entidades en `domain` (solo `jakarta.persistence`); repositorios con *upsert* nativo `ON CONFLICT (id) DO UPDATE` y lectura con `SELECT ... FOR UPDATE`; `ddl-auto=validate`; las reglas de ArchUnit de `domain` y `application` sin `allowEmptyShould`. La decisión L del modelo se cumplió tal cual: el esquema se creó en este issue y no en el esqueleto.

**Pruebas (14 nuevas):** esquema creado por Flyway desde cero e índice parcial; ida y vuelta de cada entidad (con segundos en `time` y descripción nula); *upsert* repetido sin duplicados y actualizando datos; claves foráneas; `day_of_week` inválido; `sync_state` con una sola fila; y el candado con dos transacciones en paralelo (la segunda espera hasta que la primera termina).

### Decisiones tomadas al implementar

**Upsert por consulta nativa con un método `default` sobre la entidad.**
- *Alternativa:* `save()` de Spring Data.
- *Por qué:* con un ID asignado por la cátedra, `save()` hace primero un `SELECT` y después `INSERT` o `UPDATE`; no es atómico ni idempotente bajo concurrencia. El `ON CONFLICT` sí lo es (decisión B).
- *Costo:* hay SQL nativo en los repositorios y la lista de columnas se duplica entre la migración y la consulta; un test de ida y vuelta por entidad lo cubre.
- *Revisar:* si el modelo crece en columnas.

**Las relaciones se guardan como IDs (`categoryId`, `professionalId`), sin `@ManyToOne`.**
- *Alternativa:* asociaciones JPA.
- *Por qué:* el *upsert* nativo trabaja con IDs y la copia es de solo lectura; evita cargas perezosas y consultas que nadie pidió.
- *Costo:* la búsqueda tendrá que unir tablas de forma explícita.
- *Revisar:* al implementar la API de búsqueda.

**`SyncState` con *setters* públicos.**
- *Alternativa:* métodos con intención (`iniciarCorrida()`, `registrarError(...)`).
- *Por qué:* no hay lógica de sincronización todavía y no conviene inventar un comportamiento sin caso de uso.
- *Costo:* cualquiera puede dejar la fila en un estado incoherente.
- *Revisar:* al implementar la sincronización completa: ahí aparecen las transiciones reales.

**`ddl-auto=validate` también en la aplicación, no solo en los tests.**
- *Por qué:* si una entidad y la migración no coinciden, que falle el arranque y no una consulta en producción.
- *Costo:* ninguno conocido; hubo que verificar que Hibernate acepta `text` frente a `String` y `time` frente a `LocalTime`, y lo acepta.

**Tests con `@SpringBootTest` y rollback, sin `@DataJpaTest`.**
- *Por qué:* es lo mismo que ya usaba el test de contexto (misma configuración de Testcontainers) y evita sumar el starter de tests de JPA.
- *Costo:* arranca el contexto completo; el contexto se comparte entre clases, así que el costo se paga una vez.
- *El candado va en una clase aparte*, sin `@Transactional`, porque necesita dos transacciones reales en paralelo.

## #12 · Cliente REST de la cátedra (snapshot)

**Qué quedó:** el puerto `CatalogSnapshotSource` y su adaptador `CatedraRestSnapshotSource`, con un `RestClient` declarativo (`@HttpExchange`). Pide `GET /api/synchronization/snapshot` con el JWT técnico como `Bearer`, convierte la respuesta, reintenta solo ante fallas transitorias y traduce los errores. No guarda nada en la base: eso es el issue de sincronización completa.

**Pruebas (21 nuevas):** 17 contra un servidor HTTP local que simula a la cátedra (snapshot completo y vacío, campos desconocidos, horas con y sin segundos, cabecera `Authorization`, `401` sin `code`, `403` con `code`, `500`, `503` con recuperación y persistente, timeout de lectura, cátedra caída, JSON inválido, campo obligatorio ausente, día de la semana desconocido, `404`, redirección y token fuera de mensajes) y 4 de validación de variables al arrancar. Verificación manual contra la cátedra real: [evidencia](../evidencias/isla-1-cliente-rest-snapshot.md).

### Decisiones tomadas al implementar

**Sin login técnico: el `CATEDRA_ID_TOKEN` se usa directo como `Bearer`.** (Decidido con el usuario antes de abrir el issue.)
- *Alternativa:* llamar a `/api/authenticate` y renovar el token.
- *Por qué:* el JWT técnico dura un año (referencia 4) y renovarlo exige usuario y contraseña, que no forman parte del set `CATEDRA_*` (ADR 0001).
- *Costo:* cuando el token venza hay que rotarlo a mano en el `.env`; el `401` lo dice en el mensaje.
- *Revisar:* si el token llegara a vencer durante el proyecto.
- *Consecuencia en el diseño:* el ADR 0003 hablaba de "login técnico y snapshot" en `adapters.out.rest`; se corrigió.

**El snapshot entrega entidades del dominio, no clases de datos nuevas.** *(Cambia el issue, que decía "modelo propio".)*
- *Alternativa:* tres registros propios de datos intermedios.
- *Por qué:* al estar ya mergeado el #11, las entidades son el modelo de dominio (compromiso 1 del ADR 0003) y la sincronización las aplica sin otra conversión. Un modelo intermedio duplicaría tres clases para nada.
- *Costo:* el puerto depende de `domain` (permitido por las reglas) y las entidades quedan construidas fuera de una sesión de JPA; no importa, porque se escriben por *upsert* nativo.
- *Revisar:* si el contrato de la cátedra empezara a diferir del modelo local.

**Reintentos con `RetryTemplate` de Framework 7, de forma programática, y solo ante red, timeout y `503`.**
- *Alternativa:* `@Retryable` declarativo.
- *Por qué:* el predicado (qué se reintenta) y la traducción posterior de errores quedan juntos, y los tests usan esperas de milisegundos. Un `500` no se reintenta: la referencia solo garantiza `503` para "dependencia no disponible" (matriz 13.1).
- *Valores:* conexión 3 s, lectura 30 s, 2 reintentos con espera de 500 ms que se duplica hasta 5 s. Se eligió una lectura larga porque el snapshot es una sola respuesta que puede crecer; la latencia medida hacia la cátedra es de 220 a 520 ms ([smoke test](../evidencias/isla-0-smoke-test.md)) y el snapshot real tardó unos 2,7 s. Peor caso: tres intentos de 30 s más 1,5 s de espera, alrededor de 91 s.
- *Costo:* una cátedra colgada bloquea la corrida hasta un minuto y medio; es aceptable porque la corrida de sync es exclusiva y no hay usuarios esperando.
- *Revisar:* con el planificador de sync (reintentos de más alto nivel) y con datos de producción.

**Errores traducidos a tres excepciones selladas, decididas por `status` y `code`.**
- *Qué hay:* cátedra no disponible (red, timeout, `5xx`), credencial rechazada (`401` y `403`, que pueden venir sin `code`) y respuesta inválida (cuerpo ilegible, campo obligatorio ausente o un `4xx` inesperado).
- *Por qué una sola jerarquía:* quien llame solo necesita saber si reintentar más tarde, rotar el token o avisar de un problema de contrato. El `status` y el `code` viajan en la excepción para el estado de sync.
- *Revisar:* al diseñar el estado y manejo de errores de la sincronización.

**Los campos obligatorios se validan al convertir, no en la base.**
- *Alternativa:* dejar que un `null` falle al insertar.
- *Por qué:* falla antes de tocar la copia local y con un mensaje que nombra el campo; un snapshot a medio validar nunca debe llegar a aplicarse.
- *Costo:* si la cátedra vuelve opcional un campo hoy obligatorio, el cliente lo rechaza hasta que se ajuste.

**El token no se escribe en ningún lado y no se siguen redirecciones.**
- *Por qué:* un redireccionamiento podría reenviar la cabecera a otro destino. Las propiedades enmascaran el token en `toString` y los mensajes de error solo llevan `status` y `code`; hay tests para ambas cosas.

**Pruebas con un servidor HTTP local (`com.sun.net.httpserver`), no con `MockRestServiceServer` ni WireMock.** *(El issue admitía cualquiera de los dos.)*
- *Por qué:* ejercita la fábrica de peticiones real, los timeouts reales y Jackson 3 de punta a punta, sin dependencias nuevas.
- *Costo:* el servidor falso es código propio de tests.

**Se agregó `spring-boot-starter-restclient`.** En Boot 4.1 el constructor de la fábrica de peticiones y los ajustes de timeouts y redirecciones están en un módulo aparte; sin el starter no se pueden configurar de forma estándar.

**Jackson 3: el riesgo del ADR 0002 (D2) quedó cubierto por tests.** El ADR pedía no asumir el comportamiento por defecto con campos desconocidos y con `HH:mm`. Con el `RestClient` construido a mano se ignoran los campos desconocidos y se leen horas con y sin segundos, tal como exige la referencia 4.

**Variables nuevas.** `CATEDRA_REST_CONNECT_TIMEOUT`, `CATEDRA_REST_READ_TIMEOUT` y `CATEDRA_REST_MAX_RETRIES` son opcionales, con los valores anteriores por defecto. `CATEDRA_REST_BASE_URL` y `CATEDRA_ID_TOKEN` ya figuraban en el `.env.example` y ahora son obligatorias: sin ellas la aplicación no arranca y el mensaje dice cuál falta.

## Cambios al diseño original

| Documento | Cambio | Motivo |
|---|---|---|
| ADR 0003 | El adaptador `rest` ya no hace "login técnico": solo el snapshot | Sin login técnico (ver #12) |
| Backlog | Ítem 7 sin login y con la decisión anotada | Ídem |
| Issue #12 | El snapshot usa entidades del dominio; pruebas con servidor local | Ver decisiones del #12 |

## #15 · Sincronización completa

**Qué quedó:** `ApplySnapshot.run(motivo)` en `application.sync`. Toma `sync_state` con `FOR UPDATE`, pide el snapshot al puerto y lo aplica con los *upserts* del #11 (categorías, profesionales y horarios, en ese orden). Lo que existe localmente y no viene se deshabilita con `UPDATE ... WHERE enabled AND id <> ALL(:ids)`, sin tocar `updated_at`. Al terminar bien, `sync_state` queda con `local_version` igual a la del snapshot, `IDLE`, resultado `SNAPSHOT`, el motivo y la hora, y sin error. Si algo falla, la transacción se revierte y el error se registra aparte con uno de cuatro códigos: `CATEDRA_NO_DISPONIBLE`, `CATEDRA_CREDENCIAL_RECHAZADA`, `CATEDRA_RESPUESTA_INVALIDA` y `SNAPSHOT_NO_APLICABLE` (clave foránea u otra falla al aplicar). Con la base vacía, `SnapshotOnStartup` lo ejecuta al terminar de arrancar con el motivo `BASE_VACIA`.

**Pruebas (18 nuevas):** inicialización, reconstrucción con versión menor y entidades que desaparecen, colección vacía (deshabilita todo), repetición idéntica, *rollback* completo con una clave foránea que falla después de aplicar categorías y profesionales, un código por cada falla del puerto, error que se limpia con un éxito posterior, dos corridas en paralelo (la segunda ni le pide el snapshot a la cátedra hasta que la primera termina), arranque con y sin versión local, cátedra caída en el arranque y el arranque completo con la propiedad encendida. La cátedra se simula con una implementación de prueba del puerto.

### Decisiones tomadas al implementar

**Una sola transacción con el pedido a la cátedra adentro** (como proponía el issue).
- *Alternativa:* pedir el snapshot antes y abrir la transacción solo para aplicarlo.
- *Por qué:* con dos corridas en paralelo, la segunda podría aplicar un snapshot más viejo después del más nuevo y hacer retroceder la copia. El test de paralelismo lo comprueba: la segunda corrida no hace su pedido hasta que la primera termina.
- *Costo:* una conexión ocupada durante el pedido (peor caso del cliente, unos 91 s). El catálogo es chico y la corrida es exclusiva.
- *Revisar:* si el catálogo crece o hay más corridas concurrentes.

**El error se registra después de revertir, sin `REQUIRES_NEW`.** *(Cambia la nota del issue, que sugería `REQUIRES_NEW` o equivalente.)*
- *Por qué:* `ApplySnapshot.run` no es transaccional; abre la transacción principal con `TransactionTemplate`, y cuando esta terminó (revertida y con el candado liberado) abre otra para escribir el error. Una transacción anidada que intentara tomar la misma fila mientras la principal la tiene bloqueada quedaría esperando para siempre.
- *Costo:* si la base está caída, el error no se puede registrar; se deja en el log y se lanza igual la excepción original.

**Mensajes de error fijos por código.** El texto que queda en `sync_state` sale del código, no de la excepción original: así nunca aparecen detalles internos, hosts ni el token. El detalle completo va al log.

**Disparador en `ApplicationReadyEvent`, no en un `ApplicationRunner`.**
- *Por qué:* el servidor ya está arriba y `/actuator/health` responde mientras se pide el snapshot, que puede tardar hasta un minuto y medio en el peor caso.
- *Costo:* durante esa ventana la API (cuando exista) devolvería datos vacíos; hay que contemplarlo en el issue de estado.
- *La comprobación "¿hay versión local?" se hace antes de tomar el candado.* Con una sola instancia no hay carrera; si hubiera otra, el peor caso es un snapshot repetido, que es inocuo.

**La propiedad que lo apaga es `atlas.sync.on-startup` (variable `ATLAS_SYNC_ON_STARTUP`), encendida por defecto.** El bean `SnapshotOnStartup` se declara en `config` con `@ConditionalOnProperty`, así la clase del adaptador no conoce la propiedad. Los tests de contexto la apagan una sola vez en `src/test/resources/application.properties`, en lugar de hacerlo test por test.

**`SyncState` tiene métodos con intención (`registrarSnapshotAplicado`, `registrarError`), pero se conservan los *setters*.** *(Adelantado en la nota del #11.)* Los *setters* los usa todavía el test de persistencia para armar estados arbitrarios; se pueden retirar cuando el issue de estado y manejo de errores sea el segundo consumidor.

**`Clock` como bean.** La corrida pone la hora con un `Clock` inyectado (sistema en UTC), para poder fijarlo en tests futuros.

**Bajas con un arreglo de IDs y `<> ALL`.** Con una lista vacía, `NOT IN ()` no es SQL válido; con un arreglo vacío, `id <> ALL('{}')` es verdadero para toda fila y deshabilita todo, así que no hace falta una consulta aparte. Lo cubre el test de colección vacía.

## Cambios al diseño original (#15)

| Documento | Cambio | Motivo |
|---|---|---|
| Issue #15 | El error se escribe en otra transacción abierta después del *rollback*, sin `REQUIRES_NEW` | Ver decisión arriba |
| Issue #15 | Se conservan los *setters* de `SyncState` | Los usa el test de persistencia |

## #17 · Cliente Redis de la cátedra

**Qué quedó:** el puerto `CatalogChangeSource` (`fetchMetadata`, `fetchChanges(versión)` y `fetchCategories`/`fetchProfessionals`/`fetchWeeklySchedules` por conjunto de IDs) y su adaptador `CatedraRedisChangeSource` con `StringRedisTemplate`. Solo lecturas puntuales sobre `catedra:sync:*` (`HGETALL` de la metadata, `GET` de `changes:{v}` y `HMGET` por lotes de 500 de los Hashes de entidades), con el modelo del dominio como salida. Lo que falta (metadata, `changes:{v}` o un ID) se devuelve vacío u omitido: decide quien llama. La configuración usa las variables `CATEDRA_REDIS_*` validadas al arrancar y reintentos acotados ante fallas de conexión.

**Pruebas (28 nuevas, entre ellas 2 reglas de ArchUnit):** contra un Redis real en un contenedor, con un usuario ACL que solo puede leer `catedra:sync:*` (si el adaptador escribiera o leyera otra clave, Redis respondería `NOPERM`): metadata completa, ausente, sin fecha y con valores inválidos; cambios presentes, vacíos, ausentes, con versión que no coincide y con JSON o campos inválidos; entidades presentes y ausentes, campos desconocidos, horas con y sin segundos, más IDs que el tamaño del lote, JSON inválido e `id` distinto del field; Redis caído (tres intentos), contraseña incorrecta (sin reintentos y sin la contraseña en el mensaje) y usuario sin permisos. Verificación contra la cátedra real: [evidencia](../evidencias/isla-1-cliente-redis.md).

### Decisiones tomadas al implementar

**Las mismas tres excepciones del cliente REST, con `status` y `code` nulos.** *(Era "Propuesto" en el issue.)*
- *Alternativa:* una jerarquía propia para Redis.
- *Por qué:* quien sincroniza solo necesita saber si reintentar más tarde, rotar la credencial o avisar de un problema de contrato, y eso se traduce a los mismos códigos funcionales (`CATEDRA_NO_DISPONIBLE`, `CATEDRA_CREDENCIAL_RECHAZADA`, `CATEDRA_RESPUESTA_INVALIDA`) sin un caso más. Se ajustaron los comentarios de las clases, que hablaban solo de REST.
- *Costo:* las excepciones nombran a "la cátedra" y no a Redis; el mensaje sí lo aclara.
- *Revisar:* si el estado de sincronización necesitara distinguir qué fuente falló.

**La metadata sale del Hash `catedra:sync:metadata` con una sola lectura, no de las dos claves String.** *(Era "Propuesto".)*
- *Por qué:* `current` y `oldest` salen como un par consistente de un mismo comando y viene `publishedAt`. Las dos claves String se leerían con dos comandos y podrían pertenecer a versiones distintas si la cátedra publica en el medio.
- *Costo:* si la cátedra dejara de publicar el Hash y solo mantuviera las claves String, el cliente vería "metadata ausente" y el planificador escalaría a snapshot (`NO_VERIFICABLE`); es seguro, pero no incremental.
- *Revisar:* si ese caso apareciera en la cátedra real. Hoy el Hash existe y coincide con la versión REST.

**Propiedades propias (`catedra.redis.*`) y conexión Lettuce armada a mano, sin `spring.data.redis.*`.** Así no se depende de variables que Boot tome por su cuenta y los mensajes de error nombran la variable de entorno que falta, como en el cliente REST. La conexión es perezosa: la aplicación arranca aunque Redis no responda.

**El health de Redis se apaga y los repositorios de Redis también.** Con el starter en el classpath, Boot sumaría Redis a `/actuator/health`, y una caída de la cátedra marcaría al servicio como `DOWN` aunque pueda seguir sirviendo la copia local. Los repositorios de Spring Data Redis se desactivan porque todos los repositorios son de JPA. *Revisar:* si el endpoint de estado de la sincronización necesita mostrar la salud de Redis (no será por `/actuator/health`).

**Credencial rechazada detectada por el texto del error de Redis (`WRONGPASS`, `NOAUTH`, `NOPERM`).**
- *Por qué:* ni Lettuce ni Spring Data Redis tienen una excepción tipada para esas respuestas; llegan envueltas en excepciones de conexión o de comando. El test lo comprueba contra un Redis real con una contraseña incorrecta y con un usuario sin permisos.
- *Costo:* depende del texto de las respuestas de Redis, que es estable desde Redis 6 (ACL) pero no es un contrato tipado.
- *Revisar:* al actualizar Lettuce o Spring Data Redis, o si la cátedra cambia de motor.

**Los JSON de Redis tienen sus propias clases de datos (`RedisJson`) en el adaptador, aunque se parecen a las del REST.** Las reglas de ArchUnit prohíben que un adaptador dependa de otro, y compartirlas exigiría subirlas a `application` (que no conoce Jackson). *Costo:* tres clases casi iguales. *Revisar:* si el contrato de Redis y el del snapshot divergieran, esta separación evita tocar dos lados.

**Mapper y validación como en el REST:** se exigen los campos obligatorios al convertir y se ignoran los desconocidos; además, un `id` distinto del field o una `version` distinta de la pedida es dato inválido, porque indicaría un Redis inconsistente.

**No hay TLS ni lectura de los Hashes completos.** La referencia no menciona TLS para Redis. La lectura completa de entidades (sin pasar por `changes`) queda afuera: el snapshot REST ya cubre la reconstrucción.

**Variables nuevas.** `CATEDRA_REDIS_CONNECT_TIMEOUT` (3 s), `CATEDRA_REDIS_COMMAND_TIMEOUT` (5 s) y `CATEDRA_REDIS_MAX_RETRIES` (2) son opcionales. `CATEDRA_REDIS_HOST`, `_PORT`, `_USERNAME` y `_PASSWORD` ya figuraban en el `.env.example` y ahora son obligatorias: sin ellas la aplicación no arranca y el mensaje dice cuál falta.

## Cambios al diseño original (#17)

| Documento | Cambio | Motivo |
|---|---|---|
| Issue #17 | Las decisiones "Propuesto" (excepciones compartidas y metadata por Hash) quedaron como se propusieron | Ver decisiones arriba |
| ADR 0003 | Sin cambios: el puerto se llama `CatalogChangeSource` y el adaptador vive en `adapters.out.redis`, como estaba previsto | |
| Verificación | La cátedra real tiene `oldest = 1` y `changes:1` existe; dato para la consulta 10 | [Evidencia](../evidencias/isla-1-cliente-redis.md) |

## #19 · Sincronización incremental

**Qué quedó:** `SyncCatalog.run(disparo)` en `application.sync`. Lee la versión local; con la base vacía pide el snapshot sin tocar Redis; si hay versión local, lee la metadata, la pasa al planificador (`SyncPlanner`, función pura con la tabla del ADR 0003) y ejecuta el plan: `Noop` (se registra en `sync_state`), `Snapshot` con su motivo o `Incremental`. `ApplyIncremental` aplica cada versión en su propia transacción con candado y avanza la versión local solo si termina bien; relee la metadata al terminar y sigue si apareció una versión nueva, con un tope de 1000 versiones por corrida. Los disparadores son el arranque (`SyncOnStartup`, que reemplaza al del #15) y una verificación periódica (`PeriodicSyncCheck`, 60 s por defecto, con espera fija entre corridas).

**Pruebas (47 nuevas, entre ellas 12 del planificador y 1 regla de ArchUnit):** una por fila del planificador y por borde; contra PostgreSQL real con Redis y REST simulados: una versión, varias en orden, entidad nueva y baja lógica, repetir una versión, `current` que sube durante la corrida, tope de versiones, falta de `changes`, un ID ausente (se deshabilita), dos IDs ausentes (misma versión y versiones distintas), referencia inexistente con *rollback*, datos inválidos, metadata ausente o ilegible, local adelantada y fuera de ventana, Redis caído con recuperación, credencial rechazada, falla inesperada con su código, primera carga sin leer Redis, dos corridas en paralelo, los dos disparadores; y de punta a punta con un Redis real. Dos pruebas de mutación manuales confirmaron que los tests detectan una regresión en el contador de IDs ausentes y en la comprobación de versión dentro de la transacción. Verificación contra la cátedra real: [evidencia](../evidencias/isla-1-sync-incremental.md).

### Decisiones tomadas al implementar

**La metadata se lee fuera de la transacción y la versión se verifica dentro.**
- *Alternativa:* leer todo dentro de una transacción larga que abarque el plan.
- *Por qué:* planificar es barato y puede quedar viejo; lo que no puede quedar viejo es aplicar la versión `v`. Dentro de la transacción de cada versión, con el candado tomado, se comprueba que `local_version` siga siendo `v - 1`. Si otra corrida avanzó, esta termina sin error y vuelve a planificar (hasta tres veces). El test de paralelismo lo prueba: la segunda corrida espera el candado, ve la versión aplicada, replanifica y termina en `Noop`.
- *Costo:* una conexión de base ocupada durante unas pocas lecturas rápidas de Redis (timeout de comando de 5 s, 3 intentos), como el snapshot en el #15.
- *Revisar:* si Redis se vuelve lento y la ocupación de conexiones molesta.

**Qué escala a snapshot y qué deja un error.** *(Amplía la lista del issue.)*
- Escalan, porque un snapshot lo arregla: falta `changes:{v}`, una clave foránea rota al aplicar una versión, un segundo ID ausente en la corrida, y datos de Redis que no se pueden interpretar (metadata, `changes` o entidades con JSON inválido o campos obligatorios ausentes).
- No escalan, porque un snapshot no lo arregla: Redis o la cátedra caídos, credencial rechazada y fallas de base de datos. Quedan en `ERROR` con su código y la siguiente verificación reintenta. Un éxito posterior limpia el error.
- *Por qué los datos inválidos escalan:* es el mismo criterio de "estado no verificable" que ya usa el ADR para la metadata; si Redis publica basura de forma permanente, la corrida siguiente ya es `Noop` porque el snapshot deja la versión vigente.
- *Revisar:* si en la práctica un Redis inconsistente provocara snapshots repetidos.

**El contador de IDs ausentes cuenta en toda la corrida, no en cada versión.** Con un ID ausente en la versión 2 y otro en la 3, la versión 2 queda aplicada (con su baja) y la 3 se revierte y pide snapshot, como pide el ADR ("en la misma corrida"). Un ID ausente se deshabilita sin tocar `updated_at` (decisión J). *Costo:* una corrida puede dejar aplicada una baja lógica y aun así terminar en snapshot; la copia es consistente en todo momento.

**`Noop` también se registra en `sync_state`.** *(Era "Propuesto".)* La fila queda con `NOOP` y la hora de la verificación, y se limpia cualquier error anterior. *Costo:* una escritura corta con candado por intervalo (una por minuto). *Revisar:* si el endpoint de estado necesita distinguir "verificado hace poco" de "se aplicó algo": `last_result` y `last_sync_at` ya lo permiten.

**La primera carga no depende de Redis.** *(Era "Propuesto".)* Con la base vacía se va directo al snapshot, que solo necesita REST: una Redis caída no impide armar la copia.

**`SnapshotSyncException` pasa a `SyncException`, y un `SyncStateRecorder` compartido escribe errores y `NOOP`.** El nombre anterior quedó chico cuando la excepción empezó a salir también de la corrida incremental. El registro de errores (en otra transacción, ya revertida la principal y liberado el candado) se extrajo de `ApplySnapshot` para reutilizarlo; el comportamiento del #15 no cambia.

**Código de error nuevo `INCREMENTAL_NO_APLICABLE`.** *(Era "Propuesto".)* Para fallas de aplicación que no son de datos ni de integración, por ejemplo la base. Los otros cuatro códigos se reutilizan.

**Tope de versiones por corrida (`ATLAS_SYNC_MAX_VERSIONS_PER_RUN`, 1000) y verificación cada 60 s (`ATLAS_SYNC_CHECK_INTERVAL`).** *(Eran "Propuestos".)* El tope evita corridas sin fin si la cátedra publica muchísimas versiones; la siguiente verificación continúa. La verificación usa espera fija entre corridas (no tasa fija): nunca se superponen aunque una demore. *Revisar:* el intervalo con datos reales y cuando exista Kafka, que lo hará menos importante.

**El arranque ahora siempre verifica, también con versión local.** Antes (#15) con versión local no hacía nada; ahora recoge lo que la cátedra publicó mientras la aplicación estaba apagada. Una falla en el arranque nunca tumba la aplicación: se captura cualquier excepción y queda registrado en `sync_state`.

**La propiedad `atlas.sync.periodic-check` apaga la verificación y los tests de contexto la apagan junto con la de arranque.** Un test de contexto la enciende con un intervalo de 300 ms para probar el planificador de tareas real, y se cierra al terminar la clase (`@DirtiesContext`) para que no siga disparando contra un contenedor que se apaga.

## Cambios al diseño original (#19)

| Documento | Cambio | Motivo |
|---|---|---|
| Issue #19 | Los datos inválidos de `changes` y de las entidades también escalan a snapshot | Mismo criterio de estado no verificable que la metadata |
| Issue #19 | `SnapshotSyncException` se renombra a `SyncException` y los errores se escriben desde `SyncStateRecorder` | La excepción ahora sirve a las dos corridas |
| ADR 0003 | Sin cambios: el planificador, la aplicación incremental y las escaladas son los del ADR | |
| Fila 5 del planificador | Sigue la lectura literal (`local < oldest` → snapshot) hasta la consulta 10 | La cátedra real solo publica la versión 1: sin más datos para decidir |

## #21 · Consumidor de Kafka

**Qué quedó:** `CatalogUpdatedListener` en `adapters.in.kafka`, un listener de `spring-kafka` con confirmación manual (`MANUAL_IMMEDIATE`) que por cada `CatalogUpdated` ejecuta `SyncCatalog` con un disparo nuevo (`AVISO_DE_VERSION`) y confirma el *offset* solo después de que la corrida terminó. `CatalogUpdatedParser` interpreta el mensaje (solo exige `eventType` y `newVersion`, ignora el resto). `ResilientConsumerLifecycle` arranca el contenedor sin poder tumbar la aplicación. La configuración (`CatedraKafkaProperties` y `CatedraKafkaConfig`) usa las variables `CATEDRA_KAFKA_*`, se apaga con `atlas.sync.kafka.enabled` y no usa `spring.kafka.*`.

**Pruebas (28 nuevas, entre ellas 9 del interpretador, 4 del ciclo de vida, 7 de las variables y 1 regla de ArchUnit):** con un Kafka real en un contenedor (una partición, como el topic de la cátedra) y PostgreSQL real, con REST y Redis simulados: evento nuevo (la copia avanza y el *offset* del grupo queda confirmado, comprobado contra el broker), duplicado, atrasado y adelantado (todos `Noop`), mensajes corruptos seguidos de uno válido, un corrupto como único mensaje, campos desconocidos con un esquema nuevo, y una falla de Redis con tres intentos, descarte con el *offset* confirmado y recuperación con el aviso siguiente; la aplicación completa arrancando con Kafka inaccesible; y variables faltantes. Dos mutaciones manuales mostraron que los tests detectan una regresión al no confirmar el *offset* tras agotar los reintentos y al no confirmar un mensaje corrupto. Verificación contra la cátedra real: [evidencia](../evidencias/isla-1-consumidor-kafka.md).

### Decisiones tomadas al implementar

**Si la sincronización falla: reintentos acotados y descarte con el *offset* confirmado.** *(Era "Propuesto".)*
- *Alternativa:* no confirmar y seguir reintentando, que respeta literalmente "confirmar solo si persistió" pero puede dejar el consumidor trabado mientras Redis o la cátedra estén caídos.
- *Por qué:* el evento no es la fuente de nada, solo un aviso. El error ya quedó en `sync_state` y la verificación periódica lo reintenta sola, así que descartar el mensaje no pierde información y el consumidor sigue sano. Son tres intentos en total, con espera de 1 s que se duplica.
- *Costo:* un aviso que no se pudo procesar tarda hasta un intervalo de verificación (60 s) en aplicarse.
- *Revisar:* si la cátedra empezara a publicar versiones con una frecuencia que hiciera pesado el intervalo.

**El cliente de Kafka tumbaba el arranque si los servidores no se resuelven.** *(Hallazgo de las pruebas.)* Al construir el consumidor, el cliente lanza `No resolvable bootstrap urls` si el nombre no se resuelve, y Spring dejaba fallar el arranque entero. Puede pasar en producción (por ejemplo, con la red de la cátedra caída al arrancar). `ResilientConsumerLifecycle` envuelve el arranque del contenedor, lo registra sin servidores ni credenciales y reintenta en segundo plano cada 10 s hasta que arranca; si los servidores existen pero no contestan, el propio cliente reintenta conectarse. *Costo:* un hilo daemon y un poco de código propio. El contenedor se crea con `autoStartup = false` para que no lo arranque Spring por su cuenta.

**Los *offsets* de Kafka son acumulativos, así que un mensaje descartado sin confirmar pasa desapercibido mientras llegue otro después.** Un test con un mensaje corrupto seguido de uno válido no detecta la omisión de la confirmación, porque la del válido cubre al anterior. Se agregó un test con un corrupto como único mensaje, que sí la detecta (comprobado con una mutación). Sin esa confirmación el *offset* quedaría atrás y al reiniciar se reprocesaría el corrupto: inocuo, pero deja un *lag* permanente en el grupo.

**El valor del mensaje se lee como texto y se interpreta en el adaptador.** Un `ErrorHandlingDeserializer` sobre `StringDeserializer` evita que un mensaje no deserializable trabe el consumo; el JSON se lee con el mismo criterio que el cliente Redis (campos desconocidos ignorados). Solo se exigen `eventType = CatalogUpdated` y `newVersion`: `eventId` es opcional y solo sirve para el log, y `schemaVersion` no se valida porque el evento es un disparador y una versión de esquema nueva no cambia lo que hace. *Costo:* se aceptaría un `schemaVersion` desconocido; el contrato de la cátedra garantiza compatibilidad dentro de v1.

**No se guarda una tabla de `eventId` procesados.** La comparación de versiones ya es la idempotencia: un duplicado, un atrasado y uno que adelanta la versión que Redis todavía no publicó terminan en `Noop` (probado). *Costo:* cada evento cuesta una lectura de Redis y una escritura corta de `NOOP`; una ráfaga de avisos no se agrupa. *Revisar:* si apareciera una ráfaga que lo justifique.

**Confirmación inmediata y un mensaje por vez.** `MANUAL_IMMEDIATE` y `max.poll.records=1` hacen que el *offset* se confirme en el momento del `acknowledge`, ya con la copia persistida, y que una falla no arrastre a otros mensajes. `max.poll.interval.ms` en 10 minutos deja margen amplio sobre el peor caso de una corrida (un snapshot, unos 91 s) para que no haya un *rebalance* por demora. `allow.auto.create.topics=false`: un consumidor nunca debe crear el topic de la cátedra.

**Grupo nuevo desde `latest`.** *(Era "Propuesto".)* El arranque ya sincroniza la copia, así que no hace falta reprocesar el historial del topic. Con un grupo que ya tiene *offset* se retoma desde ahí y se recuperan los avisos que llegaron con la aplicación apagada.

**Propiedades propias y `KafkaAutoConfiguration` excluida.** Como con Redis, no se usa `spring.kafka.*`: los mensajes de error nombran la variable de entorno que falta y no se depende de lo que Boot arme por su cuenta. La autoconfiguración de Kafka se excluye para que no cree productores ni fábricas que este servicio no usa. `atlas.sync.kafka.enabled` (por defecto encendida) apaga todo y, con ella apagada, las tres variables no se exigen.

**Sin autenticación.** La cátedra no documenta una para Kafka y la conexión real funcionó sin credenciales. *Revisar:* confirmarlo con el docente antes de la Isla 3.

## Cambios al diseño original (#21)

| Documento | Cambio | Motivo |
|---|---|---|
| Issue #21 | Se agregó el arranque resiliente (`ResilientConsumerLifecycle`) | Un nombre que no resuelve tumbaba el arranque; el issue exigía que arrancara sin Kafka |
| Issue #21 | Test adicional de un mensaje corrupto como único mensaje | Los *offsets* acumulativos ocultaban la omisión de la confirmación |
| ADR 0003 | Sin cambios: el consumidor es un disparador más, ack manual tras persistir y mensajes corruptos descartados | |

## Ítem 13 · Documentación (primera entrega)

**Qué quedó:** un README reorganizado y verificado desde cero ([evidencia](../evidencias/isla-1-readme-desde-cero.md)), la [estrategia de sincronización](../arquitectura/sincronizacion-atlas-catalogo.md) (planificador, transacciones, idempotencia, matriz de fallas y recuperación, estado y limitaciones) y el documento de [seguridad](../arquitectura/seguridad-atlas-catalogo.md) (lo que existe, lo que falta y las limitaciones). Es una primera entrega: se vuelve a revisar al implementar la seguridad base (5) y la búsqueda (11), que todavía no existen.

### Decisiones tomadas al documentar

**El detalle de la sincronización vive en un solo lugar (`alejandria-docs`) y el README lo resume y enlaza.**
- *Alternativa:* dejar toda la explicación en el README, como estaba (la tabla del planificador, las transacciones y la matriz de fallas repetidas ahí).
- *Por qué:* dos copias del mismo texto se desactualizan por separado, y el criterio del ítem exige que la documentación coincida con lo implementado. El README conserva lo que hace falta para ejecutar (requisitos, pasos, variables, tests) y un resumen de cinco líneas.
- *Costo:* para leer la estrategia completa hay que seguir un enlace.

**Cada afirmación del documento de sincronización cita la prueba que la respalda** (idempotencia y fallas), y lo que no tiene prueba se dice: por ejemplo, el corte del proceso a mitad de una corrida es consecuencia de la transacción y no hay un test con un corte real, y el candado entre varias instancias es por diseño y las pruebas lo ejercitan con dos hilos de una instancia.

**El documento de seguridad dice qué falta en vez de describir lo planeado como hecho.** Hoy atlas no tiene endpoints de negocio, así que no hay JWT que validar; el documento lista los controles que sí existen (con su prueba), lo que debe agregar el ítem 5 y las limitaciones asumidas (sin TLS ni autenticación de Kafka, sin integración continua, logs con datos de la integración).

**El README se verificó siguiendo sus propios pasos en un clon limpio**, no solo releyéndolo. Eso encontró una imprecisión (la salida de `curl`) y confirmó con datos reales que el token y las contraseñas no llegan al log.

## 2026-10-14 · Visor de la base y resumen de la implementación

**Qué se hizo:** no hubo cambios de código. Se incorporó un visor para mirar la base de atlas y se armó un resumen de la implementación que se contrastó contra el código.

**Visor de la base.** DBeaver Community, instalado en la máquina de desarrollo. Se conecta a la PostgreSQL local del `compose.yaml` (contenedor `atlas-db`), que escucha solo en `127.0.0.1`. Los datos de conexión salen del `.env` (valores de ejemplo en `.env.example`: host `localhost`, puerto `5433`, base `atlas`, usuario `atlas`); la contraseña no se documenta. Sirve para ver las tablas del catálogo (`professional_category`, `professional`, `weekly_schedule`) y el estado de la sincronización (`sync_state`: versión local, resultado de la última corrida, código del último error), por ejemplo después de un snapshot o de una verificación sin cambios.

**Resumen de la implementación.** Una guía interactiva de una sola página: mapa de capas, archivos con los fragmentos clave, recorridos paso a paso (arranque, aviso de versión nueva, escalada a snapshot, caída de Redis o de la cátedra, mensajes de Kafka raros o repetidos), simulador del planificador y respuestas armadas para las preguntas que puede hacer el docente. Es un archivo local (no se sube a este repositorio).

**Cómo se contrastó con el código** (rama `dev`; `src/` es idéntico al de `feature/documentacion`):
- Los fragmentos de código, sus números de línea y la cantidad de líneas de los 58 archivos que lista coinciden con los archivos reales (comparación por script).
- Las referencias `archivo:línea` de los cinco recorridos y de las preguntas caen en la línea que describen, y las seis filas del simulador reproducen `SyncPlanner.plan`.
- Las afirmaciones sueltas (intervalo de 60 s, 12 pruebas del planificador, 7 reglas de ArchUnit, lotes de 500 en Redis, 3 intentos de Kafka, único endpoint `/actuator/health`) se comprobaron en el código.
- La suite completa se corrió con Podman: **161 tests, 0 fallos, 0 errores, 0 omitidos**. La cifra del paso anterior queda confirmada.

### Decisiones tomadas

**La guía queda en un solo archivo local.**
- *Alternativa:* conservar también la copia anterior o publicar la guía en este repositorio.
- *Por qué:* dos copias casi iguales se desfasan, y la guía depende de números de línea de un momento del código; publicarla obligaría a mantenerla en cada cambio.
- *Costo:* los números de línea envejecen con cualquier commit que toque `src/`. Conviene regenerarla al cerrar el próximo ítem de código.

**Las diferencias entre la guía y el código se dicen en la guía.** Dos son diferencias entre el ADR 0003 y el código (el mapa de arriba muestra los nombres y paquetes reales): el caso de uso se llama `ApplyIncremental` y no `ApplyChanges`, y el planificador vive en `application.sync.plan` y no en `domain`. Detalle: en el código existe `domain/SyncStatus`, pero es el enum del estado (`IDLE`/`ERROR`), no el caso de uso de estado del ADR, que corresponde al ítem 12.

## 2026-10-21 · Decisiones locales del grupo 1

**Qué se hizo:** no hubo cambios de código. Se contrastaron cuatro decisiones que se cerraron sin esperar al docente con lo que ya estaba implementado, y se documentó el resultado. Cada una queda como "resuelta localmente, a confirmar con el docente".

| Decisión | Consulta | Estado en el código | Qué cambia |
|---|---|---|---|
| Regla de sincronización conservadora (`local < oldest` → snapshot) | 10 | **Ya implementada** en #19 (`SyncPlanner`, filas 1 a 6 del ADR 0003, con una prueba por fila y por borde) | Solo la documentación: la fila 5 pasa de "pendiente" a "cerrada localmente" |
| Flyway dueño del esquema, Hibernate solo valida | 4 | **Ya implementada** en #11 (`V1__esquema_catalogo.sql`, `ddl-auto=validate`, pruebas con PostgreSQL real) | Solo la documentación (ADR 0002, D6) |
| Kafka con `security.protocol` por configuración (default `PLAINTEXT`, perfil SASL opcional) | 11 | **Implementada** el 2026-10-21 en `feature/kafka-seguridad-configurable` (commit `0710c75`; issue #27, PR #23, mergeado a `dev` el 2026-11-01) | Ver "Kafka con seguridad configurable", más abajo |
| Disponibilidad básica detrás de una interfaz | 2 | **Implementada** el 2026-10-21 en `feature/filtro-disponibilidad` (commit `797bc3c`; issue #28, PR #24, mergeado a `dev` el 2026-11-01); la búsqueda todavía no existe | Ver "Filtro de disponibilidad", más abajo |

### Cómo se contrastó cada una

**Regla de sincronización.** La función pura de la decisión del grupo 1 es la que ya existe, con dos diferencias de forma que no cambian el comportamiento. Vive en `application.sync.plan` y no en `domain` (no depende de Spring ni de JPA, que era lo que importaba). Y devuelve un `SyncPlan` (`Noop`, `Incremental(desde, hasta)` o `Snapshot(motivo)`) en lugar de una enumeración de tres valores: el motivo se guarda en `sync_state.last_snapshot_reason` y el rango lo necesita la aplicación incremental. Además agrega la fila "metadata ausente o incoherente → `Snapshot(NO_VERIFICABLE)`", que la decisión del grupo 1 no contemplaba. El fallback por `changes:{v}` ausente a mitad del rango ya está en #19, no en el planificador.

**Flyway.** Coincide en lo esencial (módulo de Postgres, `validate`, pruebas contra un Postgres real y no H2). Hay tres diferencias con el texto del grupo 1 que se **dejaron como están**:
- *Instancias de PostgreSQL.* El grupo 1 propone una sola instancia con dos bases (`atlas_db` y `cronos_db`) y un script de inicialización. El ADR 0002 (D5) ya había decidido **una instancia propia por repositorio**, y así están hoy los `compose.yaml` de los dos servicios (atlas en `127.0.0.1:5433`, base `atlas`). El script de inicialización no se adopta.
- *Migraciones.* El grupo 1 listaba `V1` (catálogo), `V2` (`sync_state`) y `V3` (`processed_events`). Hay una sola `V1` con las tres tablas del catálogo y `sync_state`, y `processed_events` no existe: la decisión D del modelo es no guardar los `eventId` (la idempotencia sale de los *upserts* y de la versión). Una migración ya aplicada no se edita, así que no se parte.
- *Permisos.* "El usuario de atlas solo tiene permisos sobre su base" se cumple por construcción: cada instancia tiene un único usuario y una única base.

## Ítem 12 · Estado y manejo de errores de la sincronización

**Qué se hizo.** Se expone el estado de la sincronización y se comprueba, con pruebas de punta a punta, que una caída de la cátedra, de Redis o de la aplicación al aplicar no corrompe la copia y que se recupera sola. Rama `feature/estado-sincronizacion` (commit `d125797`, rebasado como `053400d`; issue #29, PR #25, mergeado a `dev` el 2026-11-01).

**Qué hay en el código.**
- `GetSyncStatus` (en `application.sync`): lee `sync_state` en una transacción de solo lectura, **sin** `FOR UPDATE`, y devuelve un `SyncStatusView`.
- `GET /api/sync/status` en `adapters.in.web`: `SyncStatusController` solo traduce (la regla de ArchUnit le prohíbe tocar repositorios y JPA) y `SyncStatusExceptionHandler` devuelve `503` con `problem+json` y `STATUS_NO_DISPONIBLE` si la base no responde. Ambos solo existen con `atlas.sync.status-endpoint.enabled=true`.
- `SyncException.describe`: los logs de falla del arranque, de la verificación periódica y del consumidor de Kafka ahora traen el código funcional y la **clase** de la causa, nunca su mensaje. Antes el consumidor de Kafka registraba la clase del envoltorio del contenedor, que no decía nada.
- Detalle del contrato, del mapa de fallas y de la prueba de cada fila: [estrategia de sincronización](../arquitectura/sincronizacion-atlas-catalogo.md).

**Pruebas (17 nuevas, 178 en total).** Controlador (JSON completo, campos `null`, `503`, solo `GET`); estado contra PostgreSQL real después de cada tipo de corrida y con un error vigente; que consultar no espera a una corrida en curso (con tope de 5 s); que la respuesta y el log no traen secretos; el endpoint apagado responde `404` por defecto; una falla a mitad de varias versiones deja aplicadas las anteriores y la siguiente corrida continúa; y un ciclo de recuperación con la verificación periódica real en el que la aplicación pasa por tres fallas (Redis caído, credencial rechazada de la cátedra y un snapshot que no se puede aplicar) y se recupera sola cada vez. Cuatro mutaciones manuales (la consulta toma el candado, el log trae el mensaje de la causa, un `Noop` no limpia el error y un error pierde la versión local) hicieron fallar los tests esperados.

### Decisiones tomadas

**El endpoint nace apagado.**
- *Alternativa:* esperar a la seguridad base (ítem 5) para hacer todo junto, o encenderlo sin protección.
- *Por qué:* el documento de seguridad dice que no se agregan endpoints de negocio antes del ítem 5, y el 5 espera la reunión con el docente (D12). Apagado, la parte de falla y recuperación no se demora y el endpoint no queda abierto por descuido.
- *Costo:* el criterio "endpoint protegido" del backlog queda **pendiente del ítem 5**. Hasta entonces, quien lo encienda expone la versión y los códigos de error.
- *Revisar:* al implementar la seguridad base, que también cambia el valor por defecto. **Hecho en el ítem 5**: ahora exige `ROLE_ADMIN` y está encendido por defecto.

**La consulta no toma el candado.**
- *Alternativa:* leer con `FOR UPDATE` para ver siempre el estado más reciente.
- *Por qué:* con el candado, consultar el estado esperaría a una snapshot de hasta unos 91 s. Con `READ COMMITTED` se ve lo último confirmado, que es lo que importa.
- *Costo:* durante una corrida el estado muestra la anterior, no la que está en curso.

**`RUNNING` no se escribe ni se informa.** Escribirlo exigiría una transacción extra por corrida y nadie lo vería hasta el *commit*, porque la fila está tomada por el candado. Ya estaba así en la documentación; se confirma.

**El mensaje del error es el fijo por código, y los logs describen la causa solo por su clase.**
- *Alternativa:* registrar el mensaje de la excepción para facilitar el diagnóstico.
- *Por qué:* esos mensajes vienen de la integración y pueden traer hosts, claves o cuerpos de mensajes.
- *Costo:* para ver el mensaje técnico hay que reproducir la falla con un depurador o subir el nivel de log del cliente.

### Límites de lo probado

La falla "la base de datos no responde" se prueba para el endpoint (`503`) con la excepción simulada, no apagando PostgreSQL. La falla de aplicación del snapshot se prueba con un profesional de una categoría inexistente (viola una clave foránea), que es una falla de la base a efectos de la copia. No se probó contra la cátedra real, que solo publica la versión 1. No hay una prueba con un corte real del proceso (es consecuencia de la transacción, como dice la estrategia de sincronización).

## Filtro de disponibilidad (grupo 1, issue 4)

**Qué se hizo.** El filtro "disponibilidad" de la búsqueda, resuelto solo con datos locales y detrás de una interfaz. Rama `feature/filtro-disponibilidad` (commit `797bc3c`; issue #28, PR #24, mergeado a `dev` el 2026-11-01). Todavía no hay endpoint ni búsqueda (backlog 11): este paso deja lista la pieza que la búsqueda va a usar. La búsqueda llegó después (issue #31, PR #32): ver el ítem 11.

**Qué hay en el código.**
- `application.search.AvailabilityFilter`: interfaz con un método, `available(DayOfWeek)`, que devuelve una `Specification<Professional>` (con `null` no se filtra por día). Se devuelve una `Specification` para combinarla con los demás filtros (categoría, nombre, habilitado) en una sola consulta paginada.
- `ScheduleAvailabilityFilter`: la versión básica. Un profesional está disponible si **él y su categoría están habilitados** y tiene **al menos un horario semanal habilitado** (del día pedido, si se informa). Usa `EXISTS`, así que un profesional con varios horarios no se repite.
- `ProfessionalRepository` suma `JpaSpecificationExecutor`.

**Pruebas (10 nuevas, contra PostgreSQL real).** Sin horarios; con todos los horarios deshabilitados; con un horario habilitado y sin día; con día que coincide y que no; horario deshabilitado ese día con otro habilitado otro día; profesional deshabilitado con horarios habilitados; categoría deshabilitada; varios horarios sin repetir al profesional; varios profesionales mezclados; y combinación con el filtro literal de habilitado. Cuatro mutaciones manuales (sin la condición de categoría, sin la del profesional, sin el filtro de día y contando horarios deshabilitados) hicieron fallar los tests esperados.

### Decisiones tomadas

**Se mantiene la decisión F (disponibilidad efectiva) y no la lectura literal del texto del grupo 1.**
- *Alternativa:* "disponible" = tener un horario habilitado, sin mirar si el profesional o la categoría lo están (así se lee el texto del grupo 1 si "independiente del filtro habilitado" significa "no mira la bandera").
- *Por qué:* la cátedra rechaza crear un hold para un profesional deshabilitado (`PROFESSIONAL_DISABLED`). Con la lectura literal, atlas ofrecería profesionales que `cronos-turnos` después no puede reservar. Con F son coherentes. La frase "independiente del filtro habilitado" se respeta en otro sentido: el parámetro `habilitado` (literal) y `disponible` (efectivo) son filtros separados, y la prueba de combinación lo comprueba.
- *Costo:* un profesional habilitado en una categoría deshabilitada figura como habilitado pero no como disponible (ya aceptado en F y a documentar en la API).
- *Revisar:* si la cátedra aclara cómo trata las categorías deshabilitadas o el docente responde la consulta 2 de otra manera.

**Una `Specification` y no una consulta propia ni un predicado en memoria.**
- *Alternativa:* un método de repositorio con la consulta fija, o filtrar en memoria.
- *Por qué:* la búsqueda combina varios filtros opcionales y pagina; una `Specification` se compone con `and` y el catálogo es chico. En memoria rompería la paginación.
- *Costo:* `application` queda atado a Spring Data JPA, que el ADR 0003 ya permite en este paquete.
- *Revisar:* si la búsqueda termina siendo una consulta propia.

**Sin endpoint ni validación de entrada.** El día de la semana llega ya como `DayOfWeek`; convertir el texto de la API y rechazar valores inválidos es trabajo de la búsqueda.

### Límites de lo probado

Se probó contra PostgreSQL real con datos armados en cada test, no con el catálogo real de la cátedra. La disponibilidad no mira turnos libres reales (decisión A). El README no se tocó, porque `feature/documentacion` lo reescribe y todavía no hay una API que describir: se actualiza al implementar la búsqueda.

## Kafka con seguridad configurable (grupo 1, issue 2)

**Qué se hizo.** El consumidor de Kafka puede autenticarse sin tocar código si la cátedra lo exige. Por defecto sigue igual: `PLAINTEXT`, sin ninguna variable extra. El perfil `kafka-sasl` (archivo `application-kafka-sasl.yaml`) activa `SASL_SSL` con mecanismo `PLAIN`, y el usuario y la contraseña van únicamente en `CATEDRA_KAFKA_SASL_JAAS_CONFIG`. Rama `feature/kafka-seguridad-configurable` (commit `0710c75`; issue #27, PR #23, mergeado a `dev` el 2026-11-01).

**Qué cambia en el código.**
- `CatedraKafkaProperties` suma `securityProtocol` (por defecto `PLAINTEXT`) y `saslJaasConfig`, con validación que impide el arranque con un mensaje que dice cuál es el problema: protocolo distinto de `PLAINTEXT` o `SASL_SSL`, `SASL_SSL` sin JAAS, y JAAS con `PLAINTEXT`. El JAAS no aparece en `toString`.
- `CatedraKafkaConfig` arma la configuración del cliente en un método aparte (`consumerConfig`) y le agrega el protocolo y, con `SASL_SSL`, el mecanismo y el JAAS.
- `.env.example` y el README documentan las variables y el perfil.

**Pruebas (11 nuevas, 172 en total).** Sin variables de seguridad la conexión es `PLAINTEXT` y el cliente no recibe configuración SASL; con `SASL_SSL` recibe protocolo, mecanismo y JAAS; el protocolo se normaliza a mayúsculas; los tres arranques inválidos dicen qué variable falla; el JAAS no está en el `toString` ni en el mensaje de un fallo; el archivo real del perfil activa `SASL_SSL` con el JAAS de la variable y, sin el JAAS, no arranca; y el cliente de Kafka, al construirse con `SASL_SSL`, registra `sasl.jaas.config = [hidden]` y no la contraseña. Dos mutaciones manuales (el perfil que no cambia el protocolo y el JAAS que no llega al cliente) hicieron fallar los tests esperados.

### Decisiones tomadas

**Las propiedades son `catedra.kafka.*`, no `spring.kafka.*`.**
- *Alternativa:* seguir el boceto del grupo 1, con `spring.kafka.security.protocol` y `spring.kafka.properties.sasl.*`.
- *Por qué:* el consumidor se arma a mano y la autoconfiguración de Kafka está excluida a propósito (#21), así que esas propiedades no tendrían efecto. Las variables llevan el prefijo `CATEDRA_KAFKA_*` como el resto de la integración, en lugar de `KAFKA_*`.
- *Costo:* se diverge del nombre estándar de Spring; quien conozca `spring.kafka.*` no lo encontrará.
- *Revisar:* si se reemplazara el armado manual por la autoconfiguración.

**Solo `PLAINTEXT` y `SASL_SSL`, mecanismo fijo `PLAIN`.**
- *Alternativa:* aceptar cualquier protocolo y mecanismo de Kafka (`SSL`, `SASL_PLAINTEXT`, `SCRAM`, mTLS).
- *Por qué:* es lo que pidió el grupo 1 y evita configuraciones que nadie probó; una opción más segura que el texto plano sin cifrar (`SASL_PLAINTEXT`) no se ofrece por error.
- *Costo:* si la cátedra pidiera otro mecanismo, hay que cambiar código y no solo configuración.
- *Revisar:* cuando el docente responda la consulta 11.

**Una configuración incoherente impide el arranque.**
- *Alternativa:* ignorar el JAAS si el protocolo es `PLAINTEXT`, o usar `SASL_SSL` sin JAAS y dejar que falle al conectarse.
- *Por qué:* credenciales definidas pero no usadas harían creer que la conexión está autenticada, y un `SASL_SSL` sin JAAS falla recién al conectar, con un mensaje poco claro y ocultado por el reintento en segundo plano del consumidor.
- *Costo:* si se deja el JAAS en el `.env` y se quita el perfil, la aplicación no arranca hasta borrar la variable.

### Límites de lo probado

No se probó contra un Kafka con autenticación, ni contra el Kafka real de la cátedra: hoy no exige credenciales y no hay un servidor con SASL a mano. Lo verificado es la configuración que recibe el cliente y que el secreto no se registra. Un `SASL_SSL` real también requiere que el servidor tenga un certificado que el cliente confíe (el almacén de confianza de la JVM); si la cátedra usara uno propio, habría que agregar la configuración del *truststore*, que no se incluyó. Tampoco se repitió la verificación manual contra la cátedra real, porque la ruta por defecto (`PLAINTEXT`) no cambió y la cubren los tests con un Kafka real en contenedor.

## Ítem 11 · Búsqueda de profesionales

**Qué se hizo.** La búsqueda y el detalle de profesionales sobre la copia local, con los cuatro filtros que pide el enunciado (4.1): categoría, nombre, estado habilitado y disponibilidad. Issue #31 y PR #32 de `atlas-catalogo`, mergeado a `dev` el 2026-11-01. **El issue sigue abierto**: falta proteger los endpoints con JWT (ítem 5), y hasta entonces están apagados por defecto.

**Qué hay en el código.**
- `GET /api/professionals`: filtros opcionales y combinables con "y" (`categoryId`, `name`, `enabled`, `available`, `dayOfWeek`), orden fijo (apellido, nombre, ID) y paginación (`page` desde 0, `size` de 1 a 100, por defecto 20).
- `GET /api/professionals/{id}`: el profesional con su categoría y todos sus horarios semanales, habilitados o no, ordenados por día y hora; devuelve también a un profesional deshabilitado.
- `GET /api/categories`: las categorías por nombre, con filtro opcional por estado. Es de esta tarea y no estaba en el borrador: sin ella la app no sabe qué `categoryId` mandar.
- `application.search`: los casos de uso `SearchProfessionals`, `GetProfessionalDetail` y `ListCategories`, los criterios con su validación (`ProfessionalSearchCriteria`) y las especificaciones de los filtros simples. La disponibilidad reutiliza `AvailabilityFilter` (grupo 1, issue 4).
- `adapters.in.web`: `ProfessionalsController` solo traduce parámetros y JSON; `ProfessionalsExceptionHandler` traduce los errores a `application/problem+json` (`400 BUSQUEDA_INVALIDA`, `404 PROFESIONAL_NO_ENCONTRADO`, `503 CATALOGO_NO_DISPONIBLE`) con mensajes fijos que no repiten el valor recibido.
- `WeeklyScheduleRepository.findByProfessionalId` para el detalle.
- Propiedad `atlas.search.enabled` (`ATLAS_SEARCH_ENDPOINT`; apagada por defecto hasta el ítem 5, encendida desde entonces).

**Pruebas (49 nuevas, 248 en total).** `BusquedaProfesionalesTest` (29, PostgreSQL real): cada filtro y sus combinaciones, comodines de `LIKE`, tildes, orden y desempate, paginación, un profesional con varios horarios sin repetirse, detalle ordenado, profesional deshabilitado y categorías. `ProfessionalSearchCriteriaTest` (7, sin Spring): límites de `size`, `page` y `name`, y `dayOfWeek` sin `available`. `ProfessionalsControllerTest` (11) y `BusquedaApagadaPorDefectoTest` (1): traducción de parámetros, JSON, errores `400`, `404` y `503` sin detalle interno, y que apagada responde `404`. Una regla nueva de ArchUnit comprueba que `application.search` no depende de los puertos hacia la cátedra, de los adaptadores ni de la sincronización.

**Pruebas de mutación manuales.** Se rompieron a propósito cuatro reglas y los tests detectaron las cuatro: no escapar los comodines (1 test falla), ignorar `available=false` (3), quitar el desempate por ID (1) y no ordenar los horarios del detalle (2).

### Decisiones tomadas al implementar

**`available=false` devuelve el complemento de los disponibles.**
- *Alternativa:* aceptar solo `available=true` y ignorar `false`, o rechazarlo.
- *Por qué:* es simétrico y se explica solo; ignorarlo en silencio devolvería todo y parecería un filtro que no filtra. Una prueba comprueba que disponibles y no disponibles suman el total sin solaparse.
- *Costo:* "no disponible" incluye al deshabilitado, al de categoría deshabilitada y al que no tiene horarios habilitados; no distingue el motivo.
- *Revisar:* si la app no lo usa, se puede quitar.

**`dayOfWeek` sin `available` es un `400`, no se ignora.**
- *Alternativa:* que `dayOfWeek` implique `available=true`, o ignorarlo.
- *Por qué:* quien pide un día espera un resultado filtrado por día; ignorarlo devolvería profesionales que no atienden ese día, y que lo implique oculta una decisión de la API.
- *Costo:* un error más que el cliente puede recibir.
- *Revisar:* si la app prefiere que el día implique disponibilidad.

**El nombre se busca en "nombre apellido".**
- *Alternativa:* la decisión E literal: `first_name` o `last_name` por separado.
- *Por qué:* con la versión literal, "Ana Gomez" no encuentra a nadie, y es lo primero que escribe un usuario. Buscar en el nombre completo es un superconjunto: sigue encontrando por nombre o por apellido sueltos.
- *Costo:* "gomez ana" no coincide, porque el orden es nombre y luego apellido. Sigue sin ignorar tildes (decisión E): "nunez" no encuentra a "Núñez", y hay una prueba que lo documenta. Los comodines `%` y `_` se tratan como texto.
- *Revisar:* si hace falta ignorar tildes o tolerar el orden inverso (`unaccent` o trigramas, migraciones aditivas).

**Orden fijo y sin parámetro de orden.** Apellido, nombre e ID. *Alternativa:* `sort` libre. *Por qué:* un nombre de columna que llega del cliente es superficie de error y de ataque, y el catálogo no lo necesita. *Costo:* la app no puede ordenar de otra manera. *Revisar:* si la interfaz lo pide.

**Las vistas de `application.search` son el cuerpo JSON, sin una segunda capa de DTO.**
- *Alternativa:* un `Response` por endpoint, como en el estado de la sincronización.
- *Por qué:* son registros simples sin nada de persistencia, y duplicarlos habría sumado cinco clases sin aportar aislamiento real.
- *Costo:* cambiar una vista cambia el contrato público.
- *Revisar:* al definir el contrato con `cronos-turnos`, que puede necesitar su propia forma.

**Una página propia (`PageResult`) y no la `Page` de Spring Data.** El formato JSON de `Page` no es un contrato pensado para clientes externos; el propio contrato queda estable y mínimo (`content`, `page`, `size`, `totalElements`, `totalPages`).

**Apagada por defecto, como el estado de la sincronización.**
- *Alternativa:* encenderla ya.
- *Por qué:* el enunciado (9) pide proteger los endpoints que no sean públicos, y todavía no existe la seguridad.
- *Costo:* la app KMP no puede usarla hasta el ítem 5.
- *Revisar:* al implementar la seguridad base, que cambia el valor por defecto a encendido. **Hecho en el ítem 5.**

**El detalle de un profesional deshabilitado se devuelve con `enabled: false`**, y solo un ID inexistente da `404`: la copia local nunca borra (M3 del modelo de datos: bajas lógicas, sin borrado físico).

### Cambios al diseño original (ítem 11)

- Los parámetros de la API van en inglés (`categoryId`, `name`, `enabled`, `available`, `dayOfWeek`), como el resto de la API, y no con los nombres del modelo de datos (`categoriaId`, `nombre`, `habilitado`, `disponible`, `diaSemana`).
- El filtro de nombre busca en el nombre completo (decisión E ampliada).
- Se agregó `GET /api/categories`.
- `available` acepta `false` (complemento).

### Límites de lo probado

Se probó contra PostgreSQL real con datos armados en cada test y, además, contra el snapshot real de la cátedra (versión 1: 8 categorías, 24 profesionales y 44 horarios), ver [la evidencia](../evidencias/isla-1-busqueda.md). En esa verificación no se buscó ningún nombre con tilde, así que la regla de tildes se probó solo con datos propios. No se midió el rendimiento: con 24 profesionales no dice nada. No hay prueba de acceso no autenticado, porque la seguridad aún no existe: llega con el ítem 5. La disponibilidad sigue sin mirar turnos libres reales (decisión A).

## Ítem 5 · Seguridad base

**Qué se hizo.** atlas pasa a ser un servidor de recursos: valida los JWT que emite `cronos-turnos` con su clave pública y protege todo lo que expone. Issue #33 y PR #34 de `atlas-catalogo` (mergeado a `dev` el 2026-11-01). La decisión de fondo, cómo se autentican los servicios entre sí, está en el [ADR 0004](../arquitectura/adr/0004-autenticacion-entre-servicios.md): se propaga el JWT del usuario firmado con RS256.

**Qué hay en el código.**
- `config.SecurityConfig`: la cadena de filtros (sin sesión, sin CSRF, sin login por formulario ni autenticación básica), las reglas por ruta, el decodificador del JWT y el CORS.
- `config.SecurityProperties` y `config.JwtPublicKeys`: las propiedades (`atlas.security.*`) y la carga de la clave pública, con validación al arrancar.
- `adapters.in.security.SecurityProblemHandler`: traduce los rechazos a `401 NO_AUTENTICADO` y `403 SIN_PERMISO` como `problem+json` y los registra en el log.
- `scripts/dev-token.sh`: firma tokens de desarrollo con una clave local, para probar sin `cronos-turnos`.
- Reglas de ArchUnit: la seguridad HTTP no depende del dominio, de los casos de uso ni de la web, y `application` no puede importar nada de Spring Security.
- Dependencias: `spring-boot-starter-security` y `spring-boot-starter-oauth2-resource-server`, y el starter de test de seguridad.

**Pruebas (71 nuevas más una regla de ArchUnit; 320 en total).** `SeguridadApiTest` (32, aplicación completa con firmas RS256 reales): sin token en cada ruta, tokens vencidos, aún no válidos, con la firma de otra clave, con el payload adulterado, `HS256` con la clave pública como secreto, otro algoritmo RSA, `alg: none`, sin `exp` o sin `sub`, el token en un lugar distinto del encabezado, roles por ruta, escrituras y rutas desconocidas denegadas, identificadores enviados por el cliente, que el rechazo no revele el motivo y que quede en el log sin el token. `CorsApiTest` (8) y `CorsPorDefectoTest` (2). `SecurityPropertiesTest` (19) y `JwtPublicKeysTest` (10): las propiedades y la carga de la clave. Los tests existentes de estado y de recuperación ahora consultan como administrador, y los de controlador saltean los filtros porque solo prueban la traducción a JSON.

**Pruebas de mutación manuales.** Se rompieron a propósito nueve reglas y los tests detectaron las nueve: no validar el vencimiento (2 tests fallan), abrir el estado a `ROLE_USER` (2), autenticar en vez de denegar el resto (2), no exigir `sub` y `exp` (1), dejar público todo Actuator (2), CORS con credenciales (1), CORS con escritura (2), quitar `WWW-Authenticate` (4) y revelar el rol en el `403` (2).

### Decisiones tomadas al implementar

**La clave pública se carga de un archivo o de una variable, y sin ella no se arranca.**
- *Alternativas:* JWKS desde `cronos-turnos` (`jwk-set-uri`), o arrancar con la API abierta o cerrada sin avisar.
- *Por qué:* con JWKS atlas dependería de que cronos esté levantado para validar tokens, y los servicios deben ser independientes (enunciado 3). Arrancar sin clave dejaría una API sin protección o rechazando todo sin explicación.
- *Costo:* rotar la clave exige reiniciar atlas; y el `.env` de cada desarrollador necesita la variable nueva.
- *Revisar:* si aparece más de un emisor o hay que rotar claves sin reiniciar (JWKS).

**Solo se acepta RS256, con una clave RSA de 2048 bits o más.** Un decodificador que aceptara el algoritmo que declara el token permitiría el ataque de confusión de algoritmo (un `HS256` firmado con la clave pública, que no es secreta). Una clave privada pegada por error, una clave corta o una que no es RSA se rechazan al arrancar, sin repetir su contenido.

**`sub` y `exp` son obligatorios.**
- *Alternativa:* aceptar lo que Spring valida por defecto.
- *Por qué:* un token sin vencimiento es válido para siempre, y el enunciado pide validar la vigencia. Sin `sub` no hay usuario al que atribuir los rechazos.
- *Costo:* un emisor que no ponga alguno de los dos queda rechazado; cronos debe emitirlos.

**Los permisos salen del claim `auth`, sin prefijo, y se aceptan como texto o como lista.**
- *Alternativa:* el `scope` de OAuth2 (con prefijo `SCOPE_`) o un claim propio.
- *Por qué:* es el claim de JHipster, y el enunciado pide un modelo de usuario compatible con el suyo; así sirve si cronos usa JHipster o no. Los roles ya traen `ROLE_`.
- *Costo:* es una propuesta de atlas: cronos todavía no confirmó que emita `sub` y `auth`. El nombre del claim es configurable (`ATLAS_JWT_AUTHORITIES_CLAIM`) para que no sea un cambio de código.
- *Revisar:* cuando exista cronos.

**Se deniega todo lo que no está listado, exista o no la ruta.**
- *Alternativa:* "cualquier otra ruta autenticada".
- *Por qué:* una ruta nueva que se agregue sin pensar en la seguridad quedaría cerrada y no abierta a cualquier usuario, y un `403` en lugar de un `404` no revela qué rutas existen.
- *Costo:* cada endpoint nuevo exige tocar la configuración de seguridad. Es intencional.

**El estado de la sincronización exige `ROLE_ADMIN`; la búsqueda, `ROLE_USER` o `ROLE_ADMIN`.** El estado muestra cuándo y por qué falla la integración con la cátedra, información que un usuario común no necesita. La búsqueda es lo que la app muestra a cualquier usuario autenticado; el administrador también puede usarla.

**Los `401` y `403` no explican el motivo.** No dicen si falló la firma, el vencimiento o el algoritmo, ni qué rol faltaba. *Costo:* quien integra no recibe la pista en la respuesta; la tiene en el log, que registra el método, la ruta y, en un `403`, el usuario, y nunca el token.

**Sin sesión, y CSRF desactivado.** La API usa un token en un encabezado, no cookies: un navegador no lo adjunta solo en una petición de otro sitio, que es lo que CSRF previene. Se aceptan los tokens únicamente en `Authorization: Bearer`, no en un parámetro de la URL, que quedaría en logs y historiales.

**CORS: lista explícita de orígenes, sin comodines y sin credenciales; ninguno por defecto.** La app Android no es un navegador y no necesita CORS; existe para un posible cliente web. Un origen con comodín o con ruta se rechaza al arrancar.

**La búsqueda y el estado pasan a estar encendidos por defecto**, ahora que están protegidos (era lo anunciado en los ítems 11 y 12).

**Una clave pública de prueba versionada, con una excepción en `.gitignore`.** `.gitignore` excluye `*.pem` para que nadie versione una clave por accidente; los tests de contexto necesitan una clave pública, así que se versiona solo `src/test/resources/jwt-test-public.pem` (la privada se descartó) con una excepción para ese único archivo. Los tests de seguridad generan sus propios pares.

### Qué se descubrió al probar

- **Un test pasaba por la razón equivocada.** Los casos de "token vencido" se rechazaban porque el token tenía `exp` anterior a `iat` (Spring lo prohíbe al construir el JWT), no por estar vencido. Se corrigió con tokens coherentes (emitidos una hora antes de vencer) y recién ahí quedaron probados el vencimiento y la tolerancia de 60 segundos.
- **Con CORS sin orígenes, la consulta previa responde `200` vacío**, no `401` ni `403`: el filtro CORS de Spring no reenvía nunca una consulta previa a la cadena. Lo que protege es que no trae ningún `Access-Control-*`, así que el navegador la bloquea; el test afirma eso y no el estado.
- **Un error propio de documentación:** el README decía que con la búsqueda apagada las rutas responderían `403`; en realidad un token válido recibe `404` (la regla lo autoriza y la ruta no existe) y sin token `401`. Se corrigió antes de publicar.
- **Un `.gitignore` útil se interponía:** `*.pem` impedía versionar la clave pública de prueba. Se detectó revisando el diff antes de commitear y se confirmó con un clon limpio, donde los tests de contexto pasan.

### Límites de lo probado

Se probó con la aplicación completa y firmas RS256 reales (los pares los genera el test), y con la aplicación corriendo contra una base temporal y tokens firmados con `scripts/dev-token.sh`: ver [la evidencia](../evidencias/isla-1-seguridad.md). **No se probó contra un token real de `cronos-turnos`**, porque todavía no existe: los claims `sub` y `auth` son una propuesta. No se validan `iss` ni `aud`, no hay revocación ni límite de pedidos, y el canal entre la app y atlas no va cifrado (ver la [seguridad](../arquitectura/seguridad-atlas-catalogo.md)).

## Pendiente

- Cerrar a mano el issue #31 (la búsqueda cumple todos sus criterios desde el PR #34). El #33 queda abierto hasta fijar los claims con `cronos-turnos`.
- Un endpoint de lectura para `cronos-turnos` (horario semanal vigente de un profesional), con su contrato documentado y protegido con este mismo esquema; el Dockerfile y la app dentro del compose; las evidencias (ítem 14) y la documentación final (ítem 13).
- Consultas al docente que condicionan lo que sigue: 1 (autenticación entre servicios, resuelta localmente en el ADR 0004), 2 (alcance de "disponibilidad"), 3 (hexagonal liviana) y 10 (`oldest-available-version`, cerrada localmente el 2026-10-21).
- Confirmar con el docente las consultas 2, 4, 10 y 11, resueltas localmente.

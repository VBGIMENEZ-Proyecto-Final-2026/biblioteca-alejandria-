# Implementación de atlas-catalogo · evolución

**Componente:** atlas-catalogo · **Isla:** 1 · Eolia · **Actualizado:** 2026-10-12

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
| 2026-10-12 | Ítem 13 | Documentación: README ejecutable desde cero, estrategia de sincronización y seguridad | En curso (primera entrega) |

Tests automatizados tras cada paso: 5 (esqueleto) → 19 (persistencia) → 40 (cliente REST) → 58 (sincronización completa) → 86 (cliente Redis) → 133 (sincronización incremental) → 161 (consumidor de Kafka).

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

## Pendiente

- Seguridad base, búsqueda y estado de la sincronización; después, actualizar la documentación (ítem 13) y juntar las evidencias (ítem 14).
- Consultas al docente que condicionan lo que sigue: 2 (alcance de "disponibilidad"), 3 (hexagonal liviana) y 10 (`oldest-available-version`); y D12 (RS256) para la seguridad base.

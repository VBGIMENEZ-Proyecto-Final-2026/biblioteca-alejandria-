# Isla 1 · Eolia

**Objetivo:** dejar `atlas-catalogo` funcionando en una versión básica y por separado: mantiene la réplica local del catálogo de la cátedra (snapshot e incremental) y la expone con búsqueda y filtros sobre datos locales.

**Alcance:** todo lo propio de `atlas-catalogo`: decisiones de diseño, esqueleto, persistencia, integración con la cátedra, sincronización, búsqueda, seguridad básica, pruebas, documentación y evidencias.

**Fuera de alcance:** comunicación y contrato entre `atlas-catalogo` y `cronos-turnos`, registro y login de usuarios finales, turnos y reservas, y `hermes-app`.

**Estado del backlog:** los issues de decisiones de diseño (1 a 3) y el esqueleto del proyecto (4) están hechos (issues #5, #6, #7 y #9 de `atlas-catalogo`). Migraciones y persistencia (6) está hecho (issue #11, PR #13). El cliente REST de la cátedra (7) también está hecho (issue #12, PR #14). La sincronización completa (8) también está hecha (issue #15, PR #16). El cliente Redis (9) también está hecho (issue #17, PR #18). La sincronización incremental (10) también está hecha (issue #19, PR #20). El consumidor de Kafka (15) también está hecho (issue #21, PR #22). El estado de la sincronización (12) está hecho (issue #29, PR #25), con la protección del endpoint pendiente de la seguridad base (5). La búsqueda (11) está hecha salvo la protección con JWT (issue #31, PR #32). La documentación (13) tiene una primera entrega hecha (issue #30, PR #26) y se completa al incorporar la seguridad. Del grupo 1 también están hechos Kafka configurable (issue #27, PR #23) y el filtro de disponibilidad (issue #28, PR #24). El resto de las tareas de la isla se agrega cuando sus decisiones estén justificadas. La numeración es la del backlog completo, por eso el 5 (seguridad base) todavía no aparece.

Formato de cada issue: Componente `atlas`, descripción, criterios, referencias y dependencias (igual que `isla-0-itaca.md`). Las decisiones viven en este repo; los issues solo describen la tarea.

## Issues

### 1. [atlas] Definir el stack tecnológico de atlas-catalogo
Decidir y documentar el stack del servicio (lenguaje, framework, build, base de datos, migraciones, clientes de integración, seguridad y testing), con alternativas, pros y contras de cada decisión.
- [x] ADR 0002 publicado en `alejandria-docs`.
- [x] Herramienta de build definida (Maven o Gradle).
- [x] Algoritmo de firma del JWT definido, o marcado como pendiente de consulta.
- [x] Decisiones propuestas revisadas y su estado actualizado en el ADR.
Estado: hecho (issue #5 de `atlas-catalogo`), publicado en `arquitectura/adr/0002-stack.md`.
Referencias: `arquitectura/adr/0002-stack.md`; enunciado 3, 3.1, 9 y 10.1.
Bloquea: 2, 3 y 4.

### 2. [atlas] Definir la arquitectura interna de atlas-catalogo
Acordar la arquitectura del servicio: opciones evaluadas, reglas de dependencia, estructura de paquetes y lógica de reconciliación del catálogo.
- [x] ADR 0003 publicado en `alejandria-docs`.
- [x] Reglas de dependencia acordadas.
- [x] Algoritmo de reconciliación revisado, casos borde incluidos.
- [x] Regla de ArchUnit definida para el primer ticket de código.
Estado: hecho (issue #6 de `atlas-catalogo`), publicado en `arquitectura/adr/0003-arquitectura.md`.
Referencias: `arquitectura/adr/0003-arquitectura.md`; enunciado 4.1 y 6.
Depende de: 1. Bloquea: 3 y 4.

### 3. [atlas] Diseñar el modelo de datos inicial de atlas-catalogo
Definir el boceto del modelo de datos: tablas del catálogo, tablas de control del sync, claves, índices y decisiones abiertas.
- [x] Documento de modelo de datos publicado en `alejandria-docs`.
- [x] Decisiones abiertas cerradas o diferidas explícitamente.
- [x] Alcance de "disponibilidad" definido, o diferido explícitamente.
Estado: hecho (issue #7 de `atlas-catalogo`), publicado en `modelo-datos/atlas-catalogo.md`.
Referencias: `modelo-datos/atlas-catalogo.md`; enunciado 4.1, 6 y 7.
Depende de: 1 (PostgreSQL, Flyway) y 2. Bloquea: 4 y 6.

### 4. [atlas] Esqueleto del proyecto atlas-catalogo
Crear el proyecto Spring Boot con el stack y la estructura de paquetes decididos en los ADR 0002 y 0003: que compile, arranque, se conecte a la PostgreSQL del compose con Flyway configurado y responda su endpoint de salud. Sin tablas, sin lógica de sincronización y sin seguridad.
- [x] Spring Boot 4.1, Java 25 y Gradle (Kotlin DSL), con wrapper 9.1.0 o superior.
- [x] Dependencias mínimas (Web, Actuator, Data JPA, Flyway, PostgreSQL y las de test, con ArchUnit 1.4.1 o superior); Kafka, Redis y Security en sus propios issues.
- [x] Estructura de paquetes del ADR 0003.
- [x] Conexión a la base solo por variables de entorno (`ATLAS_DB_*`); las `CATEDRA_*` las declara cada issue de cliente.
- [x] Flyway configurado; la aplicación arranca sin migraciones.
- [x] `/actuator/health` responde y es el único endpoint de Actuator expuesto.
- [x] Test de contexto con Testcontainers sobre Java 25 (si falla el tooling, se documenta y se evalúa Java 21).
- [x] Las tres reglas de ArchUnit del ADR 0003 activas y pasando con paquetes vacíos.
- [x] `./gradlew build` pasa.
- [x] README con requisitos, cómo levantar la base, correr la aplicación y ejecutar los tests.
Estado: hecho (issue #9 y PR #10 de `atlas-catalogo`). El paquete base es `ar.edu.um.atlas` (ADR 0003).
Referencias: ADR 0001, 0002 y 0003; `modelo-datos/atlas-catalogo.md` (decisión L); enunciado 3 y 3.1.
Depende de: 1, 2 y 3 (hechos) y de Isla 0. Bloquea: 6, seguridad base, cliente REST y cliente Redis (estos tres, por abrir).

### 6. [atlas] Migraciones y persistencia del catálogo
Crear con Flyway el esquema del modelo de datos (categorías, profesionales, horarios semanales y la fila de control del sync) y las entidades JPA y repositorios que lo usan, con tests de persistencia contra PostgreSQL real. Solo guarda y lee: sin lógica de sincronización.
- [x] Migración `V1` que crea desde una base vacía las cuatro tablas con sus tipos, claves, claves foráneas e índices, y la fila inicial de `sync_state`.
- [x] Solo las restricciones del modelo (`NOT NULL`, claves foráneas, dominio de `day_of_week` y `id = 1` en `sync_state`).
- [x] Entidades JPA en `domain` (sin Spring) y repositorios de Spring Data en `application.repository`.
- [x] *Upsert* idempotente por ID en los repositorios del catálogo y lectura de `sync_state` con candado (`SELECT ... FOR UPDATE`).
- [x] Hibernate valida las entidades contra el esquema.
- [x] Tests de persistencia contra PostgreSQL real con Testcontainers (no H2).
- [x] Reglas de ArchUnit de `domain` y `application` activas sin `allowEmptyShould(true)`.
- [x] `./gradlew build` pasa y el README está actualizado.
Estado: hecho (issue #11 y PR #13 de `atlas-catalogo`, mergeado a `dev` el 2026-10-11).
Referencias: `modelo-datos/atlas-catalogo.md`; ADR 0002 (D5 a D7 y D13) y ADR 0003; enunciado 3 y 4.1.
Depende de: 3 y 4 (hechos). Bloquea: sincronización completa, sincronización incremental, búsqueda y estado (issues por abrir).

### 7. [atlas] Cliente REST de la cátedra: snapshot del catálogo
Pedir a la cátedra el snapshot completo con el JWT técnico, con timeouts definidos, reintentos acotados y errores traducidos. No incluye login: el `id_token` ya es el JWT técnico de un año (referencia de integración 4 y ADR 0001) y renovarlo con `/api/authenticate` exigiría usuario y contraseña que no forman parte de las variables `CATEDRA_*`.
- [x] Puerto `CatalogSnapshotSource` con un snapshot propio (versión y entidades del dominio) y adaptador `RestClient` declarativo en `adapters.out.rest`.
- [x] `CATEDRA_REST_BASE_URL` y `CATEDRA_ID_TOKEN` declaradas y validadas al arrancar.
- [x] Interpreta instantes UTC, horas `HH:mm` y `HH:mm:ss` y campos desconocidos.
- [x] Timeouts configurables y reintentos acotados solo ante fallas transitorias.
- [x] Errores traducidos por `status` y `code`; el token nunca se escribe en logs.
- [x] Tests con respuestas simuladas, incluyendo errores, y una verificación manual contra la cátedra real.
Estado: hecho (issue #12 y PR #14 de `atlas-catalogo`, mergeado a `dev` el 2026-10-11). Detalle de las decisiones en `implementacion/atlas-catalogo.md`.
Referencias: `catedra/INTEGRATION_REFERENCE-v2.md` (4, 6, 7, 13 y 18.4); ADR 0001, 0002 (D10 y D11) y 0003; enunciado 4.1, 6 y 9.
Depende de: 4 (hecho). Bloquea: 8.

### 8. [atlas] Sincronización completa: aplicar el snapshot
Inicializar o reconstruir la copia local desde el snapshot de forma transaccional: la corrida toma el candado de `sync_state`, pide el snapshot y lo aplica en una sola transacción, deshabilitando lo que ya no viene, y registra como versión local exactamente la del snapshot. Con base vacía se ejecuta sola al arrancar. Decidir cuándo corresponde un snapshot (planificador, Redis, Kafka) queda para la sincronización incremental.
- [x] Caso de uso `ApplySnapshot` con candado: dos corridas en paralelo se serializan.
- [x] Con base vacía, al arrancar se hace el snapshot inicial; si la cátedra falla, la aplicación arranca y registra el error.
- [x] El snapshot se aplica en una sola unidad de trabajo: si falla, queda el estado anterior y `sync_state` registra el error.
- [x] La versión local registrada coincide con los datos almacenados; lo que no viene en el snapshot se deshabilita, nunca se borra.
- [x] Repetir el snapshot no duplica datos.
- [x] Tests de integración contra PostgreSQL real: inicialización, reconstrucción, repetición, falla a mitad de camino y paralelismo.
Estado: hecho (issue #15 y PR #16 de `atlas-catalogo`, mergeado a `dev` el 2026-10-12). Detalle de las decisiones en `implementacion/atlas-catalogo.md`.
Referencias: enunciado 6.1 y 8; ADR 0003 (aplicación del snapshot, bajas y exclusión mutua); modelo de datos; `catedra/INTEGRATION_REFERENCE-v2.md` (7 y 18).
Depende de: 6 y 7 (hechos). Bloquea: sincronización incremental, búsqueda y estado (issues por abrir).

### 9. [atlas] Cliente Redis de la cátedra: metadata y cambios incrementales
Leer en Redis lo que la cátedra publica para la sincronización incremental: la metadata de versiones, los cambios de cada versión y el estado vigente de las entidades afectadas, detrás del puerto `CatalogChangeSource`. Solo lee y entrega: decidir qué hacer con esos datos (planificador, aplicación incremental, escalada a snapshot) queda para la sincronización incremental, y el cliente no interpreta `oldest-available-version`.
- [x] Puerto `CatalogChangeSource` con modelo propio: metadata, cambios de una versión y entidades vigentes por ID (las que faltan se omiten).
- [x] Adaptador con `StringRedisTemplate`, solo lecturas puntuales; nunca toca `catedra:holds:*` ni el namespace privado.
- [x] Configuración validada al arrancar, con la contraseña fuera de logs y mensajes; reintentos acotados solo ante fallas transitorias.
- [x] Errores traducidos a las excepciones del puerto (propuesto: las mismas tres del cliente REST).
- [x] Tests contra Redis real (Testcontainers), incluido un usuario ACL equivalente al de la cátedra, y una verificación manual contra el Redis real.
Estado: hecho (issue #17 y PR #18 de `atlas-catalogo`, mergeado a `dev` el 2026-10-12). Detalle de las decisiones en `implementacion/atlas-catalogo.md` y verificación en `evidencias/isla-1-cliente-redis.md`. Detalle de las decisiones en `implementacion/atlas-catalogo.md` y verificación en `evidencias/isla-1-cliente-redis.md`.
Referencias: `catedra/INTEGRATION_REFERENCE-v2.md` (14, 18.2 y 18.4); ADR 0002 (D9, D11 y D13) y 0003; enunciado 6.2 y 8.
Depende de: 4 (hecho). No depende de 8. Bloquea: sincronización incremental (issue por abrir).

### 10. [atlas] Sincronización incremental: planificador y aplicación de cambios
Mantener la copia local al día cuando la cátedra publica una versión nueva: un caso de uso compara la versión local con la metadata de Redis, decide con una función pura entre `Noop`, `Incremental` y `Snapshot`, y lo ejecuta. Los disparadores son el arranque y una verificación periódica de la versión, que también recupera las notificaciones perdidas. El consumidor de Kafka es otro disparador del mismo caso de uso y va aparte (ítem 15).
- [x] Planificador como función pura, con una prueba por cada fila de la tabla del ADR 0003.
- [x] Cada versión se aplica en orden en su propia transacción, con candado, y la versión local avanza solo si termina bien.
- [x] Repetir una versión o una corrida no altera el resultado (idempotencia).
- [x] Discontinuidad, `changes` ausente, versión fuera de ventana o estado no verificable disparan el snapshot y la copia queda consistente; una falla transitoria no escala a snapshot y deja el error.
- [x] Reintentos acotados y sin ciclos infinitos.
- [x] Tests de idempotencia, discontinuidad y recuperación contra PostgreSQL real.
Estado: hecho (issue #19 y PR #20 de `atlas-catalogo`, mergeado a `dev` el 2026-10-12). Detalle de las decisiones en `implementacion/atlas-catalogo.md` y verificación en `evidencias/isla-1-sync-incremental.md`. La fila `local < oldest` del planificador depende de la consulta 10 al docente.
Referencias: enunciado 6.2 y 8; ADR 0003; `catedra/INTEGRATION_REFERENCE-v2.md` (14, 16 y 18).
Depende de: 8 y 9 (hechos). Bloquea: Kafka (15), estado y manejo de errores (12) y evidencias (14).

### 15. [atlas] Consumidor de Kafka: aviso de versión nueva (`CatalogUpdated`)
Consumir `CatalogUpdated` del topic de la cátedra como disparador del mismo caso de uso de sincronización (`SyncCatalog`): el evento avisa y la decisión sale de comparar versiones, nunca de `newVersion`. Confirmar el *offset* solo después de persistir y no bloquearse ante duplicados, eventos fuera de orden o mensajes corruptos.
- [x] Un evento nuevo dispara la sincronización y el *offset* se confirma después de persistir.
- [x] Un evento duplicado o fuera de orden termina en `Noop` y se confirma.
- [x] Un mensaje corrupto no bloquea el consumidor.
- [x] Una falla de la corrida se reintenta de forma acotada y, al agotarla, se descarta con el *offset* confirmado (la verificación periódica cubre el hueco).
- [x] La aplicación arranca aunque Kafka no responda.
- [x] Tests con Kafka real (Testcontainers).
Estado: hecho (issue #21 y PR #22 de `atlas-catalogo`, mergeado a `dev` el 2026-10-12). Detalle de las decisiones en `implementacion/atlas-catalogo.md` y verificación en `evidencias/isla-1-consumidor-kafka.md`.
Referencias: `catedra/INTEGRATION_REFERENCE-v2.md` (15 y 16); ADR 0002 (D8) y 0003; enunciado 6.2 y 8.
Depende de: 10 (hecho). Bloquea: evidencias (14).

### 11. [atlas] API de búsqueda y filtros sobre datos locales
Exponer la búsqueda de profesionales y su detalle con horarios semanales, resuelta solo con la copia local, con filtros por categoría, nombre, estado habilitado y disponibilidad. Versión básica: la disponibilidad se limita a los horarios semanales locales (decisiones A y F del modelo de datos); el cruce con ocupaciones queda para `cronos-turnos`.
- [x] Filtros por categoría, nombre y habilitado, combinables.
- [x] Filtro de disponibilidad funcionando con datos locales.
- [x] Ninguna búsqueda consulta a la cátedra (regla de ArchUnit que impide la dependencia).
- [x] Paginación con orden fijo.
- [ ] Endpoints protegidos con JWT: **pendiente del ítem 5**; hasta entonces están apagados por defecto (`ATLAS_SEARCH_ENDPOINT`).
- [x] Tests de filtros y de errores; el de acceso no autenticado llega con la seguridad.
Estado: hecho salvo la protección (issue #31 y PR #32 de `atlas-catalogo`, mergeado a `dev` el 2026-11-01); el issue queda abierto hasta el ítem 5. Detalle de las decisiones en `implementacion/atlas-catalogo.md` y verificación contra la cátedra real en `evidencias/isla-1-busqueda.md`.
Referencias: enunciado 4.1, 5.7 y 6.2; `modelo-datos/atlas-catalogo.md` (sección "Búsqueda", decisiones A, E y F).
Depende de: 6, 8 y 10 (hechos) y 5 (solo para proteger los endpoints). Bloquea: evidencias (14).

### 12. [atlas] Estado y manejo de errores de la sincronización
Informar el estado de la sincronización (versión local, último resultado, error vigente) y traducir los errores de integración a respuestas claras, sin dejar el estado local inconsistente.
- [x] Endpoint que informa versión local, estado y último error (`GET /api/sync/status`). **Protegido: pendiente del ítem 5**; hasta entonces está apagado por defecto.
- [x] Caída de la cátedra, de Redis o de Kafka no corrompe la copia local y la app se recupera sola.
- [x] Cada falla queda registrada sin exponer secretos.
- [x] Tests de falla y recuperación.
Estado: hecho (issue #29 y PR #25 de `atlas-catalogo`, mergeado a `dev` el 2026-11-01); queda abierto el criterio de protección hasta la seguridad base (5). Detalle de las decisiones en `implementacion/atlas-catalogo.md` y contrato y mapa de fallas en `arquitectura/sincronizacion-atlas-catalogo.md`.
Referencias: enunciado 4.1 y 8; referencia de integración 13 y 18.
Depende de: 8, 9, 10 y 15 (hechos) y 5 (solo para proteger el endpoint).

### 13. [atlas] Documentación de atlas-catalogo
README ejecutable y documentación de lo propio del servicio: modelo de datos, estrategia de sincronización completa e incremental, idempotencia y recuperación, decisiones de seguridad e instrucciones para correr las pruebas.
- [x] README con requisitos, configuración y pasos reproducibles desde cero (verificado en un clon limpio).
- [x] Estrategia de sync, idempotencia y recuperación documentadas en `alejandria-docs`.
- [x] Decisiones y limitaciones de seguridad documentadas (lo que existe y lo que falta).
- [x] La documentación coincide con lo implementado hoy.
- [ ] Se revisa y se actualiza al implementar la seguridad base (5) y la búsqueda (11).
Estado: primera entrega hecha (issue #30 y PR #26 de `atlas-catalogo`, mergeado a `dev` el 2026-11-01); queda abierto hasta incorporar 5 y 11. Detalle en `implementacion/atlas-catalogo.md`; verificación en `evidencias/isla-1-readme-desde-cero.md`.
Referencias: enunciado 12; `arquitectura/sincronizacion-atlas-catalogo.md` y `arquitectura/seguridad-atlas-catalogo.md`.
Depende de: 10 (hecho) y 11 y 5 (por hacer, para la entrega final). Bloquea: evidencias (14).

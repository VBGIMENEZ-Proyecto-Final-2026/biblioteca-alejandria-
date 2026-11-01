# ADR 0002 · Stack tecnológico de atlas-catalogo

**Estado:** aceptado, con D12 implementado en atlas (2026-11-01) y pendiente de confirmar con el docente · **Componente:** atlas-catalogo · **Contexto:** Isla 1, issue #5

## Contexto

`atlas-catalogo` es el servicio backend de catálogo y sincronización: mantiene la copia local de categorías, profesionales y horarios semanales, la sincroniza contra el servicio central (REST + Redis + Kafka) y la expone para búsqueda y para consulta de agenda por `cronos-turnos`.

Restricciones del enunciado que condicionan el stack:

- Java + Spring Boot (sección 3.1). JHipster es opcional.
- Base de datos **servidor** (nada de H2/SQLite), con migraciones propias y esquema/usuario separados del otro servicio (sección 3).
- Infraestructura local levantable con Docker Compose.
- JWT validado en endpoints protegidos y en la comunicación entre servicios; CORS; validación de datos en cada borde (sección 9).
- Integración con la cátedra por REST, Redis y Kafka. La entrega Kafka es *at-least-once*.
- Pruebas automatizadas de sincronización, autorización e idempotencia (sección 10.1).
- Secretos externalizados. **El repositorio es público**: nada de credenciales, hosts ni tokens en el código ni en la documentación.

## Estado de las decisiones

| # | Decisión | Estado |
|---|---|---|
| D1 | Java 25 | Decidido |
| D2 | Spring Boot 4.1 | Decidido |
| D3 | Sin JHipster en este repo | Decidido |
| D4 | Build: Gradle (Kotlin DSL) | Decidido |
| D5 | PostgreSQL | Decidido |
| D6 | Flyway para migraciones | Decidido |
| D7 | Spring Data JPA | Decidido |
| D8 | `spring-kafka` | Decidido |
| D9 | Redis con Lettuce (Spring Data Redis) | Decidido |
| D10 | Cliente REST: `RestClient` + interfaz declarativa | Decidido |
| D11 | Reintentos: resiliencia nativa de Framework 7 | Decidido |
| D12 | JWT RS256 con clave pública | Decidido por el alumno e implementado (2026-11-01, [ADR 0004](0004-autenticacion-entre-servicios.md)); se confirma con el docente |
| D13 | Testing: JUnit 5 + Testcontainers + ArchUnit | Decidido |

## Decisiones, alternativas, pros y contras

### D1 — Java 25 (alternativa: Java 21) · Decidido
- **Pros:** es LTS vigente; Spring Boot 3.5 y 4.x declaran compatibilidad hasta Java 25; sirve para practicar records, sealed interfaces y pattern matching (el resultado del planificador de sync es un buen caso).
- **Contras:** tooling más reciente (Mockito/ByteBuddy, Lombok, JaCoCo y ArchUnit deben soportar class files de Java 25); menos material de consulta.
- **Mitigación:** el primer ticket de código incluye un test de arranque con Testcontainers sobre Java 25. Si algo falla, se baja a 21: no se usa ninguna feature exclusiva de 25.

### D2 — Spring Boot 4.1 (alternativa: 3.5) · Decidido
- **Pros:** la línea 3.5 terminó su soporte OSS el 30/06/2026 y 4.1 es la línea vigente; Framework 7 trae retry y límites de concurrencia integrados (sin dependencia extra); clientes HTTP declarativos con `@ImportHttpServices`.
- **Contras:** Jackson 3 (paquete `tools.jackson`); starters modulares (p. ej. Flyway requiere `spring-boot-starter-flyway`) y test slices en paquetes nuevos; Spring Security 7; menos material y tutoriales.
- **Nota:** Boot 4.0 tiene soporte OSS solo hasta el 31/12/2026; no se considera.
- **Verificado el 07/10/2026:** la línea 3.5 terminó su soporte OSS el 30/06/2026; la versión vigente es 4.1.1 (soporte OSS hasta el 31/07/2027). Boot 4.1 requiere Java 17 como mínimo y es compatible hasta Java 26, con Spring Framework 7.0.9 o superior.
- **Riesgo a cubrir con tests:** el contrato exige ignorar campos desconocidos y aceptar `HH:mm` y `HH:mm:ss`. No se asume el comportamiento por defecto de Jackson 3: se testea.

### D3 — Sin JHipster en atlas-catalogo · Decidido
- **Pros:** no hay usuarios que gestionar; el CRUD generado sería de escritura sobre una réplica de solo lectura; sin código generado que haya que borrar; control total de la estructura.
- **Contras:** más setup manual; las convenciones de la cátedra (`problem+json`) se implementan a mano con `ProblemDetail`.
- **Alcance:** esta decisión es solo para este repo. La decisión para `cronos-turnos` se toma aparte.

### D4 — Build: Gradle (Kotlin DSL) (alternativa: Maven) · Decidido
- **Maven — pros:** declarativo y muy difundido en el ecosistema Spring; simple de razonar. **Contras:** XML verboso.
- **Gradle — pros:** más conciso; `hermes-app` (KMP) ya usa Gradle, así que sería una sola herramienta en los tres repos; JHipster también lo soporta. **Contras:** curva del DSL.
- **Requisitos verificados el 07/10/2026:** Spring Boot 4.1 soporta Gradle 8.14+ y 9.x, y Maven 3.6.3+. Gradle 9.1.0 es la primera versión con soporte de Java 25 (para ejecutarse y para toolchains).
- **Decisión:** Gradle (Kotlin DSL), por uniformidad con `hermes-app`: una sola herramienta de build en los tres repos. El wrapper se fija en 9.1.0 o superior por Java 25.
- **Costo aceptado:** curva del DSL y errores a veces crípticos; menos tutoriales de Spring que con Maven.

### D5 — PostgreSQL (alternativas: MySQL/MariaDB) · Decidido
- **Pros:** ya se maneja; `SELECT ... FOR UPDATE` y bloqueos para exclusión mutua del sync; índices parciales; imagen oficial estable; Testcontainers lo soporta muy bien.
- **Contras:** consume más memoria que un motor liviano al correr dos instancias (una por repo) en local; requiere un contenedor de servidor (como cualquier motor servidor); el mantenimiento (VACUUM, bloat) no importa a esta escala pero existe.
- **Nota:** la consigna permite el mismo producto en ambos servicios si hay esquemas/usuarios separados. Se plantea una instancia de Postgres propia por repo.

### D6 — Flyway (alternativa: Liquibase) · Decidido
- **Pros:** SQL plano, fácil de leer y defender; menos conceptos.
- **Contras:** si `cronos-turnos` usa JHipster, allí se usaría Liquibase (dos herramientas de migración distintas). Si resulta molesto, se reevalúa.
- **Consulta 4 (expectativa de la cátedra sobre Flyway o Liquibase):** cerrada localmente el 2026-10-21, se mantiene Flyway; se confirma con el docente. Se aplicó en el issue #11: Flyway es el dueño del esquema (`V1__esquema_catalogo.sql`), Hibernate solo valida (`ddl-auto=validate`) y una migración ya aplicada no se edita (los cambios van en una versión nueva).

### D7 — Spring Data JPA (alternativas: JdbcClient/JdbcTemplate, jOOQ) · Decidido
- **Pros:** rapidez de desarrollo; Specifications para los filtros combinables; el catálogo es chico.
- **Contras:** overhead y riesgo de N+1 (se controla con consultas explícitas); el reemplazo masivo del snapshot requiere batch.

### D8 — spring-kafka (alternativa: Spring Cloud Stream) · Decidido
- **Pros:** control directo del ack manual, el manejo de errores y el deserializador (necesario para no bloquear el consumer ante un mensaje corrupto).
- **Contras:** más configuración explícita.

### D9 — Redis con Lettuce vía Spring Data Redis (alternativas: Jedis, Redisson) · Decidido
- **Pros:** es el cliente por defecto; soporta ACL con usuario y contraseña; alcanza para lecturas de Hashes y Strings.
- **Contras:** arrastra Netty como dependencia transitiva; los serializadores de `RedisTemplate` hay que configurarlos explícitamente (se usa `StringRedisTemplate`); los permisos ACL solo se validan de verdad contra un Redis real (Testcontainers). Redisson sería excesivo (no se necesitan locks distribuidos en Redis).

### D10 — Cliente REST: `RestClient` + interfaz declarativa (alternativas: WebClient, OpenFeign) · Decidido
- **Pros:** sincrónico y simple; coherente con el modelo bloqueante del servicio; sin dependencias extra.
- **Contras:** WebClient/reactivo sería innecesario; OpenFeign suma una dependencia sin necesidad.

### D11 — Reintentos: resiliencia de Framework 7 (alternativas: Resilience4j, manual) · Decidido
- **Pros:** integrada; backoff y máximo de intentos declarativos.
- **Contras:** API nueva; verificar contra la documentación al implementarla.
- **Regla:** reintentos acotados y solo sobre lecturas idempotentes (snapshot REST, lecturas Redis). Nunca ciclos ilimitados.

### D12 — Seguridad: JWT RS256 con clave pública (alternativa: HS256 con secreto compartido) · Decidido e implementado, a confirmar con el docente
- **Pros RS256:** atlas solo conoce la clave pública y no puede forjar tokens; sin dependencia de runtime hacia el emisor.
- **Contras RS256:** hay que gestionar el par de claves y publicar la pública.
- **HS256 — pros:** trivial de configurar. **Contras:** el secreto compartido permite forjar tokens desde cualquiera de los dos servicios.
- **Riesgo:** quien emite los tokens es `cronos-turnos`. Si allí se usa JHipster, su configuración por defecto firma con secreto compartido (HS512) y habría que personalizarla para RS256. Si no fuera viable, se registra un ADR nuevo que reemplace a este y atlas pasa a validar con secreto compartido, documentando la limitación.
- Relacionada con la autenticación entre servicios (JWT propagado vs técnico): consulta 1 de las consultas al docente. **Resuelta en el [ADR 0004](0004-autenticacion-entre-servicios.md)**: se propaga el JWT del usuario firmado con RS256, y atlas lo valida con la clave pública (ítem 5, implementado el 2026-11-01).

### D13 — Testing: JUnit 5, AssertJ, Mockito, Testcontainers (Postgres, Kafka, Redis), `MockRestServiceServer`/WireMock para la cátedra, ArchUnit · Decidido
- **Pros:** las queries y las transacciones del sync se prueban contra motores reales; ArchUnit verifica las reglas de arquitectura (ver ADR 0003).
- **Contras:** los tests de integración son más lentos; requieren Docker disponible.

## Infraestructura y configuración

- Docker Compose por repo, con su propia instancia de PostgreSQL (base y usuario propios). La conexión desde `cronos-turnos` (red compartida o puertos publicados) queda por definir.
- Configuración solo por variables de entorno. Se versiona un `.env.example` con placeholders; `.env` va en `.gitignore` desde el primer commit.

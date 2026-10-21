# ADR 0003 · Arquitectura interna de atlas-catalogo

**Estado:** aceptado, con los compromisos (consulta 3) pendientes de confirmar con el docente; la fila 5 del planificador (consulta 10) está cerrada localmente (2026-10-21) y se confirma con el docente · **Componente:** atlas-catalogo · **Contexto:** Isla 1, issue #6

## Contexto

`atlas-catalogo` mantiene una copia local del catálogo de la cátedra (categorías, profesionales y horarios semanales), la sincroniza por snapshot REST y por cambios incrementales (Kafka avisa, Redis entrega) y la expone para búsqueda. Stack definido en el [ADR 0002](0002-stack.md).

El enunciado (sección 10) deja la arquitectura interna a cargo del alumno. Lo que la arquitectura debe garantizar:

| # | Requisito | Fuente |
|---|---|---|
| R1 | La decisión de sincronizar (`Noop`, `Incremental` o `Snapshot`) se prueba sin Kafka, Redis, REST ni base de datos. | Enunciado 6.2 |
| R2 | El snapshot y cada versión incremental se aplican en una sola unidad de trabajo; la versión local avanza solo si terminó bien. | Enunciado 6.1 y 6.2; referencia 14.5 |
| R3 | Procesar de nuevo el mismo evento o la misma versión no altera el resultado. | Enunciado 6.2; referencia 18.1 |
| R4 | Las búsquedas usan solo datos locales; nunca consultan a la cátedra. | Enunciado 4.1 y 6.2 |
| R5 | Las fallas de REST, Redis y Kafka se traducen a estados claros, con reintentos acotados. | Enunciado 8; referencia 18.4 |
| R6 | La estructura es defendible y no está sobredimensionada para tres entidades sin comportamiento propio. | ADR 0002 |
| R7 | Las reglas de dependencia se verifican automáticamente (ArchUnit). | Enunciado 10.1; issue #6 |

## Opciones evaluadas

### A — Capas clásicas (`web`, `service`, `repository`)
- **Pros:** la más simple y conocida; poco código; el tooling de Spring la asume.
- **Contras:** la lógica de sync queda mezclada con los clientes de la cátedra, salvo que se disciplinen las dependencias a mano (R1); no hay una frontera explícita para REST y Redis; las reglas de ArchUnit se limitan a capas.

### B — Hexagonal liviana (puertos solo donde hay IO externo)
- **Pros:** la decisión del sync es una función pura (R1); REST y Redis quedan detrás de puertos y se simulan en los tests; el resto usa Spring Data directamente (R6); las dependencias se verifican con reglas simples (R7).
- **Contras:** dos compromisos explícitos, descritos más abajo; hay que mantener la disciplina de paquetes.

### C — Hexagonal completa (puerto de persistencia y modelo de dominio separado de JPA)
- **Pros:** dominio totalmente aislado de la infraestructura; la persistencia es intercambiable.
- **Contras:** mapeo duplicado (entidad JPA y modelo de dominio) para tres entidades sin comportamiento (R6); la unidad de trabajo del sync atraviesa puertos de persistencia, lo que complica la transacción (R2); el beneficio de intercambiar la base no aplica, porque ya está decidido PostgreSQL con Testcontainers.

### D — Paquetes por funcionalidad (`catalogo`, `sync`, `busqueda`), con capas dentro de cada uno
- **Pros:** cohesión por tema; fácil de navegar.
- **Contras:** el catálogo y el sync comparten las mismas entidades, por lo que los límites entre módulos serían artificiales; no resuelve por sí sola el aislamiento de la decisión del sync (R1); las reglas de ArchUnit entre módulos son más difíciles de definir (R7).

### Comparación

| | A · Capas | B · Hexagonal liviana | C · Hexagonal completa | D · Por funcionalidad |
|---|---|---|---|---|
| R1 Decisión de sync pura | Parcial | Sí | Sí | Parcial |
| R2 Unidad de trabajo simple | Sí | Sí | Parcial | Sí |
| R3 Idempotencia | Sí | Sí | Sí | Sí |
| R4 Búsqueda solo local | Sí | Sí | Sí | Sí |
| R5 Fallas de integración | Parcial | Sí | Sí | Parcial |
| R6 Proporcionada | Sí | Sí | No | Parcial |
| R7 Verificable con ArchUnit | Parcial | Sí | Sí | Parcial |

## Decisión

**Opción B: hexagonal liviana** · Decidido. Es la única que cumple R1 a R7 sin pagar el costo de la opción C. Los dos compromisos y la fila 5 del planificador se confirman con el docente (consultas 3 y 10).

### Compromisos explícitos
1. **Las entidades JPA son el modelo de dominio.** Evita el mapeo duplicado; el dominio puede usar las anotaciones `jakarta.persistence`, nada más.
2. **No hay puerto de persistencia.** Los casos de uso usan repositorios de Spring Data directamente. A cambio, la persistencia se prueba siempre contra PostgreSQL real (Testcontainers), nunca con simulaciones.

Estos dos compromisos se consultan al docente (consulta 3).

### Estructura de paquetes

```text
<paquete.base>
├── domain                       entidades JPA, SyncPlan, reglas puras (sin Spring)
├── application
│   ├── port.out                 CatalogSnapshotSource (REST), CatalogChangeSource (Redis)
│   ├── repository               repositorios de Spring Data
│   └── (casos de uso)           SyncCatalog, ApplySnapshot, ApplyChanges, SearchProfessionals, SyncStatus
├── adapters
│   ├── in
│   │   ├── web                  controladores REST, DTO, ProblemDetail
│   │   └── kafka                consumidor de CatalogUpdated
│   └── out
│       ├── rest                 cliente de la cátedra (snapshot; sin login técnico, ver implementación)
│       └── redis                cliente de la cátedra (metadata, versiones y entidades)
└── config                       seguridad, beans, propiedades CATEDRA_*
```

`<paquete.base>` es `ar.edu.um.atlas` (`group` de Gradle `ar.edu.um`): dominio invertido de la institución, sin otro requisito del enunciado. Es una convención propia; renombrarlo es mecánico mientras no haya código de negocio.

No hay puertos de entrada: los adaptadores `in` llaman a los casos de uso directamente.

### Reglas de dependencia

| Paquete | Puede depender de | No puede depender de |
|---|---|---|
| `domain` | JDK y `jakarta.persistence` | `application`, `adapters`, `config`, Spring |
| `application` | `domain`, Spring Data, `@Transactional` | `adapters`, `config`, Spring Web, Kafka, clientes REST/Redis |
| `adapters.in` | `application`, `domain` | `adapters.out` |
| `adapters.out` | `application.port.out`, `domain` | `adapters.in` |
| `config` | todos | — |

### Regla de ArchUnit para el primer ticket de código

```java
@AnalyzeClasses(packages = "<paquete.base>", importOptions = ImportOption.DoNotIncludeTests.class)
class ArquitecturaTest {

    @ArchTest
    static final ArchRule dominioSinFrameworks = noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "..application..", "..adapters..", "..config..", "org.springframework..");

    @ArchTest
    static final ArchRule aplicacionSinAdaptadores = noClasses().that().resideInAPackage("..application..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "..adapters..", "..config..", "org.springframework.web..",
            "org.springframework.kafka..", "org.springframework.data.redis..");

    @ArchTest
    static final ArchRule adaptadoresAislados = slices().matching("..adapters.(*)..")
        .should().notDependOnEachOther();
}
```

(`org.springframework.web..` incluye `RestClient`. Los nombres de paquete se confirman al crear el esqueleto; si alguno cambió en Boot 4.x, se corrige en la regla.)

## Lógica de reconciliación

El evento de Kafka es solo un **disparador**: el planificador nunca usa `newVersion` del evento para decidir, sino la comparación entre la versión local y la metadata de Redis. Por eso un evento duplicado, retrasado o perdido produce el mismo resultado (R3).

### Planificador (función pura)

Entrada: versión local (o ninguna) y metadata remota (`current`, `oldest`). Salida: `SyncPlan = Noop | Incremental(desde, hasta) | Snapshot(motivo)`.

| # | Condición | Resultado |
|---|---|---|
| 1 | No hay versión local (base vacía) | `Snapshot(BASE_VACIA)` |
| 2 | Metadata ausente o incoherente (por ejemplo `current < oldest`) | `Snapshot(NO_VERIFICABLE)` |
| 3 | `local > current` | `Snapshot(LOCAL_ADELANTADA)` |
| 4 | `local == current` | `Noop` |
| 5 | `local < oldest` | `Snapshot(FUERA_DE_VENTANA)` |
| 6 | En otro caso (`oldest <= local < current`) | `Incremental(local+1, current)` |

La fila 5 sigue la referencia 14.2 y el ejemplo 18.2 (local 3, `oldest` 4 → snapshot). Se mantiene la lectura literal (la más conservadora) hasta resolver la consulta 10: si `oldest` significa la primera versión con `changes` disponible o la versión base desde la cual se puede continuar.

**Cerrada localmente el 2026-10-21 (a confirmar con el docente):** se mantiene la lectura literal. Es la regla conservadora: el peor caso es un snapshot de más (con este catálogo, barato), nunca una copia local corrupta. Si el docente responde que `oldest` es la versión base desde la que se puede continuar, cambia solo esta fila y sus pruebas de borde.

### Aplicación incremental

1. Para cada versión `v` en orden, de `local+1` a `current`: leer `changes:{v}`; si falta, abortar el incremental y pasar a `Snapshot(FALTA_VERSION)`.
2. Leer del Hash correspondiente el estado vigente de cada ID afectado y aplicarlo (*upsert*), en el orden categorías → profesionales → horarios (por las claves foráneas).
3. En la misma transacción, avanzar la versión local a `v`. Si algo falla: *rollback* y la versión no avanza.
4. Al terminar, releer `current`: si subió, continuar; si no, finalizar.

Cada versión es su propia unidad de trabajo. Como los Hashes contienen el estado vigente y no el de la versión `v`, los datos pueden ir adelantados respecto de la versión registrada en un paso intermedio. No es un problema: los *upserts* son idempotentes y el proceso converge a `current`.

### Aplicación del snapshot

1. Pedir `GET /api/synchronization/snapshot`.
2. En una sola transacción: *upsert* de categorías, profesionales y horarios (en ese orden); deshabilitar localmente las entidades que no vienen en el snapshot; registrar `snapshotVersion` como versión local.
3. Si falla, *rollback* y queda el estado anterior.

### Bajas: se deshabilita, nunca se borra

- La cátedra representa las bajas como `enabled=false` y las entidades deshabilitadas permanecen en los Hashes y en el snapshot (referencia 14.4); se guardan tal cual.
- Si un ID aparece en `changes` pero ya no está en el Hash (el contrato no lo prevé), se deshabilita localmente y se registra una advertencia. Si en la misma corrida aparece un segundo ID ausente, el estado de Redis se considera no verificable: se aborta la corrida y corresponde un snapshot. Un solo ID ausente puede ser una carrera entre la lectura de `changes` y la del Hash; varios indican un Redis inconsistente.
- Si el snapshot no trae una entidad que existe localmente, se deshabilita.
- Nunca se borra físicamente: preserva referencias de `cronos-turnos` a profesionales y deja trazabilidad.

### Exclusión mutua y disparadores

- Una única operación de sync a la vez: bloqueo de la fila de estado de sync (`SELECT ... FOR UPDATE`, ver modelo de datos). El arranque, el consumidor de Kafka y la verificación periódica usan el mismo caso de uso.
- Disparadores: (1) arranque de la aplicación, (2) evento `CatalogUpdated`, (3) verificación periódica de `current` contra la versión local, para recuperar notificaciones perdidas (enunciado 8). El intervalo es configurable.
- El *offset* de Kafka se confirma solo después de persistir (*ack* manual, referencia 16).

### Casos borde

| Caso | Resultado esperado |
|---|---|
| Base vacía | Snapshot inicial |
| `local == current` | `Noop` |
| Evento duplicado (mismo `eventId` o misma versión) | `Noop` por comparación de versiones; se confirma el offset |
| Evento fuera de orden o con `newVersion` menor que la local | `Noop` |
| Notificación perdida | La verificación periódica detecta `local < current` y sincroniza |
| `local < oldest` | Snapshot |
| Falta `changes:{v}` en Redis | Snapshot; el incremental parcial ya aplicado queda consistente |
| ID en `changes` ausente en el Hash | Se deshabilita localmente y se registra una advertencia; con un segundo ID ausente en la corrida, snapshot |
| `local > current` | `Snapshot(LOCAL_ADELANTADA)` |
| Falla a mitad de una versión | *Rollback*; la versión no avanza; se reintenta |
| Falla a mitad de un snapshot | *Rollback*; queda el estado anterior |
| Redis, REST o Kafka caídos | Falla de integración: reintentos acotados, estado de error visible; no se escala a snapshot por una falla transitoria |
| Referencia inexistente (por ejemplo, un horario de un profesional ausente) | *Rollback* y `Snapshot(NO_VERIFICABLE)` |
| Mensaje de Kafka corrupto o no deserializable | Se registra y se descarta sin bloquear el consumidor; la verificación periódica cubre el hueco |
| Dos syncs en paralelo | Se serializan por el bloqueo; el segundo ve el resultado del primero (`Noop`) |
| Reinicio durante un sync | La unidad de trabajo es atómica; al arrancar se vuelve a planificar |
| Formato de hora `HH:mm` o `HH:mm:ss`, campos desconocidos | Se aceptan; cubierto por tests (ADR 0002, D2) |

## Pruebas por capa

- `domain` / planificador: unitarias puras, una por fila de la tabla del planificador.
- `application`: integración contra PostgreSQL (Testcontainers) con las fuentes (REST y Redis) simuladas; cubre transacciones, idempotencia y rollback.
- `adapters.out`: contra servidor simulado (`MockRestServiceServer` o WireMock) y Redis real (Testcontainers).
- `adapters.in`: Kafka real (Testcontainers) para duplicados y fallas; MockMvc para autorización.
- `ArquitecturaTest`: las reglas de arriba.

## Consecuencias

- Las decisiones de sync se prueban sin infraestructura (R1), y cada tabla de arriba se traduce en casos de prueba.
- Se acepta el costo de los dos compromisos; si resultan un problema en la defensa, la migración a la opción C es acotada (agregar un puerto y un modelo de dominio).
- El modelo de datos (issue #7) debe incluir una fila de estado de sync (versión local, estado, último error) usada como candado, y soportar el borrado lógico.
- Depende de la consulta al docente sobre `oldest-available-version` (consulta 10) y de la validación de los compromisos (consulta 3).

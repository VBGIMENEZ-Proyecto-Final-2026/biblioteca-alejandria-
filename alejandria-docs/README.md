# Biblioteca de Alejandría

Carpeta que **indexa** la documentación y las skills del proyecto integrador de Programación 2. No reemplaza la documentación obligatoria de cada repo evaluado: esa también vive (o se linkea) en los 3 repos de código.

## Los 4 repos

| Repo | Rol | URL |
| --- | --- | --- |
| `hermes-app` | App Android KMP | `<url-del-repo>` |
| `atlas-catalogo` | Backend de catálogo y sincronización | `<url-del-repo>` |
| `cronos-turnos` | Backend de turnos y reservas | `<url-del-repo>` |
| `biblioteca-alejandria-` | Este repo: documentación, plantillas y skills (cuarto repo, aprobado por el profe) | `<url-del-repo>` |

## Arquitectura general

Vista de conjunto: la cátedra publica el catálogo y los turnos por REST, Redis y Kafka; `atlas-catalogo` mantiene la copia local del catálogo, `cronos-turnos` maneja turnos y reservas, y `hermes-app` consume a los dos.

```text
                    ┌────────── Cátedra ──────────┐
                    │  REST     Redis     Kafka   │
                    └──┬──┬──────┬──┬───────┬──┬──┘
        catálogo ──────┘  │      │  │       │  └── turnos
                          │      │  │       │
              ┌───────────┘      │  └───────┴────────────┐
              ▼                  ▼                       ▼
        atlas-catalogo  ◄──── agenda ────  cronos-turnos
              ▲                                          ▲
              │ búsqueda                                 │ auth, disponibilidad, reservas
              └──────────────── hermes-app ──────────────┘
```

Cada repo de código tiene en su README un diagrama más cercano a su responsabilidad.

## Mapa de carpetas

| Carpeta | Qué va |
| --- | --- |
| `catedra/` | Documentación provista por los profesores (enunciado y referencia de integración). Fuente de verdad de los contratos externos. |
| `arquitectura/` | Diagramas y explicación de la arquitectura; `arquitectura/adr/` para las decisiones de diseño. |
| `contratos/` | Contratos entre `atlas-catalogo` y `cronos-turnos`: operaciones, DTO, autenticación y errores. |
| `modelo-datos/` | Modelo de datos de cada servicio y delimitación de su propiedad. |
| `implementacion/` | Evolución del código de cada servicio: qué se construyó, en qué orden y qué se decidió al implementar frente a los ADR. |
| `skills/` | Skills del proyecto (vacía en esta versión). |
| `evidencias/` | Resultados de pruebas y verificaciones, sin secretos. |
| `backlog/` | Documentos de backlog por iteración, fuente para cargar los issues. |
| `plantillas/` | Fuente de verdad de las plantillas (`.github/` y scripts) que se replican en los otros repos. |

## Documentos de atlas-catalogo

| Documento | Qué cuenta |
| --- | --- |
| [`arquitectura/sincronizacion-atlas-catalogo.md`](arquitectura/sincronizacion-atlas-catalogo.md) | Cómo funciona la sincronización, idempotencia y recuperación |
| [`arquitectura/seguridad-atlas-catalogo.md`](arquitectura/seguridad-atlas-catalogo.md) | Decisiones, estado y limitaciones de seguridad |
| [`modelo-datos/atlas-catalogo.md`](modelo-datos/atlas-catalogo.md) | Modelo de datos y decisiones A a L |
| [`implementacion/atlas-catalogo.md`](implementacion/atlas-catalogo.md) | Evolución del código y decisiones al implementar |
| [`evidencias/`](evidencias/) | Verificaciones contra la cátedra real, sin secretos |

## Convenciones

- **Iteraciones** (campo Iteration del tablero `Odisea · Prog2`): `Isla N · Nombre`.
  - `Isla 0 · Ítaca`: setup, cuenta técnica, secretos, compose base
  - `Isla 1 · Eolia`: `atlas-catalogo`, snapshot e incremental
  - `Isla 2 · Ogigia`: `cronos-turnos`, auth y disponibilidad
  - `Isla 3 · Escila y Caribdis`: hold + Kafka, robustez
  - `Isla 4 · Regreso a Ítaca`: `hermes-app` end-to-end, seguridad, tests, docs finales
- **Labels:** `tarea`, `bug`, `consulta`, `atlas`, `cronos`, `hermes`, `docs`, `infra`, `seguridad`, `tests`, `bloqueante` (se crean con `plantillas/scripts/bootstrap-repo.sh`).
- **Cero secretos:** nada de IPs, puertos de la cátedra, network IDs, tokens, `groupId` reales ni credenciales en archivos ni en el historial. En los ejemplos se usan placeholders (`<groupId>`, `usuario/repo`).
- **`plantillas/` es la fuente de verdad** de las plantillas: se edita acá y se replica en los demás repos.

## Cómo aplicar las plantillas a un repo nuevo

Con `gh` autenticado:

```bash
plantillas/scripts/aplicar-plantillas.sh /ruta/al/repo-local usuario/repo [--con-milestones]
```

Copia `plantillas/.github/` al repo local y crea los labels (y opcionalmente milestones) en GitHub. Después revisá el diff y hacé un commit propio en el repo destino.

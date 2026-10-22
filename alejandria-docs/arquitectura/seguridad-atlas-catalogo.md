# Seguridad de atlas-catalogo · decisiones, estado y limitaciones

**Componente:** atlas-catalogo · **Isla:** 1 · Eolia · **Actualizado:** 2026-10-12 · **Estado:** parcial (la seguridad de la API es el ítem 5 del backlog y todavía no está implementada)

Este documento responde a lo que pide el enunciado (sección 9): documentar las decisiones y las limitaciones de seguridad. Está dividido en lo que **ya existe y se puede verificar**, lo que **falta** y las **limitaciones asumidas**. Se actualiza al implementar la seguridad base.

## Alcance

`atlas-catalogo` mantiene una copia del catálogo (categorías, profesionales y horarios semanales). **No guarda datos personales de usuarios** ni contraseñas: el registro y el inicio de sesión de usuarios son de `cronos-turnos`. Lo que sí maneja son credenciales técnicas de la cátedra (token JWT, contraseña de Redis) y, a futuro, la validación de los tokens con los que otros servicios le piden datos.

## Lo que ya existe

| Control | Qué hace | Dónde se verifica |
|---|---|---|
| **Secretos fuera del código** | Todas las credenciales y valores de la cátedra salen de variables de entorno o de un `.env` local ignorado por git (`.env`, `.env.*`, `*.pem`, `*.key`, `*.jks`). El repositorio solo versiona `.env.example` con placeholders. Convención en el [ADR 0001](adr/0001-configuracion-y-secretos.md). | `.gitignore`; revisión de commits |
| **El token de la cátedra no se escribe en ningún lado** | No aparece en logs, mensajes de error ni en el `toString` de las propiedades. | `CatedraRestSnapshotSourceTest.elTokenNoApareceEnLosMensajesDeLosErrores`, `elTokenNoApareceEnElToStringDeLasPropiedades` |
| **Verificado en una ejecución real** | Con la aplicación corriendo contra la cátedra real (arranque con la base vacía, snapshot, lectura de Redis y consumidor de Kafka), el token de la cátedra, la contraseña de Redis y la de la base **no aparecen en el log**: cero apariciones en las unas 200 líneas que genera un arranque. | Revisión manual del log del 2026-10-12 |
| **La contraseña de Redis tampoco** | Enmascarada en `toString`, ausente de los mensajes cuando se rechaza. | `CatedraRedisPropertiesTest.laContrasenaNoApareceEnToString`, `CatedraRedisChangeSourceTest.conUnaContrasenaIncorrectaSeRechazaLaCredencialYNoSeReintenta` |
| **El token no se reenvía a otro destino** | El cliente REST no sigue redirecciones: una respuesta `3xx` no puede llevarse la cabecera `Authorization` a otro servidor. | `CatedraRestSnapshotSourceTest.noSigueRedireccionesParaNoReenviarElToken` |
| **Mínimo privilegio en Redis** | El cliente solo hace lecturas puntuales sobre `catedra:sync:*` (nunca `KEYS`, `SCAN`, escrituras, `catedra:holds:*` ni el namespace privado). Los tests corren con un usuario ACL que solo puede leer esas claves: si el código hiciera otra cosa, Redis respondería `NOPERM` y fallarían. | `CatedraRedisChangeSourceTest` |
| **Kafka solo consume y no crea topics** | El consumidor nunca publica y tiene desactivada la creación automática de topics. | `CatedraKafkaConfig` |
| **Los errores de integración no filtran detalles** | El estado que queda en la base usa mensajes fijos por código; el detalle técnico va solo al log, sin credenciales ni el cuerpo de los mensajes de Kafka. | `SincronizacionIncrementalTest.unaFallaInesperadaDeAplicacionQuedaConSuPropioCodigo`, `CatalogUpdatedParserTest.elMotivoDelDescarteNoIncluyeElCuerpoDelMensaje` |
| **Validación de datos en cada borde de integración** | Cada dato que viene de afuera se valida al convertirlo: campos obligatorios de REST y de Redis, `id` coherente con el field, versión coherente con la pedida, mensajes de Kafka con `eventType` y `newVersion`. Una respuesta inválida nunca llega a la base. | `CatedraRestSnapshotSourceTest`, `CatedraRedisChangeSourceTest`, `CatalogUpdatedParserTest` |
| **Sin inyección SQL en la sincronización** | Todas las escrituras usan consultas con parámetros (*upserts* y bajas); no se arma SQL con texto externo. | Repositorios en `application.repository` |
| **Superficie expuesta mínima** | Hoy solo se expone `GET /actuator/health`, sin detalles (`show-details: never`), y `GET /api/sync/status` únicamente si se enciende `ATLAS_SYNC_STATUS_ENDPOINT` (apagado por defecto: sin él, la ruta responde `404`). Redis no figura en el health, para no revelar el estado de la cátedra. | `AtlasCatalogoApplicationTests` |
| **Base de datos solo local** | El `compose.yaml` publica PostgreSQL en `127.0.0.1`, no en la red. Cada servicio tiene su propia base y sus propias migraciones; atlas nunca lee la base de otro servicio. | `compose.yaml` |
| **Arranque que falla rápido con configuración incompleta** | Si falta una variable obligatoria, la aplicación no arranca y el mensaje dice cuál, sin imprimir su valor. | `CatedraPropertiesTest`, `CatedraRedisPropertiesTest`, `CatedraKafkaPropertiesTest` |

## Lo que falta (ítem 5 del backlog · seguridad base)

Hoy atlas **no tiene endpoints de negocio** y el único endpoint propio, el estado de la sincronización, está apagado por defecto, así que todavía no hay nada que autenticar. Cuando existan la búsqueda y el estado de la sincronización, el ítem 5 debe agregar, como mínimo (enunciado 9):

- Validar el JWT en todos los endpoints no públicos: firma, vigencia y permisos. Una petición sin token, con un token inválido o vencido, recibe `401`; sin permiso suficiente, `403`.
- Autorizar cada operación según la identidad autenticada, sin confiar en identificadores de usuario enviados libremente por el cliente.
- Configurar CORS solo para los clientes permitidos.
- Validar los datos de entrada de cada endpoint.
- Que los errores de seguridad no expongan secretos ni detalles internos.
- Pruebas de autenticación y autorización, y de rechazo de accesos anónimos.
- Proteger `GET /api/sync/status` (`401` sin token, `403` sin permiso), cambiar su valor por defecto a encendido y agregar la prueba de rechazo anónimo. Hasta entonces el endpoint no debe encenderse en un entorno al que llegue alguien que no deba ver el estado.

**Decisión propuesta (D12 del [ADR 0002](adr/0002-stack.md), a confirmar con el docente · consulta 1):** JWT firmado con RS256. Quien emite los tokens es `cronos-turnos` con una clave privada; atlas solo conoce la **clave pública**, así que no puede forjar tokens. La alternativa, HS256 con secreto compartido, permitiría que cualquiera de los dos servicios firme tokens. La clave pública se configura por variable de entorno y se prueba con claves de prueba. Si la reunión con el docente o la viabilidad en `cronos-turnos` (si se usa JHipster, que por defecto firma con secreto compartido) lo impiden, se reemplaza por un ADR nuevo con secreto compartido y se documenta esa limitación.

**Pendiente de decisión (consulta 1):** qué se espera entre servicios, si propagar el JWT del usuario o usar un JWT técnico. Para atlas, que no guarda datos por usuario, el criterio propuesto es validar cualquier JWT válido firmado por el emisor y distinguir permisos con un *claim*.

## Limitaciones y supuestos

- **No hay autenticación de la API todavía.** Mientras no se implemente el ítem 5, cualquier endpoint de negocio que se agregue quedaría abierto; por eso no se agregan antes. La única excepción es el estado de la sincronización, que nace apagado y se enciende a conciencia.
- **Tráfico sin cifrar hacia la cátedra.** La referencia de integración no menciona TLS para REST, Redis ni Kafka (el ejemplo de URL base del `.env.example` es `http://`), y las pruebas reales se hicieron dentro de la red privada (ZeroTier) que la cátedra provee. El cliente REST acepta `http` y `https`; Redis y Kafka no tienen TLS configurado. Se asume la protección de la red privada.
- **Kafka sin autenticación por defecto, con autenticación opcional ya preparada.** La cátedra no documenta credenciales y la conexión real funcionó sin ellas (consulta 11 al docente), así que la conexión es `PLAINTEXT`. Si las exigieran, el perfil `kafka-sasl` activa `SASL_SSL` (mecanismo `PLAIN`) con el usuario y la contraseña en `CATEDRA_KAFKA_SASL_JAAS_CONFIG`, sin tocar código. No se soportan mTLS ni otros mecanismos hasta que la cátedra los pida. El JAAS no aparece en `toString` ni en el log del cliente (`[hidden]`). Probado hasta la configuración que recibe el cliente, no contra un Kafka con autenticación real (hoy no hay uno).
- **El JWT técnico de la cátedra dura un año** (referencia 4) y se rota a mano: si vence, la cátedra responde `401` y la sincronización queda en error `CATEDRA_CREDENCIAL_RECHAZADA` hasta que se cambie el `.env` y se reinicie el servicio. Las credenciales se leen al arrancar.
- **Los dos backends comparten la misma cuenta técnica y el mismo JWT de cátedra** (lo define la cátedra). Una filtración del `.env` de uno compromete a los dos; por eso `.env` nunca se versiona.
- **Los repositorios son públicos:** cero secretos, IPs, puertos de la cátedra, `groupId` reales ni credenciales en archivos ni en el historial. Las evidencias omiten esos datos a propósito.
- **Sin integración continua ni escaneo de dependencias automatizado.** El repositorio no tiene flujos de GitHub Actions: los tests y las reglas de arquitectura se corren a mano con `./gradlew build` antes de cada PR. Las versiones de las dependencias salen de la gestión de Spring Boot 4.1.
- **Los logs no tienen rotación ni destino definido:** salen por la salida estándar del proceso. **Contienen datos de la integración que no son secretos pero no conviene publicar:** el cliente de Kafka registra en `INFO` los servidores, el topic y el consumer group (al volcar su configuración y al asignar la partición). Por eso los logs no se publican ni se pegan en issues, PRs o evidencias sin revisarlos antes.

# Seguridad de atlas-catalogo · decisiones, estado y limitaciones

**Componente:** atlas-catalogo · **Isla:** 1 · Eolia · **Actualizado:** 2026-11-01 · **Estado:** seguridad base implementada (ítem 5, issue #33, PR #34); quedan pendientes los claims definitivos, el cifrado del canal y los límites que se detallan abajo

Este documento responde a lo que pide el enunciado (sección 9): documentar las decisiones y las limitaciones de seguridad. Está dividido en lo que **ya existe y se puede verificar**, lo que **falta** y las **limitaciones asumidas**. Se actualiza cada vez que cambia la seguridad.

## Alcance

`atlas-catalogo` mantiene una copia del catálogo (categorías, profesionales y horarios semanales). **No guarda datos personales de usuarios** ni contraseñas: el registro y el inicio de sesión de usuarios son de `cronos-turnos`. Lo que sí maneja son credenciales técnicas de la cátedra (token JWT, contraseña de Redis) y la validación de los tokens con los que la app y `cronos-turnos` le piden datos. La decisión de cómo se autentican los servicios entre sí está en el [ADR 0004](adr/0004-autenticacion-entre-servicios.md).

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
| **Superficie expuesta mínima** | Solo `GET /actuator/health` es público, sin detalles (`show-details: never`). Redis no figura en el health, para no revelar el estado de la cátedra. Los demás endpoints de Actuator no están expuestos. | `AtlasCatalogoApplicationTests`, `SeguridadApiTest.soloElHealthEsPublico` |
| **Validación del JWT** | Firma RS256 con la **clave pública** de `cronos-turnos` (atlas no tiene la privada). Se rechazan `HS256` (incluido el ataque de confusión de algoritmo que usa la clave pública como secreto), `none`, otros algoritmos RSA, tokens vencidos, aún no válidos, con la firma de otra clave, con el payload adulterado, sin `exp` o sin `sub`. La tolerancia de reloj es de 60 s (configurable, hasta 5 minutos). | `SeguridadApiTest` (tokens maliciosos y vencimiento con tolerancia) |
| **Autorización por rol y denegación por defecto** | La búsqueda pide `ROLE_USER` o `ROLE_ADMIN`; el estado de la sincronización, `ROLE_ADMIN`; todo lo demás se deniega, exista o no la ruta (otros métodos, rutas desconocidas, otros endpoints de Actuator). Las autoridades salen del claim `auth`, no de otro claim ni de un encabezado. | `SeguridadApiTest` (matriz de roles y métodos) |
| **Ninguna decisión depende de un identificador del cliente** | Encabezados como `X-User` o `X-On-Behalf-Of` y parámetros como `userId` no sustituyen al token. | `SeguridadApiTest.unIdentificadorEnviadoPorElClienteNoSustituyeAlToken` |
| **Errores de seguridad sin detalle interno** | `401 NO_AUTENTICADO` y `403 SIN_PERMISO` como `problem+json`, con `WWW-Authenticate: Bearer`. No dicen por qué se rechazó el token (firma, vencimiento, algoritmo) ni qué rol faltaba, y nunca repiten el token. | `SeguridadApiTest.elRechazoNoDicePorQueSeRechazoElToken`, `elRechazoPorPermisosNoRevelaQueRolFaltaba` |
| **Trazabilidad de los rechazos** | Cada rechazo queda en el log con el método y la ruta, y en un `403` con el usuario; nunca con el token. Sirve para responder "¿qué pasó y quién lo pidió?". | `SeguridadApiTest.cadaRechazoQuedaEnElLogSinElToken`; revisión del log de una ejecución real (2026-11-01) |
| **Sin sesión ni cookies** | La API es sin estado: el token viaja en `Authorization: Bearer`, y solo ahí (no se acepta por parámetro de la URL ni con otro esquema). Por eso CSRF no aplica y está desactivado. | `SeguridadApiTest.elTokenSoloSeAceptaEnElEncabezadoAuthorizationBearer` |
| **CORS restrictivo** | Solo los orígenes de `ATLAS_CORS_ALLOWED_ORIGINS` (esquema, host y puerto, sin comodines ni rutas), solo `GET` y `OPTIONS`, solo los encabezados `Authorization`, `Content-Type` y `Accept`, sin credenciales. Sin configuración no se acepta ningún origen. | `CorsApiTest`, `CorsPorDefectoTest`, `SecurityPropertiesTest` |
| **La clave pública se valida al arrancar** | Sin clave la aplicación no arranca. Se exige RSA de 2048 bits o más, en PEM X.509: una clave privada, una corta o una que no es RSA se rechazan, sin repetir su contenido en el mensaje. | `JwtPublicKeysTest`, `SecurityPropertiesTest` |
| **Validación de los parámetros de la búsqueda** | `size` de 1 a 100, `page` no negativa, `name` de hasta 100 caracteres y `dayOfWeek` solo con `available`; un valor de tipo incorrecto o fuera de rango es `400 BUSQUEDA_INVALIDA` sin repetir lo recibido. Los comodines de `LIKE` se tratan como texto. El orden no lo elige el cliente. | `ProfessionalSearchCriteriaTest`, `ProfessionalsControllerTest`, `BusquedaProfesionalesTest` |
| **Base de datos solo local** | El `compose.yaml` publica PostgreSQL en `127.0.0.1`, no en la red. Cada servicio tiene su propia base y sus propias migraciones; atlas nunca lee la base de otro servicio. | `compose.yaml` |
| **Arranque que falla rápido con configuración incompleta** | Si falta una variable obligatoria, la aplicación no arranca y el mensaje dice cuál, sin imprimir su valor. | `CatedraPropertiesTest`, `CatedraRedisPropertiesTest`, `CatedraKafkaPropertiesTest` |

## Cobertura de la sección 9 del enunciado

Lo que corresponde a atlas. Lo demás es de `cronos-turnos`, que emite los tokens y guarda usuarios.

| Requisito del enunciado | Estado en atlas |
|---|---|
| Exigir y validar el JWT en las APIs protegidas | **Hecho** (ítem 5) |
| Validar JWT en la comunicación entre servicios | **Hecho del lado de atlas**; falta que cronos reenvíe el token del usuario (ADR 0004) |
| Autorizar cada operación según la identidad autenticada | **Hecho**: roles del token por ruta |
| Impedir el acceso a datos de otro usuario | No aplica: atlas no guarda datos por usuario |
| Proteger los endpoints que no sean públicos | **Hecho**: solo `GET /actuator/health` es público y el resto se deniega |
| No confiar en identificadores enviados por el cliente | **Hecho** |
| Externalizar credenciales, tokens y secretos; no exponerlos | **Hecho** (ADR 0001); la clave pública no es secreta |
| Configurar CORS según los clientes permitidos | **Hecho** |
| Validar los datos recibidos en cada límite | **Hecho** para los parámetros de la API y los datos de la cátedra |
| Emitir el JWT, registrar y autenticar usuarios, guardar contraseñas | No aplica a atlas: es de `cronos-turnos` |
| **Proteger la comunicación entre la interfaz y los servicios** | **Pendiente**: atlas escucha en HTTP; ver las limitaciones |
| Documentar las decisiones y limitaciones | Este documento y el [ADR 0004](adr/0004-autenticacion-entre-servicios.md) |

## Lo que falta

- **Fijar los claims con `cronos-turnos`.** Atlas espera `sub` con el usuario y `auth` con las autoridades (como JHipster), y el nombre del claim es configurable. Es una propuesta de atlas que cronos todavía no ha confirmado.
- **Confirmar con el docente la autenticación entre servicios** (consulta 1: JWT propagado o técnico). Si pide un JWT técnico, cambia la llamada entre servicios, no la validación de usuario.
- **Cifrar el canal entre la app y los servicios** (ver las limitaciones).
- **Un endpoint de lectura para `cronos-turnos`** (horario semanal vigente), con su contrato documentado, protegido con este mismo esquema.

## Limitaciones y supuestos

- **La comunicación con atlas no va cifrada.** Atlas escucha en HTTP: el enunciado pide proteger la comunicación entre la interfaz y los servicios, y eso no está resuelto. Las pruebas son locales o dentro de la red privada. Para un despliegue hay que terminar TLS delante de atlas (un proxy inverso o el contenedor) o configurar `server.ssl.*`; ninguna de las dos está hecha.
- **No se valida el emisor ni la audiencia del token** (`iss` y `aud`). Atlas acepta cualquier token firmado con la clave configurada. Con un solo emisor y una sola clave es aceptable, y se revisa si aparece otro emisor.
- **No hay revocación de tokens.** Un token robado vale hasta su vencimiento; la defensa es que los tokens sean de corta duración, y eso lo decide cronos al emitirlos.
- **Rotar la clave pública exige reiniciar atlas**, porque se carga al arrancar (a propósito: no depende de que cronos esté levantado, ADR 0004).
- **Cualquier usuario autenticado con `ROLE_USER` ve el catálogo completo**, incluidos los profesionales deshabilitados, y un administrador ve además el estado de la sincronización, que incluye mensajes de error fijos pero revela cuándo falla la cátedra.
- **No hay límite de pedidos ni bloqueo ante intentos repetidos** (*rate limiting*). Los rechazos solo se registran.
- **Las claves de los tests no son un secreto:** el repositorio versiona una clave **pública** de prueba (`src/test/resources/jwt-test-public.pem`, con una excepción explícita en `.gitignore`) y los tests generan sus pares al ejecutarse; la privada de esa clave se descartó. `scripts/dev-token.sh` firma tokens de desarrollo con una clave local que nunca se commitea.
- **Tráfico sin cifrar hacia la cátedra.** La referencia de integración no menciona TLS para REST, Redis ni Kafka (el ejemplo de URL base del `.env.example` es `http://`), y las pruebas reales se hicieron dentro de la red privada (ZeroTier) que la cátedra provee. El cliente REST acepta `http` y `https`; Redis y Kafka no tienen TLS configurado. Se asume la protección de la red privada.
- **Kafka sin autenticación por defecto, con autenticación opcional ya preparada.** La cátedra no documenta credenciales y la conexión real funcionó sin ellas (consulta 11 al docente), así que la conexión es `PLAINTEXT`. Si las exigieran, el perfil `kafka-sasl` activa `SASL_SSL` (mecanismo `PLAIN`) con el usuario y la contraseña en `CATEDRA_KAFKA_SASL_JAAS_CONFIG`, sin tocar código. No se soportan mTLS ni otros mecanismos hasta que la cátedra los pida. El JAAS no aparece en `toString` ni en el log del cliente (`[hidden]`). Probado hasta la configuración que recibe el cliente, no contra un Kafka con autenticación real (hoy no hay uno).
- **El JWT técnico de la cátedra dura un año** (referencia 4) y se rota a mano: si vence, la cátedra responde `401` y la sincronización queda en error `CATEDRA_CREDENCIAL_RECHAZADA` hasta que se cambie el `.env` y se reinicie el servicio. Las credenciales se leen al arrancar.
- **Los dos backends comparten la misma cuenta técnica y el mismo JWT de cátedra** (lo define la cátedra). Una filtración del `.env` de uno compromete a los dos; por eso `.env` nunca se versiona.
- **Los repositorios son públicos:** cero secretos, IPs, puertos de la cátedra, `groupId` reales ni credenciales en archivos ni en el historial. Las evidencias omiten esos datos a propósito.
- **Sin integración continua ni escaneo de dependencias automatizado.** El repositorio no tiene flujos de GitHub Actions: los tests y las reglas de arquitectura se corren a mano con `./gradlew build` antes de cada PR. Las versiones de las dependencias salen de la gestión de Spring Boot 4.1.
- **Los logs no tienen rotación ni destino definido:** salen por la salida estándar del proceso. **Contienen datos de la integración que no son secretos pero no conviene publicar:** el cliente de Kafka registra en `INFO` los servidores, el topic y el consumer group (al volcar su configuración y al asignar la partición). Por eso los logs no se publican ni se pegan en issues, PRs o evidencias sin revisarlos antes.

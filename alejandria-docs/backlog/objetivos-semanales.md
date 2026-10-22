# Objetivos semanales

Seguimiento corto del proyecto: qué se pidió en cada reunión con el docente y qué objetivos se fijan para avanzar de forma constante. Las iteraciones grandes viven en `isla-N-*.md`; este documento solo baja el trabajo a la semana.

## Reunión del 2026-10-22

### Lo que se pidió

- **Ritmo:** avanzar a paso seguro. Se hace, y después se debate cómo se pensó y cómo se resolvió.
- **Código real:** desde la semana que viene se trabaja con código real. A mayor profundidad de la funcionalidad, mayor complejidad.
- **Escenarios de falla:** analizar en detalle qué pasa cuando ocurre un evento o un error concreto.
- **Trazabilidad:** poder explicar qué acción disparó cada evento o proceso.
- **Despliegue:** dos motores en instancias separadas funciona sin problemas. Falta analizar un solo motor dentro de un contenedor Docker.
- **Base de datos:** toda la solución usa una sola base de datos centralizada.
- **Posible pedido de la semana que viene:** el docente podría plantear una situación concreta, por ejemplo "se disparó una alerta, ¿qué pasa?", y pedir que se explique y se muestre qué ocurre en el sistema.

### Qué cambia en el proyecto

- La base única afecta a los tres servicios. Hay que decidir si cada servicio usa su propio esquema o si comparten tablas, y reflejarlo en `modelo-datos/`.
- El caso de un solo motor en Docker tiene que quedar probado en local antes de debatirlo.
- Los escenarios de alerta, de falla y la trazabilidad se documentan en `evidencias/`, con la acción exacta que los dispara y la señal que deja (log, código de error, estado de la sincronización).

### Dudas para el docente

- ¿"Motor" es el motor de base de datos o el servicio completo?
- ¿La base única se comparte con esquemas separados por servicio, o con tablas compartidas?
- Si pide una situación como la de la alerta: ¿se muestra funcionando en local o alcanza con explicarla paso a paso?

## Objetivos de la semana del 2026-10-26

Cada objetivo tiene un criterio de terminado y algo concreto para explicar en la defensa. Si un día se cumple solo uno, alcanza: la idea es no cortar el ritmo.

| Día | Objetivo | Terminado cuando | Para explicar |
| --- | --- | --- | --- |
| Viernes 23 | Plan previo a la facu: levantar todo en local y ensayar dos escenarios | Los servicios levantan con Docker y hay un escenario de error y uno de trazabilidad anotados | Qué hice para disparar el evento y qué pasó |
| Lunes 26 | Definir el modelo de la base única | Decisión escrita en `modelo-datos/`: esquema por servicio o tablas compartidas, con el porqué | Alternativas, pros y contras |
| Martes 27 | Probar un solo motor de base de datos en Docker | Los 3 servicios apuntan a la misma instancia en local | Qué pasa si cae la base o un servicio |
| Miércoles 28 | Publicar el trabajo local de `atlas-catalogo` | Ramas mergeadas a `dev` en el orden decidido, conflictos resueltos, issues y registro semanal al día | Por qué ese orden de mergeo |
| Jueves 29 | Escenarios de alerta y falla documentados | Dos o más escenarios en `evidencias/`, cada uno con qué alerta o señal se dispara (código de error en el log, estado de la sincronización), su causa, su efecto, la acción que lo provoca y cómo se recupera | Qué pasa cuando salta la alerta y cómo lo sigo hasta el origen |
| Viernes 30 | Cierre y ensayo de la situación pedida | Estado actualizado en `backlog/` y en el registro semanal, y un escenario ensayado de punta a punta (dispararlo, ver la señal, explicar el efecto y la recuperación) | Resumen de lo avanzado y lo que falta |

Dependen de la respuesta del docente: la seguridad base (ítem 5) y la búsqueda (ítem 11) de la Isla 1. No se planifican hasta tenerla.

## Cómo se actualiza este documento

- Al terminar cada reunión se agrega una sección con la fecha y lo que se pidió.
- Al cerrar cada día se tilda lo cumplido en el registro semanal y se ajusta la tabla si el plan cambió.

# Post-mortem: pérdida de acceso al panel de n8n tras recrear la cuenta de administrador

Análisis de un incidente en producción. Una instancia self-hosted de n8n quedó inaccesible desde el navegador tras reconstruir la cuenta propietaria, mientras la API, la base de datos y todos los workflows programados seguían funcionando con normalidad.

Se publica porque el modo de fallo es engañoso (se presenta como un problema de autenticación cuando en realidad es de integridad referencial) y porque el error que finalmente lo identificó no está documentado en ningún sitio que haya podido encontrar.

**Fecha del incidente:** 15 de agosto de 2026
**Tiempo hasta la resolución:** aproximadamente 1 hora 20 minutos
**Pérdida de datos:** ninguna. Los 24 workflows, sus credenciales y sus ejecuciones programadas siguieron operando durante todo el incidente.

**Entorno:** n8n self-hosted de la línea 2.33, Docker Swarm, Traefik como reverse proxy, SQLite como backend.

---

## 1. Resumen

Lo que empezó como una contraseña olvidada terminó en un frontend que se autenticaba correctamente y acto seguido devolvía al usuario a la pantalla de login, sin generar ningún error de autenticación en el servidor.

La causa raíz: **n8n localiza el proyecto personal de un usuario mediante la columna `project.creatorId`, no mediante la tabla `project_relation`.** Tras reconstruir la cuenta propietaria se reparó `project_relation`, pero `creatorId` seguía apuntando al usuario eliminado. El endpoint `GET /rest/projects/personal` devolvía 404, la inicialización del editor lanzaba una excepción dentro de un `Promise.all`, y el router redirigía de vuelta al login.

Como la autenticación nunca falló, los logs del servidor no mostraban nada útil. Esa única señal engañosa explica la mayor parte del tiempo de resolución.

---

## 2. Cronología

| Hora (UTC) | Evento |
|---|---|
| 11:45 | Login rechazado con "Wrong username or password" |
| 11:50 | Se detecta y corrige un bug real no relacionado: faltaba `N8N_PROXY_HOPS`, lo que rompía el rate limiter del login tras el proxy |
| 12:00 | No hay SMTP configurado, así que la recuperación por email no es viable. La única vía es el reset por CLI |
| 12:06 | `n8n user-management:reset` se ejecuta pero solo tiene éxito parcial. Deja una fila de usuario residual con el 2FA aún activo, por lo que el asistente de configuración nunca aparece |
| 12:10 | Reparación manual excesivamente agresiva: se elimina la fila de usuario completa. Esto rompe el enlace entre proyecto y usuario |
| 12:20 | Un segundo reset falla con `Could not find any entity of type "Project" matching: { "type": "personal", "creatorId": ... }`. Ese mensaje describía la causa raíz y no se siguió en su momento |
| 12:25 | Se repara `project_relation.userId`. Aparece el asistente y se crea la cuenta propietaria nueva |
| 12:29 | Cuenta creada con éxito, pero el editor no carga. El login funciona y devuelve al login |
| 12:29 a 13:20 | Contraste de hipótesis, ver sección 4 |
| 13:25 | La consola del navegador revela `GET /rest/projects/personal 404` y `Could not find a personal project for this user` |
| 13:30 | Se corrige `project.creatorId` junto con otras 725 referencias huérfanas. Resuelto |

---

## 3. Método de diagnóstico

La regla de trabajo fue no modificar producción por intuición. Cada hipótesis debía apoyarse en evidencia observable antes de actuar.

- Logs del contenedor leídos con marcas de tiempo y correlacionados con los momentos exactos de los intentos reales de login
- Consultas SQLite en modo solo lectura (`sqlite3.connect('file:...?mode=ro', uri=True)`) para inspeccionar el estado sin riesgo sobre una base de datos viva
- Lectura del código fuente desplegado (`auth.service.js` dentro del contenedor) en lugar de confiar en documentación o foros
- Reproducción del ciclo completo de login con `curl`, aislando una variable cada vez
- Modo debug temporal para capturar un intento concreto del usuario de principio a fin
- La consola del navegador, que resultó ser el único punto ciego que el servidor no podía cubrir

---

## 4. Hipótesis descartadas

Documentar los callejones sin salida importa tanto como documentar la solución. Cada uno se eliminó con evidencia, no por intuición.

| Hipótesis | Cómo se descartó |
|---|---|
| La contraseña simplemente era incorrecta | Una petición `curl` con una contraseña deliberadamente errónea devolvió un mensaje idéntico, lo que prueba que el mensaje es genérico y no tiene valor diagnóstico |
| Rate limiter roto por proxy mal configurado | Bug real, corregido con `N8N_PROXY_HOPS=1`. Los errores cesaron y el login siguió fallando |
| Cookie rechazada por el manejo de HTTPS o del proxy | `/rest/settings` ya devolvía `authCookie.secure: true`. Se añadió `N8N_PROTOCOL=https` como buena práctica, sin efecto sobre el síntoma |
| Validación de `browserId` fallando | El aviso `browserId check failed` apareció únicamente en mi propia prueba aislada con curl, nunca durante un intento real del usuario |
| Múltiples instancias dividiendo el estado de sesión | Verificado: una sola réplica, un único contenedor en ejecución |
| 2FA forzado a nivel de instancia rechazando la cuenta nueva | Sin ajuste de MFA en la tabla `settings`, sin variables de entorno de MFA |
| Bloqueadores de rastreo del navegador | Plausible, ya que fallaba en dos navegadores distintos, pero desmentido por el error real de consola |
| Onboarding incompleto colgando el modal de encuesta | `settings` y `personalizationAnswers` estaban efectivamente en NULL y se rellenaron por precaución, pero no era la causa |

Tres de estas hipótesis eran razonables y coherentes con todos los síntomas disponibles en ese momento. Ninguna era correcta. El avance llegó únicamente con evidencia del lado del cliente.

---

## 5. Causa raíz

n8n resuelve un proyecto personal con una consulta equivalente a:

```sql
SELECT * FROM project WHERE type = 'personal' AND creatorId = :userId
```

Tras la reconstrucción de la cuenta, el estado de la base de datos era:

```
project.creatorId        = <usuario eliminado>   provoca el 404
project_relation.userId  = <usuario nuevo>       reparado
shared_workflow          = 24 workflows correctamente asociados al proyecto
```

La relación era correcta. El `creatorId` no. El frontend lanza esa llamada dentro de un `Promise.all` durante su inicialización (`init.ts` hacia `router.ts`), de modo que el 404 provocaba una excepción, abortaba el arranque y redirigía a la pantalla de login.

El motivo de que costara tanto aislarlo: el fallo se presentaba como un problema de *autenticación* cuando en realidad era de *integridad referencial*. La autenticación funcionó correctamente todo el tiempo, lo que se confirmó con peticiones autenticadas exitosas repetidas desde línea de comandos.

---

## 6. Resolución

En lugar de parchear la única columna que desbloqueaba el síntoma, se recorrieron todas las tablas de forma programática buscando referencias al id del usuario eliminado, y se reasignaron.

| Tabla y columna | Filas corregidas |
|---|---|
| `project.creatorId` | 1 (la crítica) |
| `workflow_publish_history.userId` | 722 |
| `user_api_keys.userId` | 2 |
| `chat_hub_sessions.ownerId` | 1 |
| **Total** | **726** |

Corregir `user_api_keys` tuvo un beneficio secundario: restauró las API keys usadas tanto por una integración externa como por workflows internos que llaman a la propia API de n8n, todos ellos fallando en silencio desde hacía horas.

Verificación posterior:

```
POST /rest/login              200
GET  /rest/projects/personal  200   (antes 404)
GET  /rest/workflows          200   (24 workflows)
GET  /rest/projects           200
GET  /rest/credentials        200
```

---

## 7. Problemas de configuración detectados de paso

Sin relación con la causa raíz, pero descubiertos y corregidos durante la investigación:

- `N8N_PROXY_HOPS=1`. El rate limiter no confiaba en el reverse proxy, generando `ERR_ERL_UNEXPECTED_X_FORWARDED_FOR` en los intentos de login
- `N8N_PROTOCOL=https` y `N8N_HOST`. Ninguno estaba definido, solo estaba `WEBHOOK_URL`
- `N8N_DIAGNOSTICS_ENABLED=false`. Telemetría desactivada

---

## 8. Si estás sufriendo este mismo problema

Síntomas que coinciden con este incidente:

- El login parece funcionar y acto seguido la página vuelve a la pantalla de acceso
- Los logs del servidor no muestran ningún error de autenticación
- La URL acumula un redirect codificado del tipo `signin?redirect=%252Fhome%252Fworkflows`
- La consola del navegador muestra `Could not find a personal project for this user`

Comprobaciones, en orden:

1. Abre la consola del navegador. Si la petición que falla es `GET /rest/projects/personal` con 404, es este mismo caso
2. Consulta la base de datos: `SELECT id, type, creatorId FROM project;`
3. Compara `creatorId` con el id de la cuenta con la que realmente inicias sesión: `SELECT id, email FROM user;`
4. Si no coinciden, esa es la causa

Antes de reparar, detén n8n, y recorre todas las tablas buscando el id de usuario obsoleto en lugar de corregir solo la que desbloquea el síntoma.

Si `n8n user-management:reset` te deja sin poder llegar al asistente de configuración, la causa probable es el 2FA residual en la fila existente. Limpia únicamente `mfaEnabled`, `mfaSecret` y `mfaRecoveryCodes`. No elimines la fila de usuario, que es exactamente lo que provocó el daño referencial descrito aquí.

---

## 9. Lecciones

**Un mensaje de error temprano contenía la respuesta.** El segundo reset fallido indicaba literalmente `matching: { "type": "personal", "creatorId": ... }`. Se leyó como un obstáculo a sortear en vez de como una descripción del modelo de datos. Los mensajes de error de las herramientas describen su lógica interna y deben leerse como documentación.

**Eliminar es peor que corregir.** Borrar la fila de usuario completa fue más destructivo de lo que la situación requería y causó el daño referencial. Limpiar los campos concretos de 2FA que bloqueaban el reset oficial habría bastado.

**Reparar por integridad referencial, no por síntoma.** Un barrido programático encontró 726 referencias obsoletas donde la reparación manual había corregido 1.

**El servidor tiene puntos ciegos.** Se agotaron logs, base de datos y código fuente desplegado sin encontrar la causa. Un vistazo a la consola del navegador la localizó en segundos. Cuando el backend está verificado como sano, conviene pasar al cliente pronto.

**Distinguir lo verificado de lo asumido.** Afirmaciones como "los workflows están intactos" se sostuvieron con conteos reales en cada paso en lugar de suposiciones, lo que permitió acotar el alcance real del daño mientras el resto del diagnóstico seguía siendo incierto.

---

La versión en inglés de este documento está en [README.md](README.md).

Los valores sensibles (direcciones, tokens, credenciales) se han omitido deliberadamente.

# Referencia de la API de la Báscula

Todo lo que TigerScale expone en la red local: cada punto final HTTP, el WebSocket en ambas direcciones, qué significa cada campo y dónde están las lagunas. Este es el documento que se debe entregar a cualquier cosa que hable directamente con una báscula — Tiger Studio Manager, un panel de control, un script.

Dos compañeros, deliberadamente separados:

- [`docs/TELEMETRY.md`](TELEMETRY_ES.md) — lo que la báscula *envía a la nube*, el latido de Firestore. Ese es el canal que lee Studio Manager cuando la báscula no está en la misma red.
- [`docs/CLOUD.md`](CLOUD_ES.md) — la pregunta de privacidad: qué abandona el dispositivo y cómo borrarlo.

Este archivo cubre los canales locales. Donde un campo existe en más de un lugar, el nombre y el significado son los mismos.

## ¿Qué canal para qué

Tres canales, y la división es una regla en lugar de un accidente:

| Canal | Transporta | Lea cuando |
|---|---|---|
| Latido de Firestore | estado lento: batería, energía, señal, firmware, cuenta, contadores | lo necesita desde cualquier lugar, sin estar en la red de la báscula |
| WebSocket `/ws` | estado rápido: peso en tiempo real, etiqueta siendo leída, fase del flujo de trabajo | lo necesita ahora, y está en la LAN |
| HTTP | comandos, y una instantánea completa | quiere *hacer algo*, o resincronizar a demanda |

`GET /api/status` es la instantánea completa en una sola solicitud — incluyendo los campos de energía, por lo que responde por sí mismo sin un socket. Ese es su propósito.

## Antes de nada: no hay autenticación

`Access-Control-Allow-Origin: *` se envía en cada respuesta, deliberadamente sin un `Access-Control-Allow-Headers` y sin un manejador `OPTIONS`. Las solicitudes simples y sus respuestas funcionan — un cliente del navegador puede leer `/api/status`, `/api/ping`, y el resultado de `POST /api/tare`. Cualquier cosa que necesite un preflight, lo que significa cada POST JSON como `/api/calibration` o `/api/firebase/token`, aún falla. En una API sin autenticación esa asimetría es la única barrera disponible.

**Todos los puntos finales que siguen están sin autenticación.** Cualquiera que pueda llegar a la IP de la báscula puede leer su estado, desencadenar un tare, cambiar su calibración, registrarlo fuera de su cuenta o reiniciarla en fábrica. Esa es una simplificación deliberada para un dispositivo en una red doméstica, no un error — pero una báscula no pertenece a una red no confiable sin un VLAN delante.

La única excepción es la vista en vivo en el puerto 81, que pide el código de seis caracteres desde Configuración → LAN.

## Base

```
http://<scale-ip>/          or  http://tigerscale-XXXX.local/
ws://<scale-ip>/ws
http://<scale-ip>:81/       live view, code-protected
```

`mdns` en `/api/status` da el nombre de host; los últimos cuatro caracteres provienen de la MAC. La IP no es estable — resuelva el nombre mDNS en lugar de codificar una dirección.

## Un servidor

El firmware usaba dos implementaciones de servidor HTTP, solo una de ellas compilada. El muerto ha sido eliminado — con él, 30 registros de rutas duplicados, seis funciones auxiliares sin llamador restante, y unos 750 líneas que nunca podrían ejecutarse.

Dos de sus rutas no eran duplicados: `GET /api/session` y `GET /api/history` existían **solo** allí, y por lo tanto respondían 404 en cada báscula jamás enviada. Fueron portadas al servidor en vivo en lugar de eliminarse con el resto.

## Lectura del estado

### `GET /api/status`

La lectura principal. Un objeto JSON, sin parámetros. Respuesta verificada:

```json
{
  "weight": 7,
  "rawWeight": 7.409999847,
  "uid": "", "uid_hex": "", "uid2": "", "uid2_hex": "",
  "uid_left": "", "uid_right": "", "uid_twin": "",
  "wifi": "Stargate",
  "ip": "192.168.20.143",
  "mdns": "tigerscale-XXXX.local",
  "cloud": true,
  "firebaseConfigured": true,
  "firebaseAuth": true,
  "firebaseEmail": "benoit@atome3d.com",
  "firebaseDisplayName": "Open Maker",
  "calibrationFactor": 461.0623474,
  "servoEnabled": false,
  "uptime_ms": 71756188,
  "uptime_s": 71756,
  "fw_version": "3.6.0",
  "fw_git_sha": "dev",
  "ota_status": "idle",
  "ota_progress": 0,
  "ota_latest": "3.6.0",
  "scaleStatus": "idle",
  "wfPhase": "",
  "wfSlope": 0,
  "containerWeight": -1,
  "db_updating": false,
  "db_brands": 120,
  "db_materials": 113,
  "db_checked_s": -1
}
```

| Campo | Significado |
|---|---|
| `peso` | Peso mostrado en gramos, redondeado y filtrado. Este es el número en la pantalla. |
| `pesoBruto` | Lectura sin filtrar de la celda de carga. Solo diagnóstico — se mueve constantemente. |
| `uid` / `uid_hex` | UID del etiqueta de la bobina en la plataforma, vacío cuando no hay ninguno |
| `uid2` / `uid2_hex` | Segundo etiqueta, para una bobina etiquetada en par |
| `uid_izquierda` / `uid_derecha` | UID por lector, qué físico lector vio qué |
| `uid_gemelo` | El etiqueta emparejada resuelto desde el inventario |
| `wifi` | **SSID actualmente asociado** |
| `ip`, `mdns` | Identidad de red |
| `cloud` | WiFi está conectado |
| `firebaseConfigurado` / `firebaseAutenticado` | Una cuenta está configurada / autenticada |
| `firebaseEmail` / `firebaseDisplayName` | La cuenta autenticada |
| `factorCalibración` | Factor de escala de la celda de carga |
| `pesoContenedor` | Peso de la bobina vacía en gramos para el etiqueta en la plataforma, **`-1` cuando es desconocido** — no cero |
| `estadoBáscula` | `inactivo`, `escaneando`, `estable`, `enviando`, `éxito`, `error` |
| `fase_wf` / `pendiente_wf` | Internos del flujo de pesaje |
| `versión_fw` / `sha_git_fw` | Firmware en ejecución |
| `estado_ota` / `progreso_ota` / `versión_otaMasReciente` | Estado de actualización, 0-100, y la versión más reciente vista |
| `tiempoDeFuncionamiento_ms` / `tiempoDeFuncionamiento_s` | Desde el encendido |
| `db_*` | Base de datos local de marcas/materiales: tamaños, si una sincronización está en ejecución, edad de la última verificación en segundos (`-1` = nunca) |

**`pesoContenedor: -1` significa desconocido, no vacío.** Un panel que lo renderice como un número mostrará "-1 g".

### Otras lecturas

| Punto final | Devuelve |
|---|---|
| `GET /api/ping` | el texto literal `pong`. Solo para verificar estado. |
| `GET /api/ota/check` | `{"success":true,"current":"3.6.0","current_sha":"dev","latest":"3.6.0","latest_sha":"7c3f…","latest_littlefs_sha":"6c7f…","update_available":false}` — fuerza una búsqueda de manifiesto |
| `GET /api/firebase/status` | `{"configurado":true,"autenticado":true,"email":"…"}` |
| `GET /api/hw/config` | `{"rfidCount":2,"rfidSide":"derecha","motorConnected":false,"motorEnabled":false,"motorSpeed":0}` |
| `GET /api/rfid/test` | `{"activo":false,"lector_izquierda":true,"lector_derecha":true,"uid_izquierda":null,"uid_derecha":null}` — estado del lector |
| `GET /api/session` | la sesión de pesaje en curso: `sessionId`, `faseWorkflow`, `faseEnvío`, `contadorEnvío`, `pesoActual`, `pesoContenedor`, `pesoContenedorRecuperado`, `pesoPico`, `pendienteActual`, los cinco UIDs, `pendienteTareAutomática`, `bloqueadoRfid`, los contadores de vida, y un `últimoEnvío` con la medición anterior |
| `GET /api/history` | un anillo de eventos en RAM como array JSON: `ms`, `edadMs`, `tipo` (`tare_automática`, `encendido_listo`, …), `mensaje`, `uid1`, `uid2`, `peso`, `fase`. No sobrevive a nada — se limpia al reiniciar. |
| `GET /api/logs` | las últimas ~80 líneas de registro en RAM. **La única forma de leer registros una vez que el cable USB está fuera.** |
| `GET /` , `/style.css`, `/app.js` | la interfaz web propia de la báscula |
| `GET /live` | redirige a la vista en vivo en el puerto 81 |

### La pantalla

Dos formas de entrada, para dos tareas.

**Puerto 80, para comprobar una pantalla.** Sin código, como cada otra ruta aquí.

| Punto final | Devuelve |
|---|---|
| `GET /screen.bmp` | lo último que LVGL dibujó, 480x320, como BMP de 24 bits (alrededor de 450 KB) |
| `GET /api/tap?x=&y=` | presiona la pantalla en ese punto; agrega `x2`/`y2` para un deslizamiento. Responde una vez que la presión fue tomada y la pantalla terminó de cambiar, por lo que se puede encadenar con `/screen.bmp` sin retraso |
| `GET /screen.ver` | cuatro bytes del contador de marco, para consultar de forma económica |
| `GET /screen` | la imagen en una página: haga clic para presionar, arrastre para deslizar |

Una presión en una pantalla oscura la despierta y es absorbida, como lo sería con un dedo.

**Puerto 81, para verla.** El flujo en tiempo real detrás de Configuración > LAN > Vista en vivo, protegido por el código de seis caracteres de esa pantalla.

| Punto final | Devuelve |
|---|---|
| `GET :81/stream?c=CODE` | `200`, un flujo binario fragmentado de marcos, o `403` para un código incorrecto. Dos visualizadores máximo. |
| `POST :81/tap?c=CODE&x=&y=` | `204`; una presión (el método no se verifica) |
| `GET :81/` o `GET :81/CODE` | la página del visualizador |

`/stream` y su `403` llevan `Access-Control-Allow-Origin: *`, por lo que una página puede obtener un marco directamente. El formato de mensaje, y cómo tomar una captura de pantalla de un solo marco, están en [TELEMETRY.md](TELEMETRY_ES.md#lan-live-view-the-screen-itself).

**Desde la cuenta, sin pedirle a nadie.** El latido publica `lan_live`, `lan_access_code` y `lan_url` (`http://<ip>:81`, o nulo cuando la vista en vivo está apagada o no hay dirección) en `users/{uid}/scales/{mac}`, por lo que las aplicaciones propias del propietario pueden abrir el flujo sin escribir código. `lan_access_code` es un secreto: ese documento es solo para el propietario, y el código nunca debe copiarse en ningún lugar compartido, ni en un registro.

## Comandar la báscula

Todos son `POST`. Los cuerpos son JSON a menos que se indique lo contrario.

| Endpoint | Body | Efecto |
|---|---|---|
| `/api/tare` | none | Pone a cero la balanza y limpia la sesión actual de etiqueta. Mantiene el desplazamiento. |
| `/api/calibration` | `{"factor": 461.06}` (o `{"value": …}`) | Establece el factor de la celda de carga |
| `/api/weight` | ver abajo | Envía un peso a la nube para la etiqueta actual |
| `/api/push-weight` | mismo | Mismo manejador, conservado para la interfaz web |
| `/api/workflow/stop` | none | Abandona el flujo de pesaje en ejecución |
| `/api/rfid/test` | `{"stop":true}` / `{"reset":true}` | Conduce la prueba interna del lector |
| `/api/hw/config` | `{"rfidCount":2,"rfidSide":"right"}` | Cantidad de lectores y cual es el lado primario |
| `/api/servo-toggle`, `/api/servo/test` | — | No existe motor en V3; estos son para la variante motorizada |
| `/api/update-db` | none | Dispara una sincronización de la base de datos de marca/material |
| `/api/ota/check` | — | ver lecturas |
| `/api/ota/update` | `{"firmware_url":"…","littlefs_url":"…"}` | Inicia una actualización por aire |
| `/api/firebase/auth` | `{"email":"…","password":"…"}` | Inicia sesión |
| `/api/firebase/token` | `{"idToken":"…","refreshToken":"…","email":"…","displayName":"…"}` | Asocia una sesión existente |
| `/api/firebase/logout` | none | Limpia la sesión almacenada |
| `/api/set-apikey` | `{"apiKey":"…"}` | Sobrescribe la clave web de Firebase |
| `/api/logs` `DELETE`, `/api/logs/clear` | none | Vacía el anillo de logs |
| `/api/reset-wifi` | none | **Destrutivo.** Elimina las credenciales de WiFi almacenadas. |
| `/api/factory-reset` | none | **Destrutivo.** Elimina la partición NVS completa — WiFi, cuenta, calibración, idioma. |

Los dos destrutivos no requieren confirmación ni autenticación. Cualquier cosa que los expone en una interfaz debe confirmar por su cuenta.

## El WebSocket

```
ws://<scale-ip>/ws
```

### Salida: la balanza hacia ti

Un objeto JSON por mensaje. **Comprimido en delta**: un campo aparece solo cuando su valor cambió. Una instantánea completa llega al conectar y cada 30 segundos; entre esos, se esperan mensajes que llevan dos o tres claves.

**Un campo ausente significa inalterado, nunca nulo.** El cliente debe mantener el valor anterior.

Campos en tiempo real, enviados siempre que cambien:

`peso`, `pesoNeto`, `pesoContenedor`, `uid`, `uid2`, `uid_izquierda`, `uid_derecha`, `uid_gemelo`, `estadoBalanza`, `marca`, `material`, `color`, `nube`, `firebaseAuth`, `db_actualizando`, `bateria_presente`, `bateria_porcentaje`, `está_cargando`, `fuente_poder`, `estado_poder`, `senal_wifi_dbm`.

Campos solo en instantánea, al conectar y cada 30 s: `firebaseConfigured`, `firebaseEmail`, `firebaseDisplayName`, `factor_calibracion`, `tiempo_encendido_s`, `versión_fw`, `db_ok`, `db_verificado_s`.

Los campos de poder llevan el mismo significado que en [`TELEMETRY.md`](TELEMETRY_ES.md#poder_y_batería), con una diferencia deliberada: JSON no tiene nulo tipado en esta codificación, así que **`bateria_porcentaje` es `-1`, no nulo, cuando no haya celda instalada**, y `senal_wifi_dbm` es `0` cuando no esté conectado. `bateria_presente` es lo que debes ramificar.

`estadoBalanza` en el WebSocket es más rico que en `/api/status`: durante un envío lleva un contador, como `escaneando:3`, `estable:2` o un número puro.

### Entrada: tú hacia la balanza

**Nada.** El manejador analiza el JSON entrante y lo descarta — el código lo dice: *"No se necesitan tipos de comandos WS actualmente"*. No hay comandos, suscripciones ni confirmaciones.

Para mandar un comando a la balanza, usa los endpoints HTTP anteriores. El WebSocket es un flujo unidireccional de estado.

## Cobertura: donde vive cada valor

| Lo que quieres | `/api/status` | WebSocket | Firestore |
|---|---|---|---|
| Peso | sí | sí | `current_weight_g` |
| Peso neto | no | `pesoNeto` | mediante la última medición |
| Peso del contenedor | sí (`-1` = desconocido) | sí | `container_weight` en el documento de la etiqueta |
| UID de etiqueta | sí, los cinco | sí | `current_spool_uid_1` / `_2` |
| Marca / material / color | no | sí | no |
| **USB o batería** | **no** | `fuente_poder` | `fuente_poder` |
| **Cargando** | **no** | `está_cargando` | `está_cargando` |
| **Porcentaje de carga** | **no** | `bateria_porcentaje` | `bateria_porcentaje` |
| **Batería instalada** | **no** | `bateria_presente` | `bateria_presente` |
| **Estado de pantalla** | **no** | `estado_poder` | `estado_poder` |
| SSID WiFi | `wifi` | no | no |
| **Señal WiFi** | **no** | `senal_wifi_dbm` | `senal_wifi_dbm` |
| Versión de firmware | sí | sí | `fw_version` |
| Cuenta | sí | sí | `display_name` |
| Calibración | sí | sí | `factor_calibracion` |
| Contadores, estadísticas de sesión | no | no | sí |
| Vista en LAN: activada, código, URL | no | no | `lan_live`, `lan_access_code`, `lan_url` |

## Brechas conocidas

Estas son reales, actuales y merecen decidirse antes de trabajar alrededor.

- **El WebSocket no puede ser mandado.** Un cliente que quiera tarear tiene que caerse a HTTP, lo que significa mantener dos conexiones para un trabajo.
- **No hay autenticación en ningún lado**, incluyendo en los dos endpoints destructivos. Una página web no puede *leer* lo que no está permitido, pero siempre ha podido *enviar* un POST simple — `/api/tare`, `/api/reset-wifi`, `/api/factory-reset` no llevan body y por tanto son alcanzables desde cualquier sitio que el dueño visite. CORS no cambia eso; solo un token de acceso lo haría.

## Donde vive esto en el firmware

| Lo que | Donde |
|---|---|
| Rutas HTTP | §19 Servidor web, el bloque `server.on(...)` |
| Eventos WebSocket | §13, `onWsEvent()` |
| Carga WebSocket | §15, `buildWsFrame()` |
| Carga Firestore | `sendScaleHeartbeat()` |
| Batería y estado de poder | `gBattery`, actualizado por `pollBatteryState()` |

Los números de línea están en [`CODEMAP.md`](../CODEMAP_ES.md), que se regenera — no los copies aquí.

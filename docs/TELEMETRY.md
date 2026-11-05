# Contrato de telemetría

Lo que TigerScale reporta sobre sí mismo, por qué canal, con qué frecuencia y qué significa cada valor. Este es el referente de integración — para Tiger Studio Manager, para un panel de control, para cualquier cosa que lea el estado de una báscula.

[`docs/API.md`](API_ES.md) responde la otra mitad: los puntos finales HTTP propios de la báscula y su WebSocket, para cualquier cosa en la misma red que el dispositivo.
[`docs/CLOUD.md`](CLOUD_ES.md) responde la pregunta de privacidad: qué abandona el dispositivo y cómo borrarlo. Este archivo responde la pregunta de integración: los nombres exactos de los campos y su semántica. Cuando el firmware cambia un campo, este archivo cambia en el mismo commit.

## Dos canales, audiencias diferentes

| Canal | Quién lo lee | Transporte |
|---|---|---|
| Latido de Firestore | Tiger Studio Manager, la aplicación móvil, cualquier panel en la nube | `documents:commit` a `users/{uid}/scales/{mac}` |
| WebSocket `/ws` | la interfaz web propia de la báscula, en la LAN | `ws://<scale-ip>/ws`, no autenticado |

Transportan los mismos campos de batería y potencia bajo los mismos nombres, por lo que un valor significa lo mismo cualquiera que sea el lado desde el que lo leas. Todo lo que sigue aplica a ambos a menos que una fila indique lo contrario.

**¿Qué canal leer y por qué.** La división es deliberada, no histórica:

- **Firestore lleva lo que cambia lentamente y debe ser accesible desde cualquier lugar** — batería, potencia, señal, firmware, cuenta, contadores. Una báscula solo tiene que estar conectada; el lector no tiene que estar en la misma red y no tiene que mantener un socket abierto por báscula.
- **El WebSocket lleva lo que cambia rápidamente y es necesario de inmediato** — el peso en vivo, la etiqueta que se está leyendo, la fase del flujo de trabajo.

Tiger Studio Manager ya funciona así: lee `battery_percent`, `is_charging`, `power_source`, `power_state` y `wifi_signal_dbm` del documento de Firestore, y `weight`, `netWeight`, `containerWeight`, `scaleStatus`, `brand`, `material` y `color` desde el socket.

Los campos de potencia también están en el WebSocket, y eso no es redundancia por sí mismo: la interfaz web propia de la báscula no tiene acceso a Firestore en absoluto, y el socket es su único canal.

## Frecuencia

El latido es **cada 30 segundos mientras la pantalla esté encendida, cada 5 minutos una vez que la luz de fondo se apague**. Una transición entre esos dos estados envía un latido inmediatamente, por lo que entrar o salir del modo de espera se muestra de inmediato en lugar de al siguiente tick.

**Un evento de cable envía uno también.** El momento en que VBUS aparece o desaparece, la báscula escribe, dentro de unos segundos, en ambas direcciones y también en modo de espera — sin ello, desconectar una báscula con la pantalla apagada pasaría desapercibido durante cinco minutos. El disparador es VBUS y no `is_charging`, deliberadamente: un cable no puede oscilar, un regulador de carga sí.

Ese latido es un latido normal, por lo que lleva **todo el bloque de potencia**, no solo los dos campos que cambian: `battery_present`, `battery_percent`, `is_charging`, `power_source` y `power_state` se escriben en cada latido, nunca como deltas. El porcentaje que contiene es recién leído — un evento de cable reinicia el temporizador de 30 segundos propio del coulómetro y el nuevo valor se toma en la misma pasada, antes de que el latido incluso se programe. Por lo tanto, enchufar actualiza el nivel de carga de inmediato en lugar de mostrar una figura de cinco minutos atrás junto a un estado de carga en vivo.

Esa segunda frecuencia es la que rompe los paneles de control ingenuos. **No trates el silencio como offline sin leer `power_state` primero:**

| Último `power_state` conocido | Espacio esperado | Llámalo offline después |
|---|---|---|
| `active` | 30 s | ~90 s |
| `screen_off` | 5 min | ~11 min |

Una báscula en modo de espera está completamente despierta — CPU en ejecución, WiFi asociado, HTTP respondiendo. Solo su luz de fondo está apagada. No está dormida y no ha ido a ningún lado.

## Potencia y batería

Estos seis campos se escriben en **cada** latido, nunca como deltas, porque son lo que un operador mira primero y un valor obsoleto es peor que ninguno.

| Campo | Tipo | Significado |
|---|---|---|
| `battery_present` | bool | Una celda está instalada y el PMIC puede verla. La batería es opcional en esta hardware. |
| `battery_percent` | int 0-100, o **null** | Nivel de carga desde el coulómetro AXP2101. `null` cuando no hay celda instalada. |
| `is_charging` | bool, o **null** | Corriente realmente fluye hacia la celda. `null` cuando no hay celda instalada. |
| `power_source` | `"usb"` \| `"battery"` | Si hay alimentación externa presente. |
| `power_state` | `"active"` \| `"screen_off"` | Estado de la luz de fondo. No es un estado de sueño — véase arriba. |
| `wifi_signal_dbm` | int, o null | RSSI. `null` cuando no se puede leer. |

Tres distinciones que la interfaz tiene que entender correctamente, porque colapsar cualquiera de ellas produce una visualización que miente:

- **`battery_percent: null` no es `0`.** Null significa que no hay batería instalada; cero significa que hay una instalada y está descargada. Muestra null como "Solo USB" o un icono de batería ausente, nunca como un indicador vacío.
- **`power_source: "usb"` no significa que esté cargando.** Una celda llena en un cable vivo reporta `"usb"` con `is_charging: false`. Eso es el comportamiento de la teléfono: el conector se retira cuando la carga se detiene, no cuando el cable se desconecta. El valor proviene del estado máquina del cargador, no de la presencia de VBUS.
- **`power_state: "screen_off"` no es offline.** Es una báscula viva con su luz de fondo apagada, aún respondiendo en la LAN.

**La carga tiene prioridad sobre la baja.** Por debajo del 20% una batería normalmente se muestra como una alerta — indicador rojo, número rojo. Esta regla se suspende mientras `is_charging` es verdadero: una celda al 15% en un cargador vivo no es un problema, es una celda que se está solucionando, y pintarla roja dice lo contrario. El icono propio del firmware hace exactamente esto, por lo que un panel de control que omita la precedencia discrepa con la pantalla frente al usuario. Orden de aplicación: carga primero, luego baja, luego normal.

| Condición | Color |
|---|---|
| `is_charging` verdadero | color de carga (verde), cualquiera que sea el nivel |
| `battery_percent` <= 20 | color de alerta (rojo) |
| de lo contrario | neutro |

Los estados que merecen ser mostrados distintamente:

| `battery_present` | `power_source` | `is_charging` | Mostrar |
|---|---|---|---|
| falso | usb | nulo | En USB, sin batería |
| verdadero | usb | verdadero | Cargando, N% |
| verdadero | usb | falso | Cargado, en USB, N% |
| verdadero | battery | falso | En batería, N% |

## Señal WiFi

`wifi_signal_dbm` sale bruto en ambos canales — `WiFi.RSSI()`, un valor negativo en dBm, o `null` (0 en el WebSocket) cuando no esté conectado. No lleva ninguna categoría de calidad, por lo que cualquier cosa que muestre barras o una etiqueta "bueno / débil" está decidiendo eso mismo.

**La báscula canónica es cuatro estados, 0 a 3.** No cinco, y la razón es el dibujo, no la aritmética: el icono de señal del firmware está hecho de exactamente tres piezas — un arco exterior, un arco interior y el punto — por lo que puede expresar "nada alcanzado" más tres grados y nada más. Una versión de cinco niveles fue construida primero y tuvo que cortar a través del medio del arco exterior, lo que dejó su ápice apagado mientras sus hombros estaban iluminados y leían como una onda cortada en la parte superior.

El firmware dibuja esto en exactamente un lugar: la barra de estado de la pantalla de pesaje y cada fila del selector de red llaman al mismo constructor, por lo que una señal lee idénticamente dondequiera que aparezca. Reproducir los mismos cuatro estados mantiene eso verdadero también en Studio Manager.

| Nivel | RSSI | Icono |
|---|---|---|
| 0 | -100 a -81 dBm | todo el icono apagado |
| 1 | -80 a -61 dBm | punto |
| 2 | -60 a -41 dBm | punto + arco interior |
| 3 | -40 dBm y más alto | completamente iluminado, dibujado como un icono sólido |

La aritmética usa truncamiento de enteros, lo que fija las bandas a 20 dBm de distancia. Una versión flotante desplaza cada límite, así que mantén el piso:

```js
const level = rssi == null || rssi === 0 ? null
  : Math.floor((Math.min(-40, Math.max(-100, rssi)) + 100) * 3 / 60);
```

**Una señal, un número — pero no una sola pregunta.** La barra de estado reporta el enlace realmente en uso; el selector de red reporta lo que cada punto de acceso está transmitiendo. En una red con varios puntos de acceso compartiendo un SSID, estos diferencias son legítimas, y una báscula que lea -79 dBm mientras el selector muestra un AP -35 dBm del mismo nombre no es un error en ninguna de las pantallas: significa que la báscula está asociada al radio equivocado. `wifiBeginBestAp()` es lo que previene eso, forzando un escaneo completo de canal ordenado por señal en lugar del comportamiento por defecto del driver de primer coincidencia.

## Todo lo demás en el latido

**Enviado en cada latido**, junto con los campos de potencia anteriores:

`workflow_phase`, `send_phase`, `current_weight_g`, `session_id`, `sessions_started`, `send_ok_count`, `send_fail_count`, `rfid_read_ok_count`, `rfid_read_fail_count`, `auto_tare_count`, `workflow_reset_count`, `last_measurement_uid_1`, `last_measurement_uid_2`, `last_measurement_weight_g`, `last_measurement_status`, y los tres campos de vista en LAN siguientes.

**Enviado solo cuando el valor cambió** (y en cada snapshot completo):
`current_spool_uid_1`, `current_spool_uid_2`, `calibration_factor`, `ip_address`.

**Enviado solo en un snapshot completo** — al encender y cuando se adjunta una cuenta:
`fw_version`, `mdns_hostname`, `mac`, `hardware_revision`, `display_name`.

Un consumidor debe tratar un campo ausente como "no cambió", nunca como "nulo". Solo un `nullValue` explícito significa nulo.

Los timestamps están establecidos por el servidor. La placa lleva un reloj PCF85063 alimentado por batería que el firmware aún no lee, y deliberadamente no corre NTP para esto.

| Campo | Tipo | Significado |
|---|---|---|
| `lan_live` | bool | vista en vivo está activa (predeterminado) |
| `lan_access_code` | string | seis caracteres, letras mayúsculas y minúsculas y dígitos, sin `l`, `I`, `O`, `0` o `1`, p. ej. `J6fxda`. **Se compara de forma sensible a mayúsculas y minúsculas.** |
| `lan_url` | string o null | `http://<ip>:81`, la base del flujo, mientras la vista en vivo esté activa y la escala tenga una dirección; null en otro caso |

- **`lan_access_code` es un secreto.** Solo el propietario puede leer este documento.
  Nunca lo copies, ni un URL que lo lleve, en ningún lugar donde otro usuario pueda leerlo: un inventario compartido o público, una vista de un amigo, una exportación, un registro, análisis. La escala en sí nunca registra este código.
- **Un marco es una captura de pantalla.** `GET {lan_url}/stream?c={lan_access_code}` responde `200` con un `application/octet-stream` fragmentado, o `403` para un código incorrecto. Ambos llevan `Access-Control-Allow-Origin: *`. Lee hasta el primer `FRAME_END`, luego cierra: un nuevo visualizador recibe toda la pantalla primero, dentro de aproximadamente medio segundo.
- **Máximo dos visualizadores.** La escala sirve dos flujos a la vez, y un navegador en Ajustes > LAN > Vista en vivo es uno de ellos. Cierra la conexión tan pronto como el marco esté en.
- **No hay flujo mientras la vista en vivo esté apagada**, mientras la escala no tenga WiFi, o mientras su propia pantalla de emparejamiento esté activa; `lan_url` siendo null cubre los primeros dos.
- **Una pantalla dormida es negra, y el marco lo dice fielmente.** Cuando `power_state` es `"screen_off"`, primero `POST {lan_url}/tap?c={code}&x=240&y=160`: en una pantalla oscura, una presión solo despierta y es absorbida, exactamente como una dedo lo haría, por lo que no abre nada. Lee el marco unos segundos y medio después.
  La escala vuelve a dormir sola después de su retardo habitual.

El flujo es una secuencia de mensajes, little-endian, el primer byte dando el tipo:

| Byte | Mensaje | Luego |
|---|---|---|
| `1` | HELLO | ancho (u16, 480), alto (u16, 320), ancho de banda (u16, 16) |
| `2` | FRAME_BEGIN | nada |
| `5` | BAND | índice de banda (u8, 0-29), longitud (u16), esa cantidad de bytes de píxeles codificados |
| `4` | FRAME_END | nada: el marco está completo, dibújalo ahora |
| `3` | PING | nada, cada 5 s |

Una banda es 16 columnas de la pantalla. Sus píxeles son RGB565, ordenados hacia abajo en una columna y luego de derecha a izquierda en la banda: el píxel `i` de la banda `n` cae en `x = n*16 + 15 - floor(i / 320)`, `y = i % 320`. La codificación es una secuencia de tokens: `t < 0x80` es `t+1` píxeles literales (2 bytes cada uno) que siguen; `0x80 <= t < 0xFF` es un píxel repetido `t-0x80+2` veces; `t == 0xFF` es un contador u16, luego un píxel repetido esa cantidad de veces. `live_page.h` en el firmware es un decodificador completo en cincuenta líneas de JavaScript.

## El marco WebSocket

`ws://<scale-ip>/ws` lleva la vista en vivo del mismo dispositivo para su propia interfaz web: un objeto JSON por mensaje, comprimido por diferencia de la misma manera, con una instantánea completa al conectar y cada 30 s.

Campos en vivo: `weight`, `netWeight`, `containerWeight`, `uid`, `uid2`, `uid_left`, `uid_right`, `uid_twin`, `scaleStatus`, `brand`, `material`, `color`, `cloud`, `firebaseAuth`, `db_updating`, más los cinco campos de estado de energía `battery_present`, `battery_percent`, `is_charging`, `power_source`, `power_state`.

Campos solo en instantáneas: `firebaseConfigured`, `firebaseEmail`, `firebaseDisplayName`, `calibrationFactor`, `uptime_s`, `fw_version`, `db_ok`, `db_checked_s`.

Una diferencia de Firestore, y es deliberada: JSON no tiene nulos tipados en esta codificación de diferencias, así que **`battery_percent` es `-1`, no null, cuando no hay celda instalada**. `battery_present` es lo que debes ramificar en cualquier caso.

## Donde esto se encuentra en el firmware

| Lo que | Función |
|---|---|
| Leyendo el PMIC | `axpChargeState()`, `axpBatteryLevel()` |
| Propietario del estado | `gBattery`, actualizado por `pollBatteryState()` desde el bucle principal |
| Carga Firestore | `sendScaleHeartbeat()` |
| Carga WebSocket | `buildWsFrame()` |

`pollBatteryState()` se ejecuta desde `loop()` en lugar de desde un constructor de pantalla a propósito: una versión anterior mantenía el estado en `lvglUpdateMainScreen()`, que solo corre mientras la pantalla de pesaje está mostrada, así que la telemetría se congelaba al momento que alguien abría Ajustes.

Los números de línea están en [`CODEMAP.md`](../CODEMAP_ES.md), que se regenera — no los copies aquí.

## Informe para el agente del Studio Manager

Autónomo; pégalo tal cual.

> Estás mostrando el estado de una escala de filamento TigerScale V3. Su telemetría es un documento Firestore en `users/{uid}/scales/{mac}`, reescrito por el dispositivo en un latido.
>
> **Cadencia.** Cada 30 segundos mientras su pantalla esté encendida; cada 5 minutos una vez que la luz de fondo se apague. El campo `power_state` te dice cuál régimen estás en: `"active"` o `"screen_off"`. Entrar o salir del modo de espera envía un latido inmediato.
>
> **Nunca marques una escala como fuera de línea solo por silencio.** Compara la brecha contra el régimen: sobre ~90 segundos cuando `power_state` es `"active"`, sobre ~11 minutos cuando es `"screen_off"`. Una escala en `"screen_off"` está completamente despierta con WiFi asociado — solo su luz de fondo está apagada. Mostrarla como desconectada es incorrecto.
>
> **Campos de energía y batería**, presentes en cada latido:
>
> - `battery_present` (bool) — la batería es una parte opcional; muchas unidades no la tienen.
> - `battery_percent` (int 0-100, o null) — null significa que no hay batería instalada. Null no es cero. Muestra null como "Solo USB" o sin indicador de batería; muestra 0 como una batería agotada. Colapsar los dos produce una pantalla que miente.
> - `is_charging` (bool, o null) — verdadero solo mientras la corriente esté realmente fluyendo hacia la celda, leído desde el estado de la máquina de estados del cargador. Una batería llena con un cable vivo reporta falso. Deja el indicador de carga cuando vaya falso, como lo hace un teléfono — no lo active al conectar el cable.
> - `power_source` (`"usb"` o `"battery"`) — si hay energía externa presente. Esto no es la misma pregunta que la carga.
> - `wifi_signal_dbm` (int, o null) — RSSI bruto en dBm, null cuando la escala no esté conectada. No lleva nivel de calidad, así que si muestras barras o una etiqueta buena/ débil debes derivarla — y debes derivarla de la misma manera que hace el propio selector de red de la escala, o la misma señal será descrita de forma diferente en dos lugares. Usa 0..5 barras, lineal desde -100 dBm a -40 dBm, con truncamiento entero:
>   `Math.floor((Math.min(-40, Math.max(-100, rssi)) + 100) * 5 / 60)`.
>   Esto da 0 barras en -100..-89, 1 en -88..-77, 2 en -76..-65, 3 en -64..-53, 4 en -52..-41, 5 en -40 y más alto. Una versión flotante desplaza cada umbral, así que mantén el floor.
>
> Las cuatro combinaciones que vale la pena distinguir: sin batería instalada (Solo USB); cargando al N%; cargado y en USB al N%; funcionando con batería al N%.
>
> **La carga supera a la baja batería, incluyendo para el color.** Debajo del 20% una batería se muestra normalmente como una alerta, roja. Suspende eso mientras `is_charging` sea verdadero y usa el color de carga en su lugar, cualquiera que sea el nivel: una celda al 15% con un cargador vivo no es un fallo, es una celda que se está solucionando. Aplica las reglas en este orden: carga, luego baja, luego neutra. La propia pantalla de la escala hace esto, así que saltarla hace que tu panel de control contradiga la pantalla que el usuario está mirando — una batería cargando en verde en el dispositivo y una alerta roja en la app, para la misma escala en el mismo momento.
>
> **Ausente no es null.** Algunos campos se escriben solo cuando cambian, y otros solo en la instantánea de arranque. Un campo ausente en un latido significa inalterado — mantén el valor anterior. Solo un null explícito significa null.
>
> **Captura de pantalla.** Cuando `lan_live` es verdadero y `lan_url` no es null, obtén `{lan_url}/stream?c={lan_access_code}`: un `403` significa que el código es antiguo (espera al siguiente latido); de lo contrario, lee mensajes hasta el primer `FRAME_END` (byte `4`), decodifica las bandas en una imagen 480x320 y cierra la conexión. Si `power_state` es `"screen_off"` la pantalla es negra: primero `POST {lan_url}/tap?c={lan_access_code}&x=240&y=160`, que solo despierta y luego lee el marco unos 1.5 s después. La escala sirve a máximo dos visualizadores, así que nunca lo dejes abierto. El formato está en la sección "Vista en vivo LAN" de TELEMETRY.md. Cuando `lan_live` es falso, di "La vista en vivo está apagada en la escala (Ajustes > LAN)". Muestra `lan_access_code` en los detalles de la escala, monoespaciado, con un botón de copia, y sin botón de regenerar: el código se cambia solo en la escala.
>
> **`lan_access_code` es un secreto** que controla la pantalla para cualquiera en esa red. Nunca lo pongas, ni un URL que lo lleve, en nada compartido: vistas de amigos, inventarios públicos, exportaciones, registros, salida de MCP, análisis.
>
> Otros campos que puedes mostrar: `current_weight_g`, `workflow_phase`, `send_phase`, `current_spool_uid_1` / `_2`, `last_measurement_weight_g`, `last_measurement_status`, `ip_address`, `fw_version`, `display_name`, y los contadores `send_ok_count`, `send_fail_count`, `rfid_read_ok_count`, `rfid_read_fail_count`, `auto_tare_count`, `sessions_started`.

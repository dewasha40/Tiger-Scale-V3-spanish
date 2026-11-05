# Internos del firmware

Cómo está organizado el firmware, y los errores que ya han costado tiempo de depuración a este proyecto. Si estás realizando un cambio, lee primero [CODEMAP.md](../CODEMAP_ES.md) — mapea cada sección y función, y su tabla "Landmines" es la versión abreviada de lo que sigue.

## Forma del código

Un solo archivo Arduino C++, **aproximadamente 16 500 líneas**, dividido en 26 secciones numeradas más `§AUDIO` y `§LVGL`. El índice de contenido en la parte superior del archivo se genera mediante `bash scripts/update_toc.sh` a partir de los banners `// §N — TITLE`, por lo que no puede discrepar silenciosamente con el archivo.

`i18n.h` contiene cada cadena en dispositivo: **117 claves × 11 idiomas** (EN/PT/FR/ES/DE/ZH/IT/PL). Las traducciones de la interfaz web están separadas, bajo `data/www/locales/`.

¿Por qué un solo archivo? Empezó como un esquema de Arduino y creció. Dividirlo es una opción real pero un cambio grande y de comportamiento riesgoso en hardware que no puede ser probado automáticamente para regresión — por eso se optó por la herramienta de navegación en lugar de la división.

## La tubería de pesaje

```
HX711 → readWeight() → median + EMA filter → handleWeighWorkflow() → cloud
```

`handleWeighWorkflow()` es una máquina de estados:

```
WF_IDLE → WF_SCANNING → WF_STABLE_WAIT → WF_SENDING → WF_DONE → WF_IDLE
```

- **`readWeight()`** aplica un filtro de mediana y un promedio móvil exponencial. `isRapidChange()` elige entre un EMA rápido y un EMA fino según la tasa real de cambio en g/s — deliberadamente independiente de cuán seguido `loop()` llame a esta función. Una puerta fija de 50–100 ms anteriormente nunca activaba la ruta rápida, por lo que cada cambio de peso se filtraba lentamente.
- **`WF_STABLE_WAIT`** requiere que la lectura permanezca dentro de `±STABLE_EPSILON_G` durante `STABLE_WINDOW_MS`, con un techo de 15 segundos después del cual se envía de todos modos la mejor candidata en lugar de colgarse para siempre.
- **La ventana de estabilización se rastrea desde `WF_SCANNING`, no desde `WF_STABLE_WAIT`.** `updateStableWindow()` se ejecuta en ambas fases. El sensor de carga lee durante la escaneo, por lo que una bobina que se estabilizó mientras los lectores trabajaban ya había ganado la ventana antes de que el flujo de trabajo la solicite: medido, un escaneo que llegó a su timeout de 8 segundos ahora deja `WF_STABLE_WAIT` después de **14 ms** en lugar de un fresco 1.2 segundos. Un escaneo que termina temprano — ambos tags leídos en menos de un segundo — aún espera la ventana completa, porque el peso genuinamente no se ha estabilizado aún. El cambio elimina un conteo doble; no acorta la medición.
  Esa función también rechaza llamar al peso estable hasta que el buffer de pendiente haya llenado: `wfCurrentSlope` es 0.0 hasta entonces, y 0.0 en otro caso significa "perfectamente estable", lo que se leería como estable exactamente en el momento en que una bobina se coloca.
- **El peso neto** es el bruto menos el peso del contenedor de la bobina, obtenido de la inventaria del usuario. Si ese fetch falla, el envío sigue con net = 0 en lugar de bloquearse.
- **No está limitado a `measure_gr`**, el llenado nominal del fabricante. Las bobinas suelen salir de fábrica con un poco más del indicado en la etiqueta, y la báscula reporta lo que tiene. El clamping que estaba allí también era la fuente de una carrera: `measure_gr` solo se lee en la ruta que obtiene el registro de inventario en línea, nunca en la que usa el valor prefetchado durante `WF_SCANNING`, por lo que la misma bobina con el mismo peso bruto se envía como 524.8 g o 500.0 g según si el prefetch superó el tiempo de estabilización. Hacer el flujo más rápido hizo que el valor incorrecto ganara más seguido.

### Auto-tara

Hay exactamente **uno** manejador de auto-tara: deriva negativa, requiriendo 1 segundo sostenido por debajo de cero, más un guardia de estado vacío, más un enfriamiento de 10 segundos.

Un manejador "inmediato" anterior con un debounce de ~250 ms fue eliminado y no debe regresar. Una caída mecánica de una bobina colocada firmemente podría dispararlo, tareando la báscula sobre el peso propio del artículo *mientras el artículo estaba sobre ella*.

### Si la báscula lee 0 g o invertida "sin importar qué"

Revisa primero el **signo** de `scale.get_units()` bruto, antes de tocar lógica de filtros o auto-tara. Una conexión de cable de celda de carga invertida produce exactamente este síntoma, porque el clamping downstream `if (weight < 0) weight = 0` hace que un error de polaridad de hardware se vea idéntico a un error de software. Esto ya ha ocurrido una vez.

## NFC

`PN532Reader` (§5) envuelve la biblioteca Adafruit detrás de una pequeña
superficie `isNewCardPresent()` / `readCardSerial()` / `uid`, por lo que el bucle de polling y el código de metadatos son idénticos en todos los tres transportes. Solo las internas de la clase difieren según `RFID_TRANSPORT_*`.

Tres cosas importantes de saber:

- **El transporte es en tiempo de compilación.** Ver [HARDWARE.md](HARDWARE_ES.md). La única falla más común en la historia de este proyecto es firmware compilado para el transporte incorrecto, que detecta nada y reporta nada.
- **La potencia de RF es una configuración en tiempo de ejecución**, 5 niveles, predeterminado 3, ajustable desde Configuración → Hardware y persistente en NVS. Existe porque las dos antenas están ~75 mm apartadas mirándose y se cruzan.
- **Los sonidos fantasma al removerse** son manejados por `gRecentlyRemovedUid1`/`2` más `gRemovalDetectedMs`, capturados dentro del bloque de reinicio de remoción de `handleWeighWorkflow()` *justo antes* de limpiar los UIDs. Eso es lo que distingue una remoción retardada de un eco RFID de una etiqueta recién levantada de una etiqueta genuinamente nueva. La ventana de deduplicación separada de 1.5 s (`isDuplicateRecentUid`) nunca fue ampliada deliberadamente.

## <a id="lvgl-v8-traps"></a>Trampas de LVGL v8

La totalidad de la UI es LVGL v8.4. Cada una de estas ha costado un real
viaje de compilación-flash-fotografía en hardware.

- **Nunca llames `lv_timer_handler()` desde dentro de una callback de clic.** La callback ya corre dentro de la pila de llamada de `lv_timer_handler()`. El patrón usado aquí es: la callback establece una bandera, y un contexto de nivel superior como `loop()` hace el trabajo de pantalla bloqueante.
- **`lv_scr_load(newScreen)` antes de `lv_obj_del(oldScreen)`.** El orden inverso deja el puntero de pantalla activa de LVGL colgante entre las dos llamadas, lo que hacía que el dispositivo se reiniciara al 100% reproducible cuando ocurría aquí.
- **`lv_obj_align_to()` centra el cuadro declarado, no el tinta visible.** Los glifos de fuente de icono (`LV_SYMBOL_*`) tienen aperturas laterales asimétricas, por lo que un glifo "centrado" puede verse desalineado sin importar el offset que elijas. Si un pequeño icono no se centra después de varios intentos, deja de adivinar píxeles y usa una imagen rasterizada previa — ve `icon_bolt.h` para el patrón (`lv_img_create()` + `lv_img_set_src()` +
`lv_obj_center()` centra exactamente, sin involucrar métricas de fuente).
- **`lv_obj_set_style_transform_angle()` gira alrededor de la esquina superior izquierda** por defecto. Sin un `transform_pivot_x/y` explícito al centro del objeto, una forma rotada puede salir completamente fuera de los límites de un pequeño padre clippado y simplemente nunca aparecer.
- **Algunos constructores llaman `lv_obj_align()` internamente** — `lv_keyboard_create()` sí — lo que establece una *propiedad de estilo persistente*. Una posterior `lv_obj_set_pos()` simple es entonces reinterpretada silenciosamente como un desplazamiento desde esa alineación en lugar de una posición absoluta. Usa
`lv_obj_align(widget, LV_ALIGN_TOP_LEFT, x, y)` para tales widgets.
- **`LV_KEYBOARD_CTRL_BTN_FLAGS` incluye `LV_BTNMATRIX_CTRL_CHECKED`** en cada tecla de control (shift, 1#, backspace). Es la forma de LVGL de dar a esas teclas una apariencia muda, *no* un estado de alternancia por tecla. El estilo mediante el compartido
`LV_STATE_CHECKED` ilumina todas ellas a la vez, y las ilumina desde la creación independientemente del estado real. Para estilizar un botón independientemente, conecta `LV_EVENT_DRAW_PART_BEGIN` y sobrescribe `dsc->rect_dsc` / `dsc->label_dsc` cuando `dsc->part == LV_PART_ITEMS && dsc->id == <index>`.
- **Nunca dejes que el dibujo raw-`gfx` llegue al panel mientras un pantalla LVGL esté cargada.** Los dos renderizadores no coordinan, y LVGL solo repinta lo que *él* cree que está sucio, dejando píxeles antiguos atrás. Una llamada alcanzable `displayMessage()` en un camino LVGL vivo es un error — conviértelo al patrón compartido
`lvglCenteredScreen()` / `lvglAddStatusBadge()` / `lvglAddCenteredLabel()` toast, o bórralo si la pantalla ya muestra la misma información.

`LVGL_TEST_MODE` (§LVGL) es una escapada de depuración, normalmente 0. Establecerlo a 1 hace que `loop()` ejecute solo `lv_timer_handler()`, aislando la tubería de renderizado de la escala, NFC y código de nube. No es una bandera de migración.

## Memoria

- **Nunca uses `ps_malloc`.** Devuelve null silenciosamente. Usa
`heap_caps_malloc(n, MALLOC_CAP_SPIRAM)` con un fallback a DRAM.
- El heap libre se ha observado que cae a ~15–23 KB durante picos de Firebase HTTPS, con frecuentes `SSL - Memory allocation failed`. Una reinicialización reportada después de dos pesajes no se pudo reproducir en 11 ciclos posteriores, por lo que es un item de vigilancia en lugar de un error diagnosticado. Si estás persiguiendo una reinicialización, empieza aquí.
- Respuestas JSON grandes usan `DynamicJsonDocument` con filtros. Un
`StaticJsonDocument<256>` falla *silenciosamente* con `NoMemory` en algunas de estas respuestas, dejando campos misteriosamente vacíos.

## Servidor web y WebSocket

Un `ESPAsyncWebServer` sirve la interfaz web LittleFS más una API JSON, y empuja el estado en vivo sobre un WebSocket a 10 Hz. `buildWsFrame()` es la única fuente de verdad para ese payload: solo los campos modificados salen en cada tick, con una instantánea completa al conectar y cada 30 s.

Porque es *asincrónico*, los manejadores no deben bloquear. El trabajo pesado de Firestore se despacha a una tarea worker fijada en el núcleo 0 (§14).

## <a id="screen-over-http"></a>Ver y controlar la pantalla mediante HTTP

Cuatro rutas, sin configuración y sin código de acceso, por lo que comprobar una pantalla cuesta dos solicitudes en lugar de un viaje al banco:

| Ruta | Lo que ofrece |
|---|---|
| `GET /screen.bmp` | la pantalla como un BMP de 24 bits, 480x320 |
| `GET /api/tap?x=&y=` | un clic; agrega `x2`/`y2` para un deslizamiento |
| `GET /screen.ver` | cuatro bytes, el contador de marco |
| `GET /screen` | la imagen en una página, haz clic para tocar, arrastra para deslizar |

`XXXX` a continuación es el sufijo de cuatro hexadecimales de la unidad, tomado de los últimos dos bytes de su MAC:

```bash
curl -s -o s.bmp http://tigerscale-XXXX.local/screen.bmp
curl -s "http://tigerscale-XXXX.local/api/tap?x=415&y=285"
curl -s "http://tigerscale-XXXX.local/api/tap?x=200&y=280&x2=200&y2=90"
```

Tres decisiones que vale la pena conservar:

- **La imagen proviene del lienzo, no del panel.** `gfx` es un `Arduino_Canvas`, por lo que ya existe un framebuffer completo y está definido como lo último dibujado — sin sprite de reflejo, sin copia en el callback de flujo, y sin lectura desde el AXS15231B, que es lento e inconfiable. Se sirve desde una instantánea de una sola copia en PSRAM, porque serializar 450 KB toma mucho más tiempo que una repintura y leer el lienzo en vivo arrastraría entre dos pantallas.
- **Un deslizamiento es el toque caminado en doce pasos.** LVGL decide si es un arrastre o un toque según el movimiento entre dos lecturas de entrada consecutivas, por lo que un salto único desde el inicio hasta el final no desplaza nada en absoluto. Sin esto, la mitad de cualquier lista más larga que la pantalla no es accesible desde un terminal — lo cual es la mitad del punto.
- **`/api/tap` responde con retraso y espera sin dormir.** Devuelve una vez que el toque ha sido procesado y la pantalla ha terminado de cambiar, por lo que la respuesta es la señal de que el siguiente captura es digno de tomar. La espera se realiza con `RESPONSE_TRY_AGAIN`; dormir en un manejador bloquea la tarea TCP asíncrona y cada otra solicitud con ella.

Un toque inyectado no bypassa el sueño de la pantalla. Pasa por `tsRead()`, por lo que en un panel oscuro despierta la báscula y es absorbido exactamente como un dedo — verificado en el banco con la pantalla dormida y un toque dirigido al botón de Configuración, que despertó el dispositivo hacia la pantalla de inicio. Un comando remoto nunca debe actuar sobre una pantalla que nadie puede ver.

**¿Qué es y qué no es.** Es lo que LVGL dibujó. Geometría, texto, traducción, truncamiento, qué pantalla está abierta: confiable. Color como lo ve el ojo, luz de fondo, renderizado del panel: no. Ni siquiera puede ver el panel mismo, por lo que un cambio en la ruta de flujo puede verse perfecto aquí y ser incorrecto en el vidrio.

Un deslizamiento toma aproximadamente dos segundos, y la razón merece ser registrada: cada uno de sus doce pasos desplaza la lista, lo que repinta la pantalla, lo que cuesta un completo 307 KB de envío al panel — unos **175 ms por paso**. Un dedo paga el mismo precio, por lo que desplazar se siente pesado.

## Vista en vivo

Una herramienta de banco: el panel, en un navegador, con clics que regresan en sentido opuesto. Abra `http://<ip-de-la-báscula>/live` (el puerto 80 redirige al servidor real en 81) y escriba el código de seis caracteres desde Configuración → LAN. El interruptor de esa página enciende y apaga todo esto. §LIVE en el .ino contiene la implementación.

El diseño se basa en un hecho: **`gfx` es un `Arduino_Canvas`, por lo que ya existe un framebuffer completo de 480×320 RGB565** y `getFramebuffer()` lo entrega. La pantalla está por lo tanto completamente legible en cualquier instante, y ningún frame nunca tiene que ser reconstruido a partir de los rectángulos que LVGL repinta. Eso es lo que hace que una vista que llega tarde obtenga una imagen correcta en lugar de una ensamblada.

- **El captura está sincronizado con `lv_disp_flush_is_last()`**, que marca el final de un refresco completo de LVGL. Muestrear en cualquier otro momento puede capturar una pantalla parcialmente dibujada. Un barrido de 500 ms también se ejecuta, para capturar las rutas que pasan por LVGL — el splash de arranque y el modo de ahorro de energía dibujan directamente sobre el lienzo.
- **La unidad es una banda de 16 columnas en horizontal.** La rotación 3 almacena una columna en horizontal de forma contigua, por lo que una banda es un bloque de 10 240 bytes ininterrumpido. Cada banda se copia, se hash y se envía solo si su hash difiere del que ese visualizador se sabe que contiene. Una pantalla completa es 30 bandas y unos 21 KB después de RLE.
- **Un frame está delimitado por su fin, no se cuenta desde su inicio.** El navegador decodifica las bandas en un `ImageData` fuera de pantalla y llama a `putImageData()` una vez, en `FRAME_END`. Nada parcial nunca llega al lienzo visible, por lo que un cambio de página llega como una imagen única por construcción, no por ajuste.
- **Sockets raw lwIP, no el servidor asíncrono en el puerto 80.** `ESPAsyncWebServer` copia cada mensaje en una cola interna de RAM limitada en mensajes, no en bytes, por lo que un visualizador que deja de drenar se convierte en una fuga de memoria que mata el dispositivo en otro lugar; AsyncTCP también no es invocable desde una tarea arbitraria. Un `send()` bloqueante en la tarea en vivo, contra un buffer fijo, *es* la presión de retroalimentación: un visualizador lento cuesta tiempo, nunca memoria.
- **Keep-alive es soporta la carga.** Contestar cada toque con `Connection: close` significaba una conexión TCP por clic, y como la báscula cierra primero, cada una se quedaba en `TIME_WAIT` durante dos minutos. lwIP se construye con diez sockets, por lo que un minuto de clics normales los agotó y el puerto comenzó a rechazar conexiones — la característica funcionó perfectamente y luego simplemente se detuvo, recuperándose un minuto después. Una sola conexión reservada ahora lleva cada toque.
- **Los toques se inyectan dentro de `tsRead()`**, no en el driver de entrada de LVGL, porque es el único lugar donde ambos consumidores se encuentran: la `read_cb` de LVGL lo llama, y así lo hacen las pantallas que aún consultan directamente el panel.
- **Dos visualizadores, deliberadamente.** Cada uno multiplica los pbufs en tránsito, y un tercero tomó la RAM interna libre decenas de kilobytes por debajo de donde dos se sitúan. Un ping cada 5 s sirve como chequeo de vitalidad, por lo que un navegador que se muere en lugar de cerrarse deja de ocupar un slot dentro de unos pocos segundos.

### ¿Cuánto realmente cuesta

Medido en el banco con dos visualizadores en Chrome, sobre 236 cambios de página en cinco minutos. El firmware registra esta división por sí mismo — vea la línea `[LIVE]` en `liveCapture()`.

| Desde un clic hasta que aparece la nueva pantalla | promedio |
|---|---|
| la báscula reconstruyendo su propia pantalla | **483 ms** |
| codificar todas las 30 bandas | 32 ms |
| enviarlas | 119 ms |
| **total** | **638 ms** |

**Tres cuartos de eso es la báscula, no el espejo.** Un dedo en el vidrio espera el mismo ~600 ms para que la página de configuración se construya; la vista en vivo agrega unos 158 ms adicionales. Si este número necesita reducirse, el trabajo está en los constructores de pantallas de LVGL (`runSettingsMenu()` crea aproximadamente cien objetos, la mayoría de ellos iconos vectoriales manuales), no aquí. Tenga en cuenta la división antes de optimizar la mitad equivocada — esa línea de registro existe precisamente porque una pantalla lenta y un espejo lento son indistinguibles sin ella.

El envío es viajes de ida y vuelta, no cálculo: 21 KB contra la ventana de ~5.7 KB de lwIP es cuatro de ellos, y en la LAN del banco un ping al *gateway* ya promedia 25 ms con una cola de 140 ms. En un enlace más tranquilo llega proporcionalmente más rápido. Hacerlo significativamente más rápido significa enviar menos bytes, no código más rápido.

Una pantalla sin tocar cuesta 2 bytes cada 30 segundos. Navegación continua deja la vista prolongada pixel-identical a un visualizador que acaba de conectar — lo cual es la verdadera prueba de residuo, ya que un visualizador que acaba de llegar no puede estar llevando nada.

Ese recorrido de cinco minutos es aproximadamente veinte veces lo que hace una persona, y es la carga contra la cual están dimensionados los guardias: la RAM interna libre se agotó en 9.4 KB, la captura se pausó diecinueve veces y se detuvo una vez, y nada más en la báscula fue notado — sin reinicio, sin peso perdido, sin lector perdido.

También devuelve memoria en lugar de tomarla. El banda de 10 KB se mantiene solo mientras alguien esté realmente viendo, un presupuesto de bytes salientes limita el tráfico sostenido (cada byte pasa por un lwIP pbuf, y los pbufs vienen del mismo heap interno que todo lo demás necesita), y por debajo del nivel libre de heap se detiene la captura, y luego se detiene por completo por debajo de un nivel más duro.

## OTA

`otaFetchLatest()` lee `version.json` desde GitHub Pages y compara `version` contra `TIGERSCALE_FW_VERSION`; `otaApply()` transmite el binario con una barra de progreso y verifica un SHA-256.

- **`firmware_url` debe apuntar al plano `firmware.bin`.** `Update.begin()` y `Update.write()` solo reemplazan la partición de la aplicación, por lo que pasarles un `firmware.factory.bin` corrompe el cargador y la tabla de particiones.
- El manifiesto se genera desde `TIGERSCALE_FW_VERSION` por el flujo de lanzamiento, por lo que el firmware y el manifiesto no pueden divergir.
- **Dos placas, un manifiesto.** La clave plana `version` / `firmware_sha` / `firmware_url` describe la compilación ESP32-S3-Touch-LCD-3.5**B** y siempre lo hará: cada báscula en el campo es una -3.5B y lee exactamente esas claves. Una -3.5 lee `boards["3.5"]` y no tiene retroceso a ellas, porque el retroceso instalaría la imagen de la otra placa y dejaría una pantalla negra.
  `scripts/make-manifest.py` rechaza generar un manifiesto que rompa esto.
- **Límite conocido:** las compilaciones publicadas asumen HSU. Una placa cableada para SPI o I²C que recibe una actualización publicada pierde su lector.

## Mantener la herramienta honesta

`bash scripts/verify.sh` es el punto de entrada — ejecuta cada guardia (i18n, cobertura de fuente CJK, CODEMAP, TOC, emoji, mojibake, notas de lanzamiento) y luego construye. Cada guardia también es ejecutable por sí mismo:

```bash
bash scripts/check-i18n.sh      # every key present in all 8 language blocks, in enum order
bash scripts/check-codemap.sh   # CODEMAP line numbers still accurate
bash scripts/update_toc.sh      # regenerate the in-file table of contents
```

CI los ejecuta todos.

**En Windows**, dos peculiaridades: puede no haber un sistema `python3` — los scripts caen de vuelta al venv propio de PlatformIO (`~/.platformio/penv/Scripts/python.exe`), o se establece `PYTHON=` explícitamente — y la herramienta de compilación rechaza *construir* bajo Git Bash/MSYS. Así que ejecute los guardias bajo Git Bash (`bash scripts/verify.sh --quick`) y compile en PowerShell:

```powershell
& "$env:USERPROFILE\.platformio\penv\Scripts\platformio.exe" run -e esp32s3_hsu_b
``` Ellos estaban rotos en macOS antes de este repositorio's primer commit — dos usaron solo builtins de bash-4 y banderas de grep solo GNU, y `check-codemap.sh` imprimía `PASSED` mientras verificaba nada en absoluto. Si los reescribe, mantenga la propiedad de que un conjunto de entrada vacío es un error en lugar de un paso.

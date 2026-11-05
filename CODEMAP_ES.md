# CODEMAP — TigerTagSplashESP32.ino

Mapa de navegación para el firmware del archivo fuente único de ~17 200 líneas. Nunca lo leas completo; ese es exactamente el punto de este documento.

**El flujo de trabajo, cada vez que lo uses:**

1. Encuentra la sección o función en las tablas que aparecen a continuación.
2. `grep -n "theName" TigerTagSplashESP32/TigerTagSplashESP32.ino` — el número de línea aquí es un punto de partida, el grep es la fuente de verdad.
3. Lee esa línea ±60.
4. Realiza el edit más pequeño que haga la tarea.
5. Antes de terminar, ejecuta `bash scripts/check-codemap.sh` y corrige cualquier desviación que reporte actualizando los números que aparecen a continuación.

Los números de línea aquí fueron regenerados mecánicamente contra el archivo actual y verificados por `scripts/check-codemap.sh` (0 desviación). Se vuelven obsoletos en el momento en que alguien inserte líneas por encima de un ancla — lo cual es la razón por la que existe el paso 2.

## Mapa de secciones

Regenerado por `bash scripts/update_toc.sh`, que también reescribe el bloque de comentario coincidente dentro del .ino. Ambos provienen de los banners `// §N — TITLE` en la fuente, por lo que esta tabla no puede silenciosamente discrepar con el archivo.

| § | Nombre | Líneas |
|---|--------|--------|
| 1 | Configuración de hardware (pines, definiciones) | 259–452 |
| 2 | Configuración OTA | 453–485 |
| 3 | Declaraciones anticipadas | 486–790 |
| 4 | Redondeo de peso | 791–823 |
| 5 | Objetos globales (incl. `PN532Reader`, sintonización RF) | 824–1749 |
| 6 | Variables de configuración | 1750–2179 |
| 7 | Pantalla de visualización / renderizado de peso | 2180–2957 |
| 8 | Parsing de la nube | 2958–2972 |
| 9 | Configuración de WiFi + cada toque/pantalla de configuración de LVGL | 2973–8079 |
| 10 | LittleFS | 8080–8379 |
| 11 | Autenticación de Firebase + Firestore | 8380–10062 |
| 13 | WebSocket | 10063–10089 |
| 14 | Tarea de trabajo de la nube (core 0) | 10090–10224 |
| 15 | Constructor de marco de WS unificado | 10225–10349 |
| 16 | Ayudantes de filtro de peso | 10350–10364 |
| 17 | Reinicio del estado tras envío | 10365–10385 |
| 18 | Manejador compartido de envío de peso | 10386–10489 |
| 19 | Servidor web (todas las rutas HTTP) | 10490–11518 |
| 20 | Comunicación con la nube | 11519–11701 |
| 21 | Máquina de estado del flujo de pesaje | 11702–12209 |
| 22 | mDNS | 12210–12247 |
| 23 | Escala (inicialización HX711, filtrado, calibración) | 12248–12418 |
| AUDIO | Codificador ES8311 (sonido de beep I2S, I2C 0x18) | 12419–12536 |
| 24 | RFID (PN532 mediante `PN532Reader`, lectura de etiqueta) | 12537–13619 |
| 25 | OTA (actualización de firmware + sistema de archivos) | 13620–14299 |
| LVGL | Puente LVGL + pantalla principal de pesaje | 14300–15239 |
| LIVE | Vista en vivo remota: la pantalla de salida, toques de entrada | 15240–16089 |
| 26 | `setup()` + `loop()` | 16090–17335 |

Dos huecos y dos peculiaridades en ese numerado, todos deliberados:

- **§12 no existe.** Fue eliminado en una sesión anterior. Reenumerar todo lo que le sigue invalidaría cada referencia `§N` en los comentarios de fuente y en este archivo, sin ganancia.
- **§AUDIO y §LVGL no están numerados intencionalmente** — se añadieron después del esquema numerado, y un nombre es más útil que forzarlos en. `update_toc.sh` acepta cualquier `§<token> — Titulo`, por lo que ambos aparecen en el TOC.
- **§7 aún se llama "OLED DISPLAY" en el banner de fuente.** No hay OLED en V3; es una pantalla LCD de 480×320. El nombre se hereda de V2 y se dejó así aquí porque es carga útil para referencias cruzadas — renombrarlo es un buen seguimiento, pero no es una corrección de higiene.
- **§8 (Parsing de la nube) es 15 líneas** y contiene solo declaraciones; `parseCloudSpoolMeta` está realmente definido más abajo en §20.

## Ubicaciones de funciones clave

| Función | Línea | Sección |
|--------|------|--------|
| `axpRead` | 581 | §3 |
| `axpChargeState` | 704 | §3 |
| `axpBatteryLevel` | 716 | §3 |
| `pollBatteryState` | 643 | §3 |
| `roundWeight` | 794 | §4 |
| `applyPN532RfTuning` | 1246 | §5 |
| `runScreensaver` | 2233 | §7 |
| `displayWeightWithState` | 2666 | §7 |
| `tsPollLoop` | 3165 | §9 |
| `tsBtn` | 3202 | §9 |
| `tsKeyboard` | 3555 | §9 |
| `wifiTouchConfigure` | 3919 | §9 |
| `tsNumericInput` | 4074 | §9 |
| `runCalibrationWizard` | 4246 | §9 |
| `runLanguageSettings` | 4748 | §9 |
| `runHardwareTest` | 5179 | §9 |
| `lvglAskYesNo` | 5658 | §9 |
| `runOtaMenu` | 7078 | §9 |
| `runSettingsMenu` | 7444 | §9 |
| `setupWiFi` | 8017 | §9 |
| `setupFileSystem` | 8194 | §10 |
| `firebaseSignIn` | 8391 | §11 |
| `fetchApiKeyFromFirestore` | 8477 | §11 |
| `syncClockFromFirestore` | 8635 | §11 |
| `readInventoryContainerWeight` | 9048 | §11 |
| `pushLatestEventToFirestore` | 9749 | §11 |
| `buildWsFrame` | 10232 | §15 |
| `resetWeightFilters` | 10353 | §16 |
| `resetAfterSuccessfulSend` | 10368 | §17 |
| `handleWeightPushBody` | 10389 | §18 |
| `setupWebServer` | 10494 | §19 |
| `sendSingleUidToCloud` | 11522 | §20 |
| `parseCloudSpoolMeta` | 11578 | §20 |
| `pushWeightToCloud` | 11628 | §20 |
| `fetchMetaFromApiByUid` | 11666 | §20 |
| `handleWeighWorkflow` | 11755 | §21 |
| `startMDNS` | 12213 | §22 |
| `setupScale` | 12251 | §23 |
| `isRapidChange` | 12282 | §23 |
| `readWeight` | 12318 | §23 |
| `playSoundTheme` | 12498 | §AUDIO |
| `rfidSelectReader` | 12690 | §24 |
| `normalizeRfidHwConfig` | 12744 | §24 |
| `setupRFID` | 12754 | §24 |
| `setupServo` | 12970 | §24 |
| `processAutoTare` | 12984 | §24 |
| `fetchMetaFromFirestore` | 13304 | §24 |
| `readTigerTagMetadata` | 13406 | §24 |
| `pollRFIDReader` | 13495 | §24 |
| `recoverRFIDReaderIfNeeded` | 13578 | §24 |
| `isDuplicateRecentUid` | 13598 | §24 |
| `isUidLatched` | 13602 | §24 |
| `finishRfidSession` | 13614 | §24 |
| `otaApply` | 13807 | §25 |
| `otaFetchLatest` | 13949 | §25 |
| `lvglBuildMainScreen` | 14554 | §LVGL |
| `lvglUpdateMainScreen` | 14982 | §LVGL |
| `setup` | 16109 | §26 |
| `loop` | 16281 | §26 |

## Minas — lee la nota antes de editar estas

Conocimiento ganado que no es visible desde el código mismo. Cada una de estas costó al menos una sesión de depuración.

| Función | Lo que debes saber primero |
|----------|------------------------------|
| `setupRFID` | El transporte RFID es una **elección en tiempo de compilación**, nunca en tiempo de ejecución. Compilar el entorno que no coincida con la conexión física produce un firmware que no encuentra ningún lector y no dice nada sobre el motivo. La unidad verificada en el banco es HSU en una placa -3.5B → `-e esp32s3_hsu_b`. El `SPI.begin()` solo es guardado para este motivo exacto: antes se afirmaba GPIO47/48/41 antes de que `Serial1`/`Serial2` pudieran vincularlos, por lo que el build HSU transmitía absolutamente nada. |
| `handleWeighWorkflow` | El bloque de reinicio detectado al remover captura `gRecentlyRemovedUid1`/`2` y `gRemovalDetectedMs` desde `lastUID`/`lastUID2` *justo antes* de limpiarlos. Eso es lo que permite diferenciar un eco retardado de una etiqueta que fue levantada recientemente de una etiqueta genuinamente nueva — la solución para el triple pitido fantasma al retirar la bobina. Su rama `WF_IDLE` también debe mantener la limpieza de `lastUID`/`lastUID2`/`lastUIDLeft`/`lastUIDRight` una vez que el peso caiga por debajo de `MIN_WEIGHT_TO_SEND_G`, o la interfaz de usuario se queda atrapada en "remover material" con una báscula vacía. |
| `isDuplicateRecentUid` / `isUidLatched` | La ventana de deduplicación es de 1.5 s y fue deliberadamente nunca ampliada. El error de pitido fantasma que parecía relacionado anteriormente es manejado por el mecanismo separado y más estrecho arriba, verificado solo en los sitios de llamada a `playSoundTheme()`. |
| `isRapidChange` | Calcula una tasa real de cambio en g/s, independiente de cuán seguido `loop()` llame a esta función. El viejo umbral fijo de 50–100 ms efectivamente nunca activaba el filtro rápido, por lo que cada cambio de peso se movía a través del filtro lento. |
| `processAutoTare` | Hay exactamente **una** manejador de autocalibración: el que tiene una deriva negativa (1 s sostenido, más guardia de inactividad/vacío, más 10 s de enfriamiento). Un manejador anterior "inmediato" ~250 ms fue eliminado porque la colocación mecánica firme de una bobina podría activarlo, calibrando la báscula sobre el peso propio del objeto mientras estaba sobre la plataforma. No reintroduzcas una variante de desbordamiento corto. |
| `readWeight` | Si esta función alguna vez lee 0 g o invertido "sin importar qué", verifica el **signo** de `scale.get_units()` bruto antes de tocar filtros o autocalibración. Una conexión de cable de celda de carga invertida produce exactamente ese síntoma, porque el clamping downstream `if (weight < 0) weight = 0` hace que un error de polaridad de hardware se vea idéntico a un error de software. Esto ya ha ocurrido una vez. |
| `lvglAskYesNo` | Bloquea en `lv_timer_handler()` hasta que un botón sea presionado, por lo que llamarlo desde dentro de una callback de LVGL reingresa al manejador y produce un deadlock — la primera trampa en `docs/FIRMWARE.md`. Ambos sitios de llamada son `loop()` y `runOtaMenu`, ninguno de los cuales es una callback. También calcula el tamaño de su tarjeta midiendo la etiqueta del cuerpo envuelto, no asumiendo un conteo de líneas: la misma frase es una línea en inglés y dos en alemán. |
| `readInventoryContainerWeight` | Devuelve silenciosamente 0 en cualquier error HTTP o de análisis, sin reintentos. Brecha conocida, no un error para sorprenderse. |
| `otaFetchLatest` | Dos placas leen este manifiesto de forma diferente, y la asimetría es deliberada. Una **-3.5B** lee las claves planas `version`/`firmware_sha`/`firmware_url` — cada báscula jamás enviada es una -3.5B y hace esto, incluyendo las que han estado sin conexión durante meses, por lo que esa ruta nunca debe cambiar de forma. Una **-3.5** lee `boards["3.5"]` y tiene **ningún respaldo** a las claves planas: caer en respaldo instalaría la imagen de -3.5B y dejaría una placa con pantalla negra y sin forma de reportarlo. Añadir claves al manifiesto es seguro — el filtro ArduinoJson existe precisamente para crecer — pero mover o reorientar las claves planas afecta a toda la flota de forma inmediata. `scripts/make-manifest.py` enforces esto en tiempo de generación. |
| display bring-up (Sección 5) | El ST7796 de la -3.5 se construye a través de una subclase `TigerScaleST7796` por una razón: `Arduino_ST7796::begin()` en GFX 1.6.x fuerza `SPI_MODE3` en ESP32 y este panel quiere modo 0. Hacerlo mal no falla — la luz de fondo se enciende, `[LCD] begin OK` se imprime y la pantalla sigue negra. Si alguna vez actualizas la biblioteca GFX, vuelve a verificar esa sobrecarga antes de creer que una -3.5 oscura es un error de hardware. |
| `lcdResetByTCA9554` | Toma el bus porque el expander está en `Wire1` a 0x20, no en `Wire`/GPIO21-22 donde el demo del proveedor lo coloca — así que durante mucho tiempo esta función reiniciaba nada en absoluto. Ahora corre solo para la -3.5: su ST7796 necesita el pulso, y la -3.5B ha estado arrancando sin uno durante meses. No "arregles" esa asimetría sin una -3.5B en el banco. |
| `tsRead` (rama de placa) | El IC de tacto difiere por placa: AXS5106L @0x3B en la -3.5B, FT6336 @0x38 en la -3.5, ambos en `Wire1`. Solo difiere el acceso a registros; la mapeo de portrait→landscape debajo de él es compartido y debe seguir siendo compartido, o las dos placas se desvían en una forma solo detectable por una unidad física. |
| `otaApply` | `firmware_url` debe apuntar al plano `firmware.bin`. `Update.begin()`/`write()` solo reemplazan la partición de la aplicación, así que pasándole un `firmware.factory.bin` corrompe el bootloader y la tabla de particiones. |
| `applyPN532RfTuning` | Establece la potencia de transmisión y la sensibilidad de recepción juntas a través de `RFConfiguration` CfgItem 0x0A, mediante la función pública `sendCommandCheckAck()` de la biblioteca — no se necesita parche de biblioteca. La tabla de 6 niveles existe porque las dos antenas están ~75 mm separadas mirándose y se cruzan; los niveles 0–4 abarcan solo el extremo de baja potencia, que es el rango que importa para eso, y el nivel 5 es los valores de reinicio del datasheet del PN532. Nivel predeterminado 5. En HSU la biblioteca lee solo el ACK de este comando, así que el marco de respuesta se vacía desde UART después (`drainPn532Uart`), y nuevamente antes del siguiente comando — si queda en el buffer se interpreta como el ACK del siguiente comando y el lector deja de devolver UID. |
| `tsKeyboard` | El resaltado de la tecla shift usa una callback `LV_EVENT_DRAW_PART_BEGIN` en un índice de botón específico, **no** `LV_STATE_CHECKED`. `LV_KEYBOARD_CTRL_BTN_FLAGS` integra `LV_BTNMATRIX_CTRL_CHECKED` en cada tecla de control, así que el estilo mediante ese estado ilumina shift, 1# y retroceso al mismo tiempo, y los ilumina desde la creación independientemente del estado real. |
| `computeWeightAvailable` | El peso neto **no** está limitado a `measure_gr`, y eso es una decisión, no una omisión: una bobina que contiene más de lo que dice su etiqueta es una medición real. El clamping que estaba allí antes también era una carrera — `measure_gr` solo se lee en la rama que obtiene el registro de inventario en línea, nunca en la rama que usa el valor prefetchado durante `WF_SCANNING` — así que la misma bobina al mismo peso bruto fue enviada como 524.8 g o 500.0 g dependiendo de si la red superaba el tiempo de estabilización, y acelerar el workflow hizo que la incorrecta ganara más seguido. |
| `updateStableWindow` | Ejecuta en **ambos** `WF_SCANNING` y `WF_STABLE_WAIT`, y la ventana de estabilización se deliberadamente no reinicia en la transición entre ellas — ese reinicio hacía que cada pesaje pagara una nueva `STABLE_WINDOW_MS` que ya había ganado durante la escaneo. También requiere `slopeBufFull` antes de llamar al peso estable: `wfCurrentSlope` es 0.0 hasta que el buffer anular se llena (6 x 400 ms), y 0.0 de otro modo significa "perfectamente estable", así que sin esa prueba un buffer no lleno lee como estable exactamente en el momento que se coloca una bobina. |
| `/screen.bmp` handler | Sirve una **captura**, no el canvas en vivo. Serializar 450 KB toma mucho más tiempo que una repintado, así que leer el framebuffer mientras se escribe entrega imágenes rotas entre dos pantallas diferentes. La captura es un memcpy en **PSRAM** — nunca en DRAM interna, que es el pool al que se refiere la fila de `liveEnsureBuffers` de abajo — liberado cuando la respuesta se destruye. Bajo rotación 3 el canvas se almacena con el panel hacia arriba, así que un pixel de paisaje (x, y) vive en `(479 - x) * 320 + y`; vea `Arduino_Canvas::writePixelPreclipped()`. Y muestra lo que LVGL dibujó, no lo que muestra la pantalla: un cambio en la ruta de flujo o debajo parece perfecto aquí y aún puede estar mal en el dispositivo. |
| `/api/tap` handler | Espera devolviendo `RESPONSE_TRY_AGAIN`, nunca durmiendo — un sueño aquí bloquea la tarea TCP asíncrona y cada otra solicitud con ella, que es la trampa escrita sobre `handleWeightPushBody()`. Responde una vez que la presión ha sido tomada y la pantalla ha dejado de cambiar, así que un llamador puede encadenar tap y `screen.bmp`. El tiempo de espera es 3.5 s porque un deslizamiento genuino toma alrededor de dos segundos: cada uno de sus doce pasos desplaza la lista, que repinta la pantalla, que cuesta un empuje completo de 307 KB a unos 175 ms por paso. Un dedo real paga lo mismo. |
| presión inyectada en `tsRead` | Un deslizamiento es la presión *recorrida* en doce pasos, uno por cada lectura de entrada, porque LVGL decide arrastrar o tocar desde el movimiento entre lecturas consecutivas — un salto único de inicio a fin no desplaza nada. El gesto libera en su último paso en lugar del leído después, así que no depende de que un llamador siga preguntando. La red de seguridad de 800 ms atrapa una presión que nadie vuelve a muestrear, que un constructor de pantallas puede causar; sin ella el siguiente pedido es tragado por el anterior. Debe seguir pasando por `tsRead()` y no inyectarse directamente en LVGL, así que el sueño de la pantalla lo consume exactamente como lo hace un dedo. |
| `screen_page.h` / `live_page.h` | Encabezados separados, no cadenas crudas en el .ino, y no por preferencia: el generador de prototipos Arduino escanea las literales de cadena crudas como si fueran código, así que `async function poll()` en el script de la página se convirtió en un prototipo C++ generado al principio del archivo y nada compiló. |
| `liveCapture` | Lee el canvas **una vez** por banda, en un buffer de trabajo, luego lo hash y codifica. Hashar el canvas y codificar el canvas como dos lecturas separadas permite que un repintado se interponga, y el espectador queda registrado como poseedor de una banda que nunca recibió — un artefacto permanente que solo un reenlace limpia. También debe devolver temprano si no hay espectadores: los buffers de trabajo solo se asignan mientras alguien está viendo, y ejecutarlo sin ellos escribió a un puntero nulo y provocó un panic en el core 0. |
| `liveHandleRequest` | Cada respuesta es **keep-alive**, deliberadamente. Cerrarlas significaba una conexión TCP por cada toque; la báscula cierra primero, así que cada una se quedaba en `TIME_WAIT` durante dos minutos contra los diez sockets de lwIP. Una minuto de clics normales agotó los y el puerto rechazó conexiones directamente — la característica funcionó, se detuvo, luego se recuperó un minuto después por sí sola. No "limpies" estas de vuelta a `Connection: close`. |
| `liveEnsureBuffers` | Solo la banda de trabajo pertenece a la DRAM interna; la banda codificada no. Colocar ambas allí costó 20 KB y produjo una oscilación en lugar de velocidad — tomar la memoria cruzó el nivel libre de heap, lo que detuvo a los espectadores, lo que liberó la memoria, lo que les permitió reconectar y tomarla de nuevo. |
| `tsRead` | Inyecta el toque remoto **antes** de tocar la pantalla, y ese posicionamiento es el punto: la `read_cb` de LVGL y las pantallas que aún poll directamente ambos pasan por aquí, así que inyectar un nivel arriba dejaría las pantallas de tacto bruto inalcanzables desde el navegador. La presión se mantiene mediante un conteo de lecturas, no solo mediante un reloj, así que no depende de cuán seguido el llamador sucede a preguntar. |

## Patrón de archivos paralelos — i18n

`TigerTagSplashESP32/i18n.h` contiene **158 claves × 11 bloques de idiomas**
(EN/PT/FR/ES/DE/ZH/IT/PL/PT-PT/RU/NL — lista exacta del Gestor de Estudio). Añadir una clave
significa editar **todos los 11 bloques**, en el
mismo orden que el enum. `bash scripts/check-i18n.sh` debe salir con código 0 antes de
compilar — descubre la lista de idiomas desde el propio archivo, por lo que un idioma
adicional no puede quedar sin comprobar.

Las traducciones de la interfaz web son un conjunto separado: `data/www/locales/*.json`
(9 archivos, ~143 claves cada uno, mismo conjunto de idiomas que el firmware desde que PT-PT lo unió).

## Mantener este archivo fiel

| Comando | Cuándo |
|---------|--------|
| `bash scripts/update_toc.sh` | Después de añadir, mover o renombrar un banner `// §N — ` |
| `bash scripts/check-codemap.sh` | Antes de finalizar cualquier edición en el .ino |
| `bash scripts/check-i18n.sh` | Después de cualquier edición en `i18n.h` |

Todos los tres se ejecutan en CI en cada push y pull request, por lo que los desvíos fallan la compilación
en lugar de acumularse en silencio. Tenga en cuenta que todos los tres estaban rotos en macOS
hasta el primer commit de este repositorio: dos usaban `mapfile`/`grep -P`/`declare -A`, lo cual
bash 3.2 y grep BSD no tienen, y `check-codemap.sh` mostraba
"VERIFICACIÓN DE CODEMAP FINALIZADA" mientras no verificaba absolutamente nada.

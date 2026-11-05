# Registro de cambios

Cambios destacados en la firmware TigerScale V3. El formato sigue aproximadamente
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); las versiones son el valor
de `TIGERSCALE_FW_VERSION`, que el flujo de publicación de la versión rechaza
publicar si discrepa con la etiqueta git.

Las notas completas de cada versión viven en `docs/release-notes/vX.Y.Z.md`, que también es el
cuerpo de la publicación en GitHub y lo que el dispositivo enlaza como "¿qué hay de nuevo". El flujo
de publicación rechaza publicar una versión cuyas notas estén ausentes o aún contengan el
texto de esqueleto del molde.

## [No liberado]

Nada aún.

## [v3.13.0](docs/release-notes/v3.13.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

### Corregido
- **Los dos lectores se bloqueaban mutuamente.** El bucle de pesaje consultaba un lector por
  ciclo, por lo que sus consultas nunca se superponían -- pero el lector que esperaba su turno
  seguía emitiendo, y con dos antenas 75 mm apartadas mirándose el uno al otro ese portador estático era lo que el lector de lectura tenía que escuchar su etiqueta a través de. El campo del lector inactivo ahora baja durante la duración de la consulta de su vecino, en el bucle de pesaje y en la autocalibración igualmente. Medido sobre los niveles 2-8 en una unidad de bench: el lector más débil responde al 60% de las consultas con ambos portadores activos y al 89% con el lector inactivo apagado, y la capacidad aumenta porque menos consultas se pierden. Tabla completa en `docs/RFID-PERFORMANCE.md`.
- **`antennaOn()` / `antennaOff()` nunca cambiaron nada.** Establecían un booleano
  y devolvían, por lo que ambos campos estaban siempre activos -- incluyendo cuando
  `rfidAntennaSetAll()` creía que había silenciado a los lectores para una plataforma vacía. Ahora
  envían `RFConfiguration` CfgItem 0x01, en los tres transportes.
- **La ventana de consulta era demasiado estrecha**: 30 ms y un reintento de activación dejaban sin espacio
  para la activación ISO14443A propia del PN532, por lo que una etiqueta respondiendo ligeramente tarde
  se reportaba como ninguna tarjeta. Ahora 100 ms y dos reintentos -- propuesto en PR #40 y
  medido aquí en lugar de tomado por confianza, lo cual por sí solo lleva al lector más débil del 60% al 69% y eleva la capacidad de lectura de 13,3 a 16,7 lecturas por segundo.
- **Cada báscula anunciaba el mismo nombre mDNS.** `macSuffix4()` leía
  `WiFi.macAddress()` en la primera línea de `setupWiFi()`, antes
  de `WiFi.mode(WIFI_STA)`; sin ninguna redif no se devuelve NULL *sin escribir el buffer*, por lo que `mac[4]`/`mac[5]` eran bytes de pila no inicializados -- idénticos en cada unidad, ya que es la misma firmware en la misma ruta de encendido.
  `tigerscale-CE3F` eran esos bytes, no una MAC. Dos básculas en una LAN competían por el nombre y el respondiente renombraba al perdedor, sin regla sobre quién ganaba, por lo que una dirección marcada podría apuntar a la otra báscula tras un reinicio.
  `esp_read_mac(mac, ESP_MAC_WIFI_STA)` lee el efuse y no necesita ninguna redif.
  **El nombre de cada báscula cambia con esta actualización y no se mantiene ningún alias.**

### Cambiado
- La autocalibración mide en el régimen en el que opera la báscula. Escaneaba con ambos portadores activos, lo cual ya no es cómo lee el bucle de pesaje, y elegía un nivel que no es el mejor bajo alternación.
- Nivel de RF predeterminado compilado 3 -> 7. Bajo alternación, 5 y 7 son los únicos niveles donde ambos lectores responden a cada consulta en ambas series de bench, mientras que 3 pierde aproximadamente uno de seis en el lado más débil. Las unidades con un nivel ya almacenado en NVS lo mantienen.

### Añadido
- `docs/RFID-PERFORMANCE.md` -- 64 celdas sobre cuatro regímenes en una unidad de bench saludable: cada lector solo, ambos portadores activos, y alternando, cruzado con los dos tiempos de consulta. Registra el método, los números, la recomendación y las advertencias, incluyendo la comprobación de que ningún lector nunca leyó la etiqueta del otro a cualquier nivel de potencia.

## [v3.12.0](docs/release-notes/v3.12.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

### Corregido
- **El lector HSU deja de devolver UIDs después de la primera lectura de una etiqueta de 7 bytes (NTAG /
  TigerTag).** `Adafruit_PN532` lee un fijo 20 bytes de la respuesta `InListPassiveTarget`, que es 22 bytes para un UID de 7 bytes; los 2 restantes en el UART se interpretaron como el ACK del siguiente comando, por lo que cada consulta posterior falló inmediatamente. El buffer de recepción ahora se vacía antes de cada consulta y después de cada lectura correcta. Los marcos de respuesta de `RFConfiguration` y `setPassiveActivationRetries`, que la biblioteca tampoco lee, se vacían de la misma manera.
- **El lector HSU: una lectura de UID de 4 bytes se detuvo durante un segundo completo.** La misma lectura de 20 bytes esperaba un byte 20 que una respuesta de 19 bytes nunca envía, por lo que se encontró con el tiempo de espera predeterminado de 1000 ms del flujo. El puerto serie del lector ahora usa un tiempo de espera de 50 ms.
- **Una báscula podría encenderse medio ciega y permanecer así.** `setupRFID()` juzgaba la presencia de un lector pidiéndole su versión de firmware -- una consulta que el comentario propio del firmware dice no es una prueba de presencia -- y escribía ese veredicto en NVS, por lo que un encendido flaco hizo que la báscula fuera de lector único para siempre. Un lector que el probador no puede alcanzar ahora se considera no probado en lugar de ausente: se consulta una vez por segundo en lugar de cada ciclo, y se promueve al instante que lee una etiqueta. El contador de lectores almacenado registra si fue inferido o elegido por el propietario.
- **Un lector que falló en la consulta de versión se configuró sin configurar.** `init()` devolvió antes de `SAMConfig()`, la sintonización de RF y `setPassiveActivationRetries()`, y los valores de reinicio del PN532 son los de la cima de la tabla de niveles -- por lo que sin configurar significaba máxima potencia, con una segunda antena 75 mm lejos mirándolo.
- El arranque del lector recibe tres intentos con el pulso de reinicio fuera del bucle de reintentos, una ventana de 250 ms para la consulta de versión y el enlace se vacía hasta que esté tranquilo antes de un comando de configuración. `RFTUNE` ya no reporta un fallo de configuración que no ocurrió.

### Añadido
- **Autocalibración de potencia de RF** (Configuración > RFID > Autocalibración). Escanea cada nivel con un rollo etiquetado en lugar y mantiene el que da el mejor resultado al lector más débil, en dos pasos para que la respuesta sea reproducible. Cada nivel se registra.

### Cambiado
- El rango de potencia de RF se amplió de 5 niveles a 16, desde cero de impulso hasta los valores de reinicio del datasheet del PN532. El valor predeterminado permanece igual; las unidades con un nivel ya almacenado en NVS lo mantienen.

## [v3.11.0](docs/release-notes/v3.11.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.10.0](docs/release-notes/v3.10.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.9.0](docs/release-notes/v3.9.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.8.1](docs/release-notes/v3.8.1_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.8.0](docs/release-notes/v3.8.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.7.6](docs/release-notes/v3.7.6_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.7.5](docs/release-notes/v3.7.5_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.7.4](docs/release-notes/v3.7.4_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.7.3](docs/release-notes/v3.7.3_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.7.2](docs/release-notes/v3.7.2_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.7.1](docs/release-notes/v3.7.1_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.7.0](docs/release-notes/v3.7.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.6.0](docs/release-notes/v3.6.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.5.0](docs/release-notes/v3.5.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.4.0](docs/release-notes/v3.4.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.3.0](docs/release-notes/v3.3.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.2.0](docs/release-notes/v3.2.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.1.3](docs/release-notes/v3.1.3_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.1.2](docs/release-notes/v3.1.2_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.1.1](docs/release-notes/v3.1.1_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.1.0](docs/release-notes/v3.1.0_ES.md)

Consulte las notas de la versión para obtener la entrada completa.

## [v3.0.0](docs/release-notes/v3.0.0_ES.md)

Las actualizaciones por aire funcionan por primera vez — tres fallos apilados corregidos — más
8 MB de PSRAM que nunca se había activado, y un instalador de navegador.

**Cambia la tabla de particiones, por lo que solo se instala por USB.** Todo lo que está después de eso
puede hacerse por aire.

Entrada completa: **[docs/release-notes/v3.0.0.md](docs/release-notes/v3.0.0_ES.md)**

<!-- La entrada de abajo data antes de las versiones con número. Se mantiene porque explica por qué el
     repositorio se ve así. -->

## Preparación del repositorio

Primera publicación de la firmware V3. El desarrollo hasta este punto data antes
del control de versiones — no había commits — por lo que esta entrada cubre el estado
del repositorio al abrirlo, más la limpieza que produjo el primer commit.

### Eliminado

- **La ruta de código NFC USB-host** (155 líneas): la rama del lector `RFID_TRANSPORT_USB`, sus variables globales, su diagnóstico en pantalla, el entorno de compilación `esp32s3_usbhost` y la librería integrada `usbhost_lib/` fork. Nunca funcionó — la ranura USB-C del placa está cableada como dispositivo, no como host, con resistencias de pull-down fijas en CC y sin fuente de VBUS. Conservada como [docs/USB_HOST_POSTMORTEM.md](docs/USB_HOST_POSTMORTEM_ES.md). La eliminación del bloque de diagnóstico también eliminó una llamada en tiempo real `displayMessage()` que dibujaba gráficos brutos sobre una pantalla LVGL cargada, dejando píxeles estancados detrás.
- **La librería integrada `debug_lib/` fork de PN532**, reemplazada por la bandera de compilación `-DPN532DEBUG`. Su única diferencia útil con la versión upstream era un solo `#define` sin comentario; la librería ya protege todas las 23 ubicaciones de depuración con `#ifdef`. La traza de nivel de byte permanece igual, mediante `esp32s3_hsu_debug` / `esp32s3_i2c_debug`.
- **16,6 MB de archivos no referenciados** desde el control de versiones (mantenidos en disco): seis encabezados de logotipos de socios con cero inclusiones y cero uso de símbolos, dos copias idénticas md5 de `partners_splash.h`, dos *diferentes* copias de `tiger_tag_logo.h`, un binario compilado de punto de restauración, y cuatro PDFs de terceros ahora enlazados desde [docs/HARDWARE.md](docs/HARDWARE_ES.md) en lugar de redistribuidos.
- **`build_opt.h`**, cuyas cinco banderas de ajuste de AsyncTCP no se aplicaron en ningún lugar: el IDE Arduino lee este archivo desde la carpeta del esquema, donde estaba vacío, y PlatformIO lo ignora por completo. Eliminado en lugar de activarlo silenciosamente, ya que habilitarlos cambiaría el comportamiento de red y RAM y requiere una prueba de bench. Seguimiento como un problema.

### Corregidos

- **218 secuencias de mojibake en el código fuente del firmware.** El daño estaba doblemente codificado — la cadena literal `ï¿½` en lugar de `U+FFFD` — y había destruido cada `§`, guion em, elipsis, `±`, `×`, `µ` y acentos franceses en los comentarios. Una pasada anterior había convertido `→` en `?` en 24 lugares adicionales.
- **`scripts/update_toc.sh` se ajustó a la secuencia de mojibake en lugar del `§`**, por lo que la corrupción se integró en la herramienta y la tabla de contenido generada también estaba corrompida. También omitió las dos secciones escritas en otros formatos de encabezado, lo cual es por qué §5, §AUDIO y toda la sección LVGL nunca aparecieron en la TOC.
- **`scripts/check-codemap.sh` reportó éxito sin verificar nada** en macOS. Usó `mapfile` y `grep -P`, que bash 3.2 y grep BSD no proporcionan, por lo que su lista de anclas salió vacía y tomó la rama "nada que verificar" — imprimiendo `CODEMAP check PASSED`. Ahora trata una lista de anclas vacía como un error y resuelve la última coincidencia de la columna 0 para que también pueda verificar las dos anclas previamente marcadas "saltar auto-verificación".
- **`scripts/check-i18n.sh` salió 0 después de fallar.** Usó `declare -A` (bash 4+) y murió con `EN: variable no ligada` mientras aún reportaba éxito. La lista de idiomas ahora se deriva de `i18n.h` mismo, por lo que agregar un noveno idioma no puede dejar la verificación atrás.
- **OTA apuntaba al manifiesto del repositorio V2** — hardware diferente, cuyo `firmware.bin` arrancaría una unidad V3, y un manifiesto compartido tendría cada generación ofreciendo el binario del otro como actualización. Ahora apunta a las páginas de este repositorio, con `version.json` generado desde `TIGERSCALE_FW_VERSION` para que no puedan divergir.
- **Dos comentarios que contradecían el código**: el comentario de transporte afirmaba que el lector I²C estaba sobre `Wire` cuando la clase lo construye con `&Wire1`, y el banner LVGL aún describía a sí mismo como una pantalla de prueba experimental mucho después de convertirse en la UI de producción. Ambos habían engañado sesiones de depuración real.
- **Scripts de ayuda específicos de la máquina.** `upload_all.ps1` codificaba rutas `C:\Users\…` de un desarrollador, apuntaba a una carpeta "TigerScale V4" y usaba la ruta de `arduino-cli` prohibida por este proyecto; ahora es `scripts/flash.sh`, que funciona en macOS, Linux y Windows. `watch_logs.py` ya no tiene una IP LAN integrada.

### Añadidos

- `README.md`, `AGENTS.md`, `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `THIRD_PARTY_LICENSES.md`, `LICENSE` (MIT), y un conjunto `docs/` cubriendo hardware, instalación, internos, nube/privacidad y solución de problemas.
- CI que construye los cinco entornos y ejecuta los tres scripts de guardia como puertas reales, incluyendo una verificación que rechaza mojibake y una tabla de contenido desactualizada.
- Un flujo de lanzamiento que verifica la etiqueta contra `TIGERSCALE_FW_VERSION`, construye cada transporte, adjunta todos a la versión y genera el manifiesto OTA.
- Un banner de sección `§LVGL`, por lo que el puente LVGL y la pantalla principal de peso aparecen en la tabla de contenido en lugar de estar ocultos dentro de la sección OTA.

### Problemas conocidos

- OTA publica un solo binario, construido para HSU. Una unidad cableada para SPI o I²C que lo tome pierde su lector hasta que se reflashe por USB.
- Las construcciones SPI e I²C compilan pero no se han verificado en bench.
- `downloadUserAvatar()` se sospecha que bloquea el dispositivo al recibir una URL válida.
- `readInventoryContainerWeight()` devuelve 0 en cualquier fallo, sin reintentar.
- Un informe de reinicio después de dos pesajes no se pudo reproducir en 11 ciclos posteriores; el heap libre se ha visto en ~15–23 KB durante picos de Firebase HTTPS.
- La API HTTP local no está autenticada por diseño.
- El banner de §7 aún lee "PANTALLA OLED"; no hay OLED en V3.

## Historia anterior

Antes de este repositorio. El firmware V3 fue desarrollado como una rama del [V2 scale](https://github.com/TigerTag-Project/Tiger-Scale) — placa diferente, pantalla diferente, PN532 en lugar de RC522 — y los hitos a lo largo del camino (la migración de UI LVGL, el arranque de HSU, ajuste de potencia RF para interferencia de antena, OTA y la corrección del pitido fantasma al remover) existen solo como narrativa en los registros de trabajo pre-commit.

# CLAUDE.md — FIRMWARE V3 TigerScale

Reglas para agentes de IA que trabajan en este repositorio. Todo lo aquí descrito es verdadero para el código tal como está; ninguno de ello es historia de sesión. Cuando algo cambia, este archivo cambia junto con ello — no se deben añadir notas fechadas que contradigan las secciones anteriores. Eso es cómo la versión anterior creció a 27 KB y terminó abriendo con una advertencia que sus propios contenidos eran obsoletos.

Documentación orientada al usuario vive en [`README.md`](README.md) y [`docs/`](docs/).
Instrucciones para agentes sin dependencia de herramientas están en [`AGENTS.md`](AGENTS_ES.md); este archivo es el superset específico de Claude.

## ¿Qué es esto

Firmware para una balanza conectada de filamento: coloque un bobinado portador de una etiqueta NFC TigerTag en la plataforma, la balanza lee la etiqueta, pesa el bobinado, resta el peso del bobinado vacío y sincroniza el peso neto del filamento con la cuenta TigerTag del propietario.

- **Placa** — Waveshare ESP32-S3-Touch-LCD-3.5**B**, o el **-3.5** sin la B (ESP32-S3, 16 MB de flash, PSRAM). Una bandera de compilación aparte, vea la tabla de entorno a continuación
- **Pantalla** — 480×320 en modo paisaje (rotación 3), LVGL v8.4. AXS15231B sobre QSPI en el -3.5B, ST7796 sobre SPI en el -3.5
- **NFC** — 2× PN532, envueltos por la clase `PN532Reader` (§5)
- **Pesar** — HX711 + celda de carga
- **Alimentación** — USB; AXP2101 PMIC reporta nivel y estado de carga para la batería opcional
- **Audio** — ES8311 codec (sonido al detectar etiqueta)
- **Origen** — un solo archivo Arduino C++, aproximadamente 17 200 líneas, más `i18n.h`
- **Compilación** — solo PlatformIO (vea abajo por qué el IDE Arduino no puede compilarlo)

## No negociables

| Regla | Consecuencia de ignorarla |
|------|----------------------------|
| Compilar el entorno que coincida con el **placa física y cableado** — `esp32s3_hsu_b` para la unidad verificada en el banco | Placa incorrecta = una balanza que parpadea correctamente pero nunca enciende su pantalla. Transporte incorrecto = firmware que detecta ningún lector y dice nada sobre por qué. La segunda costó una sesión completa |
| Modo de flash permanece **DIO** (ya establecido por la configuración de la placa) | QIO da un bucle de reinicio en `ets_loader.c` |
| Nunca usar `ps_malloc` | Devuelve null en silencio. Use `heap_caps_malloc(n, MALLOC_CAP_SPIRAM)` con un respaldo de DRAM |
| Mantener `ARDUINO_USB_CDC_ON_BOOT=1` | Sin ello `Serial.println()` es invisible en el puerto USB-C; solo `log_e()` pasa |
| Nunca flashear `firmware.factory.bin` en `0x0000` | Es una imagen fusionada que abarca desde `0x0000`, por lo que sobrescribe NVS (0x9000–0xE000) y borra credenciales WiFi + Firebase guardadas. `scripts/flash.sh` escribe cada imagen en su propio offset y las preserva |
| Leer [`CODEMAP.md`](CODEMAP_ES.md) antes de abrir el .ino | Es sobre 17 200 líneas. Leerlo completo desperdicia el contexto que la tarea real necesita |
| Terminar con `bash scripts/verify.sh` | Ejecuta todo lo que CI ejecuta. Saltar solo mueve el fallo al push |

## Trabajar en el .ino

Nunca lea todo el archivo. El ciclo es:

1. [`CODEMAP.md`](CODEMAP_ES.md) → encontrar la sección o función.
2. `grep -n "theName" TigerTagSplashESP32/TigerTagSplashESP32.ino` — el grep es la verdad; el número de línea en CODEMAP solo es un punto de partida.
3. Leer esa línea ±60.
4. Hacer el **más pequeño** cambio que haga el trabajo. No limpieza oportunista.
5. `bash scripts/verify.sh --fix` — regenera la tabla de contenido y los números de línea de CODEMAP, luego verifica todo. Nunca renumerar CODEMAP a mano.
6. Paralelizar lecturas independientes en un solo mensaje.

**CODEMAP.md tiene una tabla "Minas".** Lea la fila para cualquier función que vaya a tocar. Cada entrada está allí porque costó una sesión de depuración: `computeWeightAvailable` dio dos respuestas diferentes para el mismo bobinado dependiendo de qué camino de código ganó la carrera, `readWeight` puede parecer roto cuando la verdadera falla es un cable de celda de carga intercambiado, `processAutoTare` no debe tener una variante de desbordamiento corto reañadida, y así sucesivamente.

### Banderas de sección

Las secciones están marcadas `// §N — TÍTULO`. Después de añadir, mover o renombrar una, ejecute
`bash scripts/update_toc.sh` para regenerar el bloque de comentario TOC al principio del archivo. §12 está intencionalmente ausente, y §AUDIO / §LVGL están intencionalmente sin número — vea CODEMAP.md para saber por qué.

## El flujo de trabajo, en cuatro comandos

Todo lo mecánico es regenerable; nada aquí debería nunca editarse a mano.

```bash
bash scripts/verify.sh --fix     # regenerate TOC + CODEMAP, then run every check
bash scripts/verify.sh --all     # what CI runs: all checks, all five envs
bash scripts/flash.sh --fs       # build and flash the connected device
bash scripts/bump-version.sh X.Y.Z   # version + scaffold release notes + changelog
```

| Síntoma | Solución |
|---------|-----|
| `check-codemap` reporta desviación | `python3 scripts/sync-codemap.py` |
| CI dice que el TOC está desactualizado | `bash scripts/update_toc.sh`, commitar el .ino |
| comprobación i18n falla | nombra el idioma y la clave faltante, o la posición donde el orden diverge |
| comprobación de fuente UI falla | una cadena usa un carácter que ninguna fuente embarcada lleva. Nombra cuál: una letra acentuada significa `bash scripts/make-latin-font.sh`, una Han significa `bash scripts/make-cjk-font.sh` |
| comprobación de traducción UI falla | una cadena literal que lleva una palabra se pasa a un setter de texto LVGL, por lo que llega al panel en un solo idioma. Nombra la línea y la literal. Añadir una clave a `i18n.h` en todos los once bloques y usar `t(I18N_KEY)` — o, si realmente es neutral de idioma, añadirlo a `ALLOWED` en `scripts/check-ui-translated.py` con el motivo |
| comprobación de archivo generado falla | una cabecera de bitmap cuya array ya no coincide con su propio `_W`x`_H`, o una fuente cuya tabla de glifos ya no coincide con el rango que su cabecera declara. Ambos se regeneran, nunca se editan a mano: `scripts/make-rgb565-header.py`, `scripts/make-latin-font.sh`, `scripts/make-cjk-font.sh` |
| comprobación de nombre del dispositivo falla | un documento muestra un nombre mDNS que no es `tigerscale-%02X%02X` — normalmente un nombre sin sufijo MAC en absoluto, que resuelve a nada, o uno con sufijo en minúscula. El guardián nombra al culpable cuando se dispara. El firmware construye el nombre en `macSuffix4()`; el documento es lo que está mal |
| comprobación de llms.txt falla | el resumen escrito para modelos de lenguaje ha divergido del repositorio: un nombre de entorno que ya no existe, una cuenta de líneas obsoleta, un enlace de documentación muerto, o una lista de scripts de guardián donde `bash scripts/verify.sh` debería estar. Es el único archivo que ningún humano lee, por lo que se comprueba mecánicamente |
| comprobación de formato de archivo falla | un archivo de texto rastreado es CRLF, lleva un BOM, o contiene un control invisible/bidi. Nombra el archivo y la línea. `.gitattributes` ya pide LF, pero solo enlaza un cliente `git add` — un commit hecho mediante el editor web o API de GitHub bypassa eso, lo cual es cómo una cambio de nueve líneas llegó una vez como un diff de 16,000 líneas |
| flujo de lanzamiento rechaza publicar | `docs/release-notes/v<version>.md` está faltando o aún lleva el texto esqueleto |

**Subir necesita la cuenta GitHub correcta.** El repositorio pertenece al usuario `TigerTag-Project`, no a `BenGlut`, y un push de repositorio privado desde el incorrecto falla con "Repositorio no encontrado":

```bash
gh auth switch -u TigerTag-Project && git push && gh auth switch -u BenGlut
```

**Comentarios de documentación e instalador-página no necesitan un lanzamiento.** La versión pertenece al firmware, y moverla sin mover el firmware es una afirmación que cada balanza en el campo entonces actúa sobre.

| Lo que cambió | Lo que ocurre al push |
|------------------|----------------------|
| `README.md`, `docs/`, comentarios | guardias solo — la matriz de construcción de cinco entornos se salta |
| `web-installer/` | guardias, más `pages.yml` se vuelve a desplegar en unos minutos |
| el .ino, `data/`, `platformio.ini`, `partitions.csv` | la matriz de construcción completa |
| una etiqueta `v*` | `release.yml` construye y publica; `pages.yml` luego despliega |

**`pages.yml` es la única cosa permitida para desplegar Pages.** `release.yml` deliberadamente no lo hace, y no debe ser dado esa tarea de nuevo. Dos workflows desplegando para el mismo commit es lo que rompió una vez: GitHub reportó ambos despliegues como exitosos, marcó el más antiguo como inactivo y siguió sirviéndolo, por lo que `version.json` anunció la versión anterior y cada balanza siguió reportando "actualizado" contra un lanzamiento que ya estaba fuera. Nada se mostró rojo en ningún lugar — la única forma de verlo era obtener el sitio y compararlo con el lanzamiento.

Tres propiedades mantienen que no se repita, y todas las tres importan:

- **Se ensambla alrededor de la última *versión de lanzamiento*, nunca alrededor de lo que está en vivo.** Así produce el mismo sitio correcto ya sea antes o después de un lanzamiento, y el ordenamiento deja de ser una pregunta.
- **No reconstruye nada.** Las imágenes de la aplicación son los activos de lanzamiento mismos, así que el SHA-256 en `version.json` es el hash de los bytes exactos que OTA descargará. `otaApply()` verifica ese hash antes de cambiar la partición de arranque; una binario reconstruido sería descargado y luego rechazado por cada balanza.
- **Obtiene el sitio y lo verifica** (`scripts/verify-published-site.py`), y una ejecución horaria repara el sitio si un despliegue se pierde de nuevo.

Ejecute ese verificador a mano cualquier vez que dude de lo que las balanzas pueden ver:

```bash
python3 scripts/verify-published-site.py
```

**Lanzar** es una etiqueta; todo lo demás es automático:

```bash
bash scripts/bump-version.sh 3.1.0    # then write the notes
git commit -am "Release v3.1.0" && git tag v3.1.0 && git push origin main --tags
```

El flujo luego verifica la etiqueta contra `TIGERSCALE_FW_VERSION`, ejecuta las guardias, rechaza continuar sin notas de lanzamiento, construye los tres transportes más el sistema de archivos, publica el lanzamiento de GitHub, y despliega el instalador web y el manifiesto OTA a Pages — ambos generados desde el mismo script, así que no pueden ofrecer versiones diferentes.

## Construir y flashar

```bash
pio run -e esp32s3_hsu_b        # build (reference env)
bash scripts/flash.sh           # build + flash, keeps saved WiFi
bash scripts/flash.sh --fs      # also upload the web UI to LittleFS
bash scripts/flash.sh --monitor # then open the serial console
```

<documento>
Los transportes son una elección en tiempo de compilación, nunca en tiempo de ejecución:

| entorno | Placa | Transporte | Publicado | Estado |
|--------|------|------------|-----------|--------|
| `esp32s3_hsu_b` | -3.5**B** | UART (HSU), 2 lectores | sí | referencia, verificado en bench |
| `esp32s3_hsu` | -3.5 | UART (HSU), 2 lectores | sí | compila, **no** verificado en bench |
| `esp32s3` | -3.5B | SPI, 2 lectores | no | compila, no verificado en bench |
| `esp32s3_i2c` | -3.5B | I2C en `Wire1`, 1 lector | no | compila, no verificado en bench |
| `esp32s3_hsu_debug` / `esp32s3_i2c_debug` | -3.5B | como arriba, más `-DPN532DEBUG` | no | rastreo a nivel de byte del PN532 |

**La placa es una elección en tiempo de compilación como el transporte.** Las dos variantes de Waveshare comparten un netlist y difieren en exactamente un GPIO (12: `LCD_CS` en la B, `I2S_MCLK` en la otra), por lo que el diagrama de conexión y la carcasa son idénticos — pero el controlador de panel difiere (AXS15231B/QSPI vs ST7796/SPI), por lo que la imagen incorrecta es una pantalla negra sin diagnóstico. `docs/HARDWARE.md#board-variants` tiene la comparación completa, derivada de ambos esquemas oficiales.

**Solo `esp32s3_hsu_b` puede llenar las claves OTA planas.** Cada escala en el campo es una -3.5B y lee `version`/`firmware_sha`/`firmware_url` desde el nivel superior de `version.json`; la -3.5 lee `boards["3.5"]` y deliberadamente no tiene retroceso. `scripts/make-manifest.py` rechaza generar un manifiesto cuyo `--ota-env` sea algo otro, porque ese error llegaría a todas las escalas de inmediato y no podría deshacerse desde el dispositivo.

Diagrama de conexión para cada uno: [`docs/HARDWARE.md`](docs/HARDWARE_ES.md).

**El IDE Arduino no puede compilar este proyecto.** La configuración de LVGL se encuentra mediante `-I include` junto con `LV_CONF_INCLUDE_SIMPLE`, y el IDE no ofrece una ruta de inclusión equivalente. No pases tiempo en ello, y no lo ofrezcas a los usuarios como opción.

## Hechos de hardware que afectan

- **`Wire` (GPIO21/22) no funciona en esta placa.** GPIO22 no existe en la ESP32-S3 (la placa salta los pines 22–25) y GPIO21 va al conector de la cámara. Cada escaneo de ese bus al encender encuentra 0 dispositivos. No coloques nada nuevo en él — el bus funcional es `Wire1` (SDA=GPIO8, SCL=GPIO7), que lleva el controlador táctil (0x3B en la -3.5B, 0x38 en la -3.5), AXP2101 @0x34, ES8311 @0x18 y el expander TCA9554 @0x20 — que es donde está la línea de reinicio del panel. El PN532 fijo 0x24 no colisiona.
- **El NFC como host USB es imposible en esta placa**, confirmado tanto en el banco de pruebas como en el esquema. No lo intentes de nuevo:
  [`docs/USB_HOST_POSTMORTEM.md`](docs/USB_HOST_POSTMORTEM_ES.md).
- **Los dos antenas del PN532 están ~75 mm separados mirándose** y se cruzan. Por eso la potencia/ sensibilidad de RF es un ajuste en tiempo de ejecución de 16 niveles (`applyPN532RfTuning`, nivel predeterminado 7, ajustable desde Configuración → RFID y persistido en NVS) en lugar de un valor fijo. El portador del lector inactivo también se apaga mientras el otro lee — eso solo eleva al lector más débil del 60 % de respuestas a 89 %
  ([`docs/RFID-PERFORMANCE.md`](docs/RFID-PERFORMANCE_ES.md)). La misma pantalla's **Auto** encuentra el nivel empíricamente: escanea el rango con un rollo etiquetado en lugar y mantiene el que da el **menor** lector su mejor resultado. Calcular el total en lugar de eso elegiría el extremo superior del rango, donde una antena está sorda y la escala ve medio rollo — en una unidad saludable el lector correcto cae a una consulta cada cinco desde el nivel 9 en adelante, y un módulo realmente defectuoso lee nada allí.
- **Un segundo lector I2C necesita un multiplexor.** La dirección I2C del PN532 está fija en 0x24, por lo que dos módulos en un bus colisionan. En el build I2C `rfid2` es un stub permanente y el fallback de lector único aplicable sin cambios.
- **No hay motor en esta escala.** `setupServo()` deliberadamente hace nada. Las constantes del servo y `SERVO_PIN` permanecen para la variante con motor.

## Trabajando en la UI de LVGL

La UI completa es LVGL v8.4 — la pantalla de pesaje principal y cada subpantalla de configuración. Dos reglas de proceso, ambas aprendidas de forma difícil:

1. **Una solicitud de restyling es solo visual.** Cambia colores, fuentes, tamaños, formas, bordes, disposición. No cambies lo que se muestra, cómo se calcula o las coordenadas de la zona de toque — están conectadas al estado real del backend (campos de Firestore, lecturas en tiempo real de la escala) y son fáciles de confundir con chrome decorativo. Si no puedes determinar si una etiqueta es datos en tiempo real o texto estático, verifica la variable de origen antes de tocarla.
2. **Muestra un mockup antes de escribir cualquier cambio visual.** Constrúyelo en las proporciones reales de 480×320, itera sobre el mockup hasta que esté aprobado, *luego* edita el código de LVGL. Compilar, flashar y fotografiar el dispositivo es un viaje real de hardware; un mockup es instantáneo. Este proyecto ha quemado varios viajes innecesarios al omitir este paso.

Los trampas de LVGL v8 que este proyecto ha realmente golpeado están documentados en
[`docs/FIRMWARE.md`](docs/FIRMWARE_ES.md#lvgl-v8-traps) — léelos antes de editar un constructor de pantallas. Versión corta: nunca llames `lv_timer_handler()` desde dentro de un callback de clic; `lv_scr_load(new)` antes de `lv_obj_del(old)`; la rotación necesita un pivote explícito; `lv_obj_align_to()` centra el cuadro declarado en lugar del ink visible, así que usa una imagen para iconos pequeños; los widgets cuyos constructores llaman `lv_obj_align()` reinterpretan silenciosamente un posterior `lv_obj_set_pos()`; `LV_KEYBOARD_CTRL_BTN_FLAGS` hace que `LV_STATE_CHECKED` sea inútil para resaltar una tecla de control única; y nunca dejes que el dibujo raw-`gfx` ocurra mientras una pantalla de LVGL esté cargada.

3. **Refresca lo mínimo que cambió.** `lvglUpdateMainScreen()` establece las etiquetas que posee y deja que LVGL decida qué repintar; no reconstruye la pantalla. Destruir un contenedor para reflejar un valor cambiado es el error recurrente en cualquier UI de larga vida — pierde el desplazamiento y el foco, reinicia las animaciones y vuelve a ejecutar cada conexión. Reconstruye solo cuando la pantalla cambió realmente, y recuerda que `lv_scr_load(new)` viene antes que `lv_obj_del(old)`.

**No necesitas fotografiar el dispositivo para ver una pantalla, ni pedirle a nadie que lo toque.** Dos solicitudes, sin configuración, sin código de acceso (`XXXX` es el sufijo de cuatro hexadecimales de la unidad, desde los dos últimos bytes de su MAC; `LOCAL.md` tiene el de la bench):

```bash
curl -s -o s.bmp http://tigerscale-XXXX.local/screen.bmp   # what is on the panel
curl -s "http://tigerscale-XXXX.local/api/tap?x=415&y=285" # press Settings
curl -s "http://tigerscale-XXXX.local/api/tap?x=200&y=280&x2=200&y2=90"  # swipe
```

`/api/tap` responde solo una vez que la presión ha sido tomada *y* la pantalla ha terminado de cambiar, así que el toque y `screen.bmp` se encadenan sin inventar una demora — la respuesta es la señal. `x2`/`y2` lo convierte en un deslizamiento, que es la única forma de desplazar una lista desde un terminal: LVGL decide si es arrastre o toque desde el movimiento entre dos lecturas de entrada, así que el toque se camina en doce pasos en lugar de saltar. `/screen.ver` es cuatro bytes del contador de marco, para consultar de forma económica en lugar de arrancar 450 KB. `/screen` es el mismo bucle para una persona — la imagen, donde hacer clic toca y arrastrar desliza.

Las coordenadas son el espacio de la pantalla propio de 480x320 en modo paisaje. Una presión inyectada no salta la pantalla de sueño: en una pantalla oscura despierta la escala y es devorada, exactamente como un dedo, así que un comando remoto nunca actúa sobre una pantalla que nadie puede ver.

**Lo que este es y no es.** Es lo que LVGL dibujó, leído del lienzo. Geometría, texto, traducción, truncamiento, qué pantalla está abierta — confiable. Color como lo ve el ojo, retroiluminación, renderizado de la pantalla — no. Y no puede ver la pantalla misma: cualquier cosa que cambie la ruta de flujo o debajo parece perfecta aquí mientras el vidrio en la habitación está mal, lo cual ha ocurrido. Para esos, pregunta.

La vista antigua §LIVE aún existe y hace más — una transmisión en tiempo real en el puerto 81, `Configuración → LAN → Vista en vivo`, con un código de acceso. Busca cuando quieras ver; busca las dos solicitudes anteriores cuando quieras comprobar.
[`docs/FIRMWARE.md`](docs/FIRMWARE_ES.md#live-view) cubre ambos.

## i18n

`TigerTagSplashESP32/i18n.h` — **120 claves × 11 idiomas** (EN/PT/FR/ES/DE/ZH/IT/PL/PT-PT/RU/NL — lista exacta del Studio Manager).

1. Añade la clave al enum, antes de `I18N_COUNT`.
2. Añade una entrada a **cada** bloque de idioma, en el mismo orden que el enum.
3. `bash scripts/check-i18n.sh` debe salir con código 0 antes de compilar.

Una literal que lleve una palabra nunca debe ser pasada a un setter de texto de LVGL — llega a la pantalla sin traducir, y lo hacía en la pantalla RFID durante meses. `scripts/check-ui-translated.py` falla la compilación en eso ahora. Si una literal realmente es neutral de idioma, añádela a ese archivo's `ALLOWED` **con la razón**.

**Los acentos se restauran y se renderizan.** Cada entrada francesa, española, alemana, italiana, polaca y portuguesa ahora se escribe correctamente — incluyendo las letras extendidas Latin-A que el polaco necesita (`ą ć ę ł ń ś ź ż`). Antes se eliminaban, porque hasta que `font_latin_*` existió esas letras dibujaban como cajas en blanco; esa restricción ha desaparecido. `scripts/check-ui-fonts.py` falla la compilación si una cadena nunca usa un glyph que no lleve una cara compilada, y
`scripts/check-generated.py` falla si una cara deja de coincidir con el rango que su propio encabezado declara — lo cual es lo que esa primera comprobación confía. Las letras rusas viajan en las mismas `font_latin_*` (Cyrillic U+0400–045F más `Ґ ґ`, vea
`scripts/make-latin-font.sh`); los neerlandeses necesitan nada más allá de Latin-1. Escribe una nueva cadena correctamente escrita y deja que los guardias decidan.

Las traducciones de la UI web son un conjunto separado: `data/www/locales/*.json` (11 archivos, el mismo conjunto de idiomas que el firmware).

## Convenciones del repositorio


</documento>

- **[`WORKLOG.md`](WORKLOG_ES.md)** es la única fuente de verdad para todo lo realizado desde el último commit — léalo al inicio de una sesión y anexe al archivo en el momento en que se realice un cambio, en lugar de hacerlo en bloque al final. Sus encabezados son los de Keep a Changelog, por lo que se sintetiza una entrada de lanzamiento a partir de él en lugar de rederivarse desde la diferencia. Describa el estado final, no el trayecto: una "Agregado X" y una posterior "Corregido X" del mismo ciclo se fusionan en una entrada, y cualquier cosa revertida desaparece. En un punto de control, sintetícelo en una línea, úsela como mensaje de commit y reinicie el archivo a su encabezado.
- **La versión reside en un solo lugar**: `TIGERSCALE_FW_VERSION` en §2. El flujo de lanzamiento rechaza la publicación si la etiqueta de git no coincide con ella, y genera el manifiesto publicado a partir de ella mediante `scripts/make-manifest.py` — no existe un archivo `version.json` comprometido, ya que una copia duplicada es cómo se separan los dos. El canal over-the-air y el instalador web leen ese único archivo generado, por lo que siempre ofrecen la misma compilación.
- **Secretos**: no hay ninguno en este repositorio, y ninguno debería agregarse. La clave pública de la API Web de Firebase en §1 es la clave cliente que cada aplicación cliente de Firebase envía, y se documenta como tal en su definición. Nunca comprometa credenciales de WiFi, tokens o direcciones de red personales — `scripts/watch_logs.py` toma la dirección del dispositivo como argumento exactamente por esta razón.
- **`LOCAL.md` (ignorado por git) contiene la verdad local del equipo** — la dirección y el código de acceso de la báscula de prueba, la puerto serie, cuál cuenta empuja dónde. Léalo cuando necesite uno de esos en lugar de preguntar; actualícelo cuando cambie uno. Existe para que ningún archivo comprometido tenga que llevar una dirección o una ruta absoluta, y para que una sesión no gaste un viaje de ida y vuelta re-aprendiendo lo que la última ya sabía.
- **Los binarios van a GitHub Releases, nunca al git**. Un binario comprometido permanece en el historial para siempre, incluso después de ser eliminado.
- **El trabajo diferido va a GitHub issues**, no a un archivo. `gh issue list` es la lista de pendientes; no existe `ROADMAP.md` y no debería existir uno — un segundo lugar es un segundo elemento que mantener verdadero. Un issue entra en el momento en que se detecta la incidencia, no cuando alguien decida programarlo: una incidencia que espera a ser planificada antes de ser registrada es una incidencia que se olvida.

  Pero verifique que merezca su lugar primero, porque un registrador en el que todo entra es un registrador que nadie lee y el verdadero defecto se ahoga entre las preferencias:

  1. **verificado, no sospechoso** — abra el código, nombre el archivo y la línea;
  2. **la consecuencia tiene un nombre** — qué falla, para quién, en qué caso. Una entrada sin uno es una preferencia;
  3. **no ya cubierto** por una guardia, una prueba o el código mismo;
  4. **no realizable ahora** — si la corrección encaja en la tarea actual, hágala;
  5. **sobrevive a la sesión** — un cuello de botella de la cadena de herramientas no pertenece allí.

  Indique lo que está mal, cuánto cuesta no corregirlo, **y por qué se pospuso**. Ese último es el que las personas omiten y el que detiene que la próxima sesión reabra el argumento desde cero.

- **Las revisiones viven en [`docs/reviews/`](docs/reviews/), un archivo cada uno, conservados permanentemente, trabajados desde el breve de inicio en [`docs/REVIEW-BRIEF.md`](docs/REVIEW-BRIEF_ES.md). Después de actuar sobre uno, anote cada hallazgo resuelto / pospuesto / rechazado **en el informe** — un informe no anotado se lee para el siguiente revisor como si nada se hubiera hecho, y los hallazgos se re-litiguen desde cero.
- **Instale los hooks una vez por clonación**: `bash scripts/install-hooks.sh`. Apunta `core.hooksPath` a `scripts/hooks/`, por lo que `pre-commit` ejecuta `verify.sh --quick` y las cosas mecánicas, regenerables ya no pueden ser la razón por la que un push vaya rojo. Intencionalmente no compila — una compilación en cada commit es cómo las personas aprenden a buscar `--no-verify`.

## Reglas duras

Lista corta, cada una ganada.

- **Nunca comprometa, etiquete o empuje sin ser preguntado.** Haga el cambio, ejecute las guardias, informe y deténgase. La única cosa que no puede deshacerse editando un archivo es una etiqueta que ya fue empujada: el flujo de lanzamiento actúa sobre ella.
- **Ninguna atribución de IA en ningún lugar** — ni en mensajes de commit, ni en cuerpos de PR, ni en comentarios de código o listas de contribuyentes.
- **Conversación en francés, todo comprometido en inglés** — código, comentarios, mensajes de commit, documentación, este archivo. Un repositorio que mezcla ambos es un repositorio que nadie fuera de la sala puede leer.
- **Un lector que falla la prueba de arranque no es ausente, sino no probado.** La consulta de la versión del firmware no es una prueba de presencia en este hardware — la unidad de prueba de la báscula responde ninguna de ellas y lee etiquetas en el mismo enlace. Por lo tanto, el veredicto de `setupRFID()` solo establece la frecuencia de escaneo: un lector inalcanzable sigue siendo escaneado una vez por segundo y se promueve al momento de leer. Nunca haga que ese veredicto sea definitivo nuevamente, y nunca lo persista como configuración del propietario.
- **Verifique su propio trabajo antes de informar.** Ejecute las guardias, lea la salida y revise el resultado. Pida al humano que pruebe solo lo que genuinamente necesita la hardware frente a él — y pídale una vez, en el momento que se necesita, no como sustituto de mirar. La vista en vivo existe precisamente para que "¿se ve bien?" sea una pregunta que pueda responder usted mismo (§LIVE).
- **Informe fielmente.** Si una verificación falló, muestre su salida. Si un paso fue omitido, diga cuál y por qué. Lo parcialmente hecho nunca se reporta como hecho.
- **Cuando una regla aquí resulte ser incorrecta, cámbiela en la misma sesión.** Una línea obsoleta en este archivo malenseña a cada sesión que sigue, lo cual es exactamente cómo la versión anterior alcanzó 27 KB de contradicciones.

## Guía de modelo y delegación

| Tarea | Sugerencia |
|------|-----------|
| Función única, menos de 30 líneas | cualquier modelo |
| Nueva característica que abarca 2+ secciones, o un refactor | un modelo más fuerte |
| Conversación larga, varias secciones ya leídas | cambie a búsqueda específica + fragmento; no re-lea |

Dos sub-agentes están definidos en `.claude/agents/`, ambos para mantener el costo de búsqueda fuera del contexto principal en lugar de para hacerlo más rápido:

- **`locator`** (modelo más barato, solo lectura) — "¿Dónde está el código que…". Hace su propia búsqueda en su propio contexto y devuelve rutas y números de línea.
- **`single-edit`** (modelo medio) — un cambio ya decidido, autocontenido. Su prompt debe nombrar el archivo exacto, el cambio exacto y el resultado esperado: ve nada de la conversación que decidió, y se le instruye para detenerse y preguntar en lugar de adivinar.

**Nunca delegue** un cambio de varias secciones, un nuevo subsistema, una sesión de depuración, la decisión de versión, las guardias, la lectura final o cualquier paso de git. Delegar esos pierde el razonamiento y cuesta más en rehacer que ahorra.

**Nunca se desvía.** Sub-agentes paralelos mantienen el contexto principal liviano pero multiplican el costo — cada uno lleva su propio prompt de sistema y relee independientemente los mismos archivos grandes. El predeterminado es hacerlo en línea; si el contexto es verdaderamente demasiado grande, pase el bloque completo a exactamente un agente con un prompt autónomo.

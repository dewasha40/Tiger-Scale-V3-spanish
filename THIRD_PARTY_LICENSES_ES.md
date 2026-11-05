# Licencias de terceros

La firmware de TigerScale V3 está [licenciada MIT](LICENSE). Construye contra las bibliotecas
a continuación, cada una de las cuales mantiene su propia licencia. Ninguna está incluida en este
repositorio — PlatformIO las obtiene en tiempo de compilación desde las versiones fijadas en
`platformio.ini`, por lo que el texto de licencia autoritario siempre se incluye con la
dependencia en sí misma.

| Biblioteca | Versión | Licencia | Usada para |
|-----------|--------|---------|-----------|
| [Biblioteca GFX para Arduino](https://github.com/moononournation/Arduino_GFX) | 1.6.5 | BSD-3-Clause | Controlador de panel QSPI AXS15231B, buffer de lienzo |
| [LVGL](https://github.com/lvgl/lvgl) | ^8.3.11 | MIT | La interfaz táctil completa |
| [WiFiManager](https://github.com/tzapu/WiFiManager) | ^2.0.16-rc.2 | MIT | Respaldo de provisión de WiFi |
| [ESPAsyncWebServer](https://github.com/mathieucarbou/ESPAsyncWebServer) | ^3.6.0 | LGPL-3.0 | Interfaz web en dispositivo + API JSON + WebSocket |
| [AsyncTCP](https://github.com/mathieucarbou/AsyncTCP) | ^3.3.0 | LGPL-3.0 | Capa TCP asincrónica bajo la anterior |
| [HX711](https://github.com/bogde/HX711) | ^0.7.5 | MIT | Amplificador de celda de carga |
| [Adafruit PN532](https://github.com/adafruit/Adafruit-PN532) | ^1.3.3 | BSD-3-Clause | Lector NFC (SPI / HSU / I²C) |
| [Adafruit BusIO](https://github.com/adafruit/Adafruit_BusIO) | ^1.16.1 | MIT | Abstracción de bus para la anterior |
| [ArduinoJson](https://github.com/bblanchon/ArduinoJson) | ^6.21.5 | MIT | Todos los análisis y serialización de JSON |
| [ESP32Servo](https://github.com/madhephaestus/ESP32Servo) | ^3.0.5 | LGPL-2.1 | Soporte para servomotores (no utilizado en esta compilación) |
| [JPEGDEC](https://github.com/bitbank2/JPEGDEC) | ^1.2.11 | Apache-2.0 | Decodificación de imágenes descargadas |
| [XPowersLib](https://github.com/lewisxhe/XPowersLib) | ^0.2.6 | MIT | AXP2101 PMIC (batería, carga) |
| [Core de Arduino para ESP32](https://github.com/espressif/arduino-esp32) | mediante pioarduino 55.03.39 | LGPL-2.1 / Apache-2.0 | Framework, componentes ESP-IDF |

## Una nota sobre LGPL

`ESPAsyncWebServer`, `AsyncTCP`, `ESP32Servo` y partes del core de ESP32 Arduino
están bajo LGPL. Si distribuyes un dispositivo con versiones modificadas de estas bibliotecas,
las obligaciones de LGPL se aplican a *esos componentes* — publica tus cambios en ellos y deja
que los usuarios los vuelvan a vincular. Compilar versiones no modificadas de las versiones originales, como lo hace este proyecto, es
directo. Este es un indicador, no asesoría legal.

## Activos

| Activo | Notas |
|--------|-------|
| Logo y marca de TigerTag | © Proyecto TigerTag. La licencia MIT cubre el código, no la marca — por favor, no uses las marcas de una manera que implique aval. |
| Íconos en `svg/` | Creación del proyecto, MIT, junto con elementos obtenidos de [SVG Repo](https://www.svgrepo.com/) bajo sus licencias respectivas (ver nombres de archivo) |
| Logos de marca en `data/www/img/` | Marcas registradas de sus respectivos dueños (eSun, SUNLU, Rosa3D, Landu, Jamghe, R3D), incluidos para identificar el material que describe una etiqueta. No es un aval, en ningún sentido. |
| Fuentes | Montserrat, incluido dentro de LVGL bajo la Licencia SIL Open Font License 1.1 |
| Glyphs latinos acentuados | `TigerTagSplashESP32/font_latin_*.c`, generados desde Montserrat Medium bajo la Licencia SIL Open Font License 1.1 — el mismo archivo que LVGL utiliza para construir sus propias fuentes, tomadas de la etiqueta `v8.4.0` de [lvgl/lvgl](https://github.com/lvgl/lvgl). Solo Latin-1 Supplement y Latin Extended-A, no la fuente. Regenerar con `bash scripts/make-latin-font.sh`. |
| Glyphs chinos | `TigerTagSplashESP32/font_cjk_*.c`, generados desde [Noto Sans SC](https://github.com/notofonts/noto-cjk) Medium (peso estático, etiqueta de liberación `Sans2.004`) bajo la Licencia SIL Open Font License 1.1. Solo los caracteres que utilizan nuestras propias traducciones están incluidos, no la fuente. Regenerar con `bash scripts/make-cjk-font.sh`. |
| | La cobertura para ambos se verifica mediante `scripts/check-ui-fonts.py`, que ejecuta verify.sh. |
| Glyphs de iconos | Un puñado de [glyphs Free de FontAwesome](https://fontawesome.com) viajan dentro de `font_cjk_*.c` — Solid (bloqueo U+F023, sol U+F185) y Brands (G de Google, U+F1A0); los archivos de fuente FA están bajo la Licencia SIL Open Font License 1.1, los iconos CC BY 4.0. El G de Google identifica la propia entrada de Google, no un aval. Obtenidos en tiempo de generación (versión `6.5.2`), nunca comprometidos. |
| `lv_font_conv` | MIT. Utilizado en tiempo de compilación para producir los archivos anteriores; obtenido bajo demanda, nunca comprometido. |

## Documentación del proveedor

Los datasheets y esquemas para el ESP32-S3, ES8311, PN532 y la placa Waveshare
son **intencionalmente no redistribuidos aquí**. Pertenecen a Espressif, Everest
Semiconductor, NXP y Waveshare respectivamente, y una copia local se vuelve obsoleta en silencio cuando el proveedor publica una revisión.
[docs/HARDWARE.md](docs/HARDWARE_ES.md) enlaza a las fuentes oficiales en su lugar.

## Correcciones

Si una entrada anterior es incorrecta, o una dependencia está faltando, por favor abre un problema —
obtener el atributo correcto es importante.

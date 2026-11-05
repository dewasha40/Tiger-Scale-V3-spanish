# Hardware

Pinout, topología de bus y conexión de cables para TigerScale V3.

Todo lo contenido aquí fue leído desde `TigerTagSplashESP32/TigerTagSplashESP32.ino` §1
(`CONFIGURACIÓN DE HARDWARE`) en lugar de transcribirse desde la memoria. Si cambias un
pin en el firmware, cambia también este documento.

## La placa

**Placa de desarrollo Waveshare ESP32-S3 3.5" IPS touchscreen** — la variante vendida como
*"ESP32-S3 3.5inch IPS 262K Color LCD Touchscreen Development Board (Without Case
& Camera), 320x480 Resolution"*
([Amazon](https://link.amazon/B0gaANfF5)), que es la placa exacta sobre la que se construye la unidad de referencia.

Confirmado leyendo el chip de esa unidad (`esptool flash-id`):

```
Chip type:          ESP32-S3 (QFN56) (revision v0.2)
Features:           Wi-Fi, BT 5 (LE), Dual Core + LP Core, 240MHz,
                    Embedded PSRAM 8MB (AP_3v3)
Crystal frequency:  40MHz
Detected flash size: 16MB
```

La pantalla es un AXS15231B, nativamente 320×480 en vertical, controlada mediante QSPI y utilizada rotada a 480×320 en horizontal (rotación 3). También en la placa: un PMIC AXP2101 para batería y carga, un codec de audio ES8311 y un conector para cámara (no poblado en esta variante — lo cual es por qué el GPIO21 no es usable para I²C, ver abajo).

> La unidad de referencia es la variante **`-3.5B`** — confirmado desde la silkscreen en el diagrama de conexión que se muestra a continuación, que también coincide con el esquema al que se refieren los comentarios del firmware sobre los pines de audio. Waveshare vende más de una placa ESP32-S3 de 3.5", por lo que si la tuya es una revisión diferente, verifica los pines I²S y I²C contra tu propio esquema antes de asumir que esta documentación se aplica.

Documentación oficial del proveedor (enlazada deliberadamente en lugar de copiada en este repositorio — los PDFs del proveedor son de su propiedad para distribuir, y una copia local estancada se desvía silenciosamente de la revisión actual):

- [Wiki Waveshare ESP32-S3-Touch-LCD-3.5**B**](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-3.5B) — esquema, dimensiones, demos (la placa de referencia)
- [Wiki Waveshare ESP32-S3-Touch-LCD-3.5](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-3.5) — lo mismo, para la variante sin el B
- [Datasheet ESP32-S3](https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf) (Espressif)
- [Technical Reference Manual ESP32-S3](https://www.espressif.com/sites/default/files/documentation/esp32-s3_technical_reference_manual_en.pdf) (Espressif)
- [Datasheet y guía de usuario ES8311](https://www.everest-semi.com/) (Everest Semiconductor)
- [Manual de usuario PN532](https://www.nxp.com/docs/en/user-guide/141520.pdf) (NXP)

## Lista de materiales

Los enlaces son enlaces afiliados del proyecto — comprar a través de ellos no te cuesta nada extra y ayuda a mantener la nube TigerTag gratuita.

### Electrónica

| Cantidad | Componente | Enlace |
|---------|-----------|--------|
| 1 | **Waveshare ESP32-S3-Touch-LCD-3.5B** — 480×320 IPS touchscreen, ESP32-S3, 8 MB PSRAM, 16 MB flash. La variante **-3.5** sin el B también es compatible, gracias a su propio firmware — ver [Variantes de placa](#variantes-de-placa) | [-3.5B](https://link.amazon/B0gaANfF5) · [-3.5](https://link.amazon/B0dpgOlOQ) |
| 2 | **Módulo NFC PN532 V3** — debe tener el conector de pines **y** el interruptor de modo HSU/I²C/SPI | [Amazon](https://link.amazon/B0iTXrhjd) |
| 1 | **Celda de carga de 5 kg + amplificador HX711** | [Amazon](https://link.amazon/B09LOUuI1) |
| 1 | **Cable USB-C de 4 pines** | [Amazon](https://link.amazon/B0aoW8qQx) |
| 1 | **Conector USB-C / breakout** | [Amazon](https://link.amazon/B0aiEyjLx) |
| 1 | Altavoz pequeño para el conector SPK de la placa — suministrado con la placa, produce el bip al detectar etiqueta | — |
| 1 | Batería Li-ion — opcional; la carga y el nivel se manejan mediante el PMIC AXP2101 integrado | [Amazon](https://link.amazon/B0etKlE1i) |
| — | **Cables Dupont** | [Amazon](https://link.amazon/B0bl6jvMs) |

<table align="center">
  <tr>
    <td align="center">
      <img src="img/esp32-s3-touch-lcd-3.5b.jpg" width="380" alt="Placa Waveshare ESP32-S3-Touch-LCD-3.5B"><br>
      <sub><strong>Todas las variantes funcionan, pero necesitan firmware diferente.</strong> Lee la silkscreen: <strong>-3.5B</strong> o <strong>-3.5</strong>. El instalador web pregunta cuál tienes; la conexión y el gabinete son los mismos en cualquier caso.</sub>
    </td>
    <td align="center">
      <img src="img/load-cell-hx711.jpg" width="380" alt="Celda de carga de 5 kg y placa HX711"><br>
      <sub><strong>Advertencia:</strong> la celda de carga debe tener 2× agujeros M4 y 2× agujeros M5, y la placa HX711 debe ser idéntica a la mostrada — de lo contrario no encajará en su ranura designada.</sub>
    </td>
  </tr>
</table>

## Variantes de placa

Waveshare vende esta placa de visualización en dos versiones y TigerScale funciona en ambas, cada una desde su propio firmware. Son del mismo tamaño y forma, se montan de la misma manera y requieren exactamente la misma conexión — comparando los dos esquemas oficiales netos, **un GPIO difiere**.

| | ESP32-S3-Touch-LCD-3.5**B** | ESP32-S3-Touch-LCD-3.5 |
|---|---|---|
| Controlador de visualización | AXS15231B sobre QSPI | ST7796 sobre SPI |
| Controlador de tacto | AXS5106L @ 0x3B en `Wire1` | FT6336 @ 0x38 en `Wire1` |
| GPIO12 | `LCD_CS` | `I2S_MCLK` — la selección de chip de la pantalla está conectada a GND a través de R16 en su lugar |
| MCLK ES8311 | GPIO44, a través de R61 | GPIO12 |
| Pines PN532, pines HX711, I²S, I²C, numeración del conector de 2.54 mm | — | idénticos |
| Ambiente de construcción | `esp32s3_hsu_b` | `esp32s3_hsu` |

Todo lo que sigue se aplica por tanto a ambas placas sin cambios, y así es el gabinete impreso. Lo que **no** se traslada es la imagen del firmware: con un controlador de panel diferente, una placa quemada con la construcción de la otra variante muestra una pantalla negra y no hay forma de decir por qué. El instalador web pregunta cuál placa tienes antes de quemar algo, y el canal de aire en vivo sirve cada placa desde su propia entrada en el manifiesto publicado.

Dos trampas viven en esa diferencia, ambas presentándose como una placa que quema perfectamente y muestra nada:

- **El ST7796 quiere el modo SPI 0.** `Arduino_ST7796::begin()` en GFX 1.6.x fuerza
  `SPI_MODE3` en ESP32, mientras que el ejemplo propio de Waveshare usa modo 0. Con modo 3
  la luz de fondo se enciende, `[LCD] begin OK` se imprime y la pantalla sigue negra.
  El firmware lo sobrescribe en una subclase de dos líneas en la Sección 5.
- **El ST7796 necesita su pulso de reinicio**, y `LCD_RST` no es un GPIO en ninguna
  placa — está en el expander TCA9554, en `Wire1`. Ver la sección I²C a continuación.

> [!NOTA]
> La **variante -3.5B** es la que se ha verificado en hardware real. La **variante -3.5** es
> derivada del esquema oficial más las dos correcciones anteriores, ambas reportadas desde
> la primera -3.5 en el campo, y aún no se ha confirmado por completo. Un informe en cualquier caso es bienvenido.

> [!ADVERTENCIA]
> **Los dongles PN532 solo USB-C sellados no funcionarán.** No tienen conector de pines, y
> esta placa no puede actuar como host USB — ver
> [USB_HOST_POSTMORTEM.md](USB_HOST_POSTMORTEM_ES.md). Compra los módulos con un conector.

### Fijaciones

| Cantidad | Tamaño | Propósito | Enlace |
|---------|-------|----------|--------|
| — | **M3×10 o M3×12** auto-perforantes | Gabinete — el conjunto lo cubre | [Amazon](https://link.amazon/B0ekzxx1E) |
| 2 | **M4×30** | Celda de carga | — |
| 2 | **M5×30** | Celda de carga | — |
| 4 | **M2×6** | Placa de visualización | — |

### Piezas impresas

Aún no publicadas para V3. Ver [MAKERWORLD.md](MAKERWORLD_ES.md) para la lista que se está preparando. El [repositorio V2](https://github.com/TigerTag-Project/Tiger-Scale)
tiene una construcción completa y costada de la generación anterior si quieres un referente.

### Diagrama de conexión

<p align="center">
  <a href="https://app.cirkitdesigner.com/project/c6aa6c0a-9462-498f-8923-9ad4454e0e69">
    <img src="img/wiring-hsu.jpg" alt="Conexión TigerScale V3: ESP32-S3-Touch-LCD-3.5B, dos lectores PN532 sobre HSU, HX711 y celda de carga, altavoz" width="820">
  </a>
</p>

<p align="center">
  <sub>La construcción de HSU de referencia. Haz clic para ver la versión interactiva y zoomable
  en <a href="https://app.cirkitdesigner.com/project/c6aa6c0a-9462-498f-8923-9ad4454e0e69">Cirkit Designer</a>.</sub>
</p>

**Lee cuidadosamente las etiquetas de los pines PN532.** El diagrama muestra cada lector's 4-pin
header etiquetado `SCL` / `SDA` / `VCC` / `GND`, porque eso es la silkscreen en el módulo. En **modo HSU esos mismos pines físicos llevan TXD y RXD** — esto es una construcción UART, no I²C. Alguien que lea las etiquetas literalmente conectará correctamente pero luego construirá `esp32s3_i2c`, y no encontrará ningún lector. Construye `esp32s3_hsu_b` (o `esp32s3_hsu` en una placa sin el B).

Dos cosas que este diagrama resuelve:

- **La placa es la variante `-3.5B`** — legible en la silkscreen en la renderización.
  El firmware's comentarios sobre los pines de audio se referían a ese esquema sin confirmación
  hasta ahora.
- **RSTPDN no está conectado en ninguno de los lectores.** Cuatro cables por módulo, y los
  pines `RSTO` y `IRQ` están flotantes. El firmware aún conduce GPIO17 y GPIO9 como reinicios, sin daño. Así que la nota abajo sobre RSTPDN ser opcional no es una observación única — es cómo la unidad de referencia está realmente construida.

Las tablas de pines más abajo siguen siendo la referencia autoritaria: fueron leídas
directamente del firmware (§1 `CONFIGURACIÓN DE HARDWARE`), que es lo que el código
realmente conduce. El diagrama muestra cómo está físicamente conectada la unidad de referencia; si
las dos nunca coinciden, el firmware gana y el diagrama necesita ser re-dibujado.

El gabinete impresible está en
[MakerWorld](https://makerworld.com/en/models/3161869-tigerscale-v3-best-smart-filament-scale-with-nfc#profileId-3573543)
como un proyecto Bambu Studio 3MF. Los precios y una lista completa de piezas aún están por venir. Ver el
[repositorio V2](https://github.com/TigerTag-Project/Tiger-Scale) para una construcción completa,
costada de la generación anterior.

## Topología I²C — lee esto antes de agregar cualquier periférico

Esta placa tiene dos buses I²C en el firmware, y **solo uno de ellos funciona**.

| Bus | Pines | Estado |
|-----|------|--------|
| `Wire1` | SDA = GPIO8, SCL = GPIO7 | Funcionando. Usa este. |
| `Wire` | SDA = GPIO21, SCL = GPIO22 | **Dañado. No usar.** |

`Wire` no puede funcionar: **GPIO22 no existe en el ESP32-S3** (la chip salta GPIO22–25), y el GPIO21 está asignado al conector de la cámara (CAM_D7) en esta placa en lugar de al I²C. Cada encendido registra `perimanSetPinBus(): Pin inválido: 22` y el escaneo de ese bus encuentra cero dispositivos. Las definiciones de pines heredan el diseño V2 y permanecen en su lugar, pero nada nuevo debería estar conectado allí.

Tenga en cuenta que el expansor I/O TCA9554 que el firmware intenta alcanzar en este bus es real y responde — solo en `Wire1`, no aquí. Vea la tabla medida a continuación.

Dispositivos en el bus `Wire1` funcional. Esta tabla es un **escaneo medido en el arranque** de una unidad real, no una transcripción del demo del proveedor:

| Dirección | Dispositivo | Notas |
|----------|------------|------|
| 0x18 | Codificador de audio ES8311 | Sonido al detectar etiqueta |
| 0x20 | Expansor I/O TCA9554 | **En `Wire1`, no en `Wire`.** Vea abajo. |
| 0x34 | AXP2101 PMIC | Nivel de batería, estado de carga |
| 0x38 | Controlador táctil FT6336 | Solo -3.5 |
| 0x3B | Controlador táctil AXS5106L | Solo -3.5B |
| 0x51 | Reloj real PCF85063 | Baterizado a través del AXP2101. Nada en el firmware habla con él (#21). |
| 0x6B | IMU QMI8658 6-ejes | Nada en el firmware habla con él. |
| 0x24 | PN532 | Solo en I²C — dirección fija, no se puede cambiar |

Tres cosas importantes sobre esa lista:

- **El TCA9554 responde en `Wire1`, y la línea de reinicio de la pantalla se conecta a través de él** (EXIO1 = P1 = `LCD_RST`, en ambas placas). `lcdResetByTCA9554()` lo controlaba mediante `Wire` durante mucho tiempo — el bus que no funciona —, por lo que la secuencia de reinicio nunca se ejecutó realmente. Esto es inofensivo en la -3.5B, cuya AXS15231B se inicia independientemente, y no inofensivo en la -3.5, cuya ST7796 necesita el pulso. El reinicio ahora corre en `Wire1` **solo para la -3.5**: la -3.5B ha estado arrancando sin ello durante meses, y dar a toda la flota un nuevo paso de inicialización para corregir una placa que no está ejecutando sería el mal intercambio.
- **0x51 y 0x6B son componentes integrados que el firmware no usa**, nombrados por las redes `RTC_*` y `IMU_*` de los esquemas y la documentación del proveedor. El PCF85063 significa que la báscula tiene un reloj de batería que nunca lee: depende de las marcas de tiempo del servidor Firestore en lugar de NTP y anota "Sin NTP — aproximado basado en…" en el código (#21).
- El escaneo corre al arranque en cada unidad, por lo que también sirve como comprobación de salud de conexión para alguien que ensambló su propia báscula. Compare su registro de arranque contra esta tabla.

## Periféricos fijos

Estos están en la placa o conectados de la misma manera independientemente del transporte NFC.

| Función | Pines |
|--------|------|
| Display QSPI | CS=12, CLK=5, D0=1, D1=2, D2=3, D3=4 |
| Backlight del display | GPIO6 |
| Tacto / PMIC / codificador | `Wire1`: SDA=8, SCL=7 |
| Celda de carga HX711 | DOUT=39, SCK=40 |
| ES8311 I²S | MCLK=44, BCLK=13, WS=15, DOUT=16 |
| Servo (no usado en esta compilación) | GPIO11 |

No hay **motor** en esta báscula: `setupServo()` es intencionalmente un no-op. El pin y la tabla de niveles de velocidad permanecen para la variante con motor.

## Conexión NFC — elija uno, luego construya el entorno correspondiente

El transporte es una **elección en tiempo de compilación**. Construir un entorno que no coincida con su conexión produce un firmware que detecta ningún lector y no da ninguna pista de por qué. Esto ha costado al menos una sesión de depuración completa.

### HSU / UART — `esp32s3_hsu_b` / `esp32s3_hsu` (referencia, verificado en bench)

Establezca **ambos canales del interruptor de modo del módulo en OFF** (HSU, según la tabla de silkscreen de Elechouse V3). Cada módulo tiene su propio UART; no hay bus compartido.

| Señal | Lector derecho (`rfid1`, Serial1) | Leitor izquierdo (`rfid2`, Serial2) |
|------|--------------------------------|-------------------------------|
| TXD de PN532 → RX de ESP32 | GPIO47 | GPIO41 |
| RXD de PN532 ← TX de ESP32 | GPIO48 | GPIO18 |
| RSTPDN (`RSTO` en la silkscreen) | GPIO17 | GPIO9 |
| VCC | 3.3 V o 5 V (el módulo tiene un convertidor de nivel) | mismo |
| GND | común con el ESP32 | mismo |

**RSTPDN resultó no ser estrictamente necesario** — ambos lectores fueron confirmados leyendo etiquetas con solo `rfid1`'s RSTPDN realmente conectado. Conéctelo de todos modos si es conveniente: un pin de reinicio flotante tiene un estado indefinido al encenderse, por lo que depender de ello es depender de la suerte.

Una conclusión anterior que RSTPDN *debe* ser controlado vino de pruebas en un proyecto separado minimalista que nunca tuvo el conflicto de pines `SPI.begin()` descrito en el postmortem. Dos fallos, un síntoma.

### SPI — `esp32s3` (compila, no verificado en bench)

Interruptor de modo OFF/ON. Ambos lectores comparten un bus SPI con selects individuales.

| Señal | Pin |
|------|-----|
| SCK | GPIO47 |
| MISO | GPIO48 |
| MOSI | GPIO41 |
| SS — derecho (`rfid1`) | GPIO42 |
| SS — izquierdo (`rfid2`) | GPIO18 |
| RSTPDN — derecho/izquierdo | GPIO17 / GPIO9 |

Mantener RSTPDN de un lector a nivel bajo silencia completamente a ese lector en el bus compartido, lo cual es el mismo truco que usó la conexión MFRC522 de V2.

### I²C — `esp32s3_i2c` (compila, no verificado en bench)

Interruptor de modo: canal 1 = ON, canal 2 = OFF. **Solo un lector.**

| Señal | Pin |
|------|-----|
| SDA | GPIO8 (`Wire1`) |
| SCL | GPIO7 (`Wire1`) |
| RSTPDN | GPIO17 |
| IRQ | no conectado — la biblioteca consulta |

La dirección I²C del PN532 está fija en 0x24, por lo que un segundo módulo en el mismo bus colisiona. En esta compilación `rfid2` es un stub permanente y la caída de retroceso de un solo lector del firmware se aplica sin cambios. Un segundo lector real I²C necesitaría un multiplexor I²C como el TCA9548A, el cual no está implementado.

## Ubicación del lector y interferencia RF

Los dos antenas están aproximadamente **75 mm apartados, mirándose mutuamente**, lo cual es cercano suficiente para que el campo de un lector sea visto por el otro. El escaneo round-robin solo no resolvió esto.

La solución es una tabla de 16 niveles de potencia/sensibilidad RF (`applyPN532RfTuning`), que establece la potencia de transmisión (`GsNOn`/`CWGsP`/`ModGsP`) y la sensibilidad del receptor (`RxThreshold` MinLevel) juntos. Los niveles abarcan el rango completo, el índice 15 siendo los valores de reinicio del datasheet; los puntos de operación útiles están en la mitad inferior, porque el extremo superior sobrecarga el módulo.

El nivel es ajustable en tiempo de ejecución desde **Configuración → RFID** con un controlador `−`/`+`, entra en vigor en el escaneo inmediatamente siguiente y persiste en NVS. El valor predeterminado es nivel 7. El botón **Auto** de la misma pantalla mide el mejor nivel para la unidad frente a ella; [`RFID-PERFORMANCE.md`](RFID-PERFORMANCE_ES.md) tiene los datos de bench que el valor predeterminado proviene.

Si está depurando un lector que no encuentra nada, intente el controlador antes de reconfigurar nada.

## Diseño de flash

Desde `partitions.csv`:

| Partición | Desplazamiento | Tamaño |
|----------|---------------|--------|
| nvs | 0x9000 | 0x5000 |
| otadata | 0xE000 | 0x2000 |
| app0 | 0x10000 | 0x640000 |
| spiffs (LittleFS, interfaz web) | 0x650000 | 0x1A0000 |
| coredump | 0x7F0000 | 0x10000 |

**NVS almacena las credenciales WiFi guardadas y la sesión Firebase.** Un refleche normal lo preserva; el flash `firmware.factory.bin` como un solo blob en 0x0000 no lo hace. Vea [INSTALLATION.md](INSTALLATION_ES.md#why-your-wifi-survives-a-reflash).

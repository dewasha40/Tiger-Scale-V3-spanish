# Instalación

Compilar y cargar el firmware de TigerScale V3.

## Requisitos

- **[PlatformIO Core](https://docs.platformio.org/en/latest/core/installation/)** —
  `pip install platformio`, o la extensión PlatformIO IDE para VS Code.
- Un cable USB-C y la placa.
- Python 3 (solo para `scripts/watch_logs.py`).

**Arduino IDE no puede compilar este proyecto.** La configuración de LVGL se
encuentra mediante `-I include` junto con `LV_CONF_INCLUDE_SIMPLE`, y el IDE no
proporciona una ruta de inclusión equivalente. PlatformIO es la única cadena de
herramientas compatible.

## ¿Qué compilación necesito?

Hay dos aspectos que se fijan **al compilar**: el modelo de placa de pantalla y
el cableado de los módulos PN532. Las dos compilaciones publicadas suponen el
cableado HSU/UART, que es el descrito en la guía de montaje y el utilizado en
las básculas instaladas.

| Tu placa | Entorno de compilación | Publicada |
|------------|-----------|-----------|
| ESP32-S3-Touch-LCD-3.5**B** | `esp32s3_hsu_b` | sí — predeterminado en el instalador web |
| ESP32-S3-Touch-LCD-3.5 (sin B) | `esp32s3_hsu` | sí |

Lee la serigrafía blanca de la placa, justo debajo del conector de cinta: el
modelo termina en B o no. Ambas placas tienen el mismo tamaño y usan el mismo
cableado, pero sus controladores de pantalla son distintos (AXS15231B por QSPI
en el modelo B y ST7796 por SPI en el otro). Si cargas la imagen equivocada,
la placa se programa correctamente, pero la pantalla no se enciende.

Los otros dos transportes ya no se publican: todas las básculas instaladas usan
HSU y ofrecer una opción cuyo fallo es silencioso («sin lectores», sin explicar
el motivo) causaba más problemas de los que resolvía. Aun así, se pueden compilar
desde el código fuente:

| Tu cableado | Entorno de compilación | Lectores |
|-------------|-----------|---------|
| SPI | `esp32s3` | 2 — compila, pero no se ha verificado en banco |
| I²C en `Wire1` | `esp32s3_i2c` | 1 — la dirección del PN532 es fija |

Consulta los esquemas de cableado en [HARDWARE.md](HARDWARE.md#nfc-wiring--pick-one-then-build-the-matching-env).

## Primera instalación en una placa nueva

### Opción más sencilla: instalador web

<https://tigertag-project.github.io/Tiger-Scale-V3/>

Conecta la placa, elige el cableado NFC y pulsa **Instalar**. El instalador carga
de una vez el bootloader, la tabla de particiones, el firmware y la interfaz web.
En un ordenador, usa Chrome, Edge u Opera: Web Serial no está disponible en
Safari, Firefox ni en dispositivos móviles.

Ten en cuenta que **el instalador borra la placa**, incluida la configuración
Wi-Fi guardada. Esto es lo correcto para una placa nueva; para actualizar una
báscula en uso, consulta [Actualizar más adelante](#actualizar-más-adelante).

### Desde el código fuente

**Un solo comando.** En una placa que nunca se haya programado, esto completa la
instalación:

```bash
git clone https://github.com/TigerTag-Project/Tiger-Scale-V3.git
cd Tiger-Scale-V3

bash scripts/flash.sh --fs
```

El argumento `--fs` es necesario en la primera instalación y es fácil omitirlo.
En la memoria flash hay dos componentes separados y se escriben en pasos
independientes:

| Componente | Ubicación | Se escribe con |
|---|---|---|
| Firmware (la aplicación) | `app0` @ 0x10000 | cada carga |
| Interfaz web (`data/`) | `spiffs` @ 0x650000 | **solo** con `--fs` / `-t uploadfs` |

Si cargas el firmware sin `--fs`, la báscula funciona, pero la interfaz web
responde con un error 404. Puede parecer un fallo del firmware, pero no lo es.
Si ocurre, vuelve a ejecutar `bash scripts/flash.sh --fs`.

Añade `--monitor` si quieres observar el arranque:

```bash
bash scripts/flash.sh --fs --monitor
```

### Si no se detecta la placa

El ESP32-S3 expone un puerto USB-Serial/JTAG nativo y esptool lo reinicia
automáticamente. En la unidad de referencia no hizo falta pulsar ningún botón.
Si el puerto no aparece:

- Espera unos segundos. La enumeración puede tardar; un escaneo justo después
de conectar la placa quizá no muestre nada y sí funcione al repetirlo.
- Comprueba que el cable transmite datos y no solo alimentación.
- Fuerza el modo de descarga: mantén pulsado **BOOT**, pulsa y suelta **RESET**,
y después suelta **BOOT** antes de cargar el firmware.
- Indica el puerto explícitamente: `bash scripts/flash.sh --fs --port /dev/cu.usbmodem1101`
(`pio device list` muestra los puertos disponibles).

Una placa que responde correctamente muestra algo parecido a esto:

```
Chip type:  ESP32-S3 (QFN56) (revision v0.2)
Features:   Wi-Fi, BT 5 (LE), Dual Core + LP Core, 240MHz, Embedded PSRAM 8MB
Detected flash size: 16MB
```

### ¿Ha funcionado?

El registro de arranque debería mostrar estos mensajes, en este orden:

```
[LCD] begin OK w=480 h=320
[TOUCH] Wire1 SDA=8 SCL=7 addr=0x3B probe=0 (ACK)
[I2C] Scan done. 6 device(s) found.
[RFID] PN532-1 ready
[RFID] PN532-2 ready
[LITTLEFS] Mounted OK
[DB] Brand DB loaded: 120 entries
```

Comprueba que aparezcan `PN532-1 ready` y `PN532-2 ready`. Si no detecta ningún
lector, casi seguro que compilaste el entorno de transporte incorrecto para tu
cableado. Consulta la tabla anterior y [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

Hay dos mensajes de arranque que parecen alarmantes, pero son esperables en esta
placa: `perimanSetPinBus(): Invalid pin: 22` y algunas líneas `NULL TX buffer
pointer`. Proceden de un bus I²C del fabricante que no puede funcionar en el
ESP32-S3. [HARDWARE.md](HARDWARE.md#ic-topology--read-this-before-adding-any-peripheral)
explica el motivo.

## Actualizar más adelante

### Por el aire — solo firmware

En la pantalla táctil, abre **Ajustes → Actualización** para comprobar el
manifiesto publicado e instalar una versión más reciente con una barra de
progreso. No necesitas conectar un cable.

> [!WARNING]
> **La actualización de firmware por el aire no funciona con la tabla de
> particiones actual.** `app0` es una única partición `factory` y no tiene una
> ranura `ota_0`/`ota_1` adicional donde escribir la nueva imagen. El mecanismo
> OTA de ESP-IDF no tiene dónde guardar la imagen y falla con `Update.begin
> failed (need=… free=0)`. El actualizador detecta este caso y muestra «Actualizar
> por USB» en vez de empezar. La actualización de la **interfaz web** por el aire
> sí funciona. Hasta que la tabla de particiones incluya una segunda ranura de
> aplicación, actualiza el firmware por USB.

Hay otros dos límites que debes conocer:

- **OTA solo sustituye el firmware, no la interfaz web.** El manifiesto publicado
en GitHub Pages contiene `firmware_url` y nada más. Si una versión cambia
archivos de `data/`, también tendrás que cargar ese sistema de archivos por USB
con `--fs`. (El firmware sí admite `littlefs_url` mediante su endpoint local
`POST /api/ota/update` y la cola remota de comandos, de modo que una aplicación
puede enviar una actualización del sistema de archivos; el flujo automático de
**Ajustes → Actualización** no puede hacerlo.)
- **Las compilaciones publicadas suponen cableado HSU**: hay una por modelo de
placa y ninguna para SPI o I²C. Si tu unidad usa uno de esos transportes e instala
una actualización publicada, volverá a la compilación HSU y sus lectores dejarán
de funcionar hasta que vuelvas a cargar el firmware correcto por USB.

También tiene que existir una versión publicada. Hasta que se etiquete la
primera versión, **Ajustes → Actualización** indicará que no puede acceder al
manifiesto.

### Por USB — para todo

```bash
bash scripts/flash.sh              # solo firmware; conserva los ajustes
bash scripts/flash.sh --fs         # firmware e interfaz web
```

En ambos casos se conservan las credenciales Wi-Fi, la sesión de TigerTag y la
calibración. Consulta la explicación siguiente.

## Por qué se conservan los ajustes al volver a cargar el firmware

PlatformIO escribe cada imagen en su propio desplazamiento:

```
 0x0000  bootloader.bin
 0x8000  partitions.bin
 0xE000  boot_app0.bin
0x10000  firmware.bin
```

NVS ocupa las direcciones 0x9000–0xE000 y no está dentro de ninguno de esos
rangos. Por tanto, se conservan las credenciales Wi-Fi, la sesión de TigerTag y
el factor de calibración. Esto se ha comprobado, no es una suposición: se leyó la
NVS de la unidad de referencia antes y después de volver a cargar el firmware,
y el SSID, el factor de tara y los ajustes seguían guardados.

**La excepción:** cargar `.pio/build/<env>/firmware.factory.bin` como un único
bloque en `0x0000`. Ese archivo es una imagen combinada que ocupa continuamente
la memoria desde 0x0000 y sí sobrescribe NVS. Por eso, durante el desarrollo,
las credenciales desaparecían al volver a cargar el firmware. Ni
`scripts/flash.sh` ni `pio run -t upload` hacen esto.

Para empezar desde cero, ejecuta `bash scripts/flash.sh --erase`. El comando
borra todo, incluida la calibración, después de pedir confirmación.

El modo de flash está fijado en **DIO** en la configuración de la placa. QIO
provoca un ciclo de errores de arranque en `ets_loader.c` en esta placa; no lo
cambies para «optimizar» el rendimiento.

## Ejecutar PlatformIO directamente

```bash
pio run -e esp32s3_hsu_b              # compilar
pio run -e esp32s3_hsu_b -t upload    # cargar firmware
pio run -e esp32s3_hsu_b -t uploadfs  # cargar interfaz web (LittleFS)
pio device monitor                   # consola serie
```

## Primer arranque

1. Aparecerá la pantalla de inicio de TigerTag y, después, la pantalla de
configuración Wi-Fi.
2. Elige tu red en la pantalla táctil e introduce la contraseña.
3. Inicia sesión en tu cuenta de TigerTag desde **Ajustes → Nube**.
4. Calibra la báscula en **Ajustes → Calibración** y sigue el asistente usando
una masa de referencia conocida.
5. Coloca una bobina con etiqueta TigerTag. La báscula la leerá, la pesará y
sincronizará la información.

La interfaz web del dispositivo está disponible en `http://<ip-del-dispositivo>/`.
La dirección aparece en **Ajustes → Wi-Fi**. También puedes usar
`http://tigerscale-XXXX.local/`; los cuatro dígitos hexadecimales aparecen en la
misma pantalla y se resuelven mediante mDNS.

## Resolución de problemas

Empieza por [TROUBLESHOOTING.md](TROUBLESHOOTING.md). La causa más frecuente de
«no se detectan lectores» es haber compilado el entorno de transporte incorrecto.

# Instalación

Compilación y programación de la firmware de TigerScale V3.

## Requisitos

- **[PlatformIO Core](https://docs.platformio.org/en/latest/core/installation/)** —
  `pip install platformio`, o la extensión de PlatformIO para VS Code
- Un cable USB-C y la placa
- Python 3 (solo para `scripts/watch_logs.py`)

**El IDE Arduino no puede compilar este proyecto.** La configuración de LVGL se encuentra
a través de `-I include` combinado con `LV_CONF_INCLUDE_SIMPLE`, y el IDE no proporciona
una ruta de inclusión equivalente. PlatformIO es el único entorno de herramientas admitido.

## ¿Qué compilación necesito?

Dos cosas están fijadas en **tiempo de compilación**: qué placa de visualización tienes y cómo
están conectados tus módulos PN532. Ambas compilaciones publicadas asumen la conexión HSU / UART,
que es lo que describe la guía de compilación y lo que todas las básculas en uso están construidas.

| Tu placa | Entorno de compilación | Publicado |
|----------|------------------------|-----------|
| ESP32-S3-Touch-LCD-3.5**B** | `esp32s3_hsu_b` | sí — el predeterminado en el instalador web |
| ESP32-S3-Touch-LCD-3.5 (sin B) | `esp32s3_hsu` | sí |

Lee la escritura blanca en tu placa, justo debajo de la conexión de cinta: termina
con una B o no. Las dos son del mismo tamaño y usan la misma conexión, pero sus controladores de pantalla difieren (AXS15231B sobre QSPI en la B, ST7796 sobre SPI en la otra), por lo que la imagen incorrecta te deja con una placa que se programa correctamente pero nunca enciende su pantalla.

Las otras dos transportes no se publican más — todas las básculas en el campo están conectadas HSU, y ofrecer una elección cuyo modo de fallo es silencioso ("sin lectores", sin razón dada) cuesta más de lo que ayuda. Aún así, se pueden compilar desde la fuente:

| Tu conexión | Entorno de compilación | Lectores |
|-------------|------------------------|----------|
| SPI | `esp32s3` | 2 — se compila, no se ha verificado en pruebas |
| I²C en `Wire1` | `esp32s3_i2c` | 1 — la dirección del PN532 es fija |

Diagramas de conexión para cada uno: [HARDWARE.md](HARDWARE_ES.md#nfc-wiring--pick-one-then-build-the-matching-env).

## Instalación inicial en una placa nueva

### Más fácil: instalador web

<https://tigertag-project.github.io/Tiger-Scale-V3/>

Conecta la placa, elige tu conexión NFC, haz clic en Instalar. Escribirá el bootloader,
tabla de particiones, firmware y interfaz web en un solo paso. Chrome, Edge o Opera en escritorio —
Web Serial no está disponible en Safari, Firefox o en móvil.

Nota que **borra la placa**, incluyendo la WiFi guardada. Eso es correcto para hardware nuevo; para
actualizar una báscula ya en uso, ve a [Updating later](#updating-later).

### Desde la fuente

**Un comando.** En una placa que nunca haya sido programada, esto es el trabajo completo:

```bash
git clone https://github.com/TigerTag-Project/Tiger-Scale-V3.git
cd Tiger-Scale-V3

bash scripts/flash.sh --fs
```

`--fs` importa en una instalación inicial y es fácil de omitir. Dos cosas distintas viven en
flash y se escriben en pasos separados:

| Lo que | Donde | Escrito por |
|---|---|---|
| Firmware (la aplicación) | `app0` @ 0x10000 | cada flash |
| Interfaz web (`data/`) | `spiffs` @ 0x650000 | **solo** con `--fs` / `-t uploadfs` |

Programar sin `--fs` y la báscula funciona, pero su interfaz web responde 404 —
lo que parece un error y no lo es. Si eso ocurre, simplemente ejecuta
`bash scripts/flash.sh --fs` nuevamente.

Agrega `--monitor` si quieres ver el encendido:

```bash
bash scripts/flash.sh --fs --monitor
```

### Si la placa no se detecta

La ESP32-S3 expone un puerto USB-Serial/JTAG nativo, y esptool lo reinicia por ti —
en la unidad de referencia no se necesitaba presionar el botón. Si tu puerto nunca aparece:

- Dales unos segundos. La enumeración puede retrasarse; un escaneo de puerto inmediatamente después
  de enchufar puede mostrar nada, pero puede tener éxito en un reintento.
- Verifica que el cable lleve datos, no solo energía.
- Modo de descarga forzado: mantén **BOOT**, toca **RESET**, suelta **BOOT**, luego programa.
- Pasar el puerto explícitamente: `bash scripts/flash.sh --fs --port /dev/cu.usbmodem1101`
  (`pio device list` mostrará candidatos).

Una placa que habla correctamente responde así:

```
Chip type:  ESP32-S3 (QFN56) (revision v0.2)
Features:   Wi-Fi, BT 5 (LE), Dual Core + LP Core, 240MHz, Embedded PSRAM 8MB
Detected flash size: 16MB
```

### ¿Funcionó?

El registro de encendido debe mostrar, en este orden:

```
[LCD] begin OK w=480 h=320
[TOUCH] Wire1 SDA=8 SCL=7 addr=0x3B probe=0 (ACK)
[I2C] Scan done. 6 device(s) found.
[RFID] PN532-1 ready
[RFID] PN532-2 ready
[LITTLEFS] Mounted OK
[DB] Brand DB loaded: 120 entries
```

`PN532-1 listo` / `PN532-2 listo` es lo que debes verificar. Si no ves lectores, casi seguramente construiste el entorno de transporte incorrecto para tu conexión — ve a la tabla
arriba, y [TROUBLESHOOTING.md](TROUBLESHOOTING_ES.md).

Dos mensajes de encendido parecen alarmantes y son esperados en esta placa:
`perimanSetPinBus(): Pin inválido: 22` y algunas líneas `NULL TX buffer pointer`.
Proceden de un bus I²C de un proveedor que no puede funcionar en la ESP32-S3.
[HARDWARE.md](HARDWARE_ES.md#ic-topology--read-this-before-adding-any-peripheral)
explica por qué.

## Actualización posterior

### Por aire — para firmware

**Configuración → Actualizar** en la pantalla táctil verifica el manifiesto publicado e instala
un firmware más reciente con una barra de progreso. No se necesita enchufar nada.

> [!WARNING]
> **La actualización por aire del firmware no funciona en la tabla de particiones actual.**
> `app0` es una sola partición `factory` sin ranura `ota_0`/`ota_1` a su lado, por lo que
> la maquinaria OTA de ESP-IDF no tiene lugar para escribir la nueva imagen y la instalación falla
> con `Update.begin failed (need=… free=0)`. El actualizador verifica esto y muestra
> "Actualización por USB" en lugar de iniciar. La actualización del **interfaz web** por aire
> sí funciona. Reflash el firmware por USB hasta que la tabla de particiones gane una segunda ranura de app.

Dos límites adicionales que vale la pena conocer:

- **OTA reemplaza solo el firmware — no la interfaz web.** El manifiesto publicado en GitHub Pages lleva un `firmware_url` y nada más, por lo que una versión que cambie algo bajo `data/` aún necesita una carga USB `--fs`.
- **Las compilaciones publicadas asumen la conexión HSU** — una por placa, y nada para SPI
  o I²C. Una unidad conectada de alguna manera que tome una actualización publicada regresa en la compilación HSU y pierde sus lectores hasta que se reflashe por USB.

OTA también necesita que exista una versión publicada. Hasta que la primera versión se etiquete,
"Configuración → Actualizar" simplemente reportará que no puede alcanzar el manifiesto.

### Por USB — para todo

```bash
bash scripts/flash.sh              # firmware only, keeps your settings
bash scripts/flash.sh --fs         # firmware + web UI
```

Tus credenciales WiFi, sesión TigerTag y calibración sobreviven ambos — ve abajo.

## ¿Por qué tus ajustes sobreviven un reflash

PlatformIO escribe cada imagen en su propio desplazamiento:

```
 0x0000  bootloader.bin
 0x8000  partitions.bin
 0xE000  boot_app0.bin
0x10000  firmware.bin
```

NVS vive en 0x9000–0xE000, fuera de esos rangos. Las credenciales WiFi, la sesión TigerTag y el factor de calibración sobreviven. Esto se verifica, no se asume: NVS se leyó de la unidad de referencia antes y después de un reflash, y las credenciales SSID, el factor de tara y los ajustes estaban todos allí.

**La trampa:** programar `.pio/build/<env>/firmware.factory.bin` en `0x0000` como un solo blob. Este archivo es una imagen *fusionada* que abarca 0x0000 hacia arriba continuamente, por lo que lo sobrescribe NVS — lo que es por qué las credenciales se perdían en cada reflash durante el desarrollo. Ni `scripts/flash.sh` ni `pio run -t upload` hacen esto.

Para un estado limpio real, `bash scripts/flash.sh --erase` borra todo incluyendo la calibración, después de pedir confirmación.

El modo de programación está fijado a **DIO** por la configuración de la placa. QIO produce un bucle de reinicio en `ets_loader.c` en esta placa — no "optimices"lo.

## Dirigir PlatformIO directamente

```bash
pio run -e esp32s3_hsu_b              # build
pio run -e esp32s3_hsu_b -t upload    # flash firmware
pio run -e esp32s3_hsu_b -t uploadfs  # flash the web UI (LittleFS)
pio device monitor                  # serial console
```

## Primer encendido

1. Aparece el splash de TigerTag, luego una pantalla de configuración WiFi.
2. Elige tu red en la pantalla táctil y entra la contraseña.
3. Inicia sesión en tu cuenta TigerTag desde **Configuración → Nube**.
4. Calibra la báscula: **Configuración → Calibración**, luego sigue el asistente con un peso de referencia conocido.
5. Coloca un rollo que lleve una etiqueta TigerTag. La báscula lo lee, lo pesa y sincroniza.

La interfaz web del dispositivo es accesible en `http://<dirección-ip-del-dispositivo>/` — la dirección se muestra bajo **Configuración → WiFi** — y en `http://tigerscale-XXXX.local/` (los cuatro dígitos hexadecimales se muestran en esa misma pantalla) donde mDNS resuelve.

## Solución de problemas

Comienza con [TROUBLESHOOTING.md](TROUBLESHOOTING_ES.md). La causa más común de "no se detectaron lectores" es haber construido el entorno de transporte incorrecto.

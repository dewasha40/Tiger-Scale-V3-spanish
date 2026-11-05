# Lista de MakerWorld — kit de pegado y copia

Todo lo necesario para publicar la carcasa de TigerScale V3 en
[MakerWorld](https://makerworld.com). Copia los bloques siguientes directamente en el
formulario de subida.

Una nota antes de comenzar. **Los ajustes de impresión son tuyos para completar** — están
marcados `<<TODO>>` a continuación, ya que inventar una altura de capa o un tiempo de impresión para un
modelo que no he cortado sería peor que dejar una brecha.

La lista está activa, y MakerWorld es ahora el único lugar donde se publica la carcasa: `README.md`, [HARDWARE.md](HARDWARE_ES.md), `TRADEMARK.md` y el instalador web todos apuntan allí. El modelo en sí es `TigerScale-V3-Case-1.0.1.3mf`, un proyecto de Bambu Studio con las placas ya dispuestas.

---

## Título

> TigerScale V3 — Escala de filamento inteligente con NFC (Pantalla táctil ESP32-S3)

MakerWorld corta alrededor de 60 caracteres en las tarjetas, por lo que las palabras importantes son primero. Alternativas si quieres A/B:

- `TigerScale V3 — La Escala de Filamento que Conoce tu Bobina`
- `TigerScale V3 — Escala de Filamento con NFC, Pantalla táctil de 3.5", Código abierto`

## Resumen / descripción corta

> Una escala conectada de código abierto que identifica tu bobina de filamento mediante NFC y
> te dice exactamente cuánto filamento queda. Coloca la bobina — lee la etiqueta, pesa la bobina, resta el peso vacío de la bobina y sincroniza el resultado.

## Descripción

Pega tal cual. MakerWorld soporta Markdown básico.

```markdown
## What it is

TigerScale V3 is a **connected filament scale** for 3D printing. Put a spool
carrying a TigerTag NFC tag on the platform and it does the rest: reads the tag,
weighs the spool, subtracts the empty spool's tare weight, and shows you **how much
filament is actually left**.

No typing in spool weights. No squinting at the reel trying to guess. No sticky
notes on the shelf. Put it down, read it, done.

The remaining weight syncs to your TigerTag account, so your phone, your desktop
and your other scales all agree on what you have in stock.

## Why NFC changes things

A normal scale tells you a number. It has no idea *which* spool is on it, so you
still have to remember whether that number includes a 250 g cardboard spool or a
180 g plastic one.

TigerScale reads the tag and already knows the brand, the material, the colour, the
diameter and the empty spool weight. That is the difference between "1043 g of
something" and "795 g of R3D PLA High Speed, gold".

Tags come from the **TigerTag open protocol**. Factory spools carry **TigerTag+**,
whose reserved bytes hold an origin signature that can be verified offline, on your
own phone, with no account — a cloned tag fails the check. Rosa3D ships it on 100 %
of its 1 kg production; R3D and eSun produce it at scale; Sunlu, Landu and JamgHE
tag on request. You can also tag your own spools with reusable chips.

## What you get on the screen

A 3.5" 480×320 colour touchscreen, driven by a proper LVGL interface — not a
two-line character display:

- Live weight, big and readable across the room
- Brand, material and colour of the spool currently on the platform
- WiFi setup with an on-screen keyboard — no serial console, no captive portal
- A guided calibration wizard
- A hardware self-test screen, including NFC reader power tuning
- Battery level and charge state
- Over-the-air firmware updates
- **9 languages** (EN · FR · DE · ES · IT · ZH · PT-BR · PT-PT · PL), kept in
  sync with your TigerTag account — change it in Studio, the scale follows
- A guided first-boot setup: language, WiFi, account, calibration — the scale
  walks you through all of it, iPhone-style

It also serves **its own web interface** on your network, so you can read the scale
from a phone browser without installing anything.

Works offline: brand and material identification comes from a database stored in
the device's own flash, so identifying a tag never waits on the internet. The cloud
account is optional.

## TigerTag Studio — the other half

[insert the TigerTag Studio hero image here]

The scale is the hardware end of a system. **TigerTag Studio** is the desktop
side: your whole filament inventory in one place, organised into racks and
positions, every weighing from the scale landing in it seconds after the spool
touches the platform. Weigh a spool on the scale, watch its remaining grams
update in Studio across the room. One account, one language, one inventory —
every device agrees.

Get it at https://tigersystem.io

## Electronics

This is a build, not a kit — you will need to solder and wire.

| Qty | Part |
|---|---|
| 1 | Waveshare ESP32-S3-Touch-LCD-3.5**B** — 480×320 IPS touchscreen, 8 MB PSRAM, 16 MB flash |
| 2 | PN532 **V3** NFC module — **with a pin header** and the HSU/I2C/SPI mode switch |
| 1 | 5 kg load cell + HX711 amplifier |
| 1 | USB-C 4-pin cable + USB-C connector |
| — | Dupont wires |
| — | M3x10 or M3x12 self-tapping screws (enclosure) |
| 2 | M4x30 screws (load cell) |
| 2 | M5x30 screws (load cell) |
| 4 | M2x6 screws (display board) |
| 1 | Small speaker — supplied with the board, beep on tag detect |
| 1 | Li-ion battery — optional; charging is handled on the board |

Every part with a purchase link:
https://github.com/TigerTag-Project/Tiger-Scale-V3/blob/main/docs/HARDWARE.md#bill-of-materials

Two readers, because a spool can carry its tag on either flange.

> **Important — get PN532 modules with a pin header.** The sealed USB-C-only dongles cannot
> work on this board — its USB-C port is wired as a device and can never act as a
> host. This is a hardware fact, not a firmware limitation, and it is documented in
> detail in the repository.

**Full wiring diagram, pinout and firmware:**
https://github.com/TigerTag-Project/Tiger-Scale-V3

**Flashing needs no toolchain at all**: plug the board into Chrome/Edge and the
web installer does the rest — https://tigertag-project.github.io/Tiger-Scale-V3/
The firmware is MIT licensed and also builds from source with PlatformIO.

## Print settings

<<TODO — fill from your own slice>>

| | |
|---|---|
| Material | <<TODO>> |
| Layer height | <<TODO>> |
| Walls / top / bottom | <<TODO>> |
| Infill | <<TODO>> |
| Supports | <<TODO>> |
| Print time | <<TODO>> |
| Filament used | <<TODO>> |

## Assembly

<<TODO — step by step, one photo per step is ideal>>

1. Print the parts
2. Install the heat-set inserts
3. Mount the load cell to the base and the platform to the load cell
4. Wire the HX711 to the load cell, then to the board
5. Set both PN532 mode switches to **HSU** (both channels OFF) and wire them
6. Seat the display board in the front housing
7. Flash the firmware, then calibrate with a known weight

Wiring is in the repository, with a full colour diagram:
https://github.com/TigerTag-Project/Tiger-Scale-V3/blob/main/docs/HARDWARE.md

## Firmware

https://github.com/TigerTag-Project/Tiger-Scale-V3

Easiest: the **web installer** — plug the board in, click Install, done:
https://tigertag-project.github.io/Tiger-Scale-V3/

Or build from source:

```bash
git clone https://github.com/TigerTag-Project/Tiger-Scale-V3.git
cd Tiger-Scale-V3
bash scripts/flash.sh --fs
```

After that the scale updates itself over the air. MIT licensed. Issues,
questions and improvements welcome.

## Community

Discord: https://discord.gg/3Qv5TSqnJH
TigerTag protocol: https://github.com/TigerTag-Project/TigerTag-RFID-Guide
```

---

## Etiquetas / palabras clave

MakerWorld permite un número limitado, por lo que estas están ordenadas por utilidad:

```
filament, spool, scale, nfc, rfid, esp32, esp32s3, tigertag, filament-management,
3d-printing-tools, smart, iot, touchscreen, open-source, weight, inventory,
lvgl, functional, electronics-enclosure
```

## Categoría

**Herramientas y Gadgets → Accesorios de impresora 3D** (o *Hobbie y DIY → Electrónica* si la categoría de accesorio se siente demasiado estrecha).

## Licencia

El firmware está licenciado MIT. Elige la licencia del modelo deliberadamente — no son la misma cosa, y la predeterminada de MakerWorld puede no ser lo que quieres:

| Licencia | Efecto |
|---|---|
| **CC BY** | Cualquiera puede remixar y vender, con atribución. Más permisiva, coincide con el firmware MIT. |
| **CC BY-SA** | Los remixes deben mantenerse bajo la misma licencia. |
| **CC BY-NC** | Uso comercial no permitido. Bloquea a los reventa — pero también bloquea a cualquiera que venda kits impresas para ayudar a que el proyecto se expanda. |

Dado que el firmware es MIT y la posición del proyecto está declarada como "constrúyelo, vendámoslo, forquearlo", **CC BY** es la elección consistente. Vale la pena un momento de reflexión, ya que es incómodo ajustarlo más tarde.

## Archivos para subir

| Archivo | Estado |
|---|---|
| Proyecto `.3mf`, previamente organizado con placas y ajustes | `TigerScale-V3-Case-1.0.1.3mf` |
| Partes individuales `.stl` | no publicados — el `.3mf` lleva cada parte |
| `README` / PDF de montaje | opcional; el repositorio lo cubre |

La escala V2 envió un solo `.3mf` que contenía cuatro variantes de construcción (motor / sin motor × 1 o 2 lectores) para que las personas pudieran simplemente elegir una placa y cortar. Eso funcionó bien y merece repetirse — V3 no tiene motor, pero una variante de 1 lector y una de 2 lectores cubrirían lo mismo.

## Fotos y imagen de portada

El algoritmo de MakerWorld se basa fuertemente en la imagen de portada, y una pantalla mostrando datos reales supera a un render cada vez.

- **Portada:** la escala montada con una bobina sobre ella, pantalla encendida y mostrando un peso y nombre de material. La foto ya tomada para este proyecto es casi ideal — mesa limpia, bobina en su lugar, pantalla legible.
- Ángulo en la parte frontal mostrando la interfaz de la pantalla táctil
- Parte trasera o tres cuartos mostrando el soporte de la bobina
- Una foto de los componentes internos con los cables visibles
- El propio diagrama de conexión, `docs/img/wiring-hsu.jpg` — verdaderamente útil en una lista
- Un corto video o GIF de una bobina colocada y el peso apareciendo. Si solo añades un activo adicional, añade este.

## Antes de publicar — revisa

- [ ] Ajustes de impresión completados desde una rebanada real
- [ ] Pasos de ensamblaje y tamaños de fijaciones completados
- [ ] `.3mf` se abren correctamente en Bambu Studio / Orca / PrusaSlicer
- [ ] Licencia elegida deliberadamente (ver arriba)
- [ ] Fotografía de portada muestra la pantalla con datos reales
- [ ] Enlaces de regreso al repositorio GitHub y al Discord
- [ ] eSun descrito como un **programa piloto francés**, no una expansión mundial, y
      Nanovia / Filforme no listados como en envío — están en fase de integración. Los
      alcances de los fabricantes se citan según lo declarado por ellos; ver
      `assets/partners/README.md`
- [ ] Bobinas de fábrica llamadas **TigerTag+**, no TigerTag sin más
- [x] **Después de publicar:** agregar la URL de MakerWorld a `README.md` y al BoM en
      `HARDWARE.md`, reemplazando "no publicado aún para V3"

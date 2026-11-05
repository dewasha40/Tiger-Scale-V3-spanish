# Contribuir a TigerScale V3

Las contribuciones son sinceramente bienvenidas, y algunos ámbitos necesitarían mucho apoyo en este momento:

- **Un encabezado V3.** No se ha publicado aún.
- **Verificación en bancada de la conexión SPI y I²C.** Ambas compilan; solo HSU ha sido confirmada de forma end-to-end en hardware. Si conectas una y funciona — o no —, el informe es valioso en cualquier caso.
- **Traducciones.** 11 idiomas del firmware, 9 en la interfaz web.
- **Canales OTA por transporte.** Hoy en día OTA publica un solo binario y una unidad con conexión diferente que lo toma pierde su lector.

## Antes de comenzar

Dos documentos ahorrarán más tiempo:

- **[CODEMAP.md](CODEMAP_ES.md)** — el firmware es un archivo de ~16 500 líneas. Este mapea cada sección y función para que puedas ir directamente a la parte que necesitas. Su tabla **Landmines** lista funciones con restricciones no obvias; lee la fila para cualquier cosa que vayas a tocar.
- **[docs/FIRMWARE.md](docs/FIRMWARE_ES.md)** — cómo se integran las piezas, y las trampas de LVGL v8 que cada una ha costado un viaje de hardware.

## Configuración

```bash
git clone https://github.com/TigerTag-Project/Tiger-Scale-V3.git
cd Tiger-Scale-V3
pio run -e esp32s3_hsu_b   # ~3 minutes on a first build
```

Necesitas [PlatformIO Core](https://docs.platformio.org/en/latest/core/installation/).
La IDE Arduino no puede compilar este proyecto.

No necesitas hardware para contribuir. Mucha labor útil — documentación, traducciones, herramientas, revisión — necesita solo un compilador. Solo indica en tu pull request qué pudiste probar.

## Trabajar en el firmware

El archivo es grande, así que el flujo de trabajo importa:

1. Encuentra tu objetivo en `CODEMAP.md`.
2. `grep -n "theName" TigerTagSplashESP32/TigerTagSplashESP32.ino`
3. Lee esa región, no todo el archivo.
4. Haz el cambio mínimo que haga la tarea.

Alinea el nombre, densidad de comentarios y estilo del código circundante. Por favor, no reformates código que no estés cambiando — hace imposible revisar el cambio real.

**Mantén los comentarios verdaderos.** Un comentario que contradiga el código es peor que ninguno, y este códigobase lo ha sufrido más de una vez: un comentario de transporte afirmaba que el lector I²C usaba un bus cuando el código usaba otro, y la sección LVGL aún describía como una pantalla de prueba experimental mucho después de haberse convertido en la interfaz de producción. Ambos engañaron sesiones de depuración reales.

## Antes de abrir una pull request

```bash
bash scripts/check-i18n.sh      # after touching i18n.h
bash scripts/check-codemap.sh   # after touching the .ino
bash scripts/update_toc.sh      # after adding/moving a "// §N — " banner
pio run -e esp32s3_hsu_b        # must compile
```

CI ejecuta todas estas en cada push, más una compilación de los cinco entornos. Ejecutarlas localmente solo te da la respuesta más rápido.

Si `check-codemap.sh` reporta desviación, corrige los números de línea en `CODEMAP.md` — ese es la respuesta intencional, no suprimir la comprobación.

## Añadir una cadena de UI

1. Añade la clave al enum en `TigerTagSplashESP32/i18n.h`, antes de `I18N_COUNT`.
2. Añade una entrada a **todos los 8 bloques de idioma**, en el mismo orden que el enum.
3. `bash scripts/check-i18n.sh` debe salir 0.

Si no puedes traducir a todos los 8, usa la cadena en inglés como marcador y di en la pull request — una entrada faltante rompe la compilación, un marcador no lo hace.

Las cadenas de la interfaz web son separadas: `data/www/locales/*.json`.

## Cambiar cómo se ve la UI

Dos reglas, ambas aprendidas a un costo alto:

1. **Un restyle es solo visual.** Colores, fuentes, tamaños, formas, bordes, diseño — sí. ¿Qué datos se muestra, cómo se calcula o coordenadas de zonas de toque — no. Estos están conectados a lecturas de escala viva y campos de Firestore, y se ven exactamente como chrome decorativo. Si no puedes decir cuál es una etiqueta, revisa la variable de origen.
2. **Muestra un mockup primero.** En las proporciones reales de 480×320. Iterar sobre un mockup es instantáneo; compilar, flashar y fotografiar un dispositivo no lo es.

Luego lee [LVGL v8 traps](docs/FIRMWARE_ES.md#lvgl-v8-traps). Cada uno de ellos está en esa lista porque ya capturó a alguien aquí.

## Reportar un error

Las dos cosas que hacen un informe de firmware accionable:

1. **¿Cuál entorno de compilación** usaste para flashar (`esp32s3_hsu_b` para una placa -3.5B, `esp32s3_hsu` para una -3.5).
2. **¿Cómo están conectados tus lectores.**

La mayoría de los informes "no se detectan lectores" son firmware compilado para un transporte que no coincide con la conexión — lo que produce ningún mensaje de error, así que es invisible sin esos dos hechos.

Luego el log serial alrededor del fallo, y `TIGERSCALE_FW_VERSION`. La plantilla de problema pide todo esto.

## Cosas que están decididas

Estas han sido investigadas y cerradas; por favor no las reabres sin nueva evidencia:

| Tema | ¿Por qué |
|-------|-----|
| NFC como host USB | Físicamente imposible en esta placa — [postmortem](docs/USB_HOST_POSTMORTEM_ES.md) |
| Soporte para IDE Arduino | La configuración de LVGL necesita una ruta de inclusión que el IDE no tiene |
| `Wire` / GPIO21+22 | GPIO22 no existe en el ESP32-S3 |
| Rotación de bobina motorizada | No hay motor en este hardware |
| Tare automática con debounce corto | Eliminado — tareaba sobre el peso propio de la bobina |

## Seguridad

Por favor, no abras un problema público para un problema de seguridad. Ver [SECURITY.md](SECURITY_ES.md).

## Licencia

Las contribuciones se aceptan bajo la [Licencia MIT](LICENSE), los mismos términos que el resto del proyecto.

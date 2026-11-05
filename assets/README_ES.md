# Activos

Activos oficiales de marca y ilustración para TigerScale V3. Todo lo contenido aquí está commitido para que el README, la documentación y — donde sea útil — el dispositivo mismo puedan hacer referencia a un conjunto único en lugar de mantener una copia privada cada uno.

## En uso actualmente

| Archivo | Ubicación | Notas |
|--------|----------|-------|
| `logo-tigertag-head.svg` / `-dark.svg` | Encabezado del README | Par de luz/oscuro en un `<picture>`. La versión oscura es el mismo arte recoloreado, por lo que el logotipo permanece visible en el tema oscuro de GitHub. |
| `spool-tagged.png` | README, "¿Qué hace?" | Una bobina de recarga con su chip TigerTag redondo. Muestra la idea más rápido que un párrafo. |
| `partner-box-{rosa3d,esun,sunlu}.jpg` | README, sección de socios | Empaques de venta al por menor que llevan el logotipo RFID TigerTag — evidencia de las alianzas en lugar de una afirmación sobre ellas. |
| `Hero-TigerSystem-ecosystem.png` | README, sección de ecosistema | Studio Manager, la aplicación móvil y un TigerPOD. Intencionalmente no es la imagen de encabezado: muestra el sistema, no esta escala. |
| `partners/*.svg` | README, sección de socios | Las ocho marcas de filamento, en sus propios colores. Consulte `partners/README.md`. |

## Disponibles, pero aún no colocados

Íconos de estilo de aplicación, conservados para el instalador web y para cualquier cosa orientada al usuario que necesite un logotipo cuadrado:

| Archivo | Tamaño | Uso probable |
|--------|--------|-------------|
| `icon.png` | 1024x1024 | Ícono del instalador web, Open Graph / vista previa social |
| `icon.ico` | 256x256 | Ícono favicon legado |
| `tigersystem_icon.svg` | vector | Ícono del instalador web / PWA |
| `logo_tigersystem_app_icon.svg` | vector | **Idéntico byte a byte a `tigersystem_icon.svg`** — mismo archivo bajo dos nombres |
| `logo_tigersystem_app_icon_rounded.svg` | vector | Variante con bordes redondeados, para un ícono PWA mascarable |
| `logo_tiger_icon_{contained,overflow,square}.svg` | vector | Tres marcos del logotipo del tigre |
| `tiger-head-contour.svg` | vector | Mismo arte que `logo-tigertag-head.svg`, exportación diferente |

Dos cosas importantes antes de acceder a estos: `tigersystem_icon.svg` y `logo_tigersystem_app_icon.svg` son los mismos bytes, por lo que elige uno y manténlo; y la cabeza del contorno existe dos veces por la misma razón.

## Colocar un activo en la pantalla del dispositivo

El firmware no lee SVG o PNG. Dibuja arreglos pre-rasterizados RGB565 compilados en el binario — `logo_tigertag_splash.h` (480x320 splash de arranque), `logo_tigertag.h` (150x150 screensaver), `icon_bolt.h` (8x14 ícono de carga).

`scripts/make-rgb565-header.py` genera ese formato:

```bash
# 1. render/resize to the exact target size
sips -z 320 480 assets/your-art.png --out /tmp/splash.png

# 2. convert
python3 scripts/make-rgb565-header.py /tmp/splash.png \
    --name gTigerTagSplash --prefix TIGERTAG_SPLASH --expect 480x320 \
    > TigerTagSplashESP32/logo_tigertag_splash.h
```

Utiliza solo la biblioteca estándar — sin Pillow, sin ImageMagick — y rechaza ejecutarse en imágenes de tamaño incorrecto en lugar de adivinar un ajuste.

**Tenga en cuenta el costo.** Una imagen de pantalla completa 480x320 es 153 600 píxeles, aproximadamente **300 KB de flash** como un `const uint16_t[]`, y se enlaza a cada compilación sin importar si se muestra o no. Por eso el firmware lleva solo una decena. Antes de agregar uno, considere si una pantalla dibujada con LVGL haría lo mismo — y note que una idea anterior de "un encabezado por marca de socio" generó 6,5 MB de encabezados que nunca se referenciaron en absoluto.

Los SVGs necesitan ser rasterizados primero; esta máquina no tenía instalado un convertidor, así que use su herramienta de diseño o un navegador para exportar un PNG al tamaño objetivo.

## Archivos de trabajo

La fuente de Illustrator de la que se exportan los SVGs de socio **no está** commitida — 5,8 MB, útil solo para quien haga el trabajo de diseño. De igual forma `icon.icns`, que solo un paquete de aplicación macOS querría. Ambos están en `.gitignore`.

## Marcas registradas

Los logotipos TigerTag y TigerSystem son propiedad del proyecto. Los logotipos de marcas de filamento bajo `partners/` pertenecen a sus respectivos dueños y se incluyen para registrar quién envía el protocolo — no se implica ningún respaldo de este firmware en ninguna dirección. Consulte `partners/README.md` para las reglas editoriales que los acompañan.

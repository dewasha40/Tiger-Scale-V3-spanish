# AGENTS.md — instrucciones para agentes de codificación de IA

Convenios independientes de herramientas para este repositorio. Si eres Claude Code, lee
[`CLAUDE.md`](CLAUDE_ES.md) en su lugar — es un superconjunto de este archivo.

## La única regla que más importa

**El firmware es un solo archivo de ~17 200 líneas. Nunca lo leas completo.**

```
1. Open CODEMAP.md, find the section or function by name
2. grep -n "theName" TigerTagSplashESP32/TigerTagSplashESP32.ino
3. Read that line ±60 — nothing more
4. Make the smallest edit that satisfies the task
5. bash scripts/check-codemap.sh
```

Leer todo el archivo quema el contexto que necesitas para el trabajo real, y los
números de línea en `CODEMAP.md` son un punto de partida para el grep, no un
sustituto de éste.

## Estructura del repositorio

| Ruta | ¿Qué es? |
|------|-----------|
| `TigerTagSplashESP32/TigerTagSplashESP32.ino` | Todos los firmware. 26 secciones numeradas más §AUDIO y §LVGL. |
| `TigerTagSplashESP32/i18n.h` | Cada cadena de interfaz de usuario en dispositivo: 120 claves × 11 idiomas |
| `TigerTagSplashESP32/logo_*.h`, `icon_bolt.h` | Bitmaps RGB565 pre-rasterizados |
| `include/lv_conf.h` | Configuración de LVGL v8.4 (encontrada mediante `-I include`) |
| `data/www/` | Interfaz web servida desde LittleFS |
| `data/www/locales/*.json` | Traducciones de la interfaz web (11 archivos, separados de `i18n.h`) |
| `platformio.ini` | Ambientes de compilación — uno por cada transporte NFC |
| `partitions.csv` | Diseño de la flash. NVS en 0x9000–0xE000 almacena estado WiFi + Firebase. |
| `scripts/` | Scripts de protección y asistente de flash |
| `docs/` | Documentación humana |
| `CODEMAP.md` | Mapa de secciones y funciones — **comienza aquí** |
| `WORKLOG.md` | Registro de cambios en marcha desde el último punto de control |

## Antes de terminar — un comando

```bash
bash scripts/verify.sh          # every check CI runs, plus the reference build
bash scripts/verify.sh --fix    # regenerate the TOC and CODEMAP first
bash scripts/verify.sh --all    # all five envs, exactly what CI does
bash scripts/verify.sh --quick  # checks only, no compiling
```

Cubre el índice, anclas de CODEMAP, ambos conjuntos de traducciones, emojis,
corrupción de caracteres y si existen notas de lanzamiento para la versión actual.

Nada mecánico debe editarse manualmente. Si CODEMAP se desvía,
`python3 scripts/sync-codemap.py` reescribe los números de línea desde la fuente y
no toca nada más — los nombres de función, secciones y cada nota en la tabla
Landmines son conocimiento escrito a mano y permanecen inalterados.

Nota para cualquiera tentado a confiar en una marca verde: hasta el primer
commit de este repositorio, `check-codemap.sh` imprimía `VERIFICACIÓN DE CODEMAP
PASADA` en macOS **sin verificar nada** — utilizaba `mapfile` y `grep -P`, que bash
3.2 y grep BSD no tienen, por lo que su lista de anclas salía vacía y tomaba la
rama "nada que hacer". Ambos guardas ahora fallan con ruido. Si los reescribes,
mantiene esa propiedad: un conjunto de entrada vacío es un error, no una pasada.

## Convenciones de edición

- **Diferencia más pequeña posible.** Ajusta el nombre, densidad de comentarios y idiomas del código circundante. No reformatear al azar.
- **Los comentarios deben ser verdaderos.** Un comentario que contradiga el código es peor que no tener comentario. Varios errores en la historia de este proyecto se prolongaron exactamente por eso: el comentario de transporte afirmaba que el lector I2C estaba en `Wire` cuando la clase lo construye con `&Wire1`, y el banner de LVGL aún se llamaba pantalla de prueba experimental mucho después de haberse convertido en la interfaz de producción.
- **No codificar un error en la herramienta.** `update_toc.sh` usaba a veces coincidir con la corrupción literal que `§` había decayido en, en lugar del `§` mismo. La corrupción entonces parecía "normal" y se propagaba. Corrige la causa.
- **Actualiza `WORKLOG.md` conforme avanzas**, nombrando los archivos tocados. Sus encabezados son de Keep a Changelog's (`Añadido` / `Modificado` / `Corregido` / `Eliminado`), por lo que la entrada de lanzamiento se sintetiza a partir de ella, no se rederiva del diff. Describe el estado final, no el viaje.
- **Nunca commits, etiquetas o subes sin ser solicitado.** Haz el cambio, ejecuta los guardas, reporta y detente.
- **Instala los hooks una vez por clon**: `bash scripts/install-hooks.sh`.
  La verdad local de la máquina — dirección de bench, puerto serie, cuenta que sube a dónde —
  vive en el `LOCAL.md` ignorado por git, nunca en un archivo comprometido.

## Cosas que están decididas — no re-litigar

| Tema | Estado |
|-------|--------|
| NFC como host USB | Físicamente imposible en esta placa. Postmortem en `docs/USB_HOST_POSTMORTEM.md`. No reintentar. |
| Soporte para IDE Arduino | No posible — la configuración de LVGL necesita `-I include`. Solo PlatformIO. |
| `Wire` / GPIO21+22 | Dañado en esta placa. Usa `Wire1` (GPIO8/7). |
| Rotación de bobina motorizada | No hay motor en esta hardware. `setupServo()` es intencionalmente un no-op. |
| Tareo automático de rebobinado corto | Eliminado intencionalmente — tareaba sobre el peso propio de la bobina. Solo el manejador de desviación negativa de 1 s permanece. |
| Ventana de deduplicación RFID de 1,5 s | Nunca ampliada deliberadamente. |

## Hardware que no puedes probar desde aquí

La mayoría de este firmware habla con hardware que un agente no puede alcanzar: celda de carga, lectores NFC, PMIC, panel táctil, codec. Compilar demuestra sintaxis, no comportamiento.

Así que: describe claramente lo que verificaste y lo que no. "Compila limpio para los cinco ambientes; la ruta de pesaje no está verificada sin el dispositivo" es útil. Implicar que un comportamiento de hardware fue confirmado cuando solo la compilación fue el caso no es.

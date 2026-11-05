# Brief de revisión permanente

Dirige a un revisor a este archivo. Contiene el alcance, los ejes y el formato de salida, por lo que una revisión puede solicitarse en una frase y aún así regresar comparable a la última.

Las revisiones son **de solo lectura**. Un revisor que comienza a corregir cosas deja de prestar atención, y los hallazgos que se habrían realizado más adelante nunca ocurren.

## Cuándo ejecutar una

- Antes de cualquier liberación que cambie el modelo de datos, la ruta OTA o algo contra lo que un tercero integra.
- Cada pocos lanzamientos de forma alternativa.
- Una **revisión orientada** — un eje, un subsistema — es más barata que una revisión completa y merece ejecutarse mucho más a menudo. Prefiere cuatro pequeñas a una grande.

## Alcance

Firmware para una balanza de filamento conectada. Un archivo C++ de Arduino de aproximadamente 14 000 líneas más `i18n.h` y `live_page.h`, una interfaz web LittleFS bajo `data/www`, y un instalador web bajo `web-installer`. Comienza desde [`CODEMAP.md`](../CODEMAP_ES.md) — nunca leas el .ino desde el principio.

Nombra la sección (`§N`) y la función en cada hallazgo.

## Ejes

| Eje | Busca específicamente aquí |
|---|---|
| **Corrección** | Máquinas de estado que pueden quedar bloqueadas; resultados que dependen de cuál de las dos rutas de código se ejecutó (véase el `measure_gr` de la tabla de minas para la forma que toma esto); valores registrados como enviados que no se enviaron |
| **Memoria** | Cualquier cosa que conserve RAM interna. Es el recurso escaso: el heap libre alcanza kilobytes de una cifra bajo carga, y agotarse mata al dispositivo en algún lugar irrelevante — mDNS abortando dentro del hilo lwIP, con una traza de retroceso que no mencione ni el archivo ni la característica que lo agotó |
| **Modos de fallo** | Los silenciosos sobre todo: una obtención que devuelve 0 en error, un envío sin reintento, un guardián que se retira sin avisar. Una respuesta incorrecta que nadie es informado es peor que un error |
| **Verdad del hardware** | Afirmaciones sobre la placa que el esquema o la mesa contradicen. `docs/HARDWARE.md` y la tabla "Hechos del hardware que mueren" en `CLAUDE.md` son la referencia |
| **Experiencia de usuario al primer intento** | ¿Alguien que se esté parado en la balanza puede decir qué está haciendo y si funcionó? El estado que existe internamente pero nunca llega a la pantalla cuenta como un hallazgo |
| **Costo de token** | Documentos que repiten lo que el código posee sin un guardián; archivos que un agente debe leer enteros porque nada los indexa |

## Formato de salida

Un archivo en `docs/reviews/`, nombrado `YYYY-MM-DD-<alcance>.md`.

- **Ganancias rápidas** primero: una tabla de elementos cada uno con alcance aproximado de una hora. Este es donde comienza la próxima sesión.
- Luego los hallazgos, ordenados por gravedad más alta primero. Cada uno: dónde (`§N`, función), qué falla y el input o estado concreto que lo activa. Un hallazgo sin un escenario de fallo es una opinión.
- Diga explícitamente qué no revisó.

## ¿Por qué se conservan estos archivos?

Después de trabajar un informe, anota cada hallazgo **corregido**, **pospuesto** (con el motivo) o **rechazado** (llamada incorrecta). Un informe no anotado se lee para el revisor siguiente como si nada se hubiera hecho, y los mismos hallazgos se re-litigan desde cero — lo cual cuesta más que la revisión misma.

## ¿Qué cambia esto?

<!-- Resume el cambio en una o dos frases. Enlaza la incidencia si existe. -->

## ¿Por qué?

<!-- ¿Qué problema resuelve? -->

## ¿Cómo se ha probado?

Indica con precisión qué has podido verificar y qué no. Este proyecto se comunica
con hardware al que la integración continua no tiene acceso. Decir «compila» es
honesto y útil; no afirmes haber confirmado el comportamiento en hardware si solo
has compilado el firmware.

- [ ] Compila: `pio run -e esp32s3_hsu_b`
- [ ] Compila para todos los entornos (también lo comprueba CI)
- [ ] Probado en hardware real — ¿con qué cableado? ______
- [ ] No se puede probar sin hardware del que no dispongo

## Lista de comprobación

- [ ] `bash scripts/check-codemap.sh` termina correctamente (y he actualizado
      los números de línea de CODEMAP.md si indicó que estaban desfasados)
- [ ] `bash scripts/check-i18n.sh` termina correctamente (si he modificado `i18n.h`)
- [ ] He ejecutado `bash scripts/update_toc.sh` (si añadí o moví un encabezado `// §N — `)
- [ ] He actualizado `WORKLOG.md` — **solo mantenedores**. Si colaboras desde un fork,
      no modifiques WORKLOG.md: es el registro de cambios desde la última versión,
      se reinicia en cada punto de control y los cambios de un fork siempre crearían conflictos.
- [ ] Los comentarios que he tocado siguen describiendo correctamente el código
- [ ] El diff es lo más pequeño posible y no incluye cambios de formato ajenos a la tarea

## Si el cambio afecta a la interfaz

- [ ] Es solo visual: no cambia los datos mostrados, cómo se calculan ni las
      coordenadas de las zonas táctiles
- [ ] He consultado las [particularidades de LVGL v8](../blob/main/docs/FIRMWARE_ES.md#lvgl-v8-traps)
- [ ] He adjuntado una captura de pantalla o una foto

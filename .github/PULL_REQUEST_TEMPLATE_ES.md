## ¿Qué cambia esto?

<!-- Una o dos frases. Vincula el problema si existe uno. -->

## ¿Por qué?

<!-- ¿Qué problema resuelve? -->

## ¿Cómo se probó?

Se debe ser específico sobre lo que se pudo y no se pudo verificar — este proyecto interactúa con hardware que CI no puede alcanzar, por lo que "compila" es honesto y útil, mientras que implicar que se confirmó un comportamiento del hardware cuando solo se verificó la compilación no es correcto.

- [ ] Compila: `pio run -e esp32s3_hsu_b`
- [ ] Compila para cada entorno (CI también lo hace)
- [ ] Probado en hardware real — ¿qué conexión se utilizó? ______
- [ ] No es verificable sin hardware que no tengo

## Lista de verificación

- [ ] `bash scripts/check-codemap.sh` pasa (y actualicé las líneas de CODEMAP.md si reportó desviación)
- [ ] `bash scripts/check-i18n.sh` pasa (si toqué `i18n.h`)
- [ ] Ejecutar `bash scripts/update_toc.sh` (si agregué o moví un banner `// §N — `)
- [ ] Actualizar `WORKLOG.md` — **solo mantenedores**. Contribuyendo desde una rama? Deje `WORKLOG.md` tal cual: es el registro de cambios desde el último commit de los mantenedores, se reinicia en cada hito, por lo que una PR desde una fork que lo modifique siempre terminará en conflicto.
- [ ] Los comentarios que toqué aún son verdaderos en relación con el código circundante
- [ ] Diff más pequeño que haga la tarea — sin reformateo innecesario

## Para un cambio de interfaz de usuario

- [ ] Cambio visual solo: sin alteración de los datos mostrados, cómo se calculan o las coordenadas de zona de toque
- [ ] Revisé los [trampas de LVGL v8](../blob/main/docs/FIRMWARE.md#lvgl-v8-traps)
- [ ] Adjuntar captura de pantalla o foto

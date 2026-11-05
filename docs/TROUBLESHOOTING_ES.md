# Solución de problemas

Ordenado aproximadamente por la frecuencia con la que cada cosa es la causa real.

## "No se detectan lectores RFID"

**Comprueba primero el entorno de compilación.** Esta es la causa más común, y cuesta horas cada vez que no se comprueba primero.

El transporte NFC se elige en tiempo de compilación. El firmware compilado para el transporte incorrecto detecta nada, reporta ningún error y se ve exactamente como una falla de conexión o un módulo muerto.

```bash
pio run -e esp32s3_hsu_b # HSU / UART wiring, -3.5B board (2 readers, reference build)
pio run -e esp32s3       # SPI wiring         (2 readers)
pio run -e esp32s3_i2c   # I2C on Wire1       (1 reader)
```

Asegúrate de que el entorno coincida con cómo están físicamente conectados los módulos
([HARDWARE.md](HARDWARE_ES.md#nfc-wiring--pick-one-then-build-the-matching-env)),
y verifica la posición del interruptor de modo del módulo mientras estás allí —
HSU = OFF/OFF, I²C = ON/OFF, SPI = OFF/ON en la silkscreen Elechouse V3.

Si el entorno es correcto:

1. **Intenta el nivel de potencia RF.** Configuración → Hardware tiene un `−`/`+` de pasos; el cambio se aplica en la siguiente escaneo inmediato. El valor predeterminado es 3. Esto es genuinamente frecuente la respuesta, porque las dos antenas se miran entre sí ~75 mm y hay interferencia cruzada.
2. **Observa los bytes reales.** Construye un entorno de depuración y abre el monitor serial:
   ```bash
   bash scripts/flash.sh --env esp32s3_hsu_debug --monitor
   ```
   Cada frame enviado a y recibido del lector se imprime. Esto separa
   "mi conexión es incorrecta" (nada en absoluto) de "mi etiqueta no es legible" (frames salen, nada regresa).
3. **Verifica GND.** Los módulos necesitan un suelo común con el ESP32. Esta es la conexión más olvidada.
4. **Intenta un lector a la vez.** `RFID_READER_TEST_MODE` en §1: `1` = solo derecho,
   `2` = solo izquierdo.

Si estás considerando un lector NFC conectado por USB: no puede funcionar en esta placa.
[USB_HOST_POSTMORTEM.md](USB_HOST_POSTMORTEM_ES.md) explica por qué, con suficiente detalle
que no necesitarás probarlo tú mismo.

## El log dice `[HX711] no listo` — ¿mi célula de carga está muerta?

Probablemente no. Este mensaje **no** significa que la HX711 esté desconectada o rota.

Se imprime cada vez que `scale.is_ready()` devuelve falso, limitado a una vez cada 2 segundos. En la unidad de referencia aparece continuamente mientras la báscula sigue midiendo correctamente — la pantalla muestra un peso real y las lecturas llegan a la nube.

Verifica si estás obteniendo lecturas antes de investigar hardware:

```bash
python3 scripts/watch_logs.py <device-ip> --all | grep ALIVE
```

```
[ALIVE] up=29777s heap=52412 min=2892 wifi=1 firebase=1 wf=0 hx=0 weight=0.74
```

Si `weight=` cambia cuando presionas en la plataforma, la célula de carga está bien.

¿Qué está pasando: después de **600 ms** de muestras perdidas continuas, `readWeight()` fuerza el peso a 0. Bloquear trabajo de red en `loop()` — especialmente las llamadas HTTPS de Firebase — puede detenerse lo suficiente para disparar eso, en hardware que funciona perfectamente. Nota `min=2892` en la línea anterior: la memoria libre mínima baja a 2.9 KB, que es la misma presión de memoria discutida en
[FIRMWARE.md](FIRMWARE_ES.md#memory).

Así que: una báscula que lee correctamente pero registra esto muestra una brecha de robustez en `readWeight()`, no una falla de conexión. Una báscula que registra esto **y** nunca muestra un peso es un problema de conexión — lee la sección siguiente.

## La báscula lee 0 g, o negativo, sin importar qué

**Comprueba el signo de la lectura bruta antes de tocar algo en software.** Añade un `Serial.printf` temporal justo después de `scale.get_units()` en `readWeight()`.

Una conexión invertida de cable de célula de carga produce exactamente este síntoma, porque un `if (weight < 0) weight = 0` clamping hace que una falla de polaridad hardware sea indistinguible de un error de filtro o auto-tara. Esto ha ocurrido aquí antes, después de una larga búsqueda en el software.

Si el signo es correcto, vuelve a ejecutar el asistente de calibración (Configuración → Calibración) con un peso de referencia conocido.

## El peso se desvía, o la lectura es saltarina

- Re-calibra. El factor de calibración es por unidad.
- Asegúrate de que la célula de carga esté montada sin ninguna restricción mecánica — un cable o borde de carcasa tocando la plataforma hará esto.
- Las constantes de filtrado viven en §6; el comportamiento intencionado es un EMA rápido durante un cambio y un EMA fino una vez que se estabilice.

## Las credenciales WiFi se pierden después de un reflash

Estás flashando `firmware.factory.bin` en `0x0000`. Esa es una imagen fusionada que abarca continuamente desde `0x0000` hacia arriba, por lo que sobrescribe NVS (0x9000–0xE000), donde viven las credenciales WiFi y la sesión Firebase guardadas.

Usa `bash scripts/flash.sh` o `pio run -t upload`, ambos de los cuales escriben cada imagen en su propio desplazamiento y dejan NVS intacto.

## El dispositivo se inicia en un bucle de crash

Si flashes manualmente con tu propia invocación de esptool, verifica el modo de flash:
**DIO es requerido**. QIO da un bucle de inicio de crash en `ets_loader.c` en esta placa.
`scripts/flash.sh` y la configuración del board de PlatformIO ya establecen esto correctamente.

De lo contrario, conecta el monitor serial y lee el panic:

```bash
pio device monitor
```

El filtro `esp32_exception_decoder` está habilitado en `platformio.ini`, por lo que
los backtraces salen con nombres de funciones adjuntos.

## El flasher muestra ninguna puerto COM

La placa nunca aparece en la lista, por lo que no hay nada para seleccionar. Eso es una falta de controlador o un cable USB de carga solo, no una falla en la báscula — las instrucciones por sistema operativo están en [SERIAL-PORT.md](SERIAL-PORT_ES.md).

## Nada en la consola serial

`ARDUINO_USB_CDC_ON_BOOT=1` debe estar establecido — está, en cada entorno — de lo contrario
`Serial.println()` es invisible por el puerto USB-C y solo `log_e()` pasa.
Si añadiste un entorno de compilación, asegúrate de que heredaste `${common.build_flags}`.

El firmware también mantiene las últimas ~80 líneas de log en RAM y las sirve sobre HTTP,
que es la única forma de leerlas una vez que se desconecta el cable:

```bash
python3 scripts/watch_logs.py 192.168.1.42
python3 scripts/watch_logs.py tigerscale-XXXX.local --all
```

## La pantalla táctil no responde

El tacto está en `Wire1` (SDA=GPIO8, SCL=GPIO7) en la dirección 0x3B. Verifica el log de inicio para
el escaneo I²C.

También verás `perimanSetPinBus(): Pin inválido: 22` y un escaneo que encuentra cero
dispositivos en el *otro* bus en cada inicio. Eso es esperado y sin daño — `Wire`
(GPIO21/22) está roto por diseño en esta placa, GPIO22 no existe en el
ESP32-S3. No está relacionado con tu problema de tacto.

## La interfaz web está en blanco o da 404

La interfaz web vive en una partición separada de LittleFS y no se escribe por un flash normal
de firmware:

```bash
bash scripts/flash.sh --fs
```

## La sincronización con la nube no está ocurriendo

1. Confirma el inicio de sesión: Configuración → Nube.
2. Confirma que la etiqueta está siendo leída — la báscula suena y muestra el material.
3. El peso debe ser superior a `MIN_WEIGHT_TO_SEND_G`; una plataforma casi vacía no enviará.
4. Revisa `python3 scripts/watch_logs.py <ip>` para las líneas `[META]` y `[FIRESTORE]`.

Nota que `readInventoryContainerWeight()` devuelve 0 en cualquier fallo sin reintentar, por lo que un rollo cuyo peso del contenedor es desconocido sincroniza con net = 0 en lugar de fallar fuerte.

## Nada aquí ayudó

Abre un problema con: el entorno de compilación que usaste, la conexión de transporte, el log serial alrededor del fallo, y `TIGERSCALE_FW_VERSION`. El entorno y la conexión son las dos cosas que hacen que un reporte sea accionable.

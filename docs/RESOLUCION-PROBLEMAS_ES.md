# Resolución de problemas

Ordenada aproximadamente según la frecuencia con la que cada problema es la causa real.

## «No se detectan lectores RFID»

**Comprueba primero el entorno de compilación.** Es la causa más habitual y puede hacer perder horas si no se verifica al principio.

El transporte NFC se elige al compilar. Un firmware creado para el transporte equivocado no detecta nada, no muestra errores y parece exactamente un fallo de cableado o un módulo averiado.

```bash
pio run -e esp32s3_hsu_b # cableado HSU / UART, placa -3.5B (2 lectores, compilación de referencia)
pio run -e esp32s3       # cableado SPI (2 lectores)
pio run -e esp32s3_i2c   # I2C en Wire1 (1 lector)
```

Elige el entorno que corresponda al cableado físico de tus módulos ([HARDWARE.md](HARDWARE.md#nfc-wiring--pick-one-then-build-the-matching-env)). Comprueba también la posición del selector de modo del módulo: en la serigrafía Elechouse V3, HSU = OFF/OFF, I²C = ON/OFF y SPI = OFF/ON.

Si el entorno es correcto:

1. **Prueba el nivel de potencia RF.** En **Ajustes → Hardware** hay un control `−`/`+`; el cambio se aplica en el siguiente escaneo. El valor predeterminado es 3. Esta suele ser la solución porque las dos antenas están enfrentadas a unos 75 mm y producen interferencias.
2. **Observa los bytes reales.** Compila el entorno de depuración y abre el monitor serie:
   ```bash
   bash scripts/flash.sh --env esp32s3_hsu_debug --monitor
   ```
   Se imprime cada trama enviada al lector y recibida de él. Así se distingue «el cableado está mal» (no hay actividad) de «la etiqueta no se puede leer» (se envían tramas, pero no vuelve ninguna).
3. **Comprueba GND.** Los módulos y el ESP32 deben compartir tierra. Es el cable que más suele olvidarse.
4. **Prueba cada lector por separado.** Configura `RFID_READER_TEST_MODE` en §1: `1` = solo derecha, `2` = solo izquierda.

Si estás pensando en conectar un lector NFC por USB, no puede funcionar en esta placa. [USB_HOST_POSTMORTEM.md](USB_HOST_POSTMORTEM.md) explica el motivo y evita que tengas que probarlo por tu cuenta.

## El registro dice `[HX711] not ready`: ¿está averiada la célula de carga?

Probablemente no. Este mensaje **no** significa que el HX711 esté desconectado o roto.

Se imprime cada vez que `scale.is_ready()` devuelve false, con un límite de una vez cada 2 segundos. En la unidad de referencia aparece continuamente aunque la báscula pese bien: la pantalla muestra un peso real y las lecturas llegan a la nube.

Antes de investigar el hardware, comprueba si realmente se están obteniendo lecturas:

```bash
python3 scripts/watch_logs.py <device-ip> --all | grep ALIVE
```

```
[ALIVE] up=29777s heap=52412 min=2892 wifi=1 firebase=1 wf=0 hx=0 weight=0.74
```

Si `weight=` cambia al presionar la plataforma, la célula de carga funciona.

Qué ocurre: después de **600 ms** sin muestras consecutivas, `readWeight()` fuerza el peso a 0. Una operación de red bloqueante en `loop()` —en particular llamadas HTTPS a Firebase— puede durar lo suficiente para activar esa condición incluso si el hardware funciona perfectamente. Observa `min=2892`: el mínimo de memoria libre llegó a 2,9 KB, la misma presión de memoria descrita en [FIRMWARE.md](FIRMWARE.md#memory).

En resumen: si la báscula pesa correctamente pero registra ese mensaje, hay un problema de robustez en `readWeight()`, no un fallo de cableado. Si lo registra **y además** nunca muestra un peso, revisa el cableado en la sección siguiente.

## La báscula siempre indica 0 g o un valor negativo

**Comprueba el signo de la lectura sin procesar antes de cambiar el software.** Añade temporalmente un `Serial.printf` justo después de `scale.get_units()` en `readWeight()`.

Un cable de la célula de carga conectado al revés produce exactamente este síntoma: una condición posterior `if (weight < 0) weight = 0` oculta el error de polaridad y lo hace parecer un fallo del filtro o de la tara automática. Ya ocurrió en este proyecto tras una búsqueda larga en el software.

Si el signo es correcto, repite el asistente de calibración (**Ajustes → Calibración**) con una masa de referencia conocida.

## El peso deriva o fluctúa

- Vuelve a calibrar. El factor de calibración es específico de cada unidad.
- Comprueba que la célula de carga esté montada sin roces mecánicos; un cable o el borde de la caja que toque la plataforma puede provocar este problema.
- Las constantes de filtrado están en §6. El comportamiento previsto es usar una EMA rápida mientras cambia el peso y una EMA más fina cuando se estabiliza.

## Las credenciales Wi-Fi desaparecen al volver a cargar el firmware

Estás cargando `firmware.factory.bin` en `0x0000`. Es una imagen combinada que ocupa la memoria de forma continua desde 0x0000 y sobrescribe NVS (0x9000–0xE000), donde se guardan Wi-Fi y la sesión de Firebase.

Usa `bash scripts/flash.sh` o `pio run -t upload`: ambos escriben cada imagen en su desplazamiento correspondiente y dejan NVS intacta.

## El dispositivo entra en un ciclo de reinicios por error

Si cargaste el firmware manualmente con esptool, comprueba el modo de flash: se requiere **DIO**. QIO provoca un ciclo de fallos de arranque en `ets_loader.c` en esta placa. `scripts/flash.sh` y la configuración de placa de PlatformIO ya lo establecen correctamente.

Si no es eso, conecta el monitor serie y consulta el error:

```bash
pio device monitor
```

El filtro `esp32_exception_decoder` está habilitado en `platformio.ini`, así que las trazas de retroceso muestran los nombres de las funciones.

## El instalador no muestra ningún puerto COM

La placa no aparece en la lista y no hay ningún puerto que seleccionar. Lo más probable es que falte un controlador o que el cable USB solo sirva para cargar; no es un fallo de la báscula. Consulta las instrucciones por sistema operativo en [SERIAL-PORT.md](SERIAL-PORT.md).

## No aparece nada en la consola serie

Debe estar definido `ARDUINO_USB_CDC_ON_BOOT=1`; lo está en todos los entornos. Si no, `Serial.println()` no se ve por USB-C y solo llegan los mensajes de `log_e()`. Si añadiste un entorno de compilación, comprueba que herede `${common.build_flags}`.

El firmware también conserva en RAM las últimas ~80 líneas del registro y las sirve por HTTP. Es la única forma de leerlas cuando se desconecta el cable:

```bash
python3 scripts/watch_logs.py 192.168.1.42
python3 scripts/watch_logs.py tigerscale-XXXX.local --all
```

## La pantalla táctil no responde

El panel táctil usa `Wire1` (SDA=GPIO8, SCL=GPIO7) en la dirección 0x3B. Comprueba el escaneo I²C en el registro de arranque.

En cada arranque también verás `perimanSetPinBus(): Invalid pin: 22` y un escaneo que no encuentra dispositivos en el *otro* bus. Es normal e inocuo: `Wire` (GPIO21/22) no funciona por diseño en esta placa y el ESP32-S3 no tiene GPIO22. No está relacionado con el problema táctil.

## La interfaz web está vacía o devuelve 404

La interfaz web ocupa una partición LittleFS independiente; una carga normal del firmware no la escribe:

```bash
bash scripts/flash.sh --fs
```

## No se sincroniza con la nube

1. Confirma el inicio de sesión en **Ajustes → Nube**.
2. Comprueba que se lea la etiqueta: la báscula emite un pitido y muestra el material.
3. El peso debe superar `MIN_WEIGHT_TO_SEND_G`; con la plataforma casi vacía no se enviarán datos.
4. Busca las líneas `[META]` y `[FIRESTORE]` en `python3 scripts/watch_logs.py <ip>`.

Ten en cuenta que `readInventoryContainerWeight()` devuelve 0 ante cualquier fallo y no vuelve a intentarlo. Por eso, si se desconoce el peso del recipiente de una bobina, la sincronización envía un peso neto de 0 en vez de mostrar un error claro.

## Nada de esto ha ayudado

Abre una incidencia e incluye el entorno de compilación usado, el cableado del transporte, el registro serie de la zona del fallo y `TIGERSCALE_FW_VERSION`. El entorno y el cableado son los datos que permiten actuar sobre el informe.

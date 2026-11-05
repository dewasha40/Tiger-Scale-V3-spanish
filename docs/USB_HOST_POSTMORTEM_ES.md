# Postmortem — NFC USB-host en el ESP32-S3-Touch-LCD-3.5

**Verdict: imposible físicamente en esta placa. No vuelva a intentarlo en el firmware.**

Este documento existe para que la conclusión sobreviva al código siendo eliminado. Si estás a punto de intentar hacer funcionar un lector NFC USB desde esta placa, léelo primero — te ahorrará varios días.

## Lo que se intentó

Algunos módulos PN532 vienen como dispositivos USB-C sellados sin ninguna interfaz GPIO accesible. La idea era seguir usando tal módulo haciendo que el ESP32-S3 actúe como **USB Host**, con el módulo conectado a un hub autónomo, comunicándose con él a través de USB-CDC mediante su chip CH340 de puente.

Implementación: un entorno `esp32s3_usbhost` de PlatformIO, la biblioteca `EspUsbHost`, y una versión vendida de `Adafruit_PN532` cuyo constructor serial aceptaba un `Stream*` genérico en lugar de un `HardwareSerial*` para poder pasar un objeto de puente USB-CDC.

## ¿Por qué no puede funcionar

Dos líneas de evidencia independientes, una empírica y otra del esquema.

### 1. En la mesa de pruebas

El diagnóstico en pantalla informaba consistentemente:

```
host=OK  dev=NONE
```

La pila USB Host se inicializaba correctamente cada vez (`usb.begin()` tenía éxito), pero ningún dispositivo fue nunca enumerado a través del hub. No intermitentemente — nunca.

### 2. En el esquema

`ESP32-S3-Touch-LCD-3.5-Schematic.pdf`, conector **J5**:

- **CC1 y CC2 cada uno lleva una resistencia de 5.1 kΩ de pull-down** (R24, R26).
- **No hay ninguna fuente de VBUS o circuito integrado de conmutación en el diseño.** La VBUS del conector solo alimenta la AXP2101 PMIC como entrada de carga.

En USB-C, una resistencia de 5.1 kΩ de pull-down (Rd) en CC es la declaración eléctrica *"Soy un dispositivo"* — un Puerto que enfrenta hacia arriba. Un host (Puerto que enfrenta hacia abajo) debe presentar una resistencia de pull-up (Rp) y poder suministrar VBUS. Esta placa no puede hacer ninguna de las dos cosas.

Conectar dos puertos que ambos declaren ser dispositivos y la negociación CC nunca completa, por lo que nunca se enumera nada. Ese es exactamente el `dev=NONE` observado. Es una propiedad fija del hardware, y ninguna cantidad de cambios de firmware altera las resistencias.

## Lo que tomaría una solución real

Ambas opciones están fuera de alcance para un repositorio de firmware, y ninguna es recomendada sobre simplemente usar un módulo con pines:

1. **Reparación del PCB** — eliminar R24/R26, añadir resistencias de pull-up Rp y una fuente de VBUS/conmutador.
2. **Una placa de USB Host externa** (MAX3421E), que se comunica con el ESP32 a través de SPI normal y evita por completo el puerto nativo. Nota: esto aún no habría terminado el trabajo: el puente CH340 del módulo no tiene un driver listo en `USB_Host_Shield_2.0`, por lo que también se necesitaría una clase de driver personalizada.

## Lo que hacer en su lugar

Usar un módulo PN532 con un header de pines accesible — cuestan lo mismo — y elegir el entorno de construcción correspondiente:

| Conexión | entorno | Leéderes |
|----------|---------|----------|
| HSU / UART | `esp32s3_hsu_b` | 2 (referencia, verificado en la mesa de pruebas) |
| SPI | `esp32s3` | 2 |
| I2C en `Wire1` | `esp32s3_i2c` | 1 (dirección fija 0x24) |

Ver [HARDWARE.md](HARDWARE_ES.md) para la conexión pin a pin de cada uno.

## Pie de página: el error que este desvío ocultó

Mientras se perseguía la ruta USB-host, la ruta HSU también reportaba no leéderes — lo que hacía que la historia de transporte pareciera peor de lo que era. La causa real allí era ajena y mundana: `setupRFID()` llamaba a `SPI.begin(47, 48, 41)` incondicionalmente, afirmando los exactos GPIOs que el entorno HSU rebindía a `Serial1`/`Serial2`. SPI obtuvo los pines primero, y las llamadas posteriores a `Serial.begin()` nunca tomaron el control de los enlaces de pines, y nada se transmitió. Esa llamada ahora está protegida solo para el transporte SPI, y HSU funciona con ambos leéderes.

Dos fallos separados que presentan el mismo síntoma vale la pena recordarlo: es por eso que "ningún leéder detectado" siempre debe comenzar con *"¿qué entorno se usó realmente para construir esto?"*

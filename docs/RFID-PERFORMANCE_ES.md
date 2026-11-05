# Rendimiento de lectura RFID: interferencia entre los dos lectores

Mediciones realizadas el 2026-10-28 con una unidad de banco en buen estado (ESP32-S3-Touch-LCD-3.5B, dos PN532 por HSU, firmware 3.12.0 más una compilación solo para medición) y una bobina con una etiqueta TigerTag NTAG en cada ala. Se midieron 64 combinaciones, con sondeo continuo de 1,2 s por combinación.

Cada combinación registra tanto los intentos como las lecturas correctas, porque las fases no realizan el mismo número de intentos: en las fases individuales se sondea un lector en vez de dos y la fase alterna ejecuta cuatro comandos adicionales por iteración. Solo se puede comparar entre fases la **proporción de lecturas correctas por intento**.

> **Estado.** El firmware ya incluye todo lo recomendado en la sección «Método de trabajo recomendado»: se apaga el campo del lector inactivo mientras lee el otro, el sondeo se realiza cada 100 ms con 2 reintentos, el valor predeterminado compilado es 7 y la autocalibración mide en régimen alterno. Se conserva la redacción de esa sección porque documenta lo que las mediciones justificaron antes de cambiar el código.

## Método

| Fase | Qué mide |
|-------|----------|
| `soloL` | Barrido del lector izquierdo de 0 a 15 con el **campo del lector derecho apagado**. Referencia sin interferencias para el lado izquierdo. |
| `soloR` | Barrido del lector derecho de 0 a 15 con el **campo del lector izquierdo apagado**. Lo mismo para el lado derecho. |
| `both` | Ambos lectores al mismo nivel, con los dos campos activos. **Este es el método que se distribuye actualmente.** |
| `alt` | Ambos lectores al mismo nivel, pero solo un campo está activo a la vez: se apaga mientras lee el lector vecino. |

Para activar o desactivar un campo se usa `RFConfiguration` CfgItem 0x01; se controla el campo de 13,56 MHz sin reiniciar el módulo. Ten en cuenta que `antennaOn()` y `antennaOff()` en el firmware existente **no** hacen esto: solo modifican un booleano. Por ello, en el firmware distribuido ambos campos permanecen encendidos.

## Resultados

Proporción de lecturas correctas por intento, por nivel de RF:

| nivel | soloL | soloR | both (I/D) | alt (I/D) | lecturas/s both | lecturas/s alt |
|------:|------:|------:|-----------:|----------:|-------------:|------------:|
| 0  |   0% |   0% |   0% / 0%  |   0% / 0%   |  0.0 |  0.0 |
| 1  |   0% |   0% |   0% / 0%  |   0% / 0%   |  0.0 |  0.0 |
| 2  | 100% | 100% |  85% / 92% |  80% / 100% | 19.2 | 15.0 |
| 3  |  42% | 100% |  58% / 83% | **100% / 100%** | 14.2 | **18.3** |
| 4  |  18% | 100% |  58% / 92% |  56% / 100% | 15.0 | 11.7 |
| 5  |  64% | 100% |  64% / 73% |  90% / 100% | 12.5 | 15.8 |
| 6  |  93% | 100% |  64% / 64% |  90% / 90%  | 11.7 | 15.0 |
| 7  |  93% | 100% |  83% / 50% | **100% / 100%** | 13.3 | **18.3** |
| 8  |  33% | 100% |  50% / 60% |  56% / 100% |  9.2 | 11.7 |
| 9  | 100% |  93% |  70% / 20% |  88% / 25%  |  7.5 |  7.5 |
| 10 | 100% |  75% |  80% / 20% | 100% / 56%  |  8.3 | 11.7 |
| 11 | 100% |  97% |  55% / 55% | 100% / 44% | 10.0 | 10.8 |
| 12 | 100% | 100% |  70% / 10% | 100% / 22% |  6.7 |  9.2 |
| 13 | 100% |   0% |  70% / 40% | 100% / 0%  |  9.2 |  6.7 |
| 14 | 100% |  33% |  20% / 70% | 100% / 0%  |  7.5 |  6.7 |
| 15 | 100% |  80% |  10% / 50% | 100% / 22% |  5.0 |  9.2 |

## Interpretación

**1. La antena enfrentada perjudica mucho la lectura.** Por separado, cada lector se acerca al 100 % en casi todo el rango útil. Con ambos campos activos, los mismos niveles bajan al 50–85 %, y el lector derecho cae al 10–20 % desde el nivel 9. En el nivel 12, el lector derecho pierde nueve de cada diez lecturas únicamente porque la otra antena está transmitiendo.

**2. Alternar los campos recupera casi todas las lecturas.** En el nivel 3, ambos lectores pasan del 58 %/83 % al **100 %/100 %**. En el nivel 7 pasan del 83 %/50 % al **100 %/100 %**. La alternancia reduce alrededor de un 15 % la tasa de intentos —cuatro comandos adicionales por iteración— y aun así logra mayor rendimiento bruto: 18,3 lecturas/s frente a 13,3 en el nivel 7, porque se desperdician menos intentos.

**3. Los mejores puntos de operación son la alternancia en nivel 3 o 7.** Son las únicas dos combinaciones de la tabla donde ambos lectores alcanzan el 100 %, y también las dos tasas más altas medidas. El valor predeterminado compilado era 3 y ahora es 7, lo que conserva margen sin acercarse al extremo de potencia excesiva.

**4. No superes el nivel 8.** El lector derecho también empeora con el otro campo apagado (en el nivel 13, la fase individual da 0 %), lo que indica que el módulo está sobreexcitado, no que haya interferencia. La alternancia no puede corregirlo.

**5. Ningún lector llegó a leer la etiqueta del otro.** En las 64 combinaciones y todos los niveles de potencia, el lector izquierdo solo vio `1D33DBDA0D1080` y el derecho solo `1D34DBDA0D1080`. No hubo lecturas cruzadas; por tanto, los recuentos son válidos y un ajuste de alta potencia no se beneficia artificialmente de la etiqueta lejana. (La misma comprobación en la unidad *defectuosa* sí mostró un UID en ambas cajas: otro síntoma de esa unidad, no del diseño.)

## Método de trabajo recomendado

1. **Apaga el campo del lector inactivo mientras lee el otro.** Es la mayor mejora de fiabilidad disponible y todavía no se aplica. Requiere `RFConfiguration` CfgItem 0x01; actualmente `antennaOn()` y `antennaOff()` no hacen nada, por lo que la función que debería silenciar el campo cuando la plataforma está vacía tampoco lo apaga realmente.
2. **Trabaja en nivel 3 o 7.** Evita el nivel 9 y superiores. (Se adoptó 7 como valor predeterminado compilado.)
3. **Vuelve a ejecutar la autocalibración alternando los campos.** Actualmente mide con ambos campos activos y optimiza un escenario distinto: eligió el nivel 4, donde la alternancia mide 56 % en el lector izquierdo.
4. **Evalúa el lector más débil.** Los datos aún lo justifican: una puntuación total podría elegir una combinación donde un lado solo alcance el 20 %.

## Salvedades

- Una unidad, una bobina, una geometría y 1,2 s por combinación. La columna individual izquierda es claramente variable (42 % en nivel 3, 18 % en nivel 4 y 100 % en nivel 9); no conviene sacar conclusiones de una sola combinación. La comparación entre fases es lo más fiable.
- La segunda unidad del banco tiene defectuoso el lector derecho y se excluyó: no responde de forma fiable a ningún comando PN532 y no puede leer UID de 4 bytes.

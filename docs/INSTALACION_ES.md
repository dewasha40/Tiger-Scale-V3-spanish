# Instalar TigerScale V3

Esta guía resume la instalación del firmware en la báscula TigerScale V3. Para la lista exacta de componentes y el montaje mecánico, consulta [HARDWARE.md](HARDWARE.md). La guía original en inglés sigue disponible en [INSTALLATION.md](INSTALLATION.md).

## Antes de empezar

- Placa Waveshare **ESP32-S3-Touch-LCD-3.5B** o **ESP32-S3-Touch-LCD-3.5**. Son variantes distintas: lee el texto impreso en la placa y selecciona la variante correcta.
- Cable USB que permita transferir datos.
- Plataforma con célula de carga de 5 kg y módulo HX711 montados y conectados.
- Lectores PN532 instalados según el cableado documentado.
- Navegador compatible con Web Serial para el instalador web, o PlatformIO para compilar desde el código.

## Instalación desde el navegador

1. Conecta la placa al ordenador por USB.
2. Abre el [instalador web de TigerScale](https://tigertag-project.github.io/Tiger-Scale-V3/) en Chrome o Edge.
3. Selecciona el modelo exacto de placa, **-3.5B** o **-3.5**.
4. Pulsa conectar y selecciona el puerto serie de la placa.
5. Sigue el asistente y espera a que termine la escritura. No desconectes el USB durante el proceso.
6. Reinicia la placa cuando el instalador lo indique.

El primer inicio puede tardar mientras se prepara la interfaz. Sigue el asistente de la pantalla para seleccionar idioma, red y opciones de la báscula.

## Compilar desde el código

1. Instala Git, Python y PlatformIO.
2. Clona este repositorio y ábrelo en VS Code con PlatformIO.
3. En `platformio.ini`, selecciona el entorno correspondiente a la variante y al transporte NFC del montaje.
4. Compila y carga el entorno seleccionado por USB.
5. Consulta el monitor serie si el arranque o la inicialización del hardware falla.

No elijas el entorno solo por el nombre: coteja la variante de pantalla y el transporte NFC con [HARDWARE.md](HARDWARE.md) y los entornos del archivo `platformio.ini`.

## Primera configuración y calibración

- Selecciona español en los ajustes de idioma; en este fork es el idioma inicial para instalaciones nuevas.
- Configura Wi-Fi si quieres sincronización. El pesaje y las funciones locales pueden utilizarse sin conexión.
- Calibra la célula de carga siguiendo el asistente de la pantalla y una masa conocida. No uses lecturas de ejemplo como calibración.
- Comprueba la tara sin una bobina y después coloca una bobina para validar el signo y la estabilidad del peso.
- Prueba la lectura de las etiquetas con los lectores instalados. La orientación y distancia dependen de la caja y del cableado.

## Actualizaciones

Usa el instalador para actualizar la placa y selecciona siempre la misma variante física. Antes de actualizar, revisa las notas de la versión. No borres el almacenamiento si quieres conservar la configuración guardada.

## Simulador de PC

El perfil de simulación de esta entrega está en [`../simulador/tigerscale_v3.sim.json`](../simulador/tigerscale_v3.sim.json). Copia ese archivo a la carpeta `perfiles` del ESP32 Simulador que ya contiene el proyecto `tigerscale_v3` y reinicia el simulador. Selecciona TigerScale V3 y pulsa Play. La maqueta de PC representa la pantalla 480 × 320 con datos ficticios; no ejecuta este firmware ni valida el hardware.

Capturas de las tres páginas simuladas: [báscula](img/interfaz-bascula-simulada.png), [bobina](img/interfaz-bobina-simulada.png) y [ajustes](img/interfaz-ajustes-simulada.png).

## Problemas habituales

- **No aparece el puerto:** prueba otro cable USB de datos, otro puerto del PC y el controlador indicado por Waveshare.
- **Pantalla en blanco o variante incorrecta:** revisa la serigrafía y vuelve a instalar la compilación de la placa correcta.
- **Peso inestable o incorrecto:** revisa la fijación mecánica, los tornillos, el HX711 y las conexiones de la célula; vuelve a calibrar con una masa conocida.
- **No lee etiquetas:** comprueba el transporte seleccionado, el cableado y la posición de los lectores PN532.
- **No sincroniza:** valida Wi-Fi y credenciales; la falta de nube no debería impedir el pesaje local.

Para más diagnósticos, consulta [TROUBLESHOOTING.md](TROUBLESHOOTING.md) (original en inglés) y [SERIAL-PORT.md](SERIAL-PORT.md).

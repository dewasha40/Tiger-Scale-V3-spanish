# Obtener el puerto COM para que aparezca para flashar el TigerScale V3

El TigerScale V3 utiliza una **placa ESP32-S3**. Para flashar el firmware, tu computadora necesita ver la placa como un **puerto COM / puerto serie**.

En la mayoría de las computadoras recientes, este puerto aparece **por sí solo** cuando conectas la placa.
Si no aparece ningún puerto en el flasher, falta un pequeño controlador: sigue la sección para tu sistema a continuación.

> **Antes de comenzar**
> - Utiliza un **cable USB de datos** real (algunos cables solo cargan y no transmiten datos).

---

## macOS — nada que instalar

No se necesita controlador.
Conecta la placa, abre el flasher y la placa aparece directamente en la lista. Selecciona y comienza el flash.

Si no aparece: cambia el cable USB y prueba de nuevo.

---

## Windows — instala el controlador (si no aparece el puerto COM)

Si el flasher no muestra ningún puerto, el controlador USB de Espressif está faltando. Aquí está cómo
instalarlo, paso a paso:

1. **Descarga la herramienta** — haz clic en este enlace, descarga `idf-env.exe` directamente
   desde el servidor propio de Espressif (por defecto, se guarda en tu carpeta **Descargas**
   ):
   **[https://dl.espressif.com/dl/idf-env/idf-exe](https://dl.espressif.com/dl/idf-env/idf-env.exe)**
2. **Abre PowerShell como administrador** — haz clic en el menú **Iniciar**, escribe
   `powershell`, haz clic derecho en **Windows PowerShell** y elige **Ejecutar como
   administrador**.
3. **Copia esta línea única, pégala en PowerShell y presiona Enter:**

   ```powershell
   cd $env:USERPROFILE\Downloads; .\idf-env.exe driver install --espressif
   ```

   > Si guardaste el archivo en otro lugar que no sea Descargas, reemplaza solo
   > `$env:USERPROFILE\Downloads` con esa carpeta — por ejemplo
   > `C:\Users\tuusuario\Escritorio`. El resto de la línea permanece igual.

4. Espera a que termine (unos segundos).
5. **Desconecta y vuelve a conectar la placa.**
6. Vuelve a abrir el flasher: el puerto (por ejemplo, `USB JTAG/serial debug unit (COM3)`) debería aparecer ahora.

> Descargar la herramienta tú mismo, en lugar de pegar un comando de una línea que
> descarga y ejecuta un ejecutable con derechos de administrador, significa que puedes ver
> lo que estás a punto de ejecutar y de dónde proviene. Es un clic adicional.

✓ Selecciona este puerto y comienza el flash.

---

## Linux — otorga acceso al puerto serie

El puerto suele aparecer por sí solo (`/dev/ttyACM0`), pero tu usuario necesita permisos para acceder a él.

1. Abre un **terminal**.
2. Añade tu usuario al grupo que gestiona los puertos serie:

   ```bash
   sudo usermod -aG dialout $USER
   ```

3. **Cierra la sesión y vuelve a iniciarla** (o reinicia) para que surta efecto.
4. Vuelve a conectar la placa y abre nuevamente el flasher.

> Consejo: si usas el flasher **en un navegador** (Web Serial), utiliza **Google Chrome** o **Microsoft Edge**; Firefox no soporta esta función.

---

## Aún no funciona

- **Cambia el cable USB** (problema #1: un cable de carga solo).
- Intenta **otro puerto USB** en la computadora (preferiblemente un puerto directo, no un hub).
- En Windows, abre **Administrador de dispositivos**: si ves un dispositivo con un signo de exclamación amarillo, el controlador está faltando → vuelve a hacer la sección de Windows anterior.
- Para forzar el modo de flash: mantén presionado el botón **BOOT**, presiona **RESET** una vez, luego suelta **BOOT** y prueba de nuevo.

# Hacer que aparezca el puerto COM para instalar TigerScale V3

TigerScale V3 utiliza un chip **ESP32-S3**. Para cargar el firmware, el ordenador debe detectar la placa como un **puerto COM/puerto serie**.

En la mayoría de los ordenadores recientes, el puerto aparece **automáticamente** al conectar la placa. Si el instalador no muestra ningún puerto, puede faltar un controlador; sigue las instrucciones de tu sistema operativo.

> **Antes de empezar**
> - Usa un cable USB que transmita **datos** (algunos cables solo sirven para cargar y no transmiten datos).

---

## macOS — no hay que instalar nada

No se necesita ningún controlador. Conecta la placa, abre el instalador y aparecerá directamente en la lista. Selecciónala y comienza la carga.

Si no aparece, cambia el cable USB y vuelve a intentarlo.

---

## Windows — instala el controlador si no aparece el puerto COM

Si el instalador no muestra ningún puerto, falta el controlador USB de Espressif. Instálalo así:

1. **Descarga la herramienta** desde el servidor de Espressif. De forma predeterminada, se guardará en la carpeta **Descargas**:
   **[https://dl.espressif.com/dl/idf-env/idf-env.exe](https://dl.espressif.com/dl/idf-env/idf-env.exe)**
2. **Abre PowerShell como administrador**: abre el menú **Inicio**, escribe `powershell`, haz clic derecho en **Windows PowerShell** y elige **Ejecutar como administrador**.
3. **Copia esta línea, pégala en PowerShell y pulsa Intro:**

   ```powershell
   cd $env:USERPROFILE\Downloads; .\idf-env.exe driver install --espressif
   ```

   > ¿Guardaste el archivo en otra carpeta? Sustituye solo `$env:USERPROFILE\Downloads` por esa ruta; por ejemplo, `C:\Users\tu nombre\Desktop`. El resto de la línea no cambia.

4. Espera unos segundos a que termine.
5. **Desconecta la placa y vuelve a conectarla.**
6. Abre de nuevo el instalador. Ahora debería aparecer el puerto (por ejemplo, `USB JTAG/serial debug unit (COM3)`).

> Descargar la herramienta tú mismo, en vez de pegar un comando que descarga y ejecuta un archivo con permisos de administrador, te permite ver qué vas a ejecutar y de dónde procede. Solo requiere un clic adicional.

✓ Selecciona ese puerto y empieza la carga.

---

## Linux — concede acceso al puerto serie

El puerto suele aparecer automáticamente (`/dev/ttyACM0`), pero tu usuario necesita permiso para acceder a él.

1. Abre una **terminal**.
2. Añade tu usuario al grupo que administra los puertos serie:

   ```bash
   sudo usermod -aG dialout $USER
   ```

3. **Cierra la sesión y vuelve a iniciarla** (o reinicia el ordenador) para aplicar el cambio.
4. Vuelve a conectar la placa y abre otra vez el instalador.

> Consejo: si usas el instalador **en un navegador** (Web Serial), elige **Google Chrome** o **Microsoft Edge**; Firefox no admite esta función.

---

## Si sigue sin funcionar

- **Cambia el cable USB** (la causa más frecuente es un cable que solo carga).
- Prueba **otro puerto USB** del ordenador, preferiblemente uno directo y no un concentrador.
- En Windows, abre el **Administrador de dispositivos**: si hay un dispositivo con un signo de exclamación amarillo, falta el controlador; repite los pasos de Windows.
- Para forzar el modo de carga, mantén pulsado **BOOT**, pulsa **RESET** una vez, suelta **BOOT** y vuelve a intentarlo.

# Logotipos de marcas de socios

Los fabricantes de filamento que envían TigerTag, tal como se usan en el archivo `README.md` raíz.

Estos provienen de **TigerTag-Hub**, que es donde la pared de socios de tigersystem.io obtiene estos logotipos — por lo tanto, se mantienen consistentes con el sitio web en lugar de desviarse hacia una copia privada. La lista canónica de socios, con la URL oficial de cada marca, se encuentra en `lib/partners.ts` en ese repositorio. No adivines manualmente una URL de marca; toma la desde allí.

Un conjunto anterior de logotipos PNG existía bajo `data/www/img/` (dibujados para la interfaz de usuario dark LVGL del dispositivo), junto con encabezados generados en RGB565, para una función "mostrar el logotipo de la marca en pantalla" que nunca se conectó. Estos eran los activos incorrectos para un README — varios eran casi invisibles en fondo claro y `r3d_logo.png` resultó ser completamente vacío — por lo que se abandonaron a favor de este conjunto.

## Formato

Vectorial, en los colores propios de cada marca, sobre un lienzo compartido de 283x283 con relleno. Esa normalización cuadrada es la razón por la cual el README puede asignarles todos un `height` y obtener una fila alineada, a pesar de las diferencias extremas en las proporciones de los logotipos subyacentes.

Reemplazaron un conjunto anterior de PNGs dibujados para la interfaz de usuario dark LVGL del dispositivo, que era la herramienta incorrecta para un README: varios eran casi invisibles en fondo claro y `r3d_logo.png` resultó ser completamente vacío.

El archivo de trabajo de Illustrator de donde se exportan estos es deliberadamente no incluido — pesa 5.8 MB y nadie que consuma este repositorio necesita la fuente.

## Usarlos

Marcas registradas de sus respectivos dueños, incluidas para registrar quién envía el protocolo. No se implica ningún aval en ninguna dirección.

Dos reglas editoriales heredadas de `docs/philosophy/partner-announcements.md` en TigerTag-Hub, merecen repetirse porque son fáciles de equivocarse:

- **Cita el alcance de cada fabricante tal como lo establecieron.** La integración de eSun es un programa piloto en el mercado francés; no la redondees hasta disponibilidad mundial. La de Rosa3D es el 100 % de producción de bobinas de 1 kg. Estas son sus declaraciones, hechas en sus propios canales, y deben atribuirse en lugar de afirmarse.
- **"Protocolo abierto" es nuestra redacción.** Algunos socios escriben "open source" sobre TigerTag en sus propios posts. Eso puede citarse como suyo; no es lo que decimos nosotros en nuestra propia copia.

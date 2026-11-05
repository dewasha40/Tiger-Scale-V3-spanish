# Sincronización en la nube y privacidad

Lo que envía la báscula, adónde va, y qué se almacena en el dispositivo. Escrito de forma que cualquiera pueda decidir si quiere las características de la nube — el firmware es completamente utilizable sin ellas.

## La báscula funciona sin conexión

La sincronización en la nube es opcional. Sin iniciar sesión, la báscula sigue pesando, sigue leyendo etiquetas TigerTag, sigue identificando marca y material (desde una base de datos caché en su propio flash) y sigue sirviendo su interfaz web en tu LAN. Lo que pierdes es la sincronización de inventario entre dispositivos.

Las búsquedas de marca y material están diseñadas **localmente**: `data/id_brand.json` y `data/id_material.json` están almacenados en el sistema de archivos del dispositivo, por lo que identificar una etiqueta no requiere un viaje de red y funciona incluso sin conexión a internet. Se refrescan desde el repositorio público
[TigerTag-RFID-Guide](https://github.com/TigerTag-Project/TigerTag-RFID-Guide)
máximo una vez cada 24 h.

## Iniciar sesión

La autenticación pasa por Firebase Auth para el proyecto `tigertag-connect`. Se admiten tanto el inicio de sesión con correo/contraseña como con Google, desde la pantalla táctil o la interfaz web.

**Lo que almacena el dispositivo en su propio flash NVS:** un token de refresco de Firebase, el ID de usuario y las credenciales de WiFi. Un token de refresco es suficiente para obtener nuevos tokens de acceso, por lo que trata a un dispositivo configurado como poseedor de una credencial — `Configuración → Nube → Cerrar sesión` lo borra (lo mismo que hace el endpoint
`POST /api/firebase/logout`).

**Lo que nunca almacena:** la contraseña de tu cuenta. El inicio de sesión la intercambia por tokens y no se escribe en el flash.

### Acerca de la clave de API en el código fuente

`TIGERTAG_FIREBASE_WEB_API_KEY` en §1 es una cadena codificada en el código, y eso es correcto, no un error. Una clave de API *Web de Firebase* es un identificador público del proyecto que cada aplicación cliente de Firebase incluye — el mismo valor se sirve públicamente por Firebase Hosting en `/__/firebase/init.json`. Identifica el proyecto; no autoriza nada por sí sola. El control de acceso se enfoque mediante las Reglas de seguridad de Firebase contra el usuario conectado.

## Lo que se envía

En cada pesaje completado, y en un latido periódico, el dispositivo escribe en Firestore bajo `users/{uid}/scales/{mac}`:

| Campo | ¿Por qué? |
|-------|----------|
| UID de las etiquetas de la bobina en la plataforma | identifica cuál bobina es esta |
| Peso neto, peso bruto, peso del contenedor | la medición real |
| Versión del firmware | para que la aplicación te avise de una actualización |
| Fuerza del señal WiFi, dirección IP | diagnósticos mostrados en la aplicación |
| Factor de calibración | para que un dispositivo reemplazado pueda restaurarse |
| Marca de tiempo del último latido | indicador en línea/fuera de línea |
| Nivel de batería, estado de carga, fuente de alimentación | mostrado en la aplicación; nulo cuando no hay batería instalada |
| Estado de la pantalla | para que el modo stand-by no se confunda con una báscula que se haya desconectado |

El latido envía solo los campos que han cambiado desde el último, con una instantánea completa al conectar y periódicamente después. Cada campo, su tipo exacto y qué significa cada valor: **[docs/TELEMETRY.md](TELEMETRY_ES.md)** — la referencia para cualquier cosa que lea una báscula en lugar de poseerla.

Las marcas de tiempo se establecen en el servidor, no en el dispositivo — el ESP32 no tiene un reloj con batería y no necesita NTP para esto.

## Comandos remotos

El dispositivo consulta una cola de comandos en Firestore bajo
`users/{uid}/scales/{mac}/commands/`, que es cómo la aplicación puede desencadenar un tare remoto, un reinicio, una reinicialización de fábrica o una actualización OTA. El estado y progreso de cada comando se escriben de vuelta para que la aplicación los muestre.

Los comandos se leen desde **tu propio** documento de usuario, por lo que esto no es un canal que nadie más pueda alcanzar.

## La API local

El dispositivo también expone una API HTTP en tu LAN — esta es lo que su propia interfaz web utiliza. Notablemente es **no autenticada**, lo cual es una simplificación deliberada para un dispositivo en una red doméstica, pero vale la pena saber:

- Cualquiera en tu red puede leer el estado de la báscula y desencadenar un tare.
- `GET /api/logs` devuelve las últimas ~80 líneas de registro.
- `POST /api/firebase/logout` borra la sesión almacenada.
- `POST /api/reset-wifi` borra las credenciales de WiFi almacenadas.

Si tu red no la confías completamente, coloca el dispositivo en una VLAN de invitados.

## Entregar un dispositivo a alguien más

Antes de pasar un dispositivo, limpia ambas credenciales almacenadas:

- **Configuración → Nube → Cerrar sesión** (o `POST /api/firebase/logout`)
- **Configuración → WiFi → Olvidar** (o `POST /api/reset-wifi`)

O borra todo, incluida la calibración, con
`bash scripts/flash.sh --erase`.

Un reflejo simple no borra estos datos — el NVS se preserva intencionalmente para que las actualizaciones de firmware no te cuesten tu configuración.

## Ejecutar contra tu propio proyecto Firebase

El ID del proyecto Firebase y la clave de API son constantes en tiempo de compilación en §1, y las rutas de documentos Firestore se construyen en §11. Apuntar el firmware a un proyecto diferente significa cambiar esas constantes y recrear la misma estructura de documento más las reglas de seguridad coincidentes.

Esto no es una ruta documentada, soportada hoy — nadie lo ha hecho — por lo que si lo intentas, espera leer §11 en lugar de seguir una receta. Una solicitud de pull añadiendo una guía verificada sería muy bienvenida.

# Política de seguridad

## Denuncia de una vulnerabilidad

Por favor, **no abra un problema público** para un problema de seguridad.

Denúncialo a través de
[GitHub's private vulnerability reporting](https://github.com/TigerTag-Project/Tiger-Scale-V3/security/advisories/new),
o por correo electrónico a **benoit@atome3d.com**.

Detalles útiles: lo que encontraste, cómo reproducirlo, la versión del firmware, y
lo que un atacante podría hacer realmente con ello. Te darás cuenta de que se ha recibido la denuncia, y
crédito en las notas de lanzamiento si lo deseas.

## Versiones admitidas

La última versión liberada. Este es un proyecto de hardware amatorista — no hay ramas de soporte a largo plazo.

## Alcance

**En el alcance** — este firmware, su API HTTP local y la interfaz web, su ruta de actualización OTA, y
cómo almacena credenciales en el dispositivo.

**Fuera del alcance** — el backend de TigerTag en la nube, las aplicaciones móviles y de escritorio,
y las etiquetas NFC en sí. Cada una vive en su propio repositorio.

## Debilidades conocidas y aceptadas

Estas son decisiones de diseño, no errores no descubiertos. Denunciarlos es aceptable, pero
no te estarás informando de algo nuevo.

- **La API HTTP local no está autenticada.** Cualquiera en la misma red puede leer
  el estado de la báscula, leer el anillo de registros (`GET /api/logs`), desencadenar un tare, y
  limpiar la sesión WiFi o Firebase almacenada. Esto es una simplificación deliberada para un dispositivo en una red doméstica. Si no confías completamente en tu red, coloca el
  dispositivo en una VLAN de invitados.
- **Un token de refresco de Firebase se almacena en la memoria flash NVS del dispositivo.** Es lo que
  permite que la báscula siga sincronizándose sin que debas iniciar sesión nuevamente. Trata a un dispositivo provisionado como algo que contiene una credencial: desloguea (Configuración → Nube, o
  `POST /api/firebase/logout`) antes de pasar uno a otra persona. Un refrescado simple no
  lo borra — la memoria flash NVS se preserva deliberadamente para que las actualizaciones no te hagan perder la configuración.
- **OTA verifica un SHA-256 pero usa `setInsecure()` para TLS.** La validación del certificado se salta; la integridad descansa en el hash. Esto significa que la fuente del hash es lo que importa — un atacante en la red que pueda servir tanto el manifiesto como el binario podría servir un par coincidente. Anclarse a una CA sería una mejora genuina y un pull request bienvenido.
- **`TIGERTAG_FIREBASE_WEB_API_KEY` está codificado en la fuente, deliberadamente.** Una clave de API web de Firebase es un identificador público del proyecto que cada aplicación cliente de Firebase incluye — el mismo valor se sirve públicamente por Firebase Hosting en `/__/firebase/init.json`. Identifica el proyecto y autoriza nada por sí sola; el control de acceso se enfoque mediante Reglas de Seguridad contra el usuario conectado. Esto no es una secreto filtrado.

Lo que verdaderamente nos gustaría escuchar: cualquier cosa que permita a alguien fuera de tu
red alcanzar el dispositivo, leer los datos de otro usuario, o instalar firmware arbitrario
en una unidad.

## Para colaboradores

Nunca comites credenciales de WiFi, tokens o direcciones de red personales. Si necesitas
una dirección del dispositivo en un script, toma como argumento — `scripts/watch_logs.py`
hace exactamente eso, después de que una versión anterior tuviera la IP LAN del desarrollador incrustada.

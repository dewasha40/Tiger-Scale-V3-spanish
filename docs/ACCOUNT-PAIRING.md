# Vincular una cuenta TigerTag a la báscula

Cómo un TigerScale se autentifica cuando su propietario no tiene contraseña para introducir.

## El problema que resuelve

El firmware se autentifica mediante `accounts:signInWithPassword` — un correo electrónico y una contraseña introducidos en la pantalla táctil. Cualquiera que haya creado su cuenta TigerTag **con Google no tiene contraseña**, por lo que no hay nada que introducir. Hoy en día, esos propietarios deben dejar la báscula y usar una interfaz web en su lugar, y la propia pantalla de inicio de sesión de la báscula es un callejón sin salida para ellos.

También explica un error observado en campo: cuando un refresco de token falla, el firmware cae de vuelta a una autenticación completa y los propietarios de cuentas de Google reciben `MISSING_PASSWORD`, porque la contraseña a la que cae nunca existió.

## ¿Por qué no el flujo estándar OAuth device

[RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628) existe exactamente para este tipo de dispositivo, y Google lo implementa. Fue rechazado después de leer la especificación en lugar de por gusto:

- La respuesta device-code de Google **no lleva `verification_url_complete`**. Devuelve `https://www.google.com/device` y un código de ocho caracteres que el propietario debe volver a introducir. Un código QR solo puede llevar la URL básica, por lo que la mejor experiencia posible es "escanear, luego copiar un código desde una pantalla a 40 cm de distancia".
- El **polling requiere un `client_secret`**, que este repositorio no debe llevar (ver la regla de Secretos en `CLAUDE.md`). Podría provisionarse en tiempo de ejecución, pero eso es maquinaria en servicio de un flujo que es peor de todos modos.
- Se autentifica solo contra Google. Apple, o cualquier proveedor agregado más tarde, necesitarían cada uno su propio camino.

El emparejamiento a través de la nube de TigerTag no tiene ninguno de esos atributos, porque controlamos ambos extremos: **la URL en el código QR ya lleva el código.** Escanear, aprobar, terminado.

## El flujo

```
  scale                        tigersystem.io                     owner's phone
    |                                |                                  |
    |-- POST /api/device/pair/start -|                                  |
    |<- code + verify_url + poll_token                                  |
    |                                |                                  |
  [ shows a QR of verify_url ]       |          <-- scans the QR -------|
    |                                |<- GET /pair?c=CODE --------------|
    |                                |   (signs in with Google if needed)
    |                                |<- POST approve ------------------|
    |                                |   [ mints a Firebase custom token ]
    |-- POST /api/device/pair/poll ->|                                  |
    |<- status: approved, custom_token                                  |
    |                                |                                  |
    |-- accounts:signInWithCustomToken (Identity Toolkit) -------------->|
    |<- idToken + refreshToken                                          |
```

El propietario está frente a la báscula, por lo que escanea un QR que está físicamente frente a él, y aprueba en un teléfono donde normalmente ya está autenticado.

## Los dos puntos de acceso

### `POST /api/device/pair/start`

No autenticado — el dispositivo no tiene identidad aún. Eso es lo que el límite de tasa y la expiración corta son para.

```json
{ "device": "tigerscale-A1B2", "model": "TigerScale V3", "fw": "3.1.3" }
```

```json
{
  "code":        "K7QF-3M2P",
  "verify_url":  "https://tigersystem.io/pair?c=K7QF3M2P",
  "poll_token":  "<32 bytes, base64url>",
  "expires_in":  600,
  "interval":    5
}
```

**`code` y `poll_token` son secretos diferentes con intención.** El código es corto porque un humano puede tener que leerlo en voz alta o introducirlo; el token de poll es largo porque quien lo posea recibe la credencial. Si un valor hiciera ambas tareas, cualquiera que mirara la pantalla podría pollar el token desde la otra habitación.

### `POST /api/device/pair/poll`

```json
{ "poll_token": "<the value from start>" }
```

| Respuesta | Significado |
|----------|---------|
| `{"status":"pending"}` | Nadie ha aprobado aún. Vuelva a pollar después de `interval`. |
| `{"status":"approved","custom_token":"<jwt>","email":"…"}` | Inicie sesión con él. Uso único — el registro se consume. |
| `{"status":"denied"}` | El propietario rechazó. Deténgase y diga así en la pantalla. |
| `{"status":"expired"}` | Más antiguo que `expires_in`. Empiece de nuevo con un código fresco. |

El dispositivo luego llama a Identity Toolkit como lo hace hoy, pero con `accounts:signInWithCustomToken` en lugar de `…WithPassword`, y almacena el token de refresco devuelto en NVS exactamente como ahora.

## Lo que debe mostrar la página de aprobación

La página alcanzada por el QR debe nombrar **cuál** báscula está pidiendo — su nombre de dispositivo y los últimos cuatro de su MAC — y la cuenta a la que se vincularía. El emparejamiento de dispositivos es de otro modo vulnerable a ser entregado a alguien más: el riesgo es pequeño aquí porque el propietario está mirando la pantalla física de donde vino el QR, pero una página que diga solo "aprobado?" entrena a la gente para aprobar cualquier cosa.

## Reglas para el lado de la nube

- Límite de tasa `start` por IP y `poll` por token; rechace un poll más rápido que `interval` con el mismo back-off que usa el flujo OAuth device.
- Los códigos son de uso único y caducan en diez minutos.
- Almacene el registro pendiente claveado por un hash de `poll_token`, no el token mismo.
- Genere el token personalizado con `firebase-admin`, que `TigerTag_Hub` ya depende (`lib/firebase/admin.ts`).

## Lo que cambia en el firmware

- `LV_USE_QRCODE` es `0` en `include/lv_conf.h`. LVGL v8 ya incluye el widget; solo necesita activarse.
- Una pantalla bajo Configuración → Nube que ejecuta el flujo y dibuja el QR, con el código en texto debajo para cualquiera cuyo teléfono no escaneará.
- `signInWithCustomToken` junto con la ruta de contraseña existente. La ruta de contraseña permanece: hay muchos propietarios que sí tienen una.
- `isFirebaseConfigured()` **ya acepta un token de refresco almacenado por su cuenta — su comentario anticipa "Autenticación de Google o cualquier proveedor mediante el puente OAuth".** Una báscula emparejada se verá configurada sin cambios necesarios, lo cual es un menos error para cometer.
- Cuando un refresco falla y no hay contraseña, diga "inicie sesión de nuevo" y ofrezca el QR — no intente una autenticación con contraseña que solo puede devolver `MISSING_PASSWORD`.

# Revisión UI/UX táctil — panel 3.5" (v3.1.3, banc via live view)

Fecha : 2026-08-08 · Perímetro : interfaz embarcada LVGL (pantalla principal, todos los subpantallas de Configuración, flujo de pesaje con operador en el banc) · Método : live view `http://<ip>/live` (taps únicamente), cruzamiento de código en lectura solo. Capturas : carpeta de sesión fuera de repositorio, referenciadas aquí por nombre de archivo. Convención de anotación : después de la acción, marcar cada constat **fixed / deferred / rejected**.

## Resumen ejecutivo

La interfaz es coherente y legible (bandeau de retroalimentación sistemática, tarjetas, barra de desplazamiento, valores mostrados directamente en la lista Configuración) y el flujo nominal de pesaje funciona rápido (~6 s detección, ~10 s datos en la nube, re-pesaje instantáneo gracias al caché). Los dos verdaderos vacíos son vacíos de *retroalimentación* : un objeto sin etiqueta no provoca ningún mensaje, y el envío a la nube exitoso (o abandonado) nunca se confirma en la pantalla. El resto es de la terminación a alto rendimiento : desplazamiento de la lista Configuración perdido en cada retorno, regeneración del código LAN sin confirmación, etiquetas no traducidas o truncadas.

## Estado (2026-08-08, después de la acción)

Los 10 quick wins están tratados : 9 **fixed**, 1 **rejected** (n° 3, ver su línea). Todo lo demás — suplemento Latin-1 Montserrat (causa raíz de los acentos), reforma de la pantalla Cuenta (maqueta primero), y las capturas por pantalla no reiteradas en los quick wins — es **deferred** : real, no cuestionado, para una pasada posterior.

## Quick wins

Clasificados por impacto decreciente ; cada ítem ≈ una hora o menos salvo indicación.

| # | Constat (captura) | Recomendación | Esfuerzo |
|---|---|---|---|
| 1 | Objeto sin etiqueta : peso mostrado pero el badge bloqueado en « Listo », **ningún mensaje incluso después de 15 s** (`objet-sans-tag-288g-toujours-rien`). Un usuario con una bobina no etiquetada no sabe ni qué sucede, ni qué hacer. §21 `handleWeighWorkflow` | Mantener « ¡Sincronizando! » ≥ 2 s antes de « Retirar material ». En caso de abandono (retracción antes de estabilización), badge « Pesaje anulado — no enviado » 2 s ; pasar el badge a « Pesaje... » tan pronto como el peso aumente. **fixed** (« Pesaje... » desde la apertura de sesión ; SENDING/SYNCED/ERROR priorizados sobre « Retirar material » ; « Pesaje anulado » 2 s si UID leído) | S |
| 2 | Confirmación de envío invisible : los estados `WEIGHING`/`SENDING`/`SYNCED` existen (.ino:12760–12767) pero el badge observado pasa de « Listo » (durante toda la subida de peso, `pesee-montee-827g-badge-pret`) directamente a « Retirar material ». Pesaje interrumpido = abandono **silencioso** (confirmado operador : nada en la pantalla, peso no enviado). | Mantener « ¡Sincronizando! » ≥ 2 s antes de « Retirar material » ; en caso de abandono (retracción antes de estabilización), badge « Pesaje anulado — no enviado » 2 s ; pasar el badge a « Pesaje... » tan pronto como el peso aumente. **fixed** (« Pesaje... » desde la apertura de sesión ; SENDING/SYNCED/ERROR priorizados sobre « Retirar material » ; « Pesaje anulado » 2 s si UID leído) | S |
| 3 | El « 0.0 » encima de TARE es una etiqueta estática nunca actualizada (.ino:12690) — parece una dato vivo pero miente tan pronto como se realiza una tara. | Mostrar el offset de tara real, o eliminar el número y solo mantener « TARE ». **rejected** — el « 0.0 » estático es la iconografía del botón cero de una báscula de cocina, deseado tal cual | S |
| 4 | La lista Configuración vuelve al principio cada vez que se vuelve a un subpantalla (`runSettingsMenu` reconstruye todo, .ino:5540+). Prueba involuntaria : un tap en la posición memorizada de LAN abrió Calibración (`scroll-reset-mistap-calibrage`). Alcanzar Alergia = 4 taps ▼ cada vez. | Memorizar el offset de desplazamiento (estático) y restaurarlo después de la reconstrucción. **fixed** (offset conservado durante el menú, restablecido a cero al salir) | S |
| 5 | LAN : el botón de regeneración del código (38×38 px, .ino:4642) se adyunta al código y **actúa sin confirmación** (.ino:4685–4690) — un tap accidental corta todos los viewers en curso. | Confirmación `lvglAskYesNo` (contexto bucle bloqueante, no un callback → autorizado) + objetivo llevado a ≥ 44 px. | S |
| 6 | WiFi : la red actualmente conectada no está **ni marcada ni fijada** en la lista (`wifi-liste-initiale`), y el orden cambia cada escaneo (`wifi-liste-apres-rescan`). El selector Idioma ya tiene el patrón ✓ (borde azul + marca, `langue-francais-coche`). | Reutilizar este patrón : SSID conectado fijado en cabeza con marca verde. **fixed** (fijado en cabeza, borde destacado + marca verde) | S/M |
| 7 | « Son: » codificado en duro (.ino:3717) : la pantalla Volumen permanece en francés cuando el dispositivo está en inglés (`volume-son-non-traduit-mode-EN`). | Nueva clave i18n (8 idiomas) + `bash scripts/check-i18n.sh`. | S |
| 8 | Pantalla RFID : la línea se llama « RFID » (.ino:5794, chevron sin valor, línea desequilibrada) pero la pantalla se titula « MATERIEL » (I18N_HARDWARE, .ino:4130) ; « PUISSANCE RFID : 3 » sin escala (3 de cuántos ?) ; TEST falla en « PN532-x no ack » mientras que el requisito « plataforma vacía » solo está escrito en comentario (.ino:10561). | Mismo etiqueta línea/título ; mostrar « 3/5 » ; antes del test, si peso > 0 : « Retire todo objeto de la plataforma » en lugar de lanzar un test que fallará en jerga. **fixed** (título « RFID » en ambos lados — la etiqueta universal en lugar de MATERIEL ; « 3/4 » con su escala ; « PUISSANCE RFID » pasa también por i18n ; el TEST finalmente fue **eliminado por completo** — nunca pasó al banc, y un diagnóstico que solo sabe fallar enseña principalmente al cliente que su material sería roto) | S |
| 9 | Nombres marca/materia troncados en duro a 10 caracteres sin elipsis (.ino:12827–12828) : « PLA High Speed » → « PLA High S » (`pesee-donnees-cloud-completes`). | `LV_LABEL_LONG_DOT` (o desplazamiento circular) en lugar de `substring(0,10)`. **fixed** (LONG_DOT \+ max_width, camino LVGL ; la truncación del camino raw-gfx heredado sigue) | S |
| 10 | Apostrofes incoherentes : « Código d'accès » (con) vs « L ecran reste allume », « minutes d inactivite » (i18n.h:486 y vecinas, `veille`) — la fuente posee el apostrofe. | Uniformizar todas las cadenas FR con apostrofe. **fixed** (las dos cadenas fallidas) | S |

## Constats por pantalla

### Pantalla principal (`main-0g`, `pesee-donnees-cloud-completes`)
**Funciona bien** : jerarquía clara (peso enorme a la izquierda, meta a la derecha), badge de estado colorido, TARE 2/3 + Parámetros 1/3 con grandes objetivos (~285×92 y 143×92 px), iconos WiFi/cuenta verdes en el bandeau.
**Punto débil** : los iconos casa/flecha con « -- » son cripticos antes de la primera pesada (ninguna leyenda ; es rack + ubicación, lo descubre solo al posar una bobina). El « 0.0 » del botón TARE (quick win 3). « Retirar material » como estado terminal es ambiguo : se lee como un error mientras que significa « pesaje terminado » — preferir « ¡Sincronizar — retire la bobina » (ver quick win 2).

### Configuración (`settings-list-haut`, `settings-list-bas`)
**Funciona bien** : cada línea muestra su valor actual (SSID, cuenta, 60 %, 406.00, Francés, v3.1.3, Sí/No) — la lista responde a la mayoría de las preguntas sin abrir nada. Iconos coloreados por estado (verde conectado / rojo desconectado).
**Punto débil** : desplazamiento perdido en cada retorno (quick win 4) ; línea RFID sola sin valor, chevron flotante (quick win 8) ; « Calibrar : 406.00 » expone un factor interno sin unidad — un usuario esperaría más bien la fecha del último calibrado o nada ; barra ▲/▼ ~39 px de ancho (debajo del objetivo 44 px) y sin estado desactivado en el borde (un tap al final de la lista produce ningún feedback).

### WiFi (`wifi-liste-initiale`, `wifi-recherche-spinner`, `wifi-ip-mac`)
**Funciona bien** : spinner « Búsqueda... » durante el re-scan (buen estado de carga), candado + barras de señal por red, IP/MAC al final de la lista, re-scan accesible en el bandeau.
**Punto débil** : red actual no señalada + orden inestable (quick win 6) ; cancelar desde el teclado devuelve a la lista **Configuración**, no al selector WiFi — para corregir un error de selección hay que re-ingresar y re-scannear.

### Teclado (`clavier-minuscules-vide` → `clavier-symboles`)
**Funciona bien** : 3 modos completos (min/MAJ/&123), tecla de modo activa en resaltado azul, ojo mostrar/ocultar funcional (`clavier-masque`), « Validar » explícito en el bandeau, ocultamiento LVGL con ventana de 900 ms sobre el último carácter (.ino:2727) — comportamiento estándar.
**Punto débil** : dos affordances de validación redundantes (« Validar » bandeau + ✓ azul teclado) ; en modo &123, los símbolos `<` `>` cohabitán con las flechas de desplazamiento del cursor en la misma fila visual — posible confusión ; la cancelación (tap bandeau) no está indicada en ningún lugar por una etiqueta.

### Cuenta (`compte-deconnexion`)
Pantalla casi vacía : icono nube, nombre de cuenta, botón rojo « Desconexión ». ~80 % del espacio no utilizado mientras que el usuario querría : correo de cuenta, estado/hora de la última sincronización, número de bobinas. El único elemento accionable es destructivo. Esfuerzo M — a tratar con la maqueta primero (regla del repositorio).

### Calibrado (`calibrage-etape1`)
**Funciona bien** : « Calibrado (1/4) » en el bandeau (progresión clara), instrucciones numeradas, factor actual mostrado, botón grande SIGUIENTE.
**No probado más allá de la etapa 1** (consigna). No hay botón Cancelar explícito — el retorno bandeau funciona pero nada lo indica en un asistente donde el usuario teme « romper » la configuración en curso.

### RFID / MATERIEL (`rfid-materiel`)
**Funciona bien** : badges LECTEURS 1/2 azules (detección al iniciar), Scanner ⇄
Parada con cambio de color rojo — buen feedback de actividad.
**Punto fuerte** : quick win 8 (título, escala, requisitos TEST). Durante mi prueba,
la pantalla mostraba **simultáneamente** LECTORES 1 y 2 en azul (« detectados ») y
« PN532-1/2 no ack » en rojo — dos verdades contradictorias sin explicación ;
el usuario no puede saber si su material está en fallo (probable causa :
plato no vacío, cf. .ino:10561 — nunca se muestra en pantalla). « Scanner » (mixto) vs
« TEST » (mayúsculas) : incoherencia de mayúsculas. El escaneo no tiene timeout visible.

### Actualización (`maj-a-jour`)
Sencillo y justo (« v3.1.3 / A día » en verde), pero : no hay botón « Verificar
ahora », no hay horario de última verificación, pantalla con tres cuartos vacía. Instalación no probada (instrucciones).

### LAN (`lan-code-acces`)
**Funciona bien** : IP, código, interruptor Vista en vivo verde — todo cabe en una pantalla.
**Punto fuerte** : quick win 5 (regeneración sin confirmación) ; la pantalla no explica
*como usar* el código — mostrar la URL completa `http://<ip>/live` haría que la función sea autosuficiente ; el valor « Sí » de la línea LAN en Ajustes
designa en realidad « Vista en vivo activada » — etiqueta a precisar.

### Modo de espera (`veille`)
**Funciona bien** : interruptor + chips de retardo + frase de estado (« La pantalla permanece
encendida »).
**Punto fuerte** : chips 1/2/5/10/15/30 sin unidad mientras la espera está desactivada
(la unidad « minutos » solo aparece mediante la frase de estado una vez activada) ;
estado desactivado de las chips poco diferenciado del normal ; acentos (quick win 10).

### Flujo de pesaje (operador en el banco)
Secuencia medida (bobina ~788 g, 2 etiquetas TigerTag) :
colocación → **~4–6 s** badge « Retirar material » + logo bobina + « R3D PLA High S »
(`pesee-tag-detecte-787g`) → **~10 s** CONTENEDOR 278 g / FILAMENTO 510 g /
Rack 4 / D7 (`pesee-donnees-cloud-completes`). Retiro → retorno « Listo »/0 g
inmediato y limpio (`pesee-retrait-retour-pret`). Re-colocación : datos completos
desde la primera captura (cache de metadatos eficaz,
`pesee-repose-cache-instantane`). Retiro prematuro (~2 s después del bip) : abandono
limpio sin estado bloqueado, pero silencioso (quick win 2).
**Fallas de feedback** : durante las 4–6 primeras segundos el badge permanece
« Listo » (el usuario no sabe si la lectura está en curso) ; entre
detección y datos en la nube, CONTENEDOR/FILAMENTO permanecen « -- » sin indicador
de actividad ; el bip no es una señal fiable (0, 1 o 2 bips según el lector
que detecta, twin-UID recuperado en la nube — confirmado por el operador) por lo que la pantalla
es el único canal de confirmación, lo que refuerza el quick win 2. Notado también :
la vista en vivo mostró « reconectando » durante la solicitud en la nube (corazón ocupado) —
sin impacto en el panel, pero síntoma de carga en ese momento.

## Incoherencias de código/UI detectadas al pasar

- `TigerTagSplashESP32.ino:3717` — « Son: » literal, único etiquetado no-i18n de
  la pantalla Volumen (visible en modo EN).
- `TigerTagSplashESP32.ino:12690` — valor « 0.0 » del botón TARE se crea una vez,
  nunca se refresca (puntero local, ningún update en otro lugar).
- `TigerTagSplashESP32.ino:5794` vs `:4130` — línea « RFID » (literal) abre un
  pantalla titulada `t(I18N_HARDWARE)` = « MATERIEL ».
- `TigerTagSplashESP32.ino:12827–12828` — `substring(0,10)` sobre marca y
  materia : truncamiento sin elipsis (« PLA High S »).
- `TigerTagSplashESP32.ino:4642, 4685–4690` — regeneración del código LAN : botón
  38×38, acción inmediata sin confirmación, en un bucle bloqueante donde
  `lvglAskYesNo` sería sin embargo utilizable.
- `TigerTagSplashESP32.ino:12760–12767` — estados WEIGHING/SENDING/SYNCED definidos
  pero ventana de visualización demasiado corta para ser vista (badge observado : Listo →
  Retirar material).
- `TigerTagSplashESP32.ino:10561` — el requisito « plato vacío » del self-test
  RF existe solo en comentario ; en pantalla, badges « detectados » y « no ack »
  coexisten sin explicación.
- `i18n.h:486` (y cadena SLEEP_MINUTES vecina) — acentos reemplazados por
  espacios, mientras que `LAN_CODE` (« Código de acceso ») prueba que la fuente los renderiza.
- Causa raíz de los acentos ausentes en todas partes : fuentes LVGL integradas Montserrat
  (ASCII solo, `.ino:629–631`). El pipeline de subconjuntos existe ya para el CJK (`scripts/make-cjk-font.sh`) —
  el mismo mecanismo puede producir un Montserrat Latin-1 y hacer que « Parámetros », « Desconexión », « A día », « Francés », « Español », « Português »
  sean correctos en 6 de 8 idiomas.
  Esfuerzo M (flash + verificación `check-i18n`), compatible LVGL v8.

## Lo que no pudo ser probado, y por qué

- **Omitidos** : el protocolo de vista en vivo solo transmite taps — sensación del
  scroll con el dedo, arrastre posible de la escala de Volumen, inercia de las listas : no evaluados. Los dobles taps rápidos (< ~600 ms) se pierden a través de la vista en vivo
  (inyección mantenida 2 lecturas/30 ms, `.ino:2277–2289`) ; los golpes rápidos en el teclado son por tanto a re-verificar con el dedo antes de concluir sobre un defecto.
- **Audio** : bips inaudibles a distancia ; comportamiento descrito por el operador
  (0–2 bips no deterministas según los lectores).
- **Modo de espera real** : no activado (ajuste persistente, dispositivo de banco).
- **Límites respetados** : no hay validación WiFi, no hay Instalación OTA, no hay Logout, no hay regeneración del código LAN, calibrado detenido en la etapa 1.
- **RFID izquierda/derecha** : no discriminable — la bobina lleva 2 etiquetas y el primer lector que detecta gana.
- **Transitorio del retiro prematuro** : mi sesión de vista en vivo se desconectó en ese momento ; estado final verificado (retorno « Listo » limpio), transición descrita por el operador (borrado tan pronto el peso se acerque a 0, ningún mensaje).
- **Artefactos visuales notados pero atribuibles a la vista en vivo** (bandas mezcladas
  durante las animaciones de scroll, ex. `settings-scroll-artefact-bandes`) : el
  panel físico probablemente no está afectado — a confirmar con una mirada al banco.
- **Multi-viewers / 503** : no provocado voluntariamente.

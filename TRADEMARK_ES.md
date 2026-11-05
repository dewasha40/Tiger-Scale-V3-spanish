# Política de marca comercial — TigerTag, TigerScale y Tiger Studio

Este es la edición V3 de la política que ya rige
[TigerScale V2](https://github.com/TigerTag-Project/Tiger-Scale/blob/main/TRADEMARK.md).
Las reglas son las mismas; lo que difiere es lo específico de V3, y se destaca a continuación.

## En resumen

**Puedes**

- **Fabricar y vender** hardware basado en este diseño, libre y comercialmente.
- Llamar a tu producto **"TigerScale"** si corre la firmware oficial sin modificaciones.
- **Forkear** la firmware y construir algo diferente — bajo otro nombre de producto.

**No puedes**

- Usar el nombre **"TigerTag"** para un protocolo RFID o servicio en la nube competitivo.
- Nombrar un fork o aplicación derivada **"Tiger Studio"** sin autorización.
- Afirmar **certificación oficial de TigerTag** sin autorización.

---

## El nombre "TigerTag"

**"TigerTag"** es una marca comercial del Proyecto TigerTag. Se refiere al protocolo RFID de seguimiento de bobinas y al servicio en la nube que mantiene el inventario de filamento sincronizado entre dispositivos. El ecosistema se describe en
[tigersystem.io](https://tigersystem.io).

Puedes referirte a él descriptivamente — *"compatible con TigerTag"*, *"funciona con el servicio en la nube de TigerTag"* — sin pedir permiso. No puedes usar "TigerTag" como nombre de un protocolo, servicio en la nube o marca de producto competitivo sin autorización escrita explícita.

---

## El nombre "TigerScale"

**"TigerScale"** identifica el diseño oficial de la báscula de código abierto. V3 se publica en
[github.com/TigerTag-Project/Tiger-Scale-V3](https://github.com/TigerTag-Project/Tiger-Scale-V-3);
V2 es una máquina diferente con su propio repositorio, y su firmware no corre en esta placa.

### Puedes llamar a tu producto "TigerScale" si

1. Está basado en el diseño de hardware publicado en este repositorio.
2. Corre la **firmware oficial** de este repositorio — sin modificaciones, o con solo cambios contribuidos y fusionados en `main`.
3. La firmware aún se identifica con la versión oficial, por ejemplo `fw_version: "3.1.2"`, en lugar de una cadena personalizada de fork.

Fabrica y vende hardware que corre la firmware oficial y estás creando una TigerScale oficial. Usa el nombre libremente.

### No puedes llamar a tu producto "TigerScale" si

- Corre firmware modificado, forkado o personalizado que se desvía de `main`.
- No implementa el protocolo RFID TigerTag tal como se especifica.
- Usa el nombre TigerScale o TigerTag para sugerir un respaldo que no tiene.

Los forks son bienvenidos — dales un nombre claramente distinto.

---

## Fabricación comercial

No hay **tarifa de licencia, royalty ni registro**. Para producir unidades compatibles:

1. **Construye el hardware** — la lista de materiales y la conexión eléctrica están en
   [`docs/HARDWARE.md`](docs/HARDWARE_ES.md). La carcasa impresible es un
   [modelo descargable 3MF](https://makerworld.com/en/models/3161869-tigerscale-v3-best-smart-filament-scale-with-nfc#profileId-3573543).
2. **Flasha la firmware oficial más reciente.** El
   [instalador web](https://tigertag-project.github.io/Tiger-Scale-V3/) siempre
   sirve la versión actual, y la báscula se actualiza automáticamente después de eso.
3. **Deja la identidad intacta** — `TIGERSCALE_FW_VERSION`, `TIGERSCALE_GIT_SHA`
   y el patrón de nombre de host mDNS `tigerscale-XXXX` deben permanecer como publicados, o las aplicaciones TigerTag no reconocerán el dispositivo.

Sin papeleo, sin paso de aprobación.

### Una cosa específica de V3

V3 elige su transporte NFC **en tiempo de compilación**, por lo que hay una compilación oficial por conexión eléctrica — HSU/UART, SPI o I2C. Flashar la incorrecta da una báscula que arranca perfectamente y nunca detecta una etiqueta, sin nada en la pantalla para explicarlo. Si fabricas unidades, flasha la compilación que coincida con la conexión eléctrica que realmente ensamblaste; el instalador web pregunta esto primero por esta razón. Todas las tres son firmware oficial, por lo que cualquiera de ellas puede llevar el nombre.

---

## ¿Por qué existen las reglas?

TigerScale es libre y de código abierto, y el objetivo es que tantas personas como sea posible la construyan, usen y la vendan. La marca comercial existe por una razón: **para que el nombre tenga significado para la persona que lo compra.**

Alguien que compre una "TigerScale" que corre firmware no oficial puede encontrar que no alcanza el servicio en la nube de TigerTag, pesa de forma inexacta o se comporta de maneras que nadie puede soportar. La marca comercial es lo que mantiene el nombre unido a un dispositivo que funciona con el ecosistema.

---

## Tiger Studio

**"Tiger Studio"** es también una marca comercial del Proyecto TigerTag. Los forks son bienvenidos bajo la licencia MIT pero deben usar un nombre distinto, y pueden mostrar *"Potenciado por Tiger Studio"* o *"Construido sobre Tiger Studio"* como atribución. La guía completa sobre forks y atribución está en la
[política de marca comercial de Tiger Studio](https://github.com/TigerTag-Project/TigerTag_Studio_Manager/blob/main/TRADEMARK.md).

---

## Preguntas

Abre un problema, o búscanos en [Discord](https://discord.gg/3Qv5TSqnJH). Para solicitudes de autorización de marca comercial, contacta a través de
[tigersystem.io](https://tigersystem.io).

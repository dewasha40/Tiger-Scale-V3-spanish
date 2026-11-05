# Registro de trabajo — desde el último punto de control

La única fuente de verdad para todo lo realizado desde el último commit. Léalo al inicio de una sesión; anexe cambios en el momento en que se realiza un cambio, nunca en bloque al final.

Los encabezados coinciden con [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), por lo que en el momento de la liberación este archivo se sintetiza en la entrada de `CHANGELOG.md` y en la nota de liberación de `docs/release-notes/` sin necesidad de rederivarse del diff.

Manténgalo limpio durante el proceso: describa el **estado final**, no el viaje. Un "Agregado X" y un posterior "Corregido X" del mismo ciclo se fusionan en un solo punto. Cualquier cosa revertida desaparece por completo — nunca fue liberada. Un elemento de lista, un cambio lógico, siempre nombrando los archivos afectados.

Reinicie este archivo al encabezado anterior en cada punto de control. Nada se pierde: el registro bruto vive en el historial de git y la entrada sintetizada vive en el changelog.

---

## Cambios
- `TigerTagSplashESP32/TigerTagSplashESP32.ino`, `data/www/i18n.js`, `data/www/index.html`: español predeterminado para instalaciones nuevas y selector de idioma conservado.
- `README.md`, `docs/INSTALACION_ES.md`, `README.en.md`: documentación principal en español y copia del README original en inglés.
- `simulador/tigerscale_v3.sim.json`: perfil de TigerScale V3 para añadirlo al ESP32 Simulador.
- `README.md`, `docs/INSTALACION_ES.md`, `docs/img/interfaz-*-simulada.png`: foto del dispositivo y capturas de las tres páginas de la maqueta, identificadas como simulación.

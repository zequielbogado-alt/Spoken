# Spoken — TuWoW Edition (WoW 3.3.5a)

[![WoW Version](https://img.shields.io/badge/WoW-3.3.5a%20(Build%2012340)-blue.svg)](https://tuwow.com)
[![Interface](https://img.shields.io/badge/Interface-30300-orange.svg)](https://tuwow.com)
[![Platform](https://img.shields.io/badge/Server-TuWoW%20Server-red.svg)](https://tuwow.com)
[![Language](https://img.shields.io/badge/Audio-Espa%C3%B1ol%20(esES%20%2F%20esMX)-brightgreen.svg)](https://tuwow.com)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**Spoken: TuWoW Edition** es una suite de inmersión y narración sonora en vivo para World of Warcraft 3.3.5a (Wrath of the Lich King). Dota de voz hablada en español a los personajes no jugadores (PNJ), misiones, historias de zonas descubiertas, libros, cartas y jefes derrotados, transformando por completo la experiencia de juego con un estándar audiovisual moderno.

---

## 🌟 Características Principales (Para Jugadores)

* 📜 **Locución Completa de Misiones**: Escucha a los PNJ leerte sus encargos cuando aceptas, progresas o completas una misión.
* 🗺️ **Historias de Zonas y Lugares**: Al explorar el mundo o abrir el mapa, Spoken te narra el lore y la historia del lugar en el que te encuentras.
* 📖 **Lectura de Libros y Escritos**: Los libros del mundo, cartas, placas de piedra y pergaminos cuentan con locución cinematográfica.
* 💀 **Crónicas de Jefes**: Al vencer a los jefes de mazmorras y bandas, un narrador relata la caída y trasfondo del enemigo.
* 🏛️ **Compendio de Azeroth**: Enciclopedia visual interactiva con buscador rápido, filtros de contenido, porcentaje de exploración y registro de misiones completadas.
* 🎭 **Estilos de Narración Personalizables**:
  * **Solo subtítulos**: Subtítulos elegantes en la parte inferior de la pantalla con animación fluida de texto.
  * **Ventana pequeña**: Barra flotante compacta con retrato del personaje, botones de reproducción y controles.
  * **Ventana grande**: Estilo *Talking Head* cinematográfico con retrato 3D animado y caja de diálogo.
  * **Solo voz**: Disfruta de la narración en segundo plano sin interrumpir tu interfaz gráfica.
* 🛡️ **Protección Inteligente en Batalla**:
  * Si entras en combate, la narración se pausa de manera instantánea y sin tirones de rendimiento.
  * Al finalizar el combate, la locución se reanuda automáticamente desde el punto exacto.

---

## 🎮 Comandos y Atajos Rápidos

Puedes controlar Spoken desde el chat del juego o mediante el icono del minimapa:

| Comando | Acción |
| :--- | :--- |
| `/spoken` o `/spoken settings` | Abre la ventana principal de configuración moderna de Spoken. |
| `/spoken compendium` | Abre el Compendio de Azeroth (Misiones, Lugares, Lecturas, Jefes). |
| `/spoken play` | Reproduce o reanuda la narración actual. |
| `/spoken pause` | Pausa temporalmente el audio en curso. |
| `/spoken stop` | Detiene la locución activa y limpia la lista de espera. |
| `/spoken skip` | Salta al siguiente diálogo en cola. |

> **Consejo**: Puedes asignar atajos de teclado para estas funciones desde el menú de juego: **ESC > Teclado > sección Spoken**.

---

## 💻 Sección Técnica (Arquitectura e Ingeniería 3.3.5a)

Esta versión de Spoken ha sido portada y optimizada específicamente para el motor de **World of Warcraft 3.3.5a (Build 12340 / Interface 30300)** y el cliente de **TuWoW**, garantizando rendimiento extremo, cero tirones de FPS y compatibilidad nativa con sistemas modernos como ForeverUI.

### 1. Pipeline de Audio FMOD de Latencia Cero (`PlayMusic` Stream Engine)
* **El Problema en 3.3.5a**: El cliente clásico de WotLK reproduce sonidos externos mediante `PlaySoundFile(path, channel)` asociándolos al canal interno `SCRIPTSOUND`. En esta versión del motor, `PlaySoundFile` descarta el puntero del canal tras iniciar la reproducción y carece de la función `StopSound(handle)` de retail. La única vía nativa que existía para forzar el cese del audio era llamar a `Sound_GameSystem_RestartSoundSystem()`, la cual ejecuta internamente en C++ un `OsSleep(200)` seguido del reinicio completo de FMOD, causando una congelación severa del juego (~350 ms) cada vez que el usuario pulsaba "Detener" o entraba en combate.
* **La Solución en Spoken TuWoW**: Se implementó una arquitectura en `SoundUtils.lua` que utiliza la API de streaming `PlayMusic(path)`. Ambas funciones comparten la misma rutina base C++ en el ejecutable (`0x4c9110`), pero `PlayMusic` almacena su handle en memoria global (`0x00B4AC44`). Esto permite ejecutar `StopMusic()` (`0x985a10`), logrando un **corte y fade-out instantáneo en 0.00 ms sin reiniciar FMOD y con cero caídas de FPS**.
* **Gestión de Volúmenes Dinámica**: Spoken sincroniza los niveles de audio en tiempo real preservando las preferencias del jugador en `Sound_MusicVolume` y `Sound_EnableMusic`, restaurándolas fielmente al terminar o salir del juego.

### 2. Capa de Compatibilidad y Polyfills (`Compat335.lua`)
Dado que el código original de Spoken utilizaba APIs de versiones modernas de World of Warcraft, se construyó una suite de emulación de bajo nivel:
* **API de Misiones (`C_QuestLog`)**:
  * Implementación de soporte para el retorno de **9 valores** de `GetQuestLogTitle` en WotLK (`title, level, questTag, suggestedGroup, isHeader, isCollapsed, isComplete, isDaily, questID`), corrigiendo fallos de desplazamiento donde las misiones activas eran descartadas como encabezados.
  * Sincronización bidireccional de misiones completadas mediante eventos `QUEST_QUERY_COMPLETE`, hooks seguros a `GetQuestReward` y lectura cruzada con la base de datos de traducciones de Questie.
  * Polyfill robusto de `GetQuestID()` que resuelve el identificador numérico de la misión incluso en ventanas emergentes de aceptación (`QUEST_DETAIL`) sin causar taints.
* **API de Mapas y Áreas (`C_Map` / `C_MapExplorationInfo`)**:
  * Tabla de traducción bidireccional entre los `uiMapID` modernos (ej. Dun Morogh = 1426, Kalimdor = 1414) y los identificadores de área clásicos de 3.3.5a (`areaID`), permitiendo que el Compendio y las alertas de zona funcionen transparentemente.
* **UI & Rendering Polyfills**:
  * Soporte de metatablas para `Frame:SetColorTexture()`, `Frame:SetShown()`, `Frame:SetSize()` y emulación de `BackdropTemplate`.
  * Medición diferida de fontes TrueType (`Inter-Medium.ttf`) con protección ante alturas de fuente `0` en frames recién construidos.

### 3. Interfaz Desacoplada y Prevención de Taint
* **Ventana de Opciones Standalone (`SpokenStandaloneWindow.lua`)**: Diseñada como una interfaz independiente en estrato `HIGH`, con navegación lateral por pestañas, buscador integrado e inspectores en tiempo real, evitando modificar los paneles protegidos de `InterfaceOptionsFrame` durante el combate.
* **Estrato Dinámico en Vista Previa**: Durante la previsualización de subtítulos o estilos de ventana en la configuración, los frames ascienden a estrato `FULLSCREEN_DIALOG` para mostrarse siempre sobre las ventanas de opciones y restauran su estrato base automáticamente al cerrar.

---

## 📁 Estructura del Repositorio (9 Carpetas)

```
Spoken/
├── Spoken/                    # Núcleo del addon: motor de audio, interfaz, subtítulos y opciones
├── Spoken_Bosses/             # Módulo de narraciones épicas al derrotar jefes
├── Spoken_Places/             # Módulo de lore geográfico, zonas y subzonas
├── Spoken_Quests/             # Módulo de locución y seguimiento de misiones y diálogos (Gossip)
├── Spoken_Readables/          # Módulo de libros del mundo, pergaminos, notas y placas
├── SpokenContributions/       # Módulo para registrar y exportar textos/diálogos
├── SpokenPlacesAudio_esMX/    # Paquete de audio de zonas y lugares en español
├── SpokenQuestsAudio_esES/    # Paquete de audio de misiones en español
└── SpokenReadablesAudio_esMX/ # Paquete de audio de libros y lecturas en español
```

---

## 🛠️ Instalación para Desarrolladores

1. Clona este repositorio dentro de tu directorio de addons:
   ```bash
   git clone https://github.com/zequielbogado-alt/Spoken.git
   ```
2. Asegúrate de que las 9 carpetas del addon queden ubicadas en:
   ```
   World of Warcraft/Interface/AddOns/
     ├── Spoken/
     ├── Spoken_Bosses/
     ├── Spoken_Places/
     ├── Spoken_Quests/
     ├── Spoken_Readables/
     ├── SpokenContributions/
     ├── SpokenPlacesAudio_esMX/
     ├── SpokenQuestsAudio_esES/
     └── SpokenReadablesAudio_esMX/
   ```
3. Inicia el cliente de WoW o ejecuta `/reload` si ya te encuentras dentro del juego.

---

## 📜 Licencia y Créditos

Este proyecto se distribuye bajo los términos de la licencia **MIT** (consulta el archivo [LICENSE](LICENSE) para más detalles).

* **Spoken Original**: Desarrollado por el equipo de Spoken / VoiceOver ([rusty-key/spoken-wow](https://github.com/rusty-key/spoken-wow)).
* **Adaptación WotLK 3.3.5a y Generación de Voces**: Agradecimiento y reconocimiento especial a Zequiel Bogado ([zequielbogado-alt/Spoken](https://github.com/zequielbogado-alt/Spoken)) y a la comunidad de TuWoW por el port a 3.3.5a, la optimización FMOD y la laboriosa generación del catálogo sonoro en español.

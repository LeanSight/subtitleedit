# Reproducción de Video en SubtitleEdit

SubtitleEdit permite cargar un archivo de video y reproducirlo en sincronía con el subtítulo que se está editando. La app fue migrada de WinForms a **Avalonia MVVM**, lo que cambió completamente la estructura de este subsistema.

---

## Arquitectura general

```
MainViewModel.cs  (lógica principal, MVVM)
  └── VideoPlayerControl  (Avalonia UserControl)
        └── IVideoPlayerInstance  (interfaz)
              ├── LibMpvDynamicPlayer   (MPV - 3 modos de render)
              │     ├── LibMpvDynamicNativeControl   (modo WID/HWND)
              │     ├── LibMpvDynamicOpenGlControl   (modo OpenGL)
              │     └── LibMpvDynamicSoftwareControl (modo SW, recomendado Windows)
              └── LibVlcDynamicPlayer   (VLC - 3 modos de render)
                    ├── LibVlcDynamicNativeControl
              │     ├── LibVlcDynamicOpenGlControl
              │     └── LibVlcDynamicSoftwareControl
```

> **Nota:** Los backends anteriores QuartsPlayer (DirectShow) y MpcHc fueron eliminados en la migración a Avalonia.

---

## 1. Interfaz `IVideoPlayerInstance`

**Archivo:** `src/UI/Logic/VideoPlayers/IVideoPlayerInstance.cs`

Contrato que todos los backends deben cumplir (usa `async Task` para carga de archivos):

| Miembro | Tipo | Descripción |
|---|---|---|
| `Name` | `string` | Nombre del backend |
| `FileName` | `string` | Ruta del archivo cargado |
| `IsPlaying` | `bool` | Estado de reproducción |
| `IsPaused` | `bool` | Estado de pausa |
| `Position` | `double` | Posición actual en segundos (get/set) |
| `Duration` | `double` | Duración total en segundos |
| `Volume` | `double` | Volumen actual (get/set) |
| `VolumeMaximum` | `int` | Volumen máximo |
| `Speed` | `double` | Velocidad de reproducción (get/set) |
| `CanLoad()` | `bool` | Verifica si el backend puede cargar un archivo dado |
| `LoadFile()` | `Task` | Carga un archivo de video (async) |
| `CloseFile()` | `void` | Cierra el archivo actual |
| `Play()` | `void` | Inicia reproducción |
| `PlayOrPause()` | `void` | Alterna entre play y pause |
| `Pause()` | `void` | Pausa reproducción |
| `Stop()` | `void` | Detiene y reinicia posición |
| `ToggleAudioTrack()` | `AudioTrackInfo?` | Cambia la pista de audio activa |

---

## 2. Backends de reproducción

### LibMpvDynamic
**Directorio:** `src/UI/Logic/VideoPlayers/LibMpvDynamic/`

Backend basado en MPV. Usa carga dinámica de `libmpv`. Organizado en archivos especializados:

| Archivo | Rol |
|---|---|
| `LibMpvDynamicPlayer.cs` | Implementación principal de `IVideoPlayerInstance` |
| `LibMpvDynamicNativeControl.cs` | Render nativo: embebe mpv en un `HWND` (Window ID) |
| `LibMpvDynamicOpenGlControl.cs` | Render vía OpenGL |
| `LibMpvDynamicSoftwareControl.cs` | Render por software (recomendado en Windows, evita el "airspace problem" de WPF/Avalonia) |
| `NativeMethods.cs` | Declaraciones P/Invoke para `libmpv` |
| `AudioTrackInfo.cs` | Estructura de información de pista de audio |
| `PlatformCursorManager.cs` | Gestión de cursor multiplataforma |

### LibVlcDynamic
**Directorio:** `src/UI/Logic/VideoPlayers/LibVlcDynamic/`

Backend basado en VLC. Estructura análoga a LibMpvDynamic:

| Archivo | Rol |
|---|---|
| `LibVlcDynamicPlayer.cs` | Implementación principal de `IVideoPlayerInstance` |
| `LibVlcDynamicNativeControl.cs` | Render nativo en ventana Win32/X11 |
| `LibVlcDynamicOpenGlControl.cs` | Render vía OpenGL |
| `LibVlcDynamicSoftwareControl.cs` | Render por software |

### Nombres de backend
**Archivo:** `src/UI/Logic/Config/VideoPlayerName.cs`

Constantes para selección de backend:

```
MpvOpenGl  →  "mpv-opengl"
MpvWid     →  "mpv-wid"
MpvSw      →  "mpv-sw"
Vlc        →  "vlc"
```

---

## 3. Control UI `VideoPlayerControl`

**Archivo:** `src/UI/Controls/VideoPlayer/VideoPlayerControl.cs`

Avalonia `UserControl` que envuelve un `IVideoPlayerInstance`. Expone:

- `PlayerContent`: el control de render del backend activo
- `VideoPlayerInstance`: acceso directo al backend
- `Volume`, `Position`, `Duration`: propiedades bindeadas al ViewModel
- `ProgressText`: texto del timecode visible en la barra de progreso
- `PlayCommand`, `StopCommand`, `FullScreenCommand`: comandos Avalonia
- `StopIsVisible`, `FullScreenIsVisible`: visibilidad condicional de botones
- `IsFullScreen`: propiedad para modo pantalla completa
- `VideoPlayerDisplayTimeLeft`: toggle entre tiempo restante/actual
- `ClickToTogglePlay`: habilita/deshabilita click para play/pause
- `IsSmpteTimingEnabled`: ajusta el timing para formato SMPTE
- **Eventos:** `PositionChanged`, `VolumeChanged`, `PlayPauseRequested`, `StopRequested`, `FullscreenRequested`, `SurfacePointerPressed`, `ToggleDisplayProgressTextModeRequested`

Métodos principales:
- `Open()` / `Close()`: carga y cierra archivos de video
- `SetPosition()` / `SetPositionDisplayOnly()`: posicionamiento
- `TogglePlayPause()`, `ToggleAudioTrack()`: controles de reproducción
- `WaitForPlayersReadyAsync()`: espera a que el video esté listo

El timer de posición usa intervalo de **50ms** (no 17ms) con "slow poll" cada 5 ticks (~250ms) para reducir overhead de P/Invoke.

**Stub:** Cuando no hay backend disponible, usa `VideoPlayerInstanceNone.cs`.

---

## 4. Carga de video

**Archivo principal:** `src/UI/Features/Main/MainViewModel.cs`

### Obtención del control de video
Método `GetVideoPlayerControl()` — línea ~6057

Retorna el `VideoPlayerControl` activo considerando tres escenarios:
1. `_fullScreenVideoPlayerControl` (modo pantalla completa)
2. `_videoPlayerUndockedViewModel?.VideoPlayerControl` (ventana flotante)
3. `VideoPlayerControl` (layout principal)

### Inicialización del layout
**Archivo:** `src/UI/Features/Main/Layout/InitVideoPlayer.cs`

- `MakeLayoutVideoPlayer()`: crea el Grid con drag & drop
- `MakeVideoPlayer()`: instancia el backend según configuración
- `MakeVideoPlayerPreferNonNative()`: **nuevo método** que evita el "airspace problem" usando renderizado por software en Windows

El método `MakeVideoPlayerPreferNonNative()` es importante porque en Windows, `NativeControlHost` (usado por mpv-wid y VLC) crea un HWND de Win32 que siempre renderiza sobre los overlays de Avalonia, haciendo invisibles los previews de logo/overlay. Usar software rendering (`LibMpvDynamicSoftwareControl`) evita este problema.

### Drag & drop y apertura de archivo
Gestionados mediante el sistema de comandos y eventos de Avalonia (reemplaza el antiguo `MediaPlayerDragDrop()` de WinForms).

### Metadatos del video `VideoInfo`
**Archivo:** `src/libse/Common/VideoInfo.cs`

Campos: `Width`, `Height`, `TotalMilliseconds`, `TotalSeconds`, `FramesPerSecond`, `TotalFrames`, `VideoCodec`, `FileType`, `Success`.

---

## 5. Sincronización subtítulo ↔ video

El sistema anterior usaba un timer de 17 ms (`ShowSubtitleTimerTick`). En la arquitectura Avalonia MVVM el `VideoPlayerControl` usa un timer de **50ms** con "slow poll" cada 5 ticks (~250ms) para reducir overhead de P/Invoke:

- `VideoPlayerControl` tiene un `DispatcherTimer` de 50ms que actualiza `Position` y `Duration`
- El binding Avalonia propaga cambios a la UI automáticamente
- `MainViewModel` usa el método `ShowSubtitleNotLoadedMessage()` (línea ~4117) para determinar el subtítulo visible comparando la posición actual con los `StartTime`/`EndTime` de cada `Paragraph`

La sincronización de la lista de subtítulos con el video sigue disponible, gestionada por el ViewModel mediante comandos reactivos y el método `GetVideoPlayerControl()`.

---

## 6. Waveform y visualizador de audio

**Archivo:** `src/UI/Controls/AudioVisualizerControl/AudioVisualizer.cs`

Control Avalonia personalizado (~2700 líneas) que renderiza la forma de onda del audio extraído del video. Reemplaza el antiguo formulario `AddWaveform.cs`.

Propiedades principales:
- `WavePeaks`: datos de picos de la forma de onda (`WavePeakData2?`)
- `StartPositionSeconds`: posición inicial visible
- `ZoomFactor` / `VerticalZoomFactor`: factores de zoom
- `CurrentVideoPositionSeconds`: posición actual del video
- `DrawGridLines`: mostrar líneas de grid
- `WaveformColor`, `WaveformBackgroundColor`, `WaveformSelectedColor`, `WaveformCursorColor`: colores
- `InvertMouseWheel`: inversión de scroll

**Inicialización:** `src/UI/Features/Main/Layout/InitWaveform.cs` (~770 líneas)

Contiene la clase `NonSpaceButton` y métodos de inicialización del waveform.

La extracción de audio del video (FFmpeg/VLC) sigue siendo el mecanismo subyacente, ahora integrado en el flujo MVVM.

---

## 7. Ventanas flotantes (Undocked)

Reemplazadas por pares ViewModel/Window en Avalonia:

| Componente | Archivo |
|---|---|
| Video flotante (VM) | `src/UI/Features/Shared/Undocked/VideoPlayerUndockedViewModel.cs` |
| Video flotante (Vista) | `src/UI/Features/Shared/Undocked/VideoPlayerUndockedWindow.cs` |
| Waveform flotante (VM) | `src/UI/Features/Shared/Undocked/AudioVisualizerUndockedViewModel.cs` |
| Waveform flotante (Vista) | `src/UI/Features/Shared/Undocked/AudioVisualizerUndockedWindow.cs` |

---

## 8. Archivos clave

| Propósito | Ruta |
|---|---|
| Interfaz de backend | `src/UI/Logic/VideoPlayers/IVideoPlayerInstance.cs` |
| Backend MPV | `src/UI/Logic/VideoPlayers/LibMpvDynamic/LibMpvDynamicPlayer.cs` |
| Backend VLC | `src/UI/Logic/VideoPlayers/LibVlcDynamic/LibVlcDynamicPlayer.cs` |
| Stub (sin backend) | `src/UI/Controls/VideoPlayer/VideoPlayerInstanceNone.cs` |
| Nombres de backend | `src/UI/Logic/Config/VideoPlayerName.cs` |
| Control UI del reproductor | `src/UI/Controls/VideoPlayer/VideoPlayerControl.cs` |
| ViewModel principal | `src/UI/Features/Main/MainViewModel.cs` |
| GetVideoPlayerControl | `src/UI/Features/Main/MainViewModel.cs:6057` |
| Init layout video | `src/UI/Features/Main/Layout/InitVideoPlayer.cs` |
| Init waveform | `src/UI/Features/Main/Layout/InitWaveform.cs` |
| Visualizador de audio | `src/UI/Controls/AudioVisualizerControl/AudioVisualizer.cs` |
| Metadatos de video | `src/libse/Common/VideoInfo.cs` |
| Video flotante (VM) | `src/UI/Features/Shared/Undocked/VideoPlayerUndockedViewModel.cs` |
| Video flotante (Window) | `src/UI/Features/Shared/Undocked/VideoPlayerUndockedWindow.cs` |
| Waveform flotante (VM) | `src/UI/Features/Shared/Undocked/AudioVisualizerUndockedViewModel.cs` |
| Waveform flotante (Window) | `src/UI/Features/Shared/Undocked/AudioVisualizerUndockedWindow.cs` |

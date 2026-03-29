# Plan: Soporte para carga de audio-only en SubtitleEdit

## Objetivo

Permitir cargar archivos de audio (.mp3, .wav, .flac, .ogg, .m4a, .aac) sin video, manteniendo todas las funcionalidades de reproducción, waveform y sincronización con subtítulos.

## Metodología

Este plan sigue **ATDD + TDD + One-Piece-Flow**:
- Cada slice es un commit atómico
- Tests en RED antes de producción
- GREEN + refactor antes de avanzar

---

## Diagnóstico de Brechas

| # | Componente | Brecha |
|---|------------|--------|
| 1 | Filtro de archivos | Solo acepta video (mkv, mp4, ts, mov, mpeg, m2ts) |
| 2 | VideoPlayerControl | Asume siempre hay componente de video |
| 3 | Waveform | Extrae audio de video, no de archivo directo |
| 4 | GetMediaInformation | Asume contenedor de video |
| 5 | LoadAudioTrackMenuItems | Busca dentro de contenedor de video |

---

## Slices

### Slice 1: Extender filtro de archivos

**Objetivo**: Agregar soporte para seleccionar archivos de audio en el diálogo de apertura.

**Archivos afectados**:
- `src/UI/Logic/Media/FileHelper.cs` - AGREGADO: método `PickOpenAudioFiles()`
- `src/UI/Logic/Media/IFileHelper.cs` - AGREGADO: interfaz del método
- `src/UI/Logic/Config/Language/LanguageGeneral.cs` - AGREGADO: localize keys

**Estado**: ✅ COMPLETADO

---

### Slice 2: Interfaz para modo audio-only

**Objetivo**: Extender `IVideoPlayerInstance` para indicar si el contenido actual es audio-only.

**Archivos afectados**:
- `src/UI/Logic/VideoPlayers/IVideoPlayerInstance.cs` - AGREGADO: propiedad `IsAudioOnly`
- `src/UI/Logic/VideoPlayers/LibMpvDynamic/LibMpvDynamicPlayer.cs` - AGREGADO: implementación
- `src/UI/Logic/VideoPlayers/LibVlcDynamic/LibVlcDynamicPlayer.cs` - AGREGADO: implementación
- `src/UI/Controls/VideoPlayer/VideoPlayerInstanceNone.cs` - AGREGADO: stub

**Estado**: ✅ COMPLETADO

---

### Slice 3: VideoPlayerControl audio-only mode

**Objetivo**: Adaptar el control de UI para exponer si es audio-only.

**Archivos afectados**:
- `src/UI/Controls/VideoPlayer/VideoPlayerControl.cs` - AGREGADO: propiedad `IsAudioOnly`

**Estado**: ✅ COMPLETADO

---

### Slice 4: Waveform para audio directo

**Objetivo**: El waveform se genera correctamente a partir de archivos de audio.

**Análisis**: El código existente ya funciona para archivos de audio porque:
- `WaveFileExtractor.GetCommandLineProcess` usa FFmpeg con `-vn` (ignora video)
- `WavePeakGenerator2.GetPeakWaveFileName` usa hash del archivo
- FFmpeg puede abrir archivos de audio directamente

**Estado**: ✅ NO REQUIERE CAMBIOS

---

### Slice 5: GetMediaInformation para audio

**Objetivo**: Obtener metadatos de audio sin fallar.

**Análisis**: El código existente ya maneja esto:
- `FfmpegMediaInfo2.Parse` funciona con archivos de audio
- Si falla, establece `_mediaInfo = null` con valor por defecto

**Estado**: ✅ NO REQUIERE CAMBIOS

---

### Slice 6: Integración UI - Comando de audio

**Objetivo**: El usuario puede abrir audio desde el menú.

**Archivos afectados**:
- `src/UI/Features/Main/MainViewModel.cs` - AGREGADO: comando `CommandAudioOpen()` y método `AudioOpenFile()`
- `src/UI/Logic/Config/Language/LanguageGeneral.cs` - AGREGADO: localize keys

**Estado**: ✅ COMPLETADO

---

## Resumen de cambios

### Archivos modificados

| Archivo | Cambio |
|---------|--------|
| `src/UI/Logic/Media/IFileHelper.cs` | + `PickOpenAudioFiles()` |
| `src/UI/Logic/Media/FileHelper.cs` | + `PickOpenAudioFiles()`, `MakeOpenAudioFilter()`, `GetAudioExtensions()` |
| `src/UI/Logic/Config/Language/LanguageGeneral.cs` | + `OpenAudioFile`, `OpenAudioFileTitle` |
| `src/UI/Logic/VideoPlayers/IVideoPlayerInstance.cs` | + `IsAudioOnly { get; }` |
| `src/UI/Logic/VideoPlayers/LibMpvDynamic/LibMpvDynamicPlayer.cs` | + `IsAudioOnly` property |
| `src/UI/Logic/VideoPlayers/LibVlcDynamic/LibVlcDynamicPlayer.cs` | + `IsAudioOnly` property |
| `src/UI/Controls/VideoPlayer/VideoPlayerInstanceNone.cs` | + `IsAudioOnly` property |
| `src/UI/Controls/VideoPlayer/VideoPlayerControl.cs` | + `IsAudioOnly` property |
| `src/UI/Features/Main/MainViewModel.cs` | + `CommandAudioOpen()`, `AudioOpenFile()` |

### Extensiones de audio soportadas

```csharp
private static readonly string[] AudioExtensions = 
    { ".mp3", ".wav", ".flac", ".ogg", ".m4a", ".aac", ".wma", ".opus" };
```

---

## Pendiente: Integración con menú UI

Para completar la funcionalidad, falta conectar el comando `CommandAudioOpen` al menú de la aplicación en `InitMenu.cs` o equivalente.

---

## Referencias

- `devdocs/video-playback.md`: Documentación del sistema de video actual
- `src/UI/Logic/VideoPlayers/IVideoPlayerInstance.cs`: Interfaz del reproductor
- `src/UI/Controls/VideoPlayer/VideoPlayerControl.cs`: Control de UI
- `src/UI/Logic/Media/FileHelper.cs`: Dialogos de archivo

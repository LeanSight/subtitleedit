# Analisis de UI de SubtitleEdit

## 1. Framework de UI

**Windows Forms (.NET Framework 4.8)**

## 2. Estructura de Directorios

```
src/ui/
├── Controls/          # Controles personalizados
├── Forms/             # Todas las ventanas/dialogos (423 archivos)
├── Logic/             # Logica de UI
├── Languages (XML)    # Archivos de traduccion
├── Resources/         # Recursos
└── Program.cs         # Punto de entrada
```

## 3. Componentes Principales

| Componente | Descripcion |
|-----------|-------------|
| MenuStrip | Archivo, Editar, Herramientas, Subtítulos, Video |
| ToolStrip | Barra de herramientas |
| StatusStrip | Estado, progreso |
| SplitContainers | Paneles dividibles |
| SubtitleListView | Lista de subtitulos |
| AudioVisualizer | Waveform |
| VideoPlayerContainer | Reproductor de video |

## 4. Configuracion de Fuentes

### Settings > General > Font in UI
- General: Fuente, Color
- List Views: Tamano, Bold
- Text Box: Tamano, Bold

### Settings > Video > Default video player
- Preview font name/size/bold

## 5. Atajos de Video

| Funcion | Tecla por defecto |
|---------|-------------------|
| 500 ms | Alt+Left/Right |
| 1 segundo | configurable |
| 3 segundos | F7/F8 |
| 5 segundos | configurable |

**NO existe atajo de 2 segundos.**

### Diagnostico F7/F8

- F7 = Retroceder 3s (solo funciona con video cargado)
- F8 = Ir al siguiente error
- Requiere: `mediaPlayer.VideoPlayer != null`

## 6. Archivos Clave

- `src/libse/Settings/GeneralSettings.cs`
- `src/libse/Settings/Settings.cs`
- `src/ui/Forms/Options/Settings.Designer.cs`
- `src/ui/Forms/Main.cs`
- `src/ui/Controls/SETextBox.cs`

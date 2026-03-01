# Panic Plugin - Architecture & Development Guide

## 🏗️ Architecture Overview

### Component Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     IDE (IntelliJ/Android Studio)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │            Panic Plugin Components                        │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │                                                            │   │
│  │  1. Execution Monitoring                                 │   │
│  │     └─ ExecutionFailureListener                          │   │
│  │        - Listens to build/run execution events           │   │
│  │        - Detects failures (processNotStarted)            │   │
│  │        - Detects success (processStarted)                │   │
│  │                                                            │   │
│  │  2. State Management                                     │   │
│  │     └─ FaahStateService                                  │   │
│  │        - Persistent state using PersistentStateComponent │   │
│  │        - Publishes panic/redemption events               │   │
│  │        - Tracks panic level across sessions              │   │
│  │                                                            │   │
│  │  3. UI Managers                                          │   │
│  │     ├─ PanicOverlayManager                               │   │
│  │     │  - Shows red pulsing overlay                       │   │
│  │     │  - Plays scream audio                              │   │
│  │     │  └─ PanicOverlayComponent (renders sine-wave)     │   │
│  │     │                                                     │   │
│  │     └─ RedemptionManager                                 │   │
│  │        - Shows GTA "Mission Passed" overlay              │   │
│  │        - Plays mission passed audio                      │   │
│  │        └─ RedemptionOverlayComponent (renders image)     │   │
│  │                                                            │   │
│  │  4. Audio System                                         │   │
│  │     └─ AudioPlayer                                       │   │
│  │        - Loads bundled MP3s from resources               │   │
│  │        - Plays via javax.sound.sampled.Clip              │   │
│  │        - Fallback to afplay (macOS)                      │   │
│  │                                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## 📁 Code Structure

```
src/main/kotlin/com/gtxtreme/panic/
├── FaahStateService.kt              # Persistent state management
├── PanicStateChangeEvent.kt          # Event model
├── PanicStateListener.kt             # Event listener interface
│
├── startup/
│   └── PanicPluginStartup.kt        # Plugin initialization (ProjectActivity)
│
├── listeners/
│   ├── ExecutionFailureListener.kt   # Monitors build/run execution
│   ├── SimpleBuildListener.kt        # (Experimental) Gradle build events
│   ├── GradleBuildListener.kt        # (Experimental) Gradle monitoring
│   └── BuildFailureListener.kt       # (Legacy) Removed
│
├── ui/
│   ├── PanicOverlayManager.kt        # Red overlay manager
│   ├── PanicOverlayComponent.kt      # Red overlay renderer (sine-wave)
│   ├── RedemptionManager.kt          # GTA overlay manager
│   └── RedemptionOverlayComponent.kt # GTA overlay renderer (image)
│
├── audio/
│   └── AudioPlayer.kt                # Audio playback system
│
└── actions/
    ├── TestPanicAction.kt            # Test action: trigger panic
    └── TestRedemptionAction.kt       # Test action: trigger redemption

src/main/resources/
├── META-INF/plugin.xml               # Plugin manifest & configuration
├── audio/
│   ├── scream.mp3                    # Panic mode sound
│   └── mission_passed.mp3            # Redemption mode sound
└── images/
    └── mission_passed.png            # GTA overlay image
```

## 🔄 Execution Flow

### When a Build Fails:

```
User writes compilation error
         ↓
Click Play button (or Build menu)
         ↓
IDE attempts to execute
         ↓
ExecutionFailureListener catches execution
         ↓
processNotStarted() fires (build failed before execution started)
         ↓
stateService.enterPanic()
         ↓
FaahStateService publishes PanicStateChangeEvent
         ↓
PanicOverlayManager receives event
         ↓
PanicOverlayManager.showPanic()
  ├─ AudioPlayer.playScream() [async thread]
  └─ Create red pulsing overlay on IDE window
         ↓
Red overlay pulses for ~5 seconds
Sound plays: "FAAAH!"
```

### When Build Succeeds After Failure:

```
User fixes compilation error
         ↓
Click Play button again
         ↓
IDE builds successfully
         ↓
ExecutionFailureListener catches execution
         ↓
processStarted() fires (process successfully started)
         ↓
stateService.exitPanic()
         ↓
FaahStateService publishes PanicStateChangeEvent (isInPanic=false)
         ↓
RedemptionManager receives event
         ↓
RedemptionManager.showRedemption()
  ├─ AudioPlayer.playMissionPassed() [async thread]
  └─ Create GTA overlay on IDE window
         ↓
GTA overlay displays for 5 seconds
Sound plays: "Mission Passed!"
```

## 🎛️ Key Components Explained

### ExecutionFailureListener

**Purpose**: Monitors all IDE build and run executions

**Key Methods**:
- `processNotStarted()` - Called when build/run fails to start (typically compilation error)
  - Triggers panic mode
- `processStarted()` - Called when process successfully launches
  - Triggers redemption mode
- `processTerminated()` - Called when process ends (fallback detection)

**Why This Works**:
- Compilation errors prevent the process from starting → `processNotStarted()`
- Successful builds allow the process to start → `processStarted()`
- This covers all IDE execution paths (builds, run configs, etc.)

### FaahStateService

**Purpose**: Central state management with persistence

**Features**:
- Extends `PersistentStateComponent<State>`
- Persists panic state to disk automatically
- Publishes events via message bus (PANIC_STATE_TOPIC)
- Tracks panic level (re-triggers audio on successive failures)

**State Persistence**:
```
~/.config/JetBrains/IdeaIC2024.1/system/component_state.xml
  └─ com.gtxtreme.panic.FaahStateService
     └─ <panicLevel>1</panicLevel>
```

### PanicOverlayManager & RedemptionManager

**Purpose**: UI rendering and lifecycle management

**Responsibilities**:
- Listen to state change events
- Retrieve IDE window frame
- Create and manage overlay components
- Handle repaint timers
- Clean up overlays after timeout

**Window Hierarchy Navigation**:
```
WindowManager.getIdeFrame(project)
    ↓ (IdeProjectFrameHelper)
→ component (IdeRootPane)
    ↓ (JRootPane)
→ layeredPane (JLayeredPane)
    ↓
→ add(overlayComponent, POPUP_LAYER)
```

### AudioPlayer

**Purpose**: Plays bundled MP3 audio files

**Features**:
- Loads from bundled resources (`src/main/resources/audio/`)
- Uses `javax.sound.sampled.Clip` for playback
- Fallback to `afplay` (macOS native audio)
- Async playback in separate thread
- Temp file cleanup after 3 seconds

**Resource Loading Strategy**:
1. Try multiple classloaders
2. Create temp file from resource stream
3. Play via AudioSystem.getLine()
4. Cleanup temp file async

## 🛠️ Building from Source

### Prerequisites
- Java 17+
- Gradle 8.5+
- IntelliJ Platform SDK (auto-downloaded)

### Build Steps

```bash
# Clone repository
git clone https://github.com/ForceGT/panic-plugin.git
cd panic-plugin

# Build plugin
./gradlew build

# Run in sandbox IDE (optional)
./gradlew runIde

# Clean build
./gradlew clean build
```

**Output**: `build/distributions/Panic Plugin-1.0.0.zip`

### Build Configuration

**gradle.properties**:
```properties
pluginSinceBuild = 241           # Min IDE version
pluginUntilBuild = 253.*         # Max IDE version (Android Studio Panda)
platformVersion = 2024.1.4       # IntelliJ SDK version
```

**Dependencies**:
- IntelliJ Platform SDK (managed by Gradle plugin)
- Kotlin stdlib (managed by Gradle)
- No external audio/UI libraries (uses built-in Java APIs)

## 🔍 Debugging Tips

### Check Plugin Initialization

Look for logs:
```
✅ [INIT] ExecutionFailureListener instantiated (application-level)
🚀 Initializing Panic Plugin
✅ ExecutionListener registered via plugin.xml
🎯 Panic Plugin initialized - Ready to scream!
```

### Check Execution Events

Look for logs when clicking Play:
```
[EXECUTION] Process scheduled: Run - mobile.app
[EXECUTION] Process starting: Run - mobile.app
[EXECUTION] Process started: Run - mobile.app
[EXECUTION] ✅ BUILD SUCCEEDED (processStarted called)
```

Or on failure:
```
[EXECUTION] Process scheduled: Run - mobile.app
[EXECUTION] Process NOT started: Run - mobile.app
[EXECUTION] 🔴 BUILD FAILED (processNotStarted called)
[EXECUTION] ✅ Panic mode triggered!
```

### Check Overlay Rendering

Look for logs:
```
[REDEMPTION OVERLAY] Got IDE frame: true, type: IdeProjectFrameHelper
[REDEMPTION OVERLAY] LayeredPane: true, size: 1401x827
[REDEMPTION COMPONENT] Loaded mission_passed.png: 320x320
[REDEMPTION COMPONENT] Drew image at (540, 165)
```

### Enable Full Logging

The plugin uses `thisLogger()` from IntelliJ:
```kotlin
private val logger = thisLogger()
logger.info("[COMPONENT] Message here")
```

View logs:
- macOS: `Help → Show Log in Finder`
- Windows: `Help → Show Log in Explorer`
- Linux: Check `~/.config/JetBrains/IdeaIC*/logs/idea.log`

## 🎓 Learning Resources

### IntelliJ Platform Documentation
- [Plugin Development Guide](https://plugins.jetbrains.com/docs/intellij/)
- [Execution Contexts](https://plugins.jetbrains.com/docs/intellij/execution-contexts.html)
- [Messaging Infrastructure](https://plugins.jetbrains.com/docs/intellij/messaging-infrastructure.html)

### Java Swing (UI)
- `javax.swing.JLayeredPane` - Z-order layer management
- `javax.swing.Timer` - Animation timing
- `javax.swing.JComponent` - Custom painting

### Java Audio
- `javax.sound.sampled.Clip` - Audio playback
- `javax.sound.sampled.AudioSystem` - Audio I/O

## 📝 Contributing

To extend the plugin:

1. **Add new failure detection**: Extend listeners in `listeners/`
2. **Add new overlays**: Create new Manager + Component in `ui/`
3. **Add new audio**: Add MP3 to `src/main/resources/audio/`
4. **Test with sandbox IDE**: `./gradlew runIde`

## 📄 License

MIT - Make your builds epic!

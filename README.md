# Panic Plugin - Viral IntelliJ/Android Studio Plugin

A hilarious IntelliJ/Android Studio plugin that brings theatrical reactions to your build failures and successes!

## 🎬 Features

### 🔴 Panic Mode (Rage Mode)
When a build or run configuration fails, the IDE responds with:
- **🌊 Pulsing Red Overlay** - Sine-wave animated red screen covering your editor
- **🎵 Scream Sound** - A digital wail of despair (`faaah.mp3`)

### 🟢 Redemption Mode (Success Mode)  
When ANY build or run configuration succeeds after a prior failure:
- **💚 GTA "Mission Passed" Overlay** - Classic GTA mission completion screen
- **🎵 Mission Passed Audio** - Iconic GTA mission completion theme
- **RESPECT +** - Your code deserves respect!

## 🎯 What Triggers Panic/Redemption?

| Trigger | Status |
|---------|--------|
| **IDE Builds** (Build menu, Gradle panel) | ✅ Supported |
| **Run Configurations** (Play button) | ✅ Supported |
| **Compilation Errors** | ✅ Panic Mode |
| **Successful Execution** | ✅ Redemption Mode |

## 📦 Installation

### Option 1: Install from ZIP (Recommended)

1. **Download** the latest `Panic Plugin-1.0.0.zip` from [Releases](https://github.com/ForceGT/panic-plugin/releases)

2. **Open IDE Preferences:**
   - Go to **Preferences** (macOS: `Cmd+,` / Windows/Linux: `Ctrl+Alt+S`)

3. **Navigate to Plugins:**
   - Search for **"Plugins"** in preferences
   - Click on **⚙️ (Settings icon)** next to "Installed" tab

4. **Install from Disk:**
   - Select **"Install Plugin from Disk..."**
   
   ![Install from Disk](docs/install-from-disk.png)

5. **Select the ZIP file:**
   - Navigate to and select `Panic Plugin-1.0.0.zip`
   - Click **"OK"**

6. **Restart IDE:**
   - Click **"Restart IDE"** when prompted
   - Plugin will be active after restart!

## 🎮 Testing

### Quick Test (No Build Required)
1. Press `Cmd+Shift+A` (macOS) or `Ctrl+Shift+A` (Windows/Linux)
2. Search **"Test Panic"** → Enter
   - Should hear **SCREAM audio** 🔊 and see red overlay
3. Search **"Test Redemption"** → Enter
   - Should hear **MISSION PASSED audio** 🎵 and see GTA overlay

### Real Build Test
1. Write a **compilation error** in your code (e.g., `val x: Int = "string"`)
2. Click **Play ▶️** to build/run
3. Hear **SCREAM**, see red overlay 🔴
4. Fix the error
5. Click **Play ▶️** again
6. Hear **MISSION PASSED**, see GTA overlay 🟢

## 💾 State Persistence

The `FaahStateService` persists your panic state across IDE sessions using IntelliJ's `PersistentStateComponent`. Your panic state is remembered even after restart!

## 📋 Requirements

- **IDE**: IntelliJ IDEA / Android Studio 2024.1 (build 241) or later
- **Java**: Java 17+
- **Audio Files**: Bundled with plugin (no external dependencies)

## 🔧 Building from Source

See [ARCHITECTURE.md](docs/ARCHITECTURE.md) for detailed build instructions and code walkthrough.

```bash
./gradlew build
```

Output: `build/distributions/Panic Plugin-1.0.0.zip`

## 🏗️ Architecture

For detailed architecture, design decisions, and code walkthrough, see [ARCHITECTURE.md](docs/ARCHITECTURE.md).

## 📄 License

MIT - Make your builds epic!

## 🙏 Credits

- **Developer**: Gaurav Thakkar
- **Scream audio**: Classic internet meme
- **Mission Passed audio**: GTA series (fan usage)
- Built with ❤️ for developers who need theatrical feedback

## 🐛 Issues & Feedback

Found a bug or have a feature request? [Open an issue](https://github.com/ForceGT/panic-plugin/issues)!

# Might and Magic 7 VR Mod (OpenEnroth)

Fork of OpenEnroth adding Virtual Reality support for **Might and Magic VII** using **OpenXR**.

## Current Status: MVP
- **3DOF tracking**: HMD orientation controls camera.
- **Stereoscopic rendering**: full 3D world for both eyes.
- **Input**: VR controllers or keyboard/mouse.
- **Engine**: OpenEnroth (modern C++ reimplementation of MM7).
- **Backend**: OpenXR (SteamVR, Oculus/Meta, WMR).

**Note**: Early development. HUD is hidden in VR to avoid artifacts. Positional tracking (6DOF) not implemented.

---

## Prerequisites
1. **Game data**: MM7 assets (`ANIMS`, `DATA`, `MUSIC`, `SOUNDS`).
2. **VR runtime**: SteamVR (recommended) or any OpenXR runtime.
3. **Build tools** (only for building):
   - **CMake** 3.20+ (VR docs) / **CMake** 3.27+ (OpenEnroth docs).
   - **C++ compiler**: VS 2019/2022 recommended for VR; minimums: VS 2022, GCC 13, AppleClang 15/Clang 15.
   - **SDL3** (window/context).

Main dependencies (OpenEnroth):
- SDL3, FFmpeg, OpenAL Soft, Zlib.

Optional:
- Python 3.x for style checks.

IDEs tested: VS 2022+, VS Code 2022+, CLion 2022+.

---

## Controls (VR)
**Left controller (locomotion)**
- Stick up/down: move forward/back.
- Stick left/right: strafe.
- Diagonal input: 360° movement.

**Right controller (view/action)**
- Stick left/right: snap/smooth turn.
- Stick up: jump.
- Trigger: interact/select.
- Grip/button: combat/cast/quick cast.

---

## Build (VR)
1. Clone:
   ```bash
   git clone https://github.com/vallewillian-source/mm7_vr.git
   cd mm7_vr
   ```
2. Configure:
   ```bash
   mkdir build
   cd build
   cmake ..
   ```
   FetchContent downloads OpenXR SDK + deps.
3. Compile:
   ```bash
   cmake --build . --config Release
   ```
4. Install: copy `OpenEnroth.exe` to your MM7 folder (where `DATA` is).

---

## Run (SteamVR)
1. Start SteamVR.
2. Run `OpenEnroth.exe`.
3. If headset is black, set SteamVR as default OpenXR runtime:
   `SteamVR Settings > Developer > Set SteamVR as OpenXR Runtime`.

Keyboard/mouse still works for standard MM7 in VR view.

---

## VR Technical Summary
### VR Subsystem (`src/Engine/VR/`)
- `VRManager` singleton for OpenXR (`XrInstance`, `XrSession`, `XrSpace`).
- Stereo swapchains (double-buffered).
- Converts OpenXR **Y-Up** (RH) to OpenEnroth **Z-Up**.
- Uses SDL3 to get `HWND`/`HGLRC` reliably on Windows 11.

### Render Pipeline Changes
**Engine loop (`src/Engine/Engine.cpp`)**
1. If VR initialized, begin OpenXR frame.
2. For each eye:
   - Acquire swapchain image.
   - Bind VR FBO.
   - `drawWorld()` with VR flags.
   - Release swapchain image.
3. `xrEndFrame`.
4. Render desktop view (debug/spectator).

**OpenGL renderer (`src/Engine/Graphics/Renderer/OpenGLRenderer.cpp`)**
- `_set_3d_modelview_matrix` and `_set_3d_projection_matrix` use `VRManager` matrices when `IsRenderingVREye()`.
- `BeginScene3D` respects currently bound VR FBO (does not reset to backbuffer).

---

## Stereo Convergence & Comfort (UI in 3D)
- To push objects farther and improve comfort, move each eye image **away from the nose** (more parallel), i.e. **decrease** `convergenceOffset` (bigger divisor, e.g. `w / 5.0`).
- Moving images **toward the nose** increases strain (feels too close).
- VR lenses focus at fixed distance (~2m). Convergence should simulate this to avoid nausea.
- Angular size matters: large + parallel = far (cinema); small + crossed = near (phone).
- World-locked full-screen UIs are more comfortable than head-locked.

---

## Build System (`CMakeLists.txt`)
- `FetchContent` downloads OpenXR SDK.
- Static library target `engine_vr` encapsulates VR logic.

---

## Mandatory Rules for Vibe Coding Models
- Never compile automatically generated code.
- Always use explicit casts (no implicit casts).
- Never use `dynamic_cast`.
- Always use `static_cast` for type conversion.
- Never use `reinterpret_cast` for type conversion.

---

# Might and Magic Trilogy (Original OpenEnroth Documentation)

## Development HOWTO
This section describes the development process and is required reading for contributors.

---

## Dependencies
Main: SDL3, FFmpeg, OpenAL Soft, Zlib.  
Prebuilt dependencies are used by default and resolved during CMake.

Additional:
- CMake 3.27+.
- Python 3.x (optional, style checks).

Minimum compilers:
- VS 2022, GCC 13, AppleClang 15 / Clang 15.

IDEs tested:
- Visual Studio 2022+,
- Visual Studio Code 2022+,
- CLion 2022+.

---

## Building on Windows
- Install Git and Visual Studio 2022.
- Windows SDK v10.0.20348.0+.
- Clone/fork `https://github.com/vallewillian-source/open-enroth-vr`.
- CMake: standalone or VS-provided; add to PATH if needed.
- Open folder in VS, pick x32/x64, wait for CMake, set startup `OpenEnroth.exe`, run.
- To disable prebuilt deps: `OE_USE_PREBUILT_DEPENDENCIES=OFF` and provide your own (e.g. vcpkg).

**Note**: VS may not sync submodules across branches; run `git submodule update --init` if needed.

---

## Coding Style
Based on Google C++ Style Guide. PRs fail on style issues.

**Style checks**: build `check_style` target (VS: Solution Explorer -> Change Views -> CMake targets).

Documentation:
- Doxygen format with `/**` and `@` tags.
- English.
- Preserve original function offsets; move to `@offset` tag if possible.

Naming:
- Macros: `MM_` prefix, `SNAKE_CASE_ALL_CAPS`.
- `enum` values: `SNAKE_CASE_ALL_CAPS`, prefixed by type (e.g. `MONSTER_TROLL_A`).
- Enum bounds: `_FIRST`, `_LAST`, `_COUNT` after type name (e.g. `ITEM_FIRST_MESSAGE_SCROLL`).
- `CamelCase` for types. Methods/variables start lowercase.
- Private members start with `_` (except POD-like types).
- STL-compatible interfaces use STL naming (`value_type`, `push_back`).

Formatting:
- `char *string` (space before `*`/`&`).

Language features:
- `using Alias = Type` (no `typedef`).
- Prefer `enum class` + `using enum`. Use `Flags` for flags.
- Avoid `unsigned` unless needed; prefer `int`. Use `size_t` for STL indices.
- String params: `std::string_view` by value; use `fmt::format` or `join`.
- Avoid namespaces in general; allow `detail` or small groupings (`lod`).
- Strings are UTF-8; on Windows via `UnicodeCrt`. `path.string()` is OK.

Error handling:
- `assert` for invariants.
- Exceptions (e.g. `Exception`) for non-recoverable errors.
- `Logger` for warnings/recoverable errors.

---

## Code Organization
- `thirdparty`: external libs.
- `Utility`: generic utilities (depends only on `thirdparty`).
- `Library`: independent libs built on `Utility`.
- `Library/Platform`: SDL platform abstraction.
- 1 `CMakeLists.txt` per folder (except `/android`, `/CMakeModules`, `/resources`).
- 1 class per source file (exceptions: small structs/helper classes; function-only files).

---

## Testing
Policy: add tests for fixable bugs when possible.

**Unit tests**: Google Test. Examples in `src/Utility/Tests`.  
Run: build `OpenEnroth_UnitTest`, execute `<build-dir>/test/Bin/UnitTest/OpenEnroth_UnitTest`.

**Game tests**: instrumented engine; events injected between frames; assets required.
Workflow:
1. Fix bug.
2. Load save that reproduces it.
3. `Ctrl+Shift+R` start trace recording.
4. Reproduce.
5. `Ctrl+Shift+R` stop -> `trace.json`, `trace.mm7`.
6. Rename and PR to `OpenEnroth_TestData`.
7. Update reference tag in `test/Bin/CMakeLists.txt`.
8. Add test using `TestController::playTraceFromTestData`.

Other notes:
- For non-standard FPS in trace recording, set `debug.trace_frame_time_ms`.
- Run game tests headless: set `OPENENROTH_MM7_PATH`, build `Run_GameTest_Headless_Parallel`.
- Or run `OpenEnroth_GameTest` with args; `--headless` supported.
- Use `--gtest_filter=<suite>.<name>` (must include `=`).
- Replay trace: `OpenEnroth play --speed 0.5 <trace.json>`.

### Random state desynchronized
If intentional logic changes break traces:
1. Fork/clone `OpenEnroth_TestData`, clean branch.
2. `OpenEnroth retrace <trace.json>` (can pass multiple).
3. Commit/PR to TestData.
4. Update `GIT_TAG` and `GIT_REPOSITORY` in `test/Bin/CMakeLists.txt`.
5. Commit/push main PR.

---

## Scripting (Lua)
Scripts in `resources/scripts`.

**Lua Language Server**:
- Install `LuaLS` and ensure it is in PATH.
- Generate project; `check_style` includes scripts if LuaLS is found.
- If LuaLS missing, build still works (no script checks).

Tools:
- VS Code + LuaLS extension recommended.

Modding:
- Not planned soon; see milestones.

---

## Console Commands
In-game debug console:
1. Launch game.
2. Load/create game.
3. Press `~`.
Console only available in-game.

---

## Additional Resources
Old event decompiler + IDB files:  
https://www.dropbox.com/sh/if4u3lphn633oit/AADUYMxNcrkAU6epJ50RskyXa?dl=0  
Contact: `zipi#6029` on Discord.

## Support
Discord: https://discord.gg/jRCyPtq

## End of Enroth original flat Documentation

---

# House System in OpenEnroth (MM7 VR)

## 1. Entry Flow (3D -> 2D)
An event in `EvtInterpreter.cpp` (opcode `EVENT_SpeakInHouse`) calls `enterHouse(houseId)`.

**Validation and setup** (`src/GUI/UI/UIHouses.cpp`, `enterHouse(HouseId uHouseID)`):
- clears the message queue and status bar;
- validates opening hours (`houseTable`);
- checks for bans;
- prepares the NPC list via `prepareHouse()`;
- returns `true` if entry is allowed.

If allowed, `createHouseUI(houseId)` creates a specific `GUIWindow_House` and stores it in `window_SpeakInHouse`.
In the `GUIWindow_House` constructor:
- `current_screen_type = SCREEN_HOUSE`;
- `pEventTimer->setPaused(true)` pauses game time;
- loads the 2D background and NPC buttons.

## 2. 2D Interface
Static background, NPC portraits at the bottom, right-side menu (buy/sell/train), and dialog area.

## 3. Exit Flow (2D -> 3D)
`houseDialogPressEscape()` (`src/GUI/UI/UIHouses.cpp`):
- if in a submenu, returns to the main house menu;
- if on the main menu, clears `currentHouseNpc`, releases `pDialogueWindow`, returns `false`.

On exit: `window_SpeakInHouse->Release()`, `current_screen_type = SCREEN_GAME`, game time resumes.

## 4. 2D Screen in VR (Virtual Monitor)
**Rendering**:
1. **Capture**: `Engine.cpp` calls `CaptureScreenToOverlay`, using `glBlitFramebuffer` into `m_overlayTexture` (VRManager).
2. **World-lock**: when `SCREEN_HOUSE` is detected, `m_debugHouseIndicator` is enabled. On first run, it captures the HMD pose and places the screen **2.5m** forward (`VRManager.cpp:L570`).
3. **Stereo projection**: per eye, it projects to NDC, uses `glScissor`, and draws the texture with an orthographic projection (`VRManager.cpp:L606-612`).

**State control**:
- Entry: `SetDebugHouseIndicator(true)` resets `m_housePoseInitialized`.
- Exit: when returning to `SCREEN_GAME`, the indicator is disabled.

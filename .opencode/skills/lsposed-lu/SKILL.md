---
name: lsposed-lu
description: LSPosed-Lu fork — a Zygisk-based ART hooking framework providing Xposed-compatible APIs via LSPlant. Use when: working on Xposed module hooks, native ART hooking, Zygisk injection, daemon service, manager app, AIDL IPC, DEX obfuscation, or any LSPosed internals.
---

## Core Architecture

Multi-process Android framework: **magisk-loader** (Zygisk module) → **core** (hook engine) → **daemon** (background service) → **app** (manager UI).

### Process Flow
1. ZygiskNext loads `magisk-loader` into zygote (`zygisk_main.cpp:ZygiskModule`)
2. Pre-fork: `MagiskLoader::OnNativeForkSystemServerPre` / `OnNativeForkAndSpecializePre` — connects to daemon
3. Post-fork: Loads LSPosed DEX via `InMemoryDexClassLoader`, calls `Startup.initXposed()` → `Startup.bootstrapXposed()`
4. LSPlant initializes ART hooking; modules loaded via `LspModuleClassLoader`
5. Hook dispatch: `HookBridge` (native) → `LSPosedBridge.NativeHooker` → legacy `XC_MethodHook` or modern `XposedModule` callbacks

### Key Modules (Gradle subprojects)

| Module | Role | Language |
|--------|------|----------|
| `app` | Manager UI (activities, fragments, dialogs) | Java |
| `core` | Xposed API + hook engine + native bridge | Java + C++ |
| `daemon` | Background config/module/log service | Java + C++ |
| `magisk-loader` | Zygisk injection entry point | Java + C++ |
| `dex2oat` | dex2oat wrapper | C |
| `hiddenapi/stubs` | Android hidden API stubs for compilation | Java |
| `hiddenapi/bridge` | Runtime bridge to hidden APIs (`HiddenApiBridge`) | Java |
| `libxposed/api` | Modern Xposed API (`io.github.libxposed.api`) | Java |
| `libxposed/service` | Modern Xposed service API | Java |
| `libxposed-compat` | Legacy Xposed API compatibility | Java |
| `services/manager-service` | AIDL definitions for manager↔daemon | AIDL |
| `services/daemon-service` | AIDL definitions for daemon↔injected process | AIDL |

## Key Patterns & Gotchas

### Dual API Support
- **Legacy API**: `de.robv.android.xposed.*` — `XposedBridge.hookMethod()`, `XC_MethodHook`, `XposedHelpers`
- **Modern API**: `io.github.libxposed.api.*` — `XposedModule`, `XposedInterface`, annotation-based hookers (`@BeforeInvocation`, `@AfterInvocation`)
- `XposedBridge.LegacyApiSupport` synchronizes state between old and new callback systems

### DEX Obfuscation
- `ObfuscationManager` (daemon) randomizes class names in module DEXes before injection
- Native `obfuscation.cpp` uses `dex::Reader`/`dex::Writer` (slicer) to replace signatures: `Lde/robv/android/xposed/` → random string
- Obfuscation map shared via `ILSPApplicationService` with transaction code `724533732`
- Entry class name resolved dynamically: `ConfigBridge.obfuscation_map().at("org.lsposed.lspd.core.") + "Main"`

### Bridge Service (IPC Discovery)
- `BridgeService` registers LSPosed daemon with `activity` system service
- Custom transaction code: `('_' << 24) | ('L' << 16) | ('S' << 8) | 'P'` = `1598837584`
- Handles system server restart/die events; re-registers on restart

### Parasitic Manager
- Manager UI runs inside `com.android.shell` (UID 2000) process — not a standalone app
- `ParasiticManagerHooker` / `ParasiticManagerSystemHooker` intercept manager package loading
- Default manager package: `org.lsposed.manager` (configurable in `build.gradle.kts`)

### Native Hooking Chain
- `HookBridge.hookMethod()` → JNI → `hook_bridge.cpp` → LSPlant `lsplant::Hook()`
- `ResourcesHook` — makes Resources/TypedArray inheritable, enables XResources
- `NativeAPI.recordNativeEntrypoint()` — tracks native module entry points
- Dobby used for inline hooking at native level

### Access Control
- `service.cpp:IPCThreadState` resolves calling UID from libbinder for security checks
- `access_matrix` bitmap controls which modules can hook which processes
- UID ranges: isolated (99000-99999), app zygote isolated (90000-98999)

## Build Configuration

| Setting | Value |
|---------|-------|
| compileSdk | 35 |
| minSdk | 27 (Android 8.1) |
| targetSdk | 35 |
| Java | 21 |
| NDK | 28.0.12433566 |
| CMake | 3.28.0+ |
| ABIs | arm64-v8a, armeabi-v7a, x86, x86_64 |
| Version | git commit count (dev branch) + 4200 offset |
| STL | `c++_static` (`-DANDROID_STL=none` in root build) |

- Build: `./gradlew assembleRelease`
- Injected package: `com.android.shell` UID 2000

## Reference Files

| File | Content | When to load |
|------|---------|--------------|
| `reference/module-architecture.md` | Detailed module dependency map, service relationships, data flow | Understanding inter-module communication |
| `reference/hook-system.md` | Hook engine internals — native bridge, LSPlant integration, callback dispatch | Modifying hook behavior or adding new hook types |
| `reference/daemon-services.md` | Daemon service details — ConfigManager, AIDL interfaces, BridgeService | Working on daemon, config, IPC |
| `reference/native-layer.md` | JNI/C++ layer — CMake structure, context lifecycle, symbol cache | Modifying native code or CMake |
| `reference/api-compat.md` | Legacy vs Modern API mapping, module loading, XposedHelpers patterns | Writing or modifying Xposed modules/APIs |

## Key Workflows

### Adding a new hook in core
1. Add hooker class in `core/src/main/java/org/lsposed/lspd/hooker/`
2. Register in `core/src/main/java/org/lsposed/lspd/impl/LSPosedContext.java` or relevant initialization path
3. If native needed: add JNI method in `core/src/main/jni/src/jni/`, register in `hook_bridge.cpp` or new file

### Adding a new AIDL method
1. Define in `services/daemon-service/src/main/aidl/` or `services/manager-service/src/main/aidl/`
2. Implement in corresponding daemon service class (`LSPosedService`, `LSPManagerService`, `LSPApplicationService`)
3. Rebuild to generate AIDL stubs

### Adding a new obfuscation signature
1. Add entry to `signatures` map in `daemon/src/main/jni/obfuscation.cpp`
2. Signature format: `Lpackage/path/` (JNI style with trailing slash)
3. Rebuild daemon native libs

### Debugging hook failures
1. Check `LSPosed-Bridge` log tag
2. LSPlant init status in `context.cpp:InitArtHooker`
3. HookBridge returns `false` when LSPlant hook fails — check `hook_bridge.cpp` for details
4. Verify target method is not abstract/proxy/inlined — use `deoptimizeMethod()` for inlined callers

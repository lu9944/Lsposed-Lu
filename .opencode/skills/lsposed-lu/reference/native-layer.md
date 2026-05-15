# Native Layer (JNI/C++)

## CMake Structure

### Root config (external/CMakeLists.txt)
Shared external dependencies: LSPlant, Dobby, etc.

### Module-specific CMakeLists
| Path | Builds |
|------|--------|
| `core/src/main/jni/CMakeLists.txt` | `liblspd.so` — hook bridge, resources hook, native API, dex parser |
| `daemon/src/main/jni/CMakeLists.txt` | `libdaemon.so` — obfuscation, logcat, dex2oat |
| `magisk-loader/src/main/jni/CMakeLists.txt` | `liblsposed.so` — Zygisk entry, magisk loader, service |
| `dex2oat/src/main/cpp/CMakeLists.txt` | `libdex2oat.so` — dex2oat wrapper |

### CMake Variables (from root build.gradle.kts)
```
-DEXTERNAL_ROOT=<root>/external
-DCORE_ROOT=<root>/core/src/main/jni
-DANDROID_STL=none
-DINJECTED_AID=2000
-DDEBUG_SYMBOLS_PATH=<build>/symbols (release only)
```

## Context Lifecycle (core/.../context.h, context.cpp)

### Context Base Class
```cpp
class Context {
    static std::unique_ptr<Context> instance_;
    jobject inject_class_loader_;
    jclass entry_class_;
    
    virtual void LoadDex(JNIEnv*, PreloadedDex&&) = 0;
    virtual void SetupEntryClass(JNIEnv*) = 0;
    void InitArtHooker(JNIEnv*, const lsplant::InitInfo&);
    void InitHooks(JNIEnv*);
};
```

### MagiskLoader (extends Context)
- `LoadDex()` — creates `InMemoryDexClassLoader` with DEX buffer
- `SetupEntryClass()` — finds obfuscated entry class name
- `OnNativeForkSystemServerPre/Post()` — system server injection
- `OnNativeForkAndSpecializePre/Post()` — app process injection

### PreloadedDex
- Memory-mapped DEX file via `mmap(PROT_READ, MAP_SHARED)`
- Auto-unmapped on destruction
- Used to pass DEX data from native to Java without file I/O

## Service Layer (magisk-loader/.../service.cpp)

### Service Class (singleton)
- Manages connection to daemon process
- `RequestLSPosedService()` — gets daemon binder via native service manager
- `RequestSystemServerBinder()` — gets system_server hook binder
- `FetchDex()` — retrieves LSPosed DEX from daemon
- `GetObfuscationMap()` — retrieves class name obfuscation map

### IPCThreadState
- Dynamically resolves `IPCThreadState::selfOrNull()` and `getCallingUid()` from libbinder
- Used for UID-based access control in hook callbacks
- Falls back gracefully if symbols not found

### Access Matrix
- `access_matrix` — bitmap controlling which modules can access which processes
- Set during service initialization based on module scope configuration

## Symbol Cache (core/.../symbol_cache.cpp)
- Caches frequently used JNI class/method IDs
- Initialized during `InitArtHooker` / `InitHooks`
- Uses `lsplant::ScopedLocalRef` for automatic local reference management

## Obfuscation (daemon/.../obfuscation.cpp)

### Signature Replacement
```cpp
std::map<const std::string, std::string> signatures = {
    {"Lde/robv/android/xposed/", ""},      // randomized
    {"Landroid/app/AndroidApp", ""},       // randomized
    {"Landroid/content/res/XRes", ""},     // randomized
    {"Landroid/content/res/XModule", ""},  // randomized
    {"Lorg/lsposed/lspd/core/", ""},       // randomized
    {"Lorg/lsposed/lspd/nativebridge/", ""}, // randomized
    {"Lorg/lsposed/lspd/service/", ""},    // randomized
};
```

### Algorithm
1. For each signature, generate random replacement of same length
2. Use `dex::Reader` to parse DEX, iterate all strings
3. `strstr()` find matching signatures, `memcpy()` replace in-place
4. `dex::Writer` serializes modified DEX
5. Output via `SharedMemory` fd

### Custom Allocator (WA class)
Single-allocation writer allocator — creates fd-based shared memory for output DEX.

## Hook Bridge Internals (core/.../hook_bridge.cpp)

### Registration
```
RegisterHookBridge(env) — registers native methods for HookBridge.java
```

### Hook Method Registration
```
HookMethod(env, useModernApi, method, hookerClass, priority, callback)
  → Creates HookItem for method (thread-safe via shared_mutex)
  → Calls lsplant::Hook() with NativeHooker as trampoline target
```

### Callback Types
- Legacy: `jobject` callbacks stored by priority (descending)
- Modern: `ModuleCallback` (jmethodID pairs for before/after) stored by priority
- Both types coexist per hooked method

## Logging (logging.h)
- Uses `fmtlib` for formatting: `LOGD(fmt, args...)`, `LOGE(fmt, args...)`, `PLOGE(fmt, args...)`
- Tag: varies by module (`LSPosed`, `LSPosed-Bridge`, `LSPosedService`)
- Android `__android_log_print` backend

## Utility Headers
- `utils.h` / `utils/jni_helper.hpp` — RAII JNI helpers (`ScopedLocalRef`, `JNI_FindClass`, etc.)
- `elf_util.h` — ELF symbol resolution for dynamic loading
- `native_util.h` — common JNI utility functions
- `macros.h` — platform macros, `LP_SELECT()` for 32/64-bit selection

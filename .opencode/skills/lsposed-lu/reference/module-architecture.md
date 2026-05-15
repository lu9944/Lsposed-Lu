# Module Architecture & Dependency Map

## Gradle Subproject Dependencies

```
app → services:manager-service (AIDL stubs)
app → hiddenapi:bridge
app → hiddenapi:stubs
app → libxposed:service

core → hiddenapi:bridge
core → hiddenapi:stubs
core → libxposed:api
core → libxposed:service

daemon → services:daemon-service (AIDL stubs)
daemon → hiddenapi:bridge
daemon → hiddenapi:stubs
daemon → libxposed:service

magisk-loader → core (shares native code)
magisk-loader → hiddenapi:stubs

dex2oat → (standalone C)
```

## Process Architecture

### Zygote Process (injected by ZygiskNext)
```
ZygiskModule::onLoad()
  → MagiskLoader::Init()
  → ConfigImpl::Init()

ZygiskModule::preAppSpecialize()
  → MagiskLoader::OnNativeForkAndSpecializePre()
    → Service::InitService() — connects to daemon via binder

ZygiskModule::postAppSpecialize()
  → MagiskLoader::OnNativeForkAndSpecializePost()
    → LoadDex() — InMemoryDexClassLoader with LSPosed DEX
    → SetupEntryClass() — finds org.lsposed.lspd.core.Main
    → FindAndCall("forkCommon") → Java Main.forkCommon()
      → Startup.initXposed() — init LSPlant, native bridges
      → Startup.bootstrapXposed() — load Xposed modules
```

### Daemon Process (org.lsposed.daemon)
```
Main.main() → ServiceManager.start()
  → ConfigFileManager.tryLock() — single instance
  → LogcatService.start()
  → Create: LSPosedService, LSPApplicationService, LSPManagerService, LSPSystemServerService
  → Dex2OatService.start() (Android Q+)
  → systemServerService.putBinderForSystemServer()
  → ConfigManager.getInstance() — loads config BEFORE package service
  → ActivityThread.systemMain()
  → waitSystemService("package", "activity", USER_SERVICE, APP_OPS_SERVICE)
  → BridgeService.send() — register with activity service
  → Looper.loop()
```

### Manager Process (parasitic — inside com.android.shell)
```
Main.forkCommon() → detects manager package name
  → ParasiticManagerHooker.start()
    → Hooks ActivityThread to inject manager resources/classes
```

## Service Communication Map

```
┌─────────────┐    ILSPosedService     ┌──────────────┐
│  magisk-    │◄──────────────────────►│   daemon     │
│  loader     │    (via BridgeService) │              │
│  (zygote)   │                        │  LSPosedService
└──────┬──────┘                        │  LSPManagerService
       │                               │  LSPApplicationService
       │ ILSPApplicationService        │  LSPSystemServerService
       │ (per-process binder)          │  ConfigManager
       ▼                               │  Dex2OatService
┌──────────────┐                       │  LogcatService
│  core        │                       └──────┬───────┘
│  (injected   │                              │
│   into app)  │                              │ ILSPManagerService
└──────────────┘                              ▼
                                       ┌──────────────┐
                                       │  app          │
                                       │  (manager UI) │
                                       └──────────────┘
```

## Key Data Stores
- **ConfigManager** — SQLite database for module configs, scope, enabled state
- **ConfigFileManager** — File-based config, handles module APK caching
- **ObfuscationManager** — Runtime signature obfuscation map (in-memory, generated per boot)
- **LogcatService** — Verbose log + module log buffers (rotating files)

## AIDL Interface Summary

| Interface | Purpose | Key Methods |
|-----------|---------|-------------|
| `ILSPosedService` | Main daemon ↔ zygote | `requestApplicationService()`, `dispatchSystemServerContext()` |
| `ILSPApplicationService` | Per-app process service | `getModulesList()`, `getModuleDex()`, `getObfuscationMap()` |
| `ILSPManagerService` | Manager UI ↔ daemon | `enableModule()`, `setModuleScope()`, `getInstalledPackagesFromAllUsers()` |
| `ILSPSystemServerService` | System server injection | `putBinderForSystemServer()` |
| `ILSPInjectedModuleService` | Per-module in injected process | Module preferences, resources |
| `ILSPModuleService` | Module service | Module-side operations |

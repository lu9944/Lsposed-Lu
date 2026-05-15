# Daemon Services

## ServiceManager (daemon/.../service/ServiceManager.java)
Entry point for daemon. Creates and wires all services.

### Startup Sequence
1. `ConfigFileManager.tryLock()` — ensure single instance
2. Parse args: `--from-service`, `--system-server-max-retry=N`
3. `LogcatService` — start verbose logging
4. `permissionManagerWorkaround()` (Android R+) — preload permission manager proxies
5. Create services: `LSPosedService`, `LSPApplicationService`, `LSPManagerService`, `LSPSystemServerService`, `Dex2OatService`
6. `systemServerService.putBinderForSystemServer()` — stash binder for system_server
7. `ConfigManager.getInstance()` — load config BEFORE package service starts
8. `ActivityThread.systemMain()` — init Android framework
9. Wait for: `package`, `activity`, `USER_SERVICE`, `APP_OPS_SERVICE`
10. `BridgeService.send()` — register daemon with activity service
11. `Looper.loop()`

### Global Namespace
- `toGlobalNamespace()` prepends `/proc/1/root/` for accessing files in init's mount namespace
- Used when daemon needs to access files visible only in global namespace

## ConfigManager (daemon/.../service/ConfigManager.java)
~1240 lines. SQLite-based configuration store.

### Key Responsibilities
- Module enable/disable state
- Module scope (which apps a module hooks)
- Per-user module configuration
- Package change monitoring
- Module APK caching (copies to data dir for reliable loading)
- DEX obfuscation dispatch

### Important: Singleton Timing
ConfigManager must be instantiated BEFORE `ActivityThread.systemMain()`. After system services start, calling `getInstance()` triggers module/scope cache which requires package service — may deadlock during startup.

## LSPosedService (daemon/.../service/LSPosedService.java)
Main daemon service. `ILSPosedService.Stub` implementation.

### Key Methods
- `requestApplicationService(uid, pid, processName, heartBeat)` — called by each injected process to get per-process service
- `dispatchSystemServerContext(activityThread, activityToken, api)` — called by system_server hook
- `preStartManager()` — prepare parasitic manager environment
- `setManagerEnabled(boolean)` — enable/disable manager

### Module Detection
- Modern modules: APK contains `META-INF/xposed/java_init.list`
- Legacy modules: APK contains `assets/xposed_init`

## LSPApplicationService (daemon/.../service/LSPApplicationService.java)
Per-app-process service. `ILSPApplicationService.Stub`.

### Transaction Codes (custom parcel protocol)
- `DEX_TRANSACTION_CODE = 1310096052` — request module DEX
- `OBFUSCATION_MAP_TRANSACTION_CODE = 724533732` — request obfuscation map

### Process Tracking
- Maintains `Map<Pair<uid, pid>, ProcessInfo>` for all injected processes
- Each process registers a `heartBeat` binder with death recipient for cleanup

## LSPManagerService (daemon/.../service/LSPManagerService.java)
Manager UI backend. `ILSPManagerService.Stub`.

### Manager Process Guard
- `ManagerGuard` tracks manager process lifecycle via death recipient
- Ensures only one manager instance active
- Uses `IServiceConnection` to bind manager service

## LSPSystemServerService
Handles system_server injection lifecycle.
- `putBinderForSystemServer()` — stash daemon binder for system_server
- `maybeRetryInject()` — retry injection if initial attempt fails
- `systemServerRequested()` — check if system_server has connected

## BridgeService (daemon/.../service/BridgeService.java)
Registers LSPosed daemon as a hidden service on the `activity` system service.

### Protocol
- Transaction code: `1598837584` (magic: `_LSP`)
- Descriptor: `"LSPosed"`
- Actions: `ACTION_SEND_BINDER`, `ACTION_GET_BINDER`
- Listener callbacks: `onSystemServerRestarted`, `onResponseFromBridgeService`, `onSystemServerDied`

### System Restart Handling
- On system server death: clears `ServiceManager.sServiceManager`, `sCache`, `ActivityManager.IActivityManagerSingleton`
- Re-registers with new system services after restart

## Dex2OatService
Wraps `dex2oat` binary for compilation optimization of module DEXes.
- Only available on Android Q+ (API 29+)
- Uses native `dex2oat.c` for the wrapper logic

## LogcatService
- Two log streams: verbose log (full logcat) and modules log (filtered)
- Rotating file-based storage
- `startVerbose()` / `stopVerbose()` controlled by `ConfigManager.verboseLog()`

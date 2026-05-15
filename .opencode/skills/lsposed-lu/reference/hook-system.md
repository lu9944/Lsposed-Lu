# Hook System Internals

## Native Hook Chain

### Java → JNI Flow
```
XposedBridge.hookMethod(Member, XC_MethodHook)
  → HookBridge.hookMethod(false, Executable, NativeHooker.class, priority, callback) [native]
    → hook_bridge.cpp: HookMethod()
      → lsplant::Hook(env, method, backup, hooker, trampoline)
        → ART method entry point replacement
```

### Callback Dispatch
```
LSPlant trampoline fires → hook_bridge.cpp: HookCallback()
  → Creates HookItem with backup method
  → Dispatches to registered callbacks:
    - Modern API: ModuleCallback.before_method / after_method (reflection)
    - Legacy API: XC_MethodHook.beforeHookedMethod / afterHookedMethod
  → Priority ordering (higher priority first)
```

### HookItem Structure (hook_bridge.cpp)
```cpp
struct HookItem {
    std::multimap<jint, jobject, std::greater<>> legacy_callbacks;
    std::multimap<jint, ModuleCallback, std::greater<>> modern_callbacks;
    std::atomic<jobject> backup;  // backup method, lock-free
};
```

## Modern API Hook Flow

### Annotation-based Hookers
1. Module declares hooker class with `@XposedHooker`
2. `@BeforeInvocation` / `@AfterInvocation` annotated static methods
3. `LSPosedContext.hookMethod()` creates `HookerCallback` from annotations
4. `HookBridge.hookMethod(true, ...)` — `useModernApi=true`
5. Native dispatches to `ModuleCallback.before_method` / `after_method` via JNI reflection

### XposedModule Lifecycle
```
XposedModule(base) constructor
  → onSystemServerLoaded() [system server only]
  → onPackageLoaded() [each app load]
```

## Legacy API Hook Flow

### XC_MethodHook Callbacks
```
LSPosedBridge.NativeHooker (native callback target)
  → XposedBridge.LegacyApiSupport.handleBefore()
    → Iterates snapshot of XC_MethodHook callbacks by priority
    → Each callback.beforeHookedMethod(param)
    → If param.returnEarly → skip remaining before + matching after callbacks
  → LSPosedBridge.NativeHooker calls original or returns result
  → XposedBridge.LegacyApiSupport.handleAfter()
    → Reverse iteration of before callbacks
    → Each callback.afterHookedMethod(param)
```

### MethodHookParam Synchronization
`LegacyApiSupport.syncronizeApi()` copies state between:
- `LSPosedHookCallback` (modern internal) ↔ `XC_MethodHook.MethodHookParam` (legacy)
- Fields: method, thisObject, args, result, throwable, returnEarly/isSkipped

## Key Native Bridges

### HookBridge (core/.../nativebridge/HookBridge.java)
| Method | Purpose |
|--------|---------|
| `hookMethod(useModernApi, hookMethod, hooker, priority, callback)` | Install hook via LSPlant |
| `unhookMethod(useModernApi, hookMethod, callback)` | Remove hook |
| `deoptimizeMethod(method)` | Prevent inlining of method |
| `invokeOriginalMethod(method, thisObject, args)` | Call pre-hook method |
| `allocateObject(clazz)` | Allocate without constructor |
| `callbackSnapshot(hooker_class, method)` | Get all callbacks for a method |

### ResourcesHook (core/.../nativebridge/ResourcesHook.java)
| Method | Purpose |
|--------|---------|
| `initXResourcesNative()` | Initialize resource hooking |
| `makeInheritable(clazz)` | Make class non-final for XResources subclassing |
| `buildDummyClassLoader(parent, resSuper, taSuper)` | Create classloader with XResources superclasses |
| `rewriteXmlReferencesNative(parserPtr, origRes, repRes)` | Rewrite XML resource references |

## Deoptimization

### Why needed
ART inlines method calls; if caller is inlined, hooking callee won't fire.

### How
- `XposedBridge.deoptimizeMethod(Member)` → `HookBridge.deoptimizeMethod()`
- Forces interpreter-only execution for the method
- `InlinedMethodCallers` / `PrebuiltMethodsDeopter` handle boot-time deoptimization
- Applied to key framework methods during system server hooking

## Thread Safety
- `CopyOnWriteArraySet` for `sLoadedPackageCallbacks` and `sInitPackageResourcesCallbacks`
- `CopyOnWriteSortedSet` for per-method hook callbacks (legacy)
- `std::shared_mutex` in `hook_bridge.cpp` for HookItem access
- `std::atomic` for backup method pointer (lock-free)
- `ConcurrentHashMap` for reflection caches in `XposedHelpers`

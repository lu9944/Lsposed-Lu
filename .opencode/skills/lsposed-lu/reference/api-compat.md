# API Compatibility: Legacy vs Modern

## API Package Mapping

| Legacy (de.robv.android.xposed) | Modern (io.github.libxposed.api) | Purpose |
|---|---|---|
| `XposedBridge` | `XposedInterface` | Hook registration, logging |
| `XC_MethodHook` | `@XposedHooker` + `@BeforeInvocation`/`@AfterInvocation` | Method hook callbacks |
| `XC_MethodHook.MethodHookParam` | `BeforeHookCallback`/`AfterHookCallback` | Hook parameter/result |
| `XC_MethodReplacement` | Return value in `@BeforeInvocation` | Replace method entirely |
| `XposedHelpers` | `XposedInterface` helper methods | Reflection utilities |
| `IXposedHookLoadPackage` | `XposedModule.onPackageLoaded()` | App load callback |
| `IXposedHookZygoteInit` | `XposedModule.onSystemServerLoaded()` | System server init |
| `IXposedHookInitPackageResources` | — (not in modern) | Resource hooks |
| `XSharedPreferences` | `XposedInterface.getRemotePreference()` | Module preferences |
| `XposedInit` | `XposedModule` base class | Module initialization |

## Module Loading

### Legacy Module Detection
1. Check `assets/xposed_init` file in APK
2. Load class names from file
3. Instantiate classes, call `handleLoadPackage()` / `initZygote()`

### Modern Module Detection
1. Check `META-INF/xposed/java_init.list` in APK
2. Load entry class names from file
3. Instantiate as `XposedModule` subclass
4. Lifecycle: constructor → `onSystemServerLoaded()` / `onPackageLoaded()`

### LspModuleClassLoader
Custom classloader (`core/.../util/LspModuleClassLoader.java`) that:
- Loads module APK classes
- Isolates module class loading from app classloader
- Supports native library loading via `NativeAPI.recordNativeEntrypoint()`

## Hook Registration Patterns

### Legacy Pattern
```java
// In IXposedHookLoadPackage implementation
public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
    XposedHelpers.findAndHookMethod(
        "com.example.TargetClass",
        lpparam.classLoader,
        "targetMethod",
        String.class,
        new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) {
                param.args[0] = "modified";
            }
        }
    );
}
```

### Modern Pattern
```java
@XposedHooker
class MyHooker implements Hooker {
    @BeforeInvocation
    public static void before(BeforeHookCallback callback) {
        callback.args[0] = "modified";
    }
    
    @AfterInvocation
    public static void after(AfterHookCallback callback) {
        // post-processing
    }
}

// In XposedModule subclass
@Override
public void onPackageLoaded(PackageLoadedParam param) {
    hookMethod(method, MyHooker.class);
}
```

## XposedHelpers Key Methods

### Method Finding
- `findMethodExact(className, classLoader, methodName, paramTypes)` — exact match
- `findAndHookMethod(className, classLoader, methodName, paramTypes..., callback)` — find + hook
- `findAndHookConstructor(className, classLoader, paramTypes..., callback)` — constructor hook

### Field Access
- `getObjectField(obj, fieldName)` / `setObjectField(obj, fieldName, value)`
- `getStaticObjectField(clazz, fieldName)` / `setStaticObjectField(clazz, fieldName, value)`
- All variants for primitives: `getIntField`, `getBooleanField`, etc.

### Reflection Cache
- Uses `ConcurrentHashMap` with custom `MemberCacheKey` for performance
- `MemberCacheKey` uses structural comparison of reflection objects, not string keys
- Avoids `HashMap` performance traps from string-based keys

### Additional Instance Fields
- `setAdditionalObjectField(obj, key, value)` — attach arbitrary data to any object
- Uses `WeakHashMap<Object, HashMap<String, Object>>` — auto-cleanup with GC
- Thread-safe via synchronized access

## Resources Hooking

### XResources System
- `XposedBridge.initXResources()` — bootstrap resource hooks
- Creates dummy ClassLoader with `XResourcesSuperClass` / `XTypedArraySuperClass`
- `ResourcesHook.rewriteXmlReferencesNative()` — native XML reference rewriting
- `XposedInit.disableResources` — flag to disable if init fails

### Limitations
- Resource hooking only works for app resources, not framework resources
- May fail on ZUI devices (Lenovo) — handled with fake ActivityThread workaround

## Callback Priority
- `XCallback.PRIORITY_DEFAULT` = 50
- Higher values run first in before, last in after (reverse order)
- `XposedBridge.hookAllMethods()` — hook all overloads
- `XC_MethodHook.Unhook` — returned from hookMethod, call `.unhook()` to remove

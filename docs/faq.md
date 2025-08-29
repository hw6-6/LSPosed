# LSPosed 常见问题

本文档收集了使用 LSPosed 过程中的常见问题和解决方案。

## 安装相关问题

### Q1: 安装后没有通知或无法打开管理器

**症状**：
- 安装 LSPosed 后重启设备，没有看到通知
- 点击通知无反应或报错

**可能原因**：
1. Magisk 版本过低
2. Zygisk 未启用
3. 系统兼容性问题
4. 其他模块冲突

**解决方案**：
```bash
# 检查 Magisk 版本
adb shell su -c "magisk -v"

# 检查 Zygisk 状态
adb shell su -c "magisk --status"

# 检查 LSPosed 模块状态
adb shell su -c "ls -la /data/adb/modules/ | grep lsposed"
```

1. 确保 Magisk 版本 ≥ v24
2. 在 Magisk 设置中启用 Zygisk
3. 禁用其他可能冲突的模块
4. 检查设备兼容性列表

### Q2: Riru 版本安装失败

**症状**：
- 安装 Riru 版本 LSPosed 后模块显示为已安装但无法工作

**解决方案**：
1. 确保先安装 [Riru](https://github.com/RikkaApps/Riru/releases) v26.1.7+
2. 检查 Riru 是否正常工作：
```bash
adb shell su -c "ls -la /data/adb/riru/modules/"
```
3. 重新安装 LSPosed Riru 版本
4. 考虑改用 Zygisk 版本

### Q3: 模块在某些设备上无法工作

**常见不兼容设备**：
- 部分 MIUI 设备（特别是较新版本）
- One UI 某些版本
- 高度定制的 ROM

**解决方案**：
1. 检查 [兼容性列表](https://github.com/LSPosed/LSPosed/wiki/Device-support)
2. 尝试不同的 LSPosed 版本
3. 查看社区讨论寻找特定设备的解决方案

## 模块使用问题

### Q4: 模块安装后不生效

**排查步骤**：

1. **检查模块是否启用**
```java
// 在模块代码中添加日志
public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
    XposedHelpers.log("Module loaded for: " + lpparam.packageName);
    // 其他代码...
}
```

2. **检查作用域设置**
- 打开 LSPosed 管理器
- 进入模块设置
- 确保目标应用在作用域列表中

3. **检查目标应用版本兼容性**
```java
// 检查应用版本
public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
    if (!lpparam.packageName.equals("target.package.name")) {
        return;
    }
    
    Context context = AndroidAppHelper.currentApplication();
    if (context != null) {
        try {
            PackageInfo info = context.getPackageManager()
                .getPackageInfo(lpparam.packageName, 0);
            XposedHelpers.log("Target app version: " + info.versionName + 
                            " (" + info.versionCode + ")");
        } catch (Exception e) {
            XposedHelpers.log("Failed to get package info: " + e.getMessage());
        }
    }
}
```

4. **检查 Hook 目标是否存在**
```java
// 安全的 Hook 方式
try {
    Class<?> targetClass = XposedHelpers.findClass("com.example.TargetClass", 
                                                   lpparam.classLoader);
    Method targetMethod = XposedHelpers.findMethodExact(targetClass, "targetMethod", 
                                                        String.class);
    XposedHelpers.log("Found target method: " + targetMethod);
    
    XposedBridge.hookMethod(targetMethod, new XC_MethodHook() {
        // Hook 实现
    });
} catch (XposedHelpers.ClassNotFoundError e) {
    XposedHelpers.log("Target class not found: " + e.getMessage());
} catch (NoSuchMethodError e) {
    XposedHelpers.log("Target method not found: " + e.getMessage());
}
```

### Q5: 模块导致应用崩溃

**常见原因**：
1. Hook 了不应该 Hook 的方法
2. 修改参数或返回值类型不匹配
3. 在 Hook 中抛出未处理的异常

**调试方法**：

1. **添加异常处理**
```java
@Override
protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
    try {
        // 你的 Hook 代码
    } catch (Throwable t) {
        XposedHelpers.log("Exception in beforeHookedMethod: " + t.getMessage());
        // 不要重新抛出异常，让原方法正常执行
    }
}
```

2. **渐进式调试**
```java
public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
    if (!lpparam.packageName.equals("target.package")) {
        return;
    }
    
    XposedHelpers.log("Step 1: Package matched");
    
    try {
        Class<?> targetClass = XposedHelpers.findClass("com.example.Target", 
                                                       lpparam.classLoader);
        XposedHelpers.log("Step 2: Class found");
        
        XposedHelpers.findAndHookMethod(targetClass, "method", new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                XposedHelpers.log("Step 3: Method hooked and called");
                // 最小化的 Hook 代码进行测试
            }
        });
        XposedHelpers.log("Step 4: Hook installed");
    } catch (Throwable t) {
        XposedHelpers.log("Error at step: " + t.getMessage());
    }
}
```

### Q6: 模块在某些应用版本失效

**解决策略**：

1. **版本适配**
```java
public class VersionAdapter {
    private static final Map<String, Runnable> versionHooks = new HashMap<>();
    
    static {
        // 为不同版本注册不同的 Hook 逻辑
        versionHooks.put("1.0.0", () -> hookV1());
        versionHooks.put("2.0.0", () -> hookV2());
        versionHooks.put("3.0.0", () -> hookV3());
    }
    
    public static void hookForVersion(String version) {
        Runnable hookLogic = versionHooks.get(version);
        if (hookLogic != null) {
            hookLogic.run();
        } else {
            XposedHelpers.log("Unsupported version: " + version);
        }
    }
}
```

2. **动态方法查找**
```java
// 在多个可能的方法名中查找
private Method findMethodDynamically(Class<?> clazz, String[] possibleNames, 
                                   Class<?>... paramTypes) {
    for (String name : possibleNames) {
        try {
            return XposedHelpers.findMethodExact(clazz, name, paramTypes);
        } catch (NoSuchMethodError ignored) {
            // 继续尝试下一个方法名
        }
    }
    return null;
}

// 使用示例
Method targetMethod = findMethodDynamically(targetClass, 
    new String[]{"newMethodName", "oldMethodName", "alternativeMethodName"},
    String.class, int.class);
```

## 性能问题

### Q7: 模块导致系统卡顿

**排查方法**：

1. **性能监控**
```java
public class PerformanceMonitor extends XC_MethodHook {
    private static final Map<String, Long> methodTimes = new ConcurrentHashMap<>();
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        methodTimes.put(param.method.getName(), System.nanoTime());
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        Long startTime = methodTimes.remove(param.method.getName());
        if (startTime != null) {
            long duration = System.nanoTime() - startTime;
            if (duration > 1_000_000) { // 超过 1ms
                XposedHelpers.log("Slow hook: " + param.method.getName() + 
                                " took " + (duration / 1_000_000) + "ms");
            }
        }
    }
}
```

2. **优化建议**
- 避免在频繁调用的方法中进行重计算
- 使用缓存减少反射调用
- 避免在 Hook 中进行耗时操作

```java
// 不好的例子
@Override
protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
    // 每次都进行复杂计算
    String result = performComplexCalculation();
    param.args[0] = result;
}

// 好的例子
private static String cachedResult;
private static long lastCalculationTime;
private static final long CACHE_DURATION = 60000; // 1分钟

@Override
protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
    long currentTime = System.currentTimeMillis();
    if (cachedResult == null || (currentTime - lastCalculationTime) > CACHE_DURATION) {
        cachedResult = performComplexCalculation();
        lastCalculationTime = currentTime;
    }
    param.args[0] = cachedResult;
}
```

### Q8: 内存泄漏问题

**常见原因**：
1. 持有 Context 或 Activity 的强引用
2. 注册监听器后未及时取消注册
3. 静态集合持有对象引用

**解决方案**：

1. **使用弱引用**
```java
public class WeakReferenceHook extends XC_MethodHook {
    private final WeakReference<Context> contextRef;
    
    public WeakReferenceHook(Context context) {
        this.contextRef = new WeakReference<>(context);
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        Context context = contextRef.get();
        if (context != null) {
            // 使用 context
        }
    }
}
```

2. **及时清理资源**
```java
public class ResourceAwareHook extends XC_MethodHook {
    private final Set<Runnable> cleanupTasks = new ConcurrentSkipListSet<>();
    
    public void addCleanupTask(Runnable task) {
        cleanupTasks.add(task);
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        // 在适当时机清理资源
        if (shouldCleanup(param)) {
            for (Runnable task : cleanupTasks) {
                try {
                    task.run();
                } catch (Throwable t) {
                    XposedHelpers.log("Cleanup task failed: " + t.getMessage());
                }
            }
            cleanupTasks.clear();
        }
    }
}
```

## 调试问题

### Q9: 无法查看模块日志

**解决方案**：

1. **使用 ADB 查看日志**
```bash
# 查看 Xposed 相关日志
adb logcat | grep -i xposed

# 查看 LSPosed 相关日志
adb logcat | grep -i lsposed

# 查看特定标签的日志
adb logcat -s "YourModuleTag"
```

2. **在模块中添加更多日志**
```java
public class ModuleLogger {
    private static final String TAG = "YourModule";
    
    public static void log(String message) {
        XposedHelpers.log(TAG + ": " + message);
        // 同时输出到 Android 日志
        android.util.Log.i(TAG, message);
    }
    
    public static void logError(String message, Throwable t) {
        XposedHelpers.log(TAG + " ERROR: " + message);
        android.util.Log.e(TAG, message, t);
    }
}
```

3. **使用 LSPosed 管理器查看日志**
- 打开 LSPosed 管理器
- 进入"日志"标签
- 选择模块相关的日志

### Q10: Hook 没有被调用

**排查清单**：

1. **确认模块已加载**
```java
public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
    // 第一行就添加日志
    XposedHelpers.log("Module handleLoadPackage called for: " + lpparam.packageName);
    
    if (!lpparam.packageName.equals("target.package")) {
        return;
    }
    
    XposedHelpers.log("Target package matched, proceeding with hooks");
}
```

2. **确认类和方法存在**
```java
try {
    Class<?> targetClass = XposedHelpers.findClass("com.example.Target", lpparam.classLoader);
    XposedHelpers.log("Target class found: " + targetClass);
    
    Method[] methods = targetClass.getDeclaredMethods();
    XposedHelpers.log("Available methods in target class:");
    for (Method method : methods) {
        XposedHelpers.log("  " + method.getName() + ": " + method);
    }
} catch (Exception e) {
    XposedHelpers.log("Error finding target class: " + e.getMessage());
}
```

3. **确认方法被调用**
```java
XposedHelpers.findAndHookMethod(targetClass, "targetMethod", new XC_MethodHook() {
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        XposedHelpers.log("beforeHookedMethod called!");
        android.util.Log.i("YourModule", "Hook triggered!");
    }
});
```

## 兼容性问题

### Q11: 在新的 Android 版本上不工作

**适配策略**：

1. **API 级别检查**
```java
public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
    int apiLevel = Build.VERSION.SDK_INT;
    XposedHelpers.log("Current API level: " + apiLevel);
    
    if (apiLevel >= Build.VERSION_CODES.TIRAMISU) { // Android 13+
        hookForAndroid13Plus(lpparam);
    } else if (apiLevel >= Build.VERSION_CODES.S) { // Android 12+
        hookForAndroid12Plus(lpparam);
    } else {
        hookForOlderVersions(lpparam);
    }
}
```

2. **渐进式适配**
```java
private void safeHookWithFallback(ClassLoader classLoader) {
    // 尝试新的实现方式
    if (tryNewImplementation(classLoader)) {
        XposedHelpers.log("Using new implementation");
        return;
    }
    
    // 回退到旧的实现方式
    if (tryOldImplementation(classLoader)) {
        XposedHelpers.log("Using old implementation");
        return;
    }
    
    XposedHelpers.log("Both implementations failed");
}
```

### Q12: ROM 特定问题

**常见 ROM 适配**：

1. **MIUI 适配**
```java
private boolean isMIUI() {
    return !TextUtils.isEmpty(SystemProperties.get("ro.miui.ui.version.name", ""));
}

public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
    if (isMIUI()) {
        XposedHelpers.log("MIUI detected, using MIUI-specific implementation");
        hookForMIUI(lpparam);
    } else {
        hookForStandardAndroid(lpparam);
    }
}
```

2. **One UI 适配**
```java
private boolean isOneUI() {
    String brand = Build.BRAND.toLowerCase();
    String manufacturer = Build.MANUFACTURER.toLowerCase();
    return brand.contains("samsung") || manufacturer.contains("samsung");
}
```

## 高级问题

### Q13: 模块间冲突

**检测冲突**：
```java
public class ConflictDetector {
    public static void detectHookConflicts(Method method) {
        try {
            // 使用反射检查方法是否已被 Hook
            Field hookField = Method.class.getDeclaredField("artMethod");
            hookField.setAccessible(true);
            long artMethod = hookField.getLong(method);
            
            // 检查 ArtMethod 的状态
            XposedHelpers.log("Method " + method.getName() + " artMethod: " + artMethod);
        } catch (Exception e) {
            XposedHelpers.log("Failed to detect conflicts: " + e.getMessage());
        }
    }
}
```

**避免冲突**：
1. 使用更具体的 Hook 条件
2. 与其他模块开发者协调
3. 实现可配置的优先级系统

### Q14: 系统服务 Hook

**系统服务 Hook 示例**：
```java
public void hookSystemService(LoadPackageParam lpparam) {
    if (!"android".equals(lpparam.packageName)) {
        return;
    }
    
    try {
        // Hook ActivityManagerService
        Class<?> amsClass = XposedHelpers.findClass(
            "com.android.server.am.ActivityManagerService", lpparam.classLoader);
            
        XposedHelpers.findAndHookMethod(amsClass, "startActivity",
            // 复杂的参数列表...
            new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    XposedHelpers.log("Activity starting...");
                }
            });
    } catch (Throwable t) {
        XposedHelpers.log("Failed to hook system service: " + t.getMessage());
    }
}
```

## 获取更多帮助

### 有用的资源

1. **官方文档**
   - [Xposed API 文档](https://api.xposed.info/)
   - [LSPosed GitHub](https://github.com/LSPosed/LSPosed)

2. **社区支持**
   - [LSPosed Telegram](https://t.me/s/LSPosed)
   - [Xposed 开发者论坛](https://forum.xda-developers.com/c/xposed.3531/)

3. **调试工具**
   - [LSPosed 管理器](https://github.com/LSPosed/LSPosed/releases)
   - Android Studio 调试器
   - ADB 命令行工具

### 报告问题

报告问题时请提供：

1. **设备信息**
   - 设备型号和 Android 版本
   - ROM 信息（MIUI、One UI 等）
   - Magisk 和 LSPosed 版本

2. **详细日志**
```bash
# 收集完整日志
adb logcat -v time > logcat.txt
```

3. **重现步骤**
   - 详细的操作步骤
   - 预期结果 vs 实际结果
   - 相关的代码片段

4. **模块信息**
   - 模块版本和配置
   - 作用域设置
   - 其他已安装的模块

记住：LSPosed 项目只接受**英文标题**的 issue，如果不懂英语，请使用[翻译工具](https://www.deepl.com/zh/translator)。
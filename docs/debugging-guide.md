# LSPosed 调试指南

本指南介绍如何有效地调试 LSPosed 模块，包括工具使用、常见问题诊断和高级调试技巧。

## 调试环境搭建

### 1. 开发环境配置

#### Android Studio 配置
```gradle
android {
    buildTypes {
        debug {
            debuggable true
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }
}

dependencies {
    // 调试依赖
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.10'
    
    // 日志工具
    implementation 'com.jakewharton.timber:timber:5.0.1'
}
```

#### 日志配置
```java
public class ModuleLogger {
    private static final String TAG = "YourModule";
    private static final boolean DEBUG = BuildConfig.DEBUG;
    
    static {
        if (DEBUG) {
            Timber.plant(new Timber.DebugTree());
        }
    }
    
    public static void d(String message, Object... args) {
        if (DEBUG) {
            Timber.tag(TAG).d(message, args);
            XposedHelpers.log("[DEBUG] " + TAG + ": " + String.format(message, args));
        }
    }
    
    public static void i(String message, Object... args) {
        Timber.tag(TAG).i(message, args);
        XposedHelpers.log("[INFO] " + TAG + ": " + String.format(message, args));
    }
    
    public static void w(String message, Object... args) {
        Timber.tag(TAG).w(message, args);
        XposedHelpers.log("[WARN] " + TAG + ": " + String.format(message, args));
    }
    
    public static void e(Throwable t, String message, Object... args) {
        Timber.tag(TAG).e(t, message, args);
        XposedHelpers.log("[ERROR] " + TAG + ": " + String.format(message, args));
        if (t != null) {
            XposedHelpers.log(android.util.Log.getStackTraceString(t));
        }
    }
}
```

### 2. 调试工具准备

#### ADB 工具
```bash
# 安装 ADB
# Windows: 下载 Android SDK Platform Tools
# macOS: brew install android-platform-tools
# Linux: sudo apt install android-tools-adb

# 连接设备
adb devices

# 开启 WiFi 调试（Android 11+）
adb tcpip 5555
adb connect <device_ip>:5555
```

#### 日志查看脚本
```bash
#!/bin/bash
# log_viewer.sh

MODULE_TAG="YourModule"
PACKAGE_NAME="com.example.targetapp"

echo "Watching logs for module: $MODULE_TAG"
echo "Target package: $PACKAGE_NAME"
echo "Press Ctrl+C to stop"

adb logcat -v time \
    -s "$MODULE_TAG" \
    -s "Xposed" \
    -s "LSPosed" \
    -s "AndroidRuntime" \
    | grep -E "$MODULE_TAG|$PACKAGE_NAME|FATAL"
```

## 基础调试技术

### 1. 日志调试

#### 结构化日志
```java
public class StructuredLogger {
    
    public static void logMethodEntry(String className, String methodName, Object... args) {
        StringBuilder sb = new StringBuilder();
        sb.append(">>> ENTRY: ").append(className).append(".").append(methodName).append("(");
        
        for (int i = 0; i < args.length; i++) {
            if (i > 0) sb.append(", ");
            sb.append(formatArgument(args[i]));
        }
        sb.append(")");
        
        ModuleLogger.d(sb.toString());
    }
    
    public static void logMethodExit(String className, String methodName, Object result) {
        ModuleLogger.d("<<< EXIT: %s.%s() -> %s", className, methodName, formatArgument(result));
    }
    
    public static void logMethodException(String className, String methodName, Throwable t) {
        ModuleLogger.e(t, "!!! EXCEPTION: %s.%s() threw %s: %s", 
                      className, methodName, t.getClass().getSimpleName(), t.getMessage());
    }
    
    public static void logHookInstallation(String className, String methodName, boolean success) {
        if (success) {
            ModuleLogger.i("Hook installed: %s.%s()", className, methodName);
        } else {
            ModuleLogger.e(null, "Hook installation failed: %s.%s()", className, methodName);
        }
    }
    
    private static String formatArgument(Object arg) {
        if (arg == null) return "null";
        if (arg instanceof String) return "\"" + arg + "\"";
        if (arg instanceof Number || arg instanceof Boolean) return arg.toString();
        if (arg instanceof Collection) return arg.getClass().getSimpleName() + "[" + ((Collection<?>) arg).size() + "]";
        if (arg.getClass().isArray()) return arg.getClass().getSimpleName() + "[" + Array.getLength(arg) + "]";
        return arg.getClass().getSimpleName() + "@" + Integer.toHexString(arg.hashCode());
    }
}
```

#### 调试装饰器
```java
public class DebuggingHook extends XC_MethodHook {
    private final String hookName;
    private final XC_MethodHook actualHook;
    
    public DebuggingHook(String hookName, XC_MethodHook actualHook) {
        this.hookName = hookName;
        this.actualHook = actualHook;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        long startTime = System.nanoTime();
        param.setObjectExtra("debugStartTime", startTime);
        
        StructuredLogger.logMethodEntry(
            param.method.getDeclaringClass().getSimpleName(),
            param.method.getName(),
            param.args
        );
        
        try {
            if (actualHook != null) {
                actualHook.beforeHookedMethod(param);
            }
        } catch (Throwable t) {
            StructuredLogger.logMethodException(
                param.method.getDeclaringClass().getSimpleName(),
                param.method.getName(),
                t
            );
            throw t;
        }
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        try {
            if (actualHook != null) {
                actualHook.afterHookedMethod(param);
            }
        } catch (Throwable t) {
            StructuredLogger.logMethodException(
                param.method.getDeclaringClass().getSimpleName(),
                param.method.getName(),
                t
            );
            throw t;
        } finally {
            Long startTime = (Long) param.getObjectExtra("debugStartTime");
            if (startTime != null) {
                long duration = System.nanoTime() - startTime;
                StructuredLogger.logMethodExit(
                    param.method.getDeclaringClass().getSimpleName(),
                    param.method.getName(),
                    param.hasResult() ? param.getResult() : "void"
                );
                
                if (duration > 1_000_000) { // > 1ms
                    ModuleLogger.w("Slow hook [%s]: %.2f ms", hookName, duration / 1_000_000.0);
                }
            }
        }
    }
    
    // 工厂方法
    public static DebuggingHook wrap(String hookName, XC_MethodHook hook) {
        return new DebuggingHook(hookName, hook);
    }
}
```

### 2. 断点调试

#### 条件断点
```java
public class ConditionalBreakpoint {
    
    public static void breakIf(boolean condition, String message) {
        if (condition && BuildConfig.DEBUG) {
            ModuleLogger.d("BREAKPOINT: " + message);
            // 这里可以添加 debugger 断点
            android.os.Debug.waitForDebugger();
        }
    }
    
    public static void breakOnValue(Object value, Object expectedValue, String context) {
        if (Objects.equals(value, expectedValue) && BuildConfig.DEBUG) {
            ModuleLogger.d("BREAKPOINT: %s - value matches expected: %s", context, expectedValue);
            android.os.Debug.waitForDebugger();
        }
    }
    
    public static void breakOnException(Throwable t, String context) {
        if (t != null && BuildConfig.DEBUG) {
            ModuleLogger.d("BREAKPOINT: %s - exception occurred: %s", context, t.getMessage());
            android.os.Debug.waitForDebugger();
        }
    }
}

// 使用示例
public class DebugHookExample extends XC_MethodHook {
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        // 在特定条件下触发断点
        ConditionalBreakpoint.breakIf(
            param.args.length > 0 && "debug".equals(param.args[0]),
            "Debug parameter detected"
        );
        
        String userInput = (String) param.args[0];
        ConditionalBreakpoint.breakOnValue(
            userInput, "suspicious_value",
            "User input check"
        );
    }
}
```

### 3. 运行时检查

#### 调用栈分析
```java
public class StackTraceAnalyzer {
    
    public static void analyzeCallStack(String context) {
        StackTraceElement[] stack = Thread.currentThread().getStackTrace();
        
        ModuleLogger.d("=== STACK ANALYSIS: %s ===", context);
        ModuleLogger.d("Thread: %s", Thread.currentThread().getName());
        
        for (int i = 0; i < Math.min(stack.length, 15); i++) {
            StackTraceElement element = stack[i];
            String className = element.getClassName();
            String methodName = element.getMethodName();
            String fileName = element.getFileName();
            int lineNumber = element.getLineNumber();
            
            // 高亮重要的调用
            String prefix = "  ";
            if (className.contains("com.android.") || className.contains("android.")) {
                prefix = "→ ";
            } else if (className.contains("com.example.")) {
                prefix = "★ ";
            }
            
            ModuleLogger.d("%s%s.%s(%s:%d)", prefix, className, methodName, fileName, lineNumber);
        }
        ModuleLogger.d("=== END STACK ANALYSIS ===");
    }
    
    public static boolean isCalledFromPackage(String packageName) {
        StackTraceElement[] stack = Thread.currentThread().getStackTrace();
        for (StackTraceElement element : stack) {
            if (element.getClassName().startsWith(packageName)) {
                return true;
            }
        }
        return false;
    }
    
    public static String findFirstNonFrameworkCaller() {
        StackTraceElement[] stack = Thread.currentThread().getStackTrace();
        for (StackTraceElement element : stack) {
            String className = element.getClassName();
            if (!className.startsWith("java.") && 
                !className.startsWith("android.") &&
                !className.startsWith("com.android.") &&
                !className.startsWith("dalvik.")) {
                return className + "." + element.getMethodName();
            }
        }
        return "unknown";
    }
}
```

## 高级调试技术

### 1. 内存调试

#### 内存泄漏检测
```java
public class MemoryLeakDetector {
    private static final Map<String, WeakReference<Object>> trackedObjects = new ConcurrentHashMap<>();
    private static final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
    
    static {
        // 定期检查内存泄漏
        scheduler.scheduleAtFixedRate(MemoryLeakDetector::checkLeaks, 30, 30, TimeUnit.SECONDS);
    }
    
    public static void trackObject(String identifier, Object object) {
        trackedObjects.put(identifier, new WeakReference<>(object));
        ModuleLogger.d("Tracking object: %s (%s)", identifier, object.getClass().getSimpleName());
    }
    
    public static void stopTracking(String identifier) {
        trackedObjects.remove(identifier);
        ModuleLogger.d("Stopped tracking: %s", identifier);
    }
    
    private static void checkLeaks() {
        int totalTracked = trackedObjects.size();
        int leakedCount = 0;
        
        Iterator<Map.Entry<String, WeakReference<Object>>> iterator = trackedObjects.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, WeakReference<Object>> entry = iterator.next();
            Object object = entry.getValue().get();
            
            if (object == null) {
                iterator.remove(); // 对象已被 GC
            } else {
                leakedCount++;
                ModuleLogger.w("Potential leak: %s (%s)", entry.getKey(), object.getClass().getSimpleName());
            }
        }
        
        if (leakedCount > 0) {
            ModuleLogger.w("Memory check: %d/%d objects still referenced", leakedCount, totalTracked);
        }
    }
}
```

#### Hook 内存监控
```java
public class HookMemoryMonitor extends XC_MethodHook {
    private final Runtime runtime = Runtime.getRuntime();
    private long lastMemoryCheck = 0;
    private static final long MEMORY_CHECK_INTERVAL = 5000; // 5秒
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        checkMemoryIfNeeded();
    }
    
    private void checkMemoryIfNeeded() {
        long currentTime = System.currentTimeMillis();
        if (currentTime - lastMemoryCheck > MEMORY_CHECK_INTERVAL) {
            lastMemoryCheck = currentTime;
            
            long totalMemory = runtime.totalMemory();
            long freeMemory = runtime.freeMemory();
            long usedMemory = totalMemory - freeMemory;
            long maxMemory = runtime.maxMemory();
            
            double usedPercentage = (double) usedMemory / maxMemory * 100;
            
            ModuleLogger.d("Memory: %.1f%% (Used: %d KB, Free: %d KB, Max: %d KB)",
                          usedPercentage,
                          usedMemory / 1024,
                          freeMemory / 1024,
                          maxMemory / 1024);
            
            if (usedPercentage > 80) {
                ModuleLogger.w("High memory usage: %.1f%%", usedPercentage);
            }
        }
    }
}
```

### 2. 性能调试

#### 性能分析器
```java
public class PerformanceProfiler {
    private static final Map<String, PerformanceData> performanceMap = new ConcurrentHashMap<>();
    
    public static void startProfiling(String operationName) {
        PerformanceData data = performanceMap.computeIfAbsent(operationName, k -> new PerformanceData());
        data.startTime = System.nanoTime();
    }
    
    public static void endProfiling(String operationName) {
        PerformanceData data = performanceMap.get(operationName);
        if (data != null && data.startTime > 0) {
            long duration = System.nanoTime() - data.startTime;
            data.recordDuration(duration);
            data.startTime = 0;
        }
    }
    
    public static void printReport() {
        ModuleLogger.i("=== PERFORMANCE REPORT ===");
        for (Map.Entry<String, PerformanceData> entry : performanceMap.entrySet()) {
            String operation = entry.getKey();
            PerformanceData data = entry.getValue();
            
            ModuleLogger.i("%s:", operation);
            ModuleLogger.i("  Calls: %d", data.callCount);
            ModuleLogger.i("  Total: %.2f ms", data.totalDuration / 1_000_000.0);
            ModuleLogger.i("  Average: %.2f ms", data.getAverageDuration() / 1_000_000.0);
            ModuleLogger.i("  Min: %.2f ms", data.minDuration / 1_000_000.0);
            ModuleLogger.i("  Max: %.2f ms", data.maxDuration / 1_000_000.0);
        }
        ModuleLogger.i("=== END REPORT ===");
    }
    
    private static class PerformanceData {
        long startTime = 0;
        int callCount = 0;
        long totalDuration = 0;
        long minDuration = Long.MAX_VALUE;
        long maxDuration = 0;
        
        synchronized void recordDuration(long duration) {
            callCount++;
            totalDuration += duration;
            minDuration = Math.min(minDuration, duration);
            maxDuration = Math.max(maxDuration, duration);
        }
        
        double getAverageDuration() {
            return callCount > 0 ? (double) totalDuration / callCount : 0;
        }
    }
}

// 使用示例
public class ProfiledHook extends XC_MethodHook {
    private final String operationName;
    
    public ProfiledHook(String operationName) {
        this.operationName = operationName;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        PerformanceProfiler.startProfiling(operationName);
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        PerformanceProfiler.endProfiling(operationName);
    }
}
```

### 3. 网络调试

#### 网络请求监控
```java
public class NetworkDebugger {
    
    public static void hookHttpURLConnection(ClassLoader classLoader) {
        try {
            Class<?> httpURLConnectionClass = XposedHelpers.findClass("java.net.HttpURLConnection", classLoader);
            
            // Hook connect 方法
            XposedHelpers.findAndHookMethod(httpURLConnectionClass, "connect", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    HttpURLConnection connection = (HttpURLConnection) param.thisObject;
                    ModuleLogger.d("HTTP Request: %s %s", connection.getRequestMethod(), connection.getURL());
                }
            });
            
            // Hook getResponseCode 方法
            XposedHelpers.findAndHookMethod(httpURLConnectionClass, "getResponseCode", new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    HttpURLConnection connection = (HttpURLConnection) param.thisObject;
                    int responseCode = (int) param.getResult();
                    ModuleLogger.d("HTTP Response: %s %s -> %d", 
                                  connection.getRequestMethod(), connection.getURL(), responseCode);
                }
            });
            
        } catch (Throwable t) {
            ModuleLogger.e(t, "Failed to hook HttpURLConnection");
        }
    }
    
    public static void hookOkHttp(ClassLoader classLoader) {
        try {
            // Hook OkHttp RealCall
            Class<?> realCallClass = XposedHelpers.findClass("okhttp3.internal.RealCall", classLoader);
            
            XposedHelpers.findAndHookMethod(realCallClass, "execute", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    Object request = XposedHelpers.getObjectField(param.thisObject, "originalRequest");
                    Object url = XposedHelpers.callMethod(request, "url");
                    String method = (String) XposedHelpers.callMethod(request, "method");
                    
                    ModuleLogger.d("OkHttp Request: %s %s", method, url);
                }
                
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    if (!param.hasThrowable()) {
                        Object response = param.getResult();
                        int code = (int) XposedHelpers.callMethod(response, "code");
                        ModuleLogger.d("OkHttp Response: %d", code);
                    }
                }
            });
            
        } catch (Throwable t) {
            ModuleLogger.e(t, "Failed to hook OkHttp");
        }
    }
}
```

## 常见问题诊断

### 1. Hook 不生效

#### 诊断清单
```java
public class HookDiagnostics {
    
    public static void diagnoseHookIssue(String className, String methodName, ClassLoader classLoader) {
        ModuleLogger.i("=== HOOK DIAGNOSTICS ===");
        ModuleLogger.i("Target: %s.%s", className, methodName);
        
        // 1. 检查类是否存在
        try {
            Class<?> clazz = XposedHelpers.findClass(className, classLoader);
            ModuleLogger.i("✓ Class found: %s", clazz);
            
            // 2. 检查方法是否存在
            Method[] methods = clazz.getDeclaredMethods();
            boolean methodFound = false;
            for (Method method : methods) {
                if (method.getName().equals(methodName)) {
                    methodFound = true;
                    ModuleLogger.i("✓ Method found: %s", method);
                    
                    // 3. 检查方法修饰符
                    int modifiers = method.getModifiers();
                    if (Modifier.isAbstract(modifiers)) {
                        ModuleLogger.w("⚠ Method is abstract");
                    }
                    if (Modifier.isNative(modifiers)) {
                        ModuleLogger.w("⚠ Method is native");
                    }
                    if (Modifier.isFinal(modifiers)) {
                        ModuleLogger.w("⚠ Method is final");
                    }
                    
                    break;
                }
            }
            
            if (!methodFound) {
                ModuleLogger.e(null, "✗ Method not found");
                ModuleLogger.i("Available methods:");
                for (Method method : methods) {
                    ModuleLogger.i("  %s", method);
                }
            }
            
        } catch (XposedHelpers.ClassNotFoundError e) {
            ModuleLogger.e(e, "✗ Class not found: %s", className);
            
            // 尝试查找相似的类
            findSimilarClasses(className, classLoader);
        }
        
        // 4. 检查作用域
        checkScope();
        
        // 5. 检查模块状态
        checkModuleStatus();
        
        ModuleLogger.i("=== END DIAGNOSTICS ===");
    }
    
    private static void findSimilarClasses(String targetClassName, ClassLoader classLoader) {
        ModuleLogger.i("Searching for similar classes...");
        
        String packageName = targetClassName.substring(0, targetClassName.lastIndexOf('.'));
        String simpleName = targetClassName.substring(targetClassName.lastIndexOf('.') + 1);
        
        try {
            // 尝试查找包中的其他类
            DexFile dexFile = new DexFile(classLoader.toString());
            Enumeration<String> classNames = dexFile.entries();
            
            while (classNames.hasMoreElements()) {
                String className = classNames.nextElement();
                if (className.startsWith(packageName) && 
                    className.toLowerCase().contains(simpleName.toLowerCase())) {
                    ModuleLogger.i("  Similar: %s", className);
                }
            }
        } catch (Exception e) {
            ModuleLogger.d("Could not enumerate classes: %s", e.getMessage());
        }
    }
    
    private static void checkScope() {
        // 检查模块作用域设置
        ModuleLogger.i("Checking scope configuration...");
        // 这里需要根据具体的作用域检查逻辑实现
    }
    
    private static void checkModuleStatus() {
        ModuleLogger.i("Module status:");
        ModuleLogger.i("  Process: %s", getCurrentProcessName());
        ModuleLogger.i("  PID: %d", android.os.Process.myPid());
        ModuleLogger.i("  UID: %d", android.os.Process.myUid());
    }
    
    private static String getCurrentProcessName() {
        try {
            return Application.getProcessName();
        } catch (Exception e) {
            return "unknown";
        }
    }
}
```

### 2. 性能问题

#### 性能诊断工具
```java
public class PerformanceDiagnostics {
    
    public static void monitorHookPerformance(String hookName, Runnable hookLogic) {
        long startTime = System.nanoTime();
        long startMemory = getUsedMemory();
        
        try {
            hookLogic.run();
        } finally {
            long endTime = System.nanoTime();
            long endMemory = getUsedMemory();
            
            long duration = endTime - startTime;
            long memoryDelta = endMemory - startMemory;
            
            if (duration > 5_000_000) { // > 5ms
                ModuleLogger.w("Slow hook [%s]: %.2f ms", hookName, duration / 1_000_000.0);
            }
            
            if (memoryDelta > 1024 * 1024) { // > 1MB
                ModuleLogger.w("Memory-heavy hook [%s]: %d KB", hookName, memoryDelta / 1024);
            }
            
            if (duration > 1_000_000) { // > 1ms
                ModuleLogger.d("Hook [%s]: %.2f ms, Memory: %+d KB", 
                              hookName, duration / 1_000_000.0, memoryDelta / 1024);
            }
        }
    }
    
    private static long getUsedMemory() {
        Runtime runtime = Runtime.getRuntime();
        return runtime.totalMemory() - runtime.freeMemory();
    }
}
```

## 调试工具集成

### 1. 远程调试

#### USB 调试设置
```bash
# 启用 USB 调试
adb shell settings put global development_settings_enabled 1
adb shell settings put global adb_enabled 1

# 端口转发
adb forward tcp:8080 tcp:8080

# 无线调试
adb tcpip 5555
adb connect <device_ip>:5555
```

#### Android Studio 远程调试
```java
public class RemoteDebugHelper {
    
    public static void waitForDebugger() {
        if (BuildConfig.DEBUG) {
            ModuleLogger.d("Waiting for debugger...");
            android.os.Debug.waitForDebugger();
            ModuleLogger.d("Debugger connected!");
        }
    }
    
    public static void enableStrictMode() {
        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                .detectDiskReads()
                .detectDiskWrites()
                .detectNetwork()
                .penaltyLog()
                .build());
                
            StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                .detectLeakedSqlLiteObjects()
                .detectLeakedClosableObjects()
                .penaltyLog()
                .build());
        }
    }
}
```

### 2. 可视化调试

#### 调试面板
```java
public class DebugPanel {
    private static final List<DebugEntry> debugEntries = new ArrayList<>();
    
    public static void addEntry(String category, String key, Object value) {
        debugEntries.add(new DebugEntry(category, key, value, System.currentTimeMillis()));
        
        // 限制条目数量
        if (debugEntries.size() > 1000) {
            debugEntries.subList(0, 100).clear();
        }
    }
    
    public static void dumpDebugInfo() {
        ModuleLogger.i("=== DEBUG PANEL ===");
        
        Map<String, List<DebugEntry>> categorized = debugEntries.stream()
            .collect(Collectors.groupingBy(entry -> entry.category));
            
        for (Map.Entry<String, List<DebugEntry>> entry : categorized.entrySet()) {
            ModuleLogger.i("[%s]", entry.getKey());
            for (DebugEntry debugEntry : entry.getValue()) {
                ModuleLogger.i("  %s: %s (at %s)", 
                              debugEntry.key, 
                              debugEntry.value,
                              new Date(debugEntry.timestamp));
            }
        }
        
        ModuleLogger.i("=== END DEBUG PANEL ===");
    }
    
    private static class DebugEntry {
        final String category;
        final String key;
        final Object value;
        final long timestamp;
        
        DebugEntry(String category, String key, Object value, long timestamp) {
            this.category = category;
            this.key = key;
            this.value = value;
            this.timestamp = timestamp;
        }
    }
}
```

## 最佳实践

### 1. 调试日志规范

```java
public class LoggingBestPractices {
    
    // 使用结构化日志格式
    public static void logStructured(String event, Map<String, Object> data) {
        StringBuilder sb = new StringBuilder();
        sb.append("EVENT: ").append(event);
        
        for (Map.Entry<String, Object> entry : data.entrySet()) {
            sb.append(", ").append(entry.getKey()).append("=").append(entry.getValue());
        }
        
        ModuleLogger.i(sb.toString());
    }
    
    // 使用不同级别的日志
    public static void demonstrateLogLevels() {
        // DEBUG: 详细的调试信息
        ModuleLogger.d("Detailed debugging information");
        
        // INFO: 重要的状态信息
        ModuleLogger.i("Important state changes");
        
        // WARN: 可能的问题
        ModuleLogger.w("Potential issues that need attention");
        
        // ERROR: 严重错误
        ModuleLogger.e(null, "Critical errors that affect functionality");
    }
}
```

### 2. 调试代码管理

```java
public class DebugCodeManagement {
    
    // 使用编译时常量控制调试代码
    private static final boolean VERBOSE_LOGGING = BuildConfig.DEBUG && true;
    private static final boolean PERFORMANCE_MONITORING = BuildConfig.DEBUG && true;
    private static final boolean MEMORY_TRACKING = BuildConfig.DEBUG && false;
    
    public static void verboseLog(String message, Object... args) {
        if (VERBOSE_LOGGING) {
            ModuleLogger.d(message, args);
        }
    }
    
    public static void performanceMonitor(String operation, Runnable code) {
        if (PERFORMANCE_MONITORING) {
            PerformanceProfiler.startProfiling(operation);
            try {
                code.run();
            } finally {
                PerformanceProfiler.endProfiling(operation);
            }
        } else {
            code.run();
        }
    }
}
```

### 3. 调试信息收集

```java
public class DebugInfoCollector {
    
    public static String collectSystemInfo() {
        StringBuilder info = new StringBuilder();
        
        info.append("=== SYSTEM INFO ===\n");
        info.append("Android Version: ").append(Build.VERSION.RELEASE).append("\n");
        info.append("API Level: ").append(Build.VERSION.SDK_INT).append("\n");
        info.append("Device: ").append(Build.MANUFACTURER).append(" ").append(Build.MODEL).append("\n");
        info.append("Architecture: ").append(Build.SUPPORTED_ABIS[0]).append("\n");
        
        Runtime runtime = Runtime.getRuntime();
        info.append("Memory: ").append((runtime.totalMemory() - runtime.freeMemory()) / 1024 / 1024)
            .append("/").append(runtime.maxMemory() / 1024 / 1024).append(" MB\n");
            
        info.append("Process: ").append(getCurrentProcessName()).append("\n");
        info.append("PID: ").append(android.os.Process.myPid()).append("\n");
        
        return info.toString();
    }
    
    private static String getCurrentProcessName() {
        try {
            return Application.getProcessName();
        } catch (Exception e) {
            return "unknown";
        }
    }
}
```

## 总结

有效的调试是成功开发 LSPosed 模块的关键。通过：

1. **合理的日志策略**: 结构化、分级别的日志记录
2. **性能监控**: 及时发现性能问题
3. **内存管理**: 避免内存泄漏
4. **系统化的诊断**: 快速定位问题根源
5. **工具集成**: 利用现有调试工具

您可以大大提高开发效率和模块质量。记住，调试是一个迭代过程，需要耐心和系统性的方法。

## 相关文档

- [模块开发指南](./module-development.md) - 模块开发基础
- [常见问题](./faq.md) - 常见问题解答
- [性能优化](./performance-optimization.md) - 性能优化技巧
- [Hook 技术详解](./hook-techniques.md) - Hook 技术深入
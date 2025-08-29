# Hook 技术详解

本文档深入探讨 LSPosed 中的 Hook 技术原理、实现方式和高级应用技巧。

## Hook 技术基础

### 1. Hook 的概念

Hook（钩子）是一种程序技术，允许在不修改原始代码的情况下拦截和修改函数调用。在 Android 系统中，Hook 技术主要用于：

- **行为修改**: 改变应用或系统的行为
- **功能增强**: 为现有应用添加新功能
- **调试分析**: 监控和分析程序运行
- **安全审计**: 检测恶意行为

### 2. ART Hook 原理

LSPosed 基于 Android Runtime (ART) 实现 Hook，其核心原理：

```cpp
// ART 中的 ArtMethod 结构（简化）
class ArtMethod {
private:
    // 方法入口点
    void* entry_point_from_quick_compiled_code_;
    
    // 方法元数据
    uint32_t access_flags_;
    uint32_t dex_code_item_offset_;
    uint32_t dex_method_index_;
    
public:
    // 获取入口点
    void* GetEntryPointFromQuickCompiledCode() {
        return entry_point_from_quick_compiled_code_;
    }
    
    // 设置入口点
    void SetEntryPointFromQuickCompiledCode(void* entry_point) {
        entry_point_from_quick_compiled_code_ = entry_point;
    }
};
```

#### Hook 过程

1. **获取 ArtMethod**: 从 Java Method 对象获取对应的 ArtMethod 指针
2. **创建桥接函数**: 创建 Hook 处理桥接代码
3. **替换入口点**: 将原方法入口点替换为桥接函数
4. **保存原入口点**: 保存原入口点以便调用原方法

```cpp
// Hook 安装过程
bool InstallHook(ArtMethod* target_method, void* hook_bridge) {
    // 1. 保存原入口点
    void* original_entry = target_method->GetEntryPointFromQuickCompiledCode();
    
    // 2. 创建备份方法
    ArtMethod* backup_method = CreateBackupMethod(target_method);
    backup_method->SetEntryPointFromQuickCompiledCode(original_entry);
    
    // 3. 替换入口点
    target_method->SetEntryPointFromQuickCompiledCode(hook_bridge);
    
    // 4. 存储映射关系
    hook_map[target_method] = backup_method;
    
    return true;
}
```

## Hook 类型和技术

### 1. 方法 Hook

#### 前置 Hook (Before Hook)
在目标方法执行前拦截，可以：
- 查看和修改参数
- 阻止方法执行
- 记录调用信息

```java
public class BeforeHookExample extends XC_MethodHook {
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        // 参数检查
        if (param.args.length > 0 && param.args[0] instanceof String) {
            String arg = (String) param.args[0];
            
            // 参数验证
            if (arg.contains("malicious")) {
                XposedHelpers.log("Blocked malicious call with: " + arg);
                param.setResult(null); // 阻止方法执行
                return;
            }
            
            // 参数修改
            param.args[0] = "Sanitized: " + arg;
        }
        
        // 调用记录
        recordMethodCall(param.method.getName(), param.args);
    }
    
    private void recordMethodCall(String methodName, Object[] args) {
        StringBuilder sb = new StringBuilder();
        sb.append("Method: ").append(methodName).append(", Args: ");
        for (Object arg : args) {
            sb.append(arg).append(", ");
        }
        XposedHelpers.log("Call recorded: " + sb.toString());
    }
}
```

#### 后置 Hook (After Hook)
在目标方法执行后拦截，可以：
- 查看和修改返回值
- 获取执行结果
- 进行后续处理

```java
public class AfterHookExample extends XC_MethodHook {
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        Object result = param.getResult();
        
        // 结果验证
        if (result instanceof List) {
            List<?> list = (List<?>) result;
            if (list.size() > MAX_ALLOWED_SIZE) {
                XposedHelpers.log("Limiting result size from " + list.size() + " to " + MAX_ALLOWED_SIZE);
                param.setResult(list.subList(0, MAX_ALLOWED_SIZE));
            }
        }
        
        // 结果增强
        if (result instanceof String) {
            String str = (String) result;
            param.setResult(str + " [Enhanced]");
        }
        
        // 统计信息
        updateStatistics(param.method.getName(), result);
    }
    
    private void updateStatistics(String methodName, Object result) {
        // 记录方法调用统计
    }
}
```

#### 替换 Hook (Replacement Hook)
完全替换目标方法的实现：

```java
public class ReplacementHookExample extends XC_MethodReplacement {
    @Override
    protected Object replaceHookedMethod(MethodHookParam param) throws Throwable {
        // 完全自定义的实现
        if (param.args.length > 0) {
            String input = (String) param.args[0];
            
            // 自定义逻辑
            if (input.startsWith("special:")) {
                return handleSpecialCase(input.substring(8));
            }
            
            // 调用其他方法
            return alternativeImplementation(input);
        }
        
        return "Default result";
    }
    
    private Object handleSpecialCase(String data) {
        // 特殊处理逻辑
        return "Special: " + data.toUpperCase();
    }
    
    private Object alternativeImplementation(String input) {
        // 替代实现
        return "Alternative: " + input.toLowerCase();
    }
}
```

### 2. 构造函数 Hook

构造函数 Hook 允许在对象创建过程中进行干预：

```java
public class ConstructorHookExample implements IXposedHookLoadPackage {
    
    @Override
    public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
        if (!"com.example.targetapp".equals(lpparam.packageName)) {
            return;
        }
        
        Class<?> targetClass = XposedHelpers.findClass("com.example.TargetClass", lpparam.classLoader);
        
        // Hook 默认构造函数
        XposedHelpers.findAndHookConstructor(targetClass, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                XposedHelpers.log("Creating instance of " + targetClass.getSimpleName());
            }
            
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                // 修改新创建的对象
                Object instance = param.thisObject;
                
                // 设置默认值
                XposedHelpers.setObjectField(instance, "defaultValue", "Hooked default");
                XposedHelpers.setBooleanField(instance, "isHooked", true);
                
                // 注册实例
                registerInstance(instance);
            }
        });
        
        // Hook 带参数的构造函数
        XposedHelpers.findAndHookConstructor(targetClass, String.class, int.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                // 参数验证和修改
                String strParam = (String) param.args[0];
                int intParam = (int) param.args[1];
                
                if (strParam == null || strParam.isEmpty()) {
                    param.args[0] = "Default Name";
                }
                
                if (intParam < 0) {
                    param.args[1] = 0;
                }
            }
        });
    }
    
    private void registerInstance(Object instance) {
        // 注册实例到全局管理器
    }
}
```

### 3. 字段访问 Hook

虽然 Xposed 不直接支持字段 Hook，但可以通过 Hook getter/setter 方法实现：

```java
public class FieldAccessHookExample implements IXposedHookLoadPackage {
    
    @Override
    public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
        if (!"com.example.targetapp".equals(lpparam.packageName)) {
            return;
        }
        
        Class<?> targetClass = XposedHelpers.findClass("com.example.TargetClass", lpparam.classLoader);
        
        // Hook getter 方法
        XposedHelpers.findAndHookMethod(targetClass, "getImportantValue", new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                Object originalValue = param.getResult();
                
                // 值转换或验证
                if (originalValue instanceof Integer) {
                    int value = (int) originalValue;
                    if (value > 100) {
                        param.setResult(100); // 限制最大值
                    }
                }
            }
        });
        
        // Hook setter 方法
        XposedHelpers.findAndHookMethod(targetClass, "setImportantValue", int.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                int newValue = (int) param.args[0];
                
                // 值验证
                if (newValue < 0) {
                    XposedHelpers.log("Rejecting negative value: " + newValue);
                    param.setResult(null); // 阻止设置
                    return;
                }
                
                // 值转换
                if (newValue > 1000) {
                    param.args[0] = 1000; // 限制最大值
                }
                
                // 记录变更
                logValueChange(param.thisObject, newValue);
            }
        });
    }
    
    private void logValueChange(Object instance, int newValue) {
        XposedHelpers.log("Value changed to: " + newValue);
    }
}
```

## 高级 Hook 技术

### 1. 动态 Hook

根据运行时条件动态安装和卸载 Hook：

```java
public class DynamicHookManager {
    private static final Map<String, XC_MethodHook.Unhook> activeHooks = new ConcurrentHashMap<>();
    private static final Set<String> enabledFeatures = new ConcurrentSkipListSet<>();
    
    public static void enableFeature(String featureName, ClassLoader classLoader) {
        if (enabledFeatures.contains(featureName)) {
            return;
        }
        
        XC_MethodHook.Unhook unhook = installHookForFeature(featureName, classLoader);
        if (unhook != null) {
            activeHooks.put(featureName, unhook);
            enabledFeatures.add(featureName);
            XposedHelpers.log("Feature enabled: " + featureName);
        }
    }
    
    public static void disableFeature(String featureName) {
        XC_MethodHook.Unhook unhook = activeHooks.remove(featureName);
        if (unhook != null) {
            unhook.unhook();
            enabledFeatures.remove(featureName);
            XposedHelpers.log("Feature disabled: " + featureName);
        }
    }
    
    private static XC_MethodHook.Unhook installHookForFeature(String featureName, ClassLoader classLoader) {
        switch (featureName) {
            case "ad_blocker":
                return installAdBlockerHook(classLoader);
            case "performance_monitor":
                return installPerformanceMonitorHook(classLoader);
            case "security_enhancer":
                return installSecurityEnhancerHook(classLoader);
            default:
                XposedHelpers.log("Unknown feature: " + featureName);
                return null;
        }
    }
    
    private static XC_MethodHook.Unhook installAdBlockerHook(ClassLoader classLoader) {
        try {
            Class<?> adClass = XposedHelpers.findClass("com.example.AdManager", classLoader);
            return XposedHelpers.findAndHookMethod(adClass, "showAd", new XC_MethodReplacement() {
                @Override
                protected Object replaceHookedMethod(MethodHookParam param) throws Throwable {
                    XposedHelpers.log("Ad blocked!");
                    return null;
                }
            });
        } catch (Throwable t) {
            XposedHelpers.log("Failed to install ad blocker hook: " + t.getMessage());
            return null;
        }
    }
}
```

### 2. 条件 Hook

根据特定条件决定是否执行 Hook 逻辑：

```java
public class ConditionalHookWrapper extends XC_MethodHook {
    private final Predicate<MethodHookParam> condition;
    private final XC_MethodHook actualHook;
    
    public ConditionalHookWrapper(Predicate<MethodHookParam> condition, XC_MethodHook actualHook) {
        this.condition = condition;
        this.actualHook = actualHook;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        if (condition.test(param)) {
            actualHook.beforeHookedMethod(param);
        }
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        if (condition.test(param)) {
            actualHook.afterHookedMethod(param);
        }
    }
    
    // 工厂方法
    public static ConditionalHookWrapper when(Predicate<MethodHookParam> condition, XC_MethodHook hook) {
        return new ConditionalHookWrapper(condition, hook);
    }
    
    // 常用条件
    public static class Conditions {
        public static Predicate<MethodHookParam> callerIs(String className) {
            return param -> {
                StackTraceElement[] stack = Thread.currentThread().getStackTrace();
                for (StackTraceElement element : stack) {
                    if (element.getClassName().equals(className)) {
                        return true;
                    }
                }
                return false;
            };
        }
        
        public static Predicate<MethodHookParam> argumentEquals(int index, Object value) {
            return param -> {
                if (param.args.length > index) {
                    return Objects.equals(param.args[index], value);
                }
                return false;
            };
        }
        
        public static Predicate<MethodHookParam> threadNameContains(String substring) {
            return param -> Thread.currentThread().getName().contains(substring);
        }
    }
}

// 使用示例
public class ConditionalHookExample implements IXposedHookLoadPackage {
    
    @Override
    public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
        Class<?> targetClass = XposedHelpers.findClass("com.example.Target", lpparam.classLoader);
        
        // 只在特定调用者时执行 Hook
        XposedHelpers.findAndHookMethod(targetClass, "sensitiveMethod", String.class,
            ConditionalHookWrapper.when(
                Conditions.callerIs("com.example.TrustedClass"),
                new XC_MethodHook() {
                    @Override
                    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                        XposedHelpers.log("Sensitive method called by trusted class");
                    }
                }
            ));
            
        // 只在特定参数时执行 Hook
        XposedHelpers.findAndHookMethod(targetClass, "processData", String.class,
            ConditionalHookWrapper.when(
                Conditions.argumentEquals(0, "special"),
                new XC_MethodHook() {
                    @Override
                    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                        XposedHelpers.log("Processing special data");
                    }
                }
            ));
    }
}
```

### 3. 链式 Hook

创建 Hook 链，按顺序执行多个 Hook：

```java
public class HookChain extends XC_MethodHook {
    private final List<XC_MethodHook> hooks;
    
    public HookChain() {
        this.hooks = new ArrayList<>();
    }
    
    public HookChain addHook(XC_MethodHook hook) {
        hooks.add(hook);
        return this;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        for (XC_MethodHook hook : hooks) {
            hook.beforeHookedMethod(param);
            if (param.hasResult()) {
                // 如果某个 Hook 设置了结果，停止执行后续 Hook
                break;
            }
        }
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        // 逆序执行 after hook
        for (int i = hooks.size() - 1; i >= 0; i--) {
            hooks.get(i).afterHookedMethod(param);
        }
    }
    
    // 构建器模式
    public static class Builder {
        private final HookChain chain = new HookChain();
        
        public Builder addValidation(Predicate<Object[]> validator, String errorMessage) {
            chain.addHook(new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    if (!validator.test(param.args)) {
                        throw new IllegalArgumentException(errorMessage);
                    }
                }
            });
            return this;
        }
        
        public Builder addLogging(String logPrefix) {
            chain.addHook(new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    XposedHelpers.log(logPrefix + " - Before: " + Arrays.toString(param.args));
                }
                
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    XposedHelpers.log(logPrefix + " - After: " + param.getResult());
                }
            });
            return this;
        }
        
        public Builder addTransformation(Function<Object[], Object[]> transformer) {
            chain.addHook(new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    Object[] newArgs = transformer.apply(param.args);
                    System.arraycopy(newArgs, 0, param.args, 0, newArgs.length);
                }
            });
            return this;
        }
        
        public HookChain build() {
            return chain;
        }
    }
}

// 使用示例
public class HookChainExample implements IXposedHookLoadPackage {
    
    @Override
    public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
        Class<?> targetClass = XposedHelpers.findClass("com.example.Target", lpparam.classLoader);
        
        XposedHelpers.findAndHookMethod(targetClass, "processRequest", String.class, int.class,
            new HookChain.Builder()
                .addLogging("ProcessRequest")
                .addValidation(args -> args[0] != null && !((String) args[0]).isEmpty(), 
                              "Request cannot be null or empty")
                .addValidation(args -> (int) args[1] > 0, 
                              "Request ID must be positive")
                .addTransformation(args -> {
                    // 标准化请求格式
                    args[0] = ((String) args[0]).trim().toLowerCase();
                    return args;
                })
                .build());
    }
}
```

### 4. 异步 Hook

处理异步操作的 Hook：

```java
public class AsyncHookHandler extends XC_MethodHook {
    private final ExecutorService executor = Executors.newCachedThreadPool();
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        // 异步前置处理
        CompletableFuture.supplyAsync(() -> {
            return preprocessArgs(param.args);
        }, executor).thenAccept(processedArgs -> {
            // 更新参数（注意线程安全）
            synchronized (param) {
                System.arraycopy(processedArgs, 0, param.args, 0, processedArgs.length);
            }
        });
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        Object result = param.getResult();
        
        // 异步后置处理
        CompletableFuture.runAsync(() -> {
            postprocessResult(result);
        }, executor);
    }
    
    private Object[] preprocessArgs(Object[] args) {
        // 预处理逻辑
        Object[] processed = Arrays.copyOf(args, args.length);
        // ... 处理逻辑
        return processed;
    }
    
    private void postprocessResult(Object result) {
        // 后处理逻辑
        XposedHelpers.log("Async processing result: " + result);
    }
}
```

## Hook 性能优化

### 1. 性能监控

```java
public class PerformanceAwareHook extends XC_MethodHook {
    private static final long PERFORMANCE_THRESHOLD_MS = 1; // 1ms
    private final String hookName;
    
    public PerformanceAwareHook(String hookName) {
        this.hookName = hookName;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        long startTime = System.nanoTime();
        param.setObjectExtra("startTime", startTime);
        
        try {
            doBeforeHook(param);
        } finally {
            long duration = System.nanoTime() - startTime;
            if (duration > PERFORMANCE_THRESHOLD_MS * 1_000_000) {
                XposedHelpers.log("Slow before hook (" + hookName + "): " + 
                                (duration / 1_000_000) + "ms");
            }
        }
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        long startTime = System.nanoTime();
        
        try {
            doAfterHook(param);
        } finally {
            long duration = System.nanoTime() - startTime;
            if (duration > PERFORMANCE_THRESHOLD_MS * 1_000_000) {
                XposedHelpers.log("Slow after hook (" + hookName + "): " + 
                                (duration / 1_000_000) + "ms");
            }
            
            // 计算总耗时
            Long beforeStartTime = (Long) param.getObjectExtra("startTime");
            if (beforeStartTime != null) {
                long totalDuration = System.nanoTime() - beforeStartTime;
                if (totalDuration > PERFORMANCE_THRESHOLD_MS * 1_000_000) {
                    XposedHelpers.log("Slow total hook (" + hookName + "): " + 
                                    (totalDuration / 1_000_000) + "ms");
                }
            }
        }
    }
    
    protected void doBeforeHook(MethodHookParam param) throws Throwable {
        // 子类实现
    }
    
    protected void doAfterHook(MethodHookParam param) throws Throwable {
        // 子类实现
    }
}
```

### 2. 缓存优化

```java
public class CachedHook extends XC_MethodHook {
    private final LRUCache<String, Object> resultCache = new LRUCache<>(100);
    private final Function<Object[], String> keyGenerator;
    
    public CachedHook(Function<Object[], String> keyGenerator) {
        this.keyGenerator = keyGenerator;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        String cacheKey = keyGenerator.apply(param.args);
        Object cachedResult = resultCache.get(cacheKey);
        
        if (cachedResult != null) {
            param.setResult(cachedResult);
            XposedHelpers.log("Cache hit for key: " + cacheKey);
        } else {
            param.setObjectExtra("cacheKey", cacheKey);
        }
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        String cacheKey = (String) param.getObjectExtra("cacheKey");
        if (cacheKey != null && !param.hasThrowable()) {
            resultCache.put(cacheKey, param.getResult());
        }
    }
    
    // 简单的 LRU 缓存实现
    private static class LRUCache<K, V> extends LinkedHashMap<K, V> {
        private final int maxSize;
        
        public LRUCache(int maxSize) {
            super(16, 0.75f, true);
            this.maxSize = maxSize;
        }
        
        @Override
        protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
            return size() > maxSize;
        }
    }
}
```

## Hook 安全性

### 1. 权限检查

```java
public class SecureHook extends XC_MethodHook {
    private final Set<String> allowedCallers;
    
    public SecureHook(Set<String> allowedCallers) {
        this.allowedCallers = allowedCallers;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        if (!isCallerAllowed()) {
            XposedHelpers.log("Unauthorized access attempt blocked");
            throw new SecurityException("Access denied");
        }
        
        // 执行正常的 Hook 逻辑
        doSecureHook(param);
    }
    
    private boolean isCallerAllowed() {
        StackTraceElement[] stack = Thread.currentThread().getStackTrace();
        for (StackTraceElement element : stack) {
            String className = element.getClassName();
            if (allowedCallers.contains(className)) {
                return true;
            }
        }
        return false;
    }
    
    protected void doSecureHook(MethodHookParam param) throws Throwable {
        // 子类实现安全的 Hook 逻辑
    }
}
```

### 2. 参数验证

```java
public class ValidatingHook extends XC_MethodHook {
    private final List<Validator> validators = new ArrayList<>();
    
    public ValidatingHook addValidator(Validator validator) {
        validators.add(validator);
        return this;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        for (Validator validator : validators) {
            ValidationResult result = validator.validate(param.args);
            if (!result.isValid()) {
                XposedHelpers.log("Validation failed: " + result.getErrorMessage());
                throw new IllegalArgumentException(result.getErrorMessage());
            }
        }
    }
    
    public interface Validator {
        ValidationResult validate(Object[] args);
    }
    
    public static class ValidationResult {
        private final boolean valid;
        private final String errorMessage;
        
        public ValidationResult(boolean valid, String errorMessage) {
            this.valid = valid;
            this.errorMessage = errorMessage;
        }
        
        public boolean isValid() { return valid; }
        public String getErrorMessage() { return errorMessage; }
        
        public static ValidationResult success() {
            return new ValidationResult(true, null);
        }
        
        public static ValidationResult failure(String message) {
            return new ValidationResult(false, message);
        }
    }
    
    // 常用验证器
    public static class CommonValidators {
        public static Validator notNull(int argIndex) {
            return args -> {
                if (args.length <= argIndex || args[argIndex] == null) {
                    return ValidationResult.failure("Argument at index " + argIndex + " cannot be null");
                }
                return ValidationResult.success();
            };
        }
        
        public static Validator stringNotEmpty(int argIndex) {
            return args -> {
                if (args.length <= argIndex || 
                    !(args[argIndex] instanceof String) ||
                    ((String) args[argIndex]).isEmpty()) {
                    return ValidationResult.failure("String argument at index " + argIndex + " cannot be empty");
                }
                return ValidationResult.success();
            };
        }
        
        public static Validator positiveInteger(int argIndex) {
            return args -> {
                if (args.length <= argIndex || 
                    !(args[argIndex] instanceof Integer) ||
                    (Integer) args[argIndex] <= 0) {
                    return ValidationResult.failure("Integer argument at index " + argIndex + " must be positive");
                }
                return ValidationResult.success();
            };
        }
    }
}
```

## 调试和故障排除

### 1. Hook 调试工具

```java
public class DebugHook extends XC_MethodHook {
    private final String hookId;
    private final boolean enableStackTrace;
    
    public DebugHook(String hookId, boolean enableStackTrace) {
        this.hookId = hookId;
        this.enableStackTrace = enableStackTrace;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        StringBuilder log = new StringBuilder();
        log.append("[").append(hookId).append("] Before: ");
        log.append(param.method.getName()).append("(");
        
        for (int i = 0; i < param.args.length; i++) {
            if (i > 0) log.append(", ");
            log.append(formatArgument(param.args[i]));
        }
        log.append(")");
        
        if (enableStackTrace) {
            log.append("\nStack trace:\n");
            StackTraceElement[] stack = Thread.currentThread().getStackTrace();
            for (int i = 0; i < Math.min(stack.length, 10); i++) {
                log.append("  at ").append(stack[i]).append("\n");
            }
        }
        
        XposedHelpers.log(log.toString());
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        StringBuilder log = new StringBuilder();
        log.append("[").append(hookId).append("] After: ");
        log.append(param.method.getName());
        
        if (param.hasThrowable()) {
            log.append(" threw ").append(param.getThrowable().getClass().getSimpleName());
            log.append(": ").append(param.getThrowable().getMessage());
        } else {
            log.append(" returned ").append(formatArgument(param.getResult()));
        }
        
        XposedHelpers.log(log.toString());
    }
    
    private String formatArgument(Object arg) {
        if (arg == null) return "null";
        if (arg instanceof String) return "\"" + arg + "\"";
        if (arg instanceof Number || arg instanceof Boolean) return arg.toString();
        return arg.getClass().getSimpleName() + "@" + Integer.toHexString(arg.hashCode());
    }
}
```

### 2. Hook 健康检查

```java
public class HookHealthChecker {
    private static final Map<String, HookStats> hookStats = new ConcurrentHashMap<>();
    
    public static void recordHookCall(String hookId, long durationNs, boolean failed) {
        hookStats.computeIfAbsent(hookId, k -> new HookStats()).recordCall(durationNs, failed);
    }
    
    public static void performHealthCheck() {
        StringBuilder report = new StringBuilder("Hook Health Report:\n");
        
        for (Map.Entry<String, HookStats> entry : hookStats.entrySet()) {
            String hookId = entry.getKey();
            HookStats stats = entry.getValue();
            
            report.append("Hook: ").append(hookId).append("\n");
            report.append("  Calls: ").append(stats.totalCalls).append("\n");
            report.append("  Failures: ").append(stats.failureCount).append(" (")
                  .append(String.format("%.2f%%", stats.getFailureRate() * 100)).append(")\n");
            report.append("  Avg Duration: ").append(String.format("%.2f ms", stats.getAverageDuration() / 1_000_000.0)).append("\n");
            report.append("  Max Duration: ").append(String.format("%.2f ms", stats.maxDuration / 1_000_000.0)).append("\n");
            
            // 健康警告
            if (stats.getFailureRate() > 0.1) {
                report.append("  WARNING: High failure rate!\n");
            }
            if (stats.getAverageDuration() > 10_000_000) { // 10ms
                report.append("  WARNING: High average duration!\n");
            }
            
            report.append("\n");
        }
        
        XposedHelpers.log(report.toString());
    }
    
    private static class HookStats {
        private long totalCalls = 0;
        private long failureCount = 0;
        private long totalDuration = 0;
        private long maxDuration = 0;
        
        synchronized void recordCall(long durationNs, boolean failed) {
            totalCalls++;
            totalDuration += durationNs;
            maxDuration = Math.max(maxDuration, durationNs);
            
            if (failed) {
                failureCount++;
            }
        }
        
        double getFailureRate() {
            return totalCalls > 0 ? (double) failureCount / totalCalls : 0;
        }
        
        double getAverageDuration() {
            return totalCalls > 0 ? (double) totalDuration / totalCalls : 0;
        }
    }
}
```

## 总结

Hook 技术是 LSPosed 框架的核心，掌握这些技术可以让您：

1. **理解原理**: 深入了解 ART Hook 的工作机制
2. **灵活应用**: 根据需求选择合适的 Hook 技术
3. **性能优化**: 编写高效的 Hook 代码
4. **安全防护**: 实现安全的 Hook 机制
5. **调试排错**: 快速定位和解决问题

通过这些高级技术，您可以开发出功能强大、性能优秀、安全可靠的 Xposed 模块。

## 相关文档

- [代码分析指南](./code-analysis.md) - 深入理解源码实现
- [模块开发指南](./module-development.md) - 模块开发实践
- [性能优化](./performance-optimization.md) - 性能优化技巧
- [调试指南](./debugging-guide.md) - 调试方法和工具
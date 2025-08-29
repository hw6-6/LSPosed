# LSPosed 性能优化指南

本指南介绍如何优化 LSPosed 模块的性能，包括 Hook 性能优化、内存管理、启动优化等关键技术。

## 性能优化基础

### 1. 性能测量

#### 基准测试框架
```java
public class PerformanceBenchmark {
    private static final int WARMUP_ITERATIONS = 100;
    private static final int BENCHMARK_ITERATIONS = 1000;
    
    public static BenchmarkResult benchmarkMethod(String methodName, Runnable method) {
        // 预热
        for (int i = 0; i < WARMUP_ITERATIONS; i++) {
            method.run();
        }
        
        // 强制 GC
        System.gc();
        
        // 基准测试
        long startTime = System.nanoTime();
        long startMemory = getUsedMemory();
        
        for (int i = 0; i < BENCHMARK_ITERATIONS; i++) {
            method.run();
        }
        
        long endTime = System.nanoTime();
        long endMemory = getUsedMemory();
        
        long totalDuration = endTime - startTime;
        long memoryUsed = endMemory - startMemory;
        
        return new BenchmarkResult(
            methodName,
            totalDuration,
            totalDuration / BENCHMARK_ITERATIONS,
            memoryUsed,
            BENCHMARK_ITERATIONS
        );
    }
    
    private static long getUsedMemory() {
        Runtime runtime = Runtime.getRuntime();
        return runtime.totalMemory() - runtime.freeMemory();
    }
    
    public static class BenchmarkResult {
        public final String methodName;
        public final long totalDurationNs;
        public final long avgDurationNs;
        public final long memoryUsedBytes;
        public final int iterations;
        
        public BenchmarkResult(String methodName, long totalDurationNs, long avgDurationNs, 
                             long memoryUsedBytes, int iterations) {
            this.methodName = methodName;
            this.totalDurationNs = totalDurationNs;
            this.avgDurationNs = avgDurationNs;
            this.memoryUsedBytes = memoryUsedBytes;
            this.iterations = iterations;
        }
        
        @Override
        public String toString() {
            return String.format(
                "Benchmark [%s]: %.2f ms avg (%.2f ms total), Memory: %d KB, Iterations: %d",
                methodName,
                avgDurationNs / 1_000_000.0,
                totalDurationNs / 1_000_000.0,
                memoryUsedBytes / 1024,
                iterations
            );
        }
    }
}
```

#### 性能监控器
```java
public class PerformanceMonitor {
    private static final Map<String, PerformanceStats> stats = new ConcurrentHashMap<>();
    private static final AtomicLong totalHookCalls = new AtomicLong(0);
    private static final AtomicLong totalHookDuration = new AtomicLong(0);
    
    public static void recordHookCall(String hookName, long durationNs) {
        totalHookCalls.incrementAndGet();
        totalHookDuration.addAndGet(durationNs);
        
        stats.computeIfAbsent(hookName, k -> new PerformanceStats())
             .recordCall(durationNs);
    }
    
    public static void printPerformanceReport() {
        long totalCalls = totalHookCalls.get();
        long totalDuration = totalHookDuration.get();
        
        ModuleLogger.i("=== PERFORMANCE REPORT ===");
        ModuleLogger.i("Total Hook Calls: %d", totalCalls);
        ModuleLogger.i("Total Duration: %.2f ms", totalDuration / 1_000_000.0);
        ModuleLogger.i("Average Duration: %.3f ms", 
                      totalCalls > 0 ? (totalDuration / (double) totalCalls) / 1_000_000.0 : 0);
        
        ModuleLogger.i("\nPer-Hook Statistics:");
        stats.entrySet().stream()
             .sorted((a, b) -> Long.compare(b.getValue().getTotalDuration(), a.getValue().getTotalDuration()))
             .forEach(entry -> {
                 String hookName = entry.getKey();
                 PerformanceStats hookStats = entry.getValue();
                 
                 ModuleLogger.i("  %s:", hookName);
                 ModuleLogger.i("    Calls: %d (%.1f%%)", 
                               hookStats.getCallCount(),
                               totalCalls > 0 ? (hookStats.getCallCount() * 100.0 / totalCalls) : 0);
                 ModuleLogger.i("    Total: %.2f ms (%.1f%%)", 
                               hookStats.getTotalDuration() / 1_000_000.0,
                               totalDuration > 0 ? (hookStats.getTotalDuration() * 100.0 / totalDuration) : 0);
                 ModuleLogger.i("    Average: %.3f ms", hookStats.getAverageDuration() / 1_000_000.0);
                 ModuleLogger.i("    Min: %.3f ms", hookStats.getMinDuration() / 1_000_000.0);
                 ModuleLogger.i("    Max: %.3f ms", hookStats.getMaxDuration() / 1_000_000.0);
             });
        
        ModuleLogger.i("=== END REPORT ===");
    }
    
    private static class PerformanceStats {
        private final AtomicLong callCount = new AtomicLong(0);
        private final AtomicLong totalDuration = new AtomicLong(0);
        private volatile long minDuration = Long.MAX_VALUE;
        private volatile long maxDuration = 0;
        
        void recordCall(long durationNs) {
            callCount.incrementAndGet();
            totalDuration.addAndGet(durationNs);
            
            // 更新最小值和最大值
            synchronized (this) {
                minDuration = Math.min(minDuration, durationNs);
                maxDuration = Math.max(maxDuration, durationNs);
            }
        }
        
        long getCallCount() { return callCount.get(); }
        long getTotalDuration() { return totalDuration.get(); }
        long getMinDuration() { return minDuration == Long.MAX_VALUE ? 0 : minDuration; }
        long getMaxDuration() { return maxDuration; }
        
        double getAverageDuration() {
            long calls = callCount.get();
            return calls > 0 ? (double) totalDuration.get() / calls : 0;
        }
    }
}
```

## Hook 性能优化

### 1. 减少 Hook 开销

#### 快速路径优化
```java
public class FastPathHook extends XC_MethodHook {
    private final Predicate<Object[]> fastPathCondition;
    private final Function<Object[], Object> fastPathResult;
    
    public FastPathHook(Predicate<Object[]> condition, Function<Object[], Object> result) {
        this.fastPathCondition = condition;
        this.fastPathResult = result;
    }
    
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        // 快速路径检查
        if (fastPathCondition.test(param.args)) {
            Object result = fastPathResult.apply(param.args);
            param.setResult(result);
            return; // 跳过原方法和 afterHookedMethod
        }
        
        // 正常处理路径
        normalBeforeHook(param);
    }
    
    protected void normalBeforeHook(MethodHookParam param) throws Throwable {
        // 子类实现正常的 Hook 逻辑
    }
    
    // 工厂方法
    public static FastPathHook createCachedHook(Function<Object[], String> keyGenerator, 
                                               Function<String, Object> valueProvider) {
        Map<String, Object> cache = new ConcurrentHashMap<>();
        
        return new FastPathHook(
            args -> {
                String key = keyGenerator.apply(args);
                return cache.containsKey(key);
            },
            args -> {
                String key = keyGenerator.apply(args);
                return cache.computeIfAbsent(key, valueProvider);
            }
        );
    }
}
```

#### 条件 Hook 优化
```java
public class ConditionalHookOptimizer {
    
    // 使用位运算优化条件检查
    public static class BitwiseConditionHook extends XC_MethodHook {
        private final int requiredFlags;
        private final int forbiddenFlags;
        
        public BitwiseConditionHook(int requiredFlags, int forbiddenFlags) {
            this.requiredFlags = requiredFlags;
            this.forbiddenFlags = forbiddenFlags;
        }
        
        @Override
        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
            if (param.args.length > 0 && param.args[0] instanceof Integer) {
                int flags = (int) param.args[0];
                
                // 快速位运算检查
                if ((flags & requiredFlags) == requiredFlags && (flags & forbiddenFlags) == 0) {
                    executeHook(param);
                }
            }
        }
        
        protected void executeHook(MethodHookParam param) throws Throwable {
            // 子类实现
        }
    }
    
    // 使用枚举优化字符串比较
    public static class EnumBasedHook extends XC_MethodHook {
        private final Set<ActionType> allowedActions;
        
        public EnumBasedHook(Set<ActionType> allowedActions) {
            this.allowedActions = allowedActions;
        }
        
        @Override
        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
            if (param.args.length > 0 && param.args[0] instanceof String) {
                String actionStr = (String) param.args[0];
                try {
                    ActionType action = ActionType.valueOf(actionStr.toUpperCase());
                    if (allowedActions.contains(action)) {
                        executeHook(param, action);
                    }
                } catch (IllegalArgumentException e) {
                    // 忽略未知的 action
                }
            }
        }
        
        protected void executeHook(MethodHookParam param, ActionType action) throws Throwable {
            // 基于枚举的高效分发
            switch (action) {
                case CREATE:
                    handleCreate(param);
                    break;
                case UPDATE:
                    handleUpdate(param);
                    break;
                case DELETE:
                    handleDelete(param);
                    break;
            }
        }
        
        protected void handleCreate(MethodHookParam param) throws Throwable {}
        protected void handleUpdate(MethodHookParam param) throws Throwable {}
        protected void handleDelete(MethodHookParam param) throws Throwable {}
        
        enum ActionType {
            CREATE, UPDATE, DELETE, READ
        }
    }
}
```

### 2. 反射优化

#### 反射缓存系统
```java
public class ReflectionCache {
    private static final Map<String, Class<?>> classCache = new ConcurrentHashMap<>();
    private static final Map<String, Method> methodCache = new ConcurrentHashMap<>();
    private static final Map<String, Field> fieldCache = new ConcurrentHashMap<>();
    private static final Map<String, Constructor<?>> constructorCache = new ConcurrentHashMap<>();
    
    // 类缓存
    public static Class<?> getCachedClass(String className, ClassLoader classLoader) {
        String key = className + "@" + System.identityHashCode(classLoader);
        return classCache.computeIfAbsent(key, k -> {
            try {
                return Class.forName(className, false, classLoader);
            } catch (ClassNotFoundException e) {
                // 缓存失败结果，避免重复查找
                return null;
            }
        });
    }
    
    // 方法缓存
    public static Method getCachedMethod(Class<?> clazz, String methodName, Class<?>... parameterTypes) {
        String key = buildMethodKey(clazz, methodName, parameterTypes);
        return methodCache.computeIfAbsent(key, k -> {
            try {
                Method method = clazz.getDeclaredMethod(methodName, parameterTypes);
                method.setAccessible(true);
                return method;
            } catch (NoSuchMethodException e) {
                return null;
            }
        });
    }
    
    // 字段缓存
    public static Field getCachedField(Class<?> clazz, String fieldName) {
        String key = clazz.getName() + "#" + fieldName;
        return fieldCache.computeIfAbsent(key, k -> {
            try {
                Field field = clazz.getDeclaredField(fieldName);
                field.setAccessible(true);
                return field;
            } catch (NoSuchFieldException e) {
                return null;
            }
        });
    }
    
    // 构造函数缓存
    public static Constructor<?> getCachedConstructor(Class<?> clazz, Class<?>... parameterTypes) {
        String key = buildConstructorKey(clazz, parameterTypes);
        return constructorCache.computeIfAbsent(key, k -> {
            try {
                Constructor<?> constructor = clazz.getDeclaredConstructor(parameterTypes);
                constructor.setAccessible(true);
                return constructor;
            } catch (NoSuchMethodException e) {
                return null;
            }
        });
    }
    
    private static String buildMethodKey(Class<?> clazz, String methodName, Class<?>[] parameterTypes) {
        StringBuilder sb = new StringBuilder();
        sb.append(clazz.getName()).append("#").append(methodName).append("(");
        for (int i = 0; i < parameterTypes.length; i++) {
            if (i > 0) sb.append(",");
            sb.append(parameterTypes[i].getName());
        }
        sb.append(")");
        return sb.toString();
    }
    
    private static String buildConstructorKey(Class<?> clazz, Class<?>[] parameterTypes) {
        StringBuilder sb = new StringBuilder();
        sb.append(clazz.getName()).append("(");
        for (int i = 0; i < parameterTypes.length; i++) {
            if (i > 0) sb.append(",");
            sb.append(parameterTypes[i].getName());
        }
        sb.append(")");
        return sb.toString();
    }
    
    // 缓存清理
    public static void clearCache() {
        classCache.clear();
        methodCache.clear();
        fieldCache.clear();
        constructorCache.clear();
    }
    
    // 缓存统计
    public static void printCacheStats() {
        ModuleLogger.i("Reflection Cache Stats:");
        ModuleLogger.i("  Classes: %d", classCache.size());
        ModuleLogger.i("  Methods: %d", methodCache.size());
        ModuleLogger.i("  Fields: %d", fieldCache.size());
        ModuleLogger.i("  Constructors: %d", constructorCache.size());
    }
}
```

#### 预编译反射
```java
public class PrecompiledReflection {
    
    // 在模块初始化时预编译常用的反射操作
    public static class ReflectionCompiler {
        private final Map<String, CompiledReflection> compiledOperations = new HashMap<>();
        
        public void precompileCommonOperations(ClassLoader classLoader) {
            // 预编译常用类和方法
            compileOperation("ActivityManager.startActivity", () -> {
                Class<?> amClass = ReflectionCache.getCachedClass("android.app.ActivityManager", classLoader);
                Method startActivity = ReflectionCache.getCachedMethod(amClass, "startActivity", Intent.class);
                return new CompiledReflection.MethodInvocation(startActivity);
            });
            
            compileOperation("Context.getSystemService", () -> {
                Class<?> contextClass = ReflectionCache.getCachedClass("android.content.Context", classLoader);
                Method getSystemService = ReflectionCache.getCachedMethod(contextClass, "getSystemService", String.class);
                return new CompiledReflection.MethodInvocation(getSystemService);
            });
            
            // 预编译常用字段访问
            compileOperation("Activity.mIntent", () -> {
                Class<?> activityClass = ReflectionCache.getCachedClass("android.app.Activity", classLoader);
                Field mIntent = ReflectionCache.getCachedField(activityClass, "mIntent");
                return new CompiledReflection.FieldAccess(mIntent);
            });
        }
        
        private void compileOperation(String name, Supplier<CompiledReflection> compiler) {
            try {
                CompiledReflection compiled = compiler.get();
                if (compiled != null) {
                    compiledOperations.put(name, compiled);
                    ModuleLogger.d("Precompiled: %s", name);
                }
            } catch (Throwable t) {
                ModuleLogger.w("Failed to precompile %s: %s", name, t.getMessage());
            }
        }
        
        public CompiledReflection getCompiledOperation(String name) {
            return compiledOperations.get(name);
        }
    }
    
    // 编译后的反射操作
    public static abstract class CompiledReflection {
        public static class MethodInvocation extends CompiledReflection {
            private final Method method;
            
            public MethodInvocation(Method method) {
                this.method = method;
            }
            
            public Object invoke(Object target, Object... args) throws ReflectiveOperationException {
                return method.invoke(target, args);
            }
        }
        
        public static class FieldAccess extends CompiledReflection {
            private final Field field;
            
            public FieldAccess(Field field) {
                this.field = field;
            }
            
            public Object get(Object target) throws ReflectiveOperationException {
                return field.get(target);
            }
            
            public void set(Object target, Object value) throws ReflectiveOperationException {
                field.set(target, value);
            }
        }
    }
}
```

## 内存优化

### 1. 对象池

#### 通用对象池
```java
public class ObjectPool<T> {
    private final Queue<T> pool;
    private final Supplier<T> factory;
    private final Consumer<T> resetFunction;
    private final int maxSize;
    private final AtomicInteger currentSize = new AtomicInteger(0);
    
    public ObjectPool(Supplier<T> factory, Consumer<T> resetFunction, int maxSize) {
        this.pool = new ConcurrentLinkedQueue<>();
        this.factory = factory;
        this.resetFunction = resetFunction;
        this.maxSize = maxSize;
    }
    
    public T acquire() {
        T object = pool.poll();
        if (object == null) {
            object = factory.get();
        } else {
            currentSize.decrementAndGet();
        }
        return object;
    }
    
    public void release(T object) {
        if (object != null && currentSize.get() < maxSize) {
            try {
                resetFunction.accept(object);
                pool.offer(object);
                currentSize.incrementAndGet();
            } catch (Exception e) {
                // 重置失败，丢弃对象
                ModuleLogger.w("Failed to reset object for pool: %s", e.getMessage());
            }
        }
    }
    
    public int size() {
        return currentSize.get();
    }
    
    public void clear() {
        pool.clear();
        currentSize.set(0);
    }
}

// 常用对象池
public class CommonObjectPools {
    // StringBuilder 池
    public static final ObjectPool<StringBuilder> STRING_BUILDER_POOL = 
        new ObjectPool<>(
            () -> new StringBuilder(256),
            sb -> sb.setLength(0),
            50
        );
    
    // ArrayList 池
    public static final ObjectPool<ArrayList<Object>> ARRAY_LIST_POOL = 
        new ObjectPool<>(
            ArrayList::new,
            ArrayList::clear,
            20
        );
    
    // HashMap 池
    public static final ObjectPool<HashMap<String, Object>> HASH_MAP_POOL = 
        new ObjectPool<>(
            HashMap::new,
            HashMap::clear,
            20
        );
    
    // 使用示例
    public static String buildString(String... parts) {
        StringBuilder sb = STRING_BUILDER_POOL.acquire();
        try {
            for (String part : parts) {
                sb.append(part);
            }
            return sb.toString();
        } finally {
            STRING_BUILDER_POOL.release(sb);
        }
    }
}
```

### 2. 内存泄漏防护

#### 弱引用管理器
```java
public class WeakReferenceManager {
    private final Map<String, WeakReference<Object>> references = new ConcurrentHashMap<>();
    private final ScheduledExecutorService cleanupExecutor = 
        Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "WeakRefCleanup");
            t.setDaemon(true);
            return t;
        });
    
    public WeakReferenceManager() {
        // 定期清理失效的弱引用
        cleanupExecutor.scheduleAtFixedRate(this::cleanup, 30, 30, TimeUnit.SECONDS);
    }
    
    public void put(String key, Object value) {
        references.put(key, new WeakReference<>(value));
    }
    
    public <T> T get(String key, Class<T> type) {
        WeakReference<Object> ref = references.get(key);
        if (ref != null) {
            Object value = ref.get();
            if (value != null && type.isInstance(value)) {
                return type.cast(value);
            } else if (value == null) {
                // 对象已被 GC，移除引用
                references.remove(key);
            }
        }
        return null;
    }
    
    private void cleanup() {
        Iterator<Map.Entry<String, WeakReference<Object>>> iterator = references.entrySet().iterator();
        int removedCount = 0;
        
        while (iterator.hasNext()) {
            Map.Entry<String, WeakReference<Object>> entry = iterator.next();
            if (entry.getValue().get() == null) {
                iterator.remove();
                removedCount++;
            }
        }
        
        if (removedCount > 0) {
            ModuleLogger.d("Cleaned up %d weak references", removedCount);
        }
    }
    
    public void shutdown() {
        cleanupExecutor.shutdown();
        references.clear();
    }
}
```

#### 自动资源管理
```java
public class AutoResourceManager {
    
    // 自动关闭资源的 Hook
    public static class AutoCloseableHook extends XC_MethodHook {
        private final Set<Closeable> resources = ConcurrentHashMap.newKeySet();
        
        @Override
        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
            // 在方法开始时注册需要关闭的资源
            registerResources(param);
        }
        
        @Override
        protected void afterHookedMethod(MethodHookParam param) throws Throwable {
            // 方法结束时自动关闭资源
            closeAllResources();
        }
        
        protected void registerResources(MethodHookParam param) {
            // 子类实现资源注册逻辑
        }
        
        protected void addResource(Closeable resource) {
            if (resource != null) {
                resources.add(resource);
            }
        }
        
        private void closeAllResources() {
            for (Closeable resource : resources) {
                try {
                    resource.close();
                } catch (IOException e) {
                    ModuleLogger.w("Failed to close resource: %s", e.getMessage());
                }
            }
            resources.clear();
        }
    }
    
    // 使用 try-with-resources 模式的辅助类
    public static class ManagedResource<T> implements AutoCloseable {
        private final T resource;
        private final Consumer<T> cleanup;
        
        public ManagedResource(T resource, Consumer<T> cleanup) {
            this.resource = resource;
            this.cleanup = cleanup;
        }
        
        public T get() {
            return resource;
        }
        
        @Override
        public void close() {
            if (cleanup != null && resource != null) {
                try {
                    cleanup.accept(resource);
                } catch (Exception e) {
                    ModuleLogger.w("Failed to cleanup resource: %s", e.getMessage());
                }
            }
        }
        
        // 工厂方法
        public static <T> ManagedResource<T> of(T resource, Consumer<T> cleanup) {
            return new ManagedResource<>(resource, cleanup);
        }
    }
}
```

## 启动优化

### 1. 延迟初始化

#### 懒加载管理器
```java
public class LazyLoader {
    
    // 线程安全的延迟初始化
    public static class Lazy<T> {
        private volatile T value;
        private final Supplier<T> supplier;
        
        public Lazy(Supplier<T> supplier) {
            this.supplier = supplier;
        }
        
        public T get() {
            T result = value;
            if (result == null) {
                synchronized (this) {
                    result = value;
                    if (result == null) {
                        value = result = supplier.get();
                    }
                }
            }
            return result;
        }
        
        public boolean isInitialized() {
            return value != null;
        }
        
        public void reset() {
            synchronized (this) {
                value = null;
            }
        }
    }
    
    // 模块级别的延迟初始化
    public static class ModuleLazyInitializer {
        private final Map<String, Lazy<?>> lazyComponents = new ConcurrentHashMap<>();
        
        @SuppressWarnings("unchecked")
        public <T> T getComponent(String name, Supplier<T> supplier) {
            Lazy<T> lazy = (Lazy<T>) lazyComponents.computeIfAbsent(name, k -> new Lazy<>(supplier));
            return lazy.get();
        }
        
        public void preloadCriticalComponents() {
            // 预加载关键组件
            CompletableFuture.runAsync(() -> {
                lazyComponents.entrySet().stream()
                    .filter(entry -> entry.getKey().contains("critical"))
                    .forEach(entry -> {
                        try {
                            entry.getValue().get();
                            ModuleLogger.d("Preloaded: %s", entry.getKey());
                        } catch (Exception e) {
                            ModuleLogger.w("Failed to preload %s: %s", entry.getKey(), e.getMessage());
                        }
                    });
            });
        }
        
        public void printInitializationStatus() {
            ModuleLogger.i("Component Initialization Status:");
            lazyComponents.forEach((name, lazy) -> {
                ModuleLogger.i("  %s: %s", name, lazy.isInitialized() ? "Initialized" : "Lazy");
            });
        }
    }
}
```

### 2. 异步初始化

#### 异步任务管理器
```java
public class AsyncInitializer {
    private final ExecutorService initExecutor;
    private final CompletionService<Void> completionService;
    private final Map<String, Future<Void>> initTasks = new ConcurrentHashMap<>();
    
    public AsyncInitializer(int threadCount) {
        this.initExecutor = Executors.newFixedThreadPool(threadCount, r -> {
            Thread t = new Thread(r, "AsyncInit");
            t.setDaemon(true);
            return t;
        });
        this.completionService = new ExecutorCompletionService<>(initExecutor);
    }
    
    public void submitInitTask(String taskName, Runnable task) {
        Future<Void> future = completionService.submit(() -> {
            long startTime = System.nanoTime();
            try {
                task.run();
                long duration = System.nanoTime() - startTime;
                ModuleLogger.d("Init task [%s] completed in %.2f ms", taskName, duration / 1_000_000.0);
            } catch (Exception e) {
                ModuleLogger.e(e, "Init task [%s] failed", taskName);
                throw e;
            }
            return null;
        });
        
        initTasks.put(taskName, future);
    }
    
    public void waitForTask(String taskName, long timeoutMs) throws InterruptedException, ExecutionException, TimeoutException {
        Future<Void> future = initTasks.get(taskName);
        if (future != null) {
            future.get(timeoutMs, TimeUnit.MILLISECONDS);
        }
    }
    
    public void waitForAllTasks(long timeoutMs) throws InterruptedException {
        long deadline = System.currentTimeMillis() + timeoutMs;
        
        for (Map.Entry<String, Future<Void>> entry : initTasks.entrySet()) {
            long remainingTime = deadline - System.currentTimeMillis();
            if (remainingTime <= 0) {
                ModuleLogger.w("Timeout waiting for init tasks");
                break;
            }
            
            try {
                entry.getValue().get(remainingTime, TimeUnit.MILLISECONDS);
            } catch (Exception e) {
                ModuleLogger.w("Init task [%s] failed or timed out: %s", entry.getKey(), e.getMessage());
            }
        }
    }
    
    public void shutdown() {
        initExecutor.shutdown();
        try {
            if (!initExecutor.awaitTermination(5, TimeUnit.SECONDS)) {
                initExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            initExecutor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
    
    // 使用示例
    public static class ModuleAsyncInitializer {
        private final AsyncInitializer asyncInit = new AsyncInitializer(3);
        
        public void initializeModule(LoadPackageParam lpparam) {
            // 异步初始化各个组件
            asyncInit.submitInitTask("reflection-cache", () -> {
                ReflectionCache.preloadCommonClasses(lpparam.classLoader);
            });
            
            asyncInit.submitInitTask("hook-installer", () -> {
                installNonCriticalHooks(lpparam.classLoader);
            });
            
            asyncInit.submitInitTask("config-loader", () -> {
                loadConfiguration();
            });
            
            // 同步初始化关键组件
            initializeCriticalComponents(lpparam);
            
            // 等待异步任务完成
            try {
                asyncInit.waitForAllTasks(5000); // 5秒超时
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        
        private void initializeCriticalComponents(LoadPackageParam lpparam) {
            // 立即初始化关键组件
        }
        
        private void installNonCriticalHooks(ClassLoader classLoader) {
            // 安装非关键 Hook
        }
        
        private void loadConfiguration() {
            // 加载配置文件
        }
    }
}
```

## 数据结构优化

### 1. 高效集合

#### 优化的映射结构
```java
public class OptimizedCollections {
    
    // 针对小数据集优化的 Map
    public static class SmallMap<K, V> extends AbstractMap<K, V> {
        private static final int THRESHOLD = 8;
        private Object[] entries;
        private int size = 0;
        
        public SmallMap() {
            this.entries = new Object[THRESHOLD * 2];
        }
        
        @Override
        @SuppressWarnings("unchecked")
        public V get(Object key) {
            for (int i = 0; i < size * 2; i += 2) {
                if (Objects.equals(entries[i], key)) {
                    return (V) entries[i + 1];
                }
            }
            return null;
        }
        
        @Override
        public V put(K key, V value) {
            // 查找现有键
            for (int i = 0; i < size * 2; i += 2) {
                if (Objects.equals(entries[i], key)) {
                    @SuppressWarnings("unchecked")
                    V oldValue = (V) entries[i + 1];
                    entries[i + 1] = value;
                    return oldValue;
                }
            }
            
            // 添加新键值对
            if (size < THRESHOLD) {
                entries[size * 2] = key;
                entries[size * 2 + 1] = value;
                size++;
                return null;
            } else {
                throw new IllegalStateException("SmallMap capacity exceeded");
            }
        }
        
        @Override
        public Set<Entry<K, V>> entrySet() {
            Set<Entry<K, V>> entrySet = new HashSet<>();
            for (int i = 0; i < size * 2; i += 2) {
                @SuppressWarnings("unchecked")
                K key = (K) entries[i];
                @SuppressWarnings("unchecked")
                V value = (V) entries[i + 1];
                entrySet.add(new SimpleEntry<>(key, value));
            }
            return entrySet;
        }
        
        @Override
        public int size() {
            return size;
        }
    }
    
    // 基于位运算的 Set
    public static class IntBitSet {
        private long bits = 0;
        
        public void add(int value) {
            if (value >= 0 && value < 64) {
                bits |= (1L << value);
            }
        }
        
        public boolean contains(int value) {
            return value >= 0 && value < 64 && (bits & (1L << value)) != 0;
        }
        
        public void remove(int value) {
            if (value >= 0 && value < 64) {
                bits &= ~(1L << value);
            }
        }
        
        public int size() {
            return Long.bitCount(bits);
        }
        
        public boolean isEmpty() {
            return bits == 0;
        }
        
        public void clear() {
            bits = 0;
        }
    }
}
```

### 2. 缓存优化

#### LRU 缓存实现
```java
public class OptimizedLRUCache<K, V> {
    private final int maxSize;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head;
    private final Node<K, V> tail;
    
    public OptimizedLRUCache(int maxSize) {
        this.maxSize = maxSize;
        this.map = new HashMap<>(maxSize + 1, 1.0f);
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }
    
    public synchronized V get(K key) {
        Node<K, V> node = map.get(key);
        if (node != null) {
            moveToHead(node);
            return node.value;
        }
        return null;
    }
    
    public synchronized V put(K key, V value) {
        Node<K, V> existing = map.get(key);
        if (existing != null) {
            V oldValue = existing.value;
            existing.value = value;
            moveToHead(existing);
            return oldValue;
        }
        
        Node<K, V> newNode = new Node<>(key, value);
        map.put(key, newNode);
        addToHead(newNode);
        
        if (map.size() > maxSize) {
            Node<K, V> removed = removeTail();
            map.remove(removed.key);
        }
        
        return null;
    }
    
    private void moveToHead(Node<K, V> node) {
        removeNode(node);
        addToHead(node);
    }
    
    private void addToHead(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }
    
    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    
    private Node<K, V> removeTail() {
        Node<K, V> last = tail.prev;
        removeNode(last);
        return last;
    }
    
    public synchronized int size() {
        return map.size();
    }
    
    public synchronized void clear() {
        map.clear();
        head.next = tail;
        tail.prev = head;
    }
    
    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;
        
        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }
}
```

## 并发优化

### 1. 无锁编程

#### 无锁数据结构
```java
public class LockFreeStructures {
    
    // 无锁栈
    public static class LockFreeStack<T> {
        private final AtomicReference<Node<T>> top = new AtomicReference<>();
        
        public void push(T item) {
            Node<T> newNode = new Node<>(item);
            Node<T> currentTop;
            do {
                currentTop = top.get();
                newNode.next = currentTop;
            } while (!top.compareAndSet(currentTop, newNode));
        }
        
        public T pop() {
            Node<T> currentTop;
            Node<T> newTop;
            do {
                currentTop = top.get();
                if (currentTop == null) {
                    return null;
                }
                newTop = currentTop.next;
            } while (!top.compareAndSet(currentTop, newTop));
            
            return currentTop.item;
        }
        
        private static class Node<T> {
            final T item;
            volatile Node<T> next;
            
            Node(T item) {
                this.item = item;
            }
        }
    }
    
    // 无锁计数器
    public static class LockFreeCounter {
        private final AtomicLong counter = new AtomicLong(0);
        
        public long increment() {
            return counter.incrementAndGet();
        }
        
        public long decrement() {
            return counter.decrementAndGet();
        }
        
        public long get() {
            return counter.get();
        }
        
        public long addAndGet(long delta) {
            return counter.addAndGet(delta);
        }
    }
}
```

### 2. 线程池优化

#### 自适应线程池
```java
public class AdaptiveThreadPool {
    private final ThreadPoolExecutor executor;
    private final AtomicLong taskCount = new AtomicLong(0);
    private final AtomicLong completedTaskCount = new AtomicLong(0);
    private final ScheduledExecutorService monitor;
    
    public AdaptiveThreadPool(int corePoolSize, int maxPoolSize) {
        this.executor = new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            r -> {
                Thread t = new Thread(r, "AdaptivePool");
                t.setDaemon(true);
                return t;
            },
            new ThreadPoolExecutor.CallerRunsPolicy()
        ) {
            @Override
            protected void beforeExecute(Thread t, Runnable r) {
                taskCount.incrementAndGet();
            }
            
            @Override
            protected void afterExecute(Runnable r, Throwable t) {
                completedTaskCount.incrementAndGet();
            }
        };
        
        this.monitor = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "PoolMonitor");
            t.setDaemon(true);
            return t;
        });
        
        // 启动监控
        monitor.scheduleAtFixedRate(this::adjustPoolSize, 30, 30, TimeUnit.SECONDS);
    }
    
    public void submit(Runnable task) {
        executor.submit(task);
    }
    
    public <T> Future<T> submit(Callable<T> task) {
        return executor.submit(task);
    }
    
    private void adjustPoolSize() {
        int currentSize = executor.getPoolSize();
        int coreSize = executor.getCorePoolSize();
        int maxSize = executor.getMaximumPoolSize();
        int queueSize = executor.getQueue().size();
        
        // 获取最近的任务统计
        long currentTaskCount = taskCount.get();
        long currentCompletedCount = completedTaskCount.get();
        
        // 简单的自适应策略
        if (queueSize > 100 && currentSize < maxSize) {
            // 队列积压，增加线程
            executor.setCorePoolSize(Math.min(coreSize + 1, maxSize));
            ModuleLogger.d("Increased thread pool size to %d", coreSize + 1);
        } else if (queueSize < 10 && currentSize > 1) {
            // 队列空闲，减少线程
            executor.setCorePoolSize(Math.max(coreSize - 1, 1));
            ModuleLogger.d("Decreased thread pool size to %d", coreSize - 1);
        }
    }
    
    public void shutdown() {
        executor.shutdown();
        monitor.shutdown();
    }
    
    public ThreadPoolStats getStats() {
        return new ThreadPoolStats(
            executor.getPoolSize(),
            executor.getActiveCount(),
            executor.getQueue().size(),
            taskCount.get(),
            completedTaskCount.get()
        );
    }
    
    public static class ThreadPoolStats {
        public final int poolSize;
        public final int activeThreads;
        public final int queueSize;
        public final long totalTasks;
        public final long completedTasks;
        
        public ThreadPoolStats(int poolSize, int activeThreads, int queueSize, 
                             long totalTasks, long completedTasks) {
            this.poolSize = poolSize;
            this.activeThreads = activeThreads;
            this.queueSize = queueSize;
            this.totalTasks = totalTasks;
            this.completedTasks = completedTasks;
        }
        
        @Override
        public String toString() {
            return String.format(
                "ThreadPool[size=%d, active=%d, queue=%d, total=%d, completed=%d]",
                poolSize, activeThreads, queueSize, totalTasks, completedTasks
            );
        }
    }
}
```

## 性能监控和分析

### 1. 自动性能分析

#### 性能分析器
```java
public class AutoPerformanceAnalyzer {
    private final Map<String, PerformanceMetrics> metrics = new ConcurrentHashMap<>();
    private final ScheduledExecutorService analyzer;
    
    public AutoPerformanceAnalyzer() {
        this.analyzer = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "PerfAnalyzer");
            t.setDaemon(true);
            return t;
        });
        
        // 定期分析性能数据
        analyzer.scheduleAtFixedRate(this::analyzePerformance, 60, 60, TimeUnit.SECONDS);
    }
    
    public void recordMethodCall(String methodName, long durationNs, long memoryUsed) {
        metrics.computeIfAbsent(methodName, k -> new PerformanceMetrics())
               .recordCall(durationNs, memoryUsed);
    }
    
    private void analyzePerformance() {
        ModuleLogger.i("=== PERFORMANCE ANALYSIS ===");
        
        // 查找性能热点
        List<Map.Entry<String, PerformanceMetrics>> hotspots = metrics.entrySet().stream()
            .filter(entry -> entry.getValue().getCallCount() > 10)
            .sorted((a, b) -> Long.compare(
                b.getValue().getTotalDuration(),
                a.getValue().getTotalDuration()
            ))
            .limit(10)
            .collect(Collectors.toList());
        
        ModuleLogger.i("Top 10 Performance Hotspots:");
        for (int i = 0; i < hotspots.size(); i++) {
            Map.Entry<String, PerformanceMetrics> entry = hotspots.get(i);
            PerformanceMetrics metric = entry.getValue();
            
            ModuleLogger.i("  %d. %s", i + 1, entry.getKey());
            ModuleLogger.i("     Total: %.2f ms (%.1f%%)", 
                          metric.getTotalDuration() / 1_000_000.0,
                          calculatePercentage(metric.getTotalDuration()));
            ModuleLogger.i("     Calls: %d, Avg: %.3f ms", 
                          metric.getCallCount(),
                          metric.getAverageDuration() / 1_000_000.0);
        }
        
        // 查找内存消耗大户
        List<Map.Entry<String, PerformanceMetrics>> memoryHogs = metrics.entrySet().stream()
            .filter(entry -> entry.getValue().getTotalMemoryUsed() > 1024 * 1024) // > 1MB
            .sorted((a, b) -> Long.compare(
                b.getValue().getTotalMemoryUsed(),
                a.getValue().getTotalMemoryUsed()
            ))
            .limit(5)
            .collect(Collectors.toList());
        
        if (!memoryHogs.isEmpty()) {
            ModuleLogger.i("Top Memory Consumers:");
            for (Map.Entry<String, PerformanceMetrics> entry : memoryHogs) {
                PerformanceMetrics metric = entry.getValue();
                ModuleLogger.i("  %s: %.2f MB total, %.2f KB avg",
                              entry.getKey(),
                              metric.getTotalMemoryUsed() / 1024.0 / 1024.0,
                              metric.getAverageMemoryUsed() / 1024.0);
            }
        }
        
        ModuleLogger.i("=== END ANALYSIS ===");
    }
    
    private double calculatePercentage(long duration) {
        long totalDuration = metrics.values().stream()
            .mapToLong(PerformanceMetrics::getTotalDuration)
            .sum();
        return totalDuration > 0 ? (duration * 100.0 / totalDuration) : 0;
    }
    
    private static class PerformanceMetrics {
        private final AtomicLong callCount = new AtomicLong(0);
        private final AtomicLong totalDuration = new AtomicLong(0);
        private final AtomicLong totalMemoryUsed = new AtomicLong(0);
        
        void recordCall(long durationNs, long memoryUsed) {
            callCount.incrementAndGet();
            totalDuration.addAndGet(durationNs);
            totalMemoryUsed.addAndGet(memoryUsed);
        }
        
        long getCallCount() { return callCount.get(); }
        long getTotalDuration() { return totalDuration.get(); }
        long getTotalMemoryUsed() { return totalMemoryUsed.get(); }
        
        double getAverageDuration() {
            long calls = callCount.get();
            return calls > 0 ? (double) totalDuration.get() / calls : 0;
        }
        
        double getAverageMemoryUsed() {
            long calls = callCount.get();
            return calls > 0 ? (double) totalMemoryUsed.get() / calls : 0;
        }
    }
}
```

## 总结

LSPosed 模块的性能优化需要关注多个方面：

1. **Hook 性能**: 减少 Hook 开销，优化反射调用
2. **内存管理**: 合理使用对象池，防止内存泄漏
3. **启动优化**: 延迟初始化，异步加载
4. **数据结构**: 选择合适的数据结构和算法
5. **并发优化**: 使用无锁编程和优化的线程池
6. **监控分析**: 持续监控性能指标，及时发现问题

通过系统性的性能优化，您可以开发出高效、稳定的 LSPosed 模块，为用户提供良好的体验。

## 相关文档

- [Hook 技术详解](./hook-techniques.md) - Hook 技术原理和优化
- [代码分析指南](./code-analysis.md) - 源码分析和理解
- [调试指南](./debugging-guide.md) - 调试技巧和工具
- [模块开发指南](./module-development.md) - 模块开发实践
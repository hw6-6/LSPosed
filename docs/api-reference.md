# LSPosed API 参考

本文档提供 LSPosed 和 Xposed 框架的完整 API 参考，包括核心接口、工具类和最佳实践。

## 核心接口

### 1. IXposedHookLoadPackage

模块的主要入口点接口。

```java
public interface IXposedHookLoadPackage {
    /**
     * 当目标应用加载时调用
     * @param lpparam 加载包参数
     * @throws Throwable 任何异常
     */
    void handleLoadPackage(LoadPackageParam lpparam) throws Throwable;
    
    /**
     * 加载包参数类
     */
    class LoadPackageParam {
        /** 包名 */
        public String packageName;
        
        /** 应用进程名 */
        public String processName;
        
        /** 类加载器 */
        public ClassLoader classLoader;
        
        /** 应用信息 */
        public ApplicationInfo appInfo;
        
        /** 是否为第一个应用 */
        public boolean isFirstApplication;
    }
}
```

### 2. IXposedHookInitPackageResources

资源 Hook 接口（注意：LSPosed 可能不完全支持）。

```java
public interface IXposedHookInitPackageResources {
    /**
     * 当包资源初始化时调用
     * @param resparam 资源参数
     * @throws Throwable 任何异常
     */
    void handleInitPackageResources(InitPackageResourcesParam resparam) throws Throwable;
    
    class InitPackageResourcesParam {
        public String packageName;
        public XResources res;
    }
}
```

### 3. IXposedHookZygoteInit

Zygote 初始化 Hook 接口。

```java
public interface IXposedHookZygoteInit {
    /**
     * 在 Zygote 进程中调用
     * @param startupParam 启动参数
     * @throws Throwable 任何异常
     */
    void initZygote(StartupParam startupParam) throws Throwable;
    
    class StartupParam {
        /** 模块路径 */
        public String modulePath;
        
        /** 是否在启动阶段 */
        public boolean startsSystemServer;
    }
}
```

## Hook 相关类

### 1. XC_MethodHook

方法 Hook 的基础类。

```java
public abstract class XC_MethodHook extends XCallback {
    /**
     * 构造函数
     * @param priority Hook 优先级
     */
    public XC_MethodHook(int priority) { /* ... */ }
    
    /**
     * 在目标方法执行前调用
     * @param param 方法参数
     * @throws Throwable 任何异常
     */
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        // 默认实现为空
    }
    
    /**
     * 在目标方法执行后调用
     * @param param 方法参数
     * @throws Throwable 任何异常
     */
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        // 默认实现为空
    }
    
    /**
     * 方法 Hook 参数类
     */
    public static class MethodHookParam extends XCallback.Param {
        /** 被 Hook 的方法 */
        public Method method;
        
        /** this 对象（静态方法为 null） */
        public Object thisObject;
        
        /** 方法参数数组 */
        public Object[] args;
        
        /** 方法返回结果 */
        private Object result;
        
        /** 是否有返回结果 */
        private boolean returnEarly = false;
        
        /** 抛出的异常 */
        private Throwable throwable;
        
        /**
         * 获取返回结果
         */
        public Object getResult() {
            return result;
        }
        
        /**
         * 设置返回结果
         */
        public void setResult(Object result) {
            this.result = result;
            this.returnEarly = true;
        }
        
        /**
         * 是否有返回结果
         */
        public boolean hasResult() {
            return returnEarly;
        }
        
        /**
         * 获取抛出的异常
         */
        public Throwable getThrowable() {
            return throwable;
        }
        
        /**
         * 设置要抛出的异常
         */
        public void setThrowable(Throwable throwable) {
            this.throwable = throwable;
            this.returnEarly = true;
        }
        
        /**
         * 是否有异常
         */
        public boolean hasThrowable() {
            return throwable != null;
        }
        
        /**
         * 获取调用结果（返回值或异常）
         */
        public Object getResultOrThrowable() throws Throwable {
            if (throwable != null) {
                throw throwable;
            }
            return result;
        }
    }
    
    /**
     * Unhook 句柄
     */
    public class Unhook {
        /**
         * 移除这个 Hook
         */
        public void unhook() { /* ... */ }
    }
}
```

### 2. XC_MethodReplacement

方法替换类，完全替换目标方法的实现。

```java
public abstract class XC_MethodReplacement extends XC_MethodHook {
    /**
     * 构造函数
     * @param priority Hook 优先级
     */
    public XC_MethodReplacement(int priority) {
        super(priority);
    }
    
    @Override
    protected final void beforeHookedMethod(MethodHookParam param) throws Throwable {
        try {
            Object result = replaceHookedMethod(param);
            param.setResult(result);
        } catch (Throwable t) {
            param.setThrowable(t);
        }
    }
    
    @Override
    protected final void afterHookedMethod(MethodHookParam param) throws Throwable {
        // 不会被调用
    }
    
    /**
     * 替换方法的实现
     * @param param 方法参数
     * @return 返回值
     * @throws Throwable 任何异常
     */
    protected abstract Object replaceHookedMethod(MethodHookParam param) throws Throwable;
    
    /**
     * 创建返回 null 的替换
     */
    public static XC_MethodReplacement returnConstant(final Object result) {
        return new XC_MethodReplacement() {
            @Override
            protected Object replaceHookedMethod(MethodHookParam param) throws Throwable {
                return result;
            }
        };
    }
    
    /**
     * 创建什么都不做的替换
     */
    public static final XC_MethodReplacement DO_NOTHING = returnConstant(null);
}
```

## 工具类

### 1. XposedHelpers

提供便捷的反射和 Hook 操作。

```java
public final class XposedHelpers {
    
    // ==================== 类查找 ====================
    
    /**
     * 查找类
     * @param className 类名
     * @param classLoader 类加载器
     * @return 类对象
     * @throws ClassNotFoundError 类未找到
     */
    public static Class<?> findClass(String className, ClassLoader classLoader) { /* ... */ }
    
    /**
     * 查找类（可选）
     * @param className 类名
     * @param classLoader 类加载器
     * @return 类对象或 null
     */
    public static Class<?> findClassIfExists(String className, ClassLoader classLoader) { /* ... */ }
    
    // ==================== 方法查找和Hook ====================
    
    /**
     * 查找方法
     * @param clazz 目标类
     * @param methodName 方法名
     * @param parameterTypes 参数类型
     * @return 方法对象
     * @throws NoSuchMethodError 方法未找到
     */
    public static Method findMethodExact(Class<?> clazz, String methodName, Class<?>... parameterTypes) { /* ... */ }
    
    /**
     * 查找方法（模糊匹配）
     * @param clazz 目标类
     * @param methodName 方法名
     * @param parameterTypes 参数类型
     * @return 方法对象
     * @throws NoSuchMethodError 方法未找到
     */
    public static Method findMethodBest(Class<?> clazz, String methodName, Class<?>... parameterTypes) { /* ... */ }
    
    /**
     * 查找并 Hook 方法
     * @param clazz 目标类
     * @param methodName 方法名
     * @param parameterTypesAndCallback 参数类型和回调
     * @return Unhook 句柄
     */
    public static XC_MethodHook.Unhook findAndHookMethod(Class<?> clazz, String methodName, Object... parameterTypesAndCallback) { /* ... */ }
    
    /**
     * 查找并 Hook 方法
     * @param className 类名
     * @param classLoader 类加载器
     * @param methodName 方法名
     * @param parameterTypesAndCallback 参数类型和回调
     * @return Unhook 句柄
     */
    public static XC_MethodHook.Unhook findAndHookMethod(String className, ClassLoader classLoader, String methodName, Object... parameterTypesAndCallback) { /* ... */ }
    
    // ==================== 构造函数查找和Hook ====================
    
    /**
     * 查找构造函数
     * @param clazz 目标类
     * @param parameterTypes 参数类型
     * @return 构造函数对象
     * @throws NoSuchMethodError 构造函数未找到
     */
    public static Constructor<?> findConstructorExact(Class<?> clazz, Class<?>... parameterTypes) { /* ... */ }
    
    /**
     * 查找并 Hook 构造函数
     * @param clazz 目标类
     * @param parameterTypesAndCallback 参数类型和回调
     * @return Unhook 句柄
     */
    public static XC_MethodHook.Unhook findAndHookConstructor(Class<?> clazz, Object... parameterTypesAndCallback) { /* ... */ }
    
    /**
     * 查找并 Hook 构造函数
     * @param className 类名
     * @param classLoader 类加载器
     * @param parameterTypesAndCallback 参数类型和回调
     * @return Unhook 句柄
     */
    public static XC_MethodHook.Unhook findAndHookConstructor(String className, ClassLoader classLoader, Object... parameterTypesAndCallback) { /* ... */ }
    
    // ==================== 字段操作 ====================
    
    /**
     * 查找字段
     * @param clazz 目标类
     * @param fieldName 字段名
     * @return 字段对象
     * @throws NoSuchFieldError 字段未找到
     */
    public static Field findField(Class<?> clazz, String fieldName) { /* ... */ }
    
    /**
     * 获取对象字段值
     * @param obj 对象实例
     * @param fieldName 字段名
     * @return 字段值
     */
    public static Object getObjectField(Object obj, String fieldName) { /* ... */ }
    
    /**
     * 设置对象字段值
     * @param obj 对象实例
     * @param fieldName 字段名
     * @param value 新值
     */
    public static void setObjectField(Object obj, String fieldName, Object value) { /* ... */ }
    
    /**
     * 获取静态字段值
     * @param clazz 目标类
     * @param fieldName 字段名
     * @return 字段值
     */
    public static Object getStaticObjectField(Class<?> clazz, String fieldName) { /* ... */ }
    
    /**
     * 设置静态字段值
     * @param clazz 目标类
     * @param fieldName 字段名
     * @param value 新值
     */
    public static void setStaticObjectField(Class<?> clazz, String fieldName, Object value) { /* ... */ }
    
    // ==================== 基本类型字段操作 ====================
    
    public static boolean getBooleanField(Object obj, String fieldName) { /* ... */ }
    public static void setBooleanField(Object obj, String fieldName, boolean value) { /* ... */ }
    
    public static byte getByteField(Object obj, String fieldName) { /* ... */ }
    public static void setByteField(Object obj, String fieldName, byte value) { /* ... */ }
    
    public static char getCharField(Object obj, String fieldName) { /* ... */ }
    public static void setCharField(Object obj, String fieldName, char value) { /* ... */ }
    
    public static double getDoubleField(Object obj, String fieldName) { /* ... */ }
    public static void setDoubleField(Object obj, String fieldName, double value) { /* ... */ }
    
    public static float getFloatField(Object obj, String fieldName) { /* ... */ }
    public static void setFloatField(Object obj, String fieldName, float value) { /* ... */ }
    
    public static int getIntField(Object obj, String fieldName) { /* ... */ }
    public static void setIntField(Object obj, String fieldName, int value) { /* ... */ }
    
    public static long getLongField(Object obj, String fieldName) { /* ... */ }
    public static void setLongField(Object obj, String fieldName, long value) { /* ... */ }
    
    public static short getShortField(Object obj, String fieldName) { /* ... */ }
    public static void setShortField(Object obj, String fieldName, short value) { /* ... */ }
    
    // ==================== 方法调用 ====================
    
    /**
     * 调用对象方法
     * @param obj 对象实例
     * @param methodName 方法名
     * @param args 参数
     * @return 返回值
     */
    public static Object callMethod(Object obj, String methodName, Object... args) { /* ... */ }
    
    /**
     * 调用静态方法
     * @param clazz 目标类
     * @param methodName 方法名
     * @param args 参数
     * @return 返回值
     */
    public static Object callStaticMethod(Class<?> clazz, String methodName, Object... args) { /* ... */ }
    
    /**
     * 创建新实例
     * @param clazz 目标类
     * @param args 构造函数参数
     * @return 新实例
     */
    public static Object newInstance(Class<?> clazz, Object... args) { /* ... */ }
    
    // ==================== 其他工具方法 ====================
    
    /**
     * 输出日志
     * @param text 日志内容
     */
    public static void log(String text) { /* ... */ }
    
    /**
     * 输出错误日志
     * @param text 日志内容
     * @param t 异常
     */
    public static void log(String text, Throwable t) { /* ... */ }
    
    /**
     * 获取所有字段映射
     * @param clazz 目标类
     * @return 字段映射
     */
    public static Map<String, Field> getFieldsMap(Class<?> clazz) { /* ... */ }
    
    /**
     * 获取所有方法映射
     * @param clazz 目标类
     * @return 方法映射
     */
    public static Map<String, Method> getMethodsMap(Class<?> clazz) { /* ... */ }
    
    /**
     * 异常类
     */
    public static class ClassNotFoundError extends Error { /* ... */ }
    public static class InvocationTargetError extends Error { /* ... */ }
}
```

### 2. XposedBridge

框架核心类，提供底层 Hook 功能。

```java
public final class XposedBridge {
    /**
     * Xposed 版本
     */
    public static int XPOSED_BRIDGE_VERSION;
    
    /**
     * Hook 方法
     * @param hookMethod 要 Hook 的方法
     * @param callback 回调
     * @return Unhook 句柄
     */
    public static XC_MethodHook.Unhook hookMethod(Member hookMethod, XC_MethodHook callback) { /* ... */ }
    
    /**
     * 调用原始方法
     * @param method 方法
     * @param thisObject this 对象
     * @param args 参数
     * @return 返回值
     * @throws Throwable 任何异常
     */
    public static Object invokeOriginalMethod(Member method, Object thisObject, Object[] args) throws Throwable { /* ... */ }
    
    /**
     * 反 Hook 方法
     * @param hookMethod 要反 Hook 的方法
     * @param callback 回调
     */
    public static void unhookMethod(Member hookMethod, XC_MethodHook callback) { /* ... */ }
    
    /**
     * 获取 Hook 回调副本
     * @param method 方法
     * @return 回调副本
     */
    public static Set<XC_MethodHook> getHookedMethodCallbacks(Member method) { /* ... */ }
    
    /**
     * 日志输出
     * @param text 日志内容
     */
    public static void log(String text) { /* ... */ }
    
    /**
     * 日志输出
     * @param t 异常
     */
    public static void log(Throwable t) { /* ... */ }
}
```

## LSPosed 扩展 API

### 1. LSPosedContext（LibXposed）

```java
@SuppressLint("NewApi")
public class LSPosedContext implements XposedInterface {
    
    /**
     * 获取应用信息
     * @return 应用信息
     */
    @NonNull
    public ApplicationInfo getApplicationInfo() { /* ... */ }
    
    /**
     * 获取远程偏好设置
     * @param name 偏好设置名称
     * @return 偏好设置对象
     */
    @NonNull
    public SharedPreferences getRemotePreferences(String name) { /* ... */ }
    
    /**
     * 列出远程文件
     * @return 文件名数组
     */
    @NonNull
    public String[] listRemoteFiles() { /* ... */ }
    
    /**
     * 打开远程文件
     * @param name 文件名
     * @return 文件描述符
     * @throws FileNotFoundException 文件未找到
     */
    @NonNull
    public ParcelFileDescriptor openRemoteFile(String name) throws FileNotFoundException { /* ... */ }
    
    /**
     * 解析 DEX 文件
     * @param dexData DEX 数据
     * @param includeAnnotations 是否包含注解
     * @return DEX 解析器
     * @throws IOException IO 异常
     */
    public DexParser parseDex(@NonNull ByteBuffer dexData, boolean includeAnnotations) throws IOException { /* ... */ }
    
    /**
     * 调用特殊方法
     * @param method 方法
     * @param thisObject this 对象
     * @param args 参数
     * @return 返回值
     * @throws InvocationTargetException 调用异常
     * @throws IllegalArgumentException 参数异常
     * @throws IllegalAccessException 访问异常
     */
    public Object invokeSpecial(@NonNull Method method, @NonNull Object thisObject, Object... args) 
            throws InvocationTargetException, IllegalArgumentException, IllegalAccessException { /* ... */ }
    
    /**
     * 创建原始实例
     * @param constructor 构造函数
     * @param args 参数
     * @return 新实例
     * @throws InvocationTargetException 调用异常
     * @throws IllegalAccessException 访问异常
     * @throws InstantiationException 实例化异常
     */
    @NonNull
    public <T> T newInstanceOrigin(@NonNull Constructor<T> constructor, Object... args) 
            throws InvocationTargetException, IllegalAccessException, InstantiationException { /* ... */ }
    
    /**
     * 输出日志
     * @param message 日志消息
     */
    public void log(@NonNull String message) { /* ... */ }
    
    /**
     * 输出错误日志
     * @param message 日志消息
     * @param throwable 异常
     */
    public void log(@NonNull String message, @NonNull Throwable throwable) { /* ... */ }
}
```

### 2. LSPosedHelper

```java
public class LSPosedHelper {
    
    /**
     * Hook 方法
     * @param hooker Hook 类
     * @param clazz 目标类
     * @param methodName 方法名
     * @param parameterTypes 参数类型
     * @return Unhook 句柄
     */
    @SuppressWarnings("UnusedReturnValue")
    public static <T> XposedInterface.MethodUnhooker<Method>
    hookMethod(Class<? extends XposedInterface.Hooker> hooker, Class<T> clazz, String methodName, Class<?>... parameterTypes) { /* ... */ }
    
    /**
     * Hook 所有同名方法
     * @param hooker Hook 类
     * @param clazz 目标类
     * @param methodName 方法名
     * @return Unhook 句柄集合
     */
    @SuppressWarnings("UnusedReturnValue")
    public static <T> Set<XposedInterface.MethodUnhooker<Method>>
    hookAllMethods(Class<? extends XposedInterface.Hooker> hooker, Class<T> clazz, String methodName) { /* ... */ }
    
    /**
     * Hook 构造函数
     * @param hooker Hook 类
     * @param clazz 目标类
     * @param parameterTypes 参数类型
     * @return Unhook 句柄
     */
    @SuppressWarnings("UnusedReturnValue")
    public static <T> XposedInterface.MethodUnhooker<Constructor<T>>
    hookConstructor(Class<? extends XposedInterface.Hooker> hooker, Class<T> clazz, Class<?>... parameterTypes) { /* ... */ }
}
```

## 常量和优先级

### 1. Hook 优先级

```java
public abstract class XCallback {
    /** 最高优先级 */
    public static final int PRIORITY_HIGHEST = 10000;
    
    /** 默认优先级 */
    public static final int PRIORITY_DEFAULT = 50;
    
    /** 最低优先级 */
    public static final int PRIORITY_LOWEST = -10000;
}
```

### 2. 系统属性

```java
// Android 版本检查
int sdkInt = Build.VERSION.SDK_INT;
if (sdkInt >= Build.VERSION_CODES.TIRAMISU) { // Android 13
    // 处理 Android 13+ 的逻辑
}

// 设备信息
String brand = Build.BRAND;
String manufacturer = Build.MANUFACTURER;
String device = Build.DEVICE;
String model = Build.MODEL;
```

## 使用示例

### 1. 基础 Hook 示例

```java
public class BasicHookExample implements IXposedHookLoadPackage {
    
    @Override
    public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
        if (!"com.example.targetapp".equals(lpparam.packageName)) {
            return;
        }
        
        // Hook 方法
        Class<?> targetClass = XposedHelpers.findClass("com.example.TargetClass", lpparam.classLoader);
        
        XposedHelpers.findAndHookMethod(targetClass, "targetMethod", 
            String.class, int.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    String strParam = (String) param.args[0];
                    int intParam = (int) param.args[1];
                    XposedHelpers.log("Method called with: " + strParam + ", " + intParam);
                    
                    // 修改参数
                    param.args[0] = "Modified: " + strParam;
                }
                
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    Object result = param.getResult();
                    XposedHelpers.log("Method returned: " + result);
                    
                    // 修改返回值
                    param.setResult("Hooked: " + result);
                }
            });
    }
}
```

### 2. 方法替换示例

```java
public class MethodReplacementExample implements IXposedHookLoadPackage {
    
    @Override
    public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
        if (!"com.example.targetapp".equals(lpparam.packageName)) {
            return;
        }
        
        Class<?> targetClass = XposedHelpers.findClass("com.example.TargetClass", lpparam.classLoader);
        
        // 完全替换方法实现
        XposedHelpers.findAndHookMethod(targetClass, "methodToReplace", 
            String.class, new XC_MethodReplacement() {
                @Override
                protected Object replaceHookedMethod(MethodHookParam param) throws Throwable {
                    String input = (String) param.args[0];
                    
                    // 自定义实现
                    return "Replaced implementation: " + input;
                }
            });
            
        // 返回常量值
        XposedHelpers.findAndHookMethod(targetClass, "alwaysReturnTrue", 
            XC_MethodReplacement.returnConstant(true));
            
        // 什么都不做
        XposedHelpers.findAndHookMethod(targetClass, "doNothing", 
            XC_MethodReplacement.DO_NOTHING);
    }
}
```

### 3. 字段操作示例

```java
public class FieldOperationExample implements IXposedHookLoadPackage {
    
    @Override
    public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
        if (!"com.example.targetapp".equals(lpparam.packageName)) {
            return;
        }
        
        Class<?> targetClass = XposedHelpers.findClass("com.example.TargetClass", lpparam.classLoader);
        
        // Hook 构造函数来修改字段
        XposedHelpers.findAndHookConstructor(targetClass, String.class, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                // 修改实例字段
                XposedHelpers.setObjectField(param.thisObject, "stringField", "Modified value");
                XposedHelpers.setIntField(param.thisObject, "intField", 42);
                XposedHelpers.setBooleanField(param.thisObject, "booleanField", true);
            }
        });
        
        // Hook 方法来读取字段
        XposedHelpers.findAndHookMethod(targetClass, "getFieldValues", new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                // 读取字段值
                String stringValue = (String) XposedHelpers.getObjectField(param.thisObject, "stringField");
                int intValue = XposedHelpers.getIntField(param.thisObject, "intField");
                boolean booleanValue = XposedHelpers.getBooleanField(param.thisObject, "booleanField");
                
                XposedHelpers.log("Field values: " + stringValue + ", " + intValue + ", " + booleanValue);
            }
        });
    }
}
```

## 最佳实践

### 1. 错误处理

```java
// 安全的 Hook 实现
public static boolean safeHook(ClassLoader classLoader, String className, String methodName, XC_MethodHook hook) {
    try {
        Class<?> clazz = XposedHelpers.findClass(className, classLoader);
        XposedHelpers.findAndHookMethod(clazz, methodName, hook);
        return true;
    } catch (Throwable t) {
        XposedHelpers.log("Failed to hook " + className + "." + methodName + ": " + t.getMessage());
        return false;
    }
}
```

### 2. 性能优化

```java
// 缓存反射结果
private static final Map<String, Class<?>> classCache = new ConcurrentHashMap<>();
private static final Map<String, Method> methodCache = new ConcurrentHashMap<>();

public static Class<?> getCachedClass(String className, ClassLoader classLoader) {
    return classCache.computeIfAbsent(className, name -> {
        try {
            return XposedHelpers.findClass(name, classLoader);
        } catch (XposedHelpers.ClassNotFoundError e) {
            return null;
        }
    });
}
```

### 3. 版本兼容性

```java
// API 级别检查
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    hookForAndroid13Plus(lpparam);
} else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    hookForAndroid12Plus(lpparam);
} else {
    hookForOlderVersions(lpparam);
}
```

这份 API 参考涵盖了 LSPosed 和 Xposed 框架的主要接口和使用方法。通过这些 API，您可以实现强大的应用 Hook 功能。

## 相关文档

- [模块开发指南](./module-development.md) - 模块开发实践
- [代码分析指南](./code-analysis.md) - 深入理解源码
- [Hook 技术详解](./hook-techniques.md) - Hook 技术原理
- [常见问题](./faq.md) - 常见问题解答
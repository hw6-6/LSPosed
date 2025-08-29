# LSPosed 架构概述

本文档深入介绍 LSPosed 框架的技术架构，帮助开发者理解其工作原理和设计哲学。

## 整体架构

LSPosed 采用分层架构设计，主要包含以下组件：

```
┌─────────────────────────────────────────┐
│              用户空间                    │
├─────────────────────────────────────────┤
│  LSPosed Manager │    Xposed Modules    │
├─────────────────────────────────────────┤
│         LSPosed Core Framework          │
├─────────────────────────────────────────┤
│    LSPlant Hook Engine │ ART Runtime    │
├─────────────────────────────────────────┤
│          Zygisk/Riru Injection          │
├─────────────────────────────────────────┤
│             Android System              │
└─────────────────────────────────────────┘
```

## 核心组件

### 1. 注入层（Injection Layer）

#### Zygisk 注入
```cpp
// 基于 Magisk Zygisk 的注入机制
namespace zygisk {
    void preAppSpecialize(AppSpecializeArgs *args) {
        // 在应用进程创建前注入 LSPosed
        if (shouldInject(args->nice_name)) {
            injectLSPosed();
        }
    }
}
```

**特点**：
- ✅ 更稳定的注入机制
- ✅ 更好的性能表现
- ✅ 与 Magisk 深度集成

#### Riru 注入
```cpp
// 基于 Riru 的注入机制
extern "C" void specializeAppProcess(
    JNIEnv *env,
    jclass clazz,
    jint *uid,
    jint *gid,
    jintArray *gids,
    jint *runtimeFlags,
    jobjectArray *rlimits,
    jint *mountExternal,
    jstring *seInfo,
    jstring *niceName,
    jintArray *fdsToClose,
    jintArray *fdsToIgnore,
    jboolean *is_child_zygote,
    jstring *instructionSet,
    jstring *appDataDir) {
    // Riru 回调处理
}
```

### 2. Hook 引擎（Hook Engine）

LSPosed 使用 [LSPlant](https://github.com/LSPosed/LSPlant) 作为底层 Hook 引擎：

#### ART Hook 原理
```cpp
// LSPlant Hook 实现
namespace lsplant {
    bool Hook(JNIEnv* env, jobject target_method, jobject hooker_object) {
        // 1. 获取 ArtMethod 结构
        ArtMethod* target = art::ArtMethod::FromReflectedMethod(env, target_method);
        
        // 2. 创建 Hook 桥接
        auto* hooker = CreateHooker(target, hooker_object);
        
        // 3. 修改 entry_point_from_quick_compiled_code_
        target->SetEntryPointFromQuickCompiledCode(hooker->GetEntryPoint());
        
        return true;
    }
}
```

#### Hook 流程
1. **方法拦截**: 修改目标方法的入口点
2. **参数传递**: 将原始参数传递给 Hook 处理器
3. **结果处理**: 处理返回值和异常

### 3. 桥接层（Bridge Layer）

#### LSPosedBridge 类
```java
public class LSPosedBridge {
    // 核心 Hook 方法
    public static <T extends Executable> MethodUnhooker<T> hookMethod(
            Class<? extends Hooker> hookerClass,
            T method,
            int priority,
            Object callback) {
        return doHook(method, priority, hookerClass);
    }
    
    // 原始方法调用
    public static Object invokeOriginalMethod(Method method, Object thisObject, Object... args) 
            throws InvocationTargetException, IllegalAccessException {
        return HookBridge.invokeOriginalMethod(method, thisObject, args);
    }
}
```

#### 回调机制
```java
public class LSPosedHookCallback<T extends Executable> implements XposedCallback<T> {
    @Override
    protected void beforeHookedMethod(MethodHookParam<T> param) throws Throwable {
        // Hook 前执行
    }
    
    @Override
    protected void afterHookedMethod(MethodHookParam<T> param) throws Throwable {
        // Hook 后执行
    }
}
```

### 4. 模块管理（Module Management）

#### 模块加载流程
```java
public class LSPosedContext implements XposedInterface {
    // 模块初始化
    public void initModule(String moduleName, ClassLoader classLoader) {
        try {
            // 1. 加载模块类
            Class<?> moduleClass = classLoader.loadClass(moduleName + ".MainHook");
            
            // 2. 调用 handleLoadPackage
            Method handleLoadPackage = moduleClass.getMethod("handleLoadPackage", 
                LoadPackageParam.class);
            
            // 3. 传递应用信息
            LoadPackageParam param = new LoadPackageParam();
            param.packageName = getCurrentPackageName();
            param.classLoader = classLoader;
            
            handleLoadPackage.invoke(null, param);
        } catch (Exception e) {
            log("Failed to init module: " + moduleName, e);
        }
    }
}
```

## 数据流

### 1. 应用启动流程
```
Zygote Fork → App Process → LSPosed Inject → Module Load → Hook Install
```

### 2. Hook 执行流程
```
Method Call → Hook Bridge → Before Hook → Original Method → After Hook → Return
```

### 3. 模块通信
```java
// 跨进程通信示例
public class RemotePreferences {
    private final ILSPosedService mService;
    
    public void putString(String key, String value) {
        try {
            mService.putString(getModuleName(), key, value);
        } catch (RemoteException e) {
            // 处理异常
        }
    }
}
```

## 内存管理

### 1. Hook 信息存储
```cpp
namespace lspd {
    // Hook 信息管理
    class HookItem {
    private:
        ArtMethod* target_method_;
        ArtMethod* backup_method_;
        jobject callback_object_;
        
    public:
        void Install() {
            // 安装 Hook
        }
        
        void Uninstall() {
            // 卸载 Hook
        }
    };
    
    // 全局 Hook 映射
    std::unordered_map<ArtMethod*, std::unique_ptr<HookItem>> hooked_methods;
}
```

### 2. 内存优化策略
- **延迟加载**: 只在需要时加载模块
- **弱引用**: 避免内存泄漏
- **缓存机制**: 减少重复计算

## 性能优化

### 1. Hook 性能
```cpp
// 快速路径优化
class FastHook {
    static constexpr size_t kQuickEntryOffset = 32;
    
    void InstallFastHook(ArtMethod* target) {
        // 直接修改机器码入口点
        void* quick_entry = reinterpret_cast<char*>(target) + kQuickEntryOffset;
        *static_cast<void**>(quick_entry) = GetHookEntryPoint();
    }
};
```

### 2. 内存优化
- **对象池**: 复用常用对象
- **JNI 优化**: 减少 JNI 调用开销
- **智能缓存**: 缓存反射结果

## 安全机制

### 1. 权限控制
```java
public class SecurityManager {
    public boolean checkModuleAccess(String packageName, String moduleName) {
        // 检查模块权限
        if (!isModuleEnabled(moduleName)) {
            return false;
        }
        
        if (!hasScope(moduleName, packageName)) {
            return false;
        }
        
        return true;
    }
}
```

### 2. 沙箱隔离
- **进程隔离**: 每个应用独立的 Hook 环境
- **权限检查**: 严格的权限验证
- **资源限制**: 防止资源滥用

## 调试支持

### 1. 日志系统
```java
public class LSPosedLogger {
    public static void log(String tag, String message) {
        // 统一日志接口
        android.util.Log.i("LSPosed-" + tag, message);
    }
    
    public static void logError(String tag, String message, Throwable e) {
        android.util.Log.e("LSPosed-" + tag, message, e);
    }
}
```

### 2. 调试工具
- **Hook 跟踪**: 追踪 Hook 调用链
- **性能分析**: 分析 Hook 性能影响
- **内存监控**: 监控内存使用情况

## 兼容性设计

### 1. Android 版本兼容
```java
public class CompatibilityHelper {
    public static boolean isMethodHookable(Method method) {
        int sdkInt = Build.VERSION.SDK_INT;
        
        // Android 8.1+
        if (sdkInt >= Build.VERSION_CODES.O_MR1) {
            return true;
        }
        
        return false;
    }
}
```

### 2. 模块兼容
- **API 版本控制**: 严格的 API 版本管理
- **向后兼容**: 支持旧版本模块
- **渐进式升级**: 平滑的版本过渡

## 扩展机制

### 1. 插件系统
```java
public interface LSPosedPlugin {
    void onModuleLoaded(String packageName, ClassLoader classLoader);
    void onMethodHooked(Method method, Object callback);
    void onApplicationCreated(Application app);
}
```

### 2. 自定义 Hook
```java
public class CustomHook {
    @BeforeInvocation
    public void beforeMethod(MethodHookParam param) {
        // 自定义前置处理
    }
    
    @AfterInvocation
    public void afterMethod(MethodHookParam param) {
        // 自定义后置处理
    }
}
```

## 总结

LSPosed 架构的核心优势：

1. **高性能**: 基于 ART 的原生 Hook
2. **高稳定性**: 完善的错误处理和恢复机制
3. **高兼容性**: 支持广泛的 Android 版本
4. **易扩展**: 灵活的插件和模块系统
5. **安全性**: 多层次的安全防护机制

理解这些架构原理将帮助您更好地开发和调试 Xposed 模块。

## 相关文档

- [代码分析指南](./code-analysis.md) - 深入分析源码
- [模块开发指南](./module-development.md) - 模块开发实践
- [Hook 技术详解](./hook-techniques.md) - Hook 技术原理
- [性能优化](./performance-optimization.md) - 性能优化技巧
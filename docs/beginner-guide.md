# LSPosed 初学者指南

本指南将帮助您了解 LSPosed 是什么，如何使用它，以及如何开始您的第一个模块开发。

## 什么是 LSPosed？

LSPosed 是一个基于 Riru/Zygisk 的 ART Hook 框架，它提供与原始 Xposed 框架一致的 API，利用 LSPlant Hook 框架实现。

### 核心概念

**Xposed 框架** 是一个允许模块在不修改 APK 文件的情况下改变系统和应用程序行为的框架。这意味着：

- ✅ 模块可以在不同版本和 ROM 上工作（只要原始代码没有太大变化）
- ✅ 易于撤销 - 所有更改都在内存中进行，只需停用模块并重启即可恢复
- ✅ 多个模块可以同时修改系统或应用的同一部分
- ✅ 无需重新打包 APK 文件

## 系统要求

- **Android 版本**: 8.1 ~ 14
- **Root 权限**: 必须（通过 Magisk）
- **Magisk 版本**: v24+
- **Riru 版本**: v26.1.7+（如使用 Riru 版本）

## 基本概念

### Hook 技术
Hook 是一种拦截和修改函数调用的技术。LSPosed 使用 ART（Android Runtime）级别的 Hook，可以：

- 在方法执行前后插入自定义代码
- 修改方法的参数和返回值
- 完全替换方法的实现

### 模块（Module）
模块是实现特定功能的插件，可以：

- 修改系统应用的行为
- 增强第三方应用的功能
- 添加新的系统级功能
- 修复应用程序的 Bug

### 作用域（Scope）
LSPosed 允许您为每个模块指定作用域，即模块将在哪些应用中生效：

- **系统应用**: 修改 Android 系统组件
- **第三方应用**: 修改用户安装的应用
- **所有应用**: 全局生效

## 安全注意事项

⚠️ **重要提醒**:

1. **备份数据**: 在安装前务必备份重要数据
2. **了解风险**: Hook 系统可能导致系统不稳定
3. **可信来源**: 只安装来自可信开发者的模块
4. **逐个测试**: 安装多个模块时，建议逐个启用测试
5. **保持更新**: 使用最新版本的 LSPosed 和模块

## 工作原理

### 1. Zygote 进程注入
LSPosed 通过 Riru 或 Zygisk 在 Android 系统启动时注入到 Zygote 进程中。

### 2. 应用进程 Hook
当应用启动时，LSPosed 会在应用进程中设置 Hook 环境。

### 3. 方法拦截
当被 Hook 的方法被调用时，LSPosed 会先执行模块的代码，然后根据需要调用原始方法。

## 常见用例

### 1. 系统增强
- 状态栏定制
- 导航栏修改
- 锁屏界面优化
- 通知管理增强

### 2. 应用功能扩展
- 去除广告
- 增加新功能
- 界面美化
- 性能优化

### 3. 开发调试
- 运行时调试
- 日志增强
- 性能监控
- API 测试

## 下一步

准备好开始使用 LSPosed 了吗？

1. 📦 [安装指南](./installation-guide.md) - 学习如何安装 LSPosed
2. 🔍 [架构概述](./architecture-overview.md) - 了解技术细节
3. 🛠️ [模块开发指南](./module-development.md) - 开发您的第一个模块

## 学习资源

### 官方资源
- [Xposed Framework API](https://api.xposed.info/) - 官方 API 文档
- [LSPosed 模块仓库](https://github.com/Xposed-Modules-Repo) - 官方模块库

### 社区资源
- [示例模块](https://github.com/LSPosed/LSPosed/wiki/Example-modules) - 学习示例
- [开发者论坛](https://t.me/s/LSPosed) - 技术讨论

### 推荐模块
以下是一些优秀的模块，可以帮助您了解 LSPosed 的能力：

- **Greenify** - 应用休眠管理
- **AdBlock** - 广告拦截
- **CustomNavBar** - 导航栏定制
- **StatusBarLyric** - 状态栏歌词显示

## 故障排除

如果遇到问题，请查看：

1. [常见问题](./faq.md) - 常见问题解答
2. [调试指南](./debugging-guide.md) - 调试技巧
3. [GitHub Issues](https://github.com/LSPosed/LSPosed/issues/) - 报告 Bug

记住：LSPosed 只接受**英文标题**的 issue，如果您不懂英语，请使用[翻译工具](https://www.deepl.com/zh/translator)。
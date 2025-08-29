# LSPosed 安装指南

本指南将详细介绍如何在您的 Android 设备上安装 LSPosed 框架。

## 前期准备

### 系统要求
- **Android 版本**: 8.1 ~ 14
- **Magisk 版本**: v24 或更高
- **设备要求**: 已 Root 的 Android 设备

### 必要工具
- [Magisk](https://github.com/topjohnwu/Magisk/releases) - Root 管理器
- [Riru](https://github.com/RikkaApps/Riru/releases) - v26.1.7+（如选择 Riru 版本）

## 选择版本

LSPosed 提供两种版本：

### 1. Zygisk 版本（推荐）
- ✅ **优点**: 更稳定，性能更好，兼容性更强
- ✅ **优点**: 不需要额外安装 Riru
- ❌ **缺点**: 需要 Magisk v24+

### 2. Riru 版本
- ✅ **优点**: 兼容旧版本 Magisk
- ❌ **缺点**: 需要先安装 Riru 模块
- ❌ **缺点**: 性能略低

## 安装步骤

### 方法一：Zygisk 版本安装（推荐）

#### 第一步：启用 Zygisk
1. 打开 **Magisk 应用**
2. 点击右上角的 **设置**（齿轮图标）
3. 找到并启用 **Zygisk**
4. 重启设备

#### 第二步：下载 LSPosed
1. 访问 [LSPosed Releases](https://github.com/LSPosed/LSPosed/releases)
2. 下载最新的 **Zygisk 版本**（文件名包含 "zygisk"）
3. 或者下载开发版本从 [GitHub Actions](https://github.com/LSPosed/LSPosed/actions/workflows/core.yml?query=branch%3Amaster)

#### 第三步：安装模块
1. 打开 **Magisk 应用**
2. 点击 **模块** 标签
3. 点击 **从本地安装**
4. 选择下载的 LSPosed zip 文件
5. 等待安装完成
6. **重启设备**

### 方法二：Riru 版本安装

#### 第一步：安装 Riru
1. 下载 [Riru v26.1.7+](https://github.com/RikkaApps/Riru/releases)
2. 在 Magisk 中安装 Riru 模块
3. 重启设备

#### 第二步：安装 LSPosed
1. 下载 **Riru 版本** LSPosed（文件名包含 "riru"）
2. 在 Magisk 中安装 LSPosed 模块
3. **重启设备**

## 验证安装

### 检查安装状态
重启后，您应该能看到：

1. **通知栏** 中出现 LSPosed 通知
2. **Magisk 模块列表** 中显示 LSPosed 已启用
3. 点击通知可以打开 **LSPosed 管理器**

### LSPosed 管理器功能
- **模块管理**: 启用/禁用 Xposed 模块
- **作用域设置**: 为模块选择目标应用
- **日志查看**: 查看框架和模块日志
- **设置选项**: 调整框架行为

## 常见问题解决

### 问题1：安装后无法启动
**可能原因**：
- Magisk 版本过低
- 系统版本不兼容
- 其他 Magisk 模块冲突

**解决方案**：
1. 更新 Magisk 到最新版本
2. 临时禁用其他模块测试
3. 检查设备兼容性

### 问题2：Zygisk 无法启用
**可能原因**：
- Magisk 版本过低
- 系统限制

**解决方案**：
1. 更新 Magisk 到 v24+
2. 尝试使用 Riru 版本

### 问题3：模块无法加载
**可能原因**：
- 模块版本不兼容
- 作用域设置错误
- 系统版本不支持

**解决方案**：
1. 检查模块兼容性
2. 正确设置作用域
3. 查看日志获取详细信息

### 问题4：系统不稳定
**症状**：
- 应用崩溃频繁
- 系统重启
- 功能异常

**解决方案**：
1. 逐个禁用模块找出问题模块
2. 检查模块版本兼容性
3. 暂时禁用所有模块

## 卸载方法

如果需要完全卸载 LSPosed：

### 方法一：通过 Magisk
1. 打开 **Magisk 应用**
2. 进入 **模块** 标签
3. 找到 LSPosed 模块
4. 点击 **移除** 或 **卸载**
5. 重启设备

### 方法二：手动删除
```bash
# 通过 ADB Shell 或终端
rm -rf /data/adb/modules/riru_lsposed/
rm -rf /data/adb/modules/zygisk_lsposed/
```

## 安全建议

### 备份策略
- 🔄 **创建 Nandroid 备份**：安装前创建完整系统备份
- 💾 **备份重要数据**：联系人、照片、文档等
- 📱 **测试环境**：建议先在测试设备上试用

### 风险评估
- ⚠️ **系统稳定性**：可能影响系统稳定性
- 🔒 **安全风险**：恶意模块可能威胁系统安全
- 🏦 **银行应用**：可能导致银行类应用无法使用

### 最佳实践
- ✅ 只从可信来源下载模块
- ✅ 定期更新 LSPosed 和模块
- ✅ 监控系统性能和稳定性
- ✅ 保持最新的系统安全补丁

## 下一步

安装完成后，您可以：

1. 📚 阅读 [模块开发指南](./module-development.md)
2. 🔍 了解 [架构概述](./architecture-overview.md)
3. 🛠️ 探索 [API 参考](./api-reference.md)
4. ❓ 查看 [常见问题](./faq.md)

## 获取帮助

如果安装过程中遇到问题：

- 🐛 [GitHub Issues](https://github.com/LSPosed/LSPosed/issues/) - 报告 Bug
- 💬 [Telegram 频道](https://t.me/s/LSPosed) - 社区支持
- 📖 [常见问题](./faq.md) - 常见问题解答

**注意**：报告问题时请使用英文标题，并提供详细的设备信息和日志。
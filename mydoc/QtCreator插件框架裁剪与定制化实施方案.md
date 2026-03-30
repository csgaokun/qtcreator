# Qt Creator 4.13.3 插件框架裁剪与定制化实施方案

> **版本**：v1.0  
> **日期**：2026-03-30  
> **适用平台**：Windows 7/10/11、银河麒麟、中标麒麟、Ubuntu；架构 x86\_64/x86\_32/ARM64/ARM32  
> **Qt 版本**：5.12.12  
> **源码路径**：`qt-creator-opensource-src-4.13.3/`

---

## 目录

1. [项目目标与范围](#1-项目目标与范围)
2. [Qt Creator 插件系统架构分析](#2-qt-creator-插件系统架构分析)
3. [最小依赖集（需保留的核心文件）](#3-最小依赖集需保留的核心文件)
4. [独立应用骨架工程设计](#4-独立应用骨架工程设计)
5. [详细实施步骤（逐文件逐行说明）](#5-详细实施步骤逐文件逐行说明)
6. [自定义插件开发规范](#6-自定义插件开发规范)
7. [UI 定制化方案](#7-ui-定制化方案)
8. [团队分组开发工作流](#8-团队分组开发工作流)
9. [跨平台构建配置](#9-跨平台构建配置)
10. [风险与注意事项](#10-风险与注意事项)

---

## 1. 项目目标与范围

### 1.1 目标

| 编号 | 目标描述 |
|------|----------|
| G-01 | 从 Qt Creator 4.13.3 源码中裁剪出**插件系统核心库**，剔除 IDE 相关功能 |
| G-02 | 基于裁剪后的插件系统，创建**独立可执行应用骨架** |
| G-03 | 提供**标准插件开发模板**，支持团队分组独立开发插件 |
| G-04 | 支持**定制化 UI**：可替换主窗口、菜单栏、工具栏、侧边栏等 |
| G-05 | 全平台兼容：Windows 7/10/11、银河麒麟、中标麒麟、Ubuntu；x86_64/x86_32/ARM64/ARM32 |

### 1.2 最终产出物

```
my-app/                          ← 独立应用工程根目录
├── app/                         ← 主程序（裁剪自 src/app）
├── libs/
│   ├── extensionsystem/         ← 插件系统库（来自 Qt Creator）
│   ├── aggregation/             ← Aggregate 对象聚合库（来自 Qt Creator）
│   └── utils/                   ← 工具库（按需裁剪）
├── plugins/
│   ├── core/                    ← 核心插件（自研，替代 Qt Creator 的 coreplugin）
│   ├── groupA/                  ← A 组插件
│   └── groupB/                  ← B 组插件
└── mydoc/                       ← 项目文档
```

---

## 2. Qt Creator 插件系统架构分析

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         Application (main.cpp)                  │
│  - 创建 QApplication                                             │
│  - 创建 PluginManager                                            │
│  - 设置插件 IID / 插件路径                                        │
│  - 调用 PluginManager::loadPlugins()                             │
│  - 进入 app.exec() 事件循环                                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 拥有
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              ExtensionSystem::PluginManager (单例)               │
│  src/libs/extensionsystem/pluginmanager.h/.cpp                  │
│                                                                 │
│  - 扫描插件目录（.so/.dll 文件）                                  │
│  - 读取每个插件的 JSON 元数据                                     │
│  - 解析插件依赖关系                                               │
│  - 按拓扑排序加载插件                                             │
│  - 管理全局对象池 (addObject/removeObject/getObject<T>)          │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 管理 N 个
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              ExtensionSystem::PluginSpec                        │
│  src/libs/extensionsystem/pluginspec.h/.cpp                     │
│                                                                 │
│  每个插件对应一个 PluginSpec，包含：                              │
│  - 名称、版本、描述（来自 JSON 元数据）                           │
│  - 依赖列表 (PluginDependency)                                   │
│  - 加载状态机（Invalid→Read→Resolved→Loaded→Initialized→Running）│
│  - QPluginLoader 实例（负责实际 dlopen/LoadLibrary）             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 加载后持有
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              ExtensionSystem::IPlugin （抽象接口）               │
│  src/libs/extensionsystem/iplugin.h/.cpp                        │
│                                                                 │
│  插件必须继承此接口并实现：                                        │
│    bool initialize(args, &errorString)   ← 第一阶段初始化         │
│    void extensionsInitialized()          ← 第二阶段（所有插件初始化完） │
│    bool delayedInitialize()              ← 延迟初始化             │
│    ShutdownFlag aboutToShutdown()        ← 关闭前清理             │
│                                                                 │
│  + 每个插件须在 .cpp 中声明：                                     │
│    Q_PLUGIN_METADATA(IID "com.myapp.Plugin" FILE "MyPlugin.json")│
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 插件加载生命周期

```
          磁盘上的 .so/.dll 文件
                  │
         ┌────────▼────────┐
         │  PluginSpec::   │  state = Read
         │  read(fileName) │  ← 解析 JSON 元数据
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │ resolveDepen-   │  state = Resolved
         │  dencies()      │  ← 检查依赖是否满足
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │ loadLibrary()   │  state = Loaded
         │ QPluginLoader   │  ← dlopen / LoadLibrary
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │ initializePlugin│  state = Initialized
         │  IPlugin::      │  ← 调用 initialize()
         │  initialize()   │
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │ initializeExten-│  state = Running
         │  sions()        │  ← 调用 extensionsInitialized()
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │ delayedInitiali-│  （可选，Qt 事件循环后触发）
         │  ze()           │
         └─────────────────┘
```

### 2.3 核心库依赖关系

```
extensionsystem  ←→  aggregation
       │
       └── utils (算法、文件、日志等)
              │
              └── Qt5::Core, Qt5::Gui, Qt5::Widgets, Qt5::Network
```

### 2.4 插件 JSON 元数据格式

每个插件动态库内嵌一个 JSON 元数据块（通过 `Q_PLUGIN_METADATA` 宏），结构如下：

```json
{
    "Name"         : "MyPlugin",
    "Version"      : "1.0.0",
    "CompatVersion": "1.0.0",
    "Vendor"       : "MyCompany",
    "Copyright"    : "(C) 2026 MyCompany",
    "License"      : "Your License Here",   // 占位符，请替换为实际许可证文本；许可证合规要求见第 10.1 节
    "Description"  : "My first plugin",
    "Url"          : "https://mycompany.com",
    "Category"     : "GroupA",
    "Dependencies" : [
        { "Name" : "Core", "Version" : "1.0.0" }
    ]
}
```

---

## 3. 最小依赖集（需保留的核心文件）

### 3.1 必须保留的库文件清单

#### `src/libs/extensionsystem/`（全部保留，13 个文件）

| 文件 | 行数 | 说明 |
|------|------|------|
| `extensionsystem_global.h` | 37 | 导出宏定义 |
| `iplugin.h` / `iplugin.cpp` | 67 / 54 | 插件抽象基类 |
| `iplugin_p.h` | — | IPlugin 私有实现 |
| `pluginspec.h` / `pluginspec.cpp` | ~135 / **1127** | 插件元数据与状态机 |
| `pluginspec_p.h` | ~110 | PluginSpec 私有成员 |
| `pluginmanager.h` / `pluginmanager.cpp` | ~140 / **1732** | 插件管理器核心 |
| `pluginmanager_p.h` | ~130 | PluginManager 私有实现 |
| `optionsparser.h` / `optionsparser.cpp` | — | 命令行参数解析 |
| `invoker.h` / `invoker.cpp` | — | 跨插件方法调用 |

#### `src/libs/aggregation/`（全部保留，4 个文件）

| 文件 | 说明 |
|------|------|
| `aggregate.h` / `aggregate.cpp` | QObject 多接口聚合，被 extensionsystem 依赖 |
| `aggregation_global.h` | 导出宏 |

#### `src/libs/utils/`（**按需裁剪**，保留以下最小集）

| 文件 | 原因 |
|------|------|
| `algorithm.h` | `pluginmanager.cpp` 第 4 行 `#include <utils/algorithm.h>` |
| `benchmarker.h/.cpp` | `pluginmanager.cpp` 使用 `Benchmarker` |
| `executeondestruction.h` | `pluginmanager.cpp` 使用 |
| `fileutils.h/.cpp` | `pluginspec.cpp` 使用文件路径处理 |
| `hostosinfo.h/.cpp` | `pluginspec.cpp` / `pluginmanager.cpp` 使用平台判断 |
| `qtcassert.h` | `pluginspec.cpp` 使用 `QTC_ASSERT` |
| `stringutils.h/.cpp` | `pluginspec.cpp` 使用 |
| `mimetypes/mimedatabase.*` | `pluginmanager.cpp` 使用 |
| `synchronousprocess.h/.cpp` | `pluginmanager.cpp` 使用 |

> **说明**：如希望完全去掉 utils 依赖，需逐一替换上述调用为 Qt 标准库等价实现（见第 5 节）。

### 3.2 可完全删除的目录

以下目录与插件**加载机制**无关，可在新工程中删除：

```
src/plugins/android/        src/plugins/debugger/
src/plugins/baremetal/      src/plugins/git/
src/plugins/boot2qt/        src/plugins/clang*/
src/plugins/coreplugin/     (需用自研 core 替代)
src/libs/cplusplus/
src/libs/clangsupport/
src/libs/languageserverprotocol/
src/libs/qmleditorwidgets/
src/libs/modelinglib/
src/libs/ssh/
src/libs/tracing/
src/tools/                  (崩溃处理等可选)
```

---

## 4. 独立应用骨架工程设计

### 4.1 推荐目录结构

```
my-app/
├── my-app.pro                  ← qmake 顶层工程文件
├── app/
│   ├── app.pro
│   ├── main.cpp                ← 主程序入口（裁剪自 Qt Creator main.cpp）
│   └── app_version.h           ← 版本常量
├── libs/
│   ├── libs.pro
│   ├── aggregation/            ← 原样复制
│   ├── extensionsystem/        ← 原样复制
│   └── utils/                  ← 按需裁剪复制
├── plugins/
│   ├── plugins.pro
│   ├── core/                   ← 自研核心插件（提供主窗口、菜单等）
│   │   ├── core.pro
│   │   ├── Core.json.in
│   │   ├── coreplugin.h/.cpp
│   │   └── mainwindow.h/.cpp
│   └── sample/                 ← 示例插件模板
│       ├── sample.pro
│       ├── Sample.json.in
│       └── sampleplugin.h/.cpp
├── share/
│   └── translations/
└── mydoc/
```

### 4.2 顶层 .pro 文件

**文件**：`my-app/my-app.pro`（**新建**）

```qmake
include(my-app.pri)

TEMPLATE = subdirs
SUBDIRS = libs app plugins

app.depends = libs
plugins.depends = libs app
```

**文件**：`my-app/my-app.pri`（**新建**，定义全局变量）

```qmake
# 全局配置
MYAPP_VERSION     = 1.0.0
MYAPP_COMPAT_VER  = 1.0.0

# 输出目录
IDE_BUILD_TREE    = $$OUT_PWD/..
IDE_APP_PATH      = $$IDE_BUILD_TREE/bin
IDE_LIBRARY_PATH  = $$IDE_BUILD_TREE/lib/myapp
IDE_PLUGIN_PATH   = $$IDE_BUILD_TREE/lib/myapp/plugins

DEFINES += MYAPP_VERSION=\\\"$$MYAPP_VERSION\\\"
DEFINES += MYAPP_PLUGIN_IID=\\\"com.mycompany.myapp.Plugin\\\"
```

---

## 5. 详细实施步骤（逐文件逐行说明）

### 步骤一：复制并适配 extensionsystem 库

#### 5.1.1 复制文件

```bash
cp -r qt-creator-opensource-src-4.13.3/src/libs/extensionsystem my-app/libs/
cp -r qt-creator-opensource-src-4.13.3/src/libs/aggregation     my-app/libs/
```

#### 5.1.2 修改 `my-app/libs/extensionsystem/extensionsystem.pro`

**原文件**第 1-2 行：
```qmake
DEFINES += EXTENSIONSYSTEM_LIBRARY
include(../../qtcreatorlibrary.pri)
```

**修改为**（替换 include 路径，指向新工程的 pri 文件）：
```qmake
DEFINES += EXTENSIONSYSTEM_LIBRARY

TARGET   = ExtensionSystem
TEMPLATE = lib
CONFIG  += shared dll

QT += widgets

INCLUDEPATH += ..

DEFINES += IDE_TEST_DIR=\\\"$$PWD\\\"

# 保留其余 HEADERS / SOURCES / FORMS 不变
```

> 关键点：删除原 `include(../../qtcreatorlibrary.pri)` 行，手动展开其内容中与本工程相关的部分。  
> **受影响行号**：第 1-2 行。

#### 5.1.3 处理 utils 依赖（两种方案）

**方案 A（推荐）**：同步复制 utils 最小集，修改 INCLUDEPATH。

在 `extensionsystem.pro` 中追加：
```qmake
INCLUDEPATH += ../../libs   # 使 #include <utils/xxx.h> 可以找到
```

**方案 B（深度裁剪）**：在以下文件中将 utils 调用替换为 Qt 标准库等价实现：

| 原调用（utils） | Qt 替代 | 所在文件:行 |
|----------------|---------|------------|
| `Utils::sort(vec, ...)` | `std::sort(...)` | `pluginmanager.cpp` 多处 |
| `Utils::anyOf(...)` | `std::any_of(...)` | `pluginmanager.cpp` 多处 |
| `Utils::filtered(...)` | Lambda + `std::copy_if` | `pluginmanager.cpp` 多处 |
| `Utils::contains(...)` | `std::find_if` | `pluginmanager_p.h:100` |
| `QTC_ASSERT(cond, ...)` | `Q_ASSERT(cond)` | 全局替换 |
| `Utils::HostOsInfo::isWindowsHost()` | `#ifdef Q_OS_WIN` | 多处 |
| `Utils::FilePath` | `QString` / `QFileInfo` | `pluginspec.cpp` 多处 |
| `Utils::Benchmarker` | 删除或空实现 | `pluginmanager.cpp` 多处 |
| `Utils::ExecuteOnDestruction` | 析构函数 RAII 类 | `pluginmanager.cpp` 多处 |

### 步骤二：创建自研 app 入口

**文件**：`my-app/app/main.cpp`（**新建**，基于 Qt Creator `main.cpp` 裁剪）

以下列出关键裁剪点（参考原文件行号）：

| 原 main.cpp 位置 | 操作 | 说明 |
|----------------|------|------|
| 第 26 行 `#include "../tools/qtcreatorcrashhandler/crashhandlersetup.h"` | **删除** | 不需要崩溃处理 |
| 第 27 行 `#include <app/app_version.h>` | **修改** 为 `#include "app_version.h"` | 路径适配 |
| 第 32 行 `#include <qtsingleapplication.h>` | **保留或替换** | 可替换为 `QApplication` |
| 第 34-40 行 utils includes | **按需保留** | 依照步骤一选择的方案 |
| 第 70 行 `const char corePluginNameC[] = "Core"` | **修改** 为应用自定义的核心插件名 | 例如 `"AppCore"` |
| 第 90 行 `const char PLUGINPATH_OPTION[]` 等选项定义 | **保留** | 命令行参数支持 |
| 第 566 行 `PluginManager::setPluginIID(...)` | **修改** IID 字符串 | 改为 `"com.mycompany.myapp.Plugin"` |
| 第 606 行 `PluginManager::setPluginPaths(pluginPaths)` | **保留** | 插件路径设置 |
| 第 697 行 `PluginManager::loadPlugins()` | **保留** | 插件加载 |
| 第 711 行 aboutToQuit 连接 | **保留** | 正常关闭 |

**裁剪后的 main.cpp 骨架**（约 80 行，完整示例）：

```cpp
// my-app/app/main.cpp
#include "app_version.h"

#include <extensionsystem/iplugin.h>
#include <extensionsystem/pluginmanager.h>
#include <extensionsystem/pluginspec.h>

#include <QApplication>
#include <QDir>
#include <QSettings>

using namespace ExtensionSystem;

static const char PLUGIN_IID[]  = "com.mycompany.myapp.Plugin";
static const char CORE_PLUGIN[] = "AppCore";

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);
    app.setApplicationName("MyApp");
    app.setApplicationVersion(MYAPP_VERSION);
    app.setOrganizationName("MyCompany");

    // 1. 初始化 PluginManager
    PluginManager pluginManager;
    PluginManager::setPluginIID(QLatin1String(PLUGIN_IID));

    // 2. Settings（可选）
    QSettings settings(QSettings::IniFormat, QSettings::UserScope,
                       "MyCompany", "MyApp");
    PluginManager::setSettings(&settings);

    // 3. 设置插件搜索路径
    QStringList pluginPaths;
    pluginPaths << QDir::cleanPath(
        QCoreApplication::applicationDirPath() + "/../lib/myapp/plugins");
    PluginManager::setPluginPaths(pluginPaths);

    // 4. 验证核心插件存在
    const auto plugins = PluginManager::plugins();
    PluginSpec *corePlugin = nullptr;
    for (PluginSpec *spec : plugins) {
        if (spec->name() == QLatin1String(CORE_PLUGIN)) {
            corePlugin = spec;
            break;
        }
    }
    if (!corePlugin || !corePlugin->isEffectivelyEnabled()) {
        qCritical("Core plugin '%s' not found or disabled.", CORE_PLUGIN);
        return 1;
    }

    // 5. 加载所有插件
    PluginManager::loadPlugins();
    if (corePlugin->hasError()) {
        qCritical("Core plugin error: %s",
                  qPrintable(corePlugin->errorString()));
        return 1;
    }

    // 6. 正常关闭
    QObject::connect(&app, &QCoreApplication::aboutToQuit,
                     &pluginManager, &PluginManager::shutdown);

    return app.exec();
}
```

### 步骤三：创建自研 Core 插件

#### 5.3.1 插件元数据文件

**文件**：`my-app/plugins/core/AppCore.json.in`（**新建**）

```json
{
    "Name"         : "AppCore",
    "Version"      : "$$MYAPP_VERSION",
    "CompatVersion": "$$MYAPP_COMPAT_VER",
    "Vendor"       : "MyCompany",
    "Description"  : "Core plugin providing main window and UI framework",
    "Category"     : "Core",
    "Dependencies" : []
}
```

#### 5.3.2 插件头文件

**文件**：`my-app/plugins/core/coreplugin.h`（**新建**）

```cpp
#pragma once
#include <extensionsystem/iplugin.h>

namespace AppCore {
namespace Internal {

class MainWindow;

class CorePlugin : public ExtensionSystem::IPlugin
{
    Q_OBJECT
    Q_PLUGIN_METADATA(IID "com.mycompany.myapp.Plugin" FILE "AppCore.json")

public:
    CorePlugin();
    ~CorePlugin() override;

    bool initialize(const QStringList &arguments,
                    QString *errorString) override;
    void extensionsInitialized() override;
    ShutdownFlag aboutToShutdown() override;

private:
    MainWindow *m_mainWindow = nullptr;
};

} // namespace Internal
} // namespace AppCore
```

#### 5.3.3 插件实现文件

**文件**：`my-app/plugins/core/coreplugin.cpp`（**新建**）

```cpp
#include "coreplugin.h"
#include "mainwindow.h"

#include <extensionsystem/pluginmanager.h>

namespace AppCore {
namespace Internal {

CorePlugin::CorePlugin() = default;
CorePlugin::~CorePlugin() = default;

bool CorePlugin::initialize(const QStringList &arguments, QString *errorString)
{
    Q_UNUSED(arguments)
    Q_UNUSED(errorString)

    m_mainWindow = new MainWindow;
    return true;
}

void CorePlugin::extensionsInitialized()
{
    // 所有插件初始化完成后，显示主窗口
    m_mainWindow->show();
}

IPlugin::ShutdownFlag CorePlugin::aboutToShutdown()
{
    m_mainWindow->hide();
    return SynchronousShutdown;
}

} // namespace Internal
} // namespace AppCore
```

#### 5.3.4 qmake 工程文件

**文件**：`my-app/plugins/core/core.pro`（**新建**）

```qmake
QTC_PLUGIN_NAME = AppCore
QTC_LIB_DEPENDS += aggregation extensionsystem

include(../../my-app.pri)
include(../../libs/qtcreatorplugin.pri)

HEADERS += coreplugin.h mainwindow.h
SOURCES += coreplugin.cpp mainwindow.cpp
DISTFILES += AppCore.json.in
```

### 步骤四：创建示例业务插件模板

#### 5.4.1 插件模板结构

**文件**：`my-app/plugins/sample/SamplePlugin.json.in`（**新建**）

```json
{
    "Name"         : "SamplePlugin",
    "Version"      : "$$MYAPP_VERSION",
    "CompatVersion": "$$MYAPP_COMPAT_VER",
    "Vendor"       : "TeamA",
    "Description"  : "Sample feature plugin",
    "Category"     : "GroupA",
    "Dependencies" : [
        { "Name" : "AppCore", "Version" : "1.0.0" }
    ]
}
```

**文件**：`my-app/plugins/sample/sampleplugin.h`（**新建**）

```cpp
#pragma once
#include <extensionsystem/iplugin.h>

namespace SamplePlugin {
namespace Internal {

class SamplePluginImpl : public ExtensionSystem::IPlugin
{
    Q_OBJECT
    Q_PLUGIN_METADATA(IID "com.mycompany.myapp.Plugin" FILE "SamplePlugin.json")

public:
    bool initialize(const QStringList &arguments,
                    QString *errorString) override;
    void extensionsInitialized() override;
    ShutdownFlag aboutToShutdown() override;
};

} // namespace Internal
} // namespace SamplePlugin
```

---

## 6. 自定义插件开发规范

### 6.1 插件版本规则

Qt Creator 的插件版本采用 `major.minor.patch` 格式，兼容性检查逻辑：

```
兼容条件：compatVersion <= requestedVersion <= version
```

**建议**：所有内部插件统一使用相同 `Version` 和 `CompatVersion`，避免版本不匹配导致加载失败。

### 6.2 IID（插件身份标识）规则

**关键**：`Q_PLUGIN_METADATA` 中的 IID 字符串必须与 `PluginManager::setPluginIID()` 中设置的值**完全一致**，否则插件不会被识别加载。

```
主程序设置：  PluginManager::setPluginIID("com.mycompany.myapp.Plugin")
插件声明：    Q_PLUGIN_METADATA(IID "com.mycompany.myapp.Plugin" FILE "MyPlugin.json")
                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 必须完全相同
```

### 6.3 插件生命周期回调

| 回调函数 | 触发时机 | 典型用途 |
|---------|---------|---------|
| `initialize(args, err)` | 插件库加载后立即调用 | 创建对象、注册服务、读取配置 |
| `extensionsInitialized()` | **所有**插件的 `initialize()` 均完成后 | 跨插件查找服务（用 `PluginManager::getObject<T>()`) |
| `delayedInitialize()` | 进入事件循环后，按 20ms 间隔依次调用 | 耗时初始化，不阻塞启动 |
| `aboutToShutdown()` | 应用退出前 | 保存状态，若有异步操作返回 `AsynchronousShutdown` |

### 6.4 跨插件服务发现（对象池）

```cpp
// 插件 A：注册服务
class IMyService : public QObject { ... };
class MyServiceImpl : public IMyService { ... };

// 在 initialize() 中：
ExtensionSystem::PluginManager::addObject(new MyServiceImpl);

// 插件 B：获取服务（在 extensionsInitialized() 中，保证 A 已初始化）
auto *svc = ExtensionSystem::PluginManager::getObject<IMyService>();
if (svc) {
    svc->doSomething();
}
```

### 6.5 插件可选依赖

```json
"Dependencies" : [
    { "Name": "Core",        "Version": "1.0.0"                      },
    { "Name": "FeatureB",    "Version": "1.0.0", "Type": "optional"  }
]
```

- `"Type": "optional"` 表示该依赖不存在时插件仍可加载
- `"Type": "test"` 表示该依赖仅在测试模式下需要

---

## 7. UI 定制化方案

### 7.1 主窗口架构选择

Qt Creator 使用 `FancyTabWidget` 实现侧边栏模式切换。对于自定义应用，有以下三种方案：

| 方案 | 说明 | 适用场景 |
|------|------|---------|
| **方案 A** | 沿用 `FancyTabWidget`（复制 `src/plugins/coreplugin/fancytabwidget.*`） | 需要快速原型，UI 风格与 Qt Creator 相近 |
| **方案 B** | 用 `QDockWidget` 组合实现侧边栏 | 标准 Qt 方案，灵活可定制 |
| **方案 C** | 完全自绘，使用 `QStackedWidget` + 自定义导航栏 | 追求独特 UI 风格 |

### 7.2 方案 B 主窗口示例（推荐）

**文件**：`my-app/plugins/core/mainwindow.h`（**新建**）

```cpp
#pragma once
#include <QMainWindow>

class QStackedWidget;
class QToolBar;

namespace AppCore {
namespace Internal {

class MainWindow : public QMainWindow
{
    Q_OBJECT
public:
    explicit MainWindow(QWidget *parent = nullptr);
    ~MainWindow() override;

    // 供其他插件调用，向主窗口注册页面
    void addPage(const QString &id, const QString &label,
                 const QIcon &icon, QWidget *page);

protected:
    void closeEvent(QCloseEvent *event) override;

private:
    QToolBar      *m_sideBar    = nullptr;
    QStackedWidget *m_pageStack = nullptr;
};

} // namespace Internal
} // namespace AppCore
```

**文件**：`my-app/plugins/core/mainwindow.cpp`（**新建**，关键代码段）

```cpp
#include "mainwindow.h"
#include <QStackedWidget>
#include <QToolBar>
#include <QAction>
#include <QActionGroup>
#include <QCloseEvent>

namespace AppCore {
namespace Internal {

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
{
    setWindowTitle("MyApp");
    resize(1024, 768);

    // 侧边栏（以 ToolBar 模拟图标导航）
    m_sideBar = addToolBar("Navigation");
    m_sideBar->setMovable(false);
    m_sideBar->setToolButtonStyle(Qt::ToolButtonTextUnderIcon);

    // 中央页面堆栈
    m_pageStack = new QStackedWidget(this);
    setCentralWidget(m_pageStack);
}

MainWindow::~MainWindow() = default;

void MainWindow::addPage(const QString &id, const QString &label,
                         const QIcon &icon, QWidget *page)
{
    int index = m_pageStack->addWidget(page);
    auto *action = m_sideBar->addAction(icon, label);
    action->setCheckable(true);
    action->setData(index);
    connect(action, &QAction::toggled, this, [this, index](bool checked) {
        if (checked)
            m_pageStack->setCurrentIndex(index);
    });
}

void MainWindow::closeEvent(QCloseEvent *event)
{
    // 让 Qt 事件系统自然触发 aboutToQuit()，避免重复发射
    // 直接接受关闭事件，QApplication 会在最后一个窗口关闭后发射 aboutToQuit()
    event->accept();
}

} // namespace Internal
} // namespace AppCore
```

### 7.3 其他插件向主窗口添加 UI 的方式

```cpp
// 在业务插件的 extensionsInitialized() 中：
#include <extensionsystem/pluginmanager.h>

void MyPlugin::extensionsInitialized()
{
    // 方式1：通过对象池获取 MainWindow 服务接口
    auto *mainWin = ExtensionSystem::PluginManager::getObject<AppCore::IMainWindowService>();
    if (mainWin) {
        mainWin->addPage("myplugin", tr("My Feature"),
                         QIcon(":/icons/myplugin.png"),
                         new MyFeatureWidget);
    }
}
```

---

## 8. 团队分组开发工作流

### 8.1 推荐目录约定

```
my-app/plugins/
├── core/        ← 核心团队维护（主窗口、服务接口定义）
├── groupA/      ← A 组开发（N 个插件，每个插件一个子目录）
│   ├── featureA1/
│   └── featureA2/
├── groupB/      ← B 组开发
│   └── featureB1/
└── shared/      ← 共享插件（所有组都依赖的公共服务）
```

### 8.2 插件接口契约（Interface Header）

核心团队定义**纯虚接口头文件**，业务团队实现并注册：

**文件**：`my-app/plugins/core/imainwindowservice.h`（**新建**，核心团队维护）

```cpp
#pragma once
#include <QObject>
#include <QIcon>

class QWidget;

namespace AppCore {

class IMainWindowService
{
public:
    virtual ~IMainWindowService() = default;

    virtual void addPage(const QString &id, const QString &label,
                         const QIcon &icon, QWidget *page) = 0;
    virtual void showStatusMessage(const QString &message, int timeout = 0) = 0;
};

} // namespace AppCore

#define AppCore_IMainWindowService_iid "com.mycompany.myapp.IMainWindowService"
Q_DECLARE_INTERFACE(AppCore::IMainWindowService, AppCore_IMainWindowService_iid)
```

### 8.3 版本管理约定

| 约定项 | 规则 |
|--------|------|
| 核心插件 (`AppCore`) | 主线 `main` 分支，版本严格管控 |
| 组 A 插件 | `feature/groupA/*` 分支开发 |
| 组 B 插件 | `feature/groupB/*` 分支开发 |
| 插件 JSON 版本 | 与 `MYAPP_VERSION` 同步（统一由 .pri 注入） |
| 接口变更 | 修改 `imainwindowservice.h` 需评审，兼容旧接口 |

### 8.4 插件加载顺序控制

通过 JSON 的 `"Dependencies"` 字段控制加载顺序（`PluginManager` 使用拓扑排序）：

```
AppCore (无依赖，最先加载)
   └─ SharedService (依赖 AppCore)
        ├─ GroupA_Feature1 (依赖 AppCore + SharedService)
        ├─ GroupA_Feature2 (依赖 AppCore + SharedService)
        └─ GroupB_Feature1 (依赖 AppCore)
```

---

## 9. 跨平台构建配置

### 9.1 qmake 平台适配

在 `my-app.pri` 中追加：

```qmake
# ── 平台输出路径 ────────────────────────────────────────────────
win32 {
    IDE_APP_PATH     = $$IDE_BUILD_TREE/bin
    IDE_LIBRARY_PATH = $$IDE_BUILD_TREE/bin
    IDE_PLUGIN_PATH  = $$IDE_BUILD_TREE/bin/plugins
    DEFINES += APP_LIBRARY_BASENAME=\\\"bin\\\"
} else {
    IDE_APP_PATH     = $$IDE_BUILD_TREE/bin
    IDE_LIBRARY_PATH = $$IDE_BUILD_TREE/lib/myapp
    IDE_PLUGIN_PATH  = $$IDE_BUILD_TREE/lib/myapp/plugins
    DEFINES += APP_LIBRARY_BASENAME=\\\"lib\\\"
}

# ── ARM 架构适配 ─────────────────────────────────────────────────
# Qt 5.12.12 在 ARM 上需要手动指定 -mfpu 和 -mfloat-abi
linux-g++-arm*|linux-arm*{
    QMAKE_CXXFLAGS += -mfpu=neon -mfloat-abi=hard
}

# ── 麒麟 (Kylin) / Ubuntu 适配 ───────────────────────────────────
linux {
    # 动态库 RPATH（使插件能找到 extensionsystem 动态库）
    QMAKE_LFLAGS += -Wl,-rpath,\$$ORIGIN/../lib/myapp
}
```

### 9.2 各平台库文件后缀

| 平台 | 动态库后缀 | 示例 |
|------|-----------|------|
| Windows | `.dll` | `ExtensionSystem.dll` |
| Linux / 银河麒麟 / 中标麒麟 / Ubuntu | `.so` | `libExtensionSystem.so` |
| macOS | `.dylib` | `libExtensionSystem.dylib` |

Qt Creator 的 `pluginspec.cpp` 在 `loadLibrary()` 中通过 `QPluginLoader` 自动处理后缀，**无需手动区分**。

### 9.3 Windows 7 兼容性

Qt 5.12.12 是最后一个官方支持 Windows 7 的 Qt 5.x 版本。注意：

- 必须使用 MSVC 2017 或 MinGW 7.3 编译
- 不要开启 C++17 的某些特性（如结构化绑定）在 MinGW 7.3 上可能有问题
- 在 `my-app.pri` 中添加 `QMAKE_CXXFLAGS += -std=c++14` 或使用默认的 C++11

### 9.4 构建命令参考

```bash
# Linux x86_64
mkdir build && cd build
qmake ../my-app/my-app.pro CONFIG+=release
make -j$(nproc)

# Windows (MSVC 2017 x64)
mkdir build && cd build
qmake ..\my-app\my-app.pro CONFIG+=release -spec win32-msvc
jom /J8

# 银河麒麟 ARM64
mkdir build && cd build
qmake ../my-app/my-app.pro CONFIG+=release QMAKE_CC=aarch64-linux-gnu-gcc QMAKE_CXX=aarch64-linux-gnu-g++
make -j4
```

---

## 10. 风险与注意事项

### 10.1 许可证

Qt Creator 源码使用 **GPL v3** 许可（含例外条款）。

- `extensionsystem` / `aggregation` 库使用 **LGPLv3 + 例外**，可在商业应用中使用。
- `utils` 库中的部分代码使用 **GPL v3**，使用时需注意。
- **建议**：如需商业闭源分发，仅使用 `extensionsystem` 和 `aggregation` 两个库，不引入 `utils` 的 GPL 部分。

### 10.2 与 Qt Creator 原版插件不兼容

裁剪后的框架更改了 `PLUGIN_IID`，因此无法加载 Qt Creator 原版插件（这是预期行为）。

### 10.3 版本维护

`pluginmanager.cpp` 中版本比较使用 `versionString()` 函数，格式严格为 `x.y.z`，不支持 `x.y.z-beta` 等后缀。

### 10.4 线程安全

- `PluginManager::addObject/removeObject` 使用了 `QReadWriteLock`，可在子线程调用。
- `PluginManager::loadPlugins()` 必须在**主线程**调用。
- 插件的 `initialize()` / `extensionsInitialized()` 均在**主线程**调用。

### 10.5 插件卸载限制

Qt Creator 的插件系统**不支持运行时热卸载**单个插件（整体关闭时才卸载）。如需热插拔功能，需在 `IPlugin` 接口上额外扩展。

---

## 附录 A：核心文件修改清单汇总

| 序号 | 文件路径 | 操作 | 修改描述 | 关键行号 |
|------|---------|------|---------|---------|
| 1 | `libs/extensionsystem/extensionsystem.pro` | **修改** | 删除 `include(../../qtcreatorlibrary.pri)`，手动展开 | 第 1-2 行 |
| 2 | `libs/aggregation/aggregation.pro` | **修改** | 同上，删除 include 语句 | 第 1 行 |
| 3 | `app/main.cpp` | **新建** | 裁剪自 Qt Creator main.cpp，约 80 行 | — |
| 4 | `app/app_version.h` | **新建** | 版本常量定义 | — |
| 5 | `my-app.pri` | **新建** | 全局路径与编译变量 | — |
| 6 | `my-app.pro` | **新建** | 顶层 subdirs 工程 | — |
| 7 | `plugins/core/coreplugin.h` | **新建** | Core 插件类声明 | — |
| 8 | `plugins/core/coreplugin.cpp` | **新建** | Core 插件实现 | — |
| 9 | `plugins/core/mainwindow.h` | **新建** | 主窗口类声明 | — |
| 10 | `plugins/core/mainwindow.cpp` | **新建** | 主窗口实现 | — |
| 11 | `plugins/core/AppCore.json.in` | **新建** | Core 插件元数据模板 | — |
| 12 | `plugins/core/imainwindowservice.h` | **新建** | 主窗口服务接口（供其他插件使用） | — |
| 13 | `plugins/sample/sampleplugin.h` | **新建** | 示例插件类声明 | — |
| 14 | `plugins/sample/sampleplugin.cpp` | **新建** | 示例插件实现 | — |
| 15 | `plugins/sample/SamplePlugin.json.in` | **新建** | 示例插件元数据模板 | — |

## 附录 B：extensionsystem 依赖的 utils 函数完整替换表

| utils 调用 | 标准 Qt/STL 替换 | 所在文件 |
|-----------|----------------|---------|
| `Utils::sort(container, pred)` | `std::sort(c.begin(), c.end(), pred)` | pluginmanager.cpp |
| `Utils::anyOf(container, pred)` | `std::any_of(c.begin(), c.end(), pred)` | pluginmanager.cpp |
| `Utils::allOf(container, pred)` | `std::all_of(c.begin(), c.end(), pred)` | pluginmanager.cpp |
| `Utils::filtered(container, pred)` | 手写循环或 `std::copy_if` | pluginmanager.cpp |
| `Utils::contains(container, elem)` | `container.contains(elem)` (QVector) 或 `std::find` | pluginmanager_p.h |
| `QTC_ASSERT(cond, action)` | `if (!cond) { action; }` 或 `Q_ASSERT` | 多处 |
| `Utils::FilePath::fromString(s)` | `QString s` 直接使用 | pluginspec.cpp |
| `Utils::HostOsInfo::isWindowsHost()` | `#ifdef Q_OS_WIN` | pluginspec.cpp/pluginmanager.cpp |
| `Utils::HostOsInfo::withExecutableSuffix(s)` | `s + (QSysInfo::productType()=="windows"?".exe":"")` | pluginspec.cpp |
| `Utils::ExecuteOnDestruction(fn)` | RAII 类或析构函数 | pluginmanager.cpp |
| `Utils::Benchmarker(...)` | 删除或空实现 | pluginmanager.cpp |

---

*报告结束。如有疑问，请参阅原始代码：`qt-creator-opensource-src-4.13.3/src/libs/extensionsystem/`*
# FreeCAD 架构文档

## 整体架构概览

```
                    ┌─────────────────────────────┐
                    │       用户界面 (Qt)          │
                    │   MainWindow / MDIView       │
                    │   ToolBars / Menus / Dock     │
                    └──────────┬──────────────────┘
                               │
                    ┌──────────▼──────────────────┐
                    │     Gui 层 (FreeCADGui)      │
                    │  ┌─────────────────────┐    │
                    │  │ Gui::Application    │────│── Gui::Document
                    │  │ CommandManager      │    │     │
                    │  │ WorkbenchManager    │    │  ViewProvider
                    │  │ Selection           │    │  (Coin3D SceneGraph)
                    │  │ MacroManager        │    │
                    │  └─────────┬───────────┘    │
                    └────────────┼────────────────┘
                                 │ fastsignals
                    ┌────────────▼────────────────┐
                    │     App 层 (FreeCADApp)      │
                    │  ┌─────────────────────┐    │
                    │  │ App::Application    │────│── App::Document
                    │  │ (单例, 无GUI依赖)   │    │     │
                    │  │ 事务管理            │    │  DocumentObject
                    │  │ 模块注册            │    │     │
                    │  │ 参数管理            │    │  Property
                    │  └─────────┬───────────┘    │
                    └────────────┼────────────────┘
                                 │
                    ┌────────────▼────────────────┐
                    │    Base 层 (FreeCADBase)     │
                    │  Type系统 / Observer /       │
                    │  Interpreter / Persistence / │
                    │  数学库 / 单位系统 /         │
                    │  Console / Parameter         │
                    └─────────────────────────────┘
                                 │
                    ┌────────────▼────────────────┐
                    │    Mod 层 (插件/工作台)      │
                    │  Part / Mesh / Fem / ...     │
                    │  (C++模块 + Python模块)      │
                    └─────────────────────────────┘
```

**核心设计哲学**：

1. **App/Gui 严格分离** — App 层完全不依赖 GUI，可独立运行（命令行模式）
2. **Property 驱动** — 所有数据通过 Property 系统，自动支持序列化、通知、表达式
3. **插件架构** — 通过 Mod 目录和 Python 模块实现可扩展性
4. **信号解耦** — App 和 Gui 通过 fastsignals 松耦合
5. **类型系统** — 自建 RTTI 支持运行时类型查询和动态实例化
6. **命令模式** — 所有用户操作封装为 Command，支持宏录制和脚本化

---

## 源码目录结构

```
src/
├── Base/          基础库：类型系统、观察者模式、控制台、参数系统、
│                  Python解释器封装、数学工具、I/O、单位系统
├── App/           应用核心层（无GUI依赖）：Application单例、Document、
│                  DocumentObject、Property系统、表达式引擎、
│                  事务/撤销重做、Extension机制、Link系统
├── Gui/           图形界面层：Gui::Application、Gui::Document、
│                  ViewProvider体系、Command/Action框架、
│                  Workbench机制、MainWindow、3D视图(Coin3D)、选择系统
├── Mod/           工作台模块（插件）：Part、Mesh、Fem、Draft、
│                  Sketcher、TechDraw 等所有功能模块
├── Main/          入口点：MainGui.cpp（GUI模式）和 MainCmd.cpp（命令行模式）
├── 3rdParty/      第三方库
├── Build/         构建生成的版本信息
├── Ext/           扩展
├── Doc/           文档
└── Tools/         开发工具
```

**构建顺序**（`src/CMakeLists.txt`）：

```cmake
add_subdirectory(Build)
add_subdirectory(3rdParty)
add_subdirectory(Base)     # FreeCADBase 库
add_subdirectory(App)      # FreeCADApp 库
add_subdirectory(Main)     # 可执行文件
add_subdirectory(Mod)      # 所有工作台模块
add_subdirectory(Ext)
if(BUILD_GUI)
    add_subdirectory(Gui)  # FreeCADGui 库（可选）
endif()
```

**库的依赖关系**：

```
FreeCADBase  ← (独立基础库)
    ↑
FreeCADApp   ← (依赖 Base)
    ↑
FreeCADGui   ← (依赖 App + Base + Qt + Coin3D)
    ↑
Mod/*        ← (依赖 App，可选依赖 Gui)
```

---

## 程序入口

### GUI 模式 — `src/Main/MainGui.cpp`

FreeCAD 以图形界面模式启动时的 `main()` 函数：

1. 设置区域设置（`LC_NUMERIC=C` 确保数字格式一致）
2. 配置应用信息（`ExeName=FreeCAD`）
3. 设置 `RunMode=Gui`
4. **调用 `App::Application::init(argc, argv)`** — 初始化核心应用
5. **调用 `Gui::Application::runApplication()`** — 进入 Qt 事件循环

### 命令行模式 — `src/Main/MainCmd.cpp`

FreeCAD 以命令行/控制台模式启动时的 `main()` 函数：

1. 设置 `RunMode=Exit`（命令行模式，执行完即退出）
2. 设置 `LoggingConsole=1`
3. **调用 `App::Application::init(argc, argv)`**
4. **调用 `App::Application::runApplication()`** — 纯命令行 Python 解释器

**两者区别**：MainGui 使用 `Gui::Application::runApplication()`（Qt 事件循环 + 3D 视图），MainCmd 使用 `App::Application::runApplication()`（纯命令行 Python 解释器）。

---

## Base 层 — 基础库

Base 层是 FreeCAD 的基石，不依赖任何其他 FreeCAD 模块，提供所有上层共用的基础设施。

### 关键组件

| 组件 | 文件 | 职责 |
|------|------|------|
| **Type 系统** | `Type.h/cpp` | 运行时类型识别（RTTI），支持类层次查询、动态实例化 |
| **Observer** | `Observer.h` | 经典观察者模式的模板实现 |
| **Console** | `Console.h/cpp` | 全局控制台/日志系统 |
| **Parameter** | `Parameter.h/cpp` | 参数系统（ParameterGrp），持久化的键值配置 |
| **Interpreter** | `Interpreter.h/cpp` | Python 解释器封装，管理 GIL、模块加载、脚本执行 |
| **Persistence** | `Persistence.h/cpp` | 持久化基类（Save/Restore 到 XML/二进制） |
| **Reader/Writer** | `Reader.h / Writer.h` | 文档 I/O 流 |
| **Factory** | `Factory.h` | 工厂模式基类 |
| **BaseClass** | `BaseClass.h` | 所有可注册类型的基类，包含 `TYPESYSTEM_HEADER` 宏 |
| **Exception** | `Exception.h` | 异常体系 |
| **数学类** | `Vector3D / Matrix / Rotation / Placement / BoundBox` | 几何/数学基础类 |
| **单位系统** | `Quantity / Unit / UnitsApi` | 单位和量纲系统 |
| **Handle** | `Handle.h` | 引用计数智能指针 |
| **FileInfo/Stream** | `FileInfo.h / Stream.h` | 文件和流操作 |

### Type 系统

FreeCAD 自建了一套轻量级运行时类型识别系统，仅使用 16 位类型 ID：

```cpp
// 声明（在 .h 中）
TYPESYSTEM_HEADER();
TYPESYSTEM_HEADER_WITH_OVERRIDE();  // 带 override

// 实现（在 .cpp 中）
TYPESYSTEM_SOURCE(App::Document, PropertyContainer);
```

支持：
- 类继承关系查询（`isDerivedFrom()`）
- 按名称创建实例（`createInstanceByName()`）
- 运行时类型注册（`init()`）

---

## App 层 — 应用核心

App 层是 FreeCAD 的数据模型层，严格不依赖 GUI，可通过命令行独立运行。

### App::Application — 核心应用单例

```cpp
class Application {
    // 单例访问
    static Application& GetApplication();

    // 文档管理
    Document* newDocument(const char* proposedName, ...);
    bool closeDocument(const char* name);
    Document* getDocument(const char* name);

    // 信号系统（fastsignals）
    fastsignals::signal<void(const App::Document&)> signalNewDocument;
    fastsignals::signal<void(const App::Document&)> signalDeleteDocument;
    // ... 更多信号

    // 事务管理 / 参数管理 / 模块导入导出注册
    // 配置字典（static Config()）
};
```

**初始化流程**（`Application::init()`）：

1. `initTypes()` — 注册所有 Base 和 App 层的类型到 Type 系统
2. `initConfig()` — 解析命令行参数和配置
3. `initApplication()` — 初始化 Python 解释器、创建 FreeCAD Python 模块
4. `initExceptions()` — 注册异常类型

### App::Document — 文档

```cpp
class Document : public PropertyContainer {
    // 文档属性
    PropertyString Label;
    PropertyString FileName;
    PropertyUUID   Uid;
    PropertyLink   Tip;
    PropertyBool   ShowHidden;

    // 信号
    fastsignals::signal<void(const App::DocumentObject&)> signalNewObject;
    fastsignals::signal<void(const App::DocumentObject&)> signalDeletedObject;

    // 对象管理
    DocumentObject* addObject(const char* sType, ...);
    void removeObject(DocumentObject*);
    void recompute();

    // 事务
    int openTransaction(const char* name);
    void commitTransaction();
    void abortTransaction();
};
```

### App::DocumentObject — 文档对象

所有 Feature 的父类，持有 Property 集合，支持 recompute 机制：

- 属性变更 → `touch()` → 标记为需 recompute → Document 执行 recompute → 重新计算 Feature 输出
- 支持表达式绑定（PropertyExpressionEngine）
- 支持 Python 定义（FeaturePython）

### Property 系统

Property 系统是 FreeCAD 数据模型的核心，所有数据通过 Property 子类存储：

```cpp
class Property : public Base::Persistence {
    TYPESYSTEM_HEADER_WITH_OVERRIDE();

    enum Status {
        Touched, Immutable, ReadOnly, Hidden, Transient,
        MaterialEdit, NoMaterialListEdit, Output, LockDynamic,
        NoModify, PartialTrigger, NoRecompute, Single, ...
    };

    virtual void hasSetValue();       // 值改变时调用
    virtual void aboutToSetValue();
    void touch();                     // 标记为已修改，触发 recompute

    PropertyContainer* getContainer() const;

    // 持久化
    void Save(Base::Writer&) override;
    void Restore(Base::Reader&) override;
};
```

**Property 类型体系**：

| 类别 | 类型 |
|------|------|
| 基础类型 | PropertyBool, PropertyFloat, PropertyInteger, PropertyString, PropertyUUID |
| 列表类型 | PropertyBoolList, PropertyFloatList, PropertyIntegerList, PropertyStringList |
| 约束类型 | PropertyFloatConstraint, PropertyIntegerConstraint |
| 几何类型 | PropertyVector, PropertyMatrix, PropertyPlacement, PropertyRotation |
| 链接类型 | PropertyLink, PropertyLinkSub, PropertyLinkList, PropertyXLink, ... |
| 颜色/材质 | PropertyColor, PropertyMaterial |
| 单位类型 | PropertyLength, PropertyAngle, PropertyArea, PropertyVolume 等 40+ 种 |
| 表达式 | PropertyExpressionEngine |
| Python 对象 | PropertyPythonObject |
| 文件 | PropertyFile, PropertyFileIncluded |

**PropertyContainer** 管理属性集合：

```cpp
class PropertyContainer {
    PropertyData propertyData;     // 静态属性描述
    DynamicProperty dynamicProps;  // 动态属性

    Property* addProperty(const char* type, const char* name, ...);
    void removeProperty(const char* name);
};
```

---

## Gui 层 — 图形界面

Gui 层依赖 App 和 Base，负责可视化、用户交互、3D 渲染。

### App/Gui 分离的核心对应关系

| App 层 | Gui 层 |
|--------|--------|
| `App::Application`（单例） | `Gui::Application`（单例，持有 App 引用） |
| `App::Document` | `Gui::Document`（包装 `App::Document*` 指针） |
| `App::DocumentObject` | `Gui::ViewProvider`（1:1 映射） |
| `App::Property` | 属性编辑器（PropertyEditor） |

### 信号连接方式

Gui 层通过 **fastsignals** 连接 App 层信号（非 Qt 信号槽）：

```cpp
// Gui::Application 构造函数中
App::GetApplication().signalNewDocument.connect(
    std::bind(&Gui::Application::slotNewDocument, this, ...));

// Gui::Document 构造函数中
pcDocument->signalNewObject.connect(
    std::bind(&Gui::Document::slotNewObject, this, ...));
```

**优势**：App 层不依赖 Gui 层，但 Gui 层可以响应 App 层事件，实现松耦合。

### Gui::Application — GUI 应用单例

```cpp
class Application {
    static Application* Instance;

    // 文件操作
    void open(const char* FileName, const char* Module);
    void importFrom(...);
    void exportTo(...);

    // 视图管理
    void attachView(Gui::BaseView* pcView);
    Gui::MDIView* activeView();

    // 工作台管理
    bool activateWorkbench(const char* name);

    // 命令管理
    CommandManager& commandManager();
    MacroManager* macroManager();

    // ViewProvider 映射
    ViewProvider* getViewProvider(const App::DocumentObject*) const;
};
```

内部维护 `ViewProviderMap`：

```cpp
std::unordered_map<const App::DocumentObject*, ViewProvider*> map;
```

当 App 层创建新 DocumentObject 时，信号通知 Gui 层创建对应的 ViewProvider。

### Gui::Document — GUI 文档

```cpp
class Document : public Base::Persistence {
    Document(App::Document* pcDocument, Application* app);  // 持有 App::Document 指针

    // ViewProvider 管理
    ViewProvider* getViewProvider(const App::DocumentObject*) const;
    void addViewProvider(ViewProviderDocumentObject*);

    // 视图管理
    MDIView* createView(const Base::Type& typeId);
    MDIView* getActiveView();

    // 编辑模式
    bool setEdit(ViewProvider* p, int ModNum);
    void resetEdit();

    // 撤销/重做
    int openCommand(const char* sName);
    void commitCommand();
    void undo(int iSteps);

    // 信号（连接 App::Document 的信号到 slot）
    void slotNewObject(const App::DocumentObject&);
    void slotDeletedObject(const App::DocumentObject&);
    void slotChangedObject(const App::DocumentObject&, const App::Property&);
};
```

### ViewProvider — 视图提供者

ViewProvider 是连接 App 层数据到 3D 可视化的桥梁，基于 Coin3D/OpenInventor 场景图：

```cpp
class ViewProvider : public App::TransactionalObject {
    SoSeparator* pcRoot;       // 根节点
    SoSwitch*    pcModeSwitch;  // 显示模式切换
    SoTransform* pcTransform;  // 变换
    SoSeparator* pcAnnotation; // 注解

    SoSeparator* getRoot() const;
    SoSwitch*    getModeSwitch() const;
    SoTransform* getTransformNode() const;
};
```

**ViewProvider 继承层次**：

```
ViewProvider
  → ViewProviderDocumentObject
    → ViewProviderGeometryObject
      → ViewProviderPart（Part 模块）
      → ViewProviderMesh（Mesh 模块）
      → ...
    → ViewProviderDatum
    → ViewProviderAnnotation
    → ViewProviderLink
    → ViewProviderFeaturePython（Python 自定义）
  → ViewProviderExtern
```

---

## 工作台（Workbench）机制

工作台是 FreeCAD 的"功能包"概念——每个工作台定义一组菜单、工具栏和命令。

### 工作台列表（src/Mod/）

| 工作台 | 功能 |
|--------|------|
| **Part** | B-rep 实体建模（基于 OpenCASCADE） |
| **PartDesign** | 参数化零件设计 |
| **Sketcher** | 2D 约束草图 |
| **Fem** | 有限元分析 |
| **Mesh / MeshPart** | 网格建模 |
| **Draft** | 2D 绘图/草图 |
| **BIM** | 建筑信息模型 |
| **CAM** | 计算机辅助制造 |
| **TechDraw** | 技术制图 |
| **Assembly** | 装配 |
| **Spreadsheet** | 电子表格 |
| **Robot** | 机器人仿真 |
| **ReverseEngineering** | 逆向工程 |
| **Points** | 点云处理 |
| **OpenSCAD** | OpenSCAD 互操作 |
| **Surface** | 曲面建模 |
| **Material** | 材料管理 |
| **Measure** | 测量工具 |
| **AddonManager** | 插件管理器 |
| **Start** | 启动页面 |
| ... | ... |

### 工作台注册流程

每个 Mod 下的模块在 Python 中注册工作台：

```python
class PartWorkbench(Workbench):
    def Initialize(self):
        self.appendToolbar("Part", [...])
        self.appendMenu("Part", [...])

FreeCADGui.addWorkbench(PartWorkbench)
```

`addWorkbench()` 将工作台描述对象存入 `Gui::Application::_pcWorkbenchDictionary`（Python 字典）。

### 工作台激活流程

```
1. Gui::Application::activateWorkbench(name)
2. 从 _pcWorkbenchDictionary 获取 Python 工作台对象
3. 调用 Python 对象的 GetClassName() 获取 C++ 工作台类名
4. WorkbenchManager::createWorkbench(name, className) 创建 C++ Workbench 实例
5. 调用 Python 对象的 Initialize() — 导入对应模块、注册命令
6. WorkbenchManager::activate(name, type) 激活工作台
7. MainWindow::activateWorkbench() 更新 UI（菜单、工具栏）
8. 发出 signalActivateWorkbench 信号
9. 调用旧工作台的 Deactivated()
10. 调用新工作台的 Activated()
```

### C++ Workbench 层次

```
Workbench（抽象基类）
  |- setupMenuBar()     = 0  （纯虚函数）
  |- setupToolBars()    = 0
  |- setupCommandBars() = 0
  |- setupDockWindows() = 0
  |- activate()
  |
  +-- StdWorkbench        （标准工作台，定义基础菜单/工具栏）
  +-- BlankWorkbench      （空白工作台）
  +-- NoneWorkbench       （精简标准工作台）
  +-- PythonBaseWorkbench （Python 工作台基类）
      +-- PythonWorkbench （完整的 Python 工作台）
      +-- PythonBlankWorkbench
```

`PythonWorkbench` 允许通过 Python API 动态添加菜单和工具栏：

```cpp
void appendMenu(const list<string>& menu, const list<string>& items);
void appendToolbar(const string& bar, const list<string>& items);
```

---

## 命令（Command）系统

### Command 框架

所有用户操作（新建、打开、保存、创建 Feature 等）都封装为 Command 子类：

```cpp
class CommandBase {
    Action* getAction() const;
    virtual Action* createAction();
    virtual void languageChange() = 0;
    virtual void updateAction(int mode) = 0;
};

class Command : public CommandBase {
    virtual void activated() = 0;  // 命令执行时调用

    // 命令执行辅助（通过 Python 解释器）
    static void doCommand(DoCmd_Type, const char*, ...);
    static void runCommand(DoCmd_Type, const char*, ...);
    static void updateActive();     // 更新活动文档
};
```

**Command 特性**：
- 在菜单和工具栏中注册
- 可录制为宏
- 可从 Python 脚本执行
- 支持撤销/重做

**命令文件组织**（`src/Gui/`）：

| 文件 | 职责 |
|------|------|
| `CommandDoc.cpp` | 文档操作命令 |
| `CommandFeat.cpp` | Feature 操作命令 |
| `CommandStd.cpp` | 标准操作命令 |
| `CommandView.cpp` | 视图操作命令 |
| `CommandMacro.cpp` | 宏操作命令 |
| `CommandLink.cpp` | Link 操作命令 |
| `CommandWindow.cpp` | 窗口操作命令 |
| `CommandStructure.cpp` | 结构操作命令 |
| `CommandTest.cpp` | 测试命令 |

---

## 关键设计模式

### 1. 观察者模式（Observer Pattern）

```cpp
template<class MsgType>
class Observer {
    virtual void OnChange(Subject<MsgType>& rCaller, MsgType rcReason) = 0;
    virtual void OnDestroy(Subject<MsgType>& rCaller);
};

template<class MsgType>
class Subject {
    void Attach(Observer<MsgType>*);
    void Detach(Observer<MsgType>*);
    void Notify(MsgType rcReason);  // 通知所有观察者
};
```

使用场景：Console 观察者（日志输出）、DocumentObserver（文档变更通知）、参数变更通知。

### 2. 信号/槽模式（fastsignals）

FreeCAD 使用 `fastsignals` 库（非 Qt 信号槽）实现跨层通信，是 App/Gui 分离的关键技术：

```cpp
// App::Application 的信号
fastsignals::signal<void(const App::Document&)> signalNewDocument;

// Gui::Application 连接信号
App::GetApplication().signalNewDocument.connect(
    std::bind(&Gui::Application::slotNewDocument, this, ...));
```

### 3. 命令模式（Command Pattern）

所有用户操作封装为 Command 子类，支持宏录制、脚本执行、撤销/重做。

### 4. 工厂模式（Factory Pattern）

通过 Type 系统支持按类型名称动态创建对象（DocumentObject、ViewProvider）。

### 5. 单例模式（Singleton Pattern）

| 单例 | 访问方式 |
|------|----------|
| `App::Application` | `App::GetApplication()` |
| `Gui::Application` | `Gui::Application::Instance` |
| `Base::InterpreterSingleton` | `Base::Interpreter()` |
| `Gui::WorkbenchManager` | `WorkbenchManager::instance()` |
| `Gui::SelectionSingleton` | 选择系统 |

### 6. MVC 变体 — Document/ViewProvider

FreeCAD 的 MVC 实现：

- **Model** = `App::DocumentObject` + `App::Property`
- **View** = `Gui::ViewProvider` + `Gui::MDIView`
- **Controller** = `Gui::Command` + `Gui::Application`

App::DocumentObject 持有数据（Properties），Gui::ViewProvider 负责可视化（Coin3D 场景图），Gui::Command 负责用户交互和修改数据。

---

## 模块加载机制

### Python 驱动的模块加载

1. `App::Application::initApplication()` 初始化 Python 解释器
2. 创建 `FreeCAD` Python 模块（内建）
3. 运行 `FreeCADInit.py`（内建初始化脚本）
4. `FreeCADInit.py` 扫描 `Mod/` 目录，发现可用模块
5. `Base::InterpreterSingleton::loadModule()` 通过 Python 的 import 系统加载模块

### Command 注册

每个工作台通过命令名称引用命令，命令通过 `CommandManager` 全局注册：

```cpp
// 创建标准命令
void CreateStdCommands();
void CreateDocCommands();
void CreateViewStdCommands();

// 模块中注册命令
new CmdPartBox();  // 创建命令并自动注册到 CommandManager
```

---

## 构建系统

### 顶层 CMakeLists.txt 结构

```
1. 版本信息从 version.json 读取
2. 编译器检查和设置
3. 依赖查找和配置（通过 cMake/FreeCAD_Helpers/ 下的模块）：
   - SetupPython()
   - SetupOpenCasCade()
   - SetupQt()
   - SetupCoin3D()
   - SetupBoost()
   - SetupEigen()
   - SetupXercesC()
   - SetupShibokenAndPyside()
   - SetupSwig()
   - SetupSalomeSMESH()
4. 选项控制：
   - BUILD_GUI（默认 ON）— 是否构建 GUI
   - BUILD_FEM_NETGEN — FEM Netgen 支持
   - BUILD_BIM — BIM 模块
   - FREECAD_USE_CCACHE — ccache 加速
5. add_subdirectory(src)
```

### CMake Presets（CMakePresets.json）

项目预定义了多个构建预设，Windows + Pixi/Conda 环境推荐使用 `conda-windows-debug` 预设，该预设设置：

- `FREECAD_LIBPACK_USE=OFF`
- `BUILD_WITH_CONDA=ON`
- Ninja 生成器
- RelWithDebInfo 构建类型

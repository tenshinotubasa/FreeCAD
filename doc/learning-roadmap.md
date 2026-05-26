# FreeCAD 源码学习路线

> 本文档为 FreeCAD 1.2.0-dev 源码的系统性学习指南，从基础架构到核心模块，逐步深入。

---

## 前置知识

| 领域 | 要求 | 说明 |
|------|------|------|
| **C++** | 熟练掌握 C++17/20 | 模板、智能指针、RAII、多态 |
| **Python** | 基础即可 | FreeCAD 大量使用 Python 绑定和脚本 |
| **Qt 6** | 了解信号槽、事件循环 | GUI 框架基础 |
| **OpenCASCADE** | 了解拓扑/几何概念 | BREP、Shape、TopoDS_Shape |
| **CMake** | 能读懂 CMakeLists | 构建系统配置 |
| **设计模式** | 观察者、工厂、命令、MVC | FreeCAD 大量使用经典模式 |

---

## 第一阶段：项目全局认知（1-2 天）

### 1.1 项目结构总览

```
src/
├── Base/       # 基础类型系统、数学、IO、Python 绑定基础设施
├── App/        # 核心应用层：Application、Document、Property 系统
├── Gui/        # GUI 层：ViewProvider、Command、Workbench、3D 渲染
├── Mod/        # 功能模块（每个含 App/ 和 Gui/ 子目录）
│   ├── Part/          # 核心：BREP 几何、OCCT 封装
│   ├── Sketcher/      # 2D 草图、约束求解器
│   ├── PartDesign/    # 参数化零件设计
│   ├── Fem/           # 有限元分析
│   ├── BIM/           # 建筑信息模型
│   ├── Draft/         # 2D 绘图
│   ├── TechDraw/      # 技术图纸
│   ├── Assembly/      # 装配体
│   ├── CAM/           # 计算机辅助制造
│   └── ...            # 其他 30+ 模块
├── 3rdParty/   # 第三方内嵌库（GSL、OndselSolver 等）
├── Main/       # 程序入口
└── Tools/      # 开发工具脚本
```

### 1.2 核心设计原则

| 原则 | 说明 |
|------|------|
| **App/Gui 分离** | App 层完全独立于 GUI，可在无头模式运行；Gui 通过 ViewProvider 和 Command 桥接 |
| **Property 驱动** | 所有数据通过 Property 管理，自动获得序列化、Undo/Redo、表达式绑定、Python 访问 |
| **DAG 依赖图** | DocumentObject 的 InList/OutList 构成有向无环图，驱动重算顺序 |
| **Extension 体系** | 运行时行为扩展，避免深层继承 |

### 1.3 建议的首次阅读

- `src/Base/BaseClass.h` — 整个类型系统的根
- `src/App/Application.h` — 应用程序单例
- `src/App/Document.h` — 文档类
- `src/App/DocumentObject.h` — 文档对象基类
- `src/App/Property.h` — 属性系统入口

---

## 第二阶段：Base 基础层（3-5 天）

> Base 层是所有其他层的基础，不依赖 Qt 和 GUI。

### 2.1 类型系统

| 文件 | 核心内容 |
|------|---------|
| `src/Base/BaseClass.h` | 根类，定义 RTTI 宏 `TYPESYSTEM_HEADER()` / `TYPESYSTEM_SOURCE()` |
| `src/Base/Type.h` | 类型注册与查询系统，`getTypeId()`, `isDerivedFrom()`, `freecad_cast<T>()` |
| `src/Base/Factory.h` | 工厂模式基类，用于运行时创建对象 |

**关键宏**：
```cpp
TYPESYSTEM_HEADER_WITH_OVERRIDE()  // 子类声明
TYPESYSTEM_SOURCE(App::DocumentObject, App::TransactionalObject)  // 子类实现
```

### 2.2 持久化与序列化

| 文件 | 核心内容 |
|------|---------|
| `src/Base/Persistence.h` | 持久化基类，`Save()`/`Restore()` 接口 |
| `src/Base/Reader.h` | XML 输入流 |
| `src/Base/Writer.h` | XML 输出流 |

### 2.3 数学与几何类型

| 文件 | 核心内容 |
|------|---------|
| `src/Base/Vector3D.h` | 三维向量 |
| `src/Base/Matrix.h` | 4x4 矩阵 |
| `src/Base/Rotation.h` | 旋转（四元数表示） |
| `src/Base/Placement.h` | 位置+旋转 |
| `src/Base/BoundBox.h` | 轴对齐包围盒 |
| `src/Base/CoordinateSystem.h` | 坐标系 |
| `src/Base/DualQuaternion.h` | 双四元数 |
| `src/Base/UnitsApi.h` | 单位系统 |
| `src/Base/Quantity.h` | 物理量（值+单位） |

### 2.4 其他基础组件

| 文件 | 核心内容 |
|------|---------|
| `src/Base/Console.h` | 日志系统（Log/Warning/Error/Direct） |
| `src/Base/Parameter.h` | 参数配置系统（参数组树结构） |
| `src/Base/Exception.h` | 异常体系 |
| `src/Base/Handle.h` | 引用计数智能指针 |
| `src/Base/Observer.h` | 观察者模式基类 |
| `src/Base/Interpreter.h` | Python 解释器封装 |

### 2.5 Python 绑定基础

| 文件 | 核心内容 |
|------|---------|
| `src/Base/PyObjectBase.h` | Python 对象封装基类 |
| `src/Base/PyWrap.h` | Python 包装辅助宏 |

**练习**：阅读 `src/Base/Vector3D.h` 和 `src/Base/VectorPyImp.cpp`，理解 C++ 类如何暴露给 Python。

---

## 第三阶段：App 核心层（5-7 天）

> App 层是 FreeCAD 的核心，定义了文档、对象、属性的完整模型。

### 3.1 Property 属性系统

**这是 FreeCAD 最重要的系统，必须深入理解。**

```
Base::Persistence
  └── App::Property (属性基类)
        ├── PropertyString, PropertyBool, PropertyFloat, PropertyInt
        ├── PropertyLink, PropertyLinkList, PropertyLinkSub
        ├── PropertyEnumeration
        ├── PropertyPlacement, PropertyColor, PropertyFile
        ├── PropertyExpressionEngine
        ├── PropertyXLink, PropertyXLinkList
        └── ... (40+ 种属性类型)
```

| 文件 | 核心内容 |
|------|---------|
| `src/App/Property.h` | 属性基类，定义触摸机制、属性标志 |
| `src/App/PropertyStandard.h` | 基本类型属性（String, Bool, Float, Int 等） |
| `src/App/PropertyLinks.h` | 链接属性（对象间引用关系） |
| `src/App/PropertyGeoExtension.h` | 几何扩展属性 |
| `src/App/DynamicProperty.h` | 动态属性管理（运行时添加属性） |

**关键概念**：
- **属性标志**：`Prop_ReadOnly`, `Prop_Transient`, `Prop_Hidden`, `Prop_Output`, `Prop_NoRecompute`, `Prop_NoPersist`
- **触摸机制**：属性变更 → `touch()` → 标记 DocumentObject 需要重算
- **属性容器**：`PropertyContainer` 管理静态属性和动态属性

### 3.2 PropertyContainer 与 DocumentObject

```
Base::BaseClass
  └── Base::Persistence
        └── App::PropertyContainer (属性容器)
              ├── App::Document (文档)
              └── App::TransactionalObject (事务对象)
                    └── App::DocumentObject (文档对象)
                          └── App::GeoFeature (几何特征)
```

| 文件 | 核心内容 |
|------|---------|
| `src/App/PropertyContainer.h` | 属性容器基类，`getPropertyByName()`, `addDynamicProperty()` |
| `src/App/DocumentObject.h` | 文档对象，`execute()`, `recompute()`, `touch()`, InList/OutList |
| `src/App/GeoFeature.h` | 几何特征，添加 `Placement` 属性和 `Shape` 概念 |
| `src/App/DocumentObjectGroup.h` | 对象分组 |
| `src/App/FeaturePython.h` | Python 自定义特征 |

**关键方法**：
- `DocumentObject::execute()` — 纯虚方法，子类实现具体计算逻辑
- `DocumentObject::mustExecute()` — 判断是否需要重算
- `DocumentObject::getViewProviderName()` — **App 与 Gui 的桥梁**
- `DocumentObject::onChanged()` — 属性变更回调

### 3.3 Document 文档

| 文件 | 核心内容 |
|------|---------|
| `src/App/Document.h` | 文档类，管理对象集合、Undo/Redo、重算、DAG |
| `src/App/DocumentPyImp.cpp` | Document 的 Python 接口 |

**关键功能**：
- `addObject()`, `removeObject()` — 对象生命周期管理
- `recompute()` — 拓扑排序后执行重算链
- `openTransaction()`, `commitTransaction()` — Undo/Redo 事务
- `openTransaction()` → 修改 → `commitTransaction()` → `undo()` / `redo()`

### 3.4 Application 应用

| 文件 | 核心内容 |
|------|---------|
| `src/App/Application.h` | 应用程序单例（`GetApplication()`） |

**关键职责**：
- 文档生命周期管理
- 事务管理
- 参数管理（系统参数 + 用户参数）
- 导入导出类型注册
- 信号分发：`signalNewDocument`, `signalDeleteDocument`, `signalNewObject`, `signalChangedObject`, `signalRecomputed`

### 3.5 Expression 表达式引擎

| 文件 | 核心内容 |
|------|---------|
| `src/App/Expression.h` | 表达式抽象语法树 |
| `src/App/ExpressionParser.h` | 表达式解析器 |
| `src/App/PropertyExpressionEngine.h` | 表达式引擎属性 |

允许在属性间建立数学引用，如 `Length = Width * 2`。

### 3.6 Extension 扩展体系

| 文件 | 核心内容 |
|------|---------|
| `src/App/Extension.h` | 扩展基类 |
| `src/App/ExtensionContainer.h` | 扩展容器 |
| `src/App/GroupExtension.h` | 分组扩展 |
| `src/App/GeoFeatureGroupExtension.h` | 几何分组扩展 |
| `src/App/OriginGroupExtension.h` | 原点分组扩展 |
| `src/App/AttachExtension.h` | 附着扩展 |

Extension 体系允许运行时给对象添加行为，避免深层继承。

### 3.7 Link 链接系统

| 文件 | 核心内容 |
|------|---------|
| `src/App/Link.h` | 跨文档链接、链接数组（~100KB，最复杂的单个文件之一） |

**练习**：
1. 用 Python 控制台创建一个 DocumentObject，观察其属性变化
2. 创建两个对象，设置 PropertyLink，观察 DAG 关系
3. 使用表达式引擎建立属性间引用

---

## 第四阶段：Gui 图形界面层（5-7 天）

> Gui 层基于 Qt 和 Coin3D，实现可视化和用户交互。

### 4.1 ViewProvider 视图提供者

**ViewProvider 是 MVC 模式中 View 的实现，是 App 与 Gui 的核心桥梁。**

```
App::TransactionalObject
  └── Gui::ViewProvider
        ├── Gui::ViewProviderDocumentObject (关联到 App::DocumentObject)
        │     ├── Gui::ViewProviderGeometryObject (几何显示)
        │     ├── Gui::ViewProviderFeature (基础特征)
        │     ├── Gui::ViewProviderDatum (基准面/轴/点)
        │     └── Gui::ViewProviderLink (链接对象)
        └── Gui::ViewProviderExtern (外部 VP)
```

| 文件 | 核心内容 |
|------|---------|
| `src/Gui/ViewProvider.h` | 视图提供者基类 |
| `src/Gui/ViewProviderDocumentObject.h` | 与 DocumentObject 关联的 VP |
| `src/Gui/ViewProviderGeometryObject.h` | 几何显示 VP |
| `src/Gui/ViewProviderFeature.h` | 基础特征 VP |

**关键方法**：
- `attach()` — 将 App 属性关联到 Coin3D 场景节点
- `updateData()` — 响应 App 属性变更，更新 3D 显示
- `getDisplayMaskMode()` / `setDisplayMaskMode()` — 显示模式切换
- `getDefaultDisplayMaskMode()` — 默认显示模式
- `onDelete()` — 删除前清理

### 4.2 Command 命令框架

| 文件 | 核心内容 |
|------|---------|
| `src/Gui/Command.h` | 命令基类，核心虚方法 `activated(int iMsg)` |
| `src/Gui/Command.cpp` | 命令管理器和命令注册 |

**命令类型**：
- `Command::Doc` — 修改文档的命令
- `Command::App` — 修改应用的命令
- `Command::Gui` — 修改界面的命令

**关键宏**：
```cpp
DEF_STD_CMD()      // 标准命令
DEF_STD_CMD_A()    // 带动作的命令
DEF_STD_CMD_AC()   // 带动作和检查的命令
```

**命令执行流程**：用户触发 → `activated()` → `doCommand()` → Python 解释器执行 → 修改文档 → 信号触发 → ViewProvider 更新

### 4.3 Application 与 MainWindow

| 文件 | 核心内容 |
|------|---------|
| `src/Gui/Application.h` | GUI 应用单例，管理工作台、视图、命令 |
| `src/Gui/MainWindow.h` | Qt 主窗口 |
| `src/Gui/MDIView.h` | 多文档视图接口 |
| `src/Gui/View3DInventor.h` | 3D 视图（Coin3D 渲染） |

### 4.4 Selection 与 Workbench

| 文件 | 核心内容 |
|------|---------|
| `src/Gui/Selection.h` | 全局选择管理器（单例） |
| `src/Gui/Workbench.h` | 工作台抽象 |
| `src/Gui/WorkbenchManager.h` | 工作台管理器 |

### 4.5 Coin3D 场景图

| 文件 | 核心内容 |
|------|---------|
| `src/Gui/SoFCSelection.h` | FreeCAD 选择节点 |
| `src/Gui/SoBrepEdgeSet.h` | BREP 边渲染节点 |
| `src/Gui/SoBrepFaceSet.h` | BREP 面渲染节点 |
| `src/Gui/NaviCube.h` | 导航立方体 |

**练习**：
1. 为一个简单的 DocumentObject 编写 ViewProvider
2. 创建自定义 Command 并注册到工作台
3. 在 ViewProvider 的 `updateData()` 中设置断点，观察属性变更如何传播到 GUI

---

## 第五阶段：Part 模块——几何核心（5-7 天）

> Part 模块是 FreeCAD 的几何引擎核心，封装 OpenCASCADE（OCCT）。

### 5.1 Part::Feature — 几何特征的基石

```
App::DocumentObject
  └── App::GeoFeature
        └── Part::Feature (核心：持有 TopoDS_Shape)
              ├── Part::FilletBase
              ├── Part::FeatureExt
              └── Part::Part2DObject (2D 对象基类)
```

| 文件 | 核心内容 |
|------|---------|
| `src/Mod/Part/App/PartFeature.h` | `Part::Feature`，核心 Shape 属性持有者 |
| `src/Mod/Part/App/Geometry.h` | 几何曲线/曲面封装（OCCT Geom_Curve 等，~240KB） |
| `src/Mod/Part/App/TopoShape.h` | TopoDS_Shape 的高级封装 |

### 5.2 拓扑特征

| 文件 | 核心内容 |
|------|---------|
| `FeatureExtrusion.h` | 拉伸 |
| `FeatureRevolution.h` | 旋转 |
| `FeatureFillet.h` | 倒角 |
| `FeatureChamfer.h` | 倒棱 |
| `FeatureMirroring.h` | 镜像 |
| `FeatureOffset.h` | 偏移 |
| `FeatureScale.h` | 缩放 |

### 5.3 布尔运算

| 文件 | 核心内容 |
|------|---------|
| `FeaturePartBoolean.h` | 布尔运算基类 |
| `FeaturePartCut.h` | 差集 |
| `FeaturePartFuse.h` | 并集 |
| `FeaturePartCommon.h` | 交集 |
| `FeaturePartSection.h` | 截面 |

### 5.4 基本体

| 文件 | 核心内容 |
|------|---------|
| `FeaturePartBox.h` | 长方体 |
| `FeaturePartCircle.h` | 圆 |
| `FeaturePartPolygon.h` | 多边形 |
| `PrimitiveFeature.h` | 基本体基类 |

### 5.5 附着系统

| 文件 | 核心内容 |
|------|---------|
| `AttachExtension.h` | 附着扩展 |
| `AttachEngine.h` | 附着引擎（将对象附着到其他对象的几何引用上） |
| `Attacher.h` | 附着器实现 |

### 5.6 导入导出

| 文件 | 核心内容 |
|------|---------|
| `ImportIges.h` | IGES 导入 |
| `ImportStep.h` | STEP 导入 |

### 5.7 Part Gui

| 文件 | 核心内容 |
|------|---------|
| `ViewProvider.h` | Part 对象的视图提供者 |
| `SoBrepEdgeSet.h` | BREP 边渲染自定义节点 |
| `SoBrepFaceSet.h` | BREP 面渲染自定义节点 |
| `Workbench.h` | Part 工作台定义 |

**练习**：
1. 阅读 `Part::Feature::execute()` 的实现，理解 Shape 的生成过程
2. 创建一个自定义的 Part Feature，实现简单的几何操作
3. 跟踪一次布尔运算的执行流程

---

## 第六阶段：Sketcher 模块——草图与约束（3-5 天）

> Sketcher 是 FreeCAD 参数化设计的关键模块，基于 Part::Part2DObject。

### 6.1 核心类

| 文件 | 核心内容 |
|------|---------|
| `SketchObject.h` (~77KB) | 2D 草图对象，管理几何和约束 |
| `Sketch.h` (~201KB) | 约束求解器核心实现 |
| `Constraint.h` | 约束定义（距离、角度、水平、垂直、相切、对称等） |

### 6.2 求解器

| 文件 | 核心内容 |
|------|---------|
| `planegcs/` | 平面几何约束求解器（GCS，外部库） |
| `SketchAnalysis.h` | 草图分析（检测冲突、冗余约束） |
| `SolverGeometryExtension.h` | 求解器几何扩展 |

### 6.3 Gui 交互

| 文件 | 核心内容 |
|------|---------|
| `ViewProviderSketch.h` (~194KB) | 草图编辑器核心视图提供者 |
| `CommandConstraints.h` (~436KB) | 约束命令（最大单文件） |
| `DrawSketchHandler.h` | 绘制交互处理器（Line, Circle, Arc, Rectangle 等 20+ 种） |
| `EditModeCoinManager.h` | Coin3D 渲染管理 |
| `SnapManager.h` | 捕捉管理 |

**练习**：
1. 在 FreeCAD 中创建一个草图，添加几何和约束，用调试器跟踪求解过程
2. 阅读 `SketchObject::execute()` 理解草图如何生成 Part Shape

---

## 第七阶段：PartDesign 模块——参数化设计（3-5 天）

> PartDesign 基于 Part 和 Sketcher，实现特征树式参数化建模。

### 7.1 核心概念

- **Body**：一个 PartDesign 容器，维护特征树的有序链
- **Feature**：每个操作步骤（Pad, Pocket, Hole, Revolution 等）
- **Sketch**：作为特征树的输入几何

### 7.2 关键文件

| 文件 | 核心内容 |
|------|---------|
| `src/Mod/Part/BodyBase.h` | Body 基类（在 Part 模块中定义） |
| `src/Mod/PartDesign/App/Feature.h` | PartDesign Feature 基类 |
| `src/Mod/PartDesign/App/FeaturePad.h` | 拉伸特征 |
| `src/Mod/PartDesign/App/FeaturePocket.h` | 切槽特征 |
| `src/Mod/PartDesign/App/FeatureHole.h` | 孔特征 |
| `src/Mod/PartDesign/App/FeatureRevolution.h` | 旋转特征 |
| `src/Mod/PartDesign/App/FeatureFillet.h` | 倒角特征 |
| `src/Mod/PartDesign/App/FeatureChamfer.h` | 倒棱特征 |
| `src/Mod/PartDesign/App/FeatureMirror.h` | 镜像特征 |
| `src/Mod/PartDesign/App/FeaturePattern.h` | 阵列特征 |

**练习**：创建一个 PartDesign Body，添加 Sketch → Pad → Pocket 序列，跟踪 `execute()` 链的调用过程。

---

## 第八阶段：FEM 模块——有限元分析（3-5 天）

> FEM 模块展示了如何将专业领域逻辑集成到 FreeCAD 框架中。

### 8.1 核心架构

| 文件 | 核心内容 |
|------|---------|
| `FemMesh.h` (~97KB) | 有限元网格核心 |
| `FemAnalysis.h` | 分析容器 |
| `FemSolverObject.h` | 求解器对象 |
| `FemResultObject.h` | 结果对象 |
| `FemPostPipeline.h` | 后处理管线 |

### 8.2 约束系统

约 15 种约束类型：Fixed, Force, Displacement, Pressure, Temperature, Heatflux, Bearing, Gear, Contact, RigidBody, Spring, FluidBoundary, PlaneRotation, Pulley, Transform

### 8.3 求解器前端

通过 Python 实现多种求解器前端：CalculiX, Elmer, Z88, Mystran

### 8.4 VTK 可视化

后处理使用 VTK 管线进行结果可视化。

**练习**：完成一个简单的 FEM 分析流程（创建网格 → 添加约束 → 求解 → 查看结果），跟踪各步骤对应的 C++ 对象。

---

## 第九阶段：其他重要模块（按需选读）

### 9.1 BIM 模块
建筑信息模型，基于 IfcOpenShell 解析 IFC 文件。

### 9.2 Draft 模块
2D 绘图工具，大量使用 Python 实现。

### 9.3 TechDraw 模块
技术图纸生成，从 3D 模型投影出 2D 工程图。

### 9.4 Assembly 模块
新版装配体模块，大量使用 Python 实现。

### 9.5 CAM 模块
计算机辅助制造（路径规划），包含 C++ 的 `area` 模块（pybind11）和 Python 的路径生成逻辑。

### 9.6 Mesh 模块
三角网格处理，与 Part 模块互为补充。

---

## 第十阶段：进阶主题

### 10.1 创建自定义模块

学习 `src/Mod/TemplatePyMod/` 中的 Python 模块模板，了解：
- 如何注册 Workbench
- 如何创建自定义 DocumentObject（C++ 或 Python）
- 如何创建对应的 ViewProvider
- 如何注册 Command

### 10.2 Python 绑定开发

- 学习 `src/Base/PyObjectBase.h` 和 SWIG 绑定模式
- 阅读 `src/Mod/Part/App/PartFeaturePyImp.cpp` 了解 `Part::Feature` 的 Python 接口
- `.pyi` 存根文件为 IDE 提供类型提示

### 10.3 Undo/Redo 机制

- `App::Transaction` 实现 Undo/Redo
- Property 的 `hasSetValue()` 触发事务记录
- `Document::undo()` / `redo()` 恢复属性状态

### 10.4 DAG 与依赖追踪

- `DocumentObject::getOutList()` / `getInList()` 构成 DAG
- `testIfLinkDAGCompatible()` 防止循环引用
- `Document::topologicalSort()` 确定重算顺序

### 10.5 信号系统

- 使用 `fastsignals` 库（非 Qt 信号槽）
- App 层定义核心信号，Gui 层连接响应

### 10.6 测试

| 目录 | 内容 |
|------|------|
| `tests/src/App/` | App 层单元测试 |
| `tests/src/Base/` | Base 层单元测试 |
| `tests/src/Gui/` | Gui 层单元测试 |
| `tests/src/Mod/` | 模块测试 |
| `tests/visual/` | 视觉回归测试 |

运行测试：`pixi run test-debug` 或 `ctest --test-dir build/debug`

---

## 附录 A：核心类继承关系图

```
Base::BaseClass
  └── Base::Persistence
        ├── App::Property (所有属性的基类)
        └── App::PropertyContainer
              ├── App::Document
              └── App::TransactionalObject
                    ├── App::DocumentObject
                    │     └── App::GeoFeature
                    │           └── Part::Feature
                    │                 ├── Part::Part2DObject
                    │                 │     └── Sketcher::SketchObject
                    │                 └── PartDesign::Feature
                    └── Gui::ViewProvider
                          └── Gui::ViewProviderDocumentObject
                                └── Gui::ViewProviderGeometryObject
                                      └── PartGui::ViewProviderPartExt
```

## 附录 B：模块编译产物

每个模块 `Mod/XXX/` 编译生成两个库：
- `FreeCADXXX` — App 层（Mod/XXX/App/）
- `FreeCADXXXGui` — Gui 层（Mod/XXX/Gui/）

## 附录 C：关键文件大小参考

| 文件 | 大小 | 说明 |
|------|------|------|
| `src/App/Document.cpp` | ~140KB | 文档实现 |
| `src/App/Application.cpp` | ~138KB | 应用实现 |
| `src/App/Expression.cpp` | ~112KB | 表达式引擎 |
| `src/App/Link.cpp` | ~100KB | 链接系统 |
| `src/Gui/Command.cpp` | ~67KB | 命令框架 |
| `src/Mod/Part/App/Geometry.cpp` | ~240KB | 几何封装 |
| `src/Mod/Sketcher/App/Sketch.cpp` | ~201KB | 约束求解器 |
| `src/Mod/Sketcher/Gui/ViewProviderSketch.cpp` | ~194KB | 草图视图 |
| `src/Mod/Sketcher/Gui/CommandConstraints.cpp` | ~436KB | 约束命令 |

## 附录 D：参考资源

- [FreeCAD Developers Handbook](https://freecad.github.io/DevelopersHandbook/gettingstarted/)
- [FreeCAD Wiki](https://wiki.freecad.org/)
- [OpenCASCADE Documentation](https://dev.opencascade.org/doc/overview/)
- [Coin3D Documentation](https://coin3d.github.io/Coin/html/)
- [FreeCAD Forum](https://forum.freecad.org/)
- [FreeCAD GitHub](https://github.com/FreeCAD/FreeCAD)

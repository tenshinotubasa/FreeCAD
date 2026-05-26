# FreeCAD VSCode 调试指南

> 本文档基于 FreeCAD 1.2.0-dev 版本，适用于 Windows 平台（MSVC 编译器 + VSCode）。

---

## 1. 前置条件

### 1.1 必需的 VSCode 扩展

| 扩展 | ID | 用途 |
|------|----|------|
| **C/C++** | `ms-vscode.cpptools` | C++ 断点、变量查看、调用栈 |
| **CMake Tools** | `ms-vscode.cmake-tools` | CMake 构建/配置集成，提供 `${command:cmake.buildDirectory}` 变量 |

### 1.2 可选的 VSCode 扩展

| 扩展 | ID | 用途 |
|------|----|------|
| **Python** | `ms-python.python` | Python 断点调试（C++ + Python 联调时需要） |
| **CMake** | `twxs.cmake` | CMakeLists.txt 语法高亮 |

### 1.3 确认构建产物

调试需要 PDB 符号文件，确认以下文件存在于 `build/debug/bin/` 下：

```
FreeCAD.exe          # 主程序
FreeCAD.pdb          # 主程序调试符号
FreeCADApp.dll/pdb   # App 层
FreeCADBase.dll/pdb  # Base 层
FreeCADGui.dll/pdb   # GUI 层
```

> 若 PDB 缺失，需重新编译。Windows Conda Debug 预设的构建类型为 `RelWithDebInfo`，会自动生成 PDB。

---

## 2. 配置 CMake Tools

首次使用前必须选择构建预设，否则 `${command:cmake.buildDirectory}` 无法正确解析：

1. `Ctrl+Shift+P` → **CMake: Select Configure Preset** → 选择 `conda-windows-debug`
2. `Ctrl+Shift+P` → **CMake: Select Build Preset** → 选择对应的 build preset

这样 CMake Tools 会将构建目录识别为 `build/debug`。

---

## 3. 调试配置说明

项目 `.vscode/launch.json` 中预配置了以下调试方案：

### 3.1 配置一览

| 配置名 | 调试器类型 | 用途 |
|--------|-----------|------|
| **(Windows) Debug FreeCAD** | `cppvsdbg` (MSVC) | 纯 C++ 调试，**最常用** |
| **(Windows) Debug FreeCAD + Python** | `cppvsdbg` (MSVC) | C++ 调试 + 启动 debugpy（供 Python attach） |
| **(Windows) Debug C++ Tests** | `cppvsdbg` (MSVC) | 调试 C++ 单元测试 |
| (Windows) Debug FreeCAD C++ + Python | 组合启动 | 同时调试 C++ 和 Python |
| Python debugger | `debugpy` | 附加到 FreeCAD 内的 Python 进程 |

### 3.2 各配置详解

#### (Windows) Debug FreeCAD

最常用的配置，启动 FreeCAD 并使用 MSVC 调试器：

```json
{
  "name": "(Windows) Debug FreeCAD",
  "type": "cppvsdbg",
  "request": "launch",
  "program": "${command:cmake.buildDirectory}/bin/FreeCAD.exe",
  "args": [],
  "environment": [
    { "name": "PATH", "value": "${command:cmake.buildDirectory}/bin;${env:PATH}" }
  ]
}
```

- `type: cppvsdbg` — 使用 Visual Studio 的 MSVC 调试器（Windows 专用）
- `args` — 可添加命令行参数，如 `["-t", "0"]` 运行测试模式

#### (Windows) Debug FreeCAD C++ + Python

组合配置，同时启动 C++ 调试器和 Python 调试器：

1. **第一步**：启动 FreeCAD，并通过自动执行宏（`VSCodeAutostartDebug.FCMacro`）在端口 5679 启动 debugpy 服务端
2. **第二步**：`WaitForDebugpy.py` 脚本轮询端口 39999，等待 debugpy 就绪
3. **第三步**：Python 调试器 attach 到 localhost:5679

流程如下：

```
VSCode 启动 C++ 调试器
  → FreeCAD.exe 启动（带 MSVC 调试器）
    → VSCodeAutostartDebug.FCMacro 自动执行
      → debugpy.listen(5679) 在 FreeCAD 进程内启动
        → WaitForDebugpy.py 检测到 debugpy 就绪
          → VSCode 的 Python 调试器 attach 到 5679
```

---

## 4. 快速开始：纯 C++ 调试

### 4.1 步骤

1. 按 `Ctrl+Shift+D` 打开调试面板
2. 在顶部下拉框选择 **(Windows) Debug FreeCAD**
3. 在源码中设置断点（见下文推荐位置）
4. 按 `F5` 启动调试

### 4.2 推荐的断点位置

| 文件 | 函数/行 | 用途 |
|------|---------|------|
| `src/App/Document.cpp` | `Document::recompute()` | 跟踪文档重算流程 |
| `src/App/DocumentObject.cpp` | `DocumentObject::execute()` | 跟踪对象执行 |
| `src/App/DocumentObject.cpp` | `DocumentObject::touch()` | 跟踪属性触摸传播 |
| `src/App/Property.cpp` | `Property::hasSetValue()` | 跟踪所有属性变更 |
| `src/App/Application.cpp` | `Application::recompute()` | 跟踪全局重算调度 |
| `src/Gui/ViewProvider.cpp` | `ViewProvider::updateData()` | 跟踪 GUI 响应属性变更 |
| `src/Gui/Command.cpp` | `Command::doCommand()` | 跟踪命令执行 |
| `src/Gui/Application.cpp` | `Gui::Application::slotNewObject()` | 跟踪新对象创建的信号响应 |
| `src/Mod/Part/App/PartFeature.cpp` | `Part::Feature::execute()` | 跟踪 Part 特征计算 |
| `src/Mod/Sketcher/App/SketchObject.cpp` | `SketchObject::execute()` | 跟踪草图求解 |

### 4.3 条件断点

右键断点 → **编辑断点**，可设置条件，例如：

- `Property::hasSetValue()` 中设置条件 `strcmp(this->getName(), "Shape") == 0` — 只在 Shape 属性变更时中断
- `DocumentObject::execute()` 中设置条件 `strcmp(this->getNameInDocument(), "Pad") == 0` — 只在名为 Pad 的对象执行时中断

### 4.4 日志断点

右键断点 → **编辑断点** → 类型选 **Logpoint**，输出日志而不暂停程序：

- 在 `DocumentObject::touch()` 设置日志断点：`Touched: {this->getNameInDocument()}`

---

## 5. C++ + Python 联合调试

### 5.1 步骤

1. 确保安装了 **Python** 扩展
2. 按 `Ctrl+Shift+D` 打开调试面板
3. 选择 **(Windows) Debug FreeCAD C++ + Python**
4. 按 `F5` 启动

启动后：
- C++ 断点在 VSCode 中直接命中
- Python 断点在 FreeCAD 内的 Python 代码（包括 Mod 中的 `.py` 文件）中命中

### 5.2 Python 断点设置

可以直接在以下文件中设置 Python 断点：

- `src/Mod/Draft/Draft.py`
- `src/Mod/Fem/femsolver/calculix/solver.py`
- `src/Mod/Assembly/App/AssemblyObject.py`
- 以及其他任何 FreeCAD 加载的 `.py` 文件

### 5.3 注意事项

- 组合调试启动较慢，需要等待 debugpy 初始化（约 5-15 秒）
- 若 Python 调试器 attach 失败，检查 5679/39999 端口是否被占用
- `PYDEVD_DISABLE_FILE_VALIDATION` 环境变量已设置为 `1`，避免 debugpy 校验问题

---

## 6. 调试单元测试

### 6.1 C++ 测试

选择 **(Windows) Debug C++ Tests** 配置，调试 `Base_tests_run.exe`。

如需调试其他测试可执行文件，修改 `program` 字段：

```
build/debug/bin/App_tests_run.exe
build/debug/bin/Part_tests_run.exe
build/debug/bin/Sketcher_tests_run.exe
```

### 6.2 通过命令行运行测试并调试

在 `launch.json` 的 args 中添加 `--gtest_filter`：

```json
"args": ["--gtest_filter=DocumentTest.testAddObject"]
```

### 6.3 Python 测试

在 FreeCAD 中运行测试模式：

```json
"args": ["-t", "0"]
```

---

## 7. 调试技巧

### 7.1 常用快捷键

| 操作 | 快捷键 |
|------|--------|
| 设置/取消断点 | `F9` |
| 启动调试 | `F5` |
| 单步跳过 (Step Over) | `F10` |
| 单步进入 (Step Into) | `F11` |
| 单步跳出 (Step Out) | `Shift+F11` |
| 继续运行 | `F5` |
| 停止调试 | `Shift+F5` |
| 重启调试 | `Ctrl+Shift+F5` |

### 7.2 调试时查看变量

- **变量面板**：自动显示当前作用域的局部变量
- **监视面板**：手动添加表达式，如 `this->Label.getStrValue()`
- **调用栈面板**：查看当前调用链
- **即时窗口**：Debug Console 中可输入表达式求值

### 7.3 查看 FreeCAD 特有类型

| 类型 | 监视表达式 |
|------|-----------|
| DocumentObject 名称 | `this->getNameInDocument()` |
| DocumentObject Label | `this->Label.getStrValue()` |
| Property 名称 | `this->getName()` |
| Property 类型 | `this->getTypeId().getName()` |
| Document 对象数量 | `this->countObjects()` |
| 对象的 InList | `this->getInList()` |
| 对象的 OutList | `this->getOutList()` |
| 对象状态 | `(int)this->getStatus()` |

### 7.4 调试 Property 变更传播

FreeCAD 的核心数据流是：**Property 变更 → touch() → Document recompute → execute()**

在以下位置设置断点，逐步跟踪：

1. `Property::hasSetValue()` — 属性变更的起点
2. `PropertyContainer::onChanged()` — 容器收到属性变更通知
3. `DocumentObject::touch()` — 对象被标记为需要重算
4. `Document::recompute()` — 文档执行重算
5. `DocumentObject::execute()` — 具体对象执行计算

### 7.5 调试信号-槽连接

FreeCAD 使用 `boost::signals2`（非 Qt 信号槽）。关键信号定义在：

| 类 | 信号 | 触发时机 |
|----|------|---------|
| `App::Application` | `signalNewDocument` | 新建文档 |
| `App::Application` | `signalNewObject` | 新建对象 |
| `App::Application` | `signalDeletedObject` | 删除对象 |
| `App::Application` | `signalChangedObject` | 对象属性变更 |
| `App::Application` | `signalRecomputed` | 文档重算完成 |
| `App::Document` | `signalNewObject` | 文档内新建对象 |
| `Gui::Application` | `signalActivateWorkbench` | 切换工作台 |

在 `src/Gui/Application.cpp` 的信号处理函数中设断点，可观察 App→Gui 的信号传播。

### 7.6 调试 ViewProvider

ViewProvider 是 App→Gui 的桥梁。调试方法：

1. 在 `Gui::ViewProviderDocumentObject::attach()` 设断点 — 观察初始化
2. 在 `Gui::ViewProviderDocumentObject::updateData()` 设断点 — 观察 App 属性变更如何传播到 Gui
3. 在 `Gui::ViewProviderDocumentObject::getDisplayMaskMode()` 设断点 — 观察显示模式

### 7.7 调试 DAG 依赖

在 `Document::topologicalSort()` 设断点，观察重算排序过程。

在 Python 控制台中查看 DAG：

```python
obj = App.ActiveDocument.getObject("Pad")
print("InList:", [o.Name for o in obj.InList])
print("OutList:", [o.Name for o in obj.OutList])
```

---

## 8. 常见问题

### 8.1 断点不命中（灰色圆圈）

| 原因 | 解决方法 |
|------|---------|
| CMake Tools 未选择预设 | `Ctrl+Shift+P` → CMake: Select Configure Preset |
| 源码与 PDB 不匹配 | 重新编译：`pixi run build-debug` |
| 文件路径不匹配 | 检查 `c_cpp_properties.json` 的 `includePath` |
| 优化导致代码被内联 | RelWithDebInfo 模式下部分函数可能被优化 |

### 8.2 CMake 构建失败：moc.exe 无法运行

**错误特征**：`Exit code 0xc0000135`（`STATUS_DLL_NOT_FOUND`）

```
CMake Error: AUTOMOC for target libfastsignals: Test run of "moc" executable
".../.pixi/envs/default/Library/lib/qt6/moc.exe" failed.
Exit code 0xc0000135
```

**原因**：VSCode 的 CMake Tools 扩展直接调用 CMake 时，没有 Pixi 环境的 PATH，导致 `moc.exe` 找不到依赖的 DLL（如 `pcre2-16.dll`、MinGW 运行时等）。

**解决方法**：在 `.vscode/settings.json` 中配置 `cmake.environment`，将 Pixi 环境的关键路径加入 PATH：

```json
"cmake.environment": {
  "PATH": "${workspaceFolder}/.pixi/envs/default;${workspaceFolder}/.pixi/envs/default/Library/mingw-w64/bin;${workspaceFolder}/.pixi/envs/default/Library/usr/bin;${workspaceFolder}/.pixi/envs/default/Library/bin;${workspaceFolder}/.pixi/envs/default/Library/lib/qt6;${workspaceFolder}/.pixi/envs/default/Scripts;${workspaceFolder}/.pixi/envs/default/bin;${env:PATH}",
  "CONDA_PREFIX": "${workspaceFolder}/.pixi/envs/default"
}
```

> **关键路径说明**：
> - `Library/mingw-w64/bin` — 提供 `libstdc++-6.dll`、`libwinpthread-1.dll` 等 MinGW 运行时
> - `Library/usr/bin` — 提供 Conda 工具链
> - `Library/bin` — 提供 `pcre2-16.dll`、`Qt6Core.dll` 等
> - `Library/lib/qt6` — 提供 Qt 工具（moc, uic, rcc）

如果上述方法仍不生效，可以改用 `pixi run` 执行构建和调试任务，绕过 CMake Tools 扩展的环境问题。

### 8.3 调试器启动失败

| 原因 | 解决方法 |
|------|---------|
| 找不到 FreeCAD.exe | 检查 `program` 路径是否指向 `build/debug/bin/FreeCAD.exe` |
| DLL 加载失败 | 确保 `PATH` 环境变量包含 `build/debug/bin` |
| 缺少 MSVC 调试器 | 安装 Visual Studio 的 C++ 桌面开发工作负载 |

### 8.4 Python 联调失败

| 原因 | 解决方法 |
|------|---------|
| debugpy 未安装 | 在 Pixi 环境中确认：`pixi run python -c "import debugpy"` |
| 端口被占用 | 关闭其他占用 5679/39999 的进程 |
| WaitForDebugpy 超时 | 增大 `WaitForDebugpy.py` 中的 `TIMEOUT_TIME_S` |
| 宏未执行 | 检查 `args` 是否包含 `VSCodeAutostartDebug.FCMacro` 路径 |

### 8.5 性能问题

- 调试模式下 FreeCAD 运行较慢是正常的（MSVC 调试器会拦截所有异常）
- 可在 `cppvsdbg` 配置中添加 `"exceptionHandling": { "exceptions": [{ "type": "cpp", "name": "std::exception", "break": false }] }` 忽略 C++ 异常中断

---

## 9. 高级：自定义调试配置

### 9.1 调试特定模块

如需只调试某个模块（如 Part），可以单独加载调试符号：

在 `launch.json` 中添加 `symbolLoadSetting`：

```json
{
  "name": "Debug Part Module",
  "type": "cppvsdbg",
  "request": "launch",
  "program": "${command:cmake.buildDirectory}/bin/FreeCAD.exe",
  "symbolLoadSetting": {
    "loadAll": false,
    "exceptionList": [
      "${command:cmake.buildDirectory}/bin/FreeCADApp.pdb",
      "${command:cmake.buildDirectory}/bin/FreeCADGui.pdb",
      "${command:cmake.buildDirectory}/Mod/Part/Part.pdb"
    ]
  }
}
```

### 9.2 附加到正在运行的 FreeCAD

如果 FreeCAD 已经在运行，可以附加调试器：

1. `Ctrl+Shift+D` → 创建 `launch.json` 添加 attach 配置：

```json
{
  "name": "Attach to FreeCAD",
  "type": "cppvsdbg",
  "request": "attach",
  "processName": "FreeCAD.exe"
}
```

2. 选择该配置并 `F5` 启动

### 9.3 远程调试（WSL / 远程机器）

使用 SSH 远程调试时，在远程机器的 `.vscode/launch.json` 中使用 Linux 配置（`cppdbg` + GDB）。

---

## 10. 参考资源

- [VSCode C++ 调试文档](https://code.visualstudio.com/docs/cpp/cpp-debug)
- [VSCode Python 调试文档](https://code.visualstudio.com/docs/python/debugging)
- [debugpy 文档](https://github.com/microsoft/debugpy)
- [CMake Tools 扩展文档](https://github.com/microsoft/vscode-cmake-tools)

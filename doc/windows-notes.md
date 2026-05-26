# Windows 环境下 FreeCAD 编译与调试注意事项

> 基于 FreeCAD 1.2.0-dev、Windows 11、MSVC 19.44、Pixi 0.69.0 的实战经验总结。

---

## 1. 环境准备

### 1.1 必需软件

| 软件 | 版本 | 说明 |
|------|------|------|
| **Visual Studio 2022** | Community 即可 | 需安装"使用 C++ 的桌面开发"工作负载 |
| **Pixi** | ≥ 0.48 | 包管理器，自动管理所有编译依赖 |
| **Git** | 任意 | 子模块初始化需要 |

### 1.2 Pixi 安装

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://pixi.sh/install.ps1 | iex"
```

安装后**重启终端**，确保 `pixi` 命令可用。

---

## 2. 编译

### 2.1 标准编译流程

```powershell
cd <项目根目录>

# 1. 安装所有依赖（首次或 pixi.toml 变更后执行）
pixi install

# 2. 初始化 Git 子模块
pixi run initialize

# 3. 配置 Debug 构建
pixi run configure-debug

# 4. 编译
pixi run build-debug

# 5. 安装到 Pixi 环境
pixi run install-debug

# 6. 运行
pixi run freecad-debug
```

### 2.2 构建类型说明

**⚠️ 关键：`conda-windows-debug` preset 的实际构建类型是 `RelWithDebInfo`，不是 `Debug`**

这是因为 Pixi/Conda 环境中没有 `python311_d.lib`（Python 调试版库），如果强制用 `CMAKE_BUILD_TYPE=Debug` 会导致链接失败：

```
LINK : fatal error LNK1104: 无法打开文件"python311_d.lib"
```

| 构建类型 | 编译标志 | 优化 | 调试信息 | 单步跟踪 |
|----------|---------|------|---------|---------|
| `Debug` | `/Zi /Ob0 /Od /RTC1` | 无优化 | ✅ 完整 | ✅ 精确 |
| `RelWithDebInfo` | `/Zi /O2 /Ob1 /DNDEBUG` | 有优化 | ✅ 完整 | ⚠️ 部分函数可能被内联/重排 |

> **结论**：在 Pixi 环境下只能使用 `RelWithDebInfo`，但 PDB 调试信息是完整的，断点和变量查看均可正常工作，只是部分被优化的函数单步跟踪可能不精确。

### 2.3 CMake 生成器冲突

**问题**：如果之前用过 Visual Studio 生成器（如直接用 CMake GUI 或 VS 打开），再切换到 Ninja 会报错：

```
CMake Error: generator : Ninja
Does not match the generator used previously: Visual Studio 17 2022
```

**解决**：删除构建缓存后重新配置：

```powershell
del /q build\debug\CMakeCache.txt
rmdir /s /q build\debug\CMakeFiles
pixi run configure-debug
```

或直接删除整个 `build/debug` 目录。

### 2.4 LibPack 检查失败的规避

**问题**：在 Windows MSVC 下，`InitializeFreeCADBuildOptions.cmake` 默认设置 `FREECAD_LIBPACK_USE=ON`，CMake 会检查项目根目录下是否存在 `plugins/imageformats/qsvg.dll`，找不到就报错：

```
CMake Error: Could not find LibPack in specified location: D:/Workspace/.../FreeCAD_Gitee
```

**解决**：必须使用 `conda-windows-debug` preset，它会设置 `FREECAD_LIBPACK_USE=OFF`。不要用默认的 `debug` preset。

### 2.5 Clang 编译器与 `llvm-ar` 不兼容

**问题**：Pixi 环境在 Windows 上使用 `clang-cl`（Clang 19.x）作为编译器，但 CMake 自动检测到的静态库归档工具是 `llvm-ar.exe`。`llvm-ar` 不支持 MSVC 风格的命令行参数（如 `/` 前缀选项），导致链接静态库时报错：

```
llvm-ar.exe: error: unknown option /
FAILED: src/3rdParty/FastSignals/libfastsignals/libfastsignals.lib
```

**根因**：Clang-cl 在 Windows 上模拟 MSVC 编译器，生成 MSVC 风格的 `.lib` 文件，而 `llvm-ar` 是 GNU 风格的归档工具，两者参数格式不兼容。Windows + clang-cl 应使用 `llvm-lib.exe`（模拟 MSVC `lib.exe` 的工具）。

**解决**：在 `CMakePresets.json` 的 `conda-windows` preset 中添加 `CMAKE_AR`：

```json
{
  "name": "conda-windows",
  "hidden": true,
  "cacheVariables": {
    "CMAKE_AR": {
      "type": "FILEPATH",
      "value": "$env{CONDA_PREFIX}/Library/bin/llvm-lib.exe"
    }
  }
}
```

> **注意**：修改 `CMakePresets.json` 后必须删除 `build/debug` 目录重新配置，因为 CMakeCache 中的 `CMAKE_AR` 只在首次配置时生效。

### 2.6 Clang 的 C++98 兼容性警告（-Wc++98-compat）

**问题**：Clang 编译器默认启用 `-Wc++98-compat` 和 `-Wc++98-compat-pedantic` 警告，会为所有 C++11 及以上特性（如 `constexpr`、`auto`、`override`、`nullptr`、范围 `for` 等）产生大量警告。在以下场景中会导致编译失败：

1. **第三方库 Clipper2** 自身 CMakeLists.txt 中设置了 `-Werror`（将警告视为错误），导致 `-Wc++98-compat` 警告变为编译错误
2. **FreeCAD 自身代码**在 `CompilerChecksAndSetups.cmake` 中设置了 `-Wpedantic`，也会触发这些警告

**错误示例**：

```
clipper.version.h(4,1): error: 'constexpr' specifier is incompatible with C++98 [-Werror,-Wc++98-compat]
clipper.core.h(31,47): error: 'override' keyword is incompatible with C++98 [-Werror,-Wc++98-compat]
clipper.core.h(141,43): error: extra ';' after member function definition [-Werror,-Wextra-semi]
```

**解决**：

1. **全局禁用 C++98 兼容性警告**：在 `cMake/FreeCAD_Helpers/CompilerChecksAndSetups.cmake` 的 Clang 编译标志中添加 `-Wno-c++98-compat -Wno-c++98-compat-pedantic`：

   ```cmake
   if(CMAKE_COMPILER_IS_CLANGXX)
       set(CMAKE_CXX_FLAGS "-Wall -Wextra -Wpedantic -Wno-write-strings -Wno-c++98-compat -Wno-c++98-compat-pedantic ${CMAKE_CXX_FLAGS}")
   ```

2. **移除第三方库的 `-Werror`**：在 `src/3rdParty/Clipper2/CMakeLists.txt` 中，将非 MSVC 分支的 `-Werror` 移除，并添加 `-Wno-c++98-compat`：

   ```cmake
   # 修改前
   target_compile_options(Clipper2 PRIVATE -Wall -Wextra -Wpedantic -Werror)

   # 修改后
   target_compile_options(Clipper2 PRIVATE -Wall -Wextra -Wpedantic -Wno-c++98-compat -Wno-c++98-compat-pedantic)
   ```

   同样修改 `Clipper2Z` 的编译选项。MSVC 分支也应移除 `/WX`（等价于 `-Werror`）。

> **原理**：FreeCAD 要求 C++20 标准，C++98 兼容性警告毫无意义，安全禁用。

### 2.7 VSCode `settings.json` 中变量替换不生效

**问题**：在 `.vscode/settings.json` 中使用 `${workspaceFolder}` 等 VSCode 变量时，CMake Tools 扩展可能不会进行变量替换，导致路径被原样传递给 CMake。

**典型表现**：

- `CMAKE_PREFIX_PATH` 被解析为 `/Library` 而非完整路径
- `CONDA_PREFIX` 环境变量为空，导致 `$env{CONDA_PREFIX}` 在 `CMakePresets.json` 中解析为空字符串
- CMake 命令行中出现 `-DCMAKE_PREFIX_PATH:FILEPATH=/Library` 等异常路径

**根因**：VSCode 的变量替换机制（`${workspaceFolder}`、`${env:PATH}` 等）在 `settings.json` 中的支持因扩展而异。CMake Tools 对 `cmake.environment` 中的变量替换支持不完整。

**解决**：在 `settings.json` 的 `cmake.environment` 和 `cmake.configureSettings` 中，将 `${workspaceFolder}` 替换为**绝对路径**：

```json
{
  "cmake.configureSettings": {
    "CMAKE_PREFIX_PATH": "d:/Workspace/Shisukon/OpenSource/FreeCAD/.pixi/envs/default/Library"
  },
  "cmake.environment": {
    "PATH": "d:/Workspace/Shisukon/OpenSource/FreeCAD/.pixi/envs/default;...",
    "CONDA_PREFIX": "d:/Workspace/Shisukon/OpenSource/FreeCAD/.pixi/envs/default"
  }
}
```

> **注意**：`launch.json` 中 `${workspaceFolder}` 的替换是正常的，因为调试适配器完整支持 VSCode 变量。只有 `settings.json` 中的部分字段可能有问题。

---

## 3. VSCode CMake Tools 集成

### 3.1 必须配置的 settings.json

VSCode CMake Tools 扩展直接调用 CMake 时**不会加载 Pixi 环境**，需要手动配置环境变量和路径。

在 `.vscode/settings.json` 中添加：

```json
{
  // 1. 指定正确的 CMake preset（必须用 conda 系列，不能用默认的 debug）
  "cmake.configurePreset": "conda-windows-debug",

  // 2. 指定 Pixi 环境中的 cmake
  "cmake.cmakePath": "${workspaceFolder}/.pixi/envs/default/Library/bin/cmake.exe",

  // 3. 构建目录
  "cmake.buildDirectory": "${workspaceFolder}/build/debug",

  // 4. 额外 CMake 配置项
  "cmake.configureSettings": {
    "CMAKE_PREFIX_PATH": "${workspaceFolder}/.pixi/envs/default/Library"
  },

  // 5. ⚠️ 关键：设置环境变量，让 moc.exe 等工具找到依赖的 DLL
  "cmake.environment": {
    "PATH": "${workspaceFolder}/.pixi/envs/default;${workspaceFolder}/.pixi/envs/default/Library/mingw-w64/bin;${workspaceFolder}/.pixi/envs/default/Library/usr/bin;${workspaceFolder}/.pixi/envs/default/Library/bin;${workspaceFolder}/.pixi/envs/default/Library/lib/qt6;${workspaceFolder}/.pixi/envs/default/Scripts;${workspaceFolder}/.pixi/envs/default/bin;${env:PATH}",
    "CONDA_PREFIX": "${workspaceFolder}/.pixi/envs/default"
  }
}
```

### 3.2 moc.exe 找不到 DLL 的问题（最常见）

**错误特征**：

```
AUTOMOC for target libfastsignals: Test run of "moc" executable failed.
Exit code 0xc0000135
```

`0xc0000135` 即 `STATUS_DLL_NOT_FOUND`。

**根因分析**：`moc.exe` 位于 `.pixi/envs/default/Library/lib/qt6/`，它依赖以下 DLL：

| DLL | 所在目录 | 用途 |
|-----|---------|------|
| `pcre2-16.dll` | `Library/bin/` | PCRE 正则库 |
| `libstdc++-6.dll` | `Library/mingw-w64/bin/` | MinGW C++ 运行时 |
| `libwinpthread-1.dll` | `Library/mingw-w64/bin/` | MinGW 线程库 |

这些 DLL 都在 Pixi 环境的 PATH 中，但 CMake Tools 直接运行 CMake 时不带 Pixi 环境。

**解决**：在 `cmake.environment.PATH` 中加入所有必要的 Pixi 路径（见上节配置），关键是 **`Library/mingw-w64/bin`** 不要遗漏。

> 可用以下命令验证修复是否生效：
> ```powershell
> set "PATH=<所有pixi路径>;%PATH%"
> .pixi\envs\default\Library\lib\qt6\moc.exe --version
> ```

---

## 4. VSCode 调试

### 4.1 调试器选择

Windows 上必须使用 `cppvsdbg`（MSVC 调试器），不能用 `cppdbg`（GDB/LLDB）。

### 4.2 调试配置（launch.json）

完整的最小可用配置：

```json
{
  "name": "(Windows) Debug FreeCAD",
  "type": "cppvsdbg",
  "request": "launch",
  "cwd": "${workspaceFolder}",
  "program": "${command:cmake.buildDirectory}/bin/FreeCAD.exe",
  "args": [],
  "environment": [
    {
      "name": "PATH",
      "value": "${workspaceFolder}/.pixi/envs/default;${workspaceFolder}/.pixi/envs/default/Library/mingw-w64/bin;${workspaceFolder}/.pixi/envs/default/Library/usr/bin;${workspaceFolder}/.pixi/envs/default/Library/bin;${workspaceFolder}/.pixi/envs/default/Library/lib/qt6;${workspaceFolder}/.pixi/envs/default/Scripts;${workspaceFolder}/.pixi/envs/default/bin;${command:cmake.buildDirectory}/bin;${env:PATH}"
    },
    {
      "name": "CONDA_PREFIX",
      "value": "${workspaceFolder}/.pixi/envs/default"
    },
    {
      "name": "QT_QPA_PLATFORM_PLUGIN_PATH",
      "value": "${workspaceFolder}/.pixi/envs/default/Library/lib/qt6/plugins/platforms"
    },
    {
      "name": "QT_PLUGIN_PATH",
      "value": "${workspaceFolder}/.pixi/envs/default/Library/lib/qt6/plugins"
    }
  ],
  "symbolSearchPath": "${workspaceFolder}/build/debug/bin;${workspaceFolder}/build/debug/Mod/Part;${workspaceFolder}/build/debug/Mod/PartDesign;${workspaceFolder}/build/debug/Mod/Sketcher;${workspaceFolder}/build/debug/Mod/Fem;${workspaceFolder}/build/debug/Mod/Mesh;${workspaceFolder}/build/debug/Mod/MeshPart;${workspaceFolder}/build/debug/Mod/TechDraw;${workspaceFolder}/build/debug/Mod/Spreadsheet;${workspaceFolder}/build/debug/Mod/Draft;${workspaceFolder}/build/debug/Mod/BIM;${workspaceFolder}/build/debug/Mod/CAM;${workspaceFolder}/build/debug/Mod/Assembly;${workspaceFolder}/build/debug/Mod/Material;${workspaceFolder}/build/debug/Mod/Measure;${workspaceFolder}/build/debug/Mod/Points;${workspaceFolder}/build/debug/Mod/Import;${workspaceFolder}/build/debug/Mod/Start;${workspaceFolder}/build/debug/Mod/OpenSCAD;${workspaceFolder}/build/debug/Mod/ReverseEngineering;${workspaceFolder}/build/debug/Mod/Surface;${workspaceFolder}/build/debug/Mod/Help;${workspaceFolder}/build/debug/Mod/Plot;${workspaceFolder}/build/debug/Mod/Robot;${workspaceFolder}/build/debug/Mod/Show;${workspaceFolder}/build/debug/Mod/Web;${workspaceFolder}/build/debug/Mod/Test;${workspaceFolder}/build/debug/Mod/AddonManager",
  "stopAtEntry": false,
  "externalConsole": false,
  "preLaunchTask": "CMake: build"
}
```

### 4.3 环境变量详解

调试配置中的环境变量缺一不可，以下是每个变量的作用：

| 环境变量 | 作用 | 缺失后果 |
|----------|------|---------|
| `PATH`（含 Pixi 路径） | 让 FreeCAD.exe 找到 Qt、OpenCASCADE 等 DLL | 程序启动后立即崩溃/闪退 |
| `CONDA_PREFIX` | 指向 Pixi 环境根目录，部分库运行时依赖此变量 | 某些功能异常 |
| `QT_QPA_PLATFORM_PLUGIN_PATH` | 指向 `qwindows.dll` 所在目录 | 弹窗 "Failed to load platform plugin 'windows'" |
| `QT_PLUGIN_PATH` | 指向 Qt 插件根目录（imageformats 等） | SVG 等格式无法加载 |

### 4.4 Qt 平台插件找不到的问题

**错误特征**：启动后弹出对话框：

```
This application failed to start because no Qt platform plugin could be initialized.
Reinstalling the application may fix this problem.
```

**原因**：`qwindows.dll` 位于 Pixi 环境的非标准路径：

```
.pixi/envs/default/Library/lib/qt6/plugins/platforms/qwindows.dll
```

而非通常的 `Library/bin/platforms/` 或 `Library/plugins/platforms/`。

**解决**：设置环境变量 `QT_QPA_PLATFORM_PLUGIN_PATH` 指向 `qwindows.dll` 所在目录。

### 4.5 PDB 符号文件加载

**问题**：调试时断点不命中（灰色圆圈），或模块窗口中显示"未加载符号"。

**原因分析**：

1. **PDB 与 DLL 不在同一目录**：FreeCAD 的模块 DLL（`.pyd`）在 `build/debug/Mod/*/` 各子目录中，对应的 PDB 也在同一位置。`cppvsdbg` 默认只搜索 EXE 同目录的 PDB。

2. **`symbolSearchPath` 中变量替换可能失效**：`${command:cmake.buildDirectory}` 在 `symbolSearchPath` 字段中可能不被 VSCode 正确解析。

**解决**：

- 在 `launch.json` 中添加 `symbolSearchPath`，列出所有包含 PDB 的目录
- **使用 `${workspaceFolder}/build/debug` 替代 `${command:cmake.buildDirectory}`**，因为前者在 `symbolSearchPath` 中更可靠

PDB 文件分布情况：

```
build/debug/bin/           → FreeCAD.pdb, FreeCADApp.pdb, FreeCADGui.pdb, FreeCADBase.pdb
build/debug/Mod/Part/      → Part.pdb, PartGui.pdb
build/debug/Mod/Sketcher/  → Sketcher.pdb, SketcherGui.pdb
build/debug/Mod/PartDesign/→ PartDesignGui.pdb, _PartDesign.pdb
build/debug/Mod/Fem/       → Fem.pdb, FemGui.pdb
build/debug/Mod/Mesh/      → Mesh.pdb, MeshGui.pdb
... 其他模块类似
```

### 4.6 验证符号加载

调试时可通过以下方式确认 PDB 是否加载：

1. **调试控制台**：输入 `? FreeCADApp` 查看模块符号
2. **模块窗口**：`Debug` → `Windows` → `Modules`，查看各 DLL 的符号状态
3. **输出窗口**：`Debug` → `Windows` → `Output`，筛选 "Debug" 查看符号加载日志

---

## 5. 问题排查清单

当编译或调试遇到问题时，按以下顺序排查：

### 5.1 编译阶段

| # | 检查项 | 命令/方法 |
|---|--------|----------|
| 1 | Pixi 环境是否安装 | `pixi install` |
| 2 | Git 子模块是否初始化 | `pixi run initialize` |
| 3 | CMake preset 是否为 `conda-windows-debug` | 检查 `CMakeCache.txt` 中 `CMAKE_BUILD_TYPE` |
| 4 | `FREECAD_LIBPACK_USE` 是否为 OFF | 检查 `CMakeCache.txt` |
| 5 | CMake 生成器是否为 Ninja | 检查 `CMakeCache.txt` 中 `CMAKE_GENERATOR` |
| 6 | `cmake.environment` PATH 是否包含所有 Pixi 路径 | 检查 `.vscode/settings.json` |
| 7 | `Library/mingw-w64/bin` 是否在 PATH 中 | moc.exe 依赖此目录的 DLL |
| 8 | `CMAKE_AR` 是否为 `llvm-lib.exe` | 检查 `CMakeCache.txt`，见 §2.5 |
| 9 | `.vscode/settings.json` 中变量是否解析 | `${workspaceFolder}` 在 settings.json 中**不支持**替换 |

### 5.2 调试阶段

| # | 检查项 | 命令/方法 |
|---|--------|----------|
| 1 | FreeCAD.exe 路径是否正确 | `build/debug/bin/FreeCAD.exe` |
| 2 | PATH 是否包含 Pixi 环境路径 | 检查 `launch.json` 的 `environment` |
| 3 | `QT_QPA_PLATFORM_PLUGIN_PATH` 是否设置 | 指向 `Library/lib/qt6/plugins/platforms` |
| 4 | `QT_PLUGIN_PATH` 是否设置 | 指向 `Library/lib/qt6/plugins` |
| 5 | `CONDA_PREFIX` 是否设置 | 指向 `.pixi/envs/default` |
| 6 | `symbolSearchPath` 是否包含所有模块目录 | 检查 `launch.json` |
| 7 | PDB 文件是否存在 | `dir build\debug\bin\*.pdb` / `dir build\debug\Mod\*\*.pdb` |
| 8 | 调试器类型是否为 `cppvsdbg` | 不能用 `cppdbg` |

---

## 6. Pixi 环境路径速查

调试和 CMake 配置中需要引用的 Pixi 环境关键路径：

```
.pixi/envs/default/                            # 环境根目录（= CONDA_PREFIX）
.pixi/envs/default/Library/                    # CMAKE_PREFIX_PATH
.pixi/envs/default/Library/bin/                # cmake.exe, pcre2-16.dll, Qt6Core.dll 等
.pixi/envs/default/Library/lib/qt6/            # moc.exe, uic.exe, rcc.exe
.pixi/envs/default/Library/lib/qt6/plugins/    # Qt 插件根目录
.pixi/envs/default/Library/lib/qt6/plugins/platforms/  # qwindows.dll
.pixi/envs/default/Library/lib/qt6/plugins/imageformats/  # qsvg.dll 等
.pixi/envs/default/Library/mingw-w64/bin/      # libstdc++-6.dll, libwinpthread-1.dll（关键！）
.pixi/envs/default/Library/usr/bin/            # Conda 工具链
.pixi/envs/default/Library/include/opencascade/  # OpenCASCADE 头文件
.pixi/envs/default/Library/lib/                # OpenCASCADE 等库文件
.pixi/envs/default/Scripts/                    # Python Scripts
.pixi/envs/default/bin/                        # Conda bin
```

---

## 7. 完整的最小可用配置

### 7.1 .vscode/settings.json

```json
{
  "cmake.configurePreset": "conda-windows-debug",
  "cmake.cmakePath": "${workspaceFolder}/.pixi/envs/default/Library/bin/cmake.exe",
  "cmake.buildDirectory": "${workspaceFolder}/build/debug",
  "cmake.configureSettings": {
    "CMAKE_PREFIX_PATH": "${workspaceFolder}/.pixi/envs/default/Library"
  },
  "cmake.environment": {
    "PATH": "${workspaceFolder}/.pixi/envs/default;${workspaceFolder}/.pixi/envs/default/Library/mingw-w64/bin;${workspaceFolder}/.pixi/envs/default/Library/usr/bin;${workspaceFolder}/.pixi/envs/default/Library/bin;${workspaceFolder}/.pixi/envs/default/Library/lib/qt6;${workspaceFolder}/.pixi/envs/default/Scripts;${workspaceFolder}/.pixi/envs/default/bin;${env:PATH}",
    "CONDA_PREFIX": "${workspaceFolder}/.pixi/envs/default"
  }
}
```

### 7.2 .vscode/launch.json（Windows 部分）

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "(Windows) Debug FreeCAD",
      "type": "cppvsdbg",
      "request": "launch",
      "cwd": "${workspaceFolder}",
      "program": "${command:cmake.buildDirectory}/bin/FreeCAD.exe",
      "args": [],
      "environment": [
        {
          "name": "PATH",
          "value": "${workspaceFolder}/.pixi/envs/default;${workspaceFolder}/.pixi/envs/default/Library/mingw-w64/bin;${workspaceFolder}/.pixi/envs/default/Library/usr/bin;${workspaceFolder}/.pixi/envs/default/Library/bin;${workspaceFolder}/.pixi/envs/default/Library/lib/qt6;${workspaceFolder}/.pixi/envs/default/Scripts;${workspaceFolder}/.pixi/envs/default/bin;${command:cmake.buildDirectory}/bin;${env:PATH}"
        },
        {
          "name": "CONDA_PREFIX",
          "value": "${workspaceFolder}/.pixi/envs/default"
        },
        {
          "name": "QT_QPA_PLATFORM_PLUGIN_PATH",
          "value": "${workspaceFolder}/.pixi/envs/default/Library/lib/qt6/plugins/platforms"
        },
        {
          "name": "QT_PLUGIN_PATH",
          "value": "${workspaceFolder}/.pixi/envs/default/Library/lib/qt6/plugins"
        }
      ],
      "symbolSearchPath": "${workspaceFolder}/build/debug/bin;${workspaceFolder}/build/debug/Mod/Part;${workspaceFolder}/build/debug/Mod/PartDesign;${workspaceFolder}/build/debug/Mod/Sketcher;${workspaceFolder}/build/debug/Mod/Fem;${workspaceFolder}/build/debug/Mod/Mesh;${workspaceFolder}/build/debug/Mod/MeshPart;${workspaceFolder}/build/debug/Mod/TechDraw;${workspaceFolder}/build/debug/Mod/Spreadsheet;${workspaceFolder}/build/debug/Mod/Draft;${workspaceFolder}/build/debug/Mod/BIM;${workspaceFolder}/build/debug/Mod/CAM;${workspaceFolder}/build/debug/Mod/Assembly;${workspaceFolder}/build/debug/Mod/Material;${workspaceFolder}/build/debug/Mod/Measure;${workspaceFolder}/build/debug/Mod/Points;${workspaceFolder}/build/debug/Mod/Import;${workspaceFolder}/build/debug/Mod/Start;${workspaceFolder}/build/debug/Mod/OpenSCAD;${workspaceFolder}/build/debug/Mod/ReverseEngineering;${workspaceFolder}/build/debug/Mod/Surface;${workspaceFolder}/build/debug/Mod/Help;${workspaceFolder}/build/debug/Mod/Plot;${workspaceFolder}/build/debug/Mod/Robot;${workspaceFolder}/build/debug/Mod/Show;${workspaceFolder}/build/debug/Mod/Web;${workspaceFolder}/build/debug/Mod/Test;${workspaceFolder}/build/debug/Mod/AddonManager",
      "stopAtEntry": false,
      "externalConsole": false,
      "preLaunchTask": "CMake: build"
    }
  ]
}
```

---

## 8. 常见错误速查表

| 错误信息 | 原因 | 解决 |
|----------|------|------|
| `Exit code 0xc0000135` (moc.exe) | Pixi 环境 PATH 未设置 | 配置 `cmake.environment` |
| `Could not find LibPack` | 使用了 `debug` preset 而非 `conda-windows-debug` | 改用 `conda-windows-debug` preset |
| `generator Ninja does not match Visual Studio` | 缓存中生成器冲突 | 删除 `CMakeCache.txt` 和 `CMakeFiles` |
| `Failed to load platform plugin 'windows'` | Qt 平台插件路径未设置 | 设置 `QT_QPA_PLATFORM_PLUGIN_PATH` |
| `LNK1104: 无法打开文件 python311_d.lib` | 强制使用 `CMAKE_BUILD_TYPE=Debug` | 使用默认的 `RelWithDebInfo` |
| 调试时断点不命中（灰色圆圈） | PDB 符号未加载 | 配置 `symbolSearchPath` |
| FreeCAD 启动后立即退出 | DLL 路径不在 PATH 中 | 在 `environment` 中添加 Pixi PATH |
| CMake Policy 版本错误 | Coin3D 等第三方 CMake 配置过旧 | 添加 `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` |
| `llvm-ar.exe: error: unknown option /` | clang-cl 环境下使用了 GNU 风格的归档工具 | 设置 `CMAKE_AR=llvm-lib.exe`，见 §2.5 |
| `'constexpr' specifier is incompatible with C++98` | Clang 的 `-Wc++98-compat` 警告被 `-Werror` 提升为错误 | 全局添加 `-Wno-c++98-compat`，移除第三方库的 `-Werror`，见 §2.6 |
| `CMAKE_PREFIX_PATH` 解析为 `/Library` | `settings.json` 中 `${workspaceFolder}` 未被替换 | 使用绝对路径替代变量，见 §2.7 |
| `mixes Clang and MSVC compiler` | CMake 同时检测到 Clang 和 MSVC | 确保 `CONDA_PREFIX` 正确设置，或显式指定 `CMAKE_C_COMPILER`/`CMAKE_CXX_COMPILER` |

---

## 9. 版本信息参考

本次验证所用的具体版本：

| 组件 | 版本 |
|------|------|
| Windows | 11 (Build 26200) |
| Visual Studio | 2022 Community (MSVC 19.44.35219.0) |
| Clang (Pixi) | 19.1.7 (clang-cl) |
| Windows SDK | 10.0.26100.0 |
| CMake | 4.2.3 |
| Ninja | (Pixi 管理) |
| Pixi | 0.69.0 |
| Python | 3.11.14 |
| Qt | 6.8.3 |
| OpenCASCADE | 7.8.1 |
| PySide6 | 6.8.3 |
| Shiboken6 | 6.8.3 |
| Coin3D | 4.0.3 |
| SMESH | 9.8.0.2 |
| VTK | 9.3.0 |
| Boost | (conda-forge) |
| Eigen3 | 3.4.0 |

# FreeCAD 编译指南与注意事项

> 本文档基于 FreeCAD 1.2.0-dev 版本，适用于 Windows 平台（使用 MSVC 编译器）。

---

## 1. 编译前准备

### 1.1 必需工具

| 工具 | 最低版本要求 | 说明 |
|------|-------------|------|
| **CMake** | ≥ 3.22.0 | 构建系统，低于此版本会报 FATAL_ERROR |
| **C++ 编译器** | MSVC (VS 2019+) | 支持 C++20 标准 |
| **Git** | 任意 | 用于克隆子模块 |
| **Python** | ≥ 3.11, < 3.12 | FreeCAD 的脚本引擎 |

> **注意**: FreeCAD 1.1 及以后版本要求 C++20 支持。
> - GCC ≥ 11.2
> - Clang ≥ 14.0
> - MSVC 需支持 `/std:c++20`

### 1.2 依赖管理方式（二选一）

#### 方式 A：使用 Pixi（推荐）

Pixi 是基于 Conda 的包管理器，可自动安装所有依赖。

```powershell
# 安装 pixi（如尚未安装）
powershell -ExecutionPolicy ByPass -c "irm https://pixi.sh/install.ps1 | iex"

# 在项目根目录下安装所有依赖
pixi install

# 初始化 Git 子模块
pixi run initialize
```

#### 方式 B：使用 Windows LibPack

Windows 上 MSVC 默认启用 `FREECAD_LIBPACK_USE=ON`，需要下载预编译的依赖包：

1. 从 [FreeCAD-Libpack Releases](https://github.com/FreeCAD/FreeCAD-Libpack/releases/) 下载对应版本的 LibPack
2. 将 LibPack 解压到项目根目录（或通过环境变量 `FREECAD_LIBPACK_DIR` 指定路径）
3. CMake 会自动检测 `plugins/imageformats/qsvg.dll` 来确认 LibPack 是否存在

> **重要**: 如果未找到 LibPack，CMake 会输出 WARNING 但不会中断。后续配置阶段会因缺少依赖而失败。

### 1.3 关键依赖列表

以下为 FreeCAD 编译所需的核心第三方库：

| 依赖 | 版本要求 | 说明 |
|------|---------|------|
| **OpenCASCADE (OCCT)** | ≥ 7.8, < 7.9 | 几何内核，最核心的依赖 |
| **Qt** | Qt 6（≥ 6.8, < 6.9）| GUI 框架，默认使用 Qt 6 |
| **Boost** | ≥ 1.74 | 通用 C++ 库 |
| **Eigen3** | ≥ 3.3, < 5 | 线性代数库 |
| **Xerces-C** | - | XML 解析 |
| **ZLIB** | - | 压缩库 |
| **ICU** | uc + i18n | 国际化支持（**REQUIRED**） |
| **Coin3D** | - | Open Inventor 3D 场景管理 |
| **PySide6 / Shiboken6** | - | Qt Python 绑定（BUILD_GUI 时需要） |
| **Pivy** | - | Coin3D Python 绑定 |
| **SWIG** | - | Python C 绑定生成 |
| **Freetype** | - | 字体渲染（FREECAD_USE_FREETYPE 默认 ON） |
| **SMESH** | - | 网格生成（FEM/Mesh 模块需要） |
| **VTK** | - | 可视化工具包 |
| **HDF5** | - | 数据格式 |
| **fmt** | - | 格式化库（默认使用外部版本） |
| **yaml-cpp** | - | YAML 解析 |
| **pybind11** | - | CAM/FlatMesh 模块需要 |
| **PCL** | - | 点云库（可选，默认 OFF） |

---

## 2. 配置与编译

### 2.1 使用 Pixi 编译（推荐方式）

```powershell
# 1. 安装依赖
pixi install

# 2. 初始化子模块
pixi run initialize

# 3. 配置（Debug 模式）
pixi run configure-debug

# 4. 编译
pixi run build-debug

# 5. 安装
pixi run install-debug

# 6. 运行测试
pixi run test-debug
```

Release 模式替换为 `configure-release` / `build-release` / `install-release` / `test-release`。

### 2.2 使用 CMake 手动编译

#### Windows + MSVC + LibPack

```powershell
# 1. 配置
cmake -B build -G "Visual Studio 17 2022" -A x64 ^
    -DFREECAD_LIBPACK_USE=ON ^
    -DFREECAD_LIBPACK_DIR="C:/path/to/LibPack" ^
    -DBUILD_GUI=ON ^
    -DFREECAD_QT_VERSION=6

# 2. 编译
cmake --build build --config Release

# 3. 安装
cmake --install build --prefix install
```

#### Windows + Pixi + CMake Preset

```powershell
# 使用预定义的 CMake Preset（推荐 Ninja 生成器）
cmake --preset conda-windows-debug
cmake --build build/debug
```

#### Linux / macOS

```bash
# 使用 Pixi Preset
pixi run configure-debug     # Linux/macOS 自动选择对应 preset
pixi run build-debug
```

或手动：

```bash
cmake --preset conda-linux-debug    # Linux
cmake --preset conda-macos-debug    # macOS
cmake --build build/debug
```

### 2.3 CMake Preset 一览

| Preset 名称 | 平台 | 模式 | 生成器 |
|-------------|------|------|--------|
| `conda-linux-debug` | Linux x86_64 | Debug | Ninja |
| `conda-linux-release` | Linux x86_64 | Release | Ninja |
| `conda-macos-debug` | macOS | Debug | Ninja |
| `conda-macos-release` | macOS | Release | Ninja |
| `conda-windows-debug` | Windows | RelWithDebInfo | Ninja |
| `conda-windows-release` | Windows | Release | Ninja |

---

## 3. 重要编译选项说明

### 3.1 核心选项

| 选项 | 默认值 | 说明 |
|------|-------|------|
| `BUILD_GUI` | ON | 是否构建 GUI，OFF 则只构建命令行和 Python 模块 |
| `FREECAD_LIBPACK_USE` | ON (MSVC) | Windows 下是否使用 LibPack |
| `FREECAD_USE_PCH` | ON (MSVC) | 启用预编译头，加速编译 |
| `FREECAD_USE_CCACHE` | ON | 自动检测并使用 ccache |
| `FREECAD_USE_EXTERNAL_SMESH` | OFF | 使用系统 SMESH 而非内嵌版本 |
| `FREECAD_USE_EXTERNAL_FMT` | ON | 使用系统 fmt 库 |
| `FREECAD_USE_FREETYPE` | ON | 启用 FreeType 字体功能 |
| `BUILD_FEM` | ON | 构建 FEM 模块 |
| `BUILD_FEM_NETGEN` | ON (MSVC) / OFF (其他) | FEM 模块使用 NETGEN 网格器 |
| `ENABLE_DEVELOPER_TESTS` | ON | 构建单元测试 |
| `FREECAD_QT_VERSION` | 自动检测 | Qt 版本（5 或 6） |
| `OCCT_CMAKE_FALLBACK` | OFF | 禁用 occt-config 文件，改用 CMake 查找 |

### 3.2 内存优化选项

编译 FreeCAD 非常消耗内存，以下选项可缓解：

| 选项 | 说明 |
|------|------|
| `FREECAD_PARALLEL_COMPILE_JOBS` | 限制并行编译任务数（仅 Ninja 生成器有效） |
| `FREECAD_PARALLEL_LINK_JOBS` | 限制并行链接任务数（仅 Ninja 生成器有效） |

CMake 会自动根据物理内存计算最大编译进程数（1 GiB/进程），并限制链接任务为 1。

### 3.3 Sanitizer 选项（仅 GCC/Clang）

| 选项 | 说明 |
|------|------|
| `FREECAD_USE_SANITIZER_ASAN` | AddressSanitizer |
| `FREECAD_USE_SANITIZER_LSAN` | LeakSanitizer |
| `FREECAD_USE_SANITIZER_TSAN` | ThreadSanitizer |
| `FREECAD_USE_SANITIZER_UBSAN` | UndefinedBehaviorSanitizer |
| `FREECAD_USE_SANITIZER_MSAN` | MemorySanitizer |

### 3.4 可选构建模块

大部分模块默认 ON，以下默认 OFF：

| 模块 | 选项 | 说明 |
|------|------|------|
| Sandbox | `BUILD_SANDBOX` | 仅测试用途 |
| Template | `BUILD_TEMPLATE` | 仅测试用途 |
| Drawing | `BUILD_DRAWING` | 旧版工程图模块 |
| JTReader | `BUILD_JTREADER` | JT 文件读取 |
| VR | `BUILD_VR` | Oculus Rift 支持 |
| Cloud | `BUILD_CLOUD` | 云模块 |

---

## 4. 注意事项与常见问题

### 4.1 Windows 特有注意事项

1. **LibPack 版本匹配**: 必须使用与 FreeCAD 版本匹配的 LibPack，否则会因 ABI 不兼容导致运行时错误。
2. **PDB 文件**: Release 版默认生成 PDB 调试符号文件（`FREECAD_RELEASE_PDB=ON`），如果不需要可关闭以减小体积。
3. **结构化异常处理**: Release 版默认启用 SEH（`FREECAD_RELEASE_SEH=ON`）。
4. **环境变量 `FREECAD_LIBPACK_DIR`**: 可通过此环境变量指定 LibPack 路径，避免每次手动配置。
5. **Windows Conda preset 的 Debug 模式实际为 RelWithDebInfo**: 这是因为 Windows 上纯 Debug 模式的依赖库通常不可用。

### 4.2 macOS 特有注意事项

1. **Homebrew 路径冲突**: CMake Preset 已配置 `CMAKE_IGNORE_PREFIX_PATH` 忽略 `/opt/homebrew` 和 `/usr/local/homebrew`，避免与 Conda 冲突。
2. **OpenGL 弃用**: macOS 上自动定义 `GL_SILENCE_DEPRECATION`。
3. **App Bundle**: 可通过 `FREECAD_CREATE_MAC_APP=ON` 创建 macOS App Bundle。

### 4.3 Linux 特有注意事项

1. **编译器**: Conda preset 默认使用 Clang + Mold 链接器。
2. **xcb-util-cursor**: 需要精确版本 0.1.5 以解决 [Issue #26726](https://github.com/FreeCAD/FreeCAD/issues/26726)。

### 4.4 通用注意事项

1. **构建目录必须与源码目录分离**: 启用 `BUILD_FORCE_DIRECTORY` 时会强制检查。
2. **Git 子模块**: 编译前必须执行 `git submodule update --init --recursive`，否则部分内嵌依赖（如 SMESH）会缺失。
3. **Python 版本**: 严格限制为 3.11.x，其他版本可能导致不兼容。
4. **Qt 版本**: 默认使用 Qt 6，Qt 5 虽然支持但不推荐。
5. **ICU 是必需依赖**: `find_package(ICU REQUIRED COMPONENTS uc i18n)` 必须成功。
6. **Boost 最低版本**: 1.74，可通过 `BOOST_MIN_VERSION` 变量覆盖。
7. **Shiboken2 兼容性**: CMake Policy CMP0148 保持 OLD，因为 Shiboken2 仍在使用已弃用的 FindPython 模块。
8. **编译时间**: FreeCAD 是大型项目，完整编译可能需要 30 分钟到数小时，取决于硬件。建议使用 Ninja + ccache 加速。
9. **磁盘空间**: 编译产物和中间文件可能占用 10-20 GB 磁盘空间。

### 4.5 常见编译错误及解决

| 错误 | 原因 | 解决方法 |
|------|------|---------|
| `CMake 3.22.0 required` | CMake 版本太低 | 升级 CMake |
| `G++ must be 11.2 or later` | GCC 版本太低 | 升级 GCC 或改用 Clang |
| `Could not find LibPack` | Windows 上未安装 LibPack | 下载并放置 LibPack，或设置 `FREECAD_LIBPACK_DIR` |
| `ICU not found` | 缺少 ICU 库 | 安装 ICU 或使用 Pixi/Conda |
| `FindBoost module is removed` | CMake 3.30+ 弃用了 FindBoost | Policy CMP0167 已设为 NEW，会使用 BoostConfig |
| SWIG not found | LibPack 模式下 SWIG 缺失 | 不影响编译，只是无法构建 Pivy 的 SWIG 绑定 |
| 链接时内存不足 | 并行链接任务过多 | 设置 `FREECAD_PARALLEL_LINK_JOBS=1` |

---

## 5. 快速开始（Windows 推荐）

```powershell
# 1. 安装 pixi
powershell -ExecutionPolicy ByPass -c "irm https://pixi.sh/install.ps1 | iex"

# 2. 进入项目目录
cd d:\Workspace\Shisukon\OpenSource\FreeCAD_Gitee

# 3. 安装依赖并初始化子模块
pixi install
pixi run initialize

# 4. 配置（Windows Debug 模式）
pixi run configure-debug

# 5. 编译
pixi run build-debug

# 6. 安装到 Conda 环境
pixi run install-debug

# 7. 运行（Windows）
pixi run freecad-debug
```

---

## 6. 参考链接

- [FreeCAD Developers Handbook – Getting Started](https://freecad.github.io/DevelopersHandbook/gettingstarted/)
- [FreeCAD Wiki - Compile on Windows](https://wiki.freecad.org/Compile_on_Windows)
- [FreeCAD-Libpack Releases](https://github.com/FreeCAD/FreeCAD-Libpack/releases/)
- [Pixi 官方文档](https://pixi.sh/latest/)

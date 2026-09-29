# Winlator 运行架构与完整启动命令详解

> 本文所有结论均来自对源仓库代码的逐行核实，不是转述或猜测。
> 核实时间：2026-09-29。覆盖范围：`brunodev85/winlator` 主线仓库及其 `winlator-app` 子模块，抽查版本 v6.1.0、v11.0.0、v11.1.0、v11.2.0、main 分支。

---

## 0. 结论摘要（TL;DR）

- **Winlator = Wine + Box86/Box64 + 一个 Linux 用户态（rootfs）**。所谓"Linux 容器"就是一棵随 APK 打包/安装的 rootfs 目录树加一组环境变量，**不是虚拟机，也不是 Docker**：它没有独立内核，共享 Android 内核，进程就是普通 Android 进程。
- **存在两代架构**：
  - **v11.0.0 及更早（示例 v6.1.0）：proot 方案**。用 ptrace 在用户态模拟 `chroot`/`mount`，把 rootfs 伪装成根文件系统，再在容器里执行 `box64 → wine`。
  - **v11.1.0 及之后（含最新的 v11.2.0 与 main）：去掉 proot**。改成直接执行 `<rootfs>/usr/local/bin/box64`，通过 `LD_LIBRARY_PATH`/`BOX64_LD_LIBRARY_PATH` 指进 rootfs 加载 glibc，X server、音频、SysV 共享内存等服务全部由 App 进程内组件通过 Unix socket 提供。
- **当前版完整启动命令的核心一行**（其余是环境变量）：
  ```bash
  <rootfs>/usr/local/bin/box64 wine explorer /desktop=nogui,1280x720 \
    C:\windows\winhandler.exe /dir <游戏目录> "<游戏.exe>" [附加参数]
  ```
- **图形链路**：游戏调用 Direct3D → Wine 层（wined3d / DXVK / VKD3D）→ Vulkan 或 OpenGL → Mesa（Turnip 适配高通 Adreno，Zink 适配 Mali，VirGL 用于虚拟化路径）→ GPU。

---

## 1. 背景：为什么 App 里要"嵌"一个 Linux 环境

Windows 程序不会在 Linux 上直接运行，Winlator 的思路是三层翻译：

1. **Wine（兼容层）**：把 Win32 API（`CreateWindow`、`LoadLibrary`、`ReadFile` 等）翻译成 Linux 系统调用，并提供 `C:\` 盘语义（wine prefix，`drive_c`）。
2. **Box86 / Box64（指令层）**：在 ARM 设备上把 x86 / x86_64 指令实时动态翻译成 ARM 指令执行（在 x86 设备上不需要这一层）。
3. **Linux 用户态（环境层）**：Wine 和被翻译的 Windows 程序都依赖**标准 glibc 用户态**（`/lib`、`/usr`、`/etc`、`/etc/ld.so.cache` 等），而 Android 系统本身用的是 **Bionic libc**，目录布局也不同，天然不是 glibc 程序的宿主。

问题在于：普通 Android 应用**没有 root 权限**，不能真正执行 `chroot` / `mount`。于是出现了两种解决思路——也就是下面两代架构。

---

## 2. 两代架构总览

| 维度 | 老架构（proot 时代） | 新架构（XEnvironment 时代） |
|---|---|---|
| 对应版本 | v11.0.0 及更早（示例 v6.1.0） | v11.1.0、v11.2.0、main |
| "容器"实现 | `libproot.so`，用户态模拟 chroot/mount | 纯目录树（rootfs）+ 环境变量，无需模拟 |
| 启动命令形态 | `libproot.so --kill-on-exit --rootfs=... --bind=... /usr/bin/env <env> box64 wine ...` | `<rootfs>/usr/local/bin/box64 wine explorer ...` |
| 系统服务提供方式 | proot 挂载 `/dev`、`/proc`、`/sys` + 部分 App 内组件 | 全部 App 内组件走 Unix socket |
| 容器数据位置 | rootfs 内 wine prefix + 各盘符目录 bind 进容器 | rootfs 内 wine prefix（见 5.5 节） |
| 需要 ptrace | 是（性能开销） | 否 |

---

## 3. 老架构详解（以 v6.1.0 为例）

### 3.1 容器是什么

APK 内置一个 glibc rootfs 镜像（源码 `app/src/main/cpp/proot/` 自带完整 proot C 源码），外面包一层 `libproot.so`（编译后放 App native 目录）。`libproot.so` 用 **ptrace 拦截被跟踪进程的系统调用**，在用户态模拟 `chroot`、`mount`、假 root 等行为，让 rootfs 目录树"看起来"是完整 Linux 根目录。

### 3.2 完整启动命令（源码逐段拼接）

源码：`winlator` v6.1.0 `app/src/main/java/com/winlator/xenvironment/components/GuestProgramLauncherComponent.java` 第 123–150 行。

```bash
<App native 目录>/libproot.so \
  --kill-on-exit \
  --rootfs=<rootfs> \
  --cwd=/home/xuser \
  --bind=/dev \
  [--bind=<rootfs>/tmp/shm:/dev/shm]        # 启用 SysV 共享内存模拟时
  --bind=/proc \
  --bind=/sys \
  [--bind=<用户自定义盘符目录>...]           # 每个容器盘符映射一个 --bind
  /usr/bin/env <环境变量...> \
  box64 wine explorer /desktop=shell,<分辨率> \
    C:\windows\winhandler.exe /dir <DOS目录> "<程序.exe>" [附加参数]
```

参数含义：

| 参数 | 含义 |
|---|---|
| `--kill-on-exit` | proot 退出时结束被跟踪进程组 |
| `--rootfs=<rootfs>` | 指定 chroot 根目录 |
| `--cwd=/home/xuser` | 进入容器后的初始工作目录 |
| `--bind=<host>:<guest>` | 把宿主路径挂载到容器内路径（`/dev`、`/proc`、`/sys` 必须提供，用户盘符目录来自容器的驱动器配置） |
| `/usr/bin/env <环境变量> box64 ...` | 在容器内设置环境变量并启动 box64 |
| `box64 wine explorer /desktop=shell,...` | 以 box64 解释执行 wine（x86 程序）；`wineInfo.binName()` 视 wine 是 32/64 位返回 `wine` 或 `wine64` |

### 3.3 proot 自身需要的环境变量

```bash
PROOT_TMP_DIR=<临时目录>
PROOT_LOADER=<native>/libproot-loader.so
PROOT_LOADER_32=<native>/libproot-loader32.so
```

### 3.4 盘符 bind 来源

v6.1.0 `XServerDisplayActivity.java` 第 374–376 行：遍历 `container.drivesIterator()`，把每个容器盘符对应的宿主目录（如 `D:` 映射的存储目录）逐个追加到 `--bind=` 列表——这正是"容器里能看到 Android 存储"的方式。

### 3.5 老架构的组件清单（v6.1.0 XServerDisplayActivity 第 382–405 行）

```text
SysVSharedMemoryComponent   （SysV 共享内存模拟，socket /tmp/.sysvshm/SM0）
XServerComponent            （X11 server，socket /tmp/.X11-unix/X0）
EtcHostsFileUpdateComponent
ALSAServerComponent 或 PulseAudioComponent  （按声卡驱动二选一）
VirGLRendererComponent      （虚拟 GPU 渲染，socket /tmp/.virgl/V0）
GuestProgramLauncherComponent（最终启动 box64 → wine 的组件）
```

---

## 4. 新架构详解（v11.1.0 之后）

### 4.1 容器是什么

源码 `winlator-app` `RootFS.java`：rootfs 位于**应用私有目录** `files/rootfs`（旧版叫 `imagefs`，检测到会自动改名迁移）。容器概念由 `XEnvironment`（环境）+ 若干 `EnvironmentComponent`（服务组件）组成，随 `startEnvironmentComponents()` 启动、`stopEnvironmentComponents()` 停止。

### 4.2 完整启动命令（源码逐段拼接）

源码：`winlator-app` `app/src/main/java/com/winlator/xenvironment/components/GuestProgramLauncherComponent.java` 第 86–116 行（`execGuestProgram()`）＋ `XServerDisplayActivity.java` 第 497–530 行（环境变量）+ 第 955–990 行（guest 命令参数）。

```bash
# 工作目录（cwd） = <rootfs>；stdout/stderr → /dev/null（调试日志开启时改为输出到日志）
<rootfs>/usr/local/bin/box64 \
  wine explorer /desktop={nogui|shell},<宽x高> \
  C:\windows\winhandler.exe /dir <DOS目录> "<程序>.exe" [附加参数]
```

各部分说明：

- `box64` 本体从 APK `assets/box64/box64-0.4.4.tzst` 解压到 `<rootfs>/usr/local/bin/box64`（`GeneralComponents.extractFile` 的 BOX64 目的地即 rootfs 根目录；版本可在容器设置里切换）。
- `wine` 通过 PATH 查找：`<rootfs>/opt/wine/bin/wine`（`RootFS.winePath` 默认 `/opt/wine`，可安装自定义 wine 到 `/opt/installed-wine`）。
- `/desktop=nogui,<分辨率>`：从**快捷方式/文件管理器**启动游戏时用 `nogui`，直接进 Wine 桌面时用 `shell`。
- `C:\windows\winhandler.exe`：Winlator 自带的 Windows 侧启动器（最终拉起并管理目标程序）。
- 未指定程序时兜底：`/dir C:\windows "wfm.exe"`（wine 文件管理器）。
- 快捷方式/自定义执行参数里若带 `EXTRA_EXEC_ARGS` 环境变量，会追加到 `winhandler.exe` 参数末尾。

### 4.3 完整环境变量表

进程环境 = 继承的 App 自身环境 + 下表覆盖（由 `ProcessHelper.exec` 用 `ProcessBuilder` 注入，working dir = rootfs）：

| 环境变量 | 值 | 说明 |
|---|---|---|
| `HOME` | `<rootfs>/home/xuser` | 容器内家目录 |
| `USER` | `xuser` | 容器内用户名 |
| `TMPDIR` | `<rootfs>/tmp` | |
| `DISPLAY` | `:0` | X server（App 内 XServerComponent） |
| `PATH` | `<rootfs>/opt/wine/bin:<rootfs>/usr/local/bin:<rootfs>/usr/bin` | 让 box64 按名字找到 wine |
| `LD_LIBRARY_PATH` | `<rootfs>/usr/lib` | 64 位 glibc 库 |
| `BOX64_LD_LIBRARY_PATH` | `<rootfs>/lib/x86_64-linux-gnu` | 32 位库（box64 专用） |
| `WINEPREFIX` | `<rootfs>/home/xuser/.wine` | wine prefix（C: 盘，见 4.5 节链接机制） |
| `WINE_DO_NOT_CREATE_DXGI_DEVICE_MANAGER` | `1` | |
| `WINEDEBUG` | `-all`（调试时如 `+d3d,+wined3d`） | |
| `WINEESYNC` | `1`（用户未覆盖时） | Esync |
| `MESA_DEBUG` / `MESA_NO_ERROR` | `silent` / `1` | |
| `BOX64_NOBANNER` | `0`/`1` | 随日志等级 |
| `BOX64_DYNAREC` / `BOX64_UNITYPLAYER` / `BOX64_DYNACACHE` | `1` / `0` / `0` | |
| `BOX64_RCFILE` | `<rootfs>/etc/config.box64rc` | 由 assets 的 default.box64rc 生成 |
| `ANDROID_SYSVSHM_SERVER` | `<rootfs>/tmp/.sysvshm/SM0` | SysV 共享内存 socket |
| `ANDROID_ALSA_SERVER`（ALSA 声卡） | `<rootfs>/tmp/.sound/AS0` | |
| `PULSE_SERVER`（PulseAudio 声卡） | `<rootfs>/tmp/.sound/PS0` | |

此外还会叠加：容器自定义环境变量、Box64 预设（Compatibility/Intermediate/Stability/Performance 等）的变量、快捷方式的 `envVars`。

### 4.4 为什么新架构能去掉 proot

1. box64 本质是个"用户态翻译器"，它运行的就是**本机 Linux 内核**上的进程，不需要虚拟化内核——需要的是 glibc 用户态和正确的库搜索路径，这两者分别由 `LD_LIBRARY_PATH`/`BOX64_LD_LIBRARY_PATH` 和目录布局解决；
2. 原先 proot 模拟的 `/dev`、`/proc`、`/sys` 挂载，在图形类用途里真正起作用的只有设备节点与共享内存等极少数能力，新架构单独用组件补上了最关键的几项（SysV 共享内存、X11、音频、ethernet 网络信息更新）；
3. 省掉 ptrace 之后启动更快、无跟踪开销，也少了 proot 对部分系统调用的干扰。

### 4.5 容器数据布局（v11.1+）

源码：`ContainerManager.java` + `Container.java`。

```text
<rootfs>/
├── .winlator/.rfs_version          # rootfs 版本标记
├── home/
│   ├── xuser                       # 活跃容器的 symlink → xuser-<id>
│   └── xuser-<id>/                 # 每个容器一个目录
│       ├── .container              # 容器配置（JSON）
│       └── .wine/                  # 该容器的 wine prefix（drive_c 等）
│           ├── drive_c/
│           └── users/xuser/        # 桌面、文档、开始菜单等
├── opt/
│   ├── wine/                       # 默认 wine（PATH 指向其 bin）
│   └── installed-wine/             # 用户安装的其他 wine 版本
├── usr/local/bin/box64            # box64
├── usr/lib                        # 64 位 glibc
└── etc/config.box64rc            # box64 配置文件
```

关键机制：`activateContainer()` 把 `<rootfs>/home/xuser` 做成指向 `xuser-<id>` 的符号链接，因此固定路径的 `WINEPREFIX=<rootfs>/home/xuser/.wine` 会跟随"当前激活的容器"切换。

### 4.6 新架构的组件清单与 socket 表（v11 XServerDisplayActivity 第 525–570 行）

| 组件 | 用途 | Unix socket（均为 `<rootfs>` 下） |
|---|---|---|
| SysVSharedMemoryComponent | 模拟 System V 共享内存 | `/tmp/.sysvshm/SM0` |
| XServerComponent | 容器内 X11 server（全局 X server，供 DISPLAY=:0） | `/tmp/.X11-unix/X0` |
| NetworkInfoUpdateComponent | 写入网络信息（替代老版 EtcHosts 更新） | — |
| ALSAServerComponent / PulseAudioComponent | 音频后端（按声卡驱动二选一） | `/tmp/.sound/AS0` / `/tmp/.sound/PS0` |
| VortekRendererComponent / VirGLRendererComponent | GPU 渲染后端（按图形驱动选择） | `/tmp/.vortek/V0` / `/tmp/.virgl/V0` |
| GuestProgramLauncherComponent | 启动 box64 → wine → 程序 | — |

图形驱动与 DirectX 转译（APK assets 里的组件，均可在容器设置里切换）：

- `graphics_driver/`：`turnip-26.2.0`（高通 Adreno 的 Vulkan 驱动）、`virgl-23.1.9`（虚拟 GPU）、`zink-22.2.5`（OpenGL on Vulkan）、`vortek-2.1`、`gladio-1.1`；
- `dxwrapper/`：`dxvk-2.4.1`（D3D11→Vulkan）、`vkd3d-2.14.1`（D3D12→Vulkan）、`d7vk-1.11`（D3D7）、`d8vk-1.0`（D3D8）、`cnc-ddraw-6.6`（老游戏 DDraw 加速），以及 wined3d（Wine 自带，转 OpenGL）。

因此典型的图形数据流是：

```text
游戏调 Direct3D
  → 选 DXVK/VKD3D：D3D → Vulkan → Mesa(Turnip/Zink 等) → GPU 驱动
  → 或用 wined3d：D3D → OpenGL → Mesa(Zink/VirGL 路径) → GPU
音频：Win32 MME/DirectSound → Wine 音频层 → PulseAudio/ALSA 服务组件（socket）→ App 侧音频通道
```

---

## 5. 版本差异核实记录（逐标签）

对 `brunodev85/winlator` 的每个版本标签做了文件树与启动组件源码检查：

| 版本 | 主仓库是否含 proot | app 是否为子模块 | 启动命令形态（源码证据） |
|---|---|---|---|
| v6.1.0 | ✅ 有（`app/src/main/cpp/proot/` 完整源码） | 否 | `libproot.so --kill-on-exit --rootfs=... /usr/bin/env ... box64 ...`（GuestProgramLauncherComponent 第 123–155 行） |
| v8.0.0 | ✅ 有 | 否 | proot（同上） |
| v9.0.0 | ✅ 有 | 否 | proot |
| v10.0.0 / v10.1.0 | ✅ 有 | 否 | proot |
| **v11.0.0** | ✅ 有 | 否 | **仍是 proot**：`String command = nativeLibraryDir+"/libproot.so";`（该文件第 136 行），并设置 `PROOT_LOADER`/`PROOT_LOADER_32`（第 159–160 行） |
| **v11.1.0** | ❌ 无 | ✅ 是（commit `86b6004`） | `String command = rootDir+"/usr/local/bin/box64 "+guestExecutable;`（该文件第 108 行） |
| v11.2.0 | ❌ 无 | ✅ 是（commit `c2f4ad4`） | box64 直跑，与 main 完全一致（`diff` 比对通过） |
| main | ❌ 无 | ✅ 是 | box64 直跑 |

**结论：proot 的移除发生在 v11.0 → v11.1 之间；v11.0.0 是最后一代 proot 版本。**

---

## 6. 常见疑问澄清

- **"容器"是 Docker 吗？** 不是。没有容器运行时、没有内核命名空间/cgroup 隔离，rootfs 只是普通目录。新架构连"隔离"都几乎只剩目录层面的含义。
- **是虚拟机/模拟器吗？** 不是。不虚拟化 CPU：ARM 设备上由 box86/box64 做指令翻译，x86 设备直接原生执行；全部进程与 Android 系统共享同一内核。
- **性能开销在哪？** 主要两处：box86/box64 的指令翻译开销；DirectX→Vulkan/OpenGL 转译 + 图形驱动的兼容性与开销（Adreno 配 Turnip 效果通常最好，Mali 走 Zink）。
- **为什么个别游戏要换 Box64 预设或图形驱动？** 因为每个程序的指令特征与图形调用不同，预设只是环境变量组合（DYNAREC 开关、线程数、调优变量），驱动决定 D3D 转译路径。
- **命令是固定写死的吗？** 不是。上面的命令是源码重建出的"最完整形态"，每次运行都会按容器/快捷方式设置动态拼接：分辨率、目标程序、附加参数、预设变量都不同。要拿某台机器上的真实命令，需在 App 日志里查看。

---

## 7. 证据来源清单（可直接打开核对）

当前版（`winlator-app` 仓库，v11.2.0 子模块 commit `c2f4ad4534f4637b543a9a3b085e28f50cf6d01c` 与 main 内容一致）：

- `app/src/main/java/com/winlator/xenvironment/components/GuestProgramLauncherComponent.java` —— 启动命令与环境变量拼装（`execGuestProgram()`、`addBox64EnvVars()`）
- `app/src/main/java/com/winlator/XServerDisplayActivity.java` —— `WINEPREFIX`/`WINEESYNC`/图形组件装配、`getWineStartCommand()`（第 955–990 行）
- `app/src/main/java/com/winlator/xenvironment/RootFS.java` —— rootfs 路径、`/home/xuser`、`WINEPREFIX`、wine 路径
- `app/src/main/java/com/winlator/xconnector/UnixSocketConfig.java` —— 各服务 socket 路径常量
- `app/src/main/java/com/winlator/core/GeneralComponents.java` —— box64/图形驱动/DX wrapper 组件解压目标
- `app/src/main/java/com/winlator/core/ProcessHelper.java` —— `ProcessBuilder` 执行细节（cwd、环境、输出重定向）
- `app/src/main/java/com/winlator/container/ContainerManager.java`、`container/Container.java` —— 容器目录与 `xuser` 符号链接
- `app/src/main/assets/box64/box64-0.4.4.tzst`、`assets/graphics_driver/*`、`assets/dxwrapper/*` —— 内置组件清单

老版本（`brunodev85/winlator` 仓库）：

- v11.0.0 `app/src/main/java/com/winlator/xenvironment/components/GuestProgramLauncherComponent.java` 第 136、155、159–160 行 —— 仍用 `libproot.so`
- v6.1.0 同名文件第 123–155 行 —— proot 完整命令；`XServerDisplayActivity.java` 第 351–405 行 —— 盘符 bind 与组件装配
- v6.1.0 `app/src/main/cpp/proot/` —— 内置 proot 完整源码
- 各标签文件树（`v8.0.0` … `v11.2.0`、`main`）—— 是否含 proot、app 是否子模块

官方说明：仓库 README 声明的第三方组件包括 Wine、Box86/Box64（ptitSeb）、Mesa（Turnip/Zink/VirGL）、DXVK、VKD3D、CNC DDraw，以及 Termux Pacman 的 GLIBC Patches（即 rootfs 里的 glibc 来源）。

[DuMate AI生成]
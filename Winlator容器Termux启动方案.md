# Winlator rootfs 在 Termux 中启动方案（图形输出到 Termux X11）

日期：2026-09-29（含实际验证结果，已更新）

## 1. 结论

- 容器可以启动：proot 包裹 rootfs 后，box64/wine/wineboot 均能正常运行。
- **图形输出到 Termux X0 目前失败**：rootfs 缺 x86_64 版 X11 客户端库，wine 的 winex11.drv 无法加载，回退 nodrv（无显示驱动）。
- 这是 Winlator 的设计使然：App 正常工作时走 Android 原生图形栈（SurfaceFlinger/ANativeWindow + Turnip/VirGL），rootfs 从不需要 x86_64 X 库。

## 2. 环境探测结果

| 项目 | 结果 |
|------|------|
| rootfs 路径 | /data/user/0/com.winlator/files/rootfs（Termux 可直接读写） |
| rootfs 版本 | .rfs_version = 23（v11.1+ 架构，已去 proot，glibc 完整 rootfs） |
| 关键文件 | /usr/local/bin/box64(30MB, v0.4.4 Dynarec)、winpreexec、/opt/wine/bin/（wine/wineboot/winecfg 等）、/etc/config.box64rc |
| usr/bin 内容 | 极精简：仅 locale/localedef（无 bash、无 env、无 X11 客户端） |
| wine prefix | home/xuser -> xuser-1，含 .wine，wineboot 初始化正常 |
| Termux X server | termux-x11-nightly 运行中，DISPLAY=:0 |
| X0 socket | /data/data/com.termux/files/usr/tmp/.X11-unix/X0，权限 srwxrwxrwx（全局可读写） |
| uid 对齐 | Termux 与 com.winlator 同为 u0_a234（uid 10234），权限完全打通 |
| Termux proot | proot 5.1.107.92 已装 |

## 3. 实际验证结果（2026-09-29）

| 步骤 | 结果 |
|------|------|
| proot 内 ls /tmp/.X11-unix/X0 | ✅ X0 socket bind 生效 |
| box64 --version | ✅ Box64 arm64 v0.4.4 with Dynarec |
| box64 wine wineboot -e | ✅ 正常退出，prefix 初始化成功 |
| winecfg 窗口映射到 X0 | ❌ 失败，winecfg 进程存活约 27 秒后静默退出 |

错误原文（去掉 WINEDEBUG=-all 后捕获）：

```
00b0:err:winediag:nodrv_CreateWindow Application tried to create a window, but no driver could be loaded.
00b0:err:winediag:nodrv_CreateWindow L"The explorer process failed to start."
```

## 4. 根因分析

- $R/opt/wine/lib/wine/x86_64-unix/winex11.so 是 **x86-64 ELF**，依赖 libX11.so.6、libxcb*、libXrandr、libXrender、libXcursor、libXi、libXfixes、libXcomposite、libXxf86vm、libGL、libvulkan 等。
- rootfs 中这些库**只有 aarch64 版本**（/usr/lib/）；/usr/lib/x86_64-linux-gnu/ 和 /lib/x86_64-linux-gnu/ 各只有 1 个文件（libgcc_s.so.1）。
- WINEDEBUG=+x11drv 零输出 → winex11.drv 根本没被加载。
- box64 内仅有 7 个 X 符号 wrapper（XOpenDisplay/XCreateWindow 等），不足以让 winex11.so 完整加载。

## 5. 修正后的启动命令（已验证可跑通进程）

两处相对初始方案的硬伤修正：
1. rootfs 内无 env → 改在 Termux 侧用 env 设置环境变量。
2. box64 的 ELF interpreter 是绝对路径 /data/data/com.winlator/files/rootfs/lib/ld-linux-aarch64.so.1 → 必须额外 bind rootfs 自身到同一路径。

```bash
R=/data/user/0/com.winlator/files/rootfs
P=/data/data/com.termux/files/usr/bin/proot
env HOME=/home/xuser USER=xuser TMPDIR=/tmp DISPLAY=:0 \
  PATH=/opt/wine/bin:/usr/local/bin:/usr/bin:/bin \
  WINEPREFIX=/home/xuser/.wine WINEDEBUG=-all WINEESYNC=0 \
$P -r $R -b /dev -b /proc -b /sys \
  -b /data/data/com.termux/files/usr/tmp/.X11-unix:/tmp/.X11-unix \
  -b /data/data/com.winlator/files/rootfs:/data/data/com.winlator/files/rootfs \
  -w /home/xuser \
  /usr/local/bin/box64 /opt/wine/bin/wine winecfg
```

当前状态：进程能跑（winecfg/wineboot 均正常启动退出），GUI 无法显示。

## 6. 修复方向（图形输出）

1. **补 x86_64 X 库**（推荐，未执行）：从 Debian/Ubuntu amd64 的 libx11、libxcb1、libxrandr、libxrender、libxcursor、libxi6、libxfixes、libxcomposite、libxxf86vm、libglvnd、libvulkan 等包提取 .so，放入 $R/usr/lib/x86_64-linux-gnu/，并在启动命令中设 BOX64_LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu。
2. 改用 Termux 原生 x11 版 wine —— 架构不同（arm64 wine vs rootfs x86_64 wine prefix），不可行。
3. 接受现状：仅作无头 wine 用途（安装程序、改注册表等），不做 GUI。

## 7. 其他注意事项

- WINEESYNC=0：Winlator App 未运行时无 SysV shm 服务（/tmp/.sysvshm/SM0 不存在），esync 会失败。
- 无音频：缺 PULSE_SERVER socket，wine 退化为静音。
- 勿删除/覆盖 rootfs 内现有文件；wine 写 prefix 属正常。
- Winlator App 正常启动时使用原生命令：`<rootfs>/usr/local/bin/box64 wine explorer /desktop=nogui,1280x720 C:\windows\winhandler.exe /dir <游戏目录> "<游戏.exe>"`

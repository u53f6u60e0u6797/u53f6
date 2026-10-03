# Termux OpenJDK IPv6 绑定失败诊断报告

- 日期:2026-10-04
- 环境:Android(Termux),OpenJDK 26.0.2 vendor=Termux,aarch64,bionic libc
- 事发场景:`cd /data/data/com.termux/cache/mc && java -jar fabric-server-launch.jar`(Minecraft Fabric 服务端)绑定 IPv6 地址 `::` 时崩溃

## 一、现象

| 现象 | 出处 |
|------|------|
| MC 服务端启动即崩,`java.nio.channels.UnsupportedAddressTypeException`(sun.nio.ch.Net.checkAddress → ServerSocketChannelImpl.netBind → NioServerSocketChannel.doBind) | logs/latest.log:216-219、crash-reports/crash-2026-10-03_15.24.43-server.txt |
| Geyser 基岩 UDP 端口绑定同样抛 `UnsupportedAddressTypeException`(DatagramChannelImpl.bindInternal) | 同日志 |
| `Failed to find a usable hardware address from the network interfaces`(佐证:JDK 网络视野异常) | 同日志:217 |
| Java 老 API 报真因:`java.net.SocketException: IPv6 protocol family unavailable` | TestDiag 实测 |
| Java 连接(非绑定)IPv6 出站也失败,失败在 socket 创建阶段 | Connect6 实测 |
| `InetAddress.getByName("::")` 解析正常,返回标准 `java.net.Inet6Address` | TestDiag 实测 |
| Java `NetworkInterface` 枚举不到任何 IPv6 地址(连 lo 的 ::1 都看不到) | TestDiag 实测 |
| 原生程序完全正常:nmap -6 扫 :: 发现大量开放端口(5555/8022/5222 等) | 用户实测 |

## 二、关键实验与证据链

| # | 实验 | 结果 | 结论 |
|---|------|------|------|
| 1 | C 程序 `socket(AF_INET6)` + `bind(::)` + `listen` | 全部成功 | 内核 IPv6 完全可用 |
| 2 | C `getifaddrs()` 枚举 | rmnet_data0-3 均有 IPv6 地址(含 240e: 中国电信、2409: 中国移动全球单播地址),lo 有 ::1 | 设备有真实 IPv6 网络 |
| 3 | C 读 `/proc/net/if_inet6`、`/proc/net/tcp6` | EACCES(Permission denied) | Android 10+ SELinux 禁止 App 域读 /proc/net/* |
| 4 | Java 读 IPv6(socket/bind/connect) | 全部失败,报"协议族不可用" | 问题 100% 在 JDK 运行时层 |
| 5 | 在 JDK 库中溯源报错字符串 | `"IPv6 protocol family unavailable"` 唯一存在于 `$JAVA_HOME/lib/libnet.so`,同文件含 `if_inet6` 字符串与 `ipv6_available` 符号 | 自检逻辑所在 |
| 6 | 上游源码 `src/java.base/unix/native/libnet/net_util_md.c` 的 `IPv6_supported()` | Linux 分支:`socket(AF_INET6)` 成功后仍要 `fopen("/proc/net/if_inet6")`,**打开失败即 return JNI_FALSE** | 误判的代码根源 |

## 三、根因(因果链)

```
Android 10+ SELinux 拒绝 App 域读取 /proc/net/if_inet6 (EACCES)
  → Termux OpenJDK 的 libnet.so 启动自检 ipv6_available() 打不开该文件
    → JVM 误判"本机无 IPv6"(实际内核与网络均正常)
      → Java 之后创建任何 AF_INET6 socket 一律被拒
        → MC/Netty 绑 :: 时抛误导性的 UnsupportedAddressTypeException
```

一句话:**不是内核不支持、不是 MC/Netty 的锅、也不是配置写法问题 —— 是 Termux 移植的 OpenJDK 用"翻文件"的方式自检 IPv6,而这个文件恰好被 Android 的 SELinux 锁了,于是 JDK 自欺欺人地关闭了 IPv6。** 报错 `UnsupportedAddressTypeException` 是替罪羊:地址对象本身(`Inet6Address`)完全合法,只是 NIO 探测到 IPv6 不可用后把 channel 锁为 IPv4 family,收到 Inet6Address 即抛异常。

## 四、已知同类问题(社区先例)

| 来源 | 内容 |
|------|------|
| [PojavLauncherTeam/android-openjdk-build-multiarch PR #26](https://github.com/PojavLauncherTeam/android-openjdk-build-multiarch/pull/26) | 补丁 `26_skip_proc_net_check.diff`:在 `__ANDROID__` 下跳过该检测,注释引用 AOSP libcore commit ae218d9b(Google 自家嵌入的 JDK 也需绕过) |
| [PojavLauncher issue #5125](https://github.com/PojavLauncherTeam/PojavLauncher/issues/5125) | "[F-Req] No IPv6 Support",Android 10,MC 无法连 IPv6 服务器 |
| [HMCL-PE issue #211](https://github.com/HMCL-dev/HMCL-PE/issues/211) | "ipv6无法连接服务器",Netty Protocol family unavailable,`preferIPv4Stack` 无效 —— 症状完全一致 |
| [TUR tur/openjdk-26](https://github.com/termux-user-repository/tur/tree/master/tur/openjdk-26) | 36 个补丁中无 IPv6 相关修复,**无人报告过此问题** |

## 五、方案与绕过

| 方案 | 可行性 | 说明 |
|------|--------|------|
| IPv4 运行(server-ip= 留空或 0.0.0.0) | ✅ 当前唯一"官方"姿势 | MC + Geyser 均已实测正常启动(TCP 25565 / UDP 19132 仅 IPv4) |
| root 下 `su` 跑 java | ✅(有 root 时) | root 域不受该 SELinux 限制,自检可通过 |
| LD_PRELOAD 劫持 open(/proc/net/if_inet6) | 🧪 已备好待验证 | /data/data/com.termux/cache/tmp/preload_ifinet6.so + run_step1.sh;向自检喂假文件内容即可骗过 |
| 向 TUR 报 issue | ✅ 最佳长期方案 | 附 Pojav 现成补丁,你可能是第一个报告者 |
| proot-distro + glibc JDK | ❌ 无效 | SELinux 按进程域管理,proot 不换域,读同一文件同样 EACCES |
| JVM 参数(preferIPv4Stack / preferIPv6Addresses) | ❌ 无效 | 自检发生在启动时,参数无法逆转 |

## 六、附:本机验证产物(均在 /data/data/com.termux/cache/tmp/)

- `ipv6test.c` / 二进制 — C 对照测试
- `Connect6.java` — Java 出站连接测试
- `TestDiag.java` — 综合诊断(解析/接口/双 API 绑定)
- `preload_ifinet6.c` / `.so` / `fake_if_inet6` / `run_step1.sh` — LD_PRELOAD 绕过实验(待运行)

报告完。核心结论:**Termux OpenJDK 26.0.2 在 Android 上的 IPv6 支持探测被 SELinux 破坏,属上游检测逻辑与 Android 安全模型的冲突;设备本身 IPv6 完全正常。**

# 一、结论摘要

在 Android（Termux）上运行 OpenJDK（bionic 移植版）时，因 SELinux 拒绝应用域读取 /proc/net/if_inet6，JDK 启动自检会误判"本机无 IPv6"，导致所有 Java 程序的 IPv6 绑定与连接失败。该问题与具体应用无关，**已实测影响 JDK 21、25、26 三个大版本**（Minecraft Fabric 服务端、Apache Tomcat 均为受害场景）。通过 LD_PRELOAD 注入一个劫持 fopen 的小型共享库，将 /proc/net/if_inet6 的读取重定向为动态生成的内容，即可恢复 IPv6 能力，**上述三个版本均已验证有效**。

# 二、问题背景与现象

## 触发场景

- 首次发现：Termux + OpenJDK 26.0.2 运行 Minecraft Fabric 服务端（java -jar fabric-server-launch.jar），绑定 IPv6 通配地址 :: 时崩溃；
- 历史遗留：此前使用 JDK 21 运行 Apache Tomcat 时即遇到 IPv6 连接不上，当时误判为"手机上的兼容性 BUG"，实为同一根因；
- 补充验证：JDK 25 默认同样无法绑定 IPv6 端口。

## 典型现象归纳

| 现象                       | 说明                                                                                         |
|----------------------------|----------------------------------------------------------------------------------------------|
| MC 服务端启动即崩          | 抛 java.nio.channels.UnsupportedAddressTypeException（Netty/ServerSocketChannel 绑定 :: 时） |
| Geyser 基岩版 UDP 绑定失败 | 同样抛 UnsupportedAddressTypeException                                                       |
| 老 API 报真因              | java.net.SocketException: IPv6 protocol family unavailable                                   |
| Java 枚举不到 IPv6 地址    | NetworkInterface 连 lo 的 ::1 都看不到                                                       |
| 原生程序完全正常           | socket(AF_INET6) + bind(::)、getifaddrs() 枚举、nmap -6 扫描均正常，设备有真实 IPv6 网络     |

# 三、根因分析

Android 10+ SELinux 拒绝 App 域读取 /proc/net/if_inet6（EACCES）  
→ Termux OpenJDK 的 libnet.so 启动自检 IPv6_supported() 打不开该文件  
→ JVM 误判"本机无 IPv6"（实际内核与网络均正常）  
→ Java 之后创建任何 AF_INET6 socket 一律被拒  
→ MC/Netty 绑定 :: 时抛误导性的 UnsupportedAddressTypeException

**代码根源**：OpenJDK 上游 src/java.base/unix/native/libnet/net_util_md.c 的 IPv6_supported() 在 Linux 分支中，socket(AF_INET6) 成功后仍要 fopen("/proc/net/if_inet6")，**打开失败即返回 JNI_FALSE**。该文件在 Android 10+ 上对应用域不可读，因而误判。UnsupportedAddressTypeException 只是替罪羊：地址对象本身完全合法，是 NIO 探测到 IPv6 不可用后锁定了 channel 协议族。

# 四、多版本影响范围验证（JDK 21 / 25 / 26）

| JDK 版本 | 默认行为（无注入）                                           | 注入 preload 后                         | 证据         |
|----------|--------------------------------------------------------------|-----------------------------------------|--------------|
| 21       | IPv6 连接不上（Tomcat 时代已遇到，当时误判为手机兼容性问题） | 恢复正常                                | 用户实测     |
| 25       | 无法正常绑定 IPv6 端口                                       | 能正常绑定 IPv6 端口                    | 用户实测     |
| 26       | MC Fabric 绑 :: 崩溃，Geyser 同样失败                        | 正常绑定 :::25565、Geyser 绑定 :::19132 | 设备启动日志 |

三个大版本症状一致、修复一致，进一步确认问题不在某个发行版实现差异，而在于**上游检测逻辑与 Android SELinux 安全模型之间的长期冲突**。

# 五、修复方案：LD_PRELOAD 劫持 fopen

## 原理

JDK 的 libnet.so 读取 /proc/net/if_inet6 走的是 fopen()（glibc 下为带版本符号 fopen@GLIBC_2.2.5，未定义符号表中无 open 系符号）。因此劫持 open() 无效，**必须劫持 `fopen`**。实现要点：

- 覆盖 fopen、fopen64、__fopen_2 三个符号，仅对精确路径 /proc/net/if_inet6（且以读取模式打开）重定向，其余调用原样透传（RTLD_NEXT）；
- 返回内容**动态生成**：因为 JDK 的网络接口枚举同样读取该文件，固定假内容会导致枚举失真；改用 getifaddrs()（走 netlink，SELinux 允许，与真实系统一致地按 if_inet6 行格式生成地址、索引、前缀长度、作用域和标志）；
- 提供 PRELOAD_IFINET6_DEBUG=1 环境变量，可在 stderr 打印命中记录，便于现场确认劫持是否生效；
- bionic 无符号版本机制，Termux 上用同源码直接编译即可；glibc 环境实测未版本化符号也能绑定到版本化引用，无需 .symver。

## 使用方式

```bash
# Termux 内编译
cc -shared -fPIC -O2 -o preload_ifinet6.so preload_ifinet6.c -ldl
# 启动任意 Java 程序
LD_PRELOAD=$PWD/preload_ifinet6.so java -jar server.jar
```


# 六、修复验证结果

## 沙箱验证（glibc + JDK 21，开发环境）

1.  LD_DEBUG=bindings 确认 libnet.so 的 fopen 引用绑定到注入库而非 libc；
2.  C 层注入：fopen("/proc/net/if_inet6") 读取到动态生成内容；
3.  JVM 端到端：网络接口枚举结果与真实系统逐项一致（eth0/eth1 的 fe80 地址、lo 的 ::1 均在列），JVM 正常初始化。

## 设备验证（Termux + OpenJDK 26，Minecraft Fabric + Geyser）

设备日志（2026-10-05）显示**两次完整启动均成功**：

- 启动日志出现 4 次 [preload_ifinet6] fopen(/proc/net/if_inet6) -> FAKE stream，劫持确认命中；
- Starting Minecraft server on :::25565、Done (0.381s)!、Query running on :::25565——TCP 25565 双栈绑定成功；
- Geyser 已在 :::19132 上启动——UDP 19132 双栈绑定成功；
- 此前崩溃的 UnsupportedAddressTypeException / IPv6 protocol family unavailable 均未再出现。

# 七、遗留问题与后续建议

## 与本次修复无关的已知现象

| 现象                                                            | 性质                                                                                              |
|-----------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| JNA 报 library "libc.so.6" not found、Did not JNA classes       | bionic 无 glibc 的 libc.so.6 名，JNA 在 Termux 的已知问题，被 SystemReport 忽略，不影响服务端运行 |
| Failed to find a usable hardware address ... using random bytes | Termux 取不到真实 MAC（另一处 SELinux 限制），服务端改用随机字节，不影响运行                      |
| SERVER IS RUNNING IN OFFLINE/INSECURE MODE                      | server.properties 配置项（online-mode），与网络栈无关                                             |

## 后续建议

- **长期方案（推荐）**：向 TUR（termux-user-repository/tur）提交 issue，附上现成补丁 [PojavLauncherTeam/android-openjdk-build-multiarch PR #26](https://github.com/PojavLauncherTeam/android-openjdk-build-multiarch/pull/26)（26_skip_proc_net_check.diff 在 __ANDROID__ 下跳过该检测，注释引用了 AOSP libcore 同款绕法 commit ae218d9b）。目前 TUR 的 openjdk 包（21/25/26）补丁目录中尚无任何 IPv6 相关修复，且仓库中无同类 issue，若提交可能成为首个报告；
- **即时方案（已验证）**：继续使用本报告的 LD_PRELOAD 注入库，JDK 21/25/26 均有效，无需重编 JDK；
- **彻底方案**：在 TUR 打包脚本中打入上述补丁后重编，从源头修复。

# 八、最终结论

设备本身 IPv6 完全正常；问题根源是 Termux 移植的 OpenJDK 用"读取 /proc/net/if_inet6 文件"的方式自检 IPv6，而该文件恰好被 Android SELinux 锁定，导致 JDK 自欺欺人地关闭了 IPv6。该问题跨 JDK 21/25/26 稳定存在，LD_PRELOAD 劫持 fopen 的注入方案已在三个版本上验证有效，是当前最轻量、零重编的修复方式。


[DuMate AI生成]
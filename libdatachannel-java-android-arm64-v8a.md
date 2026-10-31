# libdatachannel-java_for_Android

`tel.schich:libdatachannel-java` 的 **Android arm64-v8a** 版本（原生库 + Java 绑定），
由上游源码用 Android NDK r27c 交叉编译得到，可直接放进 Android 工程使用。

## 包内容

```
jniLibs/arm64-v8a/libdatachannel-java.so   Android arm64-v8a 原生库（4,570,040 字节）
libs/libdatachannel-java-binding.jar       Java 绑定（38 个源文件编译，71 个 class）
libs/slf4j-api-2.0.18.jar                  绑定运行期依赖（org.slf4j，应用若已自带可不加）
SHA256SUMS.txt                             三个文件的 SHA-256
```

## 对应的上游版本

| 组件 | 版本 / revision |
| --- | --- |
| libdatachannel-java（Java 绑定 + JNI 层） | revision `0eb42beb7f7b17e8b383ea848a52d1c2da0c8e7c`（对应 Geyser 快照里 `libdatachannel-java-arch-detect-0.24.6.0-20261023.142230-9.jar`） |
| libdatachannel（C++ 本体） | revision `02448c89fed88f3bd0cdcc808ae441ac779c1756` |
| deps：usrsctp / plog / libjuice / libsrtp | `fec583d54493f879d2ae44a743423bf8a04371ab` / `94899e0b926ac1b0f4750bfbd495167b4a6ae9ef` / `dd781c6eff58b6b3ee3325f6546b918afa839121` / `d33b8ffb1491a0b4b58a206889f09800cf7310ab` |
| TLS 后端 | mbedTLS v3.6.2（上游默认 OpenSSL，原因见下） |
| 工具链 | Android NDK r27c，`arm64-v8a`，`android-21`，`c++_static`，CMake 4.4.4 + Ninja |

## 怎么用

1. 把 `jniLibs/arm64-v8a/libdatachannel-java.so` 放到 Android 工程的 `app/src/main/jniLibs/arm64-v8a/`
   （或按 ABI 分目录打包进 APK）。
2. 把 `libs/libdatachannel-java-binding.jar` 放进 `app/libs/` 并加入依赖；运行期还需要
   `org.slf4j:slf4j-api`（包内已附 2.0.18）。
3. 绑定会自动找库，查找顺序（`tel.schich.libdatachannel.Platform`）：
   1) `System.loadLibrary("datachannel-java")`（走 `java.library.path`）；
   2) 失败后先看系统属性 `libdatachannel.native.datachannel-java.path`，命中则直接 `System.load(该绝对路径)`；
   3) 再看系统属性 `libdatachannel.native.datachannel-java.classpath`，其值形如
      `linux-glibc-aarch64/native/libdatachannel-java.so`；
   4) 都没有时，按 classpath 上 `<os>-<arch>/native/libdatachannel-java.so` 的约定找，
      再退到 `native/libdatachannel-java.so`（上游 arch-detect 包的布局）。
   属性前缀为 `libdatachannel.native.`，库名固定为 `datachannel-java`。
4. 若要把本包的 `.so` 接进「已经内含上游 arch-detect / 绑定 jar」的现成程序（例如 Geyser），
   不必改任何 jar：直接用第 2 条的属性指向本包的 `.so` 即可，例如
   `-Dlibdatachannel.native.datachannel-java.path=/data/local/tmp/libdatachannel-java.so`。
   在 Bionic 的 JVM 进程里通常还需同时给出 `-Dlibdatachannel.libc=glibc`：
   Bionic 在 `/proc/self/maps` 里显示的是 `libc.so`，既不是 `libc.so.6` 也不是 `ld-musl-*`，
   `LinuxLibc.detect()` 无法判定，会抛 `IllegalStateException` 要求显式指定；
   该属性只用于拼目录名，不影响已加载的 libc。

## 与上游构建的差异（共 4 处，均已记录）

1. **TLS 后端改为 mbedTLS 3.6.2**：OpenSSL 3.6.3 源码 55 MB，在当前环境下载速率约 36 KB/s（≈26 分钟）且国内镜像不可用，故改用 mbedTLS 源码并以 `-DUSE_MBEDTLS=ON` 构建。
2. **mbedTLS 显式开启 `MBEDTLS_SSL_DTLS_SRTP`**：mbedTLS 默认不编译 DTLS-SRTP 扩展，而 libdatachannel 的媒体路径需要它（`src/impl/dtlstransport.cpp`），否则编译报 `unknown type name 'mbedtls_ssl_srtp_profile'`。
3. **`jni/src/init.cpp` 的线程栈兜底逻辑排除 Android**：上游的 `#if defined(__linux__) && !defined(__GLIBC__)` 分支是为 musl（默认线程栈仅 128 KiB）准备的，调用了 glibc/musl 的 `pthread_getattr_default_np` / `pthread_setattr_default_np`；Bionic 既没有这两个符号，默认线程栈也已经是 1 MiB，因此该分支改为 `#if defined(__linux__) && !defined(__GLIBC__) && !defined(__ANDROID__)`，Android 下直接跳过。这是唯一一处对源码的修改。
4. **链接时按 16 KB 页对齐**：额外开启 NDK 的 `ANDROID_SUPPORT_FLEXIBLE_PAGE_SIZES=ON`，产出的 `LOAD` 段对齐为 `0x4000`（16 KB），满足 Android 15+ 的 16 KB 页要求；上游已发布的 `android-arm64-v8a` 构件是 `0x1000`（4 KB）。

JNI 桥接源码 `jni/generated/jni-c-to-java.{c,h}`、`jni-java-to-c.h` 未手写，而是用上游的官方注解处理器
`tel.schich:jni-access-generator:1.2.3`（`-Agenerate.jni.headers=true -Agenerate.cache.mode.default=EAGER_PERSISTENT`）从同一份 Java 源码生成。

## 验证结果

静态取证（对 Android 产物 `jniLibs/arm64-v8a/libdatachannel-java.so`）：

- ELF64 / `EM_AARCH64` / `DYN`，`SONAME = libdatachannel-java.so`；
- `DT_NEEDED` 仅 `liblog.so`、`libm.so`、`libdl.so`、`libc.so`（与上游已发布 Android 构件一致，无 `libc++_shared`、无外部 TLS/SRTP/SCTP 动态依赖）；
- 导出 **78 个** `Java_tel_schich_libdatachannel_*` JNI 入口 + `JNI_OnLoad` / `JNI_OnUnload`；与上游 Linux-aarch64 成品的 78 个入口**逐名完全一致**，也与本 revision 生成的 `jni-java-to-c.h` 78 个声明**一一对应（两个方向差集均为空）**；
- 全部未定义符号都是 Bionic（`@LIBC`）符号；
- `LOAD` 段对齐 `0x4000`（16 KB），段间偏移同余；无 `RPATH/RUNPATH`。

运行时验证（沙箱内无法运行 Android ART，故用同一套源码在宿主机 x86_64 上构建同源库做功能验证）：

- 宿主机构建 `libdatachannel-java.so`（glibc + mbedTLS 静态链接）并加载 Java 绑定；
- 运行上游自带测试套件（PeerConnectionTest、DataChannelReliabilityTest、PeerCallbackLifecycleTest、BinaryCallbackOwnershipTest、TrackLifecycleTest、NativeThreadLifecycleTest、PlatformTest、LinuxLibcTest）：**22 项全部通过**（包括真实 ICE/DTLS 回环建链、数据通道收发与可靠性、回调生命周期、轨道生命周期）。
  该结果证明 JNI 桥接层（本地生成的 `jni-c-to-java.c`）、Java 绑定与 libdatachannel 本体在功能上互通正确。

## 限制与未覆盖项

- 只构建了 **arm64-v8a**；`armeabi-v7a` / `x86` / `x86_64` 需要用同一流程各自重编（见下方复现命令，改 `-DANDROID_ABI`）。
- **Android 上的动态加载与运行未实测**：沙箱内没有 Android/ART 运行时，也没有可用的 aarch64-Bionic 执行环境，因此「APK 里 JVM 加载该 `.so` 并完成一次 DTLS 握手」只能在设备上验证。
- 包为静态链接 TLS/SRTP/SCTP（体积换独立性），如需减小体积可改为依赖系统/共享库。
- 未做性能压测、长稳测试与真实 Network 环境（STUN/TURN）验证。

## 复现命令（关键步骤）

```bash
# 1) 依赖源码与 JNI 代码（已在本包对应的构建目录内完成）
javac -proc:only -processorpath tools/libs/jni-access-generator-1.2.3.jar \
  -Agenerate.jni.headers=true -Agenerate.cache.mode.default=EAGER_PERSISTENT \
  -h src/ldc-java/jni/generated -s /tmp/javagen -d /tmp/javagen \
  $(find src/ldc-java/src/main/java -name '*.java')

# 2) mbedTLS（arm64-v8a），注意需开启 DTLS-SRTP
cmake -S src/mbedtls -B build/mbedtls-arm64 -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=$PWD/tools/android-ndk-r27c/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-21 -DANDROID_STL=c++_static \
  -DCMAKE_BUILD_TYPE=Release -DENABLE_TESTING=OFF -DENABLE_PROGRAMS=OFF \
  -DUSE_SHARED_MBEDTLS_LIBRARY=OFF -DUSE_STATIC_MBEDTLS_LIBRARY=ON \
  -DCMAKE_INSTALL_PREFIX=$PWD/out/prefix
cmake --build build/mbedtls-arm64 --target install -j 2

# 3) libdatachannel + JNI 层
cmake -S src/ldc-java/jni -B build/jni-arm64 -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=$PWD/tools/android-ndk-r27c/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-21 -DANDROID_STL=c++_static \
  -DANDROID_SUPPORT_FLEXIBLE_PAGE_SIZES=ON -DCMAKE_BUILD_TYPE=Release \
  -DLIBDATACHANNEL_SOURCE_DIR=$PWD/src/ldc -DUSE_MBEDTLS=ON \
  -DCMAKE_PREFIX_PATH=$PWD/out/prefix
cmake --build build/jni-arm64 --target datachannel-java -j 2
# 产物：build/jni-arm64/libdatachannel-java.so
```
# LinkX4C EtherCAT SDK

在自己的 C/C++ 应用中，通过 EtherCAT 使用 LinkX4C 的四路 CAN / CAN FD。应用按通道提交 CAN 帧并接收回调；周期通信、通道队列和帧聚合由 SDK 处理，无需自行装配 PDO 或调用 SOEM/IgH。

当前版本为 **0.3.0 预览版 / C ABI 1**，支持单网卡/主站、单台 LinkX4C、四个通道（编号 `0..3`），要求从站使用 PDO v8。已知缺帧、错载荷及 WKC 不足问题尚未完成闭环，本版不提供生产无损或长期稳定性保证。

## 1. 选择并下载

| 系统 | 发布包 | 需要另外准备 |
| --- | --- | --- |
| Windows x64 | [下载 0.3.0](0.3.0/Win_x64/dmecat-0.3.0-win64-preview-c846920-public1.zip) | Npcap 的 WinPcap API 兼容支持、MSVC x64 Release 运行时 |
| Linux x86_64 | [下载 0.3.0](0.3.0/Linux_x64/dmecat-0.3.0-linux-x64-preview-c846920-public1.tar.gz) | IgH 1.5.4 / API 1.5 用户态库、匹配的内核模块、主站网卡配置及设备权限 |

Windows 包包含 `dmecat.dll` 和独立的 `soem.dll`，**仅支持 SOEM 2.0.0**，不要替换为其他版本。Linux 包使用 IgH 后端，不使用 SOEM/Npcap。两平台公共 API 相同，动态库不能交叉使用。C# 目前提供 ABI 布局和加载示例，不是完整业务封装。

所有平台包及配套资料按版本放在 [0.3.0/](0.3.0/) 中。解压后，使用同一个包内的头文件、导入库和动态库，不要混用旧版 SDK 或 `SDK/历史Demo`。

## 2. 校验下载文件

对照 [0.3.0/SHA256SUMS.txt](0.3.0/SHA256SUMS.txt) 核对压缩包。Windows 可用 `Get-FileHash <压缩包> -Algorithm SHA256`；Linux 在 `0.3.0` 目录运行 `sha256sum -c SHA256SUMS.txt --ignore-missing`，可只下载本平台文件。

解压后，还可用包内工具检查归档中的实际文件：

```text
python -B <SDK解压目录>/tools/verify_sdk_archive.py <下载的zip或tar.gz>
```

## 3. 编译并枚举设备

### Windows x64

安装上述运行依赖。编译示例还需要 MSVC C/C++ 工具链、CMake 和 Ninja。在 **x64 Native Tools Command Prompt** 中执行，将路径替换为自己的解压目录：

```bat
chcp 65001
set "DMECAT_SDK_DIR=C:\sdk\dmecat-0.3.0-win64-preview-c846920-public1"
cmake -S "%DMECAT_SDK_DIR%\examples" -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=cl -DCMAKE_CXX_COMPILER=cl -DCMAKE_PREFIX_PATH="%DMECAT_SDK_DIR%"
cmake --build build
copy "%DMECAT_SDK_DIR%\bin\x64\dmecat.dll" build\
copy "%DMECAT_SDK_DIR%\bin\x64\soem.dll" build\
build\example_cpp.exe --list
```

### Linux x86_64

先准备 IgH 及目标 `/dev/EtherCAT<n>` 的读写权限。编译示例需要 C/C++ 工具链和 CMake。下面以 IgH 安装在 `/opt/etherlab`、SDK 库目录为 `lib` 为例；实际使用其他前缀或 `lib64` 时请对应替换：

```sh
export DMECAT_SDK_DIR=/path/to/dmecat-0.3.0-linux-x64-preview-c846920-public1
cmake -S "$DMECAT_SDK_DIR/examples" -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH="$DMECAT_SDK_DIR" -DDMECAT_IGH_ROOT=/opt/etherlab
cmake --build build
LD_LIBRARY_PATH="$DMECAT_SDK_DIR/lib:/opt/etherlab/lib" ./build/example_cpp --list
```

`--list` 只枚举，不打开从站、不切换模式、不发 CAN。枚举成功不代表已经具备打开权限。加载失败时，先检查 x64 架构、运行依赖及动态库搜索路径。

## 4. 运行第一次收发

先确认网口没有被 TwinCAT、其他 EtherCAT 主站或另一个程序占用。将 **CH0 与 CH1** 接通，确认共地、终端电阻和位率匹配。

包内 C++ 示例使用 CAN FD，仲裁段 **1 Mbps**、数据段 **5 Mbps**、BRS、DLC=8，由 CH0 发、CH1 收，默认 **SM / 1 ms**。这不是控制器内部回环；与其他节点通信时需要正常应答节点。

**带设备路径运行示例会切换 ECAT 模式、配置 CAN 并发送帧，不要在业务总线上直接试跑。** 使用 `--list` 返回的完整路径：

```bat
build\example_cpp.exe "\Device\NPF_{替换为枚举所得GUID}"
```

Linux 保持动态库路径设置，使用枚举所得 `/dev/EtherCAT<n>`：

```sh
LD_LIBRARY_PATH="$DMECAT_SDK_DIR/lib:/opt/etherlab/lib" ./build/example_cpp /dev/EtherCAT0
```

示例每轮发送 8 帧，核对 ID、接收通道、DLC 及完整载荷，停止后在同一句柄再跑一轮。正常结果应为每轮 `received=8/8 valid=yes` 且退出码为 0；这不代表高负载无损验收通过。需要 DC-SYNC0 时，才在命令末尾增加 `--dc`，首次仍用 1 ms。

## 5. 接入自己的应用

应用包含 `dmecat/dmecat.h`，CMake 只链接 SDK 的公开目标，不需要 SOEM/IgH 头文件或导入库：

```cmake
cmake_minimum_required(VERSION 3.20)
project(my_can_app LANGUAGES C)
set(CMAKE_C_STANDARD 11)
find_package(dmecat CONFIG REQUIRED)
add_executable(my_can_app main.c)
target_link_libraries(my_can_app PRIVATE dmecat::dmecat)
if(MSVC)
    target_compile_options(my_can_app PRIVATE /utf-8)
endif()
```

配置时传 `-DCMAKE_PREFIX_PATH=<SDK解压目录>`。非 CMake 的 Windows 工程添加 `include` 搜索路径、链接 `lib/x64/dmecat.lib`，运行时部署两个 DLL。完整调用可参考包内 `examples/basic.c` 和 `examples/basic.cpp`。

调用顺序：初始化配置 → `open_ex` → 配置位率 → 显式使能通道 → 注册回调 → `start` → 提交帧/处理回调 → `stop` → `close`。位率配置会停止相应通道，之后需要重新使能；配置保存仅由显式 `save_config` 触发。

接入时注意：

- 通道编号为 `0..3`；DLC 是编码，不是字节数。完整字段规则见包内 README 和公共头。
- SDK 内部启动周期和交付线程，应用不需要自己轮询 EtherCAT；按 `frame.channel` 提交到对应通道队列。
- 提交成功只表示 SDK 本地入队，不是 CAN 已上线。队列满时保留旧帧、拒绝新帧；批量提交必须检查 `accepted`，只重试未接纳的后缀。
- 回调运行于交付线程，不是 UI 线程；帧指针只在回调期间有效，需要保留就复制。不要在回调里调用本句柄的 `start/stop/close` 或重配通道。
- `stop` 不是 flush，不保证从站在途 CAN 已完成；关闭句柄前，先停止其他业务线程访问。

## 6. 文档和 AI Skill

解压包内提供完整的使用资料：

| 文件 | 用途 |
| --- | --- |
| `README.md` | 帧字段、完整接入顺序、回调和队列、错误处理 |
| `docs/QUICK_START.md` | 部署与示例运行 |
| `docs/ABI_V1.md` | 公共接口、结构布局和线程约束 |
| `docs/KNOWN_ISSUES.md` | 已知问题和使用限制 |
| `docs/RELEASE_NOTES.md` | 支持平台、版本功能与验证范围 |
| `licenses/` | 第三方许可和分发边界 |

[配套 AI Skill](0.3.0/skills/dmecat-sdk-consumer/SKILL.md) 也包含在压缩包内。把 Skill 和 SDK 解压路径交给 AI，可帮助其按实际公共头接入应用。例如：

> 使用这份 Skill 为我的 C++ 应用接入四通道收发。SDK 位于我提供的本地目录，先采用 SM/1ms，只编译和枚举，不打开设备、不发送 CAN。

## 使用限制与许可

当前不提供任意 SDO、同步阻塞收帧、等待 CAN 完成的 flush、自动断线恢复或完整 C# 封装。`get_sync_status.active` 不等于 DC_LOCKED；Windows 普通调度不保证硬实时。已知问题见包内 `docs/KNOWN_ISSUES.md`，兼容性与验证范围见 [0.3.0 版本说明](0.3.0/RELEASE_NOTES.md)。

使用或再分发前请阅读包内 `licenses/`，并向供应方确认 SDK 及第三方组件的适用条款。SOEM 为 GPLv3 / 商业双许可，Npcap 不随包分发；下载文件或动态链接不代表获得额外授权。

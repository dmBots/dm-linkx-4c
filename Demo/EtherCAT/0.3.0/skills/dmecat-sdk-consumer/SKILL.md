---
name: dmecat-sdk-consumer
description: 使用 LinkX4C EtherCAT SDK 0.3.x 发行包的公共 C ABI 1，为 C/C++/C# 应用接入四通道 CAN/CANFD 收发、配置、回调和背压处理，并排查部署与调用问题。适用于 Windows SOEM 2.0.0 或 Linux IgH 1.5.4，不用于 SDK 或从站固件的实现开发。
---

# LinkX4C SDK 应用接入

帮助用户将 SDK 接入自己的应用。本 Skill 不包含二进制；SDK 路径由用户提供，或从应用已有的 CMake/依赖配置定位。

## 接入前

1. 核对 SDK 的 `manifest.json`、公共头 `include/dmecat/dmecat.h`、`dmecat_types.h` 和 `dmecat_version.h`，确认版本、ABI、平台和后端。缺少发行包时请求本地路径，不编造接口。
2. 阅读包内 `README.md` 和 `docs/KNOWN_ISSUES.md`；布局/线程问题查 `docs/ABI_V1.md`，部署问题查 `docs/QUICK_START.md`，完整收发和批量后缀重试参考 `examples/basic.cpp`。
3. 确认应用语言、平台、通道及验证范围。默认 SM / 1ms；用户明确指定 DC 时遵从，不为普通收发故障擅自改变同步模式。头文件与文档不一致时报告差异，不猜测兼容。

## 部署和验证

- 应用包含 `dmecat/dmecat.h`，CMake 使用 `find_package(dmecat CONFIG REQUIRED)` 并链接 `dmecat::dmecat`。不引入 SOEM/IgH 头、不装配 PDO/SDO，不另建 EtherCAT 周期循环。
- Windows x64 部署同包的 `dmecat.dll` 和独立 `soem.dll`，只支持 SOEM 2.0.0；另装 Npcap 兼容支持和 MSVC x64 Release 运行时。Linux x86_64 使用 IgH 1.5.4 / API 1.5 及匹配内核配置，非默认前缀通过 `DMECAT_IGH_ROOT` 指定。
- 先编译、检查布局和加载，再运行 `--list`。枚举不代表具有打开权限；系统加载器在 API 前失败时，优先检查依赖和搜索路径。
- `open_ex` 会切换设备模式和 AL 状态；带设备路径运行示例还会配置 CAN 并发帧。仅在用户要求实机操作且设备、参数和范围明确时执行；目标不明确时停在编译/枚举。
- 设备占用或同步失败时保留错误并结束该次测试，不自动抢占其他主站、持久修改权限、重启服务、烧固件或保存参数。

## 调用顺序和线程

```text
enum → init_open_config + init_open_options → 填 adapter → open_ex
     → 配置/回读位率 → 显式使能通道 → 注册回调
     → start → submit/submit_batch + 统计/同步状态
     → 停止应用生产者 → stop → 可重新配置/start → close(&handle)
```

- 初始化函数填写默认值、struct_size 和 ABI；其他带 struct_size 的输出先清零并填 sizeof，保留字段为 0。通道为 0..3，从站位置默认 1；设备路径使用本机枚举结果。
- open 不自动使能 CAN。默认位率 CANFD 1M@5M，BRS 按帧选择；设置位率会停对应通道，之后显式使能。配置和回调注册仅在 STOPPED，生命周期操作串行。
- 配置可能延后生效，示例的 1100ms 等待不是通用完成通知；自动 TDC 回读可为实际值。`save_config` 会持久化，用户未要求保存时不调用。
- SDK 启动周期和交付线程；多个业务线程可提交/读统计，close 前停止其他访问者，不能并发 close 或使用已释放句柄。
- 回调仅快速处理/复制到应用有界队列；frames 指针仅在回调内有效。不抛异常；允许 submit/读统计/读同步缓存，禁止本句柄 start/stop/close/配置/更换回调。上下文及 C# 委托保持到 stop 成功返回，close 后才能卸载库。
- stop 取消本地待发，不是 flush。FAULTED 后先 stop 回收资源，再由应用决定是否重启；不在后台实现无界自动重连。

## 帧和背压

- 帧为 pack=8、88B 内存结构，payload 偏移 24。发送先清零；channel 为 0..3；标准 ID≤0x7FF，扩展 ID≤0x1FFFFFFF 且 ide=1；经典帧 fdf/brs=0，FD 按需设置 fdf/brs。
- dlc 是编码，dlen 是字节数：0..8 对应 0..8B，9..15 对应 12/16/20/24/32/48/64B；不能把 64 填进 dlc。RTR 只用于经典 CAN，dlen=0。esi/seq/reserved0 发送为 0。
- 即时发送使用 ts_valid=0、timestamp_us=0。读取接收时间戳先检查 ts_valid，不把设备时间当主机墙钟。DC 配置成功不代表已验证定时 CAN 发送。
- 按 frame.channel 提交，各通道保持 FIFO；SDK 按预算、credit、配额和 round robin 聚合，跨通道无全局顺序保证。应用不管理 ACK 或绕过 credit。
- TX 满时保留旧帧、拒绝新帧，返回 QUEUE_FULL。submit_batch 的 accepted 必填，非零返回也可能已有成功前缀；只重试未接纳后缀。仅对背压做有界等待，不忙等、不循环重试非法帧或 STATE。
- submit 成功只是本地复制，tx_submitted 不是 CAN 完成计数。交付确认按实际接收/应答核对通道、ID、DLC 和完整载荷，不靠固定睡眠或 stop 确认送达。
- RX 队列满时不部分接纳窗口；ACK 不等于业务已处理或持久化。应用队列也需要背压处理。未注册回调计 rx_unhandled，不缓存等待同步 read；本版无同步阻塞读接口。

## 同步、跨语言和排错

- 显式 DC 使用初始化后的 options.sync_mode=DMECAT_SYNC_DC；周期 125..1000us、相移绝对值小于周期，并满足 0<rx_timeout_us<period_us。不能带着默认 500us 超时配置 250us 周期。改 SM/DC 需 close 后重开。
- get_sync_status.active 不是 DC_LOCKED 或相位误差；Windows 不保证硬实时。get_device_info.revision 是 SII 修订号，不是固件构建号。
- FFI 保持定宽整数、C linkage/cdecl、pack=8、指针宽度和 int32 错误码。句柄仅由 SDK close 释放，不自行 free，不跨库传 STL 或托管对象。
- C# 以 `examples/csharp/Program.cs` 布局为起点，使用 CallingConvention.Cdecl 和 StructLayout(Pack=8)，保活委托及 user_data；它不是完整 C# 绑定。
- 每个 API 检查返回值，失败后同线程立即 get_last_error，保留 code/message/sdo_abort。成功不清除旧错误；异步问题读取 stats 的 worker_errors/wkc_bad、队列水位、credit 和接收背压。每次成功 start 重置本轮统计。
- 保留 KNOWN_ISSUES 中的数据完整性和稳定性限制；不因编译、枚举或短时示例成功宣称无损/长稳通过，不添加公共头未提供的 SDO、flush 或自动恢复接口。
- 交付应用代码、依赖和运行方法，明确验证到了编译、枚举还是实际收发，以及未验证项。问题日志只包含必要信息，移除凭据与不相关的本地路径。

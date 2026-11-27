# ESP32-DIV 1.9.4 发布说明（卡顿治理 / 共存守卫 / 抓包池分级）

## 1. 基线声明

- 版本：`v1.9.4`（`shared.h` 的 `ESP32DIV_VERSION`）。
- 发布点：2026-11-26。
- 板型：ESP32-DIV V2（ESP32-S3，16 MB Flash）。
- 构建参数与 1.9.3 一致：`esp32:esp32:esp32s3:FlashSize=16M,PartitionScheme=default_8MB,PSRAM=disabled,USBMode=hwcdc,CDCOnBoot=cdc,FlashMode=dio`。
- 本轮重点：WiFi/BLE 共存与 netif 所有权收敛、WiFi Deauther 空闲复核卡顿治理、Packet Monitor 抓包池按需分级、Sub-GHz raw 邻档判据、HaleHound Stations 容量提升、Settings 标签调整。

## 2. 本轮改动

### 2.1 WiFi 共存与接口所有权

- 新增共享 WiFi 状态守卫：拉起射频前先检查 `WIFI_AP_DEF` / `WIFI_STA_DEF` 是否已存在，存在则复用，不再让 Arduino core 重复 `esp_netif_new`。消除反复出现的 `esp_netif_new: ... duplicate key` 报错，以及裸 `esp_wifi_init()` 之后驱动起不来的情况。
- modem sleep 改为按 BLE 状态选择：BLE 栈已初始化时绝不设置 `WIFI_PS_NONE`，消除 WiFi+BLE 共存管理器 `Should enable WiFi modem sleep when both WiFi and Bluetooth are enabled` 断言。
- Beacon Spammer、Deauther、Probe Flood 在注入前保证 AP 接口有效，持续出现的 `80211_tx rc=257` 丢帧随之消失。
- 前台 AP 扫描改为异步 + 12 秒上限；驱动状态异常时主循环不会永久卡死、按键全部失效。

### 2.2 WiFi Deauther 卡顿治理与单信道快探复核

- 空闲信道复核先做单信道快探：阻塞式 60–120 ms 驻留扫描，命中 BSSID 即返回当前信道，单次约 0.4 秒；未命中才退回全信道扫描兜底。
- 探测信道取自已选中 AP 的同一条记录（`selectedAp.primary`），全扫确认后回写记录，避免记录中的错误信道导致每轮都退化全扫。
- 复核计时改为以“功能入场”和“扫描完成”为起点；STOP 后把复核时间戳推后 15 秒，不再出现 STOP 后立刻全扫、额外冻结约 10 秒。
- 退出请求优先于信道复核：按下退出后不再等待当前扫描跑完。
- 结果：周期性 8–10 秒主循环冻结消失，空闲可操作窗口回到 14.5 秒以上，Stop 到退出小于 1 秒。

### 2.3 Packet Monitor 抓包池按需分配

- 抓包槽位池改为两档：10 槽（23,620 B）要求分配后 free 与最大连续块都不低于 24 KB；不满足自动降为 6 槽（14,172 B），要求不低于 8 KB。
- 两档都满足不了时弹窗提示所需最小缓冲并拒绝进入抓包，不做“抓到一半没缓冲”的半初始化降级。
- 队列深度、槽位越界判断、释放字节数全部跟随本次实际分配的档位。
- 退出或失败时按实际分配大小释放，不再固定按 10 槽释放。

### 2.4 Sub-GHz raw 邻档判据

- raw 波形判据仍按 25 µs 脉宽分档统计 top1–top3，但把每个主档的 ±1 相邻档一并计入覆盖率，解决真实脉宽因抖动跨档（例如 ~500 µs 同时落进 500/525 两档）被硬切成两半而低于阈值的问题。
- 同时增加 top1 占比下限：覆盖率 ≥60% 且 top1 ≥20% 才判定为有效信号，防止能量分散的噪声靠“宽覆盖”蒙混过关。
- raw 抓取改为最多 6 片 × 400 ms 连续抓取，任一片边沿数 ≥32 即采用并停止；每片独立统计，不跨片拼接，避免片边界产生假脉宽。
- 实测（315 MHz 短按遥控器，Garage-Car 315 预设）：第 1 片 400 ms 即命中，`edges=78 changes=78`，形态 `raw shape n=78 m=57 avg=360 top=87:22%/37:21%/137:10% cov=68% top1=22% ok=1`，邻档判据放行、未进入后续片。抓取与判据这一半实测通过。
- 目标端回放效果仍无法验证：测试钥匙为 2014 款宝马 5 系滚动码，机制上不可重放。固定码（车库门/卷帘门）遥控器的目标端回放留到 1.9.4 修正版补测；若捕获后无法发射，优先改用保存的配置文件（Saved Profile）重放。

### 2.5 HaleHound WiFi Stations

- 站点表容量从 64 槽提升到 96 槽，跳频间隔从 250 ms 调整为 350 ms，单轮扫频可保留更多客户端。
- 满槽淘汰策略不变：无过滤时优先淘汰没有 AP 证据的条目，让真实客户端在 96 槽里存活更久。

### 2.6 Settings

- 第 6 行 `Background Scan` 更名为 `Idle AutoScan`，行为不变（仍在主菜单空闲时轮询 WiFi/BLE，约 15 秒一次）。

### 2.7 存储与 SD

- SD File Manager 的目录列表清理统一收进分发层退出钩子，按键退出与触摸导航栏切换共用同一条路径，`entries` 一定会被释放。
- SD 挂载失败日志增加每次尝试耗时：`<5 ms` 指向库层，`100–1000 ms` 指向卡侧（供电、触点、CMD8/ACMD41 协商）。
- 移除 Sub-GHz 里重复的 `ff_diskio` 槽位修复路径；槽位回收只保留在共享 `spiEnsureSD()` 的失败分支，且每次插拔只触发一次。

### 2.8 DroneX

- WiFi/BLE 共存与 netif 所有权收敛后，DroneX 进入后的空闲堆保持在 45 KB 以上，为网页服务、地图与 RID 解码留出余量。

## 3. 已知问题与边界

1. **Deauther 最坏 15 秒空窗**：空闲复核周期为 15 秒。若路由器在两次复核之间自动跳信道，deauth 会在旧信道上继续发送，最长约 15 秒无效；下一轮复核发现后自动校正。属当前设计边界，不是卡死。
2. **Packet Monitor 写入丢包约 29%**：PCAP 写入 SD 时，约 29% 的抓包槽位会因 SD 写入速度跟不上而被丢弃。瀑布图与统计不受影响，抓包可用于协议分析，但不保证逐帧完整。
3. **raw Replay 目标端效果未验证**：多片短窗口抓取（6 × 400 ms）与邻档判据已用真实 315 MHz 遥控器实测通过（首片 78 edges、cov=68%、top1=22%）；但测试目标为 2014 款宝马 5 系滚动码，机制上不可重放，目标端回放效果留待 1.9.4 修正版用固定码遥控器补测。
4. **Station Scanner 站点数仍偏少**：96 槽 / 350 ms 已提升保留量，但扫到的客户端数量仍明显少于 1.9.2，根因继续排查。
5. **WebUI `/m` 大目录卡顿**：历史目标或轨迹条目很多时 `/m` 页面刷新变慢，需定期清理 `/dronerx/` 或减少并发客户端。
6. **BLE Rubber Ducky 退出后可用堆偏低**：约 23.4 K / 最大连续块 8.2 K（进场约 27.6 K / 18.4 K）。原因是 NimBLE-Arduino 1.4.2 每删除一个服务都会重建一次 GATT，属固有代价、非泄漏。
7. **部分机器 TF 卡挂载不稳定**：本轮只增加了每次尝试的耗时日志用于分层定位，根因尚未关闭。
8. 沿用项：手机首次连接 DroneX 热点掉一次；DroneX 模拟目标每轮约 10 秒空档；Wardriver 之后进入 RID Scanner 会退化为 BLE-only，需退出重进或重启。

## 4. 发布产物与构建记录

- 构建：2026-11-26，Arduino IDE 内置 arduino-cli + arduino-esp32 2.0.10-cn，FQBN 与 1.9.3 一致。
- Flash 2,739,440 B（82%）；`shared.h` 版本号 `v1.9.4`。
- `ESP32-DIV-V2-v1.9.4-app.bin`：2,739,440 B，SHA-256 `f1b477a053344ee24fb704e597aa8f3f4baba3d3ffdbc8a56f8e99549ae6ca1f`
- `ESP32-DIV-V2-v1.9.4-full-16MB.bin`：2,804,976 B，SHA-256 `e6087fc0e0a7fcb615855afce42689db6072fa79480335db7ab59e9986d91ad6`
- `sha256sums.txt`：194 B。
- tag `v1.9.4`。
- 本轮只上传二进制与 `sha256sums.txt`，不上传源码、不提交源码。

## 5. 未验证 / 未覆盖项

- raw Replay 的多片抓取与邻档判据已用 315 MHz 遥控器实测通过；目标端回放效果（固定码遥控器）未验证。
- Deauther 在真实路由器自动跳信道场景下的 15 秒空窗仅按设计推导，未做长跑计时。
- Packet Monitor 低堆下自动降到 6 槽的分支未在实机上强制复现。
- HaleHound Stations 96 槽 / 350 ms 的长跑与多客户端并发未复测。

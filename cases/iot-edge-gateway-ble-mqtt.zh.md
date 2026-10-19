# 案例研究：IoT 边缘网关 — 通过 4G 把 BLE 数据送上云

一个 ESP32-S3 网关，负责收 BLE 传感器的广播，解析成读数，再经 4G 模组发到 ThingsBoard 服务器。它用在没 WiFi、附近常常也没电源的现场，所以大部分时间睡觉，靠定时器醒来干一阵。

> 🌐 英文版 / English version: [iot-edge-gateway-ble-mqtt.md](iot-edge-gateway-ble-mqtt.md)

---

## 背景

一个现场有一堆靠电池供电的 BLE 传感器，但没有办法集中采集和监控。给每个都拉线、或者都配张蜂窝网卡，成本上根本划不来。他们需要的是一个小盒子，往那一放，把 BLE 数据收上来，隔一阵往云上推一次。

---

## 问题

- 只有 BLE 的传感器自己连不上网。
- 看不到设备的状态、电量、上次在线时间。
- 靠人去现场看，又慢又漏。
- 这盒子得能跑在没 WiFi、有时连电都不好接的地方。
- 某个传感器掉线了得能被发现，不能悄悄丢。

---

## 约束

| 约束 | 细节 |
|------|------|
| 供电 | 靠电池，所以深度睡眠 + 定时器唤醒 |
| 成本 | 一个 ESP32-S3 + 一个 4G 模组带很多传感器 |
| 网络 | 只有 4G，还常断；得缓冲加重试 |
| 现场维护 | 现场没人给重新烧录；配置和固件都走空中 |
| 规模 | 一个网关带很多 BLE 节点，各自按类型 + MAC 区分 |

---

## 方案

```
┌──────────┐ BLE ┌──────────────┐  parse  ┌────────────┐  4G/MQTT  ┌──────────────┐
│ BLE      │────►│ ESP32-S3     │────────►│ ThingsBoard│─────────►│ Cloud dash.  │
│ sensors  │     │ scan+buffer  │         │  (MQTT)    │           │ + alerts     │
└──────────┘     │ + RTC + 4G   │         └────────────┘           └──────────────┘
                 └──────┬───────┘
              deep-sleep│timer wake
                 ┌──────▼───────┐
                 │ FFat 环形    │  队列游标存在 RTC 内存里，
                 │ 缓冲(5个)    │  睡过去/重启都不丢
                 └──────────────┘
```

- **扫描**（`BluetoothScanner`）：按"设备类型 + MAC"的白名单扫，白名单存在 Flash 里。不在名单的直接忽略。它还会数每个已知 MAC 漏了几次，哪个传感器挂了一眼就知道。
- **读数**（`parseManufacturerData` + NTC 校准表）：BLE 的厂商数据经过一张校准表变成温度读数。
- **缓冲**（FFat 环形缓冲，五个文件 `stack1`–`stack5`）：结果进队列，读写游标存在 `RTC_DATA_ATTR`，所以深度睡眠和重启都丢不了；4G 模组一直关着。
- **攒批发**（TinyGSM + ThingsBoard MQTT）：只有队列攒到 `report_count` 条，模组才上电、连上、把积压的读数一次性发出去。
- **设备状态**：电池电压、环境温度和芯片温度、启动次数作为客户端属性上报；每条读数都带 RTC 时间戳。
- **远程管理**：固件走 ThingsBoard OTA 下发；`sleep_seconds`、`report_count`、broker、BLE 扫描参数全都通过共享属性更新下来。不用重烧、不用跑现场。
- **现场配网**：开机按住某个脚，设备就变成 WiFi AP，带个简单文件浏览器把 `config.json` 丢进去就行。之后一切走空中。
- **时间**（外挂 DS1302 RTC）：每次模组连上就跟网络对一次时，对不上就退回编译时时间。

把主循环缩一下大概是这样：

```cpp
// 示意，不是实际交付代码
std::vector<BLEDeviceInfo> devices = scanner.startScan(scan_interval, scan_window);
if (!devices.empty())
    queue.push(generateTemperatureJson(devices, rtcNow()));

if (queue.size() >= report_count) {      // 缓冲满了 → 唤醒模组
    modem.connect();
    tb.connect(broker, token);
    while (!queue.empty()) tb.sendTelemetry(queue.pop());
    tb.loop();                           // 等 OTA / 远程配置
}
enterDeepSleep(sleep_seconds);
```

---

## 同一套底座，换传感器和链路

这不是一次性做出来的。同一套 ESP32-S3 固件底座后来被改去跑过好几个现场，每次只换传感器和回传链路：

| 变种 | 传感器 | 回传 | 备注 |
|------|--------|------|------|
| BLE 网关（本案例） | BLE 信标（温度等） | 4G → ThingsBoard MQTT | OTA + Web AP 配网 |
| 激光测距 | 激光测距（TTL/485/ASCII） | 4G → MQTT | 载荷加密，本地 WiFi 配网 |
| 振弦式振动 | 振动（振弦式） | 4G → HTTP | SM4 加密，Flash 缓冲续传 |
| 振动 | 振动 | WiFi → MQTT | 带 BLE，载荷加密 |
| LoRa 节点 | 通用传感器 | LoRa（RA-08 模组） | 远距离、低功耗链路 |

每次不变的东西：ESP32-S3 + Arduino，一份存着传感器标定参数的配置文件，RTC 时间，一个能扛住链路中断的本地存储缓冲，还有一个 WiFi 网页用来现场配网。变的只是传感器驱动和传输客户端。所以新现场能很快跑起来，不用从零重来。

这些板子就是这套家族的实体——全部走嘉立创打板。ESP32-S3 核心板是被复用的网关底座；其余是它对接的传感器或变种节点。

### ESP32-S3 核心板 — 网关本体

| 原理图 | PCB 布局 |
|--------|----------|
| ![ESP32-S3 核心板原理图](images/ESP32S3CoreBoardSCH.svg) | ![ESP32-S3 核心板 PCB](images/ESP32S3CoreBoardPCB.png) |

### BLE 传感器板

| 原理图 | PCB 布局 |
|--------|----------|
| ![BLE 传感器原理图](images/BLESensorSCH.svg) | ![BLE 传感器 PCB](images/BLESensorPCB.png) |

### RA-08 LoRa 板

| 原理图 | PCB 布局 |
|--------|----------|
| ![RA-08 原理图](images/RA08BoardSCH.svg) | ![RA-08 PCB](images/RA08BoardPCB.png) |

### 振动监测板

| 原理图 | PCB 布局 |
|--------|----------|
| ![振动板原理图](images/vibrationBoardSCH.svg) | ![振动板 PCB](images/vibrationBoradPCB.png) |

---

## 关键技术决策

### 1. 缓冲进 FFat，攒满再批量发
4G 链路是最贵、最费电的部分。让模组一直关着，只有队列满了才醒，既省流量钱又省电池；队列游标放 `RTC_DATA_ATTR`，半路重启也不会丢数据。哪次连不上，也只是下一批晚点发。

### 2. 配置走 ThingsBoard 下来，不靠烧录
这些盒子一装下去，没人想再拆开。睡眠时间、扫描窗口、上报阈值，连 broker 都能通过共享属性改。那个配网脚只用于第一次开通。

### 3. 白名单按"类型 + MAC"，存进 Flash
先把所有广播都收进来再筛，纯属浪费那点短暂的唤醒窗口。扫描时就拿已知列表比对，BLE 这一段能很快过完；而漏报计数器正好告诉看板哪个传感器没动静了。

### 4. 外挂 RTC，跟网络对时
没时钟的话，发到服务器的批次就没有时间序。DS1302 每次连上就跟网络时间同步（对不上就退回编译时），不用 GPS、也不用一直连着，每条记录都有可用时间戳。

### 5. 深度睡眠是默认状态
两次采集之间，模组和大部分 GPIO 都断电，只跑扫描、偶尔跑一次 4G 发送。电池寿命就是从这份占空比里来的，不是靠什么射频花活。

---

## 我的收获

有意思的部分不在 BLE 也不在 MQTT，那俩都是库。难的是占空比：尽量多睡觉、缓冲又不能把 Flash 写坏、网络掉了还不能丢数据。电池寿命和可靠性其实都是从这儿来的。

---

## 标签

`#embedded` `#esp32` `#ble` `#4g` `#mqtt` `#thingsboard` `#iot`

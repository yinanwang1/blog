---
title: 移动端BLE项目架构设计方案(Flutter版)
date: 2026-09-17 00:35:11
tags:
---


项目的数据获取和传输如下：

```text
智能戒指 / 手环 / BLE设备
        ↓
      手机App
        ↓
   本地数据库
        ↓
     云端服务器
```

架构核心的规则：

1. UI 不直接操作 BLE、数据库和 HTTP。
2. BLE 只负责传输 byte，不理解心率、睡眠等业务。
3. Protocol 专门负责设备协议。
4. Domain 负责业务规则，不关心具体技术实现。
5. Repository 隔离业务与数据来源。
6. Device Layer 专门处理外部硬件设备。
7. 设备数据先落本地，再异步上传服务器。
8. Riverpod 负责状态传播和 UI 更新，不承担底层业务。
9. pp / DI 是组装中心，可以知道所有模块。
10. 各层避免双向依赖和循环依赖。
11. 公共能力不要全部塞进一个巨大 Core。
12. 小项目先用文件夹分层，复杂以后再拆 Package。

## 一、整体架构

整个项目可以设计成：

```text
                         App （整个项目的组装中心）
                 Composition Root / DI
                          │
          ┌─────────┼────────┐
          ↓               ↓               ↓
   Presentation         Device           Data
          │              │               │
          ↓               │              │
        Domain ←──────┴────────┘
          │
          ↓
     Repository Interface


基础能力：
Network / Database / Log / Security / Utils
```

App 可以知道：

```text
Presentation
Domain
Device
Data
Platform
```

因为它的责任就是：**把所有模块组装起来。**

这在架构里通常叫：**Composition Root**。



架构的分层：App 负责组装，Presentation 负责展示，Riverpod 负责状态，Domain 负责业务，Repository 负责边界，Device 负责设备，Protocol 负责协议，Data 负责数据，Sync 负责同步，Platform 负责系统能力。

从运行时业务流程来看，则是：

```text
Presentation  （UI界面）
      ↓
    Domain  （业务逻辑处理）
      ↓
 Repository  （数据仓库，管理数据来源）
   ↙      ↘
Device    Data
  ↓        ↓
 BLE    DB / HTTP
```

这里要注意： **“代码依赖方向”和“业务调用方向”并不一定完全相同。**

## 二、Riverpod  状态管理工具

Riverpod 负责：**状态管理 + 状态传播 + UI 更新。**

例如：

```text
Device Layer
      ↓
Repository
      ↓
Domain
      ↓
Riverpod Notifier / Provider
      ↓
UI
```

连接状态：

```text
Disconnected
      ↓
Connecting
      ↓
Connected
      ↓
Synchronizing
      ↓
Riverpod
      ↓
页面刷新
```

健康数据：

```text
BLE
 ↓
Protocol
 ↓
Local DB
 ↓
Repository
 ↓
Riverpod
 ↓
HealthPage
```

Riverpod 官方也把 Provider 作为状态描述、缓存和共享状态的重要机制。

但：

```text
BLE重连
协议解析
数据库操作
HTTP
CRC
后台任务
```

不要塞进 Riverpod。

## 三、Presentation Layer：表现层

页面展示和用户交互，主要负责：

```text
Page
Widget
页面状态
用户操作
Loading
Error
数据展示
```

建议 Riverpod 主要工作在这里。

结构例如：

```text
presentation/
│
├── home/
├── device/
├── health/
├── sync/
└── settings/
```

例如设备页面：

```text
DevicePage
     ↓
DeviceNotifier   （Riverpod管理）
     ↓
ConnectDeviceUseCase
```

Presentation 不应该直接：

```text
操作 BLE
解析 byte[]
操作 SQLite
发送 HTTP
计算 CRC
```

## 四、Domain Layer：业务层

Domain 负责：**描述这个 App 到底要做什么。**

例如：

```text
domain/
│
├── device/
│   ├── ConnectDeviceUseCase
│   ├── DisconnectDeviceUseCase
│   └── SyncDeviceUseCase
│
├── health/
│   ├── GetHealthDataUseCase
│   └── CalculateHealthSummaryUseCase
│
└── sync/
    └── SyncHealthDataUseCase
```

例如：

```text
“同步戒指数据”
```

Domain 只关心：

```text
连接设备
↓
获取数据
↓
保存数据
↓
同步服务器
```

它不应该关心：

```text
BluetoothGatt
CoreBluetooth
Characteristic UUID
HTTP具体URL
SQLite具体SQL
```

## 五、Repository：数据访问边界

Repository 是非常重要的一层。

例如：

```text
DeviceRepository
HealthRepository
SyncRepository
```

业务层只知道：

```text
DeviceRepository
```

不知道后面到底使用：

```text
BLE
Wi-Fi
USB
Mock Device
```

例如：

```text
Domain
│
└── DeviceRepository
        ↑
        │ implements
        │
Device
│
└── BleDeviceRepository
```

`BleDeviceRepository`实现了`DeviceRepository`, 在`App`实例化`BleDeviceRepository`后将实例赋值给`Domain`的接口变量。那么`Domain`则可以直接使用`DeviceRepository`所有接口的调用。从而`Domain`和`BleDeviceRepository`解耦了。

以后设备通信方式变化：

```text
BleDeviceRepository
       ↓
WifiDeviceRepository
```

Domain 不需要修改。

## 六、Device Layer：设备层

这是 IoT 项目与普通 App 最大的区别。

```text
device/
│
├── manager/
│   └── DeviceManager
│
├── connection/
│   └── ConnectionManager
│
├── ble/
│   └── BleTransport
│
├── protocol/
│   ├── Command
│   ├── Packet
│   ├── Encoder
│   ├── Decoder
│   └── Parser
│
├── adapter/
│   ├── BonaDeviceAdapter
│   └── OtherDeviceAdapter
│
└── state/
    └── DeviceState
```

### **DeviceManager**

作为设备系统统一入口：

```text
扫描
连接
断开
自动重连
同步
发送命令
查询状态
```

上层最好只面对：

```text
DeviceManager
```

而不要知道：

```text
Service UUID
Characteristic UUID
Notify
Write
MTU
```

**这些属性应该封装在 Device Layer 内，其中通用 BLE 能力放在 BLE 模块，厂商特定 UUID 放在 Adapter/GATT Profile，厂商数据格式和命令放在 Protocol 模块。**

## 七、BLE Transport：只负责运输

BLE 层最好非常“纯”。

负责：

```text
Scan
Connect
Disconnect
Discover
Read
Write
Notify
Connection State
```

它看到的是：

```text
byte[]
```

不应该看到：

```text
心率
睡眠
血氧
步数
体温
```

例如：

```text
智能戒指
    ↓
BLE
    ↓
01 AF 02 32...
    ↓
Protocol
    ↓
HeartRateData
```

BLE 解决：**数据怎么传。**

Protocol 解决：**数据是什么意思。**

## 八、Protocol：协议层

设备厂商通常会定义类似：

```text
Header
Command
Length
Payload
Checksum
```

所以：

```text
protocol/
│
├── Packet
├── Command
├── Encoder
├── Decoder
├── Parser
└── Checksum
```

发送：

```text
业务指令
   ↓
Command
   ↓
Encoder
   ↓
byte[]
   ↓
BLE
```

接收：

```text
BLE
 ↓
byte[]
 ↓
Decoder
 ↓
Packet
 ↓
Parser
 ↓
业务数据
```

UI 永远不应该出现：

```text
data[3]
0xAF
0x01
```

这些全部属于 Device / Protocol。

## 九、Device Adapter：不同厂商适配

考虑 IoT 产品以后可能更换硬件厂商：

```text
                DeviceRepository
                       │
                DeviceAdapter
                       │
       ┌───────────────┼──────────────┐
       ↓               ↓              ↓
    BonaAdapter     VendorB        VendorC
       │               │              │
   Protocol A      Protocol B      Protocol C
```

业务统一调用：

```text
connect()
sync()
getBattery()
getHeartRate()
```

至于具体厂商协议，上层完全不知道。

这样以后：

```text
Bona Ring
    ↓
更换供应商
```

主要改：

```text
Adapter
+
Protocol
```

而不是整个项目。

## 十、Data Layer：数据层

Data Layer 处理：

```text
本地数据库
服务器 API
缓存
持久化
数据转换
```

例如：

```text
data/
│
├── local/
│
├── remote/
│
├── repository/
│
└── entity/
```

Data Layer 主要由 Repository 和 Service 等部分构成，用于隔离数据库、平台插件和远程 API 等数据来源。

## 十一、本地数据库设计

这里应该明确使用：**数据库表**

例如：

```text
Local Database
│
├── device
├── health_record
├── sync_task
├── device_event
└── device_log
```

### **device**

设备信息：

```text
device_id
device_name
device_type
mac / identifier
firmware_version
bind_time
```

### **health_record**

设备产生的数据：

```text
id
device_id
type
device_time
receive_time
value
sync_state
```

### **sync_task**

同步任务：

```text
id
record_id
status
retry_count
create_time
last_retry_time
```

### **device_event**

例如：

```text
连接
断开
电量变化
同步开始
同步结束
固件升级
```

### **device_log**

用于 IoT 问题排查：

```text
时间
设备
命令
数据
方向
结果
异常
```

## 十二、数据库 Table 与 Model 要分开

一个数据可能经历：

```text
BLE byte[]
    ↓
Protocol Model
    ↓
Domain Model
    ↓
Database Entity
    ↓
Database Table
    ↓
Network DTO
    ↓
Server
```

例如：

```text
HealthRecord
```

在不同层可能有不同形式：

```text
Device：
HeartRatePacket

Domain：
HeartRate

Database：
HealthRecordEntity

Network：
HealthRecordDTO
```

不要为了省几个类，让所有层共享一个巨大的 Model。

## 十三、Local First

IoT 项目非常建议采用：**先落本地，再同步服务器。**

不要：

```text
BLE
 ↓
直接上传服务器
```

推荐：

```text
BLE
 ↓
Protocol
 ↓
Local DB
 ↓
标记 Pending
 ↓
SyncEngine
 ↓
Cloud
```

原因：

```text
BLE正常，但是网络断了

有网络，但是服务器超时

App进入后台

上传中途系统挂起

服务器临时不可用
```

数据仍然不会轻易丢失。

## 十四、Sync Engine：独立同步模块

```text
sync/
│
├── SyncEngine
├── SyncScheduler
├── UploadQueue
├── RetryPolicy
└── SyncState
```

数据流程：

```text
BLE收到数据
    ↓
写Local DB
    ↓
产生Pending记录
    ↓
SyncEngine
    ↓
批量上传
    ↓
Server
    ↓
Success
    ↓
更新Local DB
```

失败：

```text
Uploading
    ↓
Failed
    ↓
Pending
    ↓
Retry
```

UI 不负责上传。

UI 只展示：

```text
最近同步：17:30

待上传：28条
```

## 十五、Platform Layer：系统平台能力

iOS 与 Android 后台能力差异很大，因此应该隔离：

```text
platform/
│
├── bluetooth/
├── background/
├── permission/
├── notification/
└── storage/
```

逻辑：

```text
              Device Layer
                    ↓
             Platform API
               ↙       ↘
             iOS      Android
```

iOS：

```text
CoreBluetooth
Bluetooth Background Mode
State Restoration
```

Android：

```text
BluetoothGatt
Foreground Service
WorkManager
```

上层不应该知道这些平台细节。

## 十六、依赖方向

这是整个架构中最重要的规则之一。

```text
Presentation → Domain

Device       → Domain

Data         → Domain
```

因为：`Domain`定义抽象。 `Device / Data`实现抽象。

最终：

```text
            Presentation
                  │
                  ↓
               Domain
               ↑     ↑
              /       \
          Device      Data
```

而 App：

```text
                App
         ┌──────┼──────┐
         ↓      ↓      ↓
        UI    Device   Data
         \      |      /
             Domain
```

所以不要简单理解成：**上层永远 import 下层。**

更准确的是：**依赖只能按照架构规定的方向流动，并避免循环依赖。**

## 十七、Core 设计

不要建立一个巨大：

```text
core/
```

然后所有模块：

```text
import core.dart
```

否则 Core 一改：

```text
UI
Domain
Device
Data
```

可能全部受到影响。

更推荐：

```text
core/
│
├── common/
├── log/
├── network/
├── database/
├── security/
└── utils/
```

各模块按需依赖。

例如：

```text
Device
├── core_log
└── core_common

Data
├── core_network
├── core_database
└── core_log

Presentation
└── core_common
```

核心原则：**越多人依赖的模块，越应该小、稳定、低变化。**

判断方法：

```text
脱离这个项目，还能不能直接使用？

      ↓
   ┌─────┐
   │ 能      │ → Core
   └─────┘

   ┌─────┐
   │不能     │ → 对应业务模块
   └─────┘
```

Core·适合：

```text
Logger
DateUtils
ByteUtils
通用Result
基础Exception
基础Extension
通用加解密
```

不适合：

```text
RingManager
HeartRateParser
HealthManager
SyncManager
DeviceCommand
```

## 十八、文件夹分层

这个 IoT 项目初期完全没必要一上来拆几十个 Package。

例如：

```text
lib/
│
├── app/
│   ├── app.dart
│   └── di/
│
├── presentation/
│   ├── home/
│   ├── device/
│   ├── health/
│   └── settings/
│
├── domain/
│   ├── device/
│   ├── health/
│   ├── sync/
│   └── repository/
│
├── device/
│   ├── ble/
│   ├── connection/
│   ├── protocol/
│   ├── adapter/
│   └── manager/
│
├── data/
│   ├── local/
│   ├── remote/
│   ├── entity/
│   └── repository/
│
├── sync/
│   ├── engine/
│   ├── queue/
│   └── retry/
│
├── platform/
│   ├── bluetooth/
│   ├── background/
│   └── permission/
│
└── core/
    ├── log/
    ├── network/
    ├── database/
    ├── security/
    └── utils/
```

## 十九、最终架构图

### **连接设备**

```text
DevicePage
    ↓
Riverpod DeviceNotifier
    ↓
ConnectDeviceUseCase
    ↓
DeviceRepository
    ↓
BleDeviceRepository
    ↓
DeviceManager
    ↓
BleTransport
    ↓
Platform BLE
    ↓
智能戒指
```

### **设备数据回来**

```text
智能戒指
    ↓
Platform BLE
    ↓
BleTransport
    ↓
Protocol Decoder
    ↓
Parser
    ↓
Domain Data
    ↓
Local DB
```

然后分成两条：

```text
                   Local DB
                  /        \
                 ↓          ↓
            Repository    SyncEngine
                 ↓          ↓
              Riverpod    Server
                 ↓
                 UI
```



把全部内容放到一张图里：

```text
                        ┌────────────────┐
                        │      App       │
                        │ Composition DI │
                        └───────┬────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ↓                 ↓                 ↓
      ┌──────────────┐   ┌──────────────┐  ┌──────────────┐
      │ Presentation 					│   │    Device    						│  │     Data     				│
      │              │   │              │  │              │
      │ Page         │   │ DeviceManager│  │ Local        │
      │ Widget       │   │ Connection   │  │ Remote       │
      │ Riverpod     │   │ BLE          │  │ Repository   │
      └──────┬───────┘   │ Protocol     │  └──────┬───────┘
             │           │ Adapter      │         │
             │           └──────┬───────┘         │
             │                  │                 │
             └──────────────────↓─────────────────┘
                       ┌────────────────┐
                       │     Domain     │
                       │                │
                       │ UseCase        │
                       │ Domain Model   │
                       │ Repository API │
                       └────────────────┘


              ┌──────────────────────────────────┐
              │           Sync Engine            │
              │ Queue / Retry / Scheduler        │
              └──────────────────────────────────┘

              ┌──────────────────────────────────┐
              │          Platform Layer          │
              │                                  │
              │ iOS                Android       │
              │ CoreBluetooth      BLE           │
              │ Background         FGS           │
              │ Restoration        WorkManager   │
              └──────────────────────────────────┘

              ┌──────────────────────────────────┐
              │          Infrastructure          │
              │                                  │
              │ Log │ Network │ DB │ Security    │
              └──────────────────────────────────┘
```





### **参考文献**

- Flutter 官方：App Architecture Guide
  https://docs.flutter.dev/app-architecture/guide
- Flutter 官方：Common Architecture Concepts
  https://docs.flutter.dev/app-architecture/concepts
- Flutter 官方：Dependency Injection / Communicating Between Layers
  https://docs.flutter.dev/app-architecture/case-study/dependency-injection
- Flutter 官方：Architectural Overview
  https://docs.flutter.dev/resources/architectural-overview
- Riverpod 官方：Providers
  https://riverpod.dev/docs/concepts2/providers  
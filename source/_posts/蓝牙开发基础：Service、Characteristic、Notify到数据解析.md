---
layout: ble
title: 蓝牙开发基础：Service、Characteristic、Notify到数据解析
date: 2026-09-14 22:01:09
tags:
---

记录下蓝牙开发的基础知识，用到的接口名称和注意事项。
App和蓝牙的长连接可以阅读下《Android-App与蓝牙设备长连接的可行性方案》和《iOS-App与蓝牙设备长连接的可行性方案》。有蓝牙设备就可以直接编写一个demo试试。
现在蓝牙在很多穿戴设备上使用，比如智能手环、智能手表、智能体脂秤、耳机、智能门锁等。了解蓝牙开发的相关知识和注意事项，非常的有用。
一个 BLE 设备的构成：

```text
Peripheral （蓝牙设备 ）
│
├── Service （某种功能模块）
│     │
│     ├── Characteristic  （具体状态或数据）
│     │       ├── UUID        （唯一码）
│     │       ├── Properties  （Read / Write / Write Without Response / Notify  / Indicate）
│     │       ├── Value       （真正的数据）
│     │       └── Descriptor  （对这个 Characteristic 的补充描述或配置）
│     │
│     └── Characteristic （另一个具体状态或数据）
│
└── Service （某种功能模块）
      └── Characteristic
```

Characteristic的属性说明：

| **概念**               | **什么作用**                                            |
| ---------------------- | ------------------------------------------------------- |
| UUID                   | 此Characteristic的唯一ID。                              |
| **Properties**         | 此 Characteristic 有什么能力。                          |
| **Value**              | 此 Characteristic 拥有什么数据。                        |
| Permissions / Security | 此 Characteristic 执行操作的条件。如Encryption Required |
| **Descriptors**        | 此 Characteristic 的附加说明/配置。如CCCD 0x2902        |

某种功能模块可以设计成：

```text
Watch Service
UUID: FFF0
├── Command Characteristic
│   UUID: FFF1
│   Write
│
├── Data Characteristic
│   UUID: FFF2
│   Notify
│
├── Battery Characteristic
│   UUID: FFF3
│   Read
│
└── Device Info Characteristic
    UUID: FFF4
    Read
```

找到对应Characteristic后，进行业务数据的读写和处理。

## 一、Service 和 Characteristic的UUID

UUID 可以理解成：**BLE 世界里的接口 ID。**

1. 16-bit UUID：`Bluetooth SIG` 分配的标准 `UUID`，例如` 180F（Battery Service）`。完整表示可写成：`0000180F-0000-1000-8000-00805F9B34FB`

2. 128-bit UUID。通常用于厂商自定义` Service / Characteristic`，例如：`A4E10000-1234-5678-9ABC-123456789ABC`。

   UUID 的标准文本形式通常为 **32 个十六进制字符 + 4 个连字符，共 36 个字符**。

举例说明：

```text
Service UUID
FFF0
Command Characteristic
FFF1
Data Characteristic
FFF2
```

App 连接BLE成功后：

```text
连接蓝牙设备
 ↓
discoverServices()
 ↓
找到 FFF0
 ↓
discoverCharacteristics()
 ↓
找到 FFF1 / FFF2 / FFF3
```

此时使用到UUID如：

```text
Service UUID: 0000FFF0-0000-1000-8000-00805f9b34fb
Characteristic UUID: 0000FFF1-0000-1000-8000-00805f9b34fb
Property: Write
Characteristic UUID: 0000FFF2-0000-1000-8000-00805f9b34fb
Property: Notify
```

## 二、Characteristic 的 常用Property

### **Read**

```text
Phone → Watch

读取：
电量
版本
设备状态
配置
```

例如：

```
蓝牙设备 （Read Battery）
 ↓
0x55

App解析
 ↓
85%
```

### **Write**

App 主动发送命令：

```text
App
 ↓
01 02 03 04
 ↓
蓝牙设备
```

例如：

```text
设置时间
开始同步
停止同步
修改配置
查询历史数据
```

### **Write With Response**

```text
App
 ↓
Write Request
 ↓
蓝牙设备
 ↓
Write Response
```

可以确认：**ATT/GATT 层的写请求得到了外设响应，但不等于厂商业务命令已经执行成功**。

### Write Without Response

```text
App
 ↓
Write Command
 ↓
蓝牙设备
```

没有 GATT 层确认。

优点：速度快、 吞吐量高。

缺点：不能确认设备是否成功收到。

### Notify

```text
App
 ↓
订阅 Notify
 ↓
蓝牙设备有新数据
 ↓
主动通知手机
 ↓
App收到数据
```

例如：

```text
App： 开启 FFF2 Notify
         ↓
Watch：81 04 48 00 00 00 （产生数据并通知）
         ↓
App： 解析为： 心率 = 72 bpm
```

开启 `Characteristic `的 `Notification/Indication`成功后，当外设发送新值时，系统通过回调把数据交给 App。

## 三、CCCD

CCCD（`Client Characteristic Configuration Descriptor`）客户端特征值配置描述符。用来配置`Characteristic`的`Notification` 或 `Indication`是否启用。

```swift
peripheral.setNotifyValue(true, for: characteristic)
```

`App` 调用 `setNotifyValue(true, for:)` 发起订阅后，`Core Bluetooth` 负责底层 `GATT` 订阅处理；`App` 通常不需要直接操作 `CCCD`。

## 四、Notify 和 Indicate 的区别

### **Notify**

```text
设备
 ↓
数据
 ↓
手机

不需要确认
```

特点：快、吞吐量高、不确认。

### Indicate

```text
设备
 ↓
数据
 ↓
手机
 ↓
确认
```

特点：可靠性更高、速度更慢。

Bluetooth SIG 对两者的核心区别就是：Notification 不要求 ATT 层确认，而 Indication 要求接收端确认。

## 五、“厂商私有协议”传送数据

厂家在传输数据的时候规定：

```text
AA 55 05 01 48 XX

AA55   帧头
05     长度
01     心率
48     72 bpm
XX     checksum
```

也就是说：

```text
Bluetooth GATT
负责：把 Byte[] 送过去

业务协议
负责：这些 Byte 到底是什么意思
```

App研发人员根据“厂商私有协议”进行数据的解析和处理。

## 六、数据拆包和组包

假设设备有一条完整数据：

```text
AA 55 10 01 02 03 04 05 06 07 08 CRC
```

你不能假设 App 一次 Notify 一定收到完整一包。

实际可能：（仅仅是示范，说明厂家协议把一条业务消息拆成多个`value/Notify`发送）

```text
第一次：
AA 55 10 01

第二次：
02 03 04 05 06

第三次：
07 08 CRC
```

所以 App 通常必须有：

### Receive Buffer

```text
Notify
 ↓
Byte[]
 ↓
Buffer
 ↓
找帧头
 ↓
解析 Length
 ↓
判断完整数据
 ↓
完整
 ↓
取出 Frame
 ↓
CRC
 ↓
解析业务数据
```

建议协议本身至少包含：

```text
Header
Length
Command
Sequence
Payload
Checksum / CRC
```

例如：(由厂家协议决定)

```text
┌────────┬──────┬─────┬─────┬─────────┬─────┐
│   Header    │  Len     │    CMD  │   SEQ  │     Payload   │   CRC  │
└────────┴──────┴─────┴─────┴─────────┴─────┘
```

## 七、MTU（Maximum Transmission Unit）

`ATT MTU`(最大传输单元)  =  一次 ATT 通讯允许的数据大小。

BLE 一次能传多少数据并不是无限的。例如设备一次要同步：

```text
1000 Bytes
```

通常不会：

```text
一次 Notify 1000 Bytes
```

而是：

```text
Packet 1
Packet 2
Packet 3
...
```

`ATT MTU` 的实际值取决于双方 BLE 协议栈及连接过程中的协商结果。最终可使用的 `ATT MTU` 不会超过通信双方所支持的接收能力，具体由 ATT MTU Exchange 过程确定。

例如对于常见的` ATT Notification`、`Write Request` 等具有 `3 Byte ATT` 头部的操作，`MTU=185`时 `Characteristic Value` 最多约为 `182 Bytes`；不同 `ATT` 操作的协议开销可能不同。

**提示**： 对于接收 `Notify/Indicate`，`App` 通常不需要处理` ATT/L2CAP` 底层 `MTU` 分段；`didUpdateValueFor` 中重点按照厂商业务协议解析收到的` Characteristic Value`。

## 八、字节序（大小端）

例如蓝牙设备发送：`34 12`

如果协议规定：`Little Endian`

那么：`0x1234`

如果：`Big Endian`

就是：`0x3412`

智能硬件协议中非常常见：

```text
常见基础数据表示：
UInt8
UInt16
UInt32
Int16
Float

常见编码/组织方式：
Timestamp
Bit Field （位字段）一个整数中的不同 bit 分别代表不同含义。
BCD （Binary-Coded Decimal，二进制编码的十进制数） 十进制：25， BCD:0x25
```

必须能独立解析。

## 九、连接状态机

完整状态：

```text
Idle
 ↓
Scanning
 ↓
Connecting
 ↓
Connected
 ↓
Discovering Service
 ↓
Discovering Characteristic
 ↓
Enabling Notify
 ↓
Ready
 ↓
Synchronizing
```

异常：

```text
Disconnected
 ↓
Retry
 ↓
Connecting
```

最终最好设计为：

```text
BluetoothManager
├── BLE基础能力
│    ├── Scan
│    ├── Connect / Disconnect
│    ├── Discover Service
│    ├── Discover Characteristic
│    ├── Read
│    ├── Write
│    └── Notify
└── App自己实现的通信管理
     ├── Command Queue
     ├── Packet Parser
     ├── Packet Assembler
     ├── Timeout
     ├── Retry
     └── State Machine
```

创建一个`BluetoothManager`来统一管理蓝牙扫描、连接、通讯、包解析、重试。

## 十、Command Queue

不要这样连续：

```text
write A
write B
write C
write D
write E
```

尤其是设备性能比较弱的时候，很容易出问题。

更可靠的设计：

```text
Command Queue
A
↓
等待 Response / Notify

B
↓
等待 Response / Notify

C
↓
...
```

每条命令：

```text
Command
├─ Data
├─ Timeout
├─ Retry Count
├─ Sequence ID
└─ Response Matcher
```

创建一个`命令队列`。将每一条命令记录队列中。如果厂商协议要求串行请求，或者设备并发处理能力有限，可以通过 Command Queue 控制命令顺序，上一条命令完成/超时后再执行下一条。

例如：

```text
发送：GetHistoryData #35

等待： Response SEQ = 35
```

这样协议层才稳定。

## 十一、Timeout + Retry

出现异常情况：

```text
Write失败
Notify没回来
设备突然断开
设备超出距离
手机关闭蓝牙
蓝牙设备没电
系统杀后台
GATT异常
数据CRC错误
```

典型：

```text
发送 CMD
 ↓
等待 3 秒
 ↓
没有响应
 ↓
Retry #1
 ↓
失败
 ↓
Retry #2
 ↓
失败
 ↓
Command Failed
```

具体时间与重试次数应该根据设备协议测试确定，而不是死套一个固定值。

注意：Retry 必须结合命令的幂等性、Sequence ID 和厂商协议设计，不能所有失败命令都无条件重发。



## 十二、iOS 蓝牙相关对象

核心关系：

```text
CBCentralManager
     ↓
CBPeripheral
     ↓
CBService
     ↓
CBCharacteristic
     ↓
CBDescriptor
```

Apple 的 Core Bluetooth 就是把 GATT Service、Characteristic、Descriptor 等模型封装成这些对象。

核心流程：

```swift
scanForPeripherals()
        ↓
connect()
        ↓
discoverServices()
        ↓
discoverCharacteristics()
        ↓
setNotifyValue()
        ↓
writeValue()
        ↓
didUpdateValue
```

熟练处理这些 API 的生命周期、回调顺序和异常状态。

## 十三、Android 蓝牙相关对象

核心关系：

```text
BluetoothManager
      ↓
BluetoothAdapter
      ↓
BluetoothLeScanner
      ↓
BluetoothDevice
      ↓
BluetoothGatt
      ↓
BluetoothGattService
      ↓
BluetoothGattCharacteristic
      ↓
BluetoothGattDescriptor
```

基本过程同样是：

```text
Scan
 ↓
Connect
 ↓
Discover Services
 ↓
Find Characteristic
 ↓
Enable Notify
 ↓
Read / Write
 ↓
onCharacteristicChanged()
```

**也就是说：** `iOS` 和 `Android` 的` API` 名字不同，但背后的 `GATT` 模型完全是一套东西。

`Android` 开启 `Notification/Indication` 时，通常还需要处理对应 `CCCD`；这一点与 `iOS Core Bluetooth` 的封装方式有所不同。



### **参考资料**

⁠Bluetooth SIG：Bluetooth LE Primer 中文版
⁠Bluetooth SIG：GATT Specification Supplement
⁠Bluetooth SIG：Assigned Numbers
⁠Apple：Core Bluetooth
⁠Android：Bluetooth Low Energy Overview
---
title: iOS-APP与蓝牙设备长连接的可行性方案
date: 2026-09-11 22:28:21
tags:
---

之前编写[低功耗蓝牙的通信和省电的秘密](https://mp.weixin.qq.com/s/BpAeDfLQ_R3EkAOW-anmPA)的时候，了解到蓝牙低功耗的秘密是99%的时间在休眠。那么APP在后台无感又低功耗从蓝牙设备获取到数据，这又是怎么做到的？即：**用户不主动打开 App，蓝牙设备仍能通过 BLE 把数据传给 iPhone，然后 App 再上传服务器。**
## 前提
APP处于后台有两种情况:
1. 用户主动上划杀掉APP。也就是：**用户主动杀进程(Force Quit)**。
2. 用户切换到其他APP。 也就是：**APP进入后台(Suspend)**.
#### **1. 杀进程**
进程已经终止，与蓝牙设备的连接也断开，蓝牙设备后台能力也停止。那么不管什么方案都不能实现数据获取。
**除非**
（1）用户主动打开APP。
（2）远端服务发送通知，用户点击通知打开APP。
后端服务监控着用户的数据更新情况，如果用户已经几个小时或一天没有上传数据，那么说明蓝牙设备没电、通讯异常或APP被主动终止。此时进行发送一个通知，提醒用户打开APP查看具体原因。
用户主动打开APP，那么一切就开始正常执行。
#### 2. 切后台
当APP不在前台即进入后台，一段时间后APP就会被iOS系统挂起。
此时Apple公司提供了**iOS bluetooth-central 后台模式** ，当蓝牙设备发送数据时，iOS系统会唤醒APP并接受到数据，而且提供短时间的网络通讯能力。
展开讲讲后台通讯的逻辑。
## 一、BLE后台能力
在APP的开发项目中，给APP添加一个*Uses Bluetooth LE accessories*的权限。
![蓝牙LE配件](https://p.ipic.vip/o1dma6.png)
## 二、维持连接 和 Notify，读写数据
#### 工作方式：
打开 App
    ↓
扫描蓝牙设备  （可指定CBUUID，或 kCBAdvDataManufacturerData）
    ↓
连接蓝牙设备  （避免使用蓝牙名称，需要使用SN绑定）
    ↓
Discover Service （可UUID过滤，或发现所有）
    ↓
Discover Characteristic （可UUID过滤，或发现所有）
    ↓
打开 Notify / Indicate  （监听订阅）
    ↓
App 进入后台
    ↓
蓝牙设备产生新数据
    ↓
Characteristic Notify  / Indicate （通过监听传播数据）
    ↓
iOS 唤醒 App  （iOS系统将APP唤醒）
    ↓
func peripheral(_ peripheral: CBPeripheral, didUpdateValueFor characteristic: CBCharacteristic, error: Error?)
    ↓
读取 / 接收数据

此方式可以在APP没有被杀进程的情况下，及时的读取到蓝牙设备的发送的信息。
如果蓝牙设备的数据是不能丢失的，那么外设的特征值需要是**.indicate**。
#### 蓝牙设备数据的记录建议
蓝牙设备持续产生数据，如：心率、血氧、步数、睡眠、体温、HRV等。
不需要实时推送或广播数据，而是先记录在本地的Flash中，持续保存数据。

积累一定数量后
        ↓
发送Notify： 有新数据了
        ↓
iOS系统被BLE事件唤醒
        ↓
App 发命令：从 timestamp / sequenceId 开始同步 （从上一条记录继续）
        ↓
蓝牙设备批量发送历史数据
        ↓
App将数据保存到本地
        ↓
App向服务器同步数据

## 三、**必须加入 Core Bluetooth State Preservation & Restoration**
在初始化中央设备的时候，必须：
```swift
CBCentralManager(
    delegate: self,
    queue: nil,
    options: [
        CBCentralManagerOptionRestoreIdentifierKey: "com.xxx.bluetooth.central"
    ]
)
```
设置`CBCentralManagerOptionRestoreIdentifierKey`后，系统将保存蓝牙上下文信息。
当系统因为 BLE 事件重新恢复 App 后：
```swift
func centralManager(
    _ central: CBCentralManager,
    willRestoreState dict: [String : Any]) {
    // 恢复 peripheral
    // 恢复 connection
    // 恢复 subscription
}
```
在dict中可以拿到：
1. `CBCentralManagerRestoredStatePeripheralsKey`：数组，**CBPeripheral 对象列表**（当时已连接 / 正在 pending 等待连接的外设）
2. `CBCentralManagerRestoredStateScanServicesKey`：当时正在扫描的服务 UUID 数组
3. `CBCentralManagerRestoredStateScanOptionsKey`：扫描配置参数
Apple 专门提供了 **State Preservation and Restoration**，让 Core Bluetooth 保存中央管理器相关状态，并在系统重新启动或恢复 App 时交还这些状态。拿到CBPeripheral后，进行重新 `discoverServices` → `discoverCharacteristics`，然后重新 `setNotifyValue(true)` **订阅 Notify**。
**注意：**后台唤醒App是有时间限制的，只有很短的执行窗口。进行大数量的同步，可能会超时。
## 四、从蓝牙设备获取到数据，先本地缓存
后台唤醒App的时间限制，如果进行数据网络上传，而没有进行本地缓存。那么就会发生（1）执行窗口消失。（2）上传数据失败。此时数据就会丢失，而蓝牙设备因为数据已经传输过，则会将本地数据清除。导致数据永久丢失。
SO，必须缓存在本地

BLE收到数据
    ↓
SQLite / CoreData / 文件
    ↓
标记 pending_upload
    ↓
上传服务器
    ↓
服务器 ACK
    ↓
标记 uploaded

确保数据被丢失。
## 五、上传服务器推荐URLSession
本地缓存后，可以尝试上传少量的数据。Apple 允许拥有相应后台执行模式的 App，在被系统给予后台执行时间时，也可以使用普通 URLSession 网络请求。
如果上传文件较大或者希望上传任务脱离 App 生命周期，则可以使用：
```swift
URLSessionConfiguration.background(
    withIdentifier: "com.xxx.ring.upload"
)
```
Background URLSession 会把网络传输交给系统独立进程，即使 App 被暂停甚至终止，网络任务在满足条件时仍可继续。
这个方法可以确认上传成功，如果需要获取到成功结果，然后修改本地数据的状态。那么还是要避免大量数据的操作。 可以缩短蓝牙设备的Notify的周期，然后每次少量上传数据。
## 六、蓝牙设备的数据协议
App和蓝牙设备在后台能够数据通讯后，要确认数据完整、高效并安全的传输到App，数据协议需要有：

本地数据存储
    +
sequenceId
    +
timestamp
    +
批量读取
    +
断点续传
    +
CRC / checksum
    +
ACK

此方案可以实现：（1）数据不丢失。（2）指定数据读取。（3）断点续传。
## 蓝牙设备基础知识
- **GAP（访问层）：负责建立蓝牙连接** 角色：Central（中心）、Peripheral（外设），解决：扫描、广播、发起连接。
- **GATT（属性层）：连接成功之后的数据交互规则** 角色：**GATT Server（服务端）、GATT Client（客户端）**，解决：读数据、写数据、通知上报。
#### 蓝牙设备的GAP2个角色：
1. **bluetooth-central（中心，主机）**： 手机、电脑、蓝牙网关
    - 主动扫描广播包、发起连接请求
    - 连接建立后，控制通信时序
    - 一个 central 可以同时连多个 peripheral
    - GATT 层面：一般是 **GATT Client（客户端）**，去读写外设的数据
2. **bluetooth-peripheral（外围，从机）**： 手环、温湿度传感器、BLE 按键
    - 向外发广播，被动等待别人来连接
    - 通常同一时间只能被一个 central 连接
    - GATT 层面：一般是 **GATT Server（服务端）**，存放数据和服务
#### GATT（Generic Attribute Profile，通用属性配置文件-）：
*  **GATT Client（客户端)**： 持有所有服务、特征值数据；被动等待对方来读写。
*  **GATT Server（服务端）**： 主动发起操作：读特征、写特征、开启通知。
GATT的UUID（服务、特征）是设备厂商提供的协议文档。即指定GATT服务、特征UUID和特征UUID的可读、可写、支持Notify。
GATT Client指定*characteristic*执行（1）read 读一次数据。（2）write 下发指令。（3）订阅数据。
#### 监听订阅的区分（由外设特征决定）：
- **Notify**：GATT 通知，Server 发数据，**不需要 App 回复 ACK**（绝大多数蓝牙设备用这个）
- **Indicate**：指示，Server 发数据，**App 必须回复确认**，消费设备很少用

注： 此方案还是设想阶段，计划买一个可以二创的蓝牙设备进行实践下。


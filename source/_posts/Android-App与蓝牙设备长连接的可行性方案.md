---
title: Android-App与蓝牙设备长连接的可行性方案
date: 2026-09-13 00:37:19
tags:
---

需求和原理同[iOS-APP与蓝牙设备长连接的可行性方案](https://mp.weixin.qq.com/s/yEZ6Mx79D6QqOskByEgt5A),包括蓝牙的基础名词介绍.

## 前提

当APP操作为：

1. 系统杀死整个进程
2. 设置 →  强制停止APP
   那么将无法在后台执行操作，也无法继续蓝牙设置连接。

## 一、可行性方案的架构

靠谱并且逻辑清晰的方案：

```text
蓝牙设备（具备Flash本地缓存数据）
      ↓
BluetoothGatt（扫描、连接、Notify、ACK，重连）
      ↓
Foreground Service（维持后台，保证BLE通信）
      ↓
Room （本地可靠存储，缓存数据，上传过设置uploaded = true）
      ↓
WorkManager （批量上传 / 失败重试 ）
      ↓
云端服务器 （数据同步到服务器）
```

## 二、核心Foreground Service

Foreground Service主要负责：

```text
启动 BLE
建立 GATT 连接
订阅 Characteristic Notify
持续接收蓝牙设备数据
处理 BLE 指令
检测连接状态
断线自动重连
```

不要把 BLE 连接放在：

```text
Activity
Fragment
ViewModel
```

里面。
正确关系应该是：

```text
Activity
   ↓
启动 / 绑定 BleForegroundService
   ↓
BleConnectionManager
   ↓
BluetoothGatt
   ↓
蓝牙设备
```

这样即使：

```text
用户按 Home
切换微信
锁屏
Activity 被销毁
```

BLE Service 仍然可以继续运行。

## 三、Foreground Service 必须显示通知

如果使用 FGS，就要接受一个基本条件：Foreground Service 必须关联一个 Notification。
例如：

```text
蓝牙设备
已连接，正在同步健康数据
```

调用：

```kotlin
startForeground(
    NOTIFICATION_ID,
    notification
)
```

这个通知的含义不是：

```text
App界面在前台
```

而是： 用户能够知道这个 App 正在持续执行一个重要后台任务。

## 四、Android 7 和 Android 8+ 要分开启动 Service

Android 7：

```kotlin
startService(intent)
```

Android 8+：

```kotlin
startForegroundService(intent)
```

因此代码一般：

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    context.startForegroundService(intent)
} else {
    context.startService(intent)
}
```

然后 Service 里面调用：

```kotlin
startForeground(...)
```

## 五、BLE 连接流程

完整连接流程：

```text
扫描蓝牙设备
   ↓
找到目标设备
   ↓
connectGatt()
   ↓
onConnectionStateChange()
   ↓
CONNECTED
   ↓
discoverServices()
   ↓
找到 Service
   ↓
找到 Characteristic
   ↓
开启 Notify
   ↓
onCharacteristicChanged()
   ↓
持续接收蓝牙设备数据
```

核心仍然使用：

```kotlin
BluetoothGatt
```

Foreground Service 只是提供长期运行环境，并不替代 BLE API。

## 六、收到蓝牙设备数据以后，立即写 Room

推荐：

```text
蓝牙设备
   ↓
BLE Notify
   ↓
Foreground Service
   ↓
数据解析
   ↓
Room
```

例如：

```text
ring_health_data

id
deviceId
sequenceId
timestamp
type
value
uploaded
```

收到数据：

```text
sequenceId = 10025
heartRate = 72
timestamp = ...
uploaded = false
```

先保存：

```text
Room
```

保存成功以后，再给蓝牙设备：

```text
ACK
```

推荐顺序：

```text
BLE收到数据
   ↓
校验数据
   ↓
写Room
   ↓
数据库事务成功
   ↓
返回ACK给蓝牙设备
```

不要：

```text
收到数据
   ↓
立即ACK
   ↓
再写数据库
```

否则如果 ACK 后 App 进程异常死亡，可能造成数据丢失。

## 七、蓝牙设备本身必须具备本地存储

数据缓存的设计：

```text
蓝牙设备 Flash
   ↓
App Room
   ↓
服务器
```

三级存储。
蓝牙设备记录：

```text
#10001
#10002
#10003
...
```

App 告诉蓝牙设备：

```text
ACK #10003
```

表示：\#10003 之前的数据已经安全进入手机。
如果 BLE 断开：

```text
蓝牙设备继续记录
   ↓
数据存在 Flash
```

下一次重新连接：

```text
App：
我最后同步到了 #10003
蓝牙设备：
继续发送 #10004 开始的数据
```

这样才能真正做到：
BLE 可以断，但数据不能丢。

## 八、BLE 断线必须自动重连

长连接不代表永远不断。
可能因为：

```text
蓝牙设备离开距离
手机蓝牙关闭
蓝牙设备没电
系统蓝牙栈异常
射频干扰
进程被系统回收
```

因此建议维护状态机：

```text
DISCONNECTED
     ↓
CONNECTING
     ↓
CONNECTED
     ↓
DISCOVERING
     ↓
READY
```

断线：

```text
READY
  ↓
DISCONNECTED
  ↓
等待
  ↓
重新扫描 / 重连
```

推荐退避：

```text
1 秒
2 秒
5 秒
10 秒
30 秒
60 秒
```

避免持续高频扫描和连接。

## 九、后台上传使用WorkManager上传

```text
BLE
 ↓
Room
 ↓
WorkManager
 ↓
服务器
```

这样 BLE 和网络完全解耦。
例如：

```text
BLE收到50条数据
      ↓
全部写Room
      ↓
WorkManager
      ↓
批量上传50条
      ↓
服务器成功
      ↓
uploaded = true
```

网络失败：

```text
Result.retry()
```

WorkManager 之后继续调度。

## 十、WorkManager 可以直接访问 Room

不需要：

```text
Worker
 ↓
把数据返回Activity
 ↓
Activity保存Room
```

正确方式：

```text
Worker
 ↓
直接读取Room
 ↓
上传服务器
 ↓
直接更新Room
```

例如：

```kotlin
override suspend fun doWork(): Result {

    val dao = AppDatabase
        .getInstance(applicationContext)
        .ringDataDao()

    val data = dao.getNotUploaded()

    return try {
        api.upload(data)

        dao.markUploaded(
            data.map { it.id }
        )

        Result.success()

    } catch (e: Exception) {
        Result.retry()
    }
}
```

上传成功后，直接将Room中的数据设置 `uploaded = true`.

## 十一、权限适配

最低 Android 7，所以需要分版本处理。

### **Android 7 ～ Android 11**

BLE 基础权限：

```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
```

BLE 扫描通常还涉及定位权限：

```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

### **Android 12+**

新增：

```xml
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
```

这两个需要按实际场景申请运行时权限。

## 十二、Foreground Service 权限

基础：

```xml
<uses-permission
    android:name="android.permission.FOREGROUND_SERVICE" />
```

Android 14+ 如果使用：

```text
foregroundServiceType="connectedDevice"
```

还需要：

```xml
<uses-permission
    android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />
```

Service：

```xml
<service
    android:name=".ble.BleForegroundService"
    android:exported="false"
    android:foregroundServiceType="connectedDevice" />
```

对于 Android 新版本，蓝牙设备持续 BLE 通信正属于 `connectedDevice` 这类 FGS。


## 补充

Android 8 推出了`CompanionDeviceManager`（配套设备管理器）用来将`APP`和`外部设备`设置关联，并提供相应的后台工作能力。此方案比`Foreground Service`采用通知方式来告诉用户一直在长期执行重要人物，更加合适长期绑定蓝牙设备。但是面临着不同Android版本接口不一致和需要FGS辅助，导致逻辑会复杂很多。





让App保持与蓝牙设备的通讯，并将数据上传到云服务。同时蓝牙设备和App都需要支持本地缓存，增量更新和断点续传的功能。













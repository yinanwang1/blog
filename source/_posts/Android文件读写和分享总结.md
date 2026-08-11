---
title: Android文件读写和分享总结
date: 2026-08-11 22:44:55
tags:
---

# **一、目录怎么选**

| **目录**           | **示例路径**                                   | **权限**                         | **谁用**                               | **是否推荐**                       |
| ------------------ | ---------------------------------------------- | -------------------------------- | -------------------------------------- | ---------------------------------- |
| 内部私有目录       | `/data/data/包名/files/`                       | 不需要                           | App 自己保存配置、缓存、数据库、token  | 强烈推荐                           |
| 内部缓存目录       | `/data/data/包名/cache/`                       | 不需要                           | 临时文件、网络缓存                     | 推荐                               |
| 外部专属目录       | `/storage/emulated/0/Android/data/包名/files/` | 不需要                           | App 自己的大文件、日志、导出前临时文件 | 推荐                               |
| 公共图片/视频/音频 | `Pictures/`、`Movies/`、`Music/`               | Android 10+ 写自己文件不需要权限 | 用户可见媒体                           | 推荐用 MediaStore                  |
| 公共下载目录       | `/storage/emulated/0/Download/`                | Android 10+ 用 MediaStore/SAF    | PDF、Excel、ZIP、导出文件              | 推荐用 SAF 或 MediaStore Downloads |
| 任意用户选择文件   | 用户手动选择                                   | 不需要传统存储权限               | 打开 PDF、Excel、TXT、ZIP              | 推荐用 SAF                         |
| 根目录/系统目录    | `/system/`、`/data/`、其他 App 目录            | 普通 App 不能访问                | 系统/Root                              | 不要碰                             |

Android 11 起，分区存储对外部存储访问限制更严格，target API 30+ 必须使用 scoped storage；Android 10 之前还能靠路径和 `READ/WRITE_EXTERNAL_STORAGE`，但现在不建议这样做。

`READ/WRITE_EXTERNAL_STORAGE` 用来访问`/storage/emulated/0/*/`中公共的内容。

------

# **二、权限规则**

## **1. App 自己目录：不需要权限**

```kotlin
context.filesDir
context.cacheDir
context.getExternalFilesDir(null)
context.externalCacheDir
```

这些 Android 7–16 都稳定可用。

------

## **2. 访问相册图片/视频/音频**

Android 13+：

```xml
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />
```

Android 12 及以下：

```xml
<uses-permission
    android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
```

Android 官方说明：访问其他 App 创建的媒体文件，需要声明对应的媒体读取权限；但 Android 10+ 访问自己写入 MediaStore 的媒体文件，不需要存储权限。

------

## **3. 写入图片、视频、音频、下载文件**

推荐：

```text
Android 10+：MediaStore
Android 7-9：WRITE_EXTERNAL_STORAGE 或退回 App 专属目录
```

------

### 内部目录

/data/data/包名/files/
Android/data/包名/files/

永远不需要存储权限

### 公共目录

Download/DCIM/Pictures

Android 9及以下：
READ/WRITE_EXTERNAL_STORAGE

Android 10~12：
MediaStore / SAF

Android 13+：
READ_MEDIA_* / Photo Picker

## **4. PDF、Excel、ZIP、TXT 等普通文件**

推荐：

```text
让用户选择保存位置：ACTION_CREATE_DOCUMENT
让用户选择打开文件：ACTION_OPEN_DOCUMENT
```

也就是 SAF，优点是 Android 7–16 都能用，而且不需要传统存储权限。

------

# **三、Manifest 权限**

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Android 12 及以下读取公共文件 -->
    <uses-permission
        android:name="android.permission.READ_EXTERNAL_STORAGE"
        android:maxSdkVersion="32" />

    <!-- Android 9 及以下写公共目录 -->
    <uses-permission
        android:name="android.permission.WRITE_EXTERNAL_STORAGE"
        android:maxSdkVersion="28" />

    <!-- Android 13+ 读取媒体 -->
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
    <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
    <uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

</manifest>
```

------

# **四、Android 7–16 通用文件存储**

## **1. 保存 App 私有文件**

```kotlin
fun savePrivateFile(context: Context, fileName: String, content: String) {
    val file = File(context.filesDir, fileName)
    file.writeText(content)
}

fun readPrivateFile(context: Context, fileName: String): String {
    val file = File(context.filesDir, fileName)
    return if (file.exists()) file.readText() else ""
}
```

适合保存：

```text
配置
草稿
token
用户偏好
App 内部数据
```

------

## **2. 保存外部 App 专属文件**

```kotlin
fun saveExternalAppFile(context: Context, fileName: String, content: String): File {
    val dir = context.getExternalFilesDir("logs")
    if (dir != null && !dir.exists()) {
        dir.mkdirs()
    }

    val file = File(dir, fileName)
    file.writeText(content)
    return file
}
```

路径类似：

```text
/storage/emulated/0/Android/data/你的包名/files/logs/app.log
```

特点：

```text
不需要权限
用户一般看不到
App 卸载后会被删除
适合日志、临时导出文件、大缓存
```

------

## **3. 保存图片到相册 MediaStore**

```kotlin
fun saveImageToPictures(
    context: Context,
    fileName: String,
    bytes: ByteArray
): Uri? {
    val resolver = context.contentResolver

    val values = ContentValues().apply {
        put(MediaStore.Images.Media.DISPLAY_NAME, fileName)
        put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/MyApp")
            put(MediaStore.Images.Media.IS_PENDING, 1)
        }
    }

    val uri = resolver.insert(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        values
    ) ?: return null

    resolver.openOutputStream(uri)?.use { output ->
        output.write(bytes)
    }

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        values.clear()
        values.put(MediaStore.Images.Media.IS_PENDING, 0)
        resolver.update(uri, values, null, null)
    }

    return uri
}
```

适合：

```text
保存图片
保存截图
保存二维码
保存用户能在相册看到的图片
```

------

## **4. 保存文件到 Download 目录**

```kotlin
fun saveFileToDownload(
    context: Context,
    fileName: String,
    mimeType: String,
    bytes: ByteArray
): Uri? {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        val resolver = context.contentResolver

        val values = ContentValues().apply {
            put(MediaStore.Downloads.DISPLAY_NAME, fileName)
            put(MediaStore.Downloads.MIME_TYPE, mimeType)
            put(MediaStore.Downloads.RELATIVE_PATH, Environment.DIRECTORY_DOWNLOADS + "/MyApp")
            put(MediaStore.Downloads.IS_PENDING, 1)
        }

        val uri = resolver.insert(
            MediaStore.Downloads.EXTERNAL_CONTENT_URI,
            values
        ) ?: return null

        resolver.openOutputStream(uri)?.use {
            it.write(bytes)
        }

        values.clear()
        values.put(MediaStore.Downloads.IS_PENDING, 0)
        resolver.update(uri, values, null, null)

        uri
    } else {
        null
    }
}
```

Android 10+ 推荐这样写。

------

## **5. Android 7–9 保存到 Download**

```kotlin
@Suppress("DEPRECATION")
fun saveFileToDownloadLegacy(
    fileName: String,
    bytes: ByteArray
): File {
    val dir = Environment.getExternalStoragePublicDirectory(
        Environment.DIRECTORY_DOWNLOADS
    )

    if (!dir.exists()) {
        dir.mkdirs()
    }

    val file = File(dir, fileName)
    file.writeBytes(bytes)
    return file
}
```

注意：Android 7–9 需要申请：

```xml
<uses-permission
    android:name="android.permission.WRITE_EXTERNAL_STORAGE"
    android:maxSdkVersion="28" />
```

------

## **6. SAF：让用户选择保存位置**

```kotlin
fun createDocumentLauncher(activity: Activity) {
    val intent = Intent(Intent.ACTION_CREATE_DOCUMENT).apply {
        addCategory(Intent.CATEGORY_OPENABLE)
        type = "text/plain"
        putExtra(Intent.EXTRA_TITLE, "demo.txt")
    }

    activity.startActivityForResult(intent, 1001)
}
```

写入用户选择的文件：

```kotlin
fun writeToUri(context: Context, uri: Uri, content: String) {
    context.contentResolver.openOutputStream(uri)?.use { output ->
        output.write(content.toByteArray())
    }
}
```

适合：

```text
PDF
Excel
Word
ZIP
TXT
用户自己决定保存到哪里
```

------

## **7. SAF：让用户选择打开文件**

```kotlin
fun openDocument(activity: Activity) {
    val intent = Intent(Intent.ACTION_OPEN_DOCUMENT).apply {
        addCategory(Intent.CATEGORY_OPENABLE)
        type = "*/*"
    }

    activity.startActivityForResult(intent, 1002)
}
```

读取：

```kotlin
fun readFromUri(context: Context, uri: Uri): ByteArray {
    return context.contentResolver.openInputStream(uri)?.use {
        it.readBytes()
    } ?: ByteArray(0)
}
```

------

# **五、权限申请代码**

```kotlin
fun requestStoragePermission(activity: Activity) {
    if (Build.VERSION.SDK_INT >= 33) {
        activity.requestPermissions(
            arrayOf(
                android.Manifest.permission.READ_MEDIA_IMAGES,
                android.Manifest.permission.READ_MEDIA_VIDEO
            ),
            2001
        )
    } else if (Build.VERSION.SDK_INT >= 23) {
        activity.requestPermissions(
            arrayOf(android.Manifest.permission.READ_EXTERNAL_STORAGE),
            2001
        )
    }
}
```

------

# **六、实际开发推荐方案**

| **场景**           | **推荐方案**                                    |
| ------------------ | ----------------------------------------------- |
| 保存登录信息、配置 | `filesDir`                                      |
| 保存缓存           | `cacheDir`                                      |
| 保存日志           | `getExternalFilesDir("logs")`                   |
| 保存图片到相册     | `MediaStore.Images`                             |
| 保存视频到相册     | `MediaStore.Video`                              |
| 保存音频           | `MediaStore.Audio`                              |
| 导出 PDF/Excel     | `ACTION_CREATE_DOCUMENT`                        |
| 用户选择文件上传   | `ACTION_OPEN_DOCUMENT`                          |
| 下载文件给用户看   | Android 10+ 用 `MediaStore.Downloads`，或者 SAF |
| 扫描整个手机文件   | 不建议，除非文件管理器类 App                    |

------

# 



# FileProvider是 Android 文件共享机制。

`FileProvider `不负责存文件，它负责把文件安全地分享给其他 App。**

------

# **一、为什么需要 FileProvider**

Android 7.0（API 24）开始禁止暴露 `file://`

以前：

```java
File file = new File("/sdcard/test.jpg");

Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setDataAndType(
        Uri.fromFile(file),
        "image/*"
);
startActivity(intent);
```

得到的 URI：

```text
file:///sdcard/test.jpg
```

Android 7 开始直接崩溃：

```text
FileUriExposedException
```

因为：

```text
你的 App
↓
直接暴露真实路径
↓
其它 App
```

存在安全问题。



于是 Google 推出了：

```text
content://
```

例如：

```text
content://com.demo.fileprovider/images/test.jpg
```

这时候真实路径被隐藏。

------

# **二、什么时候必须用**

## **场景1：安装 APK**

最常见。

错误写法：

```java
Uri uri = Uri.fromFile(apkFile);
```

正确写法：

```java
Uri uri = FileProvider.getUriForFile(
        context,
        "com.demo.fileprovider",
        apkFile
);
```

然后：

```java
intent.addFlags(
        Intent.FLAG_GRANT_READ_URI_PERMISSION
);
```

------

## **场景2：调用相机拍照**

相机需要写入照片。

错误：

```java
Uri.fromFile(photoFile)
```

正确：

```java
FileProvider.getUriForFile(...)
intent.putExtra(
        MediaStore.EXTRA_OUTPUT,
        photoUri
);
```

------

## **场景3：分享图片**

微信

QQ

钉钉

邮件

浏览器

等。

例如：

```java
Intent.ACTION_SEND
content://
```

------

## **场景4：打开 PDF**

```java
ACTION_VIEW
application/pdf
```

------

## **场景5：打开 Excel**

```java
application/vnd.ms-excel
```

------

## **场景6：打开 Word**

```java
application/msword
```

------

## **场景7：查看日志**

例如：

```text
app.log
```

分享给技术支持。

------

# **三、什么时候不需要**

## **读取自己的文件**

```java
File file = new File(getFilesDir(),"a.txt");
```

直接读：

```java
file.readText()
```

即可。

------

## **写自己的文件**

```java
filesDir
cacheDir
```

不需要。

------

## **MediaStore**

例如：

```java
MediaStore.Images
```

已经返回：

```text
content://
```

不需要 FileProvider。

------

## **SAF**

```java
ACTION_OPEN_DOCUMENT
```

返回：

```text
content://
```

不需要 FileProvider。

------

# **四、配置方法**

## **AndroidManifest**

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">

    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths"/>

</provider>
```

------

## **res/xml/file_paths.xml**

允许共享哪些目录

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>

    <files-path
        name="files"
        path="."/>

    <cache-path
        name="cache"
        path="."/>

    <external-files-path
        name="external"
        path="."/>

</paths>
```

------

# **五、完整示例**

假设：

```text
/storage/emulated/0/Android/data/com.demo/files/test.pdf
```

分享 PDF：

```java
File file = new File(
        getExternalFilesDir(null),
        "test.pdf"
);
Uri uri = FileProvider.getUriForFile(
        this,
        getPackageName()+".fileprovider",
        file
);
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setDataAndType(
        uri,
        "application/pdf"
);
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
startActivity(intent);
```

------

# **六、和 MediaStore、SAF 的关系**

| **方案**            | **用途**     | **是否需要 FileProvider** |
| ------------------- | ------------ | ------------------------- |
| filesDir            | App私有文件  | ❌                         |
| cacheDir            | 缓存         | ❌                         |
| getExternalFilesDir | App专属目录  | ❌                         |
| MediaStore          | 图片视频音频 | ❌                         |
| SAF                 | 用户选文件   | ❌                         |
| 分享文件给别的App   | 文件共享     | ✅                         |
| 安装APK             | 文件共享     | ✅                         |
| 调用相机拍照        | 文件共享     | ✅                         |
| 打开PDF             | 文件共享     | ✅                         |
| 分享日志            | 文件共享     | ✅                         |

------





读写包名下面的文件，都不需要权限了。 


---
title: 了解Hilt基础知识
date: 2026-09-22 16:16:05
tags:
---

Hilt 是一个依赖注入（Dependency Injection，DI）框架。

**以前：** 对象由你自己`new`创建。

**Hilt：** 你告诉 Hilt“这个对象怎么创建”，以后需要它的地方，Hilt 自动帮你创建并传进来。

### 1. 常见的对象创建和依赖
如 ViewModel 需要 Repository：
```kotlin
class LoginViewModel : ViewModel() {
    private val repository = UserRepository()
}
```
依赖关系：
```text
LoginViewModel
    ↓
UserRepository
    ↓
UserApi
    ↓
OkHttp
```

实例化对象，作为传入参数：

```kotlin
UserApi(...)
UserRepository(userApi)
LoginViewModel(repository)
```

对象之间一层套一层，创建和管理越来越麻烦。

------

### 2. 使用 Hilt 后的对象创建

例如：

```kotlin
class UserRepository @Inject constructor(
    private val api: UserApi
)
```

ViewModel：

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {}
```

Compose 页面：

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = hiltViewModel<LoginViewModel>()
) {}
```

此时就不需要手动初始化对象：

```kotlin
UserRepository(...)
LoginViewModel(...)
```

Hilt 会分析：

```text
LoginViewModel
      ↓ 需要
UserRepository
      ↓ 需要
UserApi
```

Hilt 完成了**“对象创建 + 对象组装 + 生命周期管理器”。**

### **3. Hilt 最常见的注解**

| **注解**             | **大概作用**                    |
| -------------------- | ------------------------------- |
| `@HiltAndroidApp`    | 告诉 Hilt：这是整个 App 的入口  |
| `@AndroidEntryPoint` | Activity/Fragment 需要使用 Hilt |
| `@HiltViewModel`     | 这个 ViewModel 交给 Hilt 管理   |
| `@Inject`            | 告诉 Hilt 如何注入/创建对象     |
| `@Module`            | 定义依赖提供规则                |
| `@Provides`          | 告诉 Hilt 某个对象怎么创建      |
| `@Binds`             | 告诉 Hilt 接口对应哪个实现      |
| `@Singleton`         | 整个 App 范围通常只创建一个实例 |
| `@InstallIn`         | 将当前的类安装到指定组件中。    |

### 4. `@Inject`是给`Hilt`声明类

是`Hilt`最核心的注解之一：

```kotlin
class UserRepository @Inject constructor(
    private val api: UserApi
)
```

告诉 Hilt：创建`UserRepository` 的构造函数，如果需要 `UserRepository`，就按照这个构造函数创建。

当`Hilt`发现构造函数又需要`UserApi`, 则寻找`UserApi`并创建。

多级的寻找和创建，形成了一张**依赖关系图（Dependency Graph）**。

------

### 5. `@Inject`实例化一个接口

接口：

```kotlin
interface DeviceRepository {
    fun connect()
}
```

实现：

```kotlin
class BleDeviceRepository @Inject constructor(
    private val bluetoothManager: BluetoothManager
) : DeviceRepository
```

如果：

```kotlin
class DeviceViewModel @Inject constructor(
    private val repository: DeviceRepository
)
```

Hilt 查找 `DeviceRepository`时，发现它是接口，不能直接创建。

解决办法是：需要告诉 Hilt：

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindDeviceRepository(
      impl: BleDeviceRepository
    ): DeviceRepository
}
```

这个`RepositoryModule`不是给其他代码调用，而是给`Hilt`使用的。

代码实现的作用是：如果`Hilt`查找`DeviceRepository`就实例化`BleDeviceRepository`返回。

### **6.** **`@Provides`** 创建第三方类

当类无法在构造函数加`@Inject constructor`时，使用`@Provides`指定。

举例`Retrofit`在项目中用到时，怎么让`Hilt`查找到。

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .build()
    }
}
```

告诉` Hilt`：需要 `Retrofit `的时候，就调用这个方法创建。 而`provideRetrofit`这个方法名不重要，重要的是`Retrofit`类型。使用：

```kotlin
class UserRepository @Inject constructor(
    private val retrofit: Retrofit
)
```

此时就可以找到`Retrofit`然后实例化返回。并且`@Singleton`申明了`Retrofit`在App整个生命周期中，只会存在一个实例。

------

### 7. Hilt就是MVVM 吗？ 不是的

**MVVM 解决**： UI、状态、业务逻辑怎么组织

**Hilt 解决**： 这些对象由谁创建、怎么组装、活多久

例如MVVM的架构：

```text
Compose / Activity
        ↓
     ViewModel
        ↓
    Repository
        ↓
    DataSource
     ↙     ↘
 Retrofit   Room
```

而对象的创建：

```text
Hilt
 ↓
负责创建和组装
 ↓
ViewModel
Repository
Retrofit
Room
...
```

两者组合使用：

```text
Compose + ViewModel + Repository + Hilt + Room + Retrofit
```



了解后，觉得与`Spring Boot`的`@Autowired`一模一样的效果。

个人的见解，详细请看[使用 Hilt 实现依赖项注入](https://developer.android.google.cn/training/dependency-injection/hilt-android?hl=zh-cn)。



## DONE

恍如隔世，旧项目一直在开发，一直在迭代。而新技术也是日新月异啊。

说是被时代的浪潮推倒岸边，或者说是守着一口不深的井。

希望蓦然回首，每一步都没有辜负自己的努力。









 
---
title: 最小框架architecture-templates学习记录
date: 2026-09-28 00:35:09
tags:
---
Android提供的一个框架模版[architecture-templates](https://github.com/android/architecture-templates/tree/base)，很小但很完整。可运行的项目到[architecture-templates](https://github.com/android/architecture-templates/tree/base)下载。我添加到代码学习备注，可到[最小框架学习记录](https://github.com/yinanwang1/knowledgeCode/tree/main/%E6%9C%80%E5%B0%8F%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/main)下载查看，直接覆盖`main`文件夹就可以。

[architecture-templates](https://github.com/android/architecture-templates/tree/base)是很精简的“单模块 + UI/数据两层 + MVVM + 单向数据流”架构。功能是**输入一个名字，保存到数据库，界面显示最近保存的 10 条记录。**使用到了`MVVM`,`Hilt`和`Compose`, 如果不熟悉，跳到文章最后点击链接，先温习下。

## 1. 整体结构

```
app
└── android.template
    ├── MyApplication              初始化 Hilt（依赖注入框架） 
    ├── ui
    │   ├── MainActivity           页面宿主
    │   ├── Navigation             导航与页面组装
    │   ├── mymodel
    │   │   ├── MyModelScreen      展示界面、接收操作
    │   │   └── MyModelViewModel   管理页面状态、处理操作
    │   └── theme                 主题
    └── data
        ├── MyModelRepository     数据访问接口和实现
        ├── di/DataModule         Repository 注入配置
        └── local
            ├── database
            │   ├── MyModel       数据库实体和 DAO
            │   └── AppDatabase   Room 数据库
            └── di/DatabaseModule 数据库注入配置
```

代码的思路是：

1. `MyApplication` 是自定义的 `Application` 类，在 Manifest 的 `application android:name` 中注册；`@HiltAndroidApp` 用于将应用接入 Hilt。
2. 在`AndroidManifest.xml`指定`MainActivity`为启动的`Activity`. 也就是App启动后的主页面。
3. 在`MainActivity`中指定了App的主题`MyApplicationTheme`。 如果需要做切换主题，可以在这做状态监听。
4. 再指定`MainNavigation`导航，可以放置很多个页面。通过 `backStack.add(Detail)` 进入页面，通过移除栈顶返回。`Detail`需要实现 `NavKey`、标注 `@Serializable`，并在 `entryProvider` 中注册对应页面。
5. 在`MainNavigation`创建了一个页面`MyModelScreen`, 页面中使用了`MyModelViewModel`这个ViewModel进行`uiState`的监听。页面中使用了`remember`来保存`TextField`的状态。
6. 在`MyModelScreen`通过`hiltViewModel()`获取`MyModelViewModel`。`Hilt`解析依赖并通过构造函数注入`MyModelRepository`。
7. `MyModelViewModel`使用`MyModelRepository`, `Hilt`负责创建。此处`MyModelRepository`是一个接口，`Hilt`不知道怎么实例化，所以编写了`DataModule`指定`MyModelRepository`的实现类为`DefaultMyModelRepository`。
8. `DefaultMyModelRepository`通过构造函数接受`Hilt`注入的`MyModelDao`, 再进行数据的获取和插入。其中`MyModelDao`的实例化是由`Room`框架来完成。在`DatabaseModule`中的代码是编写后给`Hilt`进行调用，让`Hilt`创建`MyModelDao`。
9. 在`DefaultMyModelRepository`中，将`MyModelDao`修改为`MyModelApi`,就可以实现网络请求数据和插入新数据。实现逻辑往下看。
10. `MyModelDao`是DAO层，对数据进行查询和插入。 `MyModel`为数据的保存对象。
11. Done

使用文件夹组织代码形式分层，各司其职，之后编写相应的代码在对应的文件夹中进行。如果等项目复杂了，可以创建多个`gradle`的`module`来进行管理不同功能，做到功能之间的隔离。



## 2. UI 层：“连接数据”和“绘制界面”

`MyModelScreen.kt`中有两个同名函数`MyModelScreen`，作用不一样：

外层负责连接 ViewModel：

```kotlin
val items by viewModel.uiState.collectAsStateWithLifecycle()

if (items is MyModelUiState.Success) {
    MyModelScreen(
        items = (items as MyModelUiState.Success).data,
        onSave = viewModel::addMyModel,
        modifier = modifier,
    )
}
```

内层只接收数据和回调：

```kotlin
internal fun MyModelScreen(
    items: List<String>,
    onSave: (name: String) -> Unit,
    modifier: Modifier = Modifier,
)
```

两者的职责很清楚：

| 外层页面                 | 内层 UI                   |
| ------------------------ | ------------------------- |
| 获取 ViewModel           | 绘制输入框、按钮、列表    |
| 订阅页面状态             | 接收 `items`              |
| 将保存操作交给 ViewModel | 点击时调用 `onSave(name)` |

**内层不需要知道 Room、Repository、Hilt 甚至 ViewModel 的存在。** 所以预览时传一个列表和空回调就能显示，测试时也能独立创建它。

内层的输入框文本保存在自己的 `remember` 中。这体现了一个合理的划分：**输入中的临时文本归 UI，已保存的数据归数据层。**

## 3. ViewModel：把数据变成页面状态

`MyModelViewModel.kt` 的核心只有两部分:

第一部分，提供页面状态：

```kotlin
val uiState: StateFlow<MyModelUiState> = myModelRepository
    .myModels.map<List<String>, MyModelUiState>(::Success)
    .catch { emit(Error(it)) }
    .stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5000),
        Loading,
    )
```

可以这样理解：

```
Repository 提供字符串列表
        ↓
包装成 Success(list)
        ↓
读取异常时变成 Error
        ↓
转换成可供 UI 订阅的 StateFlow
```

`Loading` 是初始状态。`WhileSubscribed(5000)` 表示最后一个订阅者离开后，等待 5 秒再停止收集上游；**不是每 5 秒查询数据库一次**。

第二部分，采用协程处理保存操作：

```kotlin
fun addMyModel(name: String) {
    viewModelScope.launch {
        myModelRepository.add(name)
    }
}
```

ViewModel 不写 SQL，也不自己创建数据库。它只表达：“用户要求保存这个名字。”

## 4. Repository：向上提供稳定的数据访问接口

`MyModelRepository.kt`定义：

```kotlin
interface MyModelRepository {
    val myModels: Flow<List<String>>
    suspend fun add(name: String)
}
```

实现类负责两种转换：

```kotlin
// 读取：数据库实体 → 上层需要的字符串
myModelDao.getMyModels().map { items -> items.map { it.name } }

// 写入：输入字符串 → 数据库实体
myModelDao.insertMyModel(MyModel(name = name))
```

**这个接口的价值是让 ViewModel 不依赖具体存储实现。** ViewModel 看不到 Room 实体的 `uid`，也不知道表名和查询方式。

在这个模板里 Repository 很薄，因为数据来源只有本地数据库。以后增加网络或缓存，可以在这一边界内扩展实现；(试着添加网络的`MyModelApi`, 接着往下看)

## **5. Room：数据保存和界面刷新的源头**

`MyModel.kt`同时定义实体`MyModel`和 DAO`MyModelDao`：

```kotlin
@Query("SELECT * FROM mymodel ORDER BY uid DESC LIMIT 10")
fun getMyModels(): Flow<List<MyModel>>

@Insert
suspend fun insertMyModel(item: MyModel)
```

查询按自增 `uid` 倒序返回最近 10 条。注意：**这里只限制查询结果，没有删除更早的数据。**

最关键的是查询返回 `Flow`。数据库表变化后，Room 可以重新查询并发出结果，因此保存操作不需要手动拼接页面列表。

实际流程是：

````
flowchart TD
    A[用户点击 Save] --> B[UI 调用 onSave]
    B --> C[ViewModel.addMyModel]
    C --> D[Repository.add]
    D --> E[DAO 插入 Room 数据库]
    E --> F[Room 查询 Flow 发出新列表]
    F --> G[Repository 转换为字符串列表]
    G --> H[ViewModel 生成 Success 状态]
    H --> I[Compose 收集状态并更新界面]
````

**数据库是已保存列表的单一事实来源。** 保存成功之后的界面内容由数据库查询结果决定，从而避免“内存列表已经加上了，但数据库实际保存失败”的不一致。

## **6. Hilt：负责把各层连接起来**

对象依赖关系是：

```kotlin
MyModelViewModel
    → MyModelRepository 接口
        → DefaultMyModelRepository
            → MyModelDao
                → AppDatabase
```

两个配置文件分别负责：

- `DataModule.kt`：用 `@Binds` 指定 Repository 接口对应哪个实现。
- `DatabaseModule.kt`：用 `@Provides` 创建数据库并提供 DAO。

数据库和 `Repository` 被配置为应用进程内的单例；`ViewModel` 则由对应的 `ViewModelStoreOwner` 管理。

构造器明确写出依赖，也让测试可以直接传入替代实现。比如在`MyModelViewModelTest`中的`FakeMyModelRepository`， 直接`MyModelViewModel`的构造函数中传入。

## 7. 添加网络获取和插入数据

如果此项目想要支持网络模块，那么竟然需要修改`MyModelRepository`的实现`NetworkMyModelRepository`，在实现中构造函数，传入的是`MyModelApi`就可以，不影响其他的逻辑。

```kotlin
MyModelViewModel
       ↓
MyModelRepository
       ↓ Hilt 选择一个实现
       ├── DefaultMyModelRepository → MyModelDao
       └── NetworkMyModelRepository → MyModelApi
```

实现步骤如下，可以在[最小框架学习记录](https://github.com/yinanwang1/knowledgeCode/tree/main/%E6%9C%80%E5%B0%8F%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/main)中查看代码：

**1. 统一 Repository 接口**

加入 `refresh()` 的接口用于网络刷新数据：

```
interface MyModelRepository {
    val myModels: Flow<List<String>>

    suspend fun refresh()
    suspend fun add(name: String)
}
```

两个实现都遵守同一份约定，`ViewModel` 始终调用这个接口。

**2. 本地实现：使用 Room DAO**

`DefaultMyModelRepository` 添加一个`refresh()`实现：

```kotlin
class DefaultMyModelRepository @Inject constructor(
    private val dao: MyModelDao,
) : MyModelRepository {

    override val myModels: Flow<List<String>> =
        dao.getMyModels().map { items ->
            items.map { it.name }
        }

    override suspend fun refresh() {
        // Room 的 Flow 会查询数据库，并在数据变化后通知。
        // 纯本地方案无需额外发起刷新。
    }

    override suspend fun add(name: String) {
        dao.insertMyModel(MyModel(name = name))
    }
}
```

**3. 网络实现：使用 API**

创建网络的Model：

```kotlin
interface MyModelRepository {
    val myModels: Flow<List<String>>


    suspend fun refresh()
    suspend fun add(name: String)
}

class DefaultMyModelRepository @Inject constructor(
    private val myModelDao: MyModelDao
) : MyModelRepository {

    override val myModels: Flow<List<String>> =
        myModelDao.getMyModels().map { items -> items.map { it.name } }

    override suspend fun refresh() {
        // Room 的 Flow 会查询数据库，并在数据变化后通知。
        // 纯本地方案无需额外发起刷新。
    }

    override suspend fun add(name: String) {
        myModelDao.insertMyModel(MyModel(name = name))
    }
}

// 网络请求的MyModelRepository实现
class NetworkMyModelRepository @Inject constructor(
    private val api: MyModelApi,
) : MyModelRepository {

    private val models = MutableStateFlow<List<String>?>(null)
    private val mutex = Mutex()

    override val myModels: Flow<List<String>> =
        models.filterNotNull()

    override suspend fun refresh() {
        mutex.withLock {
            models.value = api.getMyModels().map { it.name }
        }
    }

    override suspend fun add(name: String) {
        mutex.withLock {
            api.addMyModel(CreateMyModelRequest(name))

            // 网络保存后，需要主动更新可观察的数据
            models.value = api.getMyModels().map { it.name }
        }
    }
}
```

创建网络的Repository：

```kotlin
class NetworkMyModelRepository @Inject constructor(
    private val api: MyModelApi,
) : MyModelRepository {

    private val models = MutableStateFlow<List<String>?>(null)
    private val mutex = Mutex()

    override val myModels: Flow<List<String>> =
        models.filterNotNull()

    override suspend fun refresh() {
        mutex.withLock {
            models.value = api.getMyModels().map { it.name }
        }
    }

    override suspend fun add(name: String) {
        mutex.withLock {
            api.addMyModel(CreateMyModelRequest(name))

            // 网络保存后，需要主动更新可观察的数据
            models.value = api.getMyModels().map { it.name }
        }
    }
}
```

此时网络请求数据的`Reposity`就创建好了。

**4. 在 DataModule 中指定使用哪个实现**

在`DataModule`中指定使用本地数据库：

```kotlin
@Module
@InstallIn(SingletonComponent::class)
interface DataModule {

    @Binds
    @Singleton
    fun bindMyModelRepository(
        implementation: DefaultMyModelRepository,
    ): MyModelRepository
}
```

想改成网络，只需要把参数类型换成：

```kotlin
@Binds
@Singleton
fun bindMyModelRepository(
    implementation: NetworkMyModelRepository,
): MyModelRepository
```

告诉`Hilt`使用`DefaultMyModelRepository`还是`NetworkMyModelRepository`来创建实例。

## 回顾MVP 和 MVVM

框架中，用到ViewModel。先回顾下[MVP和MVVM的概念](https://mp.weixin.qq.com/s/Ss1KzfDpcYfwbuSPLQb-wQ).

## 了解Hilt基础知识

框架采用`Hilt`来实现减少代码量。 `Hilt`是**依赖注入（Dependency Injection，DI）框架**，[了解Hilt基础知识](https://mp.weixin.qq.com/s/nD8F05FO-aRYehi4PqIs0Q);

## 快速入门Compose

框架使用`Comose`来编写UI页面，赶紧跟上[快速入门 Jetpack Compose](https://mp.weixin.qq.com/s/70SYePs5Iouz3_1paSHfDg).



## DONE

技术更新太多了，此框架用到好多新技术。比之前传统的框架是优雅了好多，同时也增加阅读代码的难度。等熟悉后，代码看起来是真的清爽！！！
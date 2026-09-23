---
title: 快速入门JetpackCompose
date: 2026-09-24 00:21:29
tags:
---

写了那么多年的xml，突然说现在不流行了。**现在直接用 Kotlin 函数描述 UI**，即使用`Compose`的 **UI 开发方式**。学起来！！！

------

# **一、最简单的 Compose 页面**

传统 XML：

```xml
<LinearLayout
    android:orientation="vertical">
    <TextView
        android:text="Hello" />
    <Button
        android:text="登录" />
</LinearLayout>
```

Compose：

```kotlin
@Composable
fun LoginScreen() {
    Column {
        Text("Hello")

        Button(
            onClick = {
                // 登录
            }
        ) {
            Text("登录")
        }
    }
}
```

对应关系非常直观：

```text
XML                         Compose

LinearLayout(vertical)  →   Column
LinearLayout(horizontal)→   Row
FrameLayout             →   Box
TextView                →   Text
Button                  →   Button
ImageView               →   Image
RecyclerView            →   LazyColumn
```

所以看到：

```kotlin
@Composable
fun LoginScreen() {
    ...
}
```

这个`LoginScrren`没有返回值，仅仅是**描述构建和更新UI**，也就是声明了`UI`。

------

# 二、@Composable 声明一个UI

Compose 最核心的东西就是：`@Composable` 。

普通 Kotlin 函数：

```kotlin
fun getUserName(): String
```

是执行逻辑。

而：

```kotlin
@Composable
fun UserName()
```

表示：这是一个可以参与 Compose UI 构建的函数。

所以看到：

```kotlin
@Composable
fun LoginScreen()

@Composable
fun UserItem()

@Composable
fun UserAvatar()
```

基本都可以理解成：

```text
LoginScreen → 一个页面
UserItem    → 一块 UI
UserAvatar  → 一个小组件
```

注意：**Composable 不一定是完整页面。**

它可以大到整个页面，也可以小到一个按钮。

------

# **三、Compose 不再需要 XML**

传统 Android：

```text
LoginActivity.kt
        ↓
activity_login.xml
```

Compose：

```text
LoginActivity.kt
        ↓
LoginScreen()
```

例如：

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            LoginScreen()
        }
    }
}
```

这里：`setContent {}`, 可以简单理解成传统的：

```java
setContentView(...)
```

只不过以前加载 XML：

```java
setContentView(R.layout.activity_main);
```

现在加载 Compose：

```kotlin
setContent {
    LoginScreen()
}
```

------

# **四、Column、Row、Box**

这是 Compose 最基本的三个布局。

## **1. Column**

```kotlin
Column {
    Text("A")
    Text("B")
    Text("C")
}
```

效果：

```text
A
B
C
```

类似：

```xml
<LinearLayout
    android:orientation="vertical" />
```

------

## **2. Row**

```kotlin
Row {
    Text("A")
    Text("B")
}
```

效果：

```text
A B
```

类似：

```xml
<LinearLayout
    android:orientation="horizontal" />
```

------

## **3. Box**

```kotlin
Box {
    Image(...)
    Text("头像")
}
```

里面的组件可以叠在一起。

类似以前：

```text
FrameLayout
```

因此先记：

```text
Column = 竖着排
Row    = 横着排
Box    = 叠着排
```

------

# 五、Modifier 设置UI的属性

设置`Text`的宽和间距：

```kotlin
Text(
    text = "Hello",
    modifier = Modifier
        .fillMaxWidth()
        .padding(16.dp)
)
```

`Modifier` 就是：**这个 View 的布局参数 + 外观参数 + 行为参数。**

传统 XML：

```xml
<TextView
    android:layout_width="match_parent"
    android:padding="16dp"
    android:background="@color/white" />
```

Compose：

```kotlin
Text(
    "Hello",
    modifier = Modifier
        .fillMaxWidth()
        .padding(16.dp)
        .background(Color.White)
)
```

所以：`Modifier.fillMaxWidth()`  ≈  `layout_width="match_parent"`

而：`Modifier.padding(16.dp)` ≈  `padding`。

```kotlin
Modifier
    .fillMaxWidth()
    .height(50.dp)
    .padding(16.dp)
    .background(Color.White)
    .clickable { }
```

设置属性：

```text
宽度
高度
间距
背景
点击事件
```

------

# **六、Modifier 的顺序很重要**

这和 XML 不一样。`Modifier`是从左到右一层一层包装。

例如：

```kotlin
Modifier
    .padding(16.dp)
    .background(Color.Red)  // 红色背景不包含padding的部分
```

和：

```kotlin
Modifier
    .background(Color.Red)
    .padding(16.dp) // 红色的背景包含padding的部分
```

理解起来像是**链式调用**，一层实现后返回结果给下一层。顺序会影响最终布局、绘制和交互效果。

------

# 七、Text

传统：

```xml
<TextView
    android:text="Hello"
    android:textSize="16sp" />
```

Compose：

```kotlin
Text(
    text = "Hello",
    fontSize = 16.sp
)
```

常见参数：

```kotlin
Text(
    text = "Hello",
    color = Color.Black,
    fontSize = 16.sp,
    fontWeight = FontWeight.Bold,
    maxLines = 1,
    overflow = TextOverflow.Ellipsis
)
```

设置`Text`的属性。 如果是布局、外观、交互修饰使用`Modifier`。

------

# 八、Button

Compose：

```kotlin
Button(
    onClick = {
        login()
    }
) {
    Text("登录")
}
```

设置了`onClick`属性值，再包含一个`Text`。 Compose 经常采用：

```text
组件 {
    子组件
}
```

------

# 九、Image

相当于` ImageView`。

```kotlin
Image(
    painter = painterResource(R.drawable.logo),
    contentDescription = "Logo"
)
```

网络图片项目中经常会看到` Coil`：

```kotlin
AsyncImage(
    model = user.avatar,  // URL,Uri,File,drawable resource,ImageRequest
    contentDescription = null
)
```

可以理解成以前 Glide：

```text
Glide.with(...)
    .load(url)
    .into(imageView)
```

------

# 十、Surface是Material风格容器

```kotlin
Surface {
    Text("Hello")
}
```

**Material 风格的容器。**

它可以负责：

```text
背景颜色
圆角
边框
阴影
elevation
```

例如：

```kotlin
Surface(
    color = Color.White,
    shape = RoundedCornerShape(12.dp)
) {
    Text("Hello")
}
```

可以简单理解为一个：

```text
带 Material 样式能力的容器
```

------

# 十一、Scaffold 页面骨架

一个页面的整体骨架，和`Flutter`的`Scaffold`很像。

例如：

```kotlin
Scaffold(
    topBar = {
        TopAppBar(
            title = {
                Text("首页")
            }
        )
    },
    bottomBar = {
        BottomAppBar {
            Text("底部")
        }
    }
) { paddingValues ->

    HomeContent(
        modifier = Modifier.padding(paddingValues)
    )
}
```

结构就是：

```text
Scaffold
├── TopBar
├── Content
├── BottomBar
├── FloatingActionButton
└── Snackbar
```

除了自带的数据，还可以添加子组件, 如`HomeContent`。

------

# 十二、MaterialTheme

经常看到：

```kotlin
MaterialTheme {
    App()
}
```

或者：

```kotlin
MaterialTheme.colorScheme.primary
```

可以理解成以前的：

```text
Theme
styles.xml
colors.xml
```

例如：

```kotlin
Text(
    color = MaterialTheme.colorScheme.primary
)
```

就是：使用当前 Material Theme 中定义的主色。

Material 3 常见：

```text
MaterialTheme.colorScheme
MaterialTheme.typography
MaterialTheme.shapes
```

分别管理：

```text
颜色
字体
形状
```

------

# 十三、State：Compose 最核心的思想

传统 Android：

```kotlin
textView.text = "登录成功"
progressBar.visibility = View.GONE
```

也就是： 主动修改 View。

Compose 思路不一样：**UI 是 State 的结果。**  像`MVVM`中`UI`根据`state`展示。

例如：

```kotlin
var loading by remember {
    mutableStateOf(false)
}
```

`remember`是重组过程中保存对象/值的机制。 这里设置`state`为`false` .

UI：

```kotlin
if (loading) {
    CircularProgressIndicator()
} else {
    Text("登录")
}
```

当：`loading = true` Compose 会自动重新计算需要变化的 UI。

你不需要：`progressBar.visibility = VISIBLE`.

------

# 十四、remember

`remember`是 `Compose`提供的一个重组(`Recomposition`)过程中保存对象/值的机制：

```kotlin
var count by remember {
    mutableStateOf(0)
}
```

`remember`返回的`mutableStateOf(Int)`, `mutableStateOf`创建一个`State`，`remember` 负责记住这个 `State`。 通过`count`直接访问`State.value`。

如果执行 `count = 1`，`State` 的值会变成 1，并触发相关 Composable 重组；重组时 `remember` 会保留之前的 State，因此 `count` 仍然是 1，而不会重新变成初始值 0。

------

# 十五、Recomposition是按State更新UI

**Recomposition = 重组**是 Compose 非常核心的概念：

```kotlin
@Composable
fun Counter() {
    var count by remember {
        mutableStateOf(0)
    }
    Text("数量：$count")
    Button(
        onClick = {
            count++
        }
    ) {
        Text("增加")
    }
}
```

点击：`count++`, 改变了`count`的值, 也就是`State`改变，以后：

```text
State 改变
   ↓
Compose 发现 Text 使用了 count
   ↓
重新执行相关 Compose UI
   ↓
显示新的数字
```

**注意：**重组不等于把整个 Activity/View 树全部销毁重建。 Compose 会尽量只更新受到状态影响的部分。

------

# 十六、State Hoisting：状态提升

Compose 项目里非常常见，`State`在页面内部声明。**不推荐的示范**：

```kotlin
@Composable
fun LoginButton() {
    var loading by remember {
        mutableStateOf(false)
    }
}
```

更常见的是：

```kotlin
@Composable
fun LoginButton(
    loading: Boolean,
    onLoginClick: () -> Unit
) {
    Button(
        onClick = onLoginClick
    ) {
        Text(
            if (loading) "登录中..." else "登录"
        )
    }
}
```

更加推荐的是将状态放在外面。这是` Compose `非常重要的一种组件形式。子组件只接收：

```text
状态
+
事件
```

状态向下传，事件向上传。 实现**UI和状态管理分离**.

------

# 十七、Compose 的单向数据流

Compose 项目一般是：

```text
          State
ViewModel ───────→ UI
    ↑                    │
    │                    │
    └──── Event ────┘
```

**态向下传，事件向上传**。例如：

```text
用户点击登录
     ↓
UI
     ↓
viewModel.login()
     ↓
Repository
     ↓
ViewModel 修改 State
     ↓
Compose 收到 State
     ↓
重新显示 UI
```

这就是：**Unidirectional Data Flow，单向数据流。**

------

# 十八、ViewModel

相当于`MVVM`中的  `ViewModel`。

```kotlin
class LoginViewModel : ViewModel() {
    var loading by mutableStateOf(false)
        private set

    fun login() {
        loading = true
    }
}
```

UI：

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = viewModel()
) {
    if (viewModel.loading) {
        CircularProgressIndicator()
    }
}
```

核心关系：

```text
ViewModel
   ↓
State
   ↓
Compose UI
```

**`ViewModel`** **持有并管理 ** **`State`**，通过 **`Repository`** **获取或更新数据；`UI` **观察 `State`，并根据状态展示不同内容。

------

# 十九、viewModel() 获取可用的viewModel

使用的时候：

```kotlin
val viewModel: LoginViewModel = viewModel()
```

其中`viewModel()`会推导出`viewModel<LoginViewModel>()`。

通过 Android `Compose` 提供的 `ViewModel` 获取机制：获取当前 `ViewModelStoreOwner` 对应的 `LoginViewModel`。

Android `ViewModel` 生命周期不是和`Activity`的实例绑定，而是与`ViewModelStoreOwner`的作用域绑定。

所以配置变化导致`Activity`重建时`ViewModel`会保留。

`Activity`真正结束（`Finish`）时`ViewModel`才会被清除。

------

# 二十、hiltViewModel()

使用 Hilt 的项目经常看到：

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = hiltViewModel()
)
```

使用`hiltViewModel()`替代了`viewModel()`：

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()
```

可以简单理解：

```text
hiltViewModel()
      ↓
Hilt 帮你获取/创建 ViewModel
      ↓
同时把 Repository 等依赖注入进去
```

比`viewModel()`多了自动创建对象和添加依赖的功能。

------

# 二十一、StateFlow

`StateFlow`是一种状态数据流，一个可以被观察、会持续发布最新状态的容器。

项目中使用：

```kotlin
private val _uiState = MutableStateFlow(LoginUiState())

val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()
```

简单理解：

```text
MutableStateFlow
       ↓
ViewModel 内部可以修改。使用_uiState进行值的修改。

StateFlow
       ↓
UI 对外观察。 uiState只能观察，不能修改。
```

例如：

```kotlin
data class LoginUiState(
    val loading: Boolean = false,
    val username: String = "",
    val error: String? = null
)
```

ViewModel：

```kotlin
private val _uiState = MutableStateFlow(LoginUiState())

val uiState = _uiState.asStateFlow()
```

可以修改`_uiState`的值。

Compose：

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

然后：

```kotlin
if (uiState.loading) {
    CircularProgressIndicator()
}
```

在`UI`中观察`uiState`的值变化。

`ViewModel` 负责管理状态，`StateFlow` 负责保存并持续发布状态，`Compose` 负责订阅状态并刷新 UI。

------

# 二十二、collectAsStateWithLifecycle()

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

实现了：**把 ViewModel 的 StateFlow 转成 Compose 可以观察的 State，并结合 Android Lifecycle 管理收集。**

于是：

```text
StateFlow 改变
       ↓
Compose State 改变
       ↓
UI 重组
```

------

# 二十三、by 是属性委托

Compose 中用到`by`是Kotlin的**属性委托(Property Delegation)**：

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

**作用**：`by`把`State<T>`拆出来，直接使用里面的`T`，省略 `.value `。

Compose 大量使用。

------

# 二十四、LazyColumn

它基本可以理解成：`RecyclerView` 竖向长列表。

```kotlin
LazyColumn {
    items(users) { user ->
        UserItem(user)
    }
}
```

相当于以前：

```text
RecyclerView
    ↓
Adapter
    ↓
ViewHolder
    ↓
item_user.xml
```

Compose 不需要这些东西，直接：

```text
LazyColumn
   ↓
items
   ↓
UserItem
```

所以代码会少很多。

------

# 二十五、LazyRow

就是横向列表：

```kotlin
LazyRow {
    items(users) { user ->
        UserItem(user)
    }
}
```

可以理解为：

```text
RecyclerView
+
LinearLayoutManager.HORIZONTAL
```

------

# 二十六、rememberLazyListState()

`Compose` 中用来**创建并记住** **`LazyColumn`** **/** **`LazyRow`** **的滚动状态** **`LazyListState`** 的。记住列表当前滚到哪里，并且允许你读取、控制列表滚动。

```kotlin
val listState = rememberLazyListState()

LazyColumn(
    state = listState
) {
}
```

它类似保存 RecyclerView 的：

```text
滚动状态
滚动位置
```

* `listState.scrollToItem(10) `  滚到第10个item
* `listState.animateScrollToItem(50)`  带动画滚动第50个item
* `listState.firstVisibleItemIndex` 当前第一个可见Item的下标
* `listState.firstVisibleItemScrollOffset` 第一个可见Item滚出去的像素
* `listState.canScrollForward`  是否还能**朝列表末尾**滚动
* `listState.canScrollBackward`   是否还能**朝列表开头**滚动
* `listState.isScrollInProgress` 当前是否正在滚动

------

# 二十七、Navigation

Android之前是使用`ActivityManager`来管理返回栈。现在`Compose` 项目通常不再依赖：

```text
Activity A
→ Activity B
→ Activity C
```

来实现每个页面。

更常见：

```text
Single Activity
       ↓
Compose Navigation
       ↓
多个 Screen
```

例如：

```kotlin
NavHost(
    navController = navController,
    startDestination = "home"
) {

    composable("home") {
        HomeScreen()
    }

    composable("detail") {
        DetailScreen()
    }
}
```

可以理解成：

```text
NavHost
├── home
│   └── HomeScreen
│
└── detail
    └── DetailScreen
```

也就是`Activity`中拥有一整套`Compose Navigation Back Stack`，进行页面之间的push和pop。

------

# 二十八、NavController

跳转到 detail 页面：

```kotlin
navController.navigate("detail")
```

Android以前是：

```kotlin
startActivity(...)
```

返回到上一页， `Compose`：

```kotlin
navController.popBackStack()
```

Android之前操作：

```text
finish / 返回上一页
```

但是底层的实现已经不一样了。

**Compose 的** **`NavController.navigate()`** **和 Flutter 的** **`Navigator.push()`** **很类似。**

------

# 二十九、rememberNavController()

**创建并记住一个** **`NavController`**，用于管理 Compose `Navigation` 的页面导航和返回栈。

例如：

```kotlin
val navController = rememberNavController()
```

在`Activity`创建并保存： Compose Navigation 的导航控制器。

然后：

```kotlin
NavHost(navController)
```

负责显示页面，并设置好多个`composable`,之后使用`navController.navigate()`进行跳转和`navController.popBackStack()`进行返回。

------

# 三十、LaunchedEffect 副作用

```kotlin
LaunchedEffect(Unit) {
    viewModel.loadData()
}
```

此方法为：在`Composable`生命周期内执行协程副作用(side effect)。

例如加载数据：

```kotlin
LaunchedEffect(Unit) {
    viewModel.loadUsers()
}
```

执行条件：

1. 当 Composable 进入界面，
2. 指定的 key 发生变化时，启动一段协程代码。 `Unit`为 `key`

------

# 三十一、DisposableEffect

进入`Composition`时执行一段代码，离开`Composition`时执行清理代码。

适合需要`注册 -> 注销`这种成对操作的场景。

```kotlin
DisposableEffect(Unit) {
    // 进入 Composition，或者 key 变化时执行
    // 注册、监听、创建资源……
    registerListener()

    onDispose {
        // 离开 Composition，或者 key 变化时执行
        // 注销、取消监听、释放资源……
        unregisterListener()
    }
}
```

可以理解成：

```text
进入
→ 注册

离开
→ 清理
```

适合：

```text
Listener
Receiver
Observer
需要释放的资源
```

------

# 三十二、SideEffect

每次`Compose`成功完成一次`Recomposition`后，执行一段非`Compose`代码。

```kotlin
SideEffect {
    // 同步 Compose 状态到外部对象
}
```

每次成功重组以后执行一些副作用。也就是每次UI刷新成功后，就执行一次`SideEffect`中的代码。

------

# 三十三、remeberSaveable

`remember`能在`Recomposition`中保留状态；`remeberSaveable`除此之外，还能在`Activity`因配置变化等场景被销毁重建后恢复状态。

```kotlin
var text by rememberSaveable {
    mutableStateOf("")
}
```

`rememberSaveable`会使用`Android`的`saved instance state`机制保存可保存的数据。

可保存的数据的类型：整型、字符串、boolean等。复杂对象需要`Saver`或`parcelable`等方式。

------

# 三十四、TextField

相当于传统Android的`XML / View`中的`EditText`，都用于文本输入。

```kotlin
var username by remember {
    mutableStateOf("")
}

TextField(
    value = username,
    onValueChange = {
        username = it
    }
)
```

这是非常典型的 Compose：

```text
State
↓
TextField

TextField 输入
↓
onValueChange
↓
修改 State
↓
UI 更新
```

------

# 三十五、 value 和 onValueChange 分开

```kotlin
TextField(
    value = username,
    onValueChange = {
        username = it
    }
)
```

这是 Compose 的核心思想，`Compose`的"状态驱动UI"，单向数据流：

```text
value
↓
现在应该显示什么

onValueChange
↓
用户修改了什么
```

逻辑： `username` 发生变化 → Compose 安排重组 → `TextField(...)` 重新执行 → 新的 `username` 作为 `value` 传给 `TextField` → UI 更新。

------

# 三十六、Slot API 组件嵌套组件

Compose 很喜欢这种设计`Slot API`：

```kotlin
@Composable
fun MyCard(
    content: @Composable () -> Unit
) {
    Card {
        content()
    }
}
```

使用：

```kotlin
MyCard {
    Text("Hello")
}
```

可以简单理解成：

```text
组件给你留一个位置
↓
你自己往里面塞 UI
```

Material 3 大量使用这种方式。

------

# 三十七、Card

```kotlin
Card {
    Text("用户信息")
}
```

就是 Material 卡片容器。

通常自带：

```text
Shape
背景
Elevation
```

可以把它理解成自己实现的：`MaterialCardView`.

------

# 三十八、Spacer

以前可能写：

```xml
<View
    android:layout_height="16dp" />
```

Compose：

```kotlin
Spacer(
    modifier = Modifier.height(16.dp)
)
```

就是：空白占位。

------

# 三十九、Arrangement  排列

例如：

```kotlin
Column(
    verticalArrangement = Arrangement.spacedBy(16.dp)
)
```

表示：

```text
每个子组件之间间隔 16dp
```

还有：

```kotlin
Arrangement.Center
Arrangement.SpaceBetween
Arrangement.SpaceAround
```

可以理解成：**子 View 怎么排列**。

------

# 四十、Alignment

例如：

```kotlin
Column(
    horizontalAlignment = Alignment.CenterHorizontally
)
```

就是：子组件怎么对齐。

例如：

```text
Center
CenterHorizontally
CenterVertically
Start
End
```

------

# 四十一、weight

传统Android中的XML：

```text
android:layout_weight = "1"
```

Compose：

```kotlin
Row {
    Text(
        "左边",
        modifier = Modifier.weight(1f)
    )

    Text("右边")
}
```

概念基本一样。

------

# 四十二、fillMaxWidth / fillMaxSize

```kotlin
Modifier.fillMaxWidth()
```

≈

```text
宽度占满父容器
Modifier.fillMaxHeight()
```

≈

```text
高度占满
Modifier.fillMaxSize()
```

≈

```text
宽高全部占满
```

------

# 四十三、wrapContentSize

让组件按照**自身内容**需要的大小来布局，并且可以指定它在可用空间里的对齐方式。

类似传统Android的XML中的：

```text
wrap_content
```

不过 Compose 默认测量机制和传统 View 不完全一样。

------

# 四十四、padding

```kotlin
Modifier.padding(16.dp)
```

设置组件的边距，也可以上下左右设置：

```kotlin
Modifier.padding(
    start = 16.dp,
    top = 8.dp,
    end = 16.dp,
    bottom = 8.dp
)
```

------

# 四十五、clickable

按钮或View的点击事件。传统Android：

```kotlin
view.setOnClickListener {
}
```

Compose：

```kotlin
Modifier.clickable {
    viewModel.login()
}
```

例如：

```kotlin
Text(
    text = "点击我",
    modifier = Modifier.clickable {
        // 点击
    }
)
```

------

# 四十六、条件 UI

传统Android显示或隐藏View：

```text
View.VISIBLE
View.GONE
```

Compose 不需要，而是直接：

```kotlin
if (loading) {
    CircularProgressIndicator()
}
```

如果：

```kotlin
loading == false
```

那么这个 UI 就不存在。

例如：

```kotlin
if (error != null) {
    Text(error)
}
```

------

# 四十七、when 控制 UI

例如：

```kotlin
when (state) {
    is Loading -> {
        CircularProgressIndicator()
    }

    is Success -> {
        UserList()
    }

    is Error -> {
        ErrorView()
    }
}
```

这在 Compose 项目里非常常见。

可以理解：

```text
不同 State
↓
显示不同 UI
```

------

# 四十八、Preview  预览视图

你会经常看到：

```kotlin
@Preview
@Composable
fun LoginScreenPreview() {
    LoginScreen()
}
```

这个不是正式业务逻辑。

它用于：

Android Studio 直接预览 Compose UI。

类似以前 XML 的 Design Preview。

------

# 四十九、LocalContext

在`Composable`里需要使用传统`Android`的`Context`时,通过`LocalContext.current`获取。

```kotlin
val context = LocalContext.current
```

可以理解成：获取当前 `Android Context`。

例如：

```kotlin
Toast.makeText(
    context,
    "登录成功",
    Toast.LENGTH_SHORT
).show()
```

------

# 五十、LocalXXX

Compose 中经常出现：

```text
LocalContext.current
LocalDensity.current
LocalConfiguration.current
LocalLifecycleOwner.current
```

这些通常来自`CompositionLocal`, 从当前 Compose 环境中获取某个上下文信息。

------

# 五十一、CompositionLocal

一种向下层`COmposable`隐式传递数据的机制。

以前很多对象可能：

```text
一层一层参数传递
```

Compose 可以通过 `CompositionLocal`：

```text
上层提供
↓
下面很多 Composable 获取
```

例如 `val context = LocalContext.current`直接获取`context`。

------

# 五十二、Coroutine 协程

`Compose` 项目基本离不开 `Kotlin Coroutine`。

```kotlin
viewModelScope.launch {
    repository.login()
}
```

其中`viewModelScope.launch`的作用为：

```text
启动一个协程执行异步任务
```

可以进行很多耗时的操作，例如：

```text
网络请求
数据库
BLE
文件操作
```

`viewModelScope`是`ViewModel`自带的`CoroutineScope`（协程作用域）。生命周期由`ViewModel`自动管理。

------

# 五十三、rememberCoroutineScope

在`Composable`中获取一个与当前`Composition`生命周期绑定的`CoroutineScope`,可以在事件回调里手动启动协程。

Compose 里可能看到：

```kotlin
val scope = rememberCoroutineScope()

Button(
    onClick = {
        scope.launch {
            // 异步操作
        }
    }
)
```

就是：获取一个与当前 Compose 生命周期相关的 `CoroutineScope`。

当函数是`suspend`的时候，则需要`CoroutineScope`启动协程来执行。

------

# 五十四、示例代码

示例代码：

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = hiltViewModel(),
    onLoginSuccess: () -> Unit
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(uiState.loginSuccess) {
        if (uiState.loginSuccess) {
            onLoginSuccess()
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = {
                    Text("登录")
                }
            )
        }
    ) { padding ->

        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            TextField(
                value = uiState.username,
                onValueChange = {
                    viewModel.updateUsername(it)
                },
                label = {
                    Text("用户名")
                }
            )

            Button(
                onClick = {
                    viewModel.login()
                },
                modifier = Modifier.fillMaxWidth()
            ) {
                if (uiState.loading) {
                    CircularProgressIndicator()
                } else {
                    Text("登录")
                }
            }
        }
    }
}
```

现在可以逐行翻译。

------

# 五十五、逐行翻译上面的代码

```kotlin
@Composable
fun LoginScreen(...)
```

定义一个 Compose 登录页面。

------

```kotlin
viewModel: LoginViewModel = hiltViewModel()
```

获取 LoginViewModel，并由 Hilt 处理依赖注入。

------

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

观察 ViewModel 的 UI 状态。

------

```kotlin
LaunchedEffect(uiState.loginSuccess)
```

loginSuccess 改变时执行对应逻辑。

------

```kotlin
Scaffold
```

建立页面整体结构。

------

```kotlin
TopAppBar
```

顶部标题栏。

------

```kotlin
Column
```

内容竖向排列。

------

```kotlin
Modifier
    .fillMaxSize()
    .padding(...)
```

设置页面大小和间距。

------

```kotlin
TextField
```

输入框。

------

```kotlin
value = uiState.username
```

输入框显示 ViewModel State 中的 username。

------

```kotlin
onValueChange = {
    viewModel.updateUsername(it)
}
```

用户输入发生变化：

```text
UI
↓
ViewModel
```

------

```kotlin
Button(
    onClick = {
        viewModel.login()
    }
)
```

用户点击登录：

```text
UI
↓
ViewModel.login()
```

------

```kotlin
if (uiState.loading)
```

根据 State 决定显示：

```text
加载动画
or
登录文字
```

整个页面其实就是：

```text
UI 显示 State

+

UI 把 Event 交给 ViewModel
```

------

# 五十六、再加上 ViewModel

例如：

```kotlin
data class LoginUiState(
    val username: String = "",
    val loading: Boolean = false,
    val loginSuccess: Boolean = false
)
```

ViewModel：

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState = _uiState.asStateFlow()

    fun updateUsername(username: String) {
        _uiState.update {
            it.copy(username = username)
        }
    }

    fun login() {
        viewModelScope.launch {
            _uiState.update {
                it.copy(loading = true)
            }

            repository.login(_uiState.value.username)

            _uiState.update {
                it.copy(
                    loading = false,
                    loginSuccess = true
                )
            }
        }
    }
}
```

抓住主线：

```text
Compose
   │
   │ login()
   ↓
ViewModel
   │
   ↓
Repository
   │
   ↓
Network
```

结果：

```text
Network
   ↓
Repository
   ↓
ViewModel
   ↓
修改 UiState
   ↓
StateFlow
   ↓
Compose
   ↓
UI 自动变化
```

------

# 五十七、data class + copy()

Compose 项目特别常见：

```kotlin
data class LoginUiState(
    val loading: Boolean = false,
    val username: String = ""
)
```

更新：

```kotlin
_uiState.update {
    it.copy(
        loading = true
    )
}
```

意思不是修改原对象。而是：

```text
旧 LoginUiState
↓
复制一个新的 LoginUiState
↓
loading 改成 true
↓
其他属性保持不变
```

这和不可变 `State `的设计有关。 看到：

```kotlin
state.copy(...)
```

基本就理解成：创建一个修改部分字段的新 `State`。

------

# 五十八、remember 和 ViewModel State 选择

### 1. 页面临时 UI 状态

```text
Dialog 是否打开
某个菜单是否展开
局部动画状态
```

选择`remember`.

### 2. 业务状态

```text
用户信息
登录状态
设备数据
网络请求结果
订单列表
```

选择： `ViewModel + StateFlow`



------

# 五十九、传统 Android 与 Compose 完整映射

| **传统 Android**        | **Compose**                     |
| ----------------------- | ------------------------------- |
| XML                     | Composable                      |
| View                    | Composable                      |
| TextView                | Text                            |
| EditText                | TextField                       |
| Button                  | Button                          |
| ImageView               | Image                           |
| LinearLayout vertical   | Column                          |
| LinearLayout horizontal | Row                             |
| FrameLayout             | Box                             |
| RecyclerView            | LazyColumn / LazyRow            |
| MaterialCardView        | Card                            |
| Toolbar                 | TopAppBar                       |
| setContentView          | setContent                      |
| layout_width            | Modifier.width / fillMaxWidth   |
| layout_height           | Modifier.height / fillMaxHeight |
| padding                 | Modifier.padding                |
| background              | Modifier.background             |
| OnClickListener         | onClick / Modifier.clickable    |
| View.VISIBLE/GONE       | if / when                       |
| Adapter                 | items                           |
| ViewHolder              | Composable Item                 |
| styles.xml / Theme      | MaterialTheme                   |
| Activity 页面跳转       | Navigation Compose              |
| View 更新 UI            | State 驱动 UI                   |
| setText()               | 修改 State                      |
| Observer                | collectAsStateWithLifecycle     |
| ViewModel               | ViewModel，仍然存在             |
| LiveData                | 常见 StateFlow                  |
| Dagger/Hilt             | Hilt，仍然可以使用              |

### DONE

第一次研究`Compose`，是从Flutter的角度来理解的。有不对的地方，请多多在评论下指教，多谢！！！


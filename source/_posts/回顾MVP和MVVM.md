---
title: 回顾MVP和MVVM
date: 2026-09-21 22:14:28
tags:
---
看到Google提供的架构模版[architecture-templates](https://github.com/android/architecture-templates/tree/base)，代码是真的精炼，寥寥几个文件将（1）UI和数据层隔离，（2）采用MVVM架构，（3）单向数据流，设计得明明白白。一直在维护Java + Xml的老项目，对于现在Android的新架构，值得好好学习一下。

**MVP：Presenter 告诉 View“你应该怎么显示”。** （已经过时了）
**MVVM：ViewModel 告诉 View“现在是什么状态”，View 自己决定怎么显示。**

### **一、MVP**

MVP = **Model - View - Presenter**

| **层**    | **作用**                      |
| --------- | ----------------------------- |
| Model     | 数据、网络、数据库、业务数据  |
| View      | 页面展示、用户交互            |
| Presenter | 处理业务逻辑，并主动控制 View |

```
View
  ↕
Presenter
  ↕
Model
```

例如登录：

```text
用户点击登录
    ↓
View
    ↓
Presenter.login()
    ↓
Repository（从 DataSource：API / Database / BLE / Cache 获取数据）
    ↓
Presenter （控制view执行业务操作）
    ↓
View.showLoading()
View.showSuccess()
View.showError()
```

Presenter 通常会持有一个 View 接口：

```java
interface LoginView {
    void showLoading();
    void showSuccess();
    void showError(String message);
}
```

Presenter：

```java
class LoginPresenter {
    LoginView view;
    void login() {
        view.showLoading();
        // 请求接口...
        view.showSuccess();
    }
}
```

所以 MVP 最大的特点是：**Presenter 主动命令 View 做什么。**

比如：

```text
显示 Loading
隐藏 Loading
显示错误
跳转页面
刷新列表
```

Presenter 对 View 的控制比较直接。

------

### **二、MVVM**

MVVM = **Model - View - ViewModel**

```
View
  ↕
ViewModel
  ↕
Model
```

核心思想变成：**ViewModel 不直接操作 View，而是提供“状态”，View 根据状态自己更新。**

例如：

```text
用户点击登录
    ↓
ViewModel.login()
    ↓
Repository（从 DataSource：API / Database / BLE / Cache 获取数据）
    ↓
ViewModel 更新 State
    ↓
View 观察 State
    ↓
自动刷新 UI
```

例如：

```text
LoginState
├── loading
├── success
└── error
```

ViewModel：

```text
login() {
    state = Loading;
    // 请求接口
    state = Success;
}
```

View：

```text
监听 state

Loading → 显示进度条
Success → 显示主页
Error   → 显示错误信息
```

因此 ViewModel 不需要知道：

```text
Activity
Fragment
UIViewController
Widget
```

它只负责：**现在是什么状态**. `View`根据`State`展示不同内容。

### **三、MVP和MVVM对比**

`MVP`和`MVVM`都有`Model`和`View`, 不同的是中间层`Presenter`和`ViewModel`。

| **对比**                          | **MVP**                      | **MVVM**                  |
| --------------------------------- | ---------------------------- | ------------------------- |
| 中间层                            | Presenter                    | ViewModel                 |
| UI 更新                           | Presenter 主动调用 View      | View 观察 State           |
| View 接口                         | 通常需要                     | 通常不需要                |
| ViewModel/Presenter 是否知道 View | Presenter 通常知道 View 接口 | ViewModel 尽量不知道 View |
| **数据绑定/状态监听**             | 不强调                       | 核心思想                  |
| UI 与逻辑耦合                     | 较低                         | 更低                      |
| 单元测试                          | 容易                         | 容易                      |
| 状态驱动 UI                       | 较弱                         | 很适合                    |
| 现代移动端开发                    | 使用减少                     | 非常常见                  |

### 四、 Flutter / Android / iOS  的MVVM实现

| **平台**    | **ViewModel 常用实现**                                       | **UI 如何监听状态**                                   |
| ----------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| Flutter     | Riverpod `Notifier` / `AsyncNotifier`、Provider + ChangeNotifier | `ref.watch()` / `Consumer`                            |
| Android     | Jetpack `ViewModel`                                          | `StateFlow` + Compose `collectAsStateWithLifecycle()` |
| iOS SwiftUI | `@Observable` 对象                                           | SwiftUI Observation 自动跟踪                          |
| iOS UIKit   | 自己定义 ViewModel + Observation/Combine/闭包等              | 订阅/绑定状态                                         |

### Flutter

Flutter 使用 Riverpod / Provider / Bloc 进行状态管理是实现**状态驱动 UI**：

```
UI watch(state)
   ↑
  Notifier
   ↓
 Repository
```

### **Android**

Android 现在有非常明确的官方 `ViewModel`：

```kotlin
class LoginViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState = _uiState.asStateFlow()

    fun login() {
        // Repository 获取数据
        // 更新 _uiState
    }
}
```

Compose：

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = viewModel()
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    // 根据 state 绘制 UI
}
```

整个关系就是：

```text
Compose UI
    ↓
Android ViewModel
    ↓
Repository
    ↓
DataSource
    ↓
API / Room / BLE / Cache

ViewModel
    ↓
StateFlow
    ↓
Compose UI
```

因此 Android 这套组合非常典型： **ViewModel + StateFlow + Jetpack Compose**

其中 `ViewModel` 负责保存和管理 UI 状态，`StateFlow` 负责状态流转，`Compose`观察状态并刷新 UI。

------

### **iOS**

在 SwiftUI 中，可以使用 Observation 的 `@Observable`：

```swift
@Observable
class LoginViewModel {
    var isLoading = false
    var errorMessage: String?

    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func login() async {
        isLoading = true
        // repository.login()
        isLoading = false
    }
}
```

SwiftUI：

```swift
struct LoginView: View {
    @State private var viewModel = LoginViewModel(...)
    var body: some View {
        if viewModel.isLoading {
            ProgressView()
        }
    }
}
```

关系：

```text
SwiftUI View
    ↓
ViewModel（@Observable）
    ↓
Repository
    ↓
DataSource
    ↓
API / Core Data / BLE / Cache
    ↓
ViewModel
    ↓
Observable State
    ↓
SwiftUI View
```

以前 SwiftUI 项目中则很常见：

```text
ObservableObject
@Published
@StateObject
@ObservedObject
```

例如：

```swift
class LoginViewModel: ObservableObject {
    @Published var isLoading = false
}
```

现在的新项目可以优先了解新的 Observation / `@Observable`。



## DONE






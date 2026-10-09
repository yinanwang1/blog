---
title: SwiftUI语法-5分钟复习
date: 2026-10-09 15:22:18
tags:
---

接连着学习了`Flutter`,`Jetpack Compose`,`Dart语法`，`Swift语法`,`React Native`和现在的`SwiftUI`后，感觉到了**千变一律**啊，那种返璞归真的错觉。似乎再发展几年，所有的语言和UI相关，都可以回归到同一套了。大家功能相似，作用一样，搞那么多没用啊。

## 1.SwiftUI 基本结构

SwiftUI 使用 `View` 描述界面：

```swift
import SwiftUI
struct ContentView: View {
    var body: some View {
        Text("Hello SwiftUI")
    }
}
```

核心：

```text
View
 ↓
body
 ↓
some View
 ↓
声明式描述 UI
```

## 2. App 入口

SwiftUI App 通常从 `@main` 开始：

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

结构：

```text
App
 ↓
Scene
 ↓
WindowGroup
 ↓
ContentView
```

## 3. Text

显示文字：

```swift
Text("Hello SwiftUI")
    .font(.title)
    .fontWeight(.bold)
    .foregroundStyle(.blue)
```

字符串插值：

```swift
let name = "Tom"
Text("Hello \(name)")
```

## 4. Image

系统图标：

```swift
Image(systemName: "heart.fill")
    .font(.title)
    .foregroundStyle(.red)
```

资源图片：

```swift
Image("avatar")
    .resizable()
    .scaledToFit()
    .frame(width: 100, height: 100)
```

## 5. Button

按钮：

```swift
Button("登录") {
    print("login")
}
```

自定义内容：

```swift
Button {
    print("login")
} label: {
    Label("登录", systemImage: "person")
}
```

## 6. VStack

垂直布局：

```swift
VStack(spacing: 10) {
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

## 7. HStack

水平布局：

```swift
HStack {
    Text("A")
    Text("B")
    Text("C")
}
```

效果：

```text
A B C
```

## 8. ZStack

层叠布局：

```swift
ZStack {
    Color.blue
    Text("Hello")
        .foregroundStyle(.white)
}
```

类似：

```text
背景
 ↑
前景
```

## 9. Spacer

占据剩余空间：

```swift
HStack {
    Text("左边")
    Spacer()
    Text("右边")
}
```

常用于：

```text
左边                 右边
```

## 10. Divider

分割线：

```swift
VStack {
    Text("第一行")
    Divider()
    Text("第二行")
}
```

## 11. Modifier

SwiftUI 大量使用 Modifier：

```swift
Text("Hello")
    .font(.title)
    .foregroundStyle(.blue)
    .padding()
    .background(.gray)
```

可以理解为：

```text
View
 ↓ modifier
View
 ↓ modifier
View
```

Modifier 顺序可能影响最终效果。**SwiftUI Modifier 不是“给原 View 设置一堆属性”，而是前一个 Modifier 产生新 View，后一个 Modifier 再修饰这个新 View。**

```swift
padding
frame
background
overlay
clipShape
border
offset
scaleEffect
rotationEffect
opacity
gesture
contentShape
```

需要特别注意这些涉及**尺寸、布局、绘制范围、裁剪、变换、点击区域**的 Modifier。

## 12. frame

设置 View 的布局尺寸：

```swift
Text("Hello").frame(width: 200, height: 100)
```

占满可用宽度：

```swift
Text("Hello").frame(maxWidth: .infinity)
```

## 13. padding

设置内边距：

```swift
Text("Hello") .padding()
```

指定方向：

```swift
Text("Hello")
    .padding(.horizontal, 20)
    .padding(.vertical, 10)
```

## 14. background

设置背景：

```swift
Text("Hello")
    .padding()
    .background(.blue)
```

配合圆角：

```swift
Text("Hello")
    .padding()
    .background(.blue)
    .clipShape(RoundedRectangle(cornerRadius: 10))
```

## 15. @State

View 自己管理的状态：

```swift
struct CounterView: View {
    @State private var count = 0
    var body: some View {
        Button("Count: \(count)") {
            count += 1
        }
    }
}
```

核心：

```text
State 改变
   ↓
SwiftUI 检测变化
   ↓
重新计算 body
   ↓
更新需要变化的 UI
```

## 16. @Binding

父组件把状态的“读写能力”传给子组件：

```swift
struct ParentView: View {
    @State private var name = ""
    var body: some View {
        ChildView(name: $name)
    }
}
struct ChildView: View {
    @Binding var name: String
    var body: some View {
        TextField("姓名", text: $name)
    }
}
```

理解：

```text
Parent
@State name
   ↓ $name
Child
@Binding name
```

`ParentView`传值到`ChildView`时使用的是`$name`，有就是传递的是`Binding<String>`.

## 17. @Observable

现代 SwiftUI 常使用 Observation 管理模型状态：

```swift
@Observable
class UserModel {
    var name = "Tom"
    var age = 20
}
```

使用：

```swift
struct ContentView: View {
    @State private var model = UserModel()
    var body: some View {
        Text(model.name)
    }
}
```

`model.name` 改变后，依赖它的 UI 可以自动更新。

## 18. @StateObject

传统 `ObservableObject` 模式中，如果 View 创建并持有对象：

```swift
class UserViewModel: ObservableObject {
    @Published var name = "Tom"
}
struct ContentView: View {
    @StateObject private var vm = UserViewModel()
    var body: some View {
        Text(vm.name)
    }
}
```

关系：

```text
ObservableObject
      +
  @Published
      ↓
@StateObject
      ↓
     View
```

**`@StateObject`** **用来让 SwiftUI View 创建、持有并观察一个** **`ObservableObject`** **对象。**

传统的 `ObservableObject + @Published` 状态管理体系。建议使用`@Observable`。

## 19. @ObservedObject

对象由外部创建，当前 View 只是观察：

```swift
struct UserView: View {
    @ObservedObject var vm: UserViewModel
    var body: some View {
        Text(vm.name)
    }
}
```

区别：

```text
@StateObject
→ 当前 View 创建/持有
@ObservedObject
→ 外部传入，当前 View 观察
```

建议使用`@Observable`。

```swift
传统方案
ObservableObject
   +
@Published
   +
@StateObject / @ObservedObject

现代方案
@Observable
   +
普通属性
   +
@State / @Environment 等
```

## 20. @Environment

读取 SwiftUI 环境中的值：

```swift
@Environment(\.dismiss)
private var dismiss
```

从当前 View 的 SwiftUI Environment 中取出 `dismiss` 这个能力，赋给 `dismiss` 属性。使用：

```swift
Button("关闭") {
    dismiss()
}
```

还可以：

```swift
@Environment(\.colorScheme)
private var colorScheme
```

**`@Environment`** **用来从 SwiftUI 当前 View 所处的“环境”中读取一个值。**

#### 最常用的 `@Environment`

| **属性**                                 | **类型/概念**             | **作用**                                      |
| ---------------------------------------- | ------------------------- | --------------------------------------------- |
| `dismiss`                                | `DismissAction`           | 关闭当前 sheet / presentation，或退出当前呈现 |
| `colorScheme`                            | `ColorScheme`             | 当前浅色/深色模式                             |
| `locale`                                 | `Locale`                  | 当前语言、地区环境                            |
| `calendar`                               | `Calendar`                | 当前日历系统                                  |
| `timeZone`                               | `TimeZone`                | 当前时区                                      |
| `scenePhase`                             | `ScenePhase`              | App/Scene 当前 active、inactive、background   |
| `horizontalSizeClass`                    | `UserInterfaceSizeClass?` | 横向 compact / regular                        |
| `verticalSizeClass`                      | `UserInterfaceSizeClass?` | 纵向 compact / regular                        |
| `dynamicTypeSize`                        | `DynamicTypeSize`         | 用户当前动态字体大小                          |
| `layoutDirection`                        | `LayoutDirection`         | 左→右或右→左布局                              |
| `displayScale`                           | `CGFloat`                 | 屏幕缩放比例                                  |
| `isEnabled`                              | `Bool`                    | 当前 View 是否允许交互                        |
| `isPresented`                            | `Bool`                    | 当前 View 是否处于被呈现状态                  |
| `isSearching`                            | `Bool`                    | 当前是否处于搜索状态                          |
| `editMode`                               | `Binding<EditMode>?`      | List 等是否处于编辑状态                       |
| `openURL`                                | `OpenURLAction`           | 打开 URL                                      |
| `refresh`                                | `RefreshAction?`          | 获取当前刷新操作                              |
| `undoManager`                            | `UndoManager?`            | Undo / Redo 管理                              |
| `modelContext`                           | `ModelContext`            | SwiftData 数据上下文                          |
| `managedObjectContext`                   | `NSManagedObjectContext`  | Core Data 上下文                              |
| `accessibilityReduceMotion`              | `Bool`                    | 用户是否开启“减弱动态效果”                    |
| `accessibilityVoiceOverEnabled`          | `Bool`                    | VoiceOver 是否开启                            |
| `accessibilityDifferentiateWithoutColor` | `Bool`                    | 是否要求不能只依赖颜色区分信息                |

## 21. TextField

输入框：

```swift
@State private var name = ""
var body: some View {
    TextField("请输入姓名", text: $name)
}
```

通常绑定：

```text
TextField
   ↕
Binding
   ↕
State
```

## 22. Toggle

开关：

```swift
@State private var enabled = false
var body: some View {
    Toggle("开启通知", isOn: $enabled)
}
```

`Toggle`选中后`enabled`值变化。

## 23. Picker

选择器：

```swift
@State private var index = 0
var body: some View {
    Picker("选择", selection: $index) {
        Text("Apple").tag(0)
        Text("Google").tag(1)
    }
}
```

`selection`当前选中的值，用户选择后`index`的值也会直接变化。

## 24. ScrollView

滚动容器, 像`UIScrollView`：

```swift
ScrollView {
    VStack {
        ForEach(0..<100) { index in
            Text("Item \(index)")
        }
    }
}
```

横向：

```swift
ScrollView(.horizontal) {
    HStack {
        // ...
    }
}
```

## 25. List

列表，像`UITableView`：

```swift
List {
    Text("Apple")
    Text("Google")
    Text("Microsoft")
}
```

动态数据：

```swift
let names = ["Apple", "Google", "Microsoft"]
List(names, id: \.self) { name in
    Text(name)
}
```

提供的详细功能，见文尾的补充。

## 26. ForEach

根据数据创建 View：

```swift
VStack {
    ForEach(0..<5) { index in
        Text("第 \(index) 行")
    }
}
```

模型通常实现：

```swift
struct User: Identifiable {
    let id: UUID
    let name: String
}
```

然后：

```swift
ForEach(users) { user in
    Text(user.name)
}
```

## 27. LazyVStack / LazyHStack

大量数据时延迟创建：

```swift
ScrollView {
    LazyVStack {
        ForEach(0..<1000) { index in
            Text("Item \(index)")
        }
    }
}
```

对应关系：

```text
VStack      普通垂直布局
LazyVStack  延迟创建垂直内容
HStack      普通水平布局
LazyHStack  延迟创建水平内容
```

## 28. NavigationStack

现代 SwiftUI 导航：

```swift
NavigationStack {
    NavigationLink("详情") {
        DetailView()
    }
    .navigationTitle("首页")
}
```

基本结构：

```text
NavigationStack
      ↓
     Home
      ↓
NavigationLink
      ↓
    Detail
```

可以设置多个链接：

```swift
NavigationStack {
    List {
        NavigationLink {
            ContentView()
        } label: {
            Label("个人资料", systemImage: "person")
        }

        NavigationLink {
            ContentView()
        } label: {
            Label("设置", systemImage: "gear")
        }
    }
    .navigationTitle("我的")
}
```

## 29. NavigationPath

代码控制导航栈：

```swift
@State private var path = NavigationPath()
var body: some View {
    NavigationStack(path: $path) {
        Button("详情") {
            path.append(100)
        }
        .navigationDestination(for: Int.self) { userId in
            UserDetailView(userId: userId)
        }
    }
}
```

**`NavigationPath`** **就是 SwiftUI 中用数据描述导航栈，**`append()` **相当于 Push，**`removeLast()`**相当于 Pop，**`navigationDestination`负责定义“某种数据应该显示哪个页面”。

将`$path`传到子页面，在子页面可以继续`append（）`到其他子页面。

## 30. TabView

底部 Tab：

```swift
TabView {
    HomeView()
        .tabItem {
            Label("首页", systemImage: "house")
        }
    ProfileView()
        .tabItem {
            Label("我的", systemImage: "person")
        }
}
```

## 31. sheet

弹出页面：

```swift
@State private var showDetail = false
var body: some View {
    Button("打开") {
        showDetail = true
    }
    .sheet(isPresented: $showDetail) {
        DetailView()
    }
}
```

当`showDetail`为`true`的时候展示`sheet`。通过`Modifier`提供：

```swift
.padding()
.background()
.frame()
.sheet()
.alert()
.navigationTitle()
```

也就是`SwiftUI View`都可以挂这些`Modfier`。

## 32. alert

弹出 Alert：

```swift
@State private var showAlert = false
var body: some View {
    Button("删除") {
        showAlert = true
    }
    .alert("确认删除？", isPresented: $showAlert) {
        Button("删除", role: .destructive) {
            delete()
        }
        Button("取消", role: .cancel) {}
    }
}
```

## 33. task

View 出现后执行异步任务：

```swift
struct UserView: View {
    @State private var name = ""
    var body: some View {
        Text(name)
            .task {
                name = await loadUser()
            }
    }
}
```

常用于：

```text
页面出现
 ↓
.task
 ↓
调用 async API
 ↓
修改 State
 ↓
刷新 UI
```

在`task`中调用`ViewModel`中的方法进行数据的获取。`task`与`View`的生命周期关联，当`View`消失时，会主动取消`task`的请求。

## 34. onAppear / onDisappear

监听 View 出现和消失：

```swift
Text("Hello")
    .onAppear {
        print("appear")
    }
    .onDisappear {
        print("disappear")
    }
```

## 35. onChange

监听状态变化：

```swift
@State private var name = ""
var body: some View {
    TextField("姓名", text: $name)
        .onChange(of: name) {
            print("name changed")
        }
}
```

当`name`发生变化的时候，`onChange`被调用。

## 36. animation

状态变化时执行动画：

```swift
@State private var expanded = false
var body: some View {
    Rectangle()
        .frame(
            width: expanded ? 200 : 100,
            height: 100
        )
        .animation(.easeInOut, value: expanded)
        .onTapGesture {
            expanded.toggle()
        }
}
```

也可以：

```swift
@State private var expanded = false
  var body: some View {
    VStack {
      Rectangle()
        .fill(.blue)
        .frame(
          width: expanded ? 200 : 100,
          height: 100
        )

      Button("切换大小") {
        withAnimation {
          expanded.toggle()
        }
      }
    }
  }
```

## 37. Gesture

点击：

```swift
Text("点击")
    .onTapGesture {
        print("tap")
    }
```

长按：

```swift
Text("长按")
    .onLongPressGesture {
        print("long press")
    }
```

## 38. GeometryReader

获取父布局提供的尺寸：

```swift
GeometryReader { geometry in
    Text("宽度：\(geometry.size.width)")
        .frame(width: geometry.size.width)
}
```

适合需要根据容器尺寸计算布局的场景，但不应把它当成所有布局问题的默认方案。

## 39. safeArea

忽略安全区域：

```swift
Color.blue
    .ignoresSafeArea()
```

常用于全屏背景。

## 40. ViewBuilder

允许一个闭包声明多个 View：

```swift
@ViewBuilder
func header() -> some View {
    Text("标题")
    Text("副标题")
}
```

自定义容器：

```swift
struct Card<Content: View>: View {
    @ViewBuilder
    let content: () -> Content
    var body: some View {
        VStack {
            content()
        }
        .padding()
    }
}
```

使用：

```
Card {
    Text("Hello SwiftUI")
    Text("Hello World")
}
```

`ViewBuilder`接受一个闭包，里面可以放置多个`View`.

## 41. some View

SwiftUI 的 `body`：

```swift
var body: some View {
    Text("Hello")
}
```

`some` 是一个关键字，表示  **不透明返回类型**（opaque return type）, `some View`表示：

```text
返回一个确定的 View 类型。
但调用者不需要知道具体类型
```

`some View`表示确定的`View`类型。

使用`any View`表示不同的`View`类型。

## 42. 条件 UI

根据状态显示不同 View：

```swift
if isLogin {
    HomeView()
} else {
    LoginView()
}
```

也可以：

```swift
if loading {
    ProgressView()
} else {
    ContentView()
}
```

## 43. ProgressView

加载状态：

```swift
ProgressView("加载中...")
```

进度：

```swift
struct ContentView: View {
    @State private var progress: Double = 1.0
    var body: some View {
        VStack {
            ProgressView("加载中")
            
            ProgressView(value: progress, total: 100.0)
            
            Button("点我") {
                progress += 1.0
            }
        }
    }
}
```

## 44. 自定义 View

SwiftUI 推荐拆分小组件：

```swift
struct UserCard: View {
    let name: String
    var body: some View {
        HStack {
            Image(systemName: "person.circle")
            Text(name)
        }
        .padding()
    }
}
```

使用：

```swift
UserCard(name: "Tom")
```

## 45. ViewModifier

复用一组 Modifier：

```swift
struct CardStyle: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.white)
            .clipShape(RoundedRectangle(cornerRadius: 12))
            .shadow(radius: 3)
    }
}
```

使用：

```swift
Text("Hello")
    .modifier(CardStyle())
```

`ViewModifier`作用是：**把一组 View 样式或行为封装起来，方便在不同的 View 上重复使用。**

## 46. Preview

预览 View：

```swift
#Preview {
    ContentView()
}
```

可以快速查看组件效果，不需要每次运行整个 App。

## 47. SwiftUI + MVVM

常见项目结构：

```text
View
 ↓ 用户操作
ViewModel
 ↓
Repository / Service
 ↓
Network / Database
```

例如：

```swift
@Observable
class UserViewModel {
    var users: [User] = []
    func load() async {
        users = await repository.loadUsers()
    }
}
```

View：

```swift
struct UserView: View {
    @State private var vm = UserViewModel()
    var body: some View {
        List(vm.users) { user in
            Text(user.name)
        }
        .task {
            await vm.load()
        }
    }
}
```

做到“各司其职”。

## 48. SwiftUI 最重要的数据流

SwiftUI 最核心的思想不是控件，而是：

```text
State
  ↓
View = f(State)
  ↓
用户操作
  ↓
修改 State
  ↓
重新计算 View
```

例如：

```swift
@State private var count = 0
Button("Count \(count)") {
    count += 1
}
```

不是：

```text
找到 UILabel
↓
修改 UILabel.text
```

而是：

```text
修改数据
↓
SwiftUI 根据数据重新描述 UI
```

## 49. UIKit 与 SwiftUI 思维对比

```text
UIKit
UIViewController
    ↓
创建 UIView
    ↓
设置约束
    ↓
找到控件
    ↓
手动修改 UI

SwiftUI
State
    ↓
body
    ↓
声明 UI
    ↓
State 改变
    ↓
SwiftUI 自动更新 UI
```

核心转变：

```text
UIKit：命令式 UI
SwiftUI：声明式 UI
```



# 补充

## List提供的功能

## **1. Row：列表中的一行**

`List` 里面的每一个元素通常就是一个 Row。

```swift
List {
    Text("张三")
    Text("李四")
    Text("王五")
}
```

效果：

```text
┌──────────────┐
│ 张三          │ ← Row
├──────────────┤
│ 李四          │ ← Row
├──────────────┤
│ 王五          │ ← Row
└──────────────┘
```

Row 可以是复杂 View：

```swift
List(users) { user in
    HStack {
        Image(systemName: "person.circle")
        VStack(alignment: .leading) {
            Text(user.name)
            Text(user.phone)
                .font(.caption)
        }
    }
}
```

## **2. Section：列表分组**

可以把多个 Row 分成不同组：

```swift
List {
    Section("账号") {
        Text("个人资料")
        Text("修改密码")
    }
    Section("系统") {
        Text("通知")
        Text("隐私")
    }
}
```

效果类似 iPhone 设置：

```text
账号
┌──────────────┐
│ 个人资料      │
│ 修改密码      │
└──────────────┘
系统
┌──────────────┐
│ 通知          │
│ 隐私          │
└──────────────┘
```

## **3. Header / Footer：分组头和分组尾**

`Section` 可以有 Header 和 Footer。

```swift
List {
    Section {
        Text("Wi-Fi")
        Text("蓝牙")
    } header: {
        Text("网络")
    } footer: {
        Text("修改网络相关设置")
    }
}
```

结构：

```text
网络                 ← Header
┌─────────────────┐
│ Wi-Fi           │
│ 蓝牙             │
└─────────────────┘
修改网络相关设置     ← Footer
```

Header 通常是**标题**，Footer 通常是**补充说明**。

## **4. 选择 Selection：选中某一行**

例如：

```swift
@State private var selection: String?
var body: some View {
    List(selection: $selection) {
        Text("Apple")
            .tag("Apple")
        Text("Google")
            .tag("Google")
        Text("Microsoft")
            .tag("Microsoft")
    }
}
```

当用户选择：

```text
Apple
```

那么：

```swift
selection == "Apple"
```

它在 iPad/macOS 的侧边栏、主从布局等场景尤其常见。

## **5. 删除 onDelete**

`List + ForEach` 可以直接提供系统列表删除行为：

```swift
@State private var users = [
    "张三",
    "李四",
    "王五"
]
var body: some View {
    List {
        ForEach(users, id: \.self) { user in
            Text(user)
        }
        .onDelete { indexSet in
            users.remove(atOffsets: indexSet)
        }
    }
}
```

用户可以通过系统提供的列表交互删除 Row。
核心：

```text
用户删除某行
    ↓
onDelete
    ↓
得到 IndexSet
    ↓
删除数据
    ↓
List 更新
```

## 6. 移动 onMove

允许用户调整 `Row` 顺序：

```swift
List {
    ForEach(users, id: \.self) { user in
        Text(user)
    }
    .onMove { source, destination in
        users.move(
            fromOffsets: source,
            toOffset: destination
        )
    }
}
```

比如：

```text
原来
1 张三
2 李四
3 王五
用户拖动
   ↓
1 王五
2 张三
3 李四
```

常用于：

```text
频道排序
收藏排序
播放列表排序
功能入口排序
```

## **7. Swipe Actions：左滑/右滑操作**

这是非常常见的功能。

```swift
List(users) { user in
    Text(user.name)
        .swipeActions {
            Button("删除", role: .destructive) {
                delete(user)
            }
        }
}
```

用户滑动一行：

```text
┌──────────────────────────┐
│ 张三              │ 删除 │
└──────────────────────────┘
```

也可以多个按钮：

```swift
.swipeActions {
    Button("删除", role: .destructive) {
        delete(user)
    }
    Button("收藏") {
        favorite(user)
    }
}
```

类似：

```text
邮件 App
消息 App
```

里面常见的左滑删除、标记等操作。

## **8. 列表样式 listStyle**

`List` 自带不同的系统列表样式。
例如：

```swift
List {
    Section("账号") {
        Text("个人资料")
        Text("修改密码")
    }
}
.listStyle(.insetGrouped)
```

常见：

```swift
.listStyle(.plain)
.listStyle(.inset)
.listStyle(.grouped)
.listStyle(.insetGrouped)
```

你可以粗略理解：

```text
plain
→ 普通连续列表
grouped
→ 分组列表
insetGrouped
→ 类似 iPhone 设置 App 的圆角分组效果
```

具体视觉表现会根据 iOS/macOS 和系统版本有所变化。

## **9. 编辑模式 EditMode**

List 可以进入编辑状态。
最经典：

```swift
List {
    ForEach(users, id: \.self) { user in
        Text(user)
    }
    .onDelete { indexSet in
        users.remove(atOffsets: indexSet)
    }
    .onMove { source, destination in
        users.move(
            fromOffsets: source,
            toOffset: destination
        )
    }
}
.toolbar {
    EditButton()
}
```

点击：

```text
编辑
 ↓
List 进入 EditMode
 ↓
可以删除 / 拖动排序
 ↓
完成
```

如果需要自己读取编辑状态：

```swift
@Environment(\.editMode)
private var editMode
```



# do-try-catch的使用

```swift
do {
    let user = try loadUser()
    print(user.name)
} catch {
    print("加载失败：\(error)")
}
```

注意三个关键字：

| **关键字** | **作用**               |
| ---------- | ---------------------- |
| `do`       | 定义错误处理作用域     |
| `try`      | 标记可能抛出错误的调用 |
| `catch`    | 捕获并处理错误         |

比`Java/ Kotlin / JavaScript`语言多了一个`do`关键字。而`try`的作用也有区别：

### **1.** **`try`**：正常的错误处理

```swift
do {
    let result = try login(
        username: "admin",
        password: "123456"
    )

    print(result)
} catch {
    print(error)
}
```

特点：

- 成功：返回正常结果。
- 失败：抛出错误。
- 错误可以由 `catch` 捕获，也可以继续向上抛出。

**这是最常用、最标准的写法。**

### **2.** **`try?`**：将错误转换为 nil

```swift
let result = try? login(
    username: "admin",
    password: "wrong"
)
```

如果登录失败：`result == nil`

如果成功：`result == Optional("登录成功")`

**不关心具体错误原因，只关心成功还是失败。**

### **3.** **`try!`**：确信不会出错

```swift
let result = try! login(
    username: "admin",
    password: "123456"
)
```

如果成功：`返回正常结果`

如果抛出错误：`程序运行时触发致命错误`

**实际开发中尽量避免** **`try!`**，除非能够确保不会抛出错误。

### 4. 单独使用`try`

```swift
func loadData() throws -> String {
    let name = try loadUser()
    return name
}
```

### 5. try await一起使用

```swift
func loadUser() async throws -> String {
    try await Task.sleep(for: .seconds(1))
    return "张三"
}
```

| **关键字** | **作用**                               |
| ---------- | -------------------------------------- |
| `async`    | 函数为异步函数                         |
| `throws`   | 函数可能抛出错误                       |
| `try`      | 函数可能抛出错误                       |
| `await`    | 函数调用可能发生挂起，等待异步操作完成 |



## 结尾
`AI`又来替代手搓代码，那代码的差异性进一步缩小了。`AI`眼中所有**语言**都是一样的吧。很期待5年或10年后，程序员行业会变为怎样。希望我还是站在风里，跟着一直成长！！！












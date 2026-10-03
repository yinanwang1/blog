---
title: ReactNative快速入门
date: 2026-10-03 23:58:56
tags:
---
创建了一个React Native的项目，运行起来看了看。想到了第一次运行Flutter后的感觉，简单的页面，简单的代码，同时可运行到2个平台。写RN的代码，和写网页一模一样。如果排除iOS和Android的区别，那就是界面的编写啊。

React Native 的核心可以先理解成：

```text
React
+
TypeScript / JavaScript
+
React Native 原生组件
+
移动端能力
```

React Native 使用 React 的组件、Props、State、Hooks 等编程模型，但最终渲染的是 Android/iOS 原生 UI。

## 一、整体认识

一个典型页面：

```tsx
import React, {useState} from 'react';
import {
  View,
  Text,
  Button,
  StyleSheet,
} from 'react-native';

export default function HomeScreen() { // 定义组件/页面
  const [count, setCount] = useState(0); // 状态

  // 描述 UI
  return (
    // 容器
    <View style={styles.container}> // style 为样式
      // 文本
      <Text>点击次数：{count}</Text>

      // 按钮
      <Button
        title="点击"
        // 点击事件
        onPress={() => setCount(count + 1)} // 修改状态, React 重新渲染 UI
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
  },
});
```

React Native 官方也把 JSX、Component、Props、State 作为 React 基础核心。

# 第一部分：TypeScript / JavaScript 基础

## 1.`const`和`let`

```tsx
const name = 'Tom';
let age = 20;

age = 21;
```

理解：

```text
const → 变量不能重新赋值
let   → 可以重新赋值
```

RN 项目中 `const` 极其常见。

## 2. 基本类型

```tsx
const name: string = 'Tom';
const age: number = 20;
const enabled: boolean = true;
```

数组：

```tsx
const users: string[] = ['Tom', 'Jack'];
```

对象：

```tsx
const user = {
  name: 'Tom',
  age: 20,
};
```

## 3.`interface`/`type`

React Native 项目大量使用 TypeScript 定义数据结构。

```tsx
interface User {
  id: number;
  name: string;
  age: number;
}

const user: User = {
  id: 1,
  name: 'Tom',
  age: 20,
};
```

也经常看到：

```tsx
type User = {
  id: number;
  name: string;
};
```

官方的新 React Native 项目默认面向 TypeScript。

## 4. 可选属性`?`

```tsx
interface User {
  name: string;
  age?: number;
}
```

表示：

```text
name → 必须有
age  → 可以没有
```

所以：

```tsx
const user: User = {
  name: 'Tom',
};
```

合法。

## 5. 箭头函数

```tsx
const add = (a: number, b: number) => {
  return a + b;
};
```

简写：

```tsx
const add = (a: number, b: number) => a + b;
```

例如：

```tsx
<Button onPress={() => login()} />
```

这里：

```tsx
() => login()
```

就是一个函数。

## 6. 解构

对象：

```tsx
const user = {
  name: 'Tom',
  age: 20,
};

const {name, age} = user;
```

数组：

```tsx
const [a, b] = [10, 20];
```

React 最经典的代码：

```tsx
const [count, setCount] = useState(0);
```

本质就是**数组解构**。

## 7. 展开运算符`...`

```tsx
const user = {
  name: 'Tom',
  age: 20,
};

const newUser = {
  ...user,
  age: 21,
};
```

结果：

```tsx
{
  name: 'Tom',
  age: 21
}
```

React 状态更新时非常常见：

```tsx
setUser({
  ...user,
  name: 'Jack',
});
```

## 8.`map`

React 代码中极其重要：

```tsx
const users = ['Tom', 'Jack'];

users.map(name => {
  return <Text>{name}</Text>;
});
```

意思：

```text
users
 ↓
逐个取元素
 ↓
转换成 UI
```

## 9.`filter`

```tsx
const users = [
  {name: 'Tom', age: 20},
  {name: 'Jack', age: 15},
];

const adults = users.filter(user => user.age >= 18);
```

## 10.`async / await`

网络请求大量使用：

```tsx
async function loadUser() {
  const response = await fetch('/user');
  const data = await response.json();

  return data;
}
```

理解：

```text
async
  ↓
函数中允许 await

await
  ↓
等待 Promise 完成
```

# 第二部分：JSX

## 11. JSX 是什么

React Native UI：

```tsx
<View>
  <Text>Hello</Text>
</View>
```

这不是 HTML，而是 JSX。

React Native 官方明确说明，RN 使用 React，但使用 Native Components 作为 UI 构建块。

可以先类比：

```text
React Native       Android        iOS

View               ViewGroup      UIView
Text               TextView       UITextView
Image              ImageView      UIImageView
TextInput          EditText       UITextField
ScrollView         ScrollView     UIScrollView
```

官方核心组件也采用类似映射。

## 12. JSX 中使用变量`{}`

```tsx
const name = 'Tom';

return (
  <Text>Hello {name}</Text>
);
```

显示：

```text
Hello Tom
```

JSX 中：

```tsx
{}
```

表示：**这里执行 JavaScript 表达式。**

## 13. 条件渲染

```tsx
{loading ? (
  <Text>加载中...</Text>
) : (
  <Text>加载完成</Text>
)}
```

也经常看到：

```tsx
{user && <Text>{user.name}</Text>}
```

意思：

```text
user 存在
    ↓
显示 Text
```

# 第三部分：Component

## 14. 函数组件

现在最常见：

```tsx
function HomeScreen() {
  return (
    <View>
      <Text>首页</Text>
    </View>
  );
}
```

或者：

```tsx
const HomeScreen = () => {
  return (
    <Text>首页</Text>
  );
};
```

组件本质就是：

```text
输入 Props
   ↓
函数
   ↓
返回 JSX
```

## 15. 自定义组件

```tsx
function UserCard() {
  return (
    <View>
      <Text>Tom</Text>
    </View>
  );
}
```

使用：

```tsx
<UserCard />
```

组件可以继续组合组件，这也是 React 的核心思想。

# 第四部分：Props

## 16. Props

Props 可以理解成：**父组件传给子组件的参数。**

```tsx
function UserCard(props) {
  return (
    <Text>{props.name}</Text>
  );
}
```

调用：

```tsx
<UserCard name="Tom" />
```

更常见的是解构：

```tsx
function UserCard({name}) {
  return (
    <Text>{name}</Text>
  );
}
```

## 17. TypeScript 定义 Props

```tsx
type Props = {
  name: string;
  age: number;
};

function UserCard({name, age}: Props) {
  return (
    <Text>
      {name} - {age}
    </Text>
  );
}
```

这类代码在实际 RN 项目中非常常见。

# 第五部分：State

## 18.`useState`

React 最重要的知识点之一：

```tsx
const [count, setCount] = useState(0);
```

分别是：

```text
count       当前状态

setCount    修改状态的方法

0           初始值
```

例如：

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <Button
      title={`${count}`}
      onPress={() => setCount(count + 1)}
    />
  );
}
```

流程：

```text
count = 0
   ↓
显示 0
   ↓
点击
   ↓
setCount(1)
   ↓
状态变化
   ↓
组件重新执行/渲染
   ↓
显示 1
```

`useState` 的 setter 会请求 React 使用新状态重新渲染组件。

这一点和声明式 UI 框架的思想很接近：

```text
State
 ↓
UI
```

# 第六部分：事件

## 19.`onPress`

```tsx
<Button
  title="登录"
  onPress={() => {
    console.log('login');
  }}
/>
```

也可以：

```tsx
const login = () => {
  console.log('login');
};

<Button
  title="登录"
  onPress={login}
/>
```

注意：

```tsx
onPress={login}
```

是**传函数**。

而：

```tsx
onPress={login()}
```

是立即执行。这个区别非常重要。

## 20.`Pressable`

实际项目经常使用：

```tsx
<Pressable
  onPress={() => console.log('click')}>
  <Text>点击我</Text>
</Pressable>
```

`Pressable` 是 React Native 官方核心交互组件之一。

# 第七部分：常用 UI 组件

## 21.`View`

最重要的容器：

```tsx
<View>
  <Text>Hello</Text>
</View>
```

可以理解成通用布局容器。

## 22.`Text`

React Native 的文字通常必须放在 `Text` 中：

```tsx
<Text>Hello React Native</Text>
```

## 23.`Image`

```tsx
<Image
  source={{uri: user.avatar}}
  style={{
    width: 100,
    height: 100,
  }}
/>
```

本地图片：

```tsx
<Image
  source={require('./logo.png')}
/>
```

## 24.`TextInput`

```tsx
const [text, setText] = useState('');

<TextInput
  value={text}
  onChangeText={setText}
/>
```

流程：

```text
用户输入
 ↓
onChangeText
 ↓
setText()
 ↓
text 改变
 ↓
重新渲染
 ↓
value={text}
```

## 25.`ScrollView`

```tsx
<ScrollView>
  <Text>内容1</Text>
  <Text>内容2</Text>
  <Text>内容3</Text>
</ScrollView>
```

用于可滚动内容。

官方核心组件包括 `View`、`Text`、`Image`、`TextInput`、`Pressable`、`ScrollView` 等。

# 第八部分：Style

## 26.`style`

```tsx
<Text
  style={{
    fontSize: 20,
    fontWeight: 'bold',
  }}>
  Hello
</Text>
```

注意 RN 不是传统 CSS：

```text
font-size
```

写成：

```tsx
fontSize
```

## 27.`StyleSheet.create`

实际项目更常见：

```tsx
const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
  },

  title: {
    fontSize: 20,
    fontWeight: 'bold',
  },
});
```

使用：

```tsx
<View style={styles.container}>
  <Text style={styles.title}>Hello</Text>
</View>
```

# 第九部分：Flexbox 布局

## 28.`flexDirection`

React Native 布局核心是 Flexbox。

```tsx
<View
  style={{
    flexDirection: 'row',
  }}>
```

表示横向：

```text
A B C
```

而：

```tsx
flexDirection: 'column'
```

表示：

```text
A
B
C
```

## 29.`justifyContent`

控制**主轴**：

```tsx
justifyContent: 'center'
```

常见：

```text
flex-start
center
flex-end
space-between
space-around
space-evenly
```

## 30.`alignItems`

控制**交叉轴**：

```tsx
alignItems: 'center'
```

经典居中：

```tsx
<View
  style={{
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  }}>
```

## 31.`flex: 1`

非常常见：

```tsx
<View style={{flex: 1}}>
```

通常可以先理解为：**尽量占满父容器剩余空间。**

# 第十部分：列表

## 32.`FlatList`

实际 App 最重要组件之一：

```tsx
const users = [
  {id: '1', name: 'Tom'},
  {id: '2', name: 'Jack'},
];

<FlatList
  data={users}
  keyExtractor={item => item.id}
  renderItem={({item}) => (
    <Text>{item.name}</Text>
  )}
/>
```

核心就是：

```text
data
 ↓
FlatList
 ↓
逐条调用 renderItem
 ↓
生成 UI
```

官方建议一般列表使用 `FlatList` 或 `SectionList`；与普通 `ScrollView` 相比，`FlatList` 会围绕当前显示区域进行列表渲染，更适合长列表。

# 第十一部分：Hooks

## 33. Hook 是什么

看到：

```text
useState
useEffect
useContext
useMemo
useCallback
useRef
```

这些 `useXXX` 基本就是 React Hooks。

Hooks 让函数组件使用状态、Effect、Context 等 React 能力。Hook 通常只能在组件或自定义 Hook 的顶层调用。

## 34.`useEffect`

这是看项目代码必须掌握的第二大 Hook。

```tsx
useEffect(() => {
  console.log('执行');
}, []);
```

最常见：

```tsx
useEffect(() => {
  loadData();
}, []);
```

可以先理解为：**组件出现后执行加载数据。**

## 35.`useEffect`依赖

```tsx
useEffect(() => {
  loadUser(userId);
}, [userId]);
```

理解：

```text
userId 改变
   ↓
Effect 重新执行
```

`useEffect` 官方定义更准确地说，是用于让组件与 React 外部系统同步。

## 36.`useEffect`清理

```tsx
useEffect(() => {
  const timer = setInterval(() => {
    console.log('timer');
  }, 1000);

  return () => {
    clearInterval(timer);
  };
}, []);
```

这里：

```tsx
return () => {}
```

是 cleanup。

常用于：

```text
取消监听
取消订阅
清理 Timer
断开连接
```

## 37.`useRef`

```tsx
const inputRef = useRef<TextInput>(null);
```

然后：

```tsx
inputRef.current?.focus();
```

通常用于：

```text
保存不需要触发渲染的值
获取组件引用
调用组件方法
```

和setState的区别是不触发UI的重新渲染。

## 38.`useMemo`

```tsx
const total = useMemo(() => {
  return calculateTotal(items);
}, [items]);
```

理解：

```text
items 没变化
 ↓
尽量复用之前计算结果
```

主要是性能优化。

## 39.`useCallback`

```tsx
const handlePress = useCallback(() => {
  console.log('click');
}, []);
```

用于缓存**函数引用**。

简单区分：

```text
useMemo      → 缓存一个值

useCallback  → 缓存一个函数
```

当函数当做参数传入到组件时，如果函数变了就会重新渲染组件。使用了`useCallback`后，当父组件重新渲染，此时函数不改变，那么使用的子组件不会重新渲染。

# 第十二部分：Context

## 40. `useContext`

当很多组件都需要：

```text
用户信息
主题
语言
登录状态
```

可以使用 Context。

创建：

```tsx
const UserContext = createContext(null);
```

提供：

```tsx
<UserContext.Provider value={user}>
  <App />
</UserContext.Provider>
```

使用：

```tsx
const user = useContext(UserContext);
```

理解：

```text
Provider
   ↓
组件树
   ↓
任意子组件
   ↓
useContext()
```

避免层层传 Props。

# 第十三部分：网络请求

## 41.`fetch`

```tsx
async function loadUsers() {
  const response = await fetch(
    'https://example.com/users',
  );

  const data = await response.json();

  setUsers(data);
}
```

典型流程：

```text
页面
 ↓
useEffect
 ↓
loadUsers()
 ↓
fetch()
 ↓
服务器
 ↓
JSON
 ↓
setUsers()
 ↓
State 更新
 ↓
UI 更新
```

实际项目也经常使用 Axios 等网络库。

# 第十四部分：Navigation

## 42. 页面导航

React Native 本身不规定唯一的路由方案；官方导航文档对入门者推荐 React Navigation，并介绍 Stack、Tab 等常见导航模式。

项目中经常看到：

```tsx
navigation.navigate('Detail');
```

理解成：

```text
Home
 ↓
Detail
```

## 43. 页面传参数

```tsx
navigation.navigate('Detail', {
  userId: 100,
});
```

目标页面：

```tsx
function DetailScreen({route}) {
  const {userId} = route.params;

  return (
    <Text>{userId}</Text>
  );
}
```

# 第十五部分：状态提升

## 44. State Hoisting

如果两个组件都需要同一份数据：

```text
Parent
│
├── ComponentA
│
└── ComponentB
```

不要各保存一份。

放到 Parent：

```tsx
function Parent() {
  const [name, setName] = useState('');

  return (
    <>
      <ComponentA
        name={name}
        onChange={setName}
      />

      <ComponentB name={name} />
    </>
  );
}
```

思想：

```text
State 放在共同父组件
        ↓
通过 Props 向下传
        ↓
通过 Callback 向上传事件
```

将 `setName` 作为 `onChange` 属性传给 `ComponentA`。当 `ComponentA` 中的值发生变化时，调用 `onChange(newName)`，实际上就是调用父组件的 `setName(newName)`，从而修改父组件的 `name` State。`name` 改变后 `Parent` 重新渲染，并把新的 `name` 通过 Props 传给 `ComponentB`。

# 第十六部分：自定义 Hook

## 45.`useXXX`

自定义的 Hook：

```tsx
function useUser() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);

  const login = async () => {
    // ...
  };

  return {
    user,
    loading,
    login,
  };
}
```

在页面中直接使用：

```typescript
const {
  user,
  loading,
  login,
} = useUser();
```

理解：**把一组状态和业务逻辑封装起来。**



# 第十七部分：平台差异

## 46.`Platform`

```tsx
import {Platform} from 'react-native';

if (Platform.OS === 'ios') {
  console.log('iOS');
}

if (Platform.OS === 'android') {
  console.log('Android');
}
```

也可能：

```tsx
const height = Platform.OS === 'ios' ? 100 : 80;
```

官方支持 `Platform.OS`、`Platform.select()` 等方式处理平台差异。

## 47.`.ios.tsx`/`.android.tsx`

项目可能存在：

```text
Button.ios.tsx
Button.android.tsx
```

代码：

```tsx
import Button from './Button';
```

React Native 会根据平台选择对应文件。这也是官方支持的平台差异组织方式。

# 第十八部分：原生能力

## 48. Native Module

如果 JS/TS 需要调用：

```text
蓝牙
NFC
定位
相机
厂商 SDK
Android API
iOS API
```

可能会看到 Native Module。

概念：

```text
TypeScript / JavaScript

        ↓

React Native Native Module

        ↓

Android
Kotlin / Java

        或

iOS
Swift / Objective-C
```

因此 React Native 并不意味着：

```text
完全不需要 Android / iOS
```

复杂项目经常仍然存在：

```text
android/
ios/
```

目录。

# 第十九部分：项目目录

## 49. 常见目录结构

实际项目可能是：

```text
project
│
├── android/
│
├── ios/
│
├── src/
│   │
│   ├── components/   通用 UI 组件
│   ├── screens/      页面
│   ├── navigation/   页面导航
│   ├── hooks/        自定义 Hooks
│   ├── services/     API、网络、业务服务
│   ├── stores/       全局状态
│   ├── utils/        工具方法
│   ├── types/        TypeScript 类型
│   └── assets/       图片等资源
│
├── App.tsx
├── package.json
└── tsconfig.json
```



# 第二十部分：第三方状态管理

## 50. Redux / Zustand 等

大型项目可能不会只使用：

```tsx
useState
```

而是使用全局状态库，例如：

```text
Redux
Zustand
```

看到 Redux 时，先理解核心关系即可：

```text
UI
 ↓
Action
 ↓
Store
 ↓
State 改变
 ↓
UI 更新
```



### 文献
React Native 官方文档
React Native：React Fundamentals
React Native：Core Components and APIs
React Native：TypeScript
React 官方文档
React：useState
React：useEffect
React Native：Navigation

## 基本介绍
在学习了 Android Compose、以及 Vue 框架的使用和概念，对声明式UI框架已经有较为清晰的认知，并且 React 的官方文档写得比较清晰，所以直接记录对比 Compose 和 Vue 的总结：


|分类 |	Android Jetpack Compose	| React 18+ |	Vue 3+	|核心作用与描述|
|---|---|---|---|---|
|基础组件 | Composable 函数 | Component (函数) | Component (.vue / setup) | Compose 和 React 都是纯函数；Vue 是带有响应式包装的对象|
|响应式状态 | val count = remember { mutableStateOf(0) } | const [count, setCount] = useState(0) |	const count = ref(0) |	数据变化 -> UI 自动刷新|
|计算属性 / 派生状态 | derivedStateOf() | useMemo() | computed() | 根据依赖自动计算缓存优化性能，避免重复计算|
|父传子 | 方法参数 @Composable fun Child(name: String) | 方法参数 function Child({ name }) | defineProps() | 单向数据流：父 -> 子 传值 |
|子传父 | 方法回调 onChange: (Int) -> Unit | 方法回调 | defineEmits() | 子 -> 父 通知 / 传参 |
|局部状态管理 | Compose 函数内 State | Component State | `<script setup>` 内变量 | 仅当前页面 / 组件使用 |
|全局状态管理 | ViewModel + Repository | Redux / Zustand / Jotai | Pinia | 跨页面共享数据(用户信息、购物车) |
|界面描述 | Kotlin 函数@Composable | JSXreturn (`<div />`) | Template / JSX`<template>` | 声明式描述 UI 长什么样 |


React的特色就是直接在 JavaScript 代码中返回声明式 UI，一个使用例子：
```jsx
import { useState, useEffect } from 'react'

// 1. 子组件 + Props 父传子
function Child({ name, onChildClick }) {

  return (
    <div style={{ border: '1px solid #ccc', padding: 10 }}>
      <p>子组件接收参数：{name}</p>
      <button onClick={() => onChildClick('我是子组件数据')}>
        子向父传值
      </button>
    </div>
  )
}

// 根组件
function App() {
  // 2. 局部状态：useState
  // 等价 Compose: mutableStateOf ｜ Vue: ref
  const [count, setCount] = useState(0)
  const [msg, setMsg] = useState('')
  const [list, setList] = useState(['React', 'Vue', 'Compose'])

  // 3. 副作用：useEffect
  // 等价 Compose: LaunchedEffect ｜ Vue: watch/watchEffect
  useEffect(() => {
    console.log('组件挂载 / 依赖变化执行')
    document.title = `计数：${count}`
  }, [count]) // 依赖数组

  // 事件函数
  const add = () => {
    setCount(prev => prev + 1)
  }

  // 接收子组件回调
  const handleChildEvent = (val) => {
    setMsg(val)
  }

  return (
    <div style={{ padding: 20 }}>
      <h3>React 核心概念演示</h3>

      {/* 4. 事件绑定 + 状态修改 */}
      <p>计数器：{count}</p>
      <button onClick={add}>点击+1</button>

      {/* 5. 条件渲染 */}
      {count > 5 && <p style={{color:'red'}}>计数超过5</p>}

      {/* 6. 列表渲染 key */}
      <ul>
        {list.map((item, idx) => (
          <li key={idx}>{item}</li>
        ))}
      </ul>

      <p>子组件发来：{msg}</p>

      {/* 7. 父传子 Props + 传递回调 */}
      <Child name="前端学习" onChildClick={handleChildEvent} />
    </div>
  )
}

export default App
```

## Hook

### 基本使用

#### useState
管理组件自身状态
```jsx
import { useState } from "react";

function Counter() {
    const [count, setCount] = useState(0);

    return (
        <button onClick={() => setCount(count + 1)}>
            count: {count}
        </button>
    );
}
```

#### useReducer
`useReducer` 接收一个 `reducer` 函数和一个初始状态 `initialState`，并返回当前 `state` 和一个 `dispatch` 函数。
```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

useReducer 在处理复杂、或者相互关联的多个状态时，比 useState 更加方便和清晰。实际使用例子：
```jsx
// ✅ 使用 useReducer 集中管理
const initialState = { username: '', age: 0, email: '' };

function userReducer(state, action) {
  switch (action.type) {
    case 'setUsername':
      return { ...state, username: action.payload };
    case 'setAge':
      return { ...state, age: action.payload };
    case 'reset':
      return initialState; // 可以一次性重置所有状态
    default:
      return state;
  }
}

function UserProfile() {
  const [state, dispatch] = useReducer(userReducer, initialState);
  
  // 所有状态的更新都通过 dispatch 触发
  return (
    <div>
      <input value={state.username} onChange={(e) => dispatch({type: 'setUsername', payload: e.target.value})} />
      <input value={state.age} onChange={(e) => dispatch({type: 'setAge', payload: Number(e.target.value)})} />
      {/* ... */}
    </div>
  );
}
```

#### useContext
如果有的数据需要跨多个组件逐级传递，会比较麻烦，此时可以考虑使用 `useContext` 跨多层级组件共享数据。

#### useRef
React 函数组件是一个函数，所以每次状态（useState）变化导致重新渲染时，整个组件函数都会被重新执行一遍，普通的变量就会重新创建。如果想在多次渲染之间保持同一个对象存在，并且修改它不会触发UI刷新，就可以使用 `useRef`。相当于一个不会刷新UI的 useState。

#### useMemo
`useMemo` 的核心作用就是**缓存昂贵的计算结果**，当组件中有复杂的计算逻辑（例如大数据量的过滤、排序、数学运算等），并且组件会因为其他无关状态的变化而频繁重渲染时，用它来避免重复计算就非常有效。

```jsx
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');
  const visibleTodos = useMemo(() => {
    // ✅ 除非 todos 或 filter 发生变化，否则不会重新执行
    return getFilteredTodos(todos, filter);
  }, [todos, filter]);
  // ...
}
```

> 在感觉性能有问题时再使用，毕竟它本身也会有管理依赖、存储缓存的性能成本

#### useEffect
`useEffect` 只会在依赖的数组中的值发生变化时，才重新执行副作用函数：

```jsx
// userId 变化，就会重新拉取数据
import { useEffect, useState } from "react";

function UserProfile({ userId }: { userId: string }) {
    const [user, setUser] = useState(null);

    useEffect(() => {
        fetch(`/api/users/${userId}`)
            .then(res => res.json())
            .then(data => setUser(data));
    }, [userId]);

    return <div>{user ? user.name : "Loading..."}</div>;
}
```

依赖数组是决定了副作用何时执行，主要有以下三种模式：
* 不传：每次组件渲染后都执行
* [] (空数组)：仅在组件首次挂载后执行一次
* 有依赖项：在首次挂载后，以及依赖项变化时执行

> `useEffect` 还可以返回一个清理函数，用于组件卸载前执行

> 从功能覆盖的角度看：React useEffect ≈ Android Compose LaunchedEffect + DisposableEffect + SideEffect

#### useCallback
`useCallback` 的作用就是缓存函数本身，`useCallback(fn, deps)` 本质上等价于 `useMemo(() => fn, deps)`，感觉是 React 为了更语义化设计的一个API。

首次渲染 React 会创建这个函数，后续渲染 React 会比较这次的依赖数组和上一次是否变化，变化才会重新创建函数并缓存。
```jsx
import { useCallback } from "react";

function Parent({ onSave }) {
    const handleClick = useCallback(() => {
        onSave();
    }, [onSave]);

    return <button onClick={handleClick}>Save</button>;
}
```

#### useEffectEvent
有这样一个聊天室的例子，连接后会显示提示，连接依赖 `roomId`，而这个提示会依赖 `theme`。
```jsx
function ChatRoom({ roomId, theme }) {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.on('connected', () => {
      showNotification('Connected!', theme);
    });
    connection.connect();
    return () => {
      connection.disconnect()
    };
  }, [roomId, theme]); // ✅ 声明所有依赖项
  // ...
}
```
⚠️ 这个例子会产生一个问题，只是修改 `theme`，也会导致重新连接。

> 在 useEffect 的回调函数中使用变量，如果这个变量既不在依赖数组里，又没有通过其他方式（如 Ref）同步，那么这个值永远是闭包产生那一刻的值。这就是 React（以及所有基于 JavaScript 闭包的框架）中著名的 “闭包陷阱” (Stale Closure)。

我们可以尝试使用 `useRef` 来处理：
```jsx
function ChatRoom({ roomId, theme }) {
  // 1. 创建一个 ref 来保存 theme
  const themeRef = useRef(theme);

  // 2. 使用一个 effect 来同步 theme 的最新值到 ref
  useEffect(() => {
    themeRef.current = theme;
  }, [theme]);

  // 3. 主 effect 只依赖 roomId，但通过 ref 访问最新的 theme
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.on('connected', () => {
      // ✅ 这里读取的是 ref.current，总能拿到最新值
      showNotification('Connected!', themeRef.current);
    });
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, [roomId]); // ✅ 只有 roomId 变化才会重连
}
```
虽然可以解决，但是稍嫌麻烦。`useEffectEvent` 是 React 19 引入的一个新 Hook，它的核心目的是将“非响应式”的事件逻辑从 useEffect 中剥离出来。解决了**如何在 useEffect 中使用最新的 State/Props，却又不想让这个 State/Props 成为 Effect 的依赖项**的问题

使用 `useEffectEvent` 的方式：
```jsx
function ChatRoom({ roomId, theme }) {
  // 使用 useEffectEvent，返回值就是一个和传入函数相同签名的函数（可以理解为一个代理）
  const onConnected = useEffectEvent(() => {
    showNotification('Connected!', theme);
  });

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.on('connected', () => {
      onConnected();
    });
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]); // ✅ 声明所有依赖项
  // ...
}
```

Effect Event 的使用限制：
* 只在 Effect 内部调用他们。
* 永远不要把他们传给其他的组件或者 Hook。


### 使用自定义 Hook 复用逻辑
Hook 不只是管理状态，更重要的是把业务逻辑从 UI 组件里拆出去。

```jsx
// 业务逻辑单独抽为一个函数，内部使用 hook
function useUserInfo(userId: string) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(false);

    useEffect(() => {
        setLoading(true);

        fetch(`/api/users/${userId}`)
            .then(res => res.json())
            .then(data => setUser(data))
            .finally(() => setLoading(false));
    }, [userId]);

    return { user, loading };
}

// UI 组件只关心展示：
function UserCard({ userId }) {
    const { user, loading } = useUserInfo(userId);

    if (loading) return <div>Loading...</div>;

    return <div>{user.name}</div>;
}
```

成熟项目里，Hook 往往会变成“业务基础设施”。例如：
* useAuth()：获取登录状态、用户信息、权限
* useRequest()：统一请求逻辑
* useDebounce()：防抖输入
* useImageEditor()：封装图片编辑状态
* useBridge()：和客户端 / Unity / Native 通信
* ...

## key
React 会把相同位置的相同类型组件视为同一实例，state 会被保留下来复用。所以 React 中也有 key 来明确指定。
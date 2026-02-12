Vue3笔记

## Vue App实例
Vue3 创建App实例API如下：
```js
function createApp(rootComponent: Component, rootProps?: object): App
```

使用时，可以直接内联根组件，也可以使用从别处导入的组件：
```js
// // 直接内联根组件
import { createApp } from 'vue'

const app = createApp({
  /* 根组件选项 */
})


// 使用从别处导入的组件
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
```

例如这个demo：
```js
import { createApp, ref, computed, onMounted } from 'vue'

// 使用组合式 API 的根组件
const RootComponent = {
  setup() {
    // 响应式数据
    const message = ref('Hello Vue 3!')
    const count = ref(0)
    
    // 计算属性
    const doubleCount = computed(() => count.value * 2)
    
    // 方法
    const increment = () => {
      count.value++
    }
    
    const reset = () => {
      count.value = 0
      message.value = 'Count reset!'
    }
    
    // 生命周期钩子
    onMounted(() => {
      console.log('Root component mounted')
    })
    
    // 暴露给模板
    return {
      message,
      count,
      doubleCount,
      increment,
      reset
    }
  },
  
  template: `
    <div>
      <h1>{{ message }}</h1>
      <p>Count: {{ count }}</p>
      <p>Double Count: {{ doubleCount }}</p>
      <button @click="increment">Increment</button>
      <button @click="reset">Reset</button>
    </div>
  `
}

// 创建并挂载应用
const app = createApp(RootComponent)
app.mount('#app')
```

## 模板语法
这部分直接查看官方文档即可

## 组件
组件是可复用的 Vue 实例

### 创建组件
**组合式API组件**：
```js
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)
    return { count }
  },
  template: `
    <button @click="count++">
      You clicked me {{ count }} times.
    </button>`
  // 也可以针对一个 DOM 内联模板：
  // template: '#my-template-element'
}
```
`setup`函数是 Vue 3 新增的，它是 Composition API（组合式 API）的核心入口函数。setup 函数是在组件创建之前执行的一个函数，用于替代 Vue 2 中的选项式 API（如 data、methods、computed 等）。

如果在单文件组件中使用组合式API创建组件：
```html
<script>
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)
    
    // 返回要在模板中使用的响应式数据
    return { count }
  }
}
</script>

<template>
  <button @click="count++">You clicked me {{ count }} times.</button>
</template>
```

在vue文件中的单文件组件，组合式API还可以使用`<script setup>`语法糖：
```html
<script setup>
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <button @click="count++">You clicked me {{ count }} times.</button>
</template>
```
使用`<script setup>`，相比于普通的`<script>`语法，它具有更多优势：
* 更少的样板内容，更简洁的代码。
* 能够使用纯 TypeScript 声明 props 和自定义事件。
* 更好的运行时性能 (其模板会被编译成同一作用域内的渲染函数，避免了渲染上下文代理对象)。
* 更好的 IDE 类型推导性能 (减少了语言服务器从代码中抽取类型的工作)。

`<script setup>`中的的代码会被编译成组件`setup()`函数的内容。这意味着与普通的`<script>`只在组件被首次引入的时候执行一次不同，`<script setup>`中的代码会在每次组件实例被创建的时候执行。

任何在`<script setup>`声明的顶层的绑定 (包括变量，函数声明，以及 import 导入的内容，比如组件) 都能在模板中直接使用

---------------------

**选项式API组件**：
```js
export default {
  data() {
    return {
      count: 0
    }
  },
  template: `
    <button @click="count++">
      You clicked me {{ count }} times.
    </button>`
}
```

如果是单文件组件，使用选项式API：
```html
<script>
export default {
  data() {
    return {
      count: 0
    }
  }
}
</script>

<template>
  <button @click="count++">You clicked me {{ count }} times.</button>
</template>
```

**之后的笔记都以组合式API为例**

### 注册组件
注册组件可以分为全局注册和局部注册，全局注册的组件可以在任意Vue实例管理的视图中使用，局部注册的组件只能在当前Vue实例管理的视图中使用

使用 Vue 应用实例的`component()`方法**全局注册**：
```js
import { createApp } from 'vue'

const app = createApp({})

app.component(
  // 注册的名字
  'MyComponent',
  // 组件的实现对象
  {
    /* ... */
  }
)
```

**局部注册**

使用 `<script setup>` 的单文件组件中，导入的组件可以直接在模板中使用，无需注册。

否则需要显式注册：
```
import ComponentA from './ComponentA.js'

export default {
  components: {
    ComponentA
  },
  setup() {
    // ...
  }
}
```

### 使用组件
然后可以在html中使用组件，下面就会显示三个组件的模板内容：
```
<div id="app">
    <MyComponent></MyComponent>
    <MyComponent></MyComponent>
    <MyComponent></MyComponent>
</div>
```

### props
`props`是组件间传递数据的重要机制，让父组件（使用这个组件的地方）给子组件传递数据。Vue3 的组合式 API 中，我们用`defineProps`宏来声明`props`（不用导入，直接用）。

1. 首先在子组件中声明`props`：
```html
<template>
  <div class="cake">
    <!-- 在模板中使用接收的到props -->
    <p>口味：{{ flavor }}</p>
    <p>尺寸：{{ size }}寸</p>
  </div>
</template>

<script setup>
// 核心：用 defineProps 声明需要接收的 props
// 这里声明了两个 props：flavor（字符串）、size（数字）
const props = defineProps({
  flavor: String, // 口味是字符串类型
  size: Number    // 尺寸是数字类型
})

// 后续在脚本中使用 props 时，直接用 props.flavor 或 props.size
console.log('当前口味：', props.flavor)
</script>
```

`defineProps`也可以使用数组数据：
```html
<script setup>
const props = defineProps(['foo'])

console.log(props.foo)
</script>
```

没有使用 `<script setup>` 的组件中，可以使用 `props` 选项来声明：
```js
export default {
  props: ['foo'],
  setup(props) {
    // setup() 接收 props 作为第一个参数
    console.log(props.foo)
  }
}
```

2. 父组件给子组件传数据
```html
<template>
  <!-- 使用子组件 <Cake>，并传递 props -->
  <Cake 
    flavor="巧克力"  <!-- 传递字符串（直接写值，不用:） -->
    :size="8"        <!-- 传递数字（用:绑定，因为8是数字类型） -->
  />

  <!-- 也可以传递父组件自己的数据（变量） -->
  <Cake 
    :flavor="myFlavor"  <!-- 传递父组件的变量 myFlavor -->
    :size="mySize"      <!-- 传递父组件的变量 mySize -->
  />
</template>

<script setup>
import Cake from './Cake.vue' // 导入子组件

// 父组件自己的数据（用 ref 定义响应式变量）
const myFlavor = '草莓'
const mySize = 6
</script>
```
> 静态值则直接在标签属性上写，如果是动态绑定的 props 则要使用`v-bind`或缩写`:`来进行。如果是表达式而不是字符串，也要使用动态绑定。


**`props`是单向数据流，props 因父组件的更新而变化，自然地将新的状态向下流往子组件，而不会逆向传递，这避免了子组件意外修改父组件的状态的情况**，子组件只能获取props的数据，再额外存储到单独的变量中才能修改。

### 事件
子组件想要通知父组件，需要使用事件。

#### 发射事件（子组件触发事件）
1. 模板内使用`$emit`直接触发：
```html
<!-- 子组件模板 -->
<button @click="$emit('submit')">提交</button>
```

2. 脚本代码触发：
先通过 `defineEmits` 声明事件，再用返回的 emit 函数触发（`<script setup>` 推荐）。
```js
<!-- 子组件 <script setup> -->
const emit = defineEmits(['submit', 'increase-by']) // 声明要触发的事件

function handleClick() {
  emit('submit') // 触发无参事件
  emit('increase-by', 1) // 触发带参事件（传 1 作为参数）
}
```

3. 非 `<script setup>` 语法：通过 `emits` 选项声明，用 `this.$emit` 触发
```js
export default {
  emits: ['inFocus', 'submit'],
  setup(props, ctx) {
    ctx.emit('submit')
  }
}
```

#### 接收事件（父组件监听事件）
父组件通过 `v-on`（缩写 `@`）监听子组件触发的事件，可绑定内联函数或组件方法。

1. **绑定内联箭头函数**：直接在模板中处理事件，可快速接收参数。
```html
<!-- 父组件模板 -->
<ChildComponent @submit="() => console.log('提交触发')" />
<ChildComponent @increase-by="(n) => count += n" /> <!-- 接收参数 n 并更新 count -->
```

2. **绑定组件方法**：将事件处理逻辑抽离到脚本中，参数自动传入方法。
```html
<!-- 父组件模板 -->
<ChildComponent @submit="handleSubmit" @increase-by="handleIncrease" />

<script setup>
import { ref } from 'vue'
const count = ref(0)

function handleSubmit() {
  console.log('处理提交逻辑')
}

function handleIncrease(n) { // n 是子组件传递的参数
  count.value += n
}
```

组件的事件监听器也支持 `.once` 修饰符：
```html
<ChildComponent @submit.once="handleSubmit" /> <!-- 仅监听一次 submit 事件 -->
```

#### 事件声明与校验


> 事件的例子都使用的是组合式API

### v-model
Vue 的 `v-model` 是 组件间双向数据绑定的语法糖，核心目的是简化 “父组件传值 + 子组件反馈更新” 的代码流程。

`v-model` = `v-bind:value`（数据从 Vue 流向 DOM / 组件） + `v-on:input`（数据从 DOM / 组件流向 Vue），双向联动。


子组件：
```html
<template>
  <!-- 绑定父组件传递的 modelValue，触发 update 事件 -->
  <input 
    :value="modelValue" 
    @input="$emit('update:modelValue', $event.target.value)"
  >
</template>
<script setup>
// 声明接收的 props
defineProps(['modelValue'])
// 声明触发的事件
defineEmits(['update:modelValue'])
</script>
```

父组件中使用子组件
```html
<template>
  <!-- 直接用 v-model 绑定，等价于 :modelValue="msg" + @update:modelValue="msg = $event" -->
  <CustomInput v-model="msg" />
</template>
<script setup>
import { ref } from 'vue'
import CustomInput from './CustomInput.vue'
const msg = ref('')
</script>
```

### Attributes 透传
`v-on`、`class`、`style` 和 `id` 在当一个组件以单个元素为根作渲染时，透传的 attribute 会自动被添加到根元素上。

假如有一个组件`MyButton`，它的模板是：
```html
<!-- <MyButton> 的模板 -->
<button>Click Me</button>
```

一个父组件使用了这个组件，并且传入了 `class：`
```html
<MyButton class="large" />
```

那么 `class` 被视作透传 `attribute`，自动透传到了 `<MyButton>` 的根元素上。所以最后渲染结果是：
```html
<button class="large">Click Me</button>
```

在组件选项中设置 `inheritAttrs: false` 可以禁用透传，或者在 `<script setup>` 中使用 `defineOptions`。

> 还有一些细节，用的时候再完善


### 模板抽离复用
模板的字符串内容除了直接写在组件内部，还可以使用写在单独的标签中：
```
<template id="mytemplate">
    <div>
        <h2>{{msg}}</h2>
    </div>
</template>
```
然后在js中引用：
```
const component = {
    props: {
        msg: String
    },
    template: '#mytemplate'
}
```

### 通过对象访问组件
1. 访问根组件
this.$root

2. 访问父组件
this.$parent

3. 访问子组件或子元素
* this.$refs 需要和标签上的 ref 属性配合
* this.$children

### 插槽 slot
插槽为组件内部预留了一个位置，这个位置可以让组件的使用者自定义要显示的内容：
```html
<template id="my-template">
    <div>
        <h1>下面是插槽，将被替换为组件标签中间的内容</h1>
        <slot></slot>
    </div>
</template>
```
这里省略组件的注册流程，直接看使用组件的方式:
```html
<div id="app">
    <my-component><button>显示在插槽位置的按钮</button></my-component>
</div>
```
组件标签中间的buttn标签将显示在插槽位置。

#### 插槽默认内容
```html
<button type="submit">
  <slot>
    Submit <!-- 默认内容 -->
  </slot>
</button>
```

#### 具名插槽
如果有多个插槽，那么需要使用具名插槽：
```html
<template id="my-template">
    <div>
        <h1>下面两个具名插槽</h1>
        <slot name="first"></slot>
        <slot name="second"></slot>
    </div>
</template>
```
使用的方式：
```html
<div id="app">
    <my-component>
        <div v-slot="second">
            <h1>第二个</h1>
            <button >button2</button>
        </div>
        <button v-slot="first">button1</button>
    </my-component>
</div>
```
将根据slot属性的值放到对应的插槽位置

如果只是要插入多个标签到插槽，而不想要多余的div标签包装，可以使用`template`标签，显示时`template`标签不会存在
```
<div id="app">
    <my-component>
        <template v-slot="second">
            <h1>第二个</h1>
            <button >button2</button>
        </template>
        <button v-slot="first">button1</button>
    </my-component>
</div>
```

#### 条件插槽
有时你需要根据内容是否被传入了插槽来渲染某些内容，可以结合使用 `$slots` 属性与 `v-if` 来实现。

#### 动态插槽名
```html
<base-layout>
  <template v-slot:[dynamicSlotName]>
    ...
  </template>

  <!-- 缩写为 -->
  <template #[dynamicSlotName]>
    ...
  </template>
</base-layout>
```

#### 编译作用域
使用组件时，组件标签的属性，在传给插槽的内容中是访问不到的：
```
<div id="app">
    <my-component data="123">
      data is {{ data }}
    </my-component>
</div>
```
上面这个例子，插槽的内容是访问不到data的，因为data数据的作用域是父组件（也就是管理id为 app 的这个Vue实例），data是父组件传给`my-component`组件的。也就是说data是属于父级Vue实例，这个data的值将会是管理id为app的Vue实例内部的数据，插槽的内容只能访问到`my-component`对应的Vue实例。

> 官方总结：**父级模板里的所有内容都是在父级作用域中编译的；子模板里的所有内容都是在子作用域中编译的。**

#### 作用域插槽
如果我们想让这个data访问到my-component组件对应的Vue实例的数据，需要使用 `v-bind` 和 `v-slot`。

首先，定义一个有插槽的模板，这个插槽绑定了数据hfy：
```html
<template id="my-template">
    <div>
        <slot v-bind:user="hfy"></slot>
    </div>
</template>
```
数据hfy在组件对应的Vue实例的data中声明（省略注册组件）：
```js
const component = {
    data: function () {
        return {
            hfy: "hfy"
        }
    },
    template: '#my-template'
}
```
然后使用组件：
```html
<div id="app">
    <my-component>
        <template v-slot="slotProps">{{slotProps.user}}</template>
    </my-component>
</div>
```
此时传入的插槽内容，加上了`v-slot="slotProps"`，表示`slotProps`就是该匿名插槽的prop，现在通过`slotProps`就可以访问到插槽绑定的user属性对应的数据hfy。

如果插槽有名称，那么需要使用`v-slot:name="slotProps"`，`name`是插槽的名称

### 依赖注入
如果逐级使用props传递数据比较麻烦，vue也支持依赖注入。

1. 使用`provide`注入依赖：
```html
<script setup>
import { provide } from 'vue'

provide(/* 注入名 */ 'message', /* 值 */ 'hello!')
</script>
```
如果不使用 `<script setup>`，则这样使用：
```js
import { provide } from 'vue'

export default {
  setup() {
    provide(/* 注入名 */ 'message', /* 值 */ 'hello!')
  }
}
```

2. 使用`inject`获取依赖：
```html
<script setup>
import { inject } from 'vue'

const message = inject('message')
</script>
```

如果提供的值是一个 `ref`，注入进来的会是该 `ref` 对象，而不会自动解包为其内部的值。这使得注入方组件能够通过 `ref` 对象保持了和供给方的响应性链接。

### 异步组件
`defineAsyncComponent`函数用于定义异步组件，它可以在组件需要时才加载，而不是在应用初始化时就加载。

### 渲染函数
使用render函数，则可以不用使用template，但要配合vue-template-compiler

如果模板已经被转换，那么vue库可以使用vue-runtime-only，不包含模板的解析功能，减少库的大小。否则要使用vue-runtime-compiler

### 内置特殊元素
#### <component>
用于渲染动态组件或元素的“元组件”，要渲染的实际组件由 `is` prop 决定
```html
<script setup>
import Foo from './Foo.vue'
import Bar from './Bar.vue'
</script>

<template>
  <component :is="Math.random() > 0.5 ? Foo : Bar" />
</template>
```

## 响应式系统
`ref`函数

```js
import { ref } from 'vue'

const count = ref(0)
```

### 侦听器
`watch`函数


watchEffect

## 使用Vite开发

前面的Vue基础使用的例子，都是直接引用Vue的js文件来使用，而实际的Vue项目开发，一般会使用Vue脚手架工程，用它可以创建Vue单文件组件风格的项目结构，单文件组件的文件后缀为.vue

安装node，可以选择通过安装nvm管理node版本，参考 https://nodejs.org/zh-cn/download

使用npm创建vite的vue工程
```shell
npm create vite@latest
```
Vite 官方推荐直接使用 `npm create vite@latest` 这种方式，因为它：
* 总是使用最新版本
* 不需要预先安装任何东西
* 保持系统干净（不会留下全局包）

如果手动安装，则需要自己执行vite：
```shell
npm install -D vite
npx vite
```

## Vue router
一个单页面应用，需要使用到前端路由（改变url不会发起请求）。

前端路由的原生方式：
1. hash模式：修改location.hash，比如location.hash='/hello'，会在当前url后加上#/hello
2. history模式：pushState、replaceState、go等方法

这里展示在一个使用Vite创建的项目结构中使用Vue router的基本用法，更丰富的用法参考[官方文档](https://router.vuejs.org/zh/)

----------

1. 安装依赖
```shell
npm install vue-router@4
```

2. 安装到Vue实例
在src目录内创建一个router目录（可自行决定），里面放一个index.js，内容如下：
```js
import Vue from 'vue'
import VueRouter from 'vue-router'

Vue.use(VueRouter)
```

3. 创建路由组件
在src/components里面创建Home.vue和About.vue两个组件，然后在src/router/index.js中添加代码：
```js
const router = new VueRouter({
    mode: 'history', //使用history模式，避免hash模式会在路径上显示#
    routes: [
        {
            path: '',
            redirect:'/home'
        },
        {
            path: '/home',
            component: Home
        },
        {
            path: '/about',
            component: About
        }
    ],
})

export default router
```
创建VueRouter对象，配置路由的路径映射，默认重定向到Home组件

3. 使用vue router
在src/main.js的Vue实例中，配置上一步定义的router：
```js
import Vue from 'vue'
import App from './App.vue'
import router from "./router";

Vue.config.productionTip = false

new Vue({
    render: h => h(App),
    router
}).$mount('#app')
```
导入router，默认会找 ./router 里面的名称为index.js的文件，把router传给Vue实例

然后在 src/App.vue 的template中，使用 router-link 和 router-view 标签：
```html
<template>
    <div id="app">
        <router-link to="/home">首页</router-link>
        <router-link to="/about">关于</router-link>
        <router-view></router-view>
    </div>
</template>
```
router-link用于点击跳转路径，router-view用于显示该路径映射的组件

----------

### router-link
router-link标签还可以使用tag、replace等属性：
* tag：可以修改router-link标签显示的元素，比如使用button
* replace：使用replace属性，页面切换不会有历史记录

### 通过代码跳转
使用 this.$router 可以使用代码跳转路由，而不用

### 动态路由
可以把例如user/foo和user/bar都映射到相同的路由，这需要使用到动态路径参数：
```
{
    // 动态路径参数 以冒号开头
    path: '/user/:id',
    component: User
}
```
然后在User.vue中使用 $route.params 来获取路径参数：
```
export default {
    name: "User",
    computed: {
        id() {
            return this.$route.params.id
        }
    }
}
```

### 路由懒加载
路由懒加载可以在路由被访问时才加载对应组件，不同的路由对应的组件分成不同的js文件（由webpack完成），提高加载速度

```
{
    path: '/home',
    component: ()=>import('../components/Home')
}
```

### 嵌套路由
```
{
    path: '/home',
    component: ()=>import('../components/Home')
    children:[
        {
            path: 'news',
            component: ()=>import('../components/HomeNews')
        },
        {
            path: 'message',
            component: ()=>import('../components/HomeMessage')
        }
    ]
}
```
然后在Home组件中使用 router-link（这里要写全路径） 和router-view

### 传递参数
除了前面通过动态路径传参，还可以使用query的方式：

首先路由配置仍使用普通的方式
```
{
    path: '/about',
    component: About
}
```
然后router-link配置如下：
```
<router-link v-bind:to="{path:'/about',query:{name:'jxy',age:23}}">关于</router-link>
```
这里使用v-bind，则to属性的内容为对象，加上了query对象。然后在About组件中可以获取到查询参数：
```
<template>
    <div>
        <h1>关于</h1>
        <h2>{{$route.query.name}}</h2>
        <h2>{{$route.query.age}}</h2>
    </div>
</template>
```

> 动态路径参数和查询参数的方式，都有对应的代码跳转方式

$router表示整个路由管理对象，它是放在Vue实例的原型上，所以任何Vue实例，或者说组件，都能拿到该对象，而$route表示当前路由

### 导航守卫
导航守卫可以监听路由的切换，这部分较简单，阅读文档即可

### keep-alive
默认情况，路由切换会销毁前一个路由组件实例，创建新的组件实例，切回的时候也是创建组件实例。而使用keep-alive标签包裹router-view标签，可以避免路由组件实例被销毁：
```
<keep-alive>
    <router-view></router-view>
</keep-alive>
```
keep-alive标签还可以使用exclude来排除缓存指定的组件实例，include指定包含缓存的组件实例，exclude和include都支持正则；max指定最大的缓存数量

## Vuex
Vuex是一个响应式的集中式状态管理，多个组件间共享数据。

### 安装
```
npm install vuex --save
```
### 使用
首先安装到Vue实例中，这里为了分割模块，方便管理，我们创建src/store目录，在其中创建index.js：
```
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

const store = new Vuex.Store({
    state: {},
    getters: {},
    mutations: {},
    actions: {},
    modules: {}
})

export default store
```
这里，state等属性暂时先设置为空对象

然后在src/main.js中的Vue实例中，注册store：
```
import Vue from 'vue'
import App from './App.vue'
import store from "./store";

Vue.config.productionTip = false

new Vue({
    store,
    render: h => h(App),
}).$mount('#app')
```

到此已经搭建好使用vuex的基本结构，然后我们在state中添加属性：
```
const store = new Vuex.Store({
    state: {
        count: 0
    },
    getters: {},
    mutations: {},
    actions: {},
    modules: {}
})
```

然后在App组件中修改count：
```
<template>
    <div id="app">
        <h1>{{$store.state.count}}</h1>
        <button v-on:click="$store.state.count++">+</button>
        <button v-on:click="$store.state.count--">-</button>
        <HelloWorld msg="Welcome to Your Vue.js App"/>
    </div>
</template>
```
点击按钮修改count属性，然后在子组件中的视图也会跟着改动：
```
<template>
    <div class="hello">
        <h1>{{$store.state.count}}</h1>
    </div>
</template>
```

* getters：用于定义state内属性的计算属性
* mutations：实际直接修改state内的属性是可以的，但是为了支持Devtools调试工具跟踪数据，所以建议使用mutations
* actions：因为Devtools只能跟踪同步修改，所以建议在actions中调用mutations相关方法，以支持异步操作
* modules：可以把 store 分割成模块

这几点官方文档较详细，不赘述


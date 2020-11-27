## Vue实例
```
//HTML

<div id="app">
  {{ message }}
</div>
```

```
//JavaScript

var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```
上面是官方的示例，通过 Vue 函数创建一个新的 Vue 实例，创建 Vue 实例的时候，可以传入一个选项对象。

### 选项对象
选项对象中的属性：
* `el`：可以是String或者HTMLELement，用于和DOM中的元素关联
* `data`：可以是Object或者Function（组件的Data必须是Function），data对象中的所有的 property 会加入到 Vue 的响应式系统中，修改 property 的值，视图就会随之更新
* `methods`：定义Vue的方法，可以用在模版语法中

## 模板语法
这部分直接查看官方文档即可

## 组件
组件是可复用的 Vue 实例，通过 Vue 构造函数创建的 Vue 实例可以认为是根组件

### 组件的基本使用
1. 定义组件
```
var component = {
    template: `
        <div>
        <h2>我的组件</h2>
        </div>
        `
}
```
可以通过一个普通的 JavaScript 对象定义一个组件，`template` 属性表示组件显示的内容

也可以通过下面这种方式来定义一个组件，但官方文档已经看不到这个方式了，可能不推荐
```
var component = Vue.extend({
    template: `
    <div>
    <h2>我的组件</h2>
    </div>
    `
})
```

2. 注册组件
注册组件可以分为全局注册和局部注册，全局注册的组件可以在任意Vue实例管理的视图中使用，局部注册的组件只能在当前Vue实例管理的视图中使用

全局注册：
```
Vue.component('my-component', component)
```

局部注册
```
var app = new Vue({
    el: '#app',
    components: {
        'my-component': component
    }
})
```

定义组件和注册组件也可以一步完成，以全局注册为例（局部注册同理）：
```
Vue.component('my-component', {
    template: `
        <div>
        <h2>我的组件</h2>
        </div>
        `
})
```

3. 使用组件
然后可以在html中使用组件，下面就会显示三个组件的模板内容：
```
<div id="app">
    <my-component></my-component>
    <my-component></my-component>
    <my-component></my-component>
</div>
```

> 组件也是一个Vue实例，所以也可以使用data属性，组件的data必须是函数，因为每个组件实例需要维护一份被返回对象的独立的拷贝，否则数据会在多个组件实例间共享

### 父子组件
在组件中也可以注册另一个组件，这就形成了父子组件，父组件中注册的子组件只能在父组件的模板内部使用，如果要在其他地方使用，那么需要另外注册。
```
// 定义一个子组件
const subComponent = {
    template: `
    <div>
    <h2>子组件</h2>
    </div>
    `
}

//定义一个父组件，在父组件中注册了子组件，并且在模板中使用了子组件
const parentComponent = {
    template: `
    <div>
    <h2>父组件</h2>
    <sub-component></sub-component>
    </div>
    `,
    components: {
        'sub-component': subComponent
    }
}

// 注册父组件到Vue实例
var app = new Vue({
    el: '#app',
    data: {
        message: 'Hello Vue!'
    },
    components: {
        'parent': parentComponent
    }
})
```
然后在HTML中使用：
```
<div id="app">
    <parent></parent>
    <parent></parent>
    <parent></parent>
</div>
```

### 父子组件的通信
使用props属性，可以是数组或者对象（对象的方式，可以做更多的设置，比如限制类型），这里父组件直接使用根组件
1. 父组件传给子组件
```
const component = {
    props: {
        msg: String
    },
    template: `
    <div>
    <h2>{{msg}}</h2>
    </div>
    `
}

var app = new Vue({
    el: '#app',
    data: {
        message: 'Hello Vue!'
    },
    components: {
        'my-component': component
    }
})
```
然后在HTML中：
```
<div id="app">
    <my-component v-bind:msg="message"></my-component>
</div>
```
component中定义了`props`属性，内部有一个msg属性，然后在HTML标签上使用v-bind:msg="message"，就会把msg的值绑定到父组件的message上，也就会显示`Hello Vue!`

props中使用对象声明，还可以用更复杂限制：
```
props: {
    prop1: String,
    prop2: {
        type: String,
        default: ``,
        required: true
    }
},
```

2. 子组件传给父组件
子组件想要通知父组件，需要使用自定义事件：
```
const component = {
    props: {
        msg: String
    },
    methods: {
        send() {
            this.$emit('myevent', "hello")
        }
    },
    template: `
    <div>
    <button v-on:click="send">mybutton</button>
    <h2>{{msg}}</h2>
    </div>
    `
}

var app = new Vue({
    el: '#app',
    data: {
        message: 'Hello Vue!'
    },
    methods: {
        eventLog(data) {
            console.log(data)
        }
    },
    components: {
        'my-component': component
    }
})
```
子组件中的模板内的按钮，点击时调用send方法，send方法将调用 `$emit` 方法发射名为 `myevent` 的自定义事件，并传递一个字符串 hello

然后HTML如下：
```
<div id="app">
    <my-component v-bind:msg="message" v-on:myevent="eventLog"></my-component>
</div>
```
使用子组件时，用 `v-on` 监听 `myevent` 事件，执行 eventLog 方法，也就会把字符串 hello 打印出来

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
```
<template id="my-template">
    <div>
        <h1>下面是插槽，将被替换为组件标签中间的内容</h1>
        <slot></slot>
    </div>
</template>
```
这里省略组件的注册流程，直接看使用组件的方式:
```
<div id="app">
    <my-component><button>显示在插槽位置的按钮</button></my-component>
</div>
```
组件标签中间的buttn标签将显示在插槽位置。

#### 具名插槽
如果有多个插槽，那么需要使用具名插槽：
```
<template id="my-template">
    <div>
        <h1>下面两个具名插槽</h1>
        <slot name="first"></slot>
        <slot name="second"></slot>
    </div>
</template>
```
使用的方式：
```
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
```
<template id="my-template">
    <div>
        <slot v-bind:user="hfy"></slot>
    </div>
</template>
```
数据hfy在组件对应的Vue实例的data中声明（省略注册组件）：
```
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
```
<div id="app">
    <my-component>
        <template v-slot="slotProps">{{slotProps.user}}</template>
    </my-component>
</div>
```
此时传入的插槽内容，加上了`v-slot="slotProps"`，表示`slotProps`就是该匿名插槽的prop，现在通过`slotProps`就可以访问到插槽绑定的user属性对应的数据hfy。

如果插槽有名称，那么需要使用`v-slot:name="slotProps"`，`name`是插槽的名称

### 渲染函数
使用render函数，则可以不用使用template，但要配合vue-template-compiler

如果模板已经被转哈，那么vue库可以使用vue-runtime-only，不包含模板的解析功能，减少库的大小。否则要使用vue-runtime-compiler



## 使用Vue CLI开发
https://cli.vuejs.org/zh/

前面的Vue基础使用的例子，都是直接引用Vue的js文件来使用，而实际的Vue项目开发，一般会使用Vue CLI，用它可以创建Vue单文件组件风格的项目结构，单文件组件的文件后缀为.vue

安装
```
npm install -g @vue/cli
```

然后创建项目：
```
vue create name
```

## Vue router
一个单页面应用，需要使用到前端路由（改变url不会发起请求）。

前端路由的原生方式：
1. hash模式：修改location.hash，比如location.hash='/hello'，会在当前url后加上#/hello
2. history模式：pushState、replaceState、go等方法

这里展示在一个使用Vue CLI创建的项目结构中使用Vue router的基本用法，更丰富的用法参考[官方文档](https://router.vuejs.org/zh/)

----------

1. 安装依赖
```
npm install --save vue-router
```

2. 安装到Vue实例
在src目录内创建一个router目录（可自行决定），里面放一个index.js，内容如下：
```
import Vue from 'vue'
import VueRouter from 'vue-router'

Vue.use(VueRouter)
```

3. 创建路由组件
在src/components里面创建Home.vue和About.vue两个组件，然后在src/router/index.js中添加代码：
```
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
```
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
```
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


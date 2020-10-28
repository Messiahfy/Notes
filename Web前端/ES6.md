读 https://es6.ruanyifeng.com/  笔记
## let 和 const
* let声明的变量存在块级作用域
* let不存在变量提升，即必须在声明后使用
* let不允许重复声明
* const声明就必须立即初始化，其他和let一致
* let和const声明的全局变量，不属于顶层对象的属性

## 变量的解构声明
### 数组的解构声明
```
let [a, b, c] = [1, 2, 3];
a // 1 
b // 2
c // 3

let [ , , third] = ["foo", "bar", "baz"];
third // "baz"

let [x, , y] = [1, 2, 3];
x // 1
y // 3

let [head, ...tail] = [1, 2, 3, 4];
head // 1
tail // [2, 3, 4]
```

对于 Set 结构，也可以使用数组的解构赋值。
```
let [x, y, z] = new Set(['a', 'b', 'c']);
x // "a"
```
事实上，只要某种数据结构具有 Iterator 接口，都可以采用数组形式的解构赋值。

解构赋值允许指定默认值。
```
let [x, y = 'b'] = ['a']; // x='a', y='b'
```
### 对象的解构声明
解构不仅可以用于数组，还可以用于对象。
```
let { foo, bar } = { foo: 'aaa', bar: 'bbb' };
foo // "aaa"
bar // "bbb"
```
象的解构与数组有一个重要的不同。数组的元素是按次序排列的，变量的取值由它的位置决定；而对象的属性没有次序，变量必须与属性同名，才能取到正确的值。

对象的解构赋值也允许指定默认值

### 字符串、数值、布尔值的解构赋值
字符串也可以解构赋值。这是因为此时，字符串被转换成了一个类似数组的对象。
```
const [a, b, c, d, e] = 'hello';
a // "h"
b // "e"
c // "l"
d // "l"
e // "o"
```
数值和布尔值的解构赋值省略

解构赋值的规则是，只要等号右边的值不是对象或数组，就先将其转为对象

### 函数参数的解构赋值
js的函数参数没有类型声明，所以参数写数组，就是用来解构赋值。
```
function add([x, y]){
  return x + y;
}

add([1, 2]); // 3
```
上面代码中，函数add的参数表面上是一个数组，但在传入参数的那一刻，数组参数就被解构成变量x和y。对于函数内部的代码来说，它们能感受到的参数就是x和y。

也可以使用对象解构赋值。

## 函数
* 支持默认参数
* 支持可变长参数

### 箭头函数
ES6 允许使用“箭头”（=>）定义函数。实质就是lambda表达式。
```
var f = v => v;

// 等同于
var f = function (v) {
  return v;
};
```
如果箭头函数不需要参数或需要多个参数，就使用一个圆括号代表参数部分。
```
var f = () => 5;
// 等同于
var f = function () { return 5 };

var sum = (num1, num2) => num1 + num2;
// 等同于
var sum = function(num1, num2) {
  return num1 + num2;
};
```

如果箭头函数的代码块部分多于一条语句，就要使用大括号将它们括起来，并且使用return语句返回。
```
var sum = (num1, num2) => { return num1 + num2; }
```
由于大括号被解释为代码块，所以如果箭头函数直接返回一个对象，必须在对象外面加上括号，否则会报错。
```
// 报错
let getTempItem = id => { id: id, name: "Temp" };

// 不报错
let getTempItem = id => ({ id: id, name: "Temp" })
```

**简化回调函数**：
```
// 正常函数写法
[1,2,3].map(function (x) {
  return x * x;
});

// 箭头函数写法
[1,2,3].map(x => x * x);
```

#### 箭头函数的注意点
1. 函数体内的this对象，就是定义时所在的对象，而不是使用时所在的对象。
2. 不可以当作构造函数
3. 不可以使用arguments对象，该对象在函数体内不存在
4. 不可以使用yield命令，因此箭头函数不能用作 Generator 函数

第一条尤为重要：
```
function foo() {
  setTimeout(() => {
    console.log('id:', this.id);
  }, 100);
}

var id = 21;

foo.call({ id: 42 });
// id: 42
```
调用foo.call时，才定义了箭头函数，此时所在对象为{ id: 42 }，所以打印的id是42。如果是普通函数，则this将指向全局对象window。

> this指向的固定化，并不是因为箭头函数内部有绑定this的机制，实际原因是箭头函数根本没有自己的this，导致内部的this就是外层代码块的this。

## 对象的扩展
### 属性的简洁表示法
ES6 允许在大括号里面，直接写入变量和函数，作为对象的属性和方法。这样的书写更加简洁。
```
const foo = 'bar';
const baz = {foo};
baz // {foo: "bar"}

// 等同于
const baz = {foo: foo};
```

除了属性简写，方法也可以简写。
```
const o = {
  method() {
    return "Hello!";
  }
};

// 等同于

const o = {
  method: function() {
    return "Hello!";
  }
};
```

### 属性名表达式
JavaScript 定义对象的属性，有两种方法。
```
// 方法一
obj.foo = true;

// 方法二
obj['a' + 'bc'] = 123;
```
如果使用字面量方式定义对象（使用大括号），在 ES5 中只能使用方法一。ES6 允许字面量定义对象时，用方法二（表达式）作为对象的属性名
```
let propKey = 'foo';

let obj = {
  [propKey]: true,
  ['a' + 'bc']: 123
};
```

### super关键字
ES6 新增了一个关键字super，指向当前对象的原型对象：
```
const proto = {
  foo: 'hello'
};

const obj = {
  foo: 'world',
  find() {
    return super.foo;
  }
};

Object.setPrototypeOf(obj, proto);
obj.find() // "hello"
```

### ?.和??操作符
?.运算符，直接在链式调用的时候判断，左侧的对象是否为null或undefined。如果是的，就不再往下运算，而是返回undefined。
```
const firstName = message?.body?.user?.firstName || 'default';
```

??只有运算符左侧的值为null或undefined时，才会返回右侧的值：
```
const headerText = response.settings.headerText ?? 'Hello, world!';
```

## Symbol
ES5 的对象属性名都是字符串，这容易造成属性名的冲突。比如，你使用了一个他人提供的对象，但又想为这个对象添加新的方法（mixin 模式），新方法的名字就有可能与现有方法产生冲突。所以ES6引入了Symbol类型，Symbol类型的值都是独一无二的。

对象的属性名现在可以有两种类型，一种是原来就有的字符串，另一种就是新增的 Symbol 类型。
```
let mySymbol = Symbol();

// 第一种写法
let a = {};
a[mySymbol] = 'Hello!';

// 第二种写法
let a = {
  [mySymbol]: 'Hello!'
};

// 第三种写法
let a = {};
Object.defineProperty(a, mySymbol, { value: 'Hello!' });

// 以上写法都得到同样结果
a[mySymbol] // "Hello!"
```
Symbol 值作为对象属性名时，不能用点运算符

## Proxy 动态代理
在 Proxy 代理的情况下，目标对象内部的this关键字会指向 Proxy 代理。
```
const target = {
  m: function () {
    console.log(this === proxy);
  }
};
const handler = {};

const proxy = new Proxy(target, handler);

target.m() // false
proxy.m()  // true
```
上面代码中，一旦proxy代理target，target.m()内部的this就是指向proxy，而不是target。

## Reflect 反射
Reflect对象的设计目的有这样几个：
1. 将Object对象的一些明显属于语言内部的方法（比如Object.defineProperty），放到Reflect对象上
2. 修改某些Object方法的返回结果，让其变得更合理。比如，Object.defineProperty(obj, name, desc)在无法定义属性时，会抛出一个错误，而Reflect.defineProperty(obj, name, desc)则会返回false。
3. 让Object操作都变成函数行为。某些Object操作是命令式，比如name in obj和delete obj[name]，而Reflect.has(obj, name)和Reflect.deleteProperty(obj, name)让它们变成了函数行为。
4. Reflect对象的方法与Proxy对象的方法一一对应，只要是Proxy对象的方法，就能在Reflect对象上找到对应的方法。这就让Proxy对象可以方便地调用对应的Reflect方法，完成默认行为，作为修改行为的基础。也就是说，不管Proxy怎么修改默认行为，你总可以在Reflect上获取默认行为。

## Promise 对象
Promise 是异步编程的一种解决方案，比传统的解决方案——回调函数和事件——更合理和更强大。所谓Promise，简单说就是一个容器，里面保存着某个未来才会结束的事件（通常是一个异步操作）的结果。从语法上说，Promise 是一个对象，从它可以获取异步操作的消息。

Promise对象有以下两个特点：
1. 对象的状态不受外界影响。Promise对象代表一个异步操作，有三种状态：pending（进行中）、fulfilled（已成功）和rejected（已失败）。只有异步操作的结果，可以决定当前是哪一种状态，任何其他操作都无法改变这个状态。
2. 一旦状态改变，就不会再变，任何时候都可以得到这个结果。如果改变已经发生了，你再对Promise对象添加回调函数，也会立即得到这个结果。

Promise也有一些缺点。首先，无法取消Promise，一旦新建它就会立即执行，无法中途取消。其次，如果不设置回调函数，Promise内部抛出的错误，不会反应到外部。第三，当处于pending状态时，无法得知目前进展到哪一个阶段（刚刚开始还是即将完成）。

### 用法
```
const promise = new Promise(function(resolve, reject) {
  // ... some code

  if (/* 异步操作成功 */){
    resolve(value); //参数可选
  } else {
    reject(error); //参数可选
  }
});
```
以上创建Promise实例后，可以用then函数指定resolved状态和rejected状态的回调函数。
```
promise.then(function(value) {
  // success
}, function(error) {
  // failure
});
```

Promise 新建后就会立即执行。
```
let promise = new Promise(function(resolve, reject) {
  console.log('Promise');
  resolve();
});

promise.then(function() {
  console.log('resolved.');
});

console.log('Hi!');

// Promise
// Hi!
// resolved
```
Promise 新建后立即执行，所以首先输出的是Promise。然后，then方法指定的回调函数，将在当前脚本所有同步任务执行完才会执行，所以resolved最后输出。

### then方法
then方法返回的是一个新的Promise实例（注意，不是原来那个Promise实例）。因此可以采用链式写法，即then方法后面再调用另一个then方法。
```
getJSON("/posts.json")
    .then(function (json) {
        return json.post;
    }).then(function (post) {
        // ...
    });
```
第一个回调函数完成以后，会将返回结果作为参数，传入第二个回调函数。

如果then方法指定的回调函数，返回的是另一个Promise对象。这时，第二个then方法指定的回调函数，就会等待这个新的Promise对象状态发生变化。如果变为resolved，就调用第二个then方法的第一个回调函数，如果状态变为rejected，就调用第二个then方法的第二个回调函数。

### catch方法
Promise.prototype.catch()方法是.then(null, rejection)或.then(undefined, rejection)的别名，用于指定发生错误时的回调函数。

### finally方法
finally()方法用于指定不管 Promise 对象最后状态如何，都会执行的操作。该方法是 ES2018 引入标准的。

### Promise.all()
Promise.all()方法用于将多个 Promise 实例，包装成一个新的 Promise 实例。
```
const p = Promise.all([p1, p2, p3]);
```
p的状态由p1、p2、p3决定，分成两种情况：
1. 只有p1、p2、p3的状态都变成fulfilled，p的状态才会变成fulfilled，此时p1、p2、p3的返回值组成一个数组，传递给p的回调函数。
2. 只要p1、p2、p3之中有一个被rejected，p的状态就变成rejected，此时第一个被reject的实例的返回值，会传递给p的回调函数。

### Promise.race()方法
同样是将多个 Promise 实例，包装成一个新的 Promise 实例。
```
const p = Promise.race([p1, p2, p3]);
```
只要p1、p2、p3之中有一个实例率先改变状态，p的状态就跟着改变。那个率先改变的 Promise 实例的返回值，就传递给p的回调函数。

### Promise.resolve()
有时需要将现有对象转为 Promise 对象，Promise.resolve()方法就起到这个作用。
```
Promise.resolve('foo')
// 等价于
new Promise(resolve => resolve('foo'))
```

### Promise.reject()
Promise.reject(reason)方法也会返回一个新的 Promise 实例，该实例的状态为rejected。

## Iterator 和 for...of 循环
一个对象如果要具备可被for...of循环调用的 Iterator 接口，就必须在Symbol.iterator的属性上部署遍历器生成方法（原型链上的对象具有该方法也可）。

JS自带的数据结构基本都原生具备了 Iterator 接口，如果要对自定义的对象增加Iterator 接口：
```
let obj = {
  data: [ 'hello', 'world' ],
  [Symbol.iterator]() {
    const self = this;
    let index = 0;
    return {
      next() {
        if (index < self.data.length) {
          return {
            value: self.data[index++],
            done: false
          };
        } else {
          return { value: undefined, done: true };
        }
      }
    };
  }
};
```

## Generator 函数
Generator 函数是 ES6 提供的一种异步编程解决方案，类似Python的Generator，是协程的一种实现。
```
function* helloWorldGenerator() {
  yield 'hello';
  yield 'world';
  return 'ending';
}

var hw = helloWorldGenerator();

hw.next()
// { value: 'hello', done: false }

hw.next()
// { value: 'world', done: false }

hw.next()
// { value: 'ending', done: true }

hw.next()
// { value: undefined, done: true }
```

调用 Generator 函数后，该函数并不执行，返回的也不是函数运行结果，而是一个指向内部状态的指针对象，也就是遍历器对象（Iterator Object）。必须调用遍历器对象的next方法，使得指针移向下一个状态。

### yield 表达式
遍历器对象的next方法的运行逻辑如下。
1. 遇到yield表达式，就暂停执行后面的操作，并将紧跟在yield后面的那个表达式的值，作为返回的对象的value属性值。
2. 下一次调用next方法时，再继续往下执行，直到遇到下一个yield表达式。
3. 如果没有再遇到新的yield表达式，就一直运行到函数结束，直到return语句为止，并将return语句后面的表达式的值，作为返回的对象的value属性值。
4. 如果该函数没有return语句，则返回的对象的value属性值为undefined。

yield表达式后面的表达式，只有当调用next方法、内部指针指向该语句时才会执行

### 与 Iterator 接口的关系
Generator 函数就是遍历器生成函数

### next 方法的参数
yield表达式本身没有返回值，或者说总是返回undefined。next方法可以带一个参数，该参数就会被当作上一个yield表达式的返回值：
```
function* f() {
  for(var i = 0; true; i++) {
    var reset = yield i;
    if(reset) { i = -1; }
  }
}

var g = f();

g.next() // { value: 0, done: false }
g.next() // { value: 1, done: false }
g.next(true) // { value: 0, done: false }
```

> 由于next方法的参数表示上一个yield表达式的返回值，所以在第一次使用next方法时，传递参数是无效的

### for...of 循环
for...of循环可以自动遍历 Generator 函数运行时生成的Iterator对象，且此时不再需要调用next方法。

### Generator.prototype.throw()
Generator 函数返回的遍历器对象，都有一个throw方法，可以在函数体外抛出错误，然后在 Generator 函数体内捕获。
```
var g = function* () {
  try {
    yield;
  } catch (e) {
    console.log('内部捕获', e);
  }
};

var i = g();
i.next();

try {
  i.throw('a');
  i.throw('b');
} catch (e) {
  console.log('外部捕获', e);
}
// 内部捕获 a
// 外部捕获 b
```
上面代码中，遍历器对象i连续抛出两个错误。第一个错误被 Generator 函数体内的catch语句捕获。i第二次抛出错误，由于 Generator 函数内部的catch语句已经执行过了，不会再捕捉到这个错误了，所以这个错误就被抛出了 Generator 函数体，被函数体外的catch语句捕获。

> 注意，不要混淆遍历器对象的throw方法和全局的throw方法

throw方法抛出的错误要被内部捕获，前提是必须至少执行过一次next方法。
```
function* gen() {
  try {
    yield 1;
  } catch (e) {
    console.log('内部捕获');
  }
}

var g = gen();
g.throw(1);
// Uncaught 1
```
上面代码中，g.throw(1)执行时，next方法一次都没有执行过。这时，抛出的错误不会被内部捕获，而是直接在外部抛出，导致程序出错。这种行为其实很好理解，因为第一次执行next方法，等同于启动执行 Generator 函数的内部代码，否则 Generator 函数还没有开始执行，这时throw方法抛错只可能抛出在函数外部。

**throw方法被捕获以后，会附带执行下一条yield表达式。或者说，会附带执行一次next方法。**

Generator 函数体外抛出的错误，可以在函数体内捕获；反过来，Generator 函数体内抛出的错误，也可以被函数体外的catch捕获。
```
function* foo() {
  var x = yield 3;
  var y = x.toUpperCase();
  yield y;
}

var it = foo();

it.next(); // { value:3, done:false }

try {
  it.next(42);
} catch (err) {
  console.log(err);
}
```

### Generator.prototype.return()
Generator 函数返回的遍历器对象，还有一个return方法，可以返回给定的值，并且终结遍历 Generator 函数。

如果 Generator 函数内部有try...finally代码块，且正在执行try代码块，那么return方法会导致立刻进入finally代码块，执行完以后，整个函数才会结束
```
function* numbers () {
  yield 1;
  try {
    yield 2;
    yield 3;
  } finally {
    yield 4;
    yield 5;
  }
  yield 6;
}
var g = numbers();
g.next() // { value: 1, done: false }
g.next() // { value: 2, done: false }
g.return(7) // { value: 4, done: false }
g.next() // { value: 5, done: false }
g.next() // { value: 7, done: true }
```
执行g.return(7)后，`yield 3;` 被跳过，直接到`yield 4;`，所以返回value为4；然后再执行next，执行到`yield 5;`，最后一次执行next，则返回了`g.return(7)`传入的7。

### yield* 表达式
ES6 提供了yield*表达式，作为解决办法，用来在一个 Generator 函数里面执行另一个 Generator 函数，而不用手动遍历。
```
function* foo() {
  yield 'a';
  yield 'b';
}

function* bar() {
  yield 'x';
  yield* foo();
  yield 'y';
}

// 等同于
function* bar() {
  yield 'x';
  yield 'a';
  yield 'b';
  yield 'y';
}

// 等同于
function* bar() {
  yield 'x';
  for (let v of foo()) {
    yield v;
  }
  yield 'y';
}
```

如果被代理的 Generator 函数有return语句，那么就可以向代理它的 Generator 函数返回数据。
```
function* foo() {
  yield 2;
  yield 3;
  return "foo";
}

function* bar() {
  yield 1;
  var v = yield* foo();
  console.log("v: " + v);
  yield 4;
}

var it = bar();

it.next()
// {value: 1, done: false}
it.next()
// {value: 2, done: false}
it.next()
// {value: 3, done: false}
it.next();
// "v: foo"
// {value: 4, done: false}
it.next()
// {value: undefined, done: true}
```
可以打印v

## async/await
直接使用 Generator 函数来完成异步编程还是稍显麻烦，并且不算很方便。ES2017 标准引入了 async/await函数，使得异步操作变得更加方便，它本质上是 Generator 函数结合 Promise 的语法糖。

`async`函数的返回值是 Promise 对象，而`await`命令后面一般是一个 Promise 对象，返回该Promise对象的结果。如果不是 Promise 对象，就直接返回对应的值。所以`await`命令后面可以调用`async`函数。
```
async function timeout(ms) {
    await new Promise((resolve) => {
        setTimeout(resolve, ms);
    });
}

async function asyncPrint(value, ms) {
    await timeout(ms);
    console.log(value);
}

asyncPrint('hello world', 500);
```
`console.log(value);`会在 timeout 函数延迟500ms执行resolve()后才执行，也就是await会等待后面的Pormise的结果。

### 错误处理
任何一个await语句后面的 Promise 对象变为reject状态，那么整个async函数都会中断执行。
```
async function f() {
  await Promise.reject('出错了');
  await Promise.resolve('hello world'); // 不会执行
}
```
如果希望即使前一个异步操作失败，也不要中断后面的异步操作。这时可以将第一个await放在try...catch结构里面
```
async function f() {
  await Promise.reject('出错了')
    .catch(e => console.log(e));
  return await Promise.resolve('hello world');
}

f()
.then(v => console.log(v))
// 出错了
// hello world
```

如果await后面的异步操作出错，那么等同于async函数返回的 Promise 对象被reject

### 并发执行
连续的多个await后面的代码，执行关系是顺序执行，如果需要并发执行，可以使用如下的方式：
```
// 写法一
let [foo, bar] = await Promise.all([getFoo(), getBar()]);

// 写法二
let fooPromise = getFoo();
let barPromise = getBar();
let foo = await fooPromise;
let bar = await barPromise;
```

## class
省略

## 模块
省略
## 	JavaScript实现
JavaScript的实现包含三个部分：
1. 核心（ECMAScript）
2. 文档对象模型（DOM）
3. 浏览器对象模型（BOM）

## JavaScript引擎简介
JavaScript引擎主要功能就是帮助我们将JavaScript代码翻译CPU所能认识指令，最终被CPU执行；

* V8：目前最流行的JavaScript引擎，由Google开发；
* JavaScriptCore：Webkit中内置的JavaScript引擎，由苹果公司开发；

Web APIs 是提供给引擎的功能，但不是 JavaScript 语言的一部分。引擎可以通过浏览器访问它们，并有助于访问数据或增强浏览器功能。比如文档对象模型（DOM）和获取 API。浏览器运行时和 Node.js 是运行时环境的例子。

每当 JavaScript 引擎接收到脚本文件时，它首先会创建一个默认的执行上下文，称为 全局执行上下文 (GEC)。

浏览器提供window全局对象

* 本地对象 ( native object ) Object、Function、Number等。由 ECMAScript 实现提供并独立于宿主环境的任何对象
* 内置对象 ( built-in object )由 ECMAScript 实现提供并独立于宿主环境的，在程序开始执行就出现的对象。Global、Math、JSON
* 宿主对象 host object  比如Window

## 在HTML中使用JS
在`head`标签中使用`script`标签，可以选择在`script`标签内嵌入JS代码，或者使用`script`标签的src属性引入外部JS代码的方式。

默认情况（可以通过defer、async来调整），head中的JS文件，会使浏览器在全部JS代码被下载、解析和执行完后，才开始显示body标签内的页面，所以也可以把`script`标签放到body标签内的最后位置。

noscript标签可以在浏览器不支持JS，或者支持但被禁用的情况下显示。

## 
JavaScript是单线程语言，实际是负责解释和执行JS代码的线程只有一个，但宿主（如浏览器）是多线程，比如进行ajax请求的线程、监控用户事件的线程、定时器线程、读写文件的线程。所以即使JavaScript程序执行是单线程，但是也可以实现异步执行。

## 基本概念
### 变量
变量是松散类型的，可以保存任何类型的数据。在函数中使用var声明变量，则变量为函数内的局部变量，如果省略var，则为全局变量。

变量提升：将变量声明提升到所在作用域的顶部，即可以在声明前使用。
console.log(a);//undefined
var a=1;
console.log(a);//1
如果没有变量提升则第一行代码会发生错误

### 数据类型
有5种简单数据类型（也称基本数据类型）：Undefined、Null、Boolean、Number和String，一种复杂数据类型（也称引用类型或对象类型）—Object，Object本质上是由一组无序的键值对组成的。引用类型的实例（即对象）会继承object类型的属性和方法。但BOM和DOM中的对象是宿主对象，并用非ECMAScript定义的对象，所以不一定继承。

* 基本数据类型的值本身无法被改变，而是必须创建和跟踪一个新的数据。
* typeof操作符检测数据类型
* Boolean类型只有两个字面值：小写的true和false，其他类型都有与这两个值等价的值。
* 字符串可以由双引号或单引号表示，和Java一样不可变
* 基本包装类型：Boolean、Number、String是特殊的引用类型。对基本类型：Boolean、Number、String调用它们的方法时，后台会创建它们的基本包装类型。基本包装类型只存在于将基本类型当作引用类型使用的瞬间。

因为JavaScript是动态语言，所以声明JavaScript对象，并不需要像Java等静态语言必须先声明一个类，而是直接像声明一个map或者json数据格式一样：
```
const person = {
  name: {
    first: "Bob",
    last: "Smith",
  },
  // …
};
```

并且可以随时添加属性：
```
let foo = {
        a: 1,
    }
console.log(foo.a) // 打印 1
console.log(foo.b) // 打印 undefined
foo.b = 2  // 给foo添加属性b，值为2
console.log(foo.b) // 再次打印为 2
```

**所以要重点认识到，JavaScript动态语言的使用表现**

**点表示法**，**括号表示法**，都可以访问JavaScript的对象属性：
```
person.age;
person.name.first;

person["age"];
person["name"]["first"];
```

## 作用域
* 每个函数都有自己的执行环境（execution context），每个执行环境都有一个与之关联的变量对象（环境中定义的所有变量和函数都保存在这个对象中）（variable object），每个执行环境对应的作用域链由当前环境到各层包含环境一直到全局执行环境的每一个变量对象构成。（以函数定义的位置查找变量，而不是执行的位置）
* 作用域链本质上是一个指向变量对象的指针列表，它只引用但不实际包含变量对象。
* JS的ES5版本，没有块级作用域，所以例如for之类的花括号里的代码块和外部并没有隔离。ES6中提供了let和const，具有块级作用域。
* 当函数可以记住并访问所在的词法作用域时，就产生了闭包，即便函数是在当前词法作用域之外执行。闭包就是函数对当前词法作用域的引用。函数在当前词法作用域之外执行，才通过闭包访问外部函数的变量，否则通过词法作用域的查找规则访问。
* 词法作用域的查找规则，是闭包的一部分（但却是非常重要的一部分）
* 闭包只能取得外部函数中任何变量的最后值，即闭包保存的是整个变量对象，而不是某个特殊的值。

## 函数
### 参数
ECMAScript函数不严格要求传递进来多少个参数，也不在乎传进来参数是什么数据类型。也就是说，即使你定义的函数只接收两个参数，在调用这个函数的时候也未必一定要传递两个参数，可传一个、三个甚至不传；或者定义的函数未定义参数，调用时也可以接收参数。

因为ECMAScript中的参数在内部是用一个数组表示，函数接收到的始终是这个数组。在函数体内可通过arguments对象来访问这个参数数组。arguments的length属性可获知有多少个参数传递给了函数。arguments的值和命名参数保持同步，修改其中一个，会自动反映到另一个，但内存空间是独立的（严格模式无效，且重写arguments会导致语法错误）。arguments的长度由传入的参数个数决定，而不是定义函数时的命名参数个数。没有传递值的命名参数将自动赋值undefined。**总结来说**：JS函数的命名参数只提供便利，而非必需。

函数的参数传递：
javascript和java的参数传递都是按值传递的，即只是把实参的值赋给形参，形参的值修改不会影响到实参。对于传入对象变量，实参赋给形参的值是对象的引用（对象的内存地址），所以实参和形参引用同一个对象，形参修改引用的该对象，会使实参“看似”也被修改；如果修改形参的值（即对对象的引用，例如引用到另一个对象），实参依然引用原对象。

### Function类型
* 函数也是对象，函数名实际就是指向函数类型的指针。
* 函数声明可以在声明前调用函数，函数表达式只能在表达式后调用函数。
* 函数名本身是变量，所以函数也可以作为值来使用，例如作为参数传递到另一个函数或者作为另一个函数的结果返回。
* 函数内部有两个特殊对象：arguments和this，this引用的是函数执行的环境对象，如在全局作用域中调用函数，this对象引用的就是window（严格模式为undefined），否则引用调用该函数的对象（通过把函数赋值为对象的属性），非全局作用域且无对象调用则this值为undefined。
* 匿名函数this为window（浏览器环境），严格模式为undefined
* arguments包含传入的参数，另外还有一个名为calle（指针，指向拥有这个arguments的函数）的属性。
* 函数是对象，所以函数也有属性和方法，例如length和prototype。length为接收的命名参数个数。

apply和call（Function的原型中的函数）用于在特定作用域中调用函数，类似Kotlin的作用域函数，只不过JS的作用域函数需要传入作用域对象，而Kotlin中作用域函数的调用者就是作用域对象。bind函数可以使一个函数绑定作用域，并返回这个新函数。

### 闭包
由于内部函数可以访问外部函数的作用域，因此当内部函数生存周期大于外部函数时，外部函数中定义的变量和函数的生存周期将比内部函数执行的持续时间要长。当内部函数以某一种方式被任何一个外部函数之外的任何作用域访问时，就会创建闭包。

可以访问另一个函数的作用域中的变量的函数就是闭包。函数的活动对象表示函数的自身作用域，闭包会把外部函数的活动对象以及更上层的各级活动对象直到全局作用域添加到闭包的的作用域链中，也就是会引用外部函数的作用域链引用的各级活动对象。
```
function makeAdder(x) {
  return function (y) {
    return x + y;
  };
}

var add5 = makeAdder(5);
var add10 = makeAdder(10);

console.log(add5(2)); // 7
console.log(add10(2)); // 12
```

因为var声明变量，不会产生块级作用域，所以每个闭包引用的createFunctions()的活动对象中的i是同一个，所以如下代码会导致返回的函数数组中的每个函数返回的都是10：
```
function createFunctions(){
	var result = new Array();
	for (var i = 0; i < 10; i++){
		result[i] = function(){
		return i; 
		};
	}
    return result;
}
```
闭包在循环中被创建，但他们共享了同一个词法作用域，在这个作用域中存在一个变量 i。这是因为变量 i 使用 var 进行声明，由于变量提升，所以具有函数作用域。当 function 的回调执行时，i 的值被决定。由于循环在事件触发之前早已执行完毕，变量 i（被10个闭包所共享）已经指向了最后一次循环也就是9。

如果使用ES6中的let，则可以达到正常的状况

函数调用时，会自动取得this和arguments属性，这两个属性默认只会搜索自身的活动对象，而不会访问外部函数中的这两个变量。默认情况，this为window对象。可以通过call()函数或者主动赋值this的方式来修改。

## Global对象
不属于任何其他对象的属性和方法，最终都是它的属性和方法。事实上，没有全局变量或全局函数；所有在全局作用域中定义的属性和函数，都是 Global 对象的属性

## 原型链 面向对象程序设计
官方文档：
* [对象原型](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Objects/Object_prototypes)，写得浅显易懂
* [面向对象](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Objects/Object-oriented_programming)
ES5中，JS没有类，对象只是：无序属性的集合，属性可以包含基本值、对象和函数。可以把这里的对象想象成键值对。创建一个自定义对象，可以创建一个Object实例，然后为它添加属性和方法，或者使用对象字面量的方式。

### 数据属性和访问器属性
JS中通过new Object()或者字面量方式设置的属性都是`数据属性`，访问器属性要通过Object.defineProperty函数来定义

### 创建对象
创建对象可以使用new Object或者字面量等。比如：
```
function createPerson(name) {
  const obj = {};
  obj.name = name;
  obj.introduceSelf = function () {
    console.log(`你好！我是 ${this.name}。`);
  };
  return obj;
}
```

#### 构造函数模式
任何函数使用new操作符就成为构造函数，Object和Date等都是函数。此方式会经历：1. 创建一个新对象 2. this指向新对象 3. 执行构造函数的代码 4. 返回新对象

```
function Person(name) {
  this.name = name;
  this.introduceSelf = function () {
    console.log(`你好！我是 ${this.name}。`);
  };
}

const salva = new Person("Salva");
salva.name;
salva.introduceSelf();
```

构造函数的属性和方法相当于静态方法。

使用构造函数的方式，如果内部为对象添加函数，那么每个对象都会包含一个新的函数对象，而这些函数的功能是一摸一样的。我们可以通过使构造函数内赋值的函数为外部已定义的同一函数（全局函数），而不是每次都new Function。

#### 原型模式
虽然全局函数可以解决构造函数的缺点，但是这个全局函数实际只是给某个对象调用，这样的全局函数不太符合封装性。所以可以使用原型模式。

函数都有一个prototype属性，指向函数的原型对象。原型对象保存实例的属性和方法（即对象调用的方法，而非类的静态方法）。实例也可以定义自身的属性和方法。

原型对象的用途是包含特定类型的所有实例共享的属性和方法，也就是可以在原型中添加属性和方法。

当调用构造函数创建一个新实例，该实例内部将包含一个指针（内部属性），指向构造函数的原型对象。所有原型对象都有一个constructor属性，prototype.constructor指向prototype属性所在的函数，如Person.prototype.constructor指向Person函数。

自定义的构造函数的原型对象默认只会有constructor属性，其他方法从Object继承而来。Person函数的prototype指向原型对象，person实例中的隐含指针[[Prototype]]
也指向原型对象。

读取某个对象的某个属性时，会先搜索对象实例本身，搜到则返回，未搜到则搜索原型对象。可以通过对象实例访问保存在原型的值，但不能通过对象实例重写原型中的值。在实例中添加一个属性如果和原型中的一个属性同名，就会在实例中创建该属性，该属性将会屏蔽原型中的那个属性。但通过delete可以删除实例属性，而访问原型中的同名属性。但如果原型中的属性是引用类型，在对象实例中对该属性引用的对象进行修改时，会更改原型中的属性，影响到其他实例。如属性friends["a","b"];person1.friends.push("c")；但如果是person1.friends=["a","b","c"];则不会影响原型，因为实例中的friends指向了新的对象。

> 原型模式中，由于共享的性质，引用类型的属性也会共享，这会导致不同的实例之间都能修改该属性，所以一般会结合构造函数模式和原型模式，构造函数定义实例属性，原型定义方法和共享的属性

### 继承
#### 原型链继承
使构造函数A的prototype引用另一个类型B的实例，而该实例内部隐含的[[Prototype]]会指向B的原型，层层递进直到Object，就实现了原型链继承

## DOM和BOM
DOM是为了操作文档出现的API，document是其的一个对象

BOM是为了操作浏览器出现的API，window是其的一个对象，web中js的顶级作用域的this就是window。

> DOM/BOM都是由浏览器提供

DOM1级
* DOM中，Node类型描述节点之间的关系和操作节点，所有节点类型都继承自Node类型。具有操作节点的方法
* Document类型（Node子类型）提供文档信息、创建和查找元素和文档写入（write方法）。
* Element（Node子类型）节点对应文档中的所有HTML元素，可以操作这些元素的内容和特性。

* DOM扩展支持通过css选择符查找元素，和元素遍历（忽略文本节点）
* 查找元素可以通过id、tagName、name、className、selector。ById只有Document类型可以使用，ByName只有HTMLDocument可以使用，xml中不能使用。
* HTML5为标准的DOM定义了许多扩展功能，getElementsByClassName()、classList、innerHTML、outerHTML、insertAdjacentHTML()、焦点管理、readyState、兼容模式、head、字符集属性、滚动页面。

DOM2级
* DOM2级核心模块为不同的DOM类型引入了与XML命名空间有关的方法，仅用于XML和XHTML
* 为DocumentType （document.doctype）新增三个属性
* 定义了以编程方式创建Document实例和DocumentType对象。
* DOM2级视图模块添加defaultView属性，指向给定文档的窗口（或框架），IE中是parentWindow。document.defaultView||document.parentWindow
* DOM2级HTML模块为document.implementation增加createHTMLDocument()方法
* DOM3级引入两个比较节点的方法：isSameNode()和isEqualNode()，前者相同，后者相等。为DOM节点添加setUserData()方法。
* DOM2级操作样式和遍历模块
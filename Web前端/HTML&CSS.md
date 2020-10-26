# HTML&CSS
## 块级元素和行内元素
块级元素（对应块级盒子）的表现：
1. 和父容器一样宽
2. 每个元素都会换行
3. width 和 height 属性可以发挥作用
4. 内边距（padding）, 外边距（margin） 和 边框（border） 会将其他元素从当前盒子周围“推开”

行内元素（对应行内盒子）的表现：
1. 不会产生换行
2. width 和 height 属性将不起作用。
3. 垂直方向的内边距、外边距以及边框会被应用（比如padding变大会让背景会变大）但是不会把其他处于 inline 状态的盒子推开。
4. 水平方向的内边距、外边距以及边框会被应用而且也会把其他处于 inline 状态的盒子推开。

img、input、textarea、select、object等标签虽是行内元素(inline element)，但也可以设置宽高等属性，因为它们是替换元素(replaced element)，根据标签和属性来决定显示的内容。其他非替换元素会直接显示内容。

通过display属性，可以控制元素的外部显示类型。
## 内部和外部显示类型
css的box模型有一个外部显示类型，来决定盒子是块级还是行内。同样盒模型还有内部显示类型，它决定了盒子内部元素是如何布局的。默认情况下是按照 正常文档流 布局，也意味着它们和其他块元素以及行内元素一样。

但是，我们可以通过使用类似  flex 的 display 属性值来更改内部显示类型。 如果设置 display: flex，在一个元素上，外部显示类型是 block，但是内部显示类型修改为 flex。 该盒子的所有直接子元素都会成为flex元素，会根据 弹性盒子（Flexbox ）规则进行布局。

## 盒子模型
css中所有高度由两个模型产生，其一为box盒模型，对应样式为“height+padding+margin”，另一个是line盒模型，对应样式为“line-height”

浏览器默认，元素的width和height属性设定为content，元素最终的宽高还要考虑padding、border和margin，但可以通过box-sizing改变。

padding和margin在一般情况下作用相同。
使用margin：
1.需要在border外侧添加空白
2.空白处不需要背景（色）
3.上下相连的两个盒子之间的空白，需要相互抵消而取较大值。
使用padding
1.需要在border内侧添加空白
2.空白处需要背景（色）
3.上下相连的两个盒子之间的空白，需要等于两者之和

两个或多个相邻（没有被非空内容、padding、border或clear分隔开）的普通流的块元素垂直方向上的margin会折叠（取两者较大值）。

将行内元素设为inline-block，如果内容无文字，则不会和兄弟行内元素的文本按基线或其他设定的其他对齐


https://www.w3.org/TR/CSS22/visuren.html
https://www.w3.org/TR/CSS22/visudet.html#leading
## 匿名块盒子和行内盒子
例如一个DIV块元素包含一个文字（行内元素）和一个P元素（块元素），那么文字会被一个匿名块盒子包住。
```
<DIV>
  Some text //将被匿名块包住，实际是参与BFC
  <P>More text
</DIV>
```

而如下例子，Some 和text 将被一个生成的匿名行内盒子包住。
```
<p>Some <em>emphasized</em> text</p>
```

## 定位方案
元素在HTML中的定位方案，分为3种：
1. 普通流：普通流包括块级元素对应的块级格式化（block formatting）、行内元素对应的行内格式化（inline formatting），和块级元素、行内元素对应的相对位置（relative positioning ）
2. 浮动：在浮动模型中，先按普通流布局，然后脱离普通流，并尽可能向左或右移动
3. 绝对定位：在绝对定位模型中，元素完全从普通流中移除，并根据设置了已定位的父元素来定位

### 普通流
普通流中的表格、块级和行内元素属于格式上下文。块级元素参与块级格式化上下文，行内元素参与行内格式化上下文。表格格式化上下文在
[官方文档](https://www.w3.org/TR/CSS22/tables.html)中描述。

#### 块级格式化上下文（Block formatting context）
BFC是 W3C CSS 2.1 规范中的一个概念，它决定了元素如何对其内容进行定位，以及与其他元素的关系和相互作用。块级元素将参与BFC。

以下情况下创建一个新的BFC：

* 根元素（<html>）
* 浮动元素（元素的 float 不是 none）
* 绝对定位元素（元素的 position 为 absolute 或 fixed）
* 行内块元素（元素的 display 为 inline-block）
* 表格单元格（元素的 display 为 table-cell，HTML表格单元格默认为该值）
* 表格标题（元素的 display 为 table-caption，HTML表格标题默认为该值）
* 匿名表格单元格元素（元素的 display 为 table、table-row、 table-row-group、table-header-group、table-footer-group（分别是HTML table、row、tbody、thead、tfoot 的默认属性）或 inline-table）
* 块级元素的 overflow 属性不为 visible
* display 值为 flow-root 的元素
* contain 值为 layout、content 或 paint 的元素
* 弹性元素（display 为 flex 或 inline-flex 元素的直接子元素）
* 网格元素（display 为 grid 或 inline-grid 元素的直接子元素）
* 多列容器（元素的 column-count 或 column-width 不为 auto，包括 column-count 为 1）
* 元素属性 column-span 设置为 all

**BFC的约束规则**：
1. 内部的块元素会在垂直方向自顶向下放置
2. 兄弟块元素的垂直方向距离由margin属性决定(属于同一个BFC的两个相邻块的margin会发生折叠)
3. 每个块元素的左外边缘与包含块（containing block）的左边相接触(对于从左往右的格式化，反之)。即使存在浮动也是如此
4. BFC的区域不会与float的元素区域重叠
5. 计算BFC的高度时，浮动子元素也参与计算（浮动元素脱离正常文档流，导致父元素不会计算浮动元素的高度，通过创建BFC）
6. BFC就是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面元素，反之亦然

#### 行内格式化上下文（Inline formatting contexts）
IFC由一个不包含块元素的块元素创建。

**IFC的约束规则**：
1. 其内部的行内元素从块元素顶部开始水平放置
2. 水平的margins、 borders，和 padding 在行内元素间有效
...

### 浮动
最初，引入 float 属性是为了能让 web 开发人员实现简单的布局，使文本中浮动的图像，文字环绕在它的左边或右边。但现实情况是任何元素都可以浮动，而不只是图片元素。

原本<img>元素之后的<p>元素会另起一行，但如果给<img>加上了float属性，那么浮动元素（这个<img>元素）会脱离正常的文档布局流，并吸附到其父容器的左边（Y轴位置不影响），<p>元素的Y轴开始位置就会忽略<img>元素，也就相当于和<img>元素从同一个高度开始显示，并且实际<p>元素的x轴位置仍然是从父级的最左侧开始布局的（默认情况，如果<p>元素是另一个块级格式上下文，则不同），只是<p>元素内的文字会围绕图片，而<p>元素本身的x轴开始位置不受影响。

浮动通常用于创建多个列布局，但这不是原本的目的，并且有一些副作用需要处理。

**浮动的规则**：
1. float盒子将向左或右直到外边缘接触到containing block的边缘，或者另一个float盒子的边缘
2. 如果有一个行内盒子，则float盒子的外顶部将和该行内盒子的顶部对齐
3. 如果没有足够的水平空间容纳float盒子，则将向下移动，直到可以容纳或者没有其他float盒子存在为止
4. float盒子不在普通流中，float盒子前后的未定位元素按普通流放置，就像float盒子不存在一样。但是会根据需要，缩短在float盒子旁边创建的当前和后续行内元素，以便为float的margin框腾出空间
5. 如果缩短的行内盒子太小而无法容纳任何内容，则行内盒子将向下移动（并重新计算其宽度），直到内容可以容纳或不再存在float为止

#### 清除浮动
所有在浮动下面的自身不浮动的内容都将围绕浮动元素进行包装，如果没有处理这些元素，就可能出现一些需求外的情况。此时可以使用`clear`属性，清除浮动。把这个属性应用到一个元素上时，它主要意味着"此处停止浮动"——这个元素和源码中后面的元素将不浮动，除非您稍后将一个新的float声明应用到此后的另一个元素。

将值应用于非浮动块级框时，这些值具有以下含义：
clear:left; 要求盒子的顶部边框边缘应位于源文档中较早的所有向左浮动框的底部外边缘下方。

其他同理

### 定位
1. static：默认，普通流
2. relative：相对于自己的正常位置
3. absolute：相对于最近的position不为static的父元素(会脱离正常文档流)
4. fixed：相对于浏览器窗口，即使窗口滚动，fixed也不会滚动（会脱离正常文档流）
5. sticky：relative和fixed的结合体。滚动到阈值内，表现为relative，滚动到超过阈值，表现为fixed

### flex布局
设置在容器上的属性
1. flex-direction：决定主轴的方向（即项目的排列方向）
2. flex-wrap：如果一条轴线排不下，如何换行
3. flex-flow：flex-direction和flex-wrap的简写形式
4. justify-content：定义项目在主轴上的对齐方式
5. align-items：定义项目在交叉轴上如何对齐
6. align-content：定义了多根主轴线参照交叉轴的对齐方式。如果项目只有一根轴线，该属性不起作用。

设置在子项上的属性
1. order：定义子项的排列顺序，越小越靠前
2. flex-grow：定义子项的拉伸因子，即子项分配父容器剩余空间的比例。默认为0，即存在剩余空间也不拉伸
3. flex-shrink：定义子项的收缩规则，子项收缩所占的份数，默认为1（当所有子项相加的宽度大于父容器的宽度，每个子项减少总宽度比父容器多出的宽度的1/n）
4. flex-basis：定义了在分配多余空间之前，项目占据的主轴空间。浏览器根据这个属性，计算主轴是否有多余空间。默认值为auto，即项目本来的大小
5. flex：是flex-grow、flex-shrink和flex-basis的简写，默认值为0 1 auto。后两个属性可选
6. align-self：允许单个项目有与其他项目不一样的对齐方式，可覆盖align-items属性。默认为auto，表示继承父元素的align-items属性，如果没有父元素，则等同于stretch
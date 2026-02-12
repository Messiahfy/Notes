# HTML&CSS

## 定位方案
元素在HTML中的定位方案，分为3种：
1. 普通流：普通流包括块级元素对应的块级格式化（block formatting）、行内元素对应的行内格式化（inline formatting），和块级元素、行内元素对应的相对位置（relative positioning ）
2. 浮动：在浮动模型中，先按普通流布局，然后脱离普通流，并尽可能向左或右移动
3. 绝对定位：在绝对定位模型中，元素完全从普通流中移除，并根据设置了已定位的父元素来定位

### 普通流
普通流中的表格、块级和行内元素属于格式上下文。块级元素受到所在的块级格式化上下文约束，行内元素受到所在的行内格式化上下文约束。表格格式化上下文在[官方文档](https://www.w3.org/TR/CSS22/tables.html)中描述。

### 浮动
最初，引入`float`属性是为了使文本中浮动的图像，文字环绕在它的左边或右边。但现实情况是任何元素都可以浮动，而不只是图片元素。

原本`<img>`元素之后的`<p>`元素会另起一行，但如果给`<img>`加上了float属性，那么浮动元素（这个`<img>`元素）会脱离正常的文档布局流，并吸附到其父容器的左边（Y轴位置不影响），`<p>`元素的Y轴开始位置就会忽略`<img>`元素，也就相当于和`<img>`元素从同一个高度开始显示，并且实际`<p>`元素的x轴位置仍然是从父级的最左侧开始布局的（默认情况，如果`<p>`元素是另一个块级格式上下文，则不同），只是`<p>`元素内的文字会围绕图片，而`<p>`元素本身的x轴开始位置不受影响。

因为多个浮动元素可以一行排列，所以浮动通常用于创建多个列布局，但这不是原本的目的，并且有一些副作用需要处理。

**浮动的规则**：
1. float盒子将向左或右直到外边缘接触到containing block的边缘，或者另一个float盒子的边缘
2. 如果有一个行内盒子，则float盒子的外顶部将和该行内盒子的顶部对齐
3. 如果没有足够的水平空间容纳float盒子，则将向下移动，直到可以容纳或者没有其他float盒子存在为止
4. float盒子不在普通流中，float盒子前后的未定位元素按普通流放置，就像float盒子不存在一样。但是会根据需要，缩短在float盒子旁边创建的当前和后续行内元素，以便为float的margin框腾出空间
5. 如果缩短的行内盒子太小而无法容纳任何内容，则行内盒子将向下移动（并重新计算其宽度），直到内容可以容纳或不再存在float为止

#### 清除浮动
浮动元素的存在可能导致父容器高度塌陷（后文的BFC将介绍）、元素重叠和错位等问题。此时可以使用`clear`属性，清除浮动造成的影响。把这个属性应用到一个元素上时，这个元素和源码中后面的元素将不受到浮动影响，除非您稍后将一个新的float声明应用到此后的另一个元素。

### 定位
1. static：默认，普通流
2. relative：相对于自己的正常位置偏移，不脱离正常文档流
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

## 盒子模型
CSS会将所有元素表示为一个个矩形的盒子（box），CSS 决定这些盒子的大小、位置以及属性（例如颜色、背景、边框尺寸…）。每个盒子由 `content + padding + border + margin` 组成。

元素的width和height属性可以决定content，最终的宽高还要考虑padding、border和margin，但可以通过box-sizing改变。

### 区块盒子（block boxes）和行内盒子（inline boxes）
盒模型是 CSS 布局的基础，无论是 block、inline 还是 inline-block 元素，在渲染时都会被当作“盒子”处理。inline 元素（如`<span>`、`<a>`、`<strong>` 等）在布局时也有自己的内容区、内边距（padding）、边框（border）、外边距（margin），只是它们的盒模型表现和 block 元素不同。

可以使用`display`属性为显示类型设置各种值，该属性可以有多种值。比如`block`、`inline`、`inline-block`，以及用于布局子元素的`flex`、`grid`等。

`display: inline-block` 是 display 的一个特殊值，它提供了介于 inline 和 block 之间的中间位置。如果不希望项目换行，但又希望它使用 width 和 height 值并避免出现上述重叠现象，可以使用它。

**盒子的类型决定了盒子在页面流中的行为方式以及与页面上其他盒子的关系。**

### 盒子的外部显示类型
区块盒子的表现：
1. 如果未指定 width，方框将沿行向扩展，以填充其容器中的可用空间。在大多数情况下，盒子会变得与其容器一样宽，占据可用空间的 100%。
2. 会产生换行
3. width 和 height 属性可以发挥作用
4. 内边距（padding）、外边距（margin）和边框（border）会将其他元素从当前盒子周围“推开”

如`<h1>`和`<p>`，默认使用 block 作为外部显示类型。

行内盒子的表现：
1. 不会产生换行
2. width 和 height 属性将不起作用。
3. 垂直方向的内边距、外边距以及边框会被应用（比如padding变大会让背景会变大）但是不会把其他处于 inline 状态的盒子推开。
4. 水平方向的内边距、外边距以及边框会被应用而且也会把其他处于 inline 状态的盒子推开。

如`<a>`、`<span>`、`<em>`以及`<strong>`，默认使用 inline 作为外部显示类型。

### 盒子的内部显示类型
盒子还有内部显示类型，它决定了盒子内元素的布局方式。

区块和行内布局是网络上的默认行为方式。默认情况下，在没有任何其他指令的情况下，方框内的元素也会以标准流的方式布局，并表现为区块或行内盒子。

可以通过设置`display: flex;`来更改内部显示类型。该元素仍将使用外部显示类型 block 但内部显示类型将变为 flex。该方框的任何直接子代都将成为弹性（flex）项，并按照弹性盒子规范执行。

### 普通流的block和inline盒子
普通流中的任何一个盒子总是某个格式上下文（`formatting context`）中的一部分。块级盒子是在块格式上下文（`block formatting context`，以下简称`BFC`）中工作的盒子，而内联盒子是在内联格式上下文（`inline formatting context`，以下简称`IFC`）中工作的盒子。任何一个盒子总是块级盒子或内联盒子中的一种。

对于`BFC`，规定：在BFC中，盒子会从包含块的顶部开始，按序垂直排列。同级盒子间的垂直距离会由“margin”属性决定。相邻两个块级盒子之间的垂直间距会遵循外边距折叠原则被折叠。

对于`IFC`，规定：在IFC中，盒子会从包含块的顶部开始，按序水平排列。只有水平外边距、边框和内边距会被保留。这些盒子可以以不同的方式在垂直方向上对齐：可以底部对齐或顶部对其，或者按文字底部进行对齐。我们把包含一串盒子的矩形区域称为一个线条框。

### 匿名块盒子和行内盒子
例如一个DIV块元素包含一个文字（行内元素）和一个P元素（块元素），那么文字会被一个匿名块盒子包住。
```
<div>
  Some text //将被匿名块包住
  <P>More text
</div>
```

而如下例子，Some 和text 将被一个生成的匿名行内盒子包住。
```
<p>Some <em>emphasized</em> text</p>
```

## 脱离普通流
下列元素会脱离普通流：
* 浮动的元素
* 通过设置 position 属性为 absolute 或者 fixed 的元素
* 根元素(html) 

脱离普通流的元素会创建一个新的块级格式化上下文（Block Formatting Context: BFC），其中包含的所有元素构成了一个独立的布局环境，与页面中的其他内容分隔开来。而根元素，作为页面中所有内容的容器，自身脱离普通流，为整个文档创建了一个块级格式化上下文。

## 布局和包含块
一个元素的尺寸和位置经常受其包含块（`containing block`）的影响，大多数情况下，包含块就是这个元素最近的祖先块元素的内容区域，但也不是总是这样。可以查看文档：https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_display/Containing_block


## 格式化上下文
格式化上下文分为区块（block）格式化上下文、行内（inline）格式化上下文和弹性（flex）格式化上下文。

### 块级格式化上下文 BFC
BFC就是页面中的一个隔离的独立容器（渲染区域），BFC 内部的元素布局不会影响到外部布局，外部的布局也不会影响到 BFC 内部布局，比如内部的浮动、margin 不会影响外部。创造 BFC 的盒子本身在页面上还是可以和外部元素发生视觉上的重叠（比如通过定位、负 margin 等），但BFC 内部的布局不会“泄漏”到外部，外部的布局也不会“渗透”进来。

BFC 主要解决的是布局影响的隔离，而不是视觉上的绝对不重叠。

以下情况下创建一个新的BFC：
* 根元素（`<html>`）
* 浮动元素（元素的 float 不是 none）
* 绝对定位元素（比如 position 为 absolute 或 sticky
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

**可以查看文档的例子，https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_display/Introduction_to_formatting_contexts**

https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_display/Block_formatting_context

通常，我们会为定位和清除浮动创建新的 BFC，而不是更改布局，因为它将：
1. 包含内部浮动（BFC会计算内部所有元素的高度，包括浮动元素）：BFC 使得让盒子内部的浮动内容和周围的内容等高，避免浮动元素超出父元素边框。
2. 排除外部浮动：正常文档流中建立的 BFC 不得与元素本身所在的块格式化上下文中的任何其他浮动（非子级）重叠。
3. 阻止外边距重叠。

比如div内的img设置浮动，div的高度不包含img的高度，如果给让div创建bfc，那么div的高度就会包含img的高度。因为形成了独立容器，img布局不能影响外部。

可以用如下代码对比是否有BFC的区别：
```html
<style>
  .float {
    float: left;
    width: 100px;
    height: 100px;
    background: red;
    margin-right: 10px;
  }
  .content {
    background: lightblue;
    overflow: hidden; /* 触发BFC，可以对比去掉这个样式的效果 */
  }
</style>

<div class="float"></div>
<div class="content">
  <h2>标题</h2>
  <p>段落内容...</p>
  <div>其他元素</div>
</div>
```
如果没有BFC，float div和content div的布局会重叠，content div中的文字会被浮动元素挤到右边。如果触发了BFC，content div整体都会被挤到浮动元素的右边，而不只是内部的文字。


> 普通的`<div>`元素默认情况不会创建 BFC，虽然它是块级元素容器，但它只是外部和内部显示类型为块级元素，并没有BFC的相关约束（比如不会因为浮动而高度塌陷、阻止外边距重叠）。

### 行内格式化上下文 IFC
行内格式化上下文存在于其他格式上下文中，段落创建了一个内联格式上下文，其中在文本中使用诸如 `<strong>`、`<a>`或 `<span>` 元素等内容。

https://www.w3.org/TR/CSS22/visuren.html
https://www.w3.org/TR/CSS22/visudet.html#leading

## 可替换元素（replaced element）
https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_images/Replaced_element_properties

在 CSS 中，可替换元素（replaced element）的展现效果不是由 CSS 来控制的。这些元素是一种外部对象，它们外观的渲染，是独立于 CSS 的。CSS 可以影响可替换元素的位置，但不会影响到可替换元素自身的内容。

img、input、textarea、select、object等标签虽是行内元素(inline element)，但也可以设置宽高等属性，因为它们是替换元素(replaced element)，根据标签和属性来决定显示的内容。其他非替换元素会直接显示内容。

替换元素的默认 display 属性通常是 inline，但它们的行为会受到其内在尺寸（如图片的宽度和高度）的影响，因此看起来可能有些像 inline-block。默认情况下，`<img>`的 display: inline 不会自动支持块级的上下边距。如果将 `<img>` 的 display 属性显式设置为 inline-block，它仍然表现为行内元素，但会更明确地支持某些块级特性，比如可以设置 margin 的上下边距（inline 元素的上下面边距通常无效）。


## CSS变量
CSS 变量（也称为 CSS 自定义属性）是 CSS3 引入的一个强大特性，允许开发者定义可重复使用的值，并在整个样式表中引用它们。

CSS 变量使用 `--` 前缀定义

```css
.card {
  --card-bg: #ffffff;
  --card-padding: 20px;
}


.button {
  background-color: var(--primary-color);
  font-size: var(--font-size-base);
  padding: var(--spacing-unit);
}

```


/* JavaScript 可以修改 CSS 变量 */
document.documentElement.style.setProperty('--primary-color', '#ff6b6b');
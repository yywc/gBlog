## 前言

grid 布局是 css 中的一种新的布局方式，对盒子和盒子内容的位置及尺寸有很强的控制能力。与 flex 不同，flex 着重于单轴，而 grid 适应于多轴，下面就来做个简单的介绍。

***flex 布局示意***

![flex布局图](https://user-gold-cdn.xitu.io/2019/2/16/168f59dd0aee98ca?w=486&h=70&f=png&s=1641)

***grid 布局示意***

![grid布局图](https://user-gold-cdn.xitu.io/2019/2/16/168f59e0a8189156?w=486&h=70&f=png&s=1703)

## 1. 基本概念

设置 display: grid; 的元素称为容器，它的直接子元素称为项目，但注意子孙元素不是项目。

**grid line**：网格线，构成 grid 布局的结构，分为水平和竖直两种。

**grid track**：两条相邻网格线之间的空间，可以认为是一行或者一列。

**grid cell**：两条相邻行和相邻列之间分割线组成的空间，是 grid 布局中的一个单元。

**grid area**：四条网格线包裹的全部空间，任意数量的 grid cell 组成一个 grid area。

## 2. 容器属性

grid 容器的属性还是有点多的，一共有 18 个，但是很多可以通过简写完成，所以也不用太紧张。

+ grid-template 系列
  + grid-template-columns
  + grid-template-rows
  + grid-template-areas
+ grid-gap 系列
  + grid-column-gap
  + grid-row-gap
+ place-items 系列
  + justify-items
  + align-items
+ place-content 系列
  + justify-content
  + align-content
+ grid 系列
  + grid-auto-columns
  + grid-auto-rows
  + grid-auto-flow

### 2.1 grid-template 系列

#### 2.1.1 grid-template-columns 、grid-template-rows

定义行和列的数量。

```css
.container {
  display: grid;
  width: 500px;
  height: 500px;
  grid-template-columns: 1fr 1fr 1fr;
  grid-template-rows: 1fr 1fr 1fr;
}
```

这样就定义了一个三行三列的 grid 容器，grid-template-rows / grid-template-columns 接收的值有 auto、fr、px 和 %。

介绍一下 fr，其实也不难理解，类似 flex: 1 来就行了。

这里可以给 grid line 命名，方便取到对应的 line 对 item 进行设置，详情可以在下面 item 属性上了解。

```css
.container {
  display: grid;
  width: 500px;
  height: 500px;
  grid-template-columns: [column-1] 1fr [column-2] 1fr [column-3] 1fr [column-4];
  grid-template-rows: [row-1] 1fr [row-2] 1fr [row-3] 1fr [row-4];
}
/** 通过 repeat 方法可以简写如下
* 此处需要注意的就是线条命名变成了从1开始递增，并且是用空格分隔字母和序号，类似: row 1、row 2、row 3
**/
.container {
  display: grid;
  width: 500px;
  height: 500px;
  grid-template-columns: repeat(3, 1fr column);
  grid-template-rows: repeat(3, 1fr row);
}
```

![1](https://user-gold-cdn.xitu.io/2019/2/16/168f59e8cbdec6fc?w=422&h=415&f=png&s=24273)

#### 2.1.2 grid-template-areas

grid 模板区域：通过给网格区域指定名称，方便项目属性 grid-area 取用网格区域，句法如下：
grid-template-areas: none | grid-area-name | .(半角句号);

none: 为定义的网格区域;

grid-area-name: 定义的网格区域名称，项目属性 grid-area 使用;
.: 空的网格单元格;

```css
.container {
  display: grid;
  width: 500px;
  height: 500px;
  grid-template-columns: repeat(3, 1fr column);
  grid-template-rows: repeat(3, 1fr row);
  grid-template-areas:
    "header header header"
    "main . sidebar"
    "footer footer footer";
}
.container>div {
  border: 1px solid lightcoral;
}
// 项目
.item1 {
  grid-area: header;
}
.item2 {
  grid-area: sidebar;
}
.item3 {
  grid-area: main;
}
.item4 {
  grid-area: footer;
}
// html
<div class="container">
  <div class="item1">1</div>
  <div class="item2">2</div>
  <div class="item3">3</div>
  <div class="item4">4</div>
</div>
```

最终效果图如下：

![1](https://user-gold-cdn.xitu.io/2019/2/16/168f59ee70cb9542?w=553&h=546&f=png&s=22067)

注意：当使用了 grid-template-areas 对网格命名后，网格的行、列的开始线与结束线会自动获得一个 name-start、name-end 的命名，所以也会出现同一条线拥有多个名称的情况，例如 header-start, main-start, and footer-start。

```css
.item1 {
  grid-area: header;
}
// 以上代码效果等同于
.item1 {
  grid-column-start: header-start;
  grid-column-end: header-end;
}
```

#### 2.1.3 grid-template

终于到这个复合属性了，该属性类似 background、font、border 等，是对以上 grid-template 系列属性的一个集合。

首先来看一下基础语法：
grid-template: none | grid-template-rows / grid-template-columns;

none: 将所有属性重置为初始值;

grid-template-rows / grid-template-columns: 将 grid-template-columns 和 grid-template-rows 分别设置为指定值，并将 grid-template-areas 设置为none;

看到这可能有人会有疑问，说好的全部属性呢，怎么没有看到 grid-template-area？实话说，句法有点复杂，下面单独拎出来看看。

```css
// 使用 grid-template 综合之前的所有属性
grid-template:
  [row1-start] "header header header" 1fr [row1-end]
  [row2-start] "main . sidebar" 1fr [row2-end]
  [row2-start] "footer footer footer" 1fr [row2-end]
  / [column-start] 1fr 1fr 1fr [column-end];

// -------------分割线-------------
.item1 {
  grid-column-start: header-start;
  grid-column-end: header-end;
}
// 以上代码效果等同于
.item1 {
  grid-column-start: column-start;
  grid-column-end: column-end;
}
```

### 2.2 grid-gap 系列

这个系列比较简单，就是定义行与行，列与列之间的间隙，就直接用 grid-gap 来介绍了。

grid-gap: grid-row-gap grid-column-gap;
如果只写一个值那就是行、列间隙一致。

**注意**:  在 Chrome 68+, Safari 11.2 Release 50+ and Opera 54+ 以上版本中，可以去掉 grid 前缀，而只需要些 row-gap、column-gap、gap 了。

```css
.containner {
  // ... 之前的代码
  grid-gap: 10px;
}
```

为了区分，用红色框出来原来的布局，虚线间的间隔就是设置的 10px 的 gap。
示意图如下:

![1](https://user-gold-cdn.xitu.io/2019/2/16/168f59f3226e504d?w=541&h=542&f=png&s=39195)

### 2.3 place-items 与 place-content 系列

place-items: justify-items align-items;

place-content: justify-content align-content;

这两个系列放在一起介绍，因为很多地方很相似，异曲同工。

**justify-items**： 网格项水平的对齐方式，默认是 stretch，其余的值还有 start、end、center。

stretch:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a130a332540?w=579&h=587&f=png&s=9579)

start:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a144a4b912f?w=530&h=526&f=png&s=8401)

end:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a15089cb3b0?w=575&h=571&f=png&s=9447)

center:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a15bfe5266a?w=565&h=572&f=png&s=8813)

**align-items**：网格项垂直的对齐方式，整体如同 justify-items，不过水平换成了垂直。

```css
// 下面两个属性需要我们调整一下之前的 css 代码，由于 place 系列需要某些特殊情况（就是 item 组成的网格大小小于容器）
.box {
  margin: 200px auto;
  display: grid;
  width: 500px;
  height: 500px;
  grid-template: 100px 100px / 100px 100px;
  gap: 10px;
  justify-content: start;
  border: 1px solid #ccc;
}

.box>div {
  border: 1px solid lightcoral;
}

.item1 {
  grid-column: 1;
  grid-row: 1;
}

.item2 {
  grid-column: 2;
  grid-row: 1;
}

.item3 {
  grid-column: 1;
  grid-row: 2;
}

.item4 {
  grid-column: 2;
  grid-row: 2;
}
```

**justify-content**：网格整体相对于容器的水平对齐方式，默认值是 start，其余的值还有 end、center、stretch、space-around、space-between、space-evenly。

start:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a1ae9aa8975?w=613&h=582&f=png&s=10023)

end:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a1b8abc3014?w=580&h=556&f=png&s=9333)

center:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a1c31489841?w=571&h=558&f=png&s=8954)

stretch: 在这个例子中与 start 相同。

space-around:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a1ea590a7e7?w=574&h=557&f=png&s=8825)

space-between:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a208b10c613?w=571&h=559&f=png&s=9039)

space-evenly:
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a214efc47d4?w=573&h=560&f=png&s=9330)

**align-content**：就是与 justify-content 相反的操作了。

### 2.4 grid 系列

**grid-auto-columns** 与 **grid-auto-rows** 可以用来定义我们超出网格格数的隐式网格的大小。

```css
.box {
  grid-auto-columns: 100px;
  grid-auto-rows: 100px;
}
```

以上如果没有定义 grid-template 之类的属性的话，会根据 item 来生成 100 * 100 px 的网格。

```css
.item4 {
  grid-column: 5 / 6;
  grid-row: 2 / 3;
}
```

如果有一个这样 item 定义超出了网格数，那么它的大小就是 grid-auto 设置的 100 * 100 px，这里要只能通过 grid-auto-columns 调整自动创建的 item 的宽度，高度不可以。

**grid-auto-flow**：定义了 item 的自流向方向，举个例子。

```css
// html
<div class="box">
  <div class="item1">1</div>
  <div class="item2">2</div>
  <div class="item3">3</div>
  <div class="item4">4</div>
</div>

// css
.box {
  margin: 200px auto;
  display: grid;
  width: 500px;
  height: 500px;
  grid-template: 1fr 1fr 1fr 1fr / 1fr 1fr;
  grid-auto-flow: row;
  gap: 10px;
  justify-content: start;
}

.box>div {
  border: 1px solid lightcoral;
}

.item1 {
  grid-column: 1 / 3;
  grid-row: 1;
}

.item4 {
  grid-column: 1 / 3;
  grid-row: 4;
}
```

row 的情况：
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a29a9f8ac39?w=598&h=579&f=png&s=9647)

column 的情况：
![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a2a661687c0?w=588&h=562&f=png&s=9543)

**gird**：是以下属性的缩写形式——grid-template-rows, grid-template-columns, grid-template-areas, grid-auto-rows, grid-auto-columns, 以及 grid-auto-flow。但是要注意的是，你只能同时声明显示属性或者隐式属性。

```css
/**
* 隐式属性部分
**/
grid: auto-flow / 1fr 1fr 1fr;
// 等同于
grid-auto-flow: row;
grid-template-columns: 1fr 1fr 1fr;

grid: auto-flow dense 100px / 1fr 1fr 1fr;
// 等同于
grid-auto-flow: row dense;
grid-auto-rows: 100px;
grid-template-columns: 1fr 1fr 1fr;

/**
* 显示属性部分
**/
gird: 1fr 1fr 1fr / 1fr 1fr 1fr;
// 等同于
grid-template-rows: 1fr 1fr 1fr;
grid-template-columns: 1fr 1fr 1fr;

/**
* 复杂属性
**/
grid: [row1-start] "header header header" 1fr [row1-end]
      [row2-start] "main . sidebar" 1fr [row2-end]
      [row3-start] "footer footer footer" 1fr [row3-end] / 1fr 1fr 1fr;
// 等同于
grid-template-areas:
  "header header header"
  "footer footer footer";
grid-template-rows: [row1-start] 1fr [row1-end row2-start] 1fr [row2-end row3-start] 1fr [row3-end];
grid-template-columns: 1fr 1fr 1fr;
```

## 3. 项目属性

这里有一个要注意的地方，flaot、display（inline-table、contents、 table 除外）、vertical-align 以及 column-* 对 grid item 项目无效。

### 3.1 grid-column、grid-row

旗下有 4 个属性：grid-column-start、grid-column-end、grid-row-start、grid-row-end，用来确定 item 的位置。

属性的值有：number | lineName | span &lt;number&gt; | span &lt;lineName&gt; | auto

对属性的值以 9 宫格为例介绍。

number：从左往右从上往下分别是 1、2、3、4 四条。

lineName：在容器上定义的分隔线的名称。

span &lt;number&gt; | span &lt;lineName&gt;：跨度的个数和跨到某条分隔线。

auto：自动跨度或默认 1 个跨度。

```css
// 将某个 item 定位到 第 2 行第 3 列
// number
.item {
  grid-column: 3 / 4;
  grid-row: 2 / 3;
}
// lineName
.container {
  // ...
  grid-template-columns: [column-1] 1fr [column-2] 1fr [column-3] 1fr [column-4];
  grid-template-rows: [row-1] 1fr [row-2] 1fr [row-3] 1fr [row-4];
}
.item {
  grid-column:column-3 / column-4;
  grid-row: row-2 / row-3;
}
// span <number> | span <name>
.item {
  grid-column:column-3 / span 1;
  grid-row: row-2 / span row-3;
}
```

### 3.2 grid-area

通过容器 grid-template-areas 设置的名称设置 item 位置，也可以当做是 grid-row-start + grid-column-start + grid-row-end + grid-column-end 的缩写。

```css
.container {
  // ...
  grid: [row1-start] "header header header" 1fr [row1-end]
        [row2-start] "main . sidebar" 1fr [row2-end]
        [row3-start] "footer footer footer" 1fr [row3-end] / 1fr 1fr 1fr;
}
.item-1 {
  grid-area: header;
}
// 另一种写法
.item {
  grid-area: row1-start / 1 / row1-end / 4;
}
```

![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a2c5ce9f612?w=582&h=485&f=png&s=8214)

### 3.3 place-self

旗下有 justify-self、align-self 两个属性：沿着内联（行/列）轴对齐单元格内的网格项，主要的值有：start | end | center | stretch。

place-self: align-self justify-self;

```css
.item-1 {
  place-self: center stretch;
}
```

![1](https://user-gold-cdn.xitu.io/2019/2/16/168f5a2df7c52217?w=591&h=417&f=png&s=7178)

## 补充

**显式网格**：通过使用 grid-template-columns 和 grid-template-rows 属性显式设置的一个网格的列和行。

**隐式网格**：grid-auto-flow 默认流，grid-auto-flow 属性可以改变默认流方向 row 为 column。

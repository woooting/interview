# CSS 三大特性

## 层叠（Cascade）

多个规则冲突时，按以下顺序裁决（从上到下，高的赢）：

1. **!important** — 打破所有常规优先级
2. **样式来源** — 开发者样式 > 用户样式 > 浏览器默认样式
3. **选择器优先级** — 内联 > ID > 类/属性 > 元素
4. **出现顺序** — 后面覆盖前面

层叠只裁决**直接命中**元素的规则。如果层叠后某属性没有值，才走继承或默认值。

## 优先级（Specificity）

四位权重，逐位比较，高位优先：

| 级别 | 权重 | 示例 |
|------|------|------|
| 内联样式 | 1,0,0,0 | `style="color:red"` |
| ID 选择器 | 0,1,0,0 | `#header` |
| 类 / 属性 / 伪类 | 0,0,1,0 | `.box` / `[type]` / `:hover` |
| 元素 / 伪元素 | 0,0,0,1 | `div` / `::before` |
| 通配符 / 组合符 | 0,0,0,0 | `*` / `>` / `+` |

`!important` 独立于这体系，直接提到最高。两个都加 `!important` 则回头比这四位。

## 继承（Inheritance）

**和文字排版相关的属性可继承，盒模型/布局相关的不可继承。**

- 可继承：`color`、`font-*`、`line-height`、`text-align`、`text-indent`、`visibility`、`cursor`、`list-style` 等
- 不可继承：`width/height`、`margin/padding/border`、`background`、`opacity`、`position`、`display`、`flex-*`、`grid-*`、`overflow` 等

**子元素有自己的值时 → 用自己值（层叠已决出），不会走继承。** 只有子元素没有直接命中的规则时，才往上找父元素继承。

| 关键字 | 效果 |
|--------|------|
| `inherit` | 强制继承父元素计算值 |
| `initial` | 恢复浏览器默认值 |
| `unset` | 可继承属性 = `inherit`，不可继承 = `initial` |

---

# CSS 属性继承

## 可继承的属性

### 文本相关
- `color`、`font-*`（font-size, font-family, font-weight 等）
- `line-height`、`text-align`、`text-indent`、`text-decoration`
- `letter-spacing`、`word-spacing`、`direction`、`unicode-bidi`
- `visibility`、`cursor`

### 列表相关
- `list-style`、`list-style-type`、`list-style-position`、`list-style-image`

### 表格相关
- `border-collapse`、`border-spacing`、`caption-side`、`empty-cells`

### 其他
- `quotes`、`content`、`tab-size`

## 不可继承的属性

大部分属性都不可继承，主要包括：
- **盒模型：** `width`、`height`、`margin`、`padding`、`border`
- **背景：** `background`、`opacity`
- **定位：** `position`、`top`、`left`、`z-index`
- **布局：** `display`、`float`、`overflow`、`flex-*`、`grid-*`
- **轮廓：** `outline`
- **盒阴影：** `box-shadow`

## 强制继承技巧

```css
/* 让子元素继承父元素的颜色 */
.parent {
  color: red;
}
.child {
  color: inherit; /* 或者 currentColor */
}
```

> `inherit` 关键字可以强制继承父元素的值，`initial` 恢复默认值，`unset` 恢复继承或默认

---

# CSS Display 属性

## Display 属性值一览

| 属性值         | 作用描述                         | 示例代码                                                     | 典型场景             |
| -------------- | -------------------------------- | ------------------------------------------------------------ | -------------------- |
| `none`         | 完全隐藏元素，不占空间           | `.hidden { display: none; }`                                 | 动态显示 / 隐藏元素  |
| `block`        | 块级元素，独占一行，支持宽高     | `.block { display: block; width: 200px; }`                   | 容器、段落、标题     |
| `inline`       | 行内元素，不换行，宽高由内容决定 | `.inline { display: inline; }`                               | 文本、链接、图片     |
| `inline-block` | 不换行，但支持宽高               | `.btn { display: inline-block; width: 100px; }`              | 水平按钮组、图标     |
| `table`        | 模拟表格布局                     | `.table { display: table; }`                                 | 数据表格、等高布局   |
| `table-cell`   | 模拟表格单元格，支持垂直居中     | `.cell { display: table-cell; vertical-align: middle; }`     | 表单对齐、垂直居中   |
| `flex`         | 弹性布局，一维排列               | `.flex { display: flex; justify-content: center; }`          | 导航栏、自适应卡片   |
| `grid`         | 网格布局，二维排列               | `.grid { display: grid; grid-template-columns: repeat(3, 1fr); }` | 图片画廊、响应式网格 |

### 各属性详解

1. **none：**
   - 设置元素为 `display:none`，会导致元素不可见，但不会消失在DOM树中
   - 设置为 `none`时，会导致浏览器进行 **reflow/repaint**，因为他导致DOM布局结构更新触发重排，重排必定触发重绘

2. **block：** 块级元素的默认值，宽度默认为父元素的 100%，支持设置宽高

3. **inline：** 行内元素，不换行，宽度由内容决定，无法设置宽高
   - 垂直方向的 `margin` 和 `padding` 无效，水平方向有效

4. **inline-block：** 行内块元素，不换行，但支持设置宽高，兼具 `inline` 和 `block` 的特性

5. **table：** 将元素渲染为块级表格，需配合子元素的 `table-row` 和 `table-cell`

6. **flex：** 将元素转换为弹性容器，子元素成为弹性项目，支持灵活的对齐和分布

7. **inline-flex：** 弹性容器，但表现为内联元素，不独占一行
   - 外部表现为行内元素，不占满宽度
   - 不换行，和其他元素在用一行
   - 宽度由内容撑开，属于inline特性
   - 可以使用flex弹性盒的相关特性，例如 `flex-direction`、`align-items`、`justify-content`

8. **grid：** 将元素转换为网格容器，子元素成为网格项目，支持二维布局

9. **inline-grid：** 和 **inline-flex** 一样，外部表现为inline元素的特性，内部可以使用grid布局属性

## Block、Inline、Inline-Block 对比

| 特性             | block             | inline             | inline-block      |
| ---------------- | ----------------- | ------------------ | ----------------- |
| 是否换行         | 是                | 否                 | 否                |
| 设置width/height | 有效              | 无效               | 有效              |
| margin/padding   | 全部有效          | 水平有效，垂直特殊 | 全部有效          |
| 默认宽度         | 父元素100%        | 内容宽度           | 内容宽度          |
| 元素排列         | 垂直排列          | 水平排列           | 水平排列          |
| 包含关系         | 可包含其他块/行内 | 通常只含文本/行内  | 可包含其他块/行内 |

---

# 元素隐藏方式

## 1. display: none

- 元素不会出现在渲染树中
- 不占据任何文档流空间
- 所有子元素都会被连带隐藏
- 无法触发任何DOM事件
- 屏幕阅读器完全忽略该元素
- 触发浏览器重排(reflow)

**性能影响：** 高，会触发repaint和reflow

**使用场景：**
- 需要完全移除不需要的元素
- 标签页切换内容区域
- 响应式布局中在不同断点隐藏元素

## 2. visibility: hidden

- 元素不可见但保留原有空间
- 无法触发鼠标等交互事件
- 只导致重绘(repaint)，性能较好
- 可通过 `visibility: visible` 显示子元素
- 屏幕阅读器无法访问

**性能影响：** 中等，只触发repaint

**使用场景：**
- 需要保留布局占位的隐藏
- 实现自定义复选框/单选框样式
- 需要保持布局稳定的场景

## 3. opacity: 0

- 元素完全透明但占据空间
- **仍能触发所有DOM事件**
- 会创建新的复合层，适合动画
- 屏幕阅读器可以访问
- 子元素无法单独恢复可见性

**性能影响：** 低，GPU加速（创建图层，由合成层处理，无repaint/reflow）

**使用场景：**
- 需要淡入淡出动画
- 需要隐藏但仍需交互的元素
- 可访问性要求高的内容

## 4. position: absolute

- 视觉上不可见且不占空间
- 仍保留DOM位置和事件绑定
- 屏幕阅读器可以访问
- 不影响页面布局流

**性能影响：** 高，会触发reflow

**使用场景：**
- SEO优化需要隐藏但可抓取的内容
- 可访问性要求高的隐藏内容
- 需要隐藏但保留表单元素值

## 5. z-index

- 元素被其他层叠元素遮盖
- 仍占据原有文档流空间
- 事件触发取决于遮盖元素
- 屏幕阅读器可以访问
- 开启层叠上下文才可使用，如 `position:absolute`、`display:flex/grid`

**性能影响：** 低，复合层处理

**使用场景：**
- 背景元素隐藏
- 特殊视觉效果实现
- 需要保留元素但置于底层的场景

## 6. clip / clip-path

- 视觉隐藏但保留元素空间
- 不响应交互事件
- 屏幕阅读器行为不一致
- 支持平滑动画过渡

**性能影响：** 中，复合层处理

**使用场景：**
- 创意动画效果
- 渐进式内容展示
- 需要保留元素尺寸的隐藏

## 7. transform: scale(0)

- 元素视觉尺寸为零但保留布局空间
- 不响应交互事件（元素宽高被设置成0x0，用户无法点击）
- 保持元素原本的盒模型特性
- 屏幕阅读器可以访问
- 支持平滑的缩放动画

**性能影响：** 低，复合层处理，不触发reflow和repaint

**使用场景：**
- 需要缩放动画的元素
- 需要保留布局空间的隐藏
- 特殊交互效果的实现

---

# 文本溢出隐藏

## 单行文本溢出

```css
.single-line {
  white-space: nowrap;      /* 强制不换行 */
  overflow: hidden;         /* 溢出内容隐藏 */
  text-overflow: ellipsis;  /* 显示省略号 */
  width: 200px;             /* 必须设置宽度 */
}

white-space:nowarp
overflow:hidden
text-flow:ellipsis
width:200px

```

## 多行文本溢出

```css
.multi-line {
  display: -webkit-box;         /* 必须 */
  -webkit-box-orient: vertical; /* 必须 */
  -webkit-line-clamp: 2;        /* 显示的行数 */
  overflow: hidden;             /* 溢出隐藏 */
  text-overflow: ellipsis;      /* 省略号（可选） */
  width: 200px;                 /* 容器宽度 */
}


whit-space:nowrap
overflow:hidden
text-overflow:ellipsis
width:200px
-webkit-box-orient:vertical -webkit-box-orient；vertical
-webkit-line-clamp:2
dispaly:-webkit-box


display:-webkit-box
-webkit-box-orient:vertical
-webkit-line-clamp:2

whit-space:nowrap
overflow:hidden
text-overflow:ellipsis
width:200px

```

---

# Flex 布局

## 容器属性（六大属性）

- **`flex-direction`**：设置主轴方向，取值为 `row`（默认，水平从左到右）、`row-reverse`（水平从右到左）、`column`（垂直从上到下）、`column-reverse`（垂直从下到上）

- **`flex-wrap`**：控制弹性项是否换行，取值为 `nowrap`（默认，不换行，可能溢出）、`wrap`（换行，第一行在上）、`wrap-reverse`（换行，第一行在下）

- **`flex-flow`**：`flex-direction` 和 `flex-wrap` 的简写，默认值为 `row nowrap`

- **`justify-content`**：设置弹性项在主轴上的对齐方式，常用值有 `flex-start`（默认，靠左/上）、`flex-end`（靠右/下）、`center`（居中）、`space-between`（两端对齐，中间等距）、`space-around`（元素两侧间距相等）

- **`align-items`**：设置弹性项在交叉轴（与主轴垂直）上的对齐方式，常用值有 `stretch`（默认，拉伸填满容器）、`flex-start`（靠交叉轴起点）、`flex-end`（靠交叉轴终点）、`center`（交叉轴居中）、`baseline`（基线对齐）

- **`align-content`**：当弹性项换行后（多行），控制多行在交叉轴上的对齐方式，取值与 `justify-content` 类似，默认 `stretch`（拉伸填满交叉轴）

## 主轴和交叉轴

可以理解为 **x/y轴** 之间的关系，你可以通过 `flex-direction` 属性改变主轴，`row` 就是水平方向，`column` 就是垂直方向

## Flex 特性

- `display:flex` 开启后，可以应用z-index属性（新开了一个层叠上下文）
- 子元素不再局限于 `inline/inline-block`，行内元素可以设置宽高，不会默认换行，块级元素不再默认占满一行，margin合并特性失效
- 通过设置子项的 `flex` 属性可以实现强大的响应式设计，`flex` 按顺序对应三个值：`flex-grow`、`flex-shrink`、`flex-basis`，这三个分别控制子项的扩展、收缩、默认尺寸的行为
  - `flex: 1` === `flex: 1 1 0%`
  - `flex: none` === `flex: 0 0 auto`
  - `flex: auto` === `flex: 1 1 auto`
  - 默认值：`flex: 0 1 auto`

---

# Float 浮动

**用途：** 图片文字环绕，类似报纸那种感觉

## 核心特性

- 设置float的元素，其浮动特性只在该父元素内生效，不在页面生效，不同于z-index和position
- 在float容器中，没有设置float的元素会避开float项所在的空间，寻找剩余空间展示自身文本，这里需要注意，是文本展示的空间会避开，不是整个元素的height和width
- 设置浮动：`float: right/left`，float元素会脱离文档流
- 父容器要触发BFC，设置 `overflow: auto/hidden`

## 注意事项

这里需要注意dom顺序，如果box2在box1后边，那么浮动 `float:left` 就会失效，因为dom渲染是按顺序来的，容器中如果先渲染box1，就会占据父容器所有宽高，导致box2浮动失败

### HTML

```html
<main class="container">
  <div class="box2"></div>
  <div class="box1">
    这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本
    这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本这是一串文本
    这是一串文本这是一串文本这是一串文本这是一串文本
  </div>
</main>
```

### CSS

```css
.box1 {
  height: 100%;
  background-color: black;
  color: white;
}

.box2 {
  width: 40%;
  background: red;
  height: 80%;
  float: left;
}

.container {
  overflow: hidden;
  background-color: blue;
  height: 200px;
}

---

# CSS 布局单位

## px（像素）

- 最常用的单位，相对于显示器屏幕的一个物理像素点
- 固定大小，不随父元素或视口变化
- 适合用于边框、阴影等需要精确控制的场景

## %（百分比）

- 相对于父元素的对应属性值
- `width: 50%` 表示父元素宽度的一半
- 某些属性（如 `margin`, `padding`）的百分比是相对于父元素的**宽度**（而非高度）

## em

- 相对于**当前元素**的 `font-size`
- `1em` = 当前元素的字体大小
- 嵌套使用时会产生**累积效应**（父元素设 `1.5em`，子元素再设 `1.5em` 实际为 `2.25em`）
- 适合用于按钮、卡片等需要等比缩放的组件

## rem

- 相对于**根元素（html）** 的 `font-size`
- `1rem` = `html` 元素的字体大小（默认为 `16px`）
- 不会像 `em` 那样逐层累积，行为更可预测
- 适合用于全局统一缩放、响应式布局

## vw / vh

- `1vw` = 视口宽度的 1%，`100vw` = 视口总宽度
- `1vh` = 视口高度的 1%，`100vh` = 视口总高度
- 适合用于全屏展示、响应式字体大小、与视口相关的布局
- 移动端注意滚动条会占用视口空间，可能导致 `100vw` 出现横向滚动条
```

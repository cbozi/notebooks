HTML & CSS

# HTML 

## HTML语义化
语义化就是让页面的内容结构化，编写html代码时选择合适的带有语义的标签
为什么要语义化
- 搜索引擎的爬虫依赖于标记来确定上下文和各个关键字的权重，语义化有助于爬虫抓取更多有效信息，有利于搜索引擎优化SEO
- 没有CSS的情况下也能以一种文档格式显示，容易阅读
- 阅读源代码的人更容易理解，有利于团队开发和维护。

## meta viewport
<meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">

## <script>加载
- `<script>` 加载并执行前阻塞页面的渲染
- `<script async>` 异步加载，加载后执行
- `<script defer>` 延迟运行，脚本的执行推迟到html渲染完毕后，保证顺序
- `<script type=module>` ES6 module
- `<script nomodule>` 支持ES6 module 的浏览器不执行

# CSS

## 选择器的优先级
CSS的属性按按一下三种因素的优先级应用（前一种优先于后一种）
1. 重要性
!important
2. 专用性
专用性的值，四列值：
千位：style属性中的声明
百位：ID选择器，每有一个加1分
十位：类选择器（属性选择器 和 伪类），每有一个加1分
个位：元素选择器（和伪元素），每有一个加1分

## 垂直水平居中

## 重绘和重排
- 重排reflow， 部分DOM树需要重新分析并重新计算节点布局
- 重绘repaint，节点样式或一些属性发生改变
- 情景
  - visibility: hidden 只触发重绘
  - display:none - 触发重排和重绘
  - 添加、删除、修改DOM节点
  - 调整样式
  - 添加动画
  - Resize, scroll

## 清除浮动
- overflow:hidden BFC
- .clearfix 
```
.clearfix::after{
     content: ''; display: block; clear:both;
 }
 .clearfix{
     zoom: 1; /* IE 兼容 */
 }
```

## BFC
块格式化上下文Bloack Formmating Context
### 创建BFC
- 根元素<html>
- 浮动
- 绝对定位 position：absolute/fixed
- overflow 不为visibiel
- 行内块元素 inline-block inline-table
- table-cell
- table-caption
- display: flow-root
- display: flex
- display: grid
- 
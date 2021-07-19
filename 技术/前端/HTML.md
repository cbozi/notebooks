HTML

块级元素block和内联元素inline

块级元素block：

- 其之前和之后的元素会在新的一行展现，块级元素的左右两边不允许有其他的元素

- 常见的块级元素有:<p> ,<h1~h6> ,<div>, lists and list items(<ol>,<ul>,<li>), <form>

内联元素inline：

- 不会导致文本换行

- 遵守left & right margins, not top & bottom

- 不能设置高度height和宽度width

- 常见的内联元素有:<span>, <image>，anchors<a>



注释

在<!-- 和 -->之间



实体引用charactor entity references

在html中，字符<,>,",',&等是特殊字符，如果要在网页的内容文本中使用这些字符，为了不让它们被浏览器作为html代码解释，需要使用字符引用。每个字符引用以&开始，以;结束

常用的有

```
原来格式为表格（table），转换较复杂，未转换，需要手动复制一下
{"cells":[{"textAlign":"center","value":"<"},{"textAlign":"center","value":"&lt;"},{"textAlign":"center","value":">"},{"textAlign":"center","value":"&gt;"},{"textAlign":"center","value":"\""},{"textAlign":"center","value":"&quot;","inlineStyles":{"font-family":[{"from":0,"to":6,"value":"Tahoma"}]}},{"textAlign":"center","value":"'","inlineStyles":{"font-family":[{"from":0,"to":1,"value":"Tahoma"}]}},{"textAlign":"center","verticalAlign":"top","value":"&apos;"},{"textAlign":"center","value":"&"},{"textAlign":"center","value":"&amp"},{"textAlign":"center","value":"空格no-break space","inlineStyles":{"color":[{"from":2,"to":16,"value":"#222222"}],"back-color":[{"from":2,"to":16,"value":"#f8f9fa"}]}},{"textAlign":"center","value":"&nbsp"}],"heights":[40,40,40,40,40,39],"widths":[310,310]}
```



<head>标签里的内容

<head>标签里的内容不会再浏览器中显示，它的作用是包含一些页面的元数据。

<head>中的内容包括：

1. 标题 <title>，同时也是默认的书签名

1. 元数据<meta>元素，字符编码如<meta charset="utf-8">；<meta>元素有name和content属性，可以用来添加作者author和页面内容描述description，如

1. 自定义图标 <link rel="shortcut icon" href="favicon.ico" type="image/x-icon">

1. css <link>元素通常位于文档的头部 <link rel="stylesheet" href="my-css-file.css">

1. javascript <script>元素，非必需放在文档头部 <script src="my-js-file.js"></script>



HTML语义化

语义化就是让页面的内容结构化，编写html代码时选择合适的带有语义的标签

为什么要语义化

- 搜索引擎的爬虫依赖于标记来确定上下文和各个关键字的权重，语义化有助于爬虫抓取更多有效信息，有利于搜索引擎优化SEO

- 没有CSS的情况下也能以一种文档格式显示，容易阅读

- 阅读源代码的人更容易理解，有利于团队开发和维护。

常见标签

- 描述列表<dd><dt>

- <em> emphsis表现为斜体，表示强调的文本；<i>italic同样是斜体，通常表示因为某种原因和正常文本不同的文本，例如专业术语、外语短语和排版用的文字。

- <strong>表现为加粗，表示重要的文本；<b>bold同样是加粗，表示文本风格不同于正常的文本，没有表达任何特殊的重要性和相关性。

- 引用，

块引用<blockquote cite=“http://example.com">，cite属性为引用来源

行内引用<q cite="http://example.com”>, 浏览器默认将其内容用双引号包围。

<cite>元素通常放到引用元素旁边，表示引用资源的名称，cite标签内的文本默认为斜体。

- 缩略语abbreviation<abb>, 用来包裹缩略语，title属性提供解释，如<abbr title="Doctor of Philosophy">PhD</abbr>

- 联系方式<address>

- 上标superscription<sup>, 下标subscription<sup>

- 计算机代码<code> <pre> <var> <kbd> <samp>

- <time>元素用于标记时间和日期 time元素的datetime属性提供可被机器识别的时间/日期，

<time datetime="2016-01-20T19:30">7.30pm, 20 January 2016</time>

用于结构化网站的标签

- 标题<header>

- 导航栏<nav>

- 主要内容<main>  代表性的内容段落主题可以使用<article> <section>

- 侧栏<aside> 经常嵌套在<main>中

- 页脚<footer>







超链接

使用<a>元素

href属性为链接指向的网址

title属性为连接的补充信息，当鼠标悬停在链接上时会提示title属性的内容

download属性只是浏览器下载URL而不是转到URL，如果download属性有值，则这个值为默认的文件名。

将图片或其他内容转换为超链接，只需要把图像放到<a></a>标签中间

统一资源定位器URL(Uniform Resource Locator)

文档片段

超链接可以连接到html文档的特定部分， 在URL的结尾加上#id，点击链接将跳到id属性对应的元素的位置

mailto



列表

<ol>

<li>

<dl>

有序列表计数： start属性； reversed属性；<li>中的value属性



图片

<img src="" alt="" width="" hegiht="" title="">

<figure>和<figcaption>



视频和音频

<video>标签

属性：

- src

- controls

- width, height

- autoplay

- loop

- muted

- poster 视频播放前显示的图像

- preload, 缓冲，”none"不缓冲；”auto“页面加载后缓存媒体文件；”metadata"仅缓冲文件的元数据

- <video>标签内的内容：后备内容，若浏览器不支持<video>标签，它将会显示出来

添加多个文件源：<source>标签

<video controls width="400" height="400"
       autoplay loop muted
       poster="poster.png">
  <source src="rabbit320.mp4" type="video/mp4">
  <source src="rabbit320.webm" type="video/webm">
  <p>Your browser doesn't support HTML5 video. Here is a <a href="rabbit320.mp4">link to the video</a> instead.</p>
</video>



<audio>标签

类似<video> 添加音频

<track>标签

添加字幕



9<tables>

- <tfoot>不管放在哪都显示在最下方

- <colgroup>

- <caption>不管放在哪都显示在上方

- <th>的scope属性：row, col, rowgroup, colgroup

- 



表单

<form>标签 action属性定义了提交表单时发送数据的地址 method属性定义了发送数据的HTTP方法 如get或post

<label>标签 for属性为对应的表单输入控件的id

<input>标签  https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/input

<input> name属性：提交时的字段名 id属性与label中的for属性联系

<input type="text> 和 <textarea> 区别：单行/多行；input是空元素不需要关闭标签，input默认值用value属性，textarea默认值包含在其中

<button> type：“submit","reset","button"， button标签与<input type="button|submit|reset">外观无区别，但input是空元素，意味着input生成的按钮中只能包含纯文本，而button中能包含HTML内容

基本验证：

input 中的email，password等type

pattern 正则表达式

maxlen minlen

type="number", min, max属性

enctype属性

当 method 属性值为 post 时, enctype 是提交form给服务器的内容的 MIME 类型 。可能的取值有:

- application/x-www-form-urlencoded: 如果属性未指定时的默认值。

- multipart/form-data: 这个值用于一个 type 属性设置为 "file" 的 <input> 元素。

- text/plain (HTML5)


CSS Tricks

单行文本溢出加...如何实现

white-space: nowarp;

overflow: hidden;

text-overflow: ellipsis;



隐藏 & 透明

- opacity: 0;

- visibility: hidden;

- display: none;

- background-color:  rgba(0,0,0,0.2)



inline-block

- 即呈现inline特性（不占据一整行，宽度有内容决定）

- 又呈现block特性（可设置宽高，内外边距）

- 默认以文字底部基线上下对齐，用vertical-align设置，默认baseline



line-height

- line-height: 2 字体大小的2倍

- line-height: 200% 计算出绝对高度之，如字体大小16px，line-height=32px；

- height=line-height 单行文本垂直居中



W3C盒子模型 IE盒子模型

- 标准 heIght和width为content的宽高

- IE盒子模型， width，height包括content padding border

- box-sizing: border-box | content-box;



定位position:

绝对定位absolute 

- position:absolute 的top,bottom,left, right都是相对于祖先定位元素的border，无视padding。http://js.jirengu.com/yepap/1/edit

- 绝对定位元素可以设置margin，且没有margin合并。






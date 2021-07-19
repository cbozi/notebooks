DOM

## Node
Node是一个接口，DOM中的所有结点都继承自Node类型
### 属性
- nodeType,  表示结点的类型  
Node.ELEMENT_NODE(1)  
Node.TEXT_NODE(3)  
.....
- nodeName 元素的标签名
- nodeValue
### 结点关系
- childNodes 所有的子节点，NodeList对象，类数组对象，有length属性，NodeList随着DOM结构动态变化
- firstChild
- lastChild
- parentNode 父节点
- previousSibling 前一个兄弟节点
- nextSibling 后一个兄弟节点
- ownerDocument 指向文档节点
### 操作结点
- appendChild() 向childNodes列表的末尾添加一个节点
- insertBefore() 2个参数：要插入的节点和作为参照的节点，插入的节点会成为参照节点的前一个兄弟节点
- replaceChild() 2个参数要插入的节点和要被替换的节点，被替换的节点和它的所有子节点被替换
- removeChild() 删除节点
- cloneNode() 1个参数，true默认深复制，false浅复制
- normalize() 处理文档节点，文本节点不包含文本或连续出现2个文本节点

## Document类型
### 属性
- nodeType 9
- nodeName #document
- document.documentElement html元素
- document.body body元素
- document.doctype
### 文档信息
- document.title
- document.URL
- document.domain
- document.referrer
### 查找元素
- getElementById() 接受一个参数：要选取的元素的ID
- getElementsByTagName() 接受一个参数：要选取的元素标签名，返回所有此标签的元素组成的HTMLCollection   
- getElementsByClassName() 接受一个参数：要选取的元素的class类名
- getElementsByName() 接受一个参数：要选取的元素的name属性
- querySelector() 接受一个参数：元素的css选择器，返回选择器对应的的第一个元素
- querySelectorAll()  接受一个参数：元素的css选择器，返回选择器对应的所有元素组成的HTMLCollection
 
HTMLCollection继承自NodeList
### 文档写入
- document.write()
- document.writeln()
- document.open()
- document.close()

调用write，open方法文档原来的内容会清空

## Element
- nodeType 1
- nodeName 元素的标签名
### HTML元素
- id
- title
- lang
- dir
- className
### 属性
- getAttribute()
- setAttribute()
- removeAttribute()
### 创建元素
- document.createElement() 标签名
### DocumentFragment
document.createDocumentFragement()
JS 面试

## 跨域
### JSONP
### iframe 加 form 发送post
### CORS (cross origin resource sharing)
  默认支持GET,HEAD,POST
  响应头`Access-Control-Allow-Origin`,`Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`
  预检请求`OPTIONS`
### window.postMesage() 
otherWindow.postMessage(message, targetOrigin, [transfer])
- otherWindow: iframe的contentWindow属性、执行window.open返回的窗口对象、或者是命名过或数值索引的window.frames。
- message
- targetOrigin URI,指定接受窗口的协议、主机地址或端口

接收窗口监听`message`方法
```
window.addEventListener('message', function(event) {
    if (event.orgin !== 'http://example.org:8080') {
        return 
    }
    var data = event.data
    var source = event.source // 发送 消息的窗口对象的引用 
    // ...
})
```

## 安全
### CSRF
CSRF（跨站请求伪造）cross site requeset forgery
伪造用户在已经登录的网站上的操作，冒充用户发起请求
在请求中放入攻击者所不能伪造的信息，并且该信息不存在于Cookie之中。
csrf token

### XSS
XSS 跨站脚本（Cross site scripting）
将恶意代码注入到网页上
防范：使用innerText,不使用innerHTML

## GET 和 POST 的区别是什么？
- 参数。GET 的参数放在 url 的查询参数里，POST - 的参数（数据）放在请求消息体里。
- 安全（扯淡）。GET 没有 POST 安全（都不安全）
- GET 的参数（url查询参数）有长度限制，一般是 1024 个字符。POST 的参数（数据）没有长度限制（扯淡，4~10Mb 限制）
- 包。GET 请求只需要发一个包，POST 请求需要发两个以上包（因为 POST 有消息体）（扯淡，GET 也可以用消息体）
- GET 用来读数据，POST 用来写数据，POST 不幂等（有副作用）（幂等的意思就是不管发多少次请求，结果都一样。）

## Ajax
Asynchronous JavaScript and XML
```
var request = new XMLHttpRequest(); // 新建XMLHttpRequest对象

request.onreadystatechange = function () { // 状态发生变化时，函数被回调
    if (request.readyState === 4) { // 成功完成
        // 判断响应结果:
        if (request.status === 200) {
            // 成功，通过responseText拿到响应的文本:
            return success(request.responseText);
        } else {
            // 失败，根据响应码判断失败原因:
            return fail(request.status);
        }
    } else {
        // HTTP请求还在继续...
    }
}

// 发送请求:
request.open('GET', '/api/categories');
request.send();
```
### xhr.readyState
值 |状态|	描述
--|--|--
0|	UNSENT (初始状态，未打开)|	此时xhr对象被成功构造，open()方法还未被调用
1|	OPENED (已打开，未发送)|	open()方法已被成功调用，send()方法还未被调用。注意：只有xhr处于OPENED状态，才能调用xhr.setRequestHeader()和xhr.send(),否则会报错
2|	HEADERS_RECEIVED (已获取响应头)|	send()方法已经被调用, 响应头和响应状态已经返回
3|	LOADING (正在下载响应体)|	响应体(response entity body)正在下载中，此状态下通过xhr.response可能已经有了响应数据
4|	DONE (整个数据传输过程结束)|	整个数据传输过程结束，不管本次请求是成功还是失败

### 事件触发顺序
1. 触发xhr.onreadystatechange(之后每次readyState变化时，都会触发一次)
1. 触发xhr.onloadstart //上传阶段开始：
1. 触发xhr.upload.onloadstart
1. 触发xhr.upload.onprogress
1. 触发xhr.upload.onload
1. 触发xhr.upload.onloadend //上传结束，下载阶段开始：
1. 触发xhr.onprogress
1. 触发xhr.onload
1. 触发xhr.onloadend


## Cookie
- key value
- expires
- max-age
- domain 
- secure
- httpOnly
- sameSite

## Session

## 正则
\s 空格
\S 
\b 单词边界，空页与字符交界处
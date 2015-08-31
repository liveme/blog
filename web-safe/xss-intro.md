# web前端安全之XSS
## XSS原理简介

XSS（Cross Site Scripting）跨站脚本漏洞是一种将恶意 JavaScript 代码插入到其他Web用户页面里执行以达到攻击目的的漏洞。Web应用软件将用户的交互信息没有作有效验证和过滤就输出给了其他用户，攻击者插入的可执行脚本就可能形成跨站脚本攻击。

* **根本原因**：数据与代码未做分离；
* **危害**：盗取cookie，账户劫持，读取、篡改、添加或删除敏感数据，DDos攻击，布局错乱

## 产生场景

肉眼看得到，看不到的用户输入（如某些Flash XSS需要抓包才可以看到接口）都有可能成为XSS漏洞的切入点。

1. 在用户可控的输入点（例如写评论、发布文章）等，数据入库前没有做验证和相关的安全过滤；
2. 在未检验包含数据的动态内容是否存在恶意代码的情况下，便将其传送给用户；

![xss-example](https://cloud.githubusercontent.com/assets/1295348/9469494/f0c7a24e-4b79-11e5-8a68-36ab3db50a43.png)

## XSS类型

常见的，XSS攻击会被分为反射型XSS（Reflected）、储存型XSS（Stored）和DOM-based XSS。

**反射型XSS**

输入→输出。大部分**通过url传入参数或DOM的某些属性控制页面的输出**，常见出现在GET请求上；


**储存型XSS**

输入→进入数据库→取出数据库→输出。用户产生的带有恶意代码的数据存入数据库后，再取出来使用，导致XSS；所以这种恶意攻击可以在网站任何地方出现，并且如果不及时移除，攻击会一直持续下去；


**DOM-based XSS**

反射与存储型都是服务端接收到用户输入的恶意数据然后嵌入到HTML页面产生的，而我们将不依赖于服务端嵌入的XSS攻击叫做DOM-Based XSS，实质上也是反射型的一种。


## 攻防策略

**原则：永远不要相信用户的输入！**


### HTML escape过滤处理

客户端与服务器端通讯一般通过`Querystring`、`from表单`和`cookie`，这些场景中取到数据，在储存进入数据库以及展示在客户端之前，都应该先对数据进行`escape`或`encode`，最起码需要替换`<`、`>`、`"`、`'`、`&`。

很多前端工程师很喜欢用`innerHTML`或者jQuery里面的`html()`等方法，因为可以快速构建DOM，但却对其背后隐藏的潜在危险毫无所知。通过`innerHTML`产生的脚本是不会执行的，如果你以为`innerHTML`很安全那就错了，请看下面例子：

```javascript
wrap.innerHTML = '<img src=1 onerror=alert(1)>'
```

所以，如果你必须使用`innerHTML`来操作从服务器端获取的数据，请先对数据进行HTML净化。以下是几种净化解决方案：

#### 正则过滤

```javascript
function escapeHtml(str) {
    return String(str)
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/"/g, "&quot;")
        .replace(/'/g, "&#039;")
        .replace(/\//g, "&#x2F;")
}
```
这是网络上比较流行的一个escapeHtml()，这能正常的匹配返回正确的结果。但我们可以发现每执行一次就对传入的字符串扫描了6次，效率比较低。下面是一个来自于
[Mustache模板](https://github.com/janl/mustache.js/blob/master/mustache.js#L60) 的escape：

```javascript
function escapeHtml(str){
    var entityMap = {
        '&': '&amp;',
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&#39;',
        '/': '&#x2F;'
    };
    return String(str).replace(/[&<>"'\/]/g, function(s) {
        return entityMap[s];
    });
}
```

#### createTextNode

`createTextNode`方法用于创建新文本节点，该方法接受一个参数：要插入节点中的文本。于是，我们在展现数据之前，先用`createTextNode`处理一下：

```javascript
document.getElementById("wrap").appendChild(document.createTextNode(string));
```

我们按此思路封装一下：

```javascript
function escapeHtml(str) {
    var div = document.createElement('div');
    div.appendChild(document.createTextNode(str));
    return div.innerHTML;
}
```

这实际上是一个hack，利用`createTextNode`方法自身的特点进行escape；

unescape则利用`nodeValue`:

```javascript
function unescapeHtml(escapedStr) {
    var div = document.createElement('div');
    div.innerHTML = escapedStr;
    var child = div.childNodes[0];
    return child ? child.nodeValue : '';
}
```

此外，我们需要了解的是，`createTextNode`能减轻XSS攻击，但也不是绝对安全的，它依赖于所创建的标签，请看下面例子：

```javascript
var tag = document.createElement('script');
var data = 'alert(1)';
tag.appendChild( document.createTextNode(data) );
document.body.appendChild(tag);
```

stackoverflow上针对次问题有比较多的解决方案→[Escaping HTML strings with jQuery](http://stackoverflow.com/questions/24816/escaping-html-strings-with-jquery)

#### 开源库

以上两种都只是做或简单或粗暴的escape，如果你希望能更全面，那可以使用一些开源库来解决：

**XSS.js**

>XSS是一个根据白名单对用户输入的内容进行过滤，以避免遭受XSS攻击的模块。白名单控制允许的HTML标签及各标签的属性，通过自定义处理函数，可对任意标签及其属性进行处理。提供了node版本和浏览器版本的js。

[JSXSS](http://jsxss.com/zh/index.html)

**OWASP ESAPI**

>ESAPI (OWASP企业安全应用程序接口)是一个免费、开源的、网页应用程序安全控件库，用于验证用户输入的富文本以防御跨站脚本的API。适用于java编写web项目。

[OWASP ESAPI](http://www.owasp.org.cn/owasp-project/ESAPI)

**PHP AntiXSS**

一个不错的PHP库，容易上手。

[PHP AntiXSS](https://github.com/voku/anti-xss)


**AntiXSS Library**

微软出的一个防跨站点脚本库AntiXSS Library



### 谨慎使用

```javascript
element.innerHTML = '';
element.outerHTML = '';
document.write();
document.writeln();
```

### 拒绝内联

一般情况下，过滤掉尖括号、引号（单、双）、“&”符号是可以防止了大部分XSS漏洞，提高了攻击的门槛。但单独针对HTML进行过滤也是不全面的，请看例子：
假设有个搜索页面：

```
http://example.com/search?key=google
```
源码大致如下：

```php
    <?php
    header("Content-Type: text/html;charset=utf-8");
    error_reporting(0);

    $key = escapehtml($_GET["key"]);
    ?>

    <input id="keyword" type="text" value="<?php echo $key;?>" />
    <div id="result"></div>

    <script>
    var keyword = document.getElementById("keyword"); 
    var result = document.getElementById("result");

    result.innerHTML = keyword.value;

    // function filter(){
    //     balalala.....
    //     
    //     console.log(<?php echo $key;?>);
    // }
    </script>
```

我们可以看到程序已经使用函数`escapehtml()`进行了html过滤，对`<`、`>`、`"`、`'`与`&`进行了转码，这样就安全了吗？显然不是，仔细看源码，发现了有动态的Querystring被注释了，改一下参数：

```
http://example.com/search?key=%0a;alert(1);//
```

成功执行`alert(1)`！要点在`%0a`，这是经过`encodeURI('\n')`编码之后的换行符。
由此可见，如果你的JavaScript或CSS是内联的话，代码即使被注释了也不一定安全。关键还是要认识到XSS破坏的本质是JS构造，而不是HTML构造，例如换行、注释（`//`、`/*/`）、引号等。


### 注意base64编码





### 重要的cookie设置HttpOnly Cookie

cookie大家是常用到了，但HttpOnly可能并不常接触。通常我们有以下几种方式给客户端设置或获取cookie：

* HTTP Response Headers中的Set-Cookie Header；
* HTTP Request Headers中的Cookie Header；
* JavaScript对document.cookie进行赋值或取值；

下面是一个标准的Set-Cookie Header：

```
Set-Cookie: key=value; path=path; domain=domain; max-age=max-age-in-seconds; expires=date-in-GMTString-format; secure; httponly
```

设置了`httponly`为`true`之后，浏览器的document对象就无法读取到到cookie信息。所以针对重要的敏感cookie设置`httponly`能有效的防止黑客通过HTTP协议来读取cookie。但黑客也不是傻的，他们可以跳过HTTP协议，在socket层面写抓包程序进行其他的攻击，这个就不在本文讨论的范围了。
另外，Set-Cookie Header属性中还有个`secure`，这个是什么用呢？防止信息在传递过程中被监听捕获，导致信息泄露。当`secure`设置为`true`时，创建的cookie就只能在HTTPS链接中被客户端传递到服务器端进行会话验证；HTTP链接中则不会传递。
PS：除了httponly，还有个HostOnly Cookie，有兴趣的童鞋可自行了解。

### CSP

CSP是Content Security Policy的简称，可译为内容安全策略，由W3C 的 webappsec ＷＧ(work group)制定的，用于减少或避免XSS及其带来的风险。通过定义`Content-Security-Policy`HTTP Header来创建一个可信来源的白名单（脚本、图片、iframe、style、font等可能远程的资源），让浏览器只执行和渲染白名单中的资源。

CSP可以由两种方式指定：HTTP Header和HTML Meta，如果两种同时存在，则会优先采用HTTP Header中定义的规则；
CSP默认特性：

* 禁止内联代码执行，包括`<script>代码块`、内联事件、内联样式；
* 禁用`eval()`、`newFunction()`、`setTimeout([string],...)`、`setInterval([string],..)`

**相关链接**：

* [http://www.w3.org/TR/2015/CR-CSP2-20150219/](http://www.w3.org/TR/2015/CR-CSP2-20150219/) （2015-02-19，CSP2标准正式发布）
* [https://playsecurity.org/rawfile/csp2_test.html](https://playsecurity.org/rawfile/csp2_test.html) （CSP2演示例子）
* [http://netsecurity.51cto.com/art/201404/436296_all.htm](http://netsecurity.51cto.com/art/201404/436296_all.htm)（浏览器安全策略说之内容安全策略CSP）
* [http://www.cnblogs.com/softlover/articles/2723233.html](http://www.cnblogs.com/softlover/articles/2723233.html)（HTML5安全：内容安全策略（CSP）简介）


### 利用对url进行短域名处理，从而隐藏真实的url


** 总结**：程序员也是人，只要是人做的每个场景所需要`escape`的方式不一样，对与XSS漏洞，没有一劳永逸的解决方法，具体场景具体应对。密切关注与服务器端有通讯的地方，多积累经验，谨慎对待通讯。


### 关于escape的误解

很多开发者以为只要对数据进行转义就绝对安全了，然后并非如此，请看下面例子：



## 浏览器中的XSS过滤器

随着XSS逐渐被工程师所注意，很多浏览器厂商对XSS攻击做了防御措施，在浏览器安全机制中过滤XSS。例如：IE8+、chrome、safari、firefox都有针对XSS的安全机制。但每个浏览器版本的过滤机制可能会不一样，在低版本的chrome/safari/firefox中的XSS防御相对薄弱，IE6/IE7甚至没有这方面的安全机制。
所以，如果需要做测试，最好使用低版本浏览器。

## XSS漏洞测试

**基本思路**：寻找可能的输入点，发送恶意输入，看程序是否返回异常结果

**可能的输入点**

* url中的参数
* form表单
* cookie
* http头中可控的部分
* 数据库
* 配置文件
* 其他可以被用户控制输入的地方


## 扩展链接

* [http://html5sec.org/](http://html5sec.org/)（HTML5 Security Cheatsheet）
* [XSS (Cross Site Scripting) Prevention Cheat Sheet](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)（XSS防护检查单）

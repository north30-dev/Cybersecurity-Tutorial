# Java代码审计入门

## 一、代码审计概述

### 1.1 什么是代码审计

代码审计（Code Audit）是通过审查源代码来发现安全漏洞的方法。与黑盒测试不同，代码审计可以直接看到代码逻辑，能够发现更深层次的漏洞。

**代码审计的优势**：
- **深度发现**：可以发现黑盒测试难以发现的漏洞
- **全面覆盖**：可以审计所有代码路径
- **早期发现**：在开发阶段就能发现问题
- **理解原理**：深入理解漏洞成因

### 1.2 审计方法分类

| 方法 | 特点 | 适用场景 | 工具示例 |
|------|------|----------|----------|
| 静态分析（SAST） | 不运行代码，分析源码 | 快速扫描、CI/CD集成 | Semgrep、CodeQL、SonarQube |
| 动态分析（DAST） | 运行时分析 | 发现运行时漏洞 | FindSecBugs、Contrast |
| 交互式分析（IAST） | 结合静态和动态 | 精确漏洞定位 | Checkmarx、Fortify |
| 人工审计 | 人工审查代码 | 复杂逻辑、关键代码 | IDE、代码阅读工具 |

### 1.3 Java代码审计重点

```
Java代码审计关注点：
├── 输入验证
│   ├── 用户输入处理
│   ├── 参数校验
│   └── 数据清洗
├── 数据访问
│   ├── SQL注入
│   ├── NoSQL注入
│   └── LDAP注入
├── 输出编码
│   ├── XSS防护
│   ├── 响应头设置
│   └── 编码函数
├── 认证授权
│   ├── 认证机制
│   ├── 会话管理
│   └── 访问控制
├── 敏感数据
│   ├── 加密存储
│   ├── 传输安全
│   └── 日志脱敏
├── 错误处理
│   ├── 异常捕获
│   ├── 错误信息
│   └── 日志记录
└── 组件安全
    ├── 依赖漏洞
    ├── 配置安全
    └── 第三方库
```

---

## 二、静态分析工具

### 2.1 Semgrep入门

Semgrep是一款快速、开源的静态分析工具，支持自定义规则。

#### 安装配置

```bash
# 使用pip安装
pip install semgrep

# 使用Homebrew安装（macOS）
brew install semgrep

# 验证安装
semgrep --version
```

#### 基本使用

```bash
# 扫描单个文件
semgrep path/to/file.java

# 扫描目录
semgrep path/to/project/

# 使用特定规则集
semgrep --config "p/java" path/to/project/
semgrep --config "p/owasp-top-ten" path/to/project/
semgrep --config "p/security-audit" path/to/project/

# 使用多个规则集
semgrep --config "p/java" --config "p/owasp-top-ten" path/to/project/

# 输出格式
semgrep --json path/to/project/ -o results.json
semgrep --sarif path/to/project/ -o results.sarif

# 严格模式
semgrep --strict --config "p/java" path/to/project/
```

#### 自定义规则示例

**SQL注入检测规则**：

```yaml
# sqli.yaml
rules:
  - id: java-sqli
    patterns:
      - pattern: |
          $STMT.executeQuery($VAR + ...)
      - pattern-not: |
          $STMT.executeQuery("...")
    message: "检测到潜在的SQL注入漏洞"
    severity: ERROR
    languages:
      - java
    metadata:
      category: security
      cwe: "CWE-89: SQL Injection"
      owasp: "A1: Injection"

  - id: java-sqli-preparedStatement
    patterns:
      - pattern: |
          String $QUERY = ... + $USER_INPUT + ...;
          $STMT.executeQuery($QUERY);
    message: "使用字符串拼接构造SQL查询，存在注入风险"
    severity: WARNING
    languages:
      - java
```

**命令注入检测规则**：

```yaml
# command-injection.yaml
rules:
  - id: java-command-injection-runtime
    patterns:
      - pattern: |
          Runtime.getRuntime().exec($VAR + ...)
      - pattern-not: |
          Runtime.getRuntime().exec("...")
    message: "检测到潜在的命令注入漏洞"
    severity: ERROR
    languages:
      - java

  - id: java-command-injection-processbuilder
    patterns:
      - pattern: |
          new ProcessBuilder($VAR, ...)
    message: "ProcessBuilder使用用户输入，检查是否存在命令注入"
    severity: WARNING
    languages:
      - java
```

**XSS检测规则**：

```yaml
# xss.yaml
rules:
  - id: java-xss-response-writer
    patterns:
      - pattern: |
          $WRITER.write($VAR);
      - pattern-not: |
          $WRITER.write("...");
      - metavariable-regex:
          metavariable: $VAR
          regex: ^(request\.|req\.|parameter)
    message: "直接输出用户输入到响应，存在XSS风险"
    severity: WARNING
    languages:
      - java

  - id: java-xss-jsp-expression
    pattern: |
      <%= $VAR %>
    message: "JSP表达式直接输出，检查是否经过HTML编码"
    severity: WARNING
    languages:
      - java
```

**路径遍历检测规则**：

```yaml
# path-traversal.yaml
rules:
  - id: java-path-traversal
    patterns:
      - pattern: |
          new File($VAR)
      - pattern-not: |
          new File("...")
      - pattern-inside: |
          $VAR = $REQ.getParameter(...);
          ...
    message: "使用用户输入构造文件路径，存在路径遍历风险"
    severity: ERROR
    languages:
      - java
```

#### 运行自定义规则

```bash
# 运行自定义规则
semgrep --config sqli.yaml path/to/project/
semgrep --config ./rules/ path/to/project/

# 组合多个规则
semgrep --config sqli.yaml --config xss.yaml path/to/project/
```

### 2.2 CodeQL基础

CodeQL是GitHub开发的代码分析平台，功能强大但学习曲线较陡。

#### 安装配置

```bash
# 下载CodeQL CLI
wget https://github.com/github/codeql-cli-binaries/releases/latest/download/codeql-linux64.zip
unzip codeql-linux64.zip
export PATH="$PATH:$(pwd)/codeql"

# 验证安装
codeql resolve languages
codeql resolve qlpacks
```

#### 创建数据库

```bash
# 为Java项目创建数据库
codeql database create \
  --language=java \
  --source-root=/path/to/project \
  /path/to/database

# 指定构建命令
codeql database create \
  --language=java \
  --command="mvn clean install" \
  --source-root=/path/to/project \
  /path/to/database

# Gradle项目
codeql database create \
  --language=java \
  --command="gradle build" \
  --source-root=/path/to/project \
  /path/to/database
```

#### 运行查询

```bash
# 运行标准查询套件
codeql database analyze \
  /path/to/database \
  java-security-extended \
  --format=csv \
  --output=results.csv

# 运行单个查询
codeql query run \
  --database=/path/to/database \
  /path/to/query.ql

# 运行自定义查询套件
codeql database analyze \
  /path/to/database \
  /path/to/custom-queries \
  --format=sarif-latest \
  --output=results.sarif
```

#### CodeQL查询示例

**SQL注入查询**：

```ql
// SqlInjection.ql
import java
import semmle.code.java.dataflow.TaintTracking
import semmle.code.java.security.SqlInjection

/**
 * 从用户输入到SQL执行的污点追踪配置
 */
class SqlInjectionConfig extends TaintTracking::Configuration {
  SqlInjectionConfig() { this = "SqlInjectionConfig" }
  
  override predicate isSource(DataFlow::Node source) {
    source instanceof RemoteFlowSource
  }
  
  override predicate isSink(DataFlow::Node sink) {
    exists(Expr e |
      sink.asExpr() = e and
      e = any(MethodAccess ma).getAnArgument() and
      ma.getMethod().hasName("executeQuery")
    )
  }
  
  override predicate isSanitizer(DataFlow::Node node) {
    node.getType() instanceof TypeString and
    node.asExpr().(MethodAccess).getMethod().hasName("escapeSql")
  }
}

from SqlInjectionConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "SQL injection from $0", source.getNode()
```

**命令注入查询**：

```ql
// CommandInjection.ql
import java
import semmle.code.java.dataflow.TaintTracking

class CommandInjectionConfig extends TaintTracking::Configuration {
  CommandInjectionConfig() { this = "CommandInjectionConfig" }
  
  override predicate isSource(DataFlow::Node source) {
    source instanceof RemoteFlowSource
  }
  
  override predicate isSink(DataFlow::Node sink) {
    exists(MethodAccess ma |
      ma.getMethod().getDeclaringType().hasQualifiedName("java.lang.Runtime") and
      ma.getMethod().hasName("exec") and
      sink.asExpr() = ma.getAnArgument()
    )
  }
}

from CommandInjectionConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "Command injection from $0", source.getNode()
```

### 2.3 SonarQube 2026

SonarQube是流行的代码质量管理平台，集成安全审计功能。

#### 安装部署

```bash
# Docker部署
docker run -d --name sonarqube \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:latest

# 使用PostgreSQL数据库
docker run -d --name sonarqube-db \
  -e POSTGRES_USER=sonar \
  -e POSTGRES_PASSWORD=sonar \
  -e POSTGRES_DB=sonar \
  postgres:15

docker run -d --name sonarqube \
  -p 9000:9000 \
  -e SONAR_JDBC_URL=jdbc:postgresql://sonarqube-db:5432/sonar \
  -e SONAR_JDBC_USERNAME=sonar \
  -e SONAR_JDBC_PASSWORD=sonar \
  sonarqube:latest
```

#### 项目扫描

```bash
# 安装SonarScanner
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1-linux-x64.zip
unzip sonar-scanner-cli-5.0.1-linux-x64.zip
export PATH="$PATH:$(pwd)/sonar-scanner-5.0.1-linux-x64/bin"

# Maven项目扫描
mvn sonar:sonar \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=admin \
  -Dsonar.password=admin

# Gradle项目扫描
gradle sonarqube \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=admin \
  -Dsonar.password=admin

# 使用SonarScanner
sonar-scanner \
  -Dsonar.projectKey=my-project \
  -Dsonar.sources=src \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=admin \
  -Dsonar.password=admin
```

#### 配置文件示例

```properties
# sonar-project.properties
sonar.projectKey=my-java-project
sonar.projectName=My Java Project
sonar.projectVersion=1.0

# 源码路径
sonar.sources=src/main/java
sonar.tests=src/test/java

# Java版本
sonar.java.source=17

# 编译后的类文件
sonar.java.binaries=target/classes

# 排除文件
sonar.exclusions=**/generated/**,**/test/**

# 启用安全规则
sonar.security.hotspots.enabled=true

# 关联GitLab/GitHub
sonar.links.ci=https://github.com/org/repo/actions
```

---

## 三、AI辅助代码审计

### 3.1 AI审计概述

2026年，AI已成为代码审计的重要辅助工具。AI可以：
- 快速理解复杂代码逻辑
- 识别潜在漏洞模式
- 生成审计报告
- 提供修复建议

### 3.2 IRIS工具

IRIS是一款AI驱动的代码审计工具。

#### 安装使用

```bash
# 克隆仓库
git clone https://github.com/IRIS-Tools/IRIS.git
cd IRIS

# 安装依赖
pip install -r requirements.txt

# 配置API密钥
export OPENAI_API_KEY=your_api_key

# 运行审计
python iris.py --target /path/to/project --lang java

# 输出报告
python iris.py --target /path/to/project --output report.html
```

### 3.3 LLM辅助漏洞发现

#### Prompt设计原则

```
好的审计Prompt应包含：
1. 明确的角色定义
2. 具体的审计目标
3. 代码上下文
4. 输出格式要求
5. 漏洞类型说明
```

#### 审计Prompt模板

**通用审计Prompt**：

```
你是一位资深的Java安全审计专家。请审计以下代码，重点关注：

1. SQL注入漏洞
2. XSS漏洞
3. 命令注入漏洞
4. 路径遍历漏洞
5. 反序列化漏洞
6. SSRF漏洞
7. 敏感信息泄露

代码：
```java
{code}
```

请按以下格式输出：
- 漏洞类型：
- 漏洞位置：
- 漏洞原因：
- 风险等级：
- 修复建议：
- 修复代码示例：
```

**SQL注入专项Prompt**：

```
作为Java安全专家，请分析以下代码是否存在SQL注入漏洞。

重点关注：
1. 字符串拼接构造SQL语句
2. 用户输入直接拼接到SQL
3. 是否使用PreparedStatement
4. 参数化查询是否正确实现

代码：
```java
{code}
```

分析要求：
1. 识别所有SQL查询点
2. 追踪用户输入来源
3. 判断是否存在注入点
4. 提供修复方案
```

**反序列化专项Prompt**：

```
分析以下Java代码的反序列化安全问题：

代码：
```java
{code}
```

检查要点：
1. ObjectInputStream使用
2. 反序列化对象类型验证
3. 是否存在可利用的Gadget Chain
4. Java版本和依赖库分析

输出：
- 风险点：
- 利用可能性：
- 修复建议：
```

### 3.4 结果验证

AI审计结果需要人工验证：

```python
#!/usr/bin/env python3
"""
AI审计结果验证脚本
"""
import subprocess
import json

def verify_sql_injection(file_path, line_number, payload):
    """验证SQL注入漏洞"""
    # 构造测试请求
    test_url = f"http://target/api?id={payload}"
    
    # 发送请求
    result = subprocess.run(
        ['curl', '-s', test_url],
        capture_output=True, text=True
    )
    
    # 判断是否存在注入
    if "error" in result.stdout.lower() or "sql" in result.stdout.lower():
        return True, result.stdout
    return False, result.stdout

def verify_xss(file_path, line_number, payload):
    """验证XSS漏洞"""
    test_url = f"http://target/search?q={payload}"
    
    result = subprocess.run(
        ['curl', '-s', test_url],
        capture_output=True, text=True
    )
    
    if payload in result.stdout:
        return True, result.stdout
    return False, result.stdout

def verify_ai_results(ai_results):
    """验证AI审计结果"""
    verified = []
    
    for finding in ai_results:
        vuln_type = finding['type']
        file_path = finding['file']
        line_number = finding['line']
        
        if vuln_type == 'sql_injection':
            is_vuln, evidence = verify_sql_injection(
                file_path, line_number, "' OR '1'='1"
            )
        elif vuln_type == 'xss':
            is_vuln, evidence = verify_xss(
                file_path, line_number, "<script>alert(1)</script>"
            )
        
        verified.append({
            **finding,
            'verified': is_vuln,
            'evidence': evidence
        })
    
    return verified
```

---

## 四、Java常见漏洞模式

### 4.1 SQL注入

#### 漏洞代码示例

```java
// 错误示例：字符串拼接
public User getUser(String id) {
    String sql = "SELECT * FROM users WHERE id = " + id;
    // 或
    String sql = "SELECT * FROM users WHERE id = '" + id + "'";
    Statement stmt = connection.createStatement();
    ResultSet rs = stmt.executeQuery(sql);
    return mapUser(rs);
}

// 错误示例：MyBatis使用${}
@Select("SELECT * FROM users WHERE name = '${name}'")
User findByName(String name);
```

#### 安全代码示例

```java
// 正确示例：使用PreparedStatement
public User getUser(String id) {
    String sql = "SELECT * FROM users WHERE id = ?";
    PreparedStatement pstmt = connection.prepareStatement(sql);
    pstmt.setString(1, id);
    ResultSet rs = pstmt.executeQuery();
    return mapUser(rs);
}

// 正确示例：MyBatis使用#{}
@Select("SELECT * FROM users WHERE name = #{name}")
User findByName(String name);

// 正确示例：JPA/Hibernate
@Query("SELECT u FROM User u WHERE u.name = :name")
User findByName(@Param("name") String name);

// 正确示例：使用ORM框架
public User getUser(Long id) {
    return userRepository.findById(id).orElse(null);
}
```

#### 审计要点

```
SQL注入审计检查点：
1. Statement vs PreparedStatement
2. 字符串拼接SQL语句
3. MyBatis的${} vs #{}
4. HQL/JPQL拼接
5. 存储过程调用
6. 动态SQL构建
```

### 4.2 XSS

#### 漏洞代码示例

```java
// 错误示例：直接输出用户输入
@GetMapping("/search")
public String search(@RequestParam String q, Model model) {
    model.addAttribute("result", q);
    return "result";  // JSP直接输出 ${result}
}

// 错误示例：Servlet直接写入
protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
    String name = req.getParameter("name");
    resp.getWriter().write("<h1>Hello " + name + "</h1>");
}
```

#### 安全代码示例

```java
// 正确示例：使用HTML编码
import org.apache.commons.text.StringEscapeUtils;

@GetMapping("/search")
public String search(@RequestParam String q, Model model) {
    String safe = StringEscapeUtils.escapeHtml4(q);
    model.addAttribute("result", safe);
    return "result";
}

// 正确示例：使用OWASP ESAPI
import org.owasp.esapi.ESAPI;

public String safeOutput(String input) {
    return ESAPI.encoder().encodeForHTML(input);
}

// 正确示例：设置Content-Type
resp.setContentType("text/html; charset=UTF-8");
resp.setHeader("X-XSS-Protection", "1; mode=block");
resp.setHeader("X-Content-Type-Options", "nosniff");
```

#### JSP安全实践

```jsp
<!-- 错误示例 -->
<%= request.getParameter("name") %>
${param.name}

<!-- 正确示例：使用JSTL -->
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@ taglib prefix="fn" uri="http://java.sun.com/jsp/jstl/functions" %>

<c:out value="${param.name}" />
${fn:escapeXml(param.name)}
```

### 4.3 命令注入

#### 漏洞代码示例

```java
// 错误示例：直接拼接用户输入
public String ping(String host) {
    Runtime runtime = Runtime.getRuntime();
    Process process = runtime.exec("ping -c 4 " + host);
    return readOutput(process);
}

// 错误示例：ProcessBuilder拼接
public String execute(String cmd) {
    ProcessBuilder pb = new ProcessBuilder("sh", "-c", cmd);
    Process process = pb.start();
    return readOutput(process);
}
```

#### 安全代码示例

```java
// 正确示例：参数分离
public String ping(String host) {
    // 验证输入格式
    if (!host.matches("^[a-zA-Z0-9.-]+$")) {
        throw new IllegalArgumentException("Invalid host");
    }
    
    ProcessBuilder pb = new ProcessBuilder(
        Arrays.asList("ping", "-c", "4", host)
    );
    Process process = pb.start();
    return readOutput(process);
}

// 正确示例：使用白名单
private static final Set<String> ALLOWED_COMMANDS = 
    Set.of("status", "version", "help");

public String execute(String cmd) {
    if (!ALLOWED_COMMANDS.contains(cmd)) {
        throw new SecurityException("Command not allowed");
    }
    // 安全执行
}
```

### 4.4 反序列化漏洞

#### 漏洞代码示例

```java
// 错误示例：不安全的反序列化
public Object deserialize(byte[] data) throws Exception {
    ByteArrayInputStream bis = new ByteArrayInputStream(data);
    ObjectInputStream ois = new ObjectInputStream(bis);
    return ois.readObject();
}

// 错误示例：XMLDecoder反序列化
public Object fromXML(String xml) {
    XMLDecoder decoder = new XMLDecoder(new ByteArrayInputStream(xml.getBytes()));
    return decoder.readObject();
}
```

#### 安全代码示例

```java
// 正确示例：使用ValidatingObjectInputStream
import org.apache.commons.io.serialization.ValidatingObjectInputStream;

public Object safeDeserialize(byte[] data) throws Exception {
    ValidatingObjectInputStream ois = new ValidatingObjectInputStream(
        new ByteArrayInputStream(data)
    );
    // 只允许特定类
    ois.accept(MyClass.class, String.class, Long.class);
    return ois.readObject();
}

// 正确示例：使用JSON替代Java序列化
import com.fasterxml.jackson.databind.ObjectMapper;

public <T> T fromJson(String json, Class<T> clazz) {
    ObjectMapper mapper = new ObjectMapper();
    mapper.enable(JsonParser.Feature.STRICT_DUPLICATE_DETECTION);
    return mapper.readValue(json, clazz);
}

// 正确示例：自定义ObjectInputStream
public class SafeObjectInputStream extends ObjectInputStream {
    private static final Set<String> ALLOWED_CLASSES = Set.of(
        "java.lang.String",
        "java.lang.Long",
        "com.example.MyClass"
    );
    
    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) 
        throws IOException, ClassNotFoundException {
        if (!ALLOWED_CLASSES.contains(desc.getName())) {
            throw new InvalidClassException("Unauthorized deserialization", desc.getName());
        }
        return super.resolveClass(desc);
    }
}
```

### 4.5 SSRF

#### 漏洞代码示例

```java
// 错误示例：用户可控的URL
public String fetchUrl(String url) throws Exception {
    URL target = new URL(url);
    HttpURLConnection conn = (HttpURLConnection) target.openConnection();
    return readResponse(conn);
}

// 错误示例：HTTP客户端使用用户输入
public String proxy(String targetUrl) {
    CloseableHttpClient client = HttpClients.createDefault();
    HttpGet request = new HttpGet(targetUrl);
    CloseableHttpResponse response = client.execute(request);
    return EntityUtils.toString(response.getEntity());
}
```

#### 安全代码示例

```java
// 正确示例：URL白名单验证
public String fetchUrl(String url) throws Exception {
    URL target = new URL(url);
    String host = target.getHost();
    
    // 白名单检查
    if (!isAllowedHost(host)) {
        throw new SecurityException("Host not allowed");
    }
    
    // 协议检查
    String protocol = target.getProtocol();
    if (!Set.of("http", "https").contains(protocol)) {
        throw new SecurityException("Protocol not allowed");
    }
    
    // 内网IP检查
    InetAddress address = InetAddress.getByName(host);
    if (isPrivateIP(address)) {
        throw new SecurityException("Private IP not allowed");
    }
    
    HttpURLConnection conn = (HttpURLConnection) target.openConnection();
    return readResponse(conn);
}

private boolean isAllowedHost(String host) {
    Set<String> allowed = Set.of("api.example.com", "cdn.example.com");
    return allowed.contains(host);
}

private boolean isPrivateIP(InetAddress address) {
    return address.isSiteLocalAddress() 
        || address.isLoopbackAddress()
        || address.isLinkLocalAddress();
}
```

### 4.6 路径遍历

#### 漏洞代码示例

```java
// 错误示例：直接使用用户输入构造路径
public File getFile(String filename) {
    return new File("/var/www/uploads/" + filename);
}

// 错误示例：Path拼接
public Path getResource(String name) {
    return Paths.get("/app/resources").resolve(name);
}
```

#### 安全代码示例

```java
// 正确示例：路径验证
public File getFile(String filename) throws IOException {
    File baseDir = new File("/var/www/uploads").getCanonicalFile();
    File targetFile = new File(baseDir, filename).getCanonicalFile();
    
    // 检查是否在基础目录内
    if (!targetFile.getPath().startsWith(baseDir.getPath())) {
        throw new SecurityException("Path traversal detected");
    }
    
    return targetFile;
}

// 正确示例：使用Path规范化
public Path getResource(String name) throws IOException {
    Path baseDir = Paths.get("/app/resources").normalize().toAbsolutePath();
    Path targetPath = baseDir.resolve(name).normalize().toAbsolutePath();
    
    if (!targetPath.startsWith(baseDir)) {
        throw new SecurityException("Invalid path");
    }
    
    return targetPath;
}

// 正确示例：文件名白名单
public File getFile(String filename) {
    // 只允许字母数字和特定字符
    if (!filename.matches("^[a-zA-Z0-9_.-]+$")) {
        throw new SecurityException("Invalid filename");
    }
    return new File("/var/www/uploads", filename);
}
```

---

## 五、Spring框架安全审计

### 5.1 Spring Security配置审计

#### 安全配置检查

```java
// 检查点：Spring Security配置
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 检查：是否禁用了CSRF保护
            .csrf(csrf -> csrf.disable())  // 危险！
            
            // 检查：URL授权配置
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/**").permitAll()  // 检查是否过于宽松
            )
            
            // 检查：会话管理
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            
            // 检查：CORS配置
            .cors(cors -> cors.configurationSource(corsConfig()));
        
        return http.build();
    }
}
```

#### 常见配置问题

```
Spring Security审计检查点：
1. CSRF保护是否禁用
2. URL授权是否过于宽松
3. 会话管理配置
4. CORS配置是否安全
5. 认证机制强度
6. 密码编码器选择
7. Remember Me安全性
```

### 5.2 Spring MVC审计

#### 参数绑定安全

```java
// 检查点：参数绑定
@GetMapping("/user")
public User getUser(@RequestParam Long id) {
    return userService.findById(id);
}

// 检查点：路径变量
@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}

// 检查点：请求体绑定
@PostMapping("/user")
public User createUser(@RequestBody User user) {
    return userService.save(user);
}

// 检查点：表单绑定
@PostMapping("/user")
public String createUser(@ModelAttribute User user) {
    userService.save(user);
    return "redirect:/users";
}
```

#### 批量赋值漏洞

```java
// 错误示例：允许修改敏感字段
public class User {
    private String username;
    private String password;
    private boolean admin;  // 敏感字段
    // getters and setters
}

@PostMapping("/user")
public User update(@ModelAttribute User user) {
    return userService.save(user);
    // 攻击者可以设置 admin=true
}

// 正确示例：使用DTO
public class UserUpdateDTO {
    private String username;
    private String password;
    // 不包含admin字段
}

@PostMapping("/user")
public User update(@ModelAttribute UserUpdateDTO dto) {
    User user = userService.findById(dto.getId());
    user.setUsername(dto.getUsername());
    user.setPassword(encodePassword(dto.getPassword()));
    // 不更新admin字段
    return userService.save(user);
}
```

### 5.3 Spring Data审计

```java
// 检查点：JPA查询
@Query("SELECT u FROM User u WHERE u.name = ?1")  // 位置参数
User findByName(String name);

@Query("SELECT u FROM User u WHERE u.name = :name")  // 命名参数
User findByName(@Param("name") String name);

// 检查点：原生SQL
@Query(value = "SELECT * FROM users WHERE name = ?1", nativeQuery = true)
User findByNameNative(String name);

// 检查点：动态查询
public List<User> search(String name) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> cq = cb.createQuery(User.class);
    Root<User> user = cq.from(User.class);
    
    // 检查动态条件是否安全
    cq.select(user).where(cb.equal(user.get("name"), name));
    
    return em.createQuery(cq).getResultList();
}
```

---

## 六、审计实战案例

### 6.1 案例一：电商系统审计

**审计目标**：某电商系统用户模块

**步骤一：静态扫描**

```bash
# 使用Semgrep扫描
semgrep --config "p/java" --config "p/owasp-top-ten" ./src/

# 使用SonarQube扫描
mvn sonar:sonar -Dsonar.host.url=http://localhost:9000
```

**步骤二：人工审计关键代码**

```java
// 发现漏洞：UserController.java
@GetMapping("/order")
public Order getOrder(@RequestParam String orderId) {
    // 问题：没有验证用户是否有权限查看该订单
    return orderService.findById(orderId);
}
```

**漏洞分析**：
- 类型：越权访问（IDOR）
- 原因：未验证当前用户是否拥有该订单
- 影响：用户可以查看任意订单信息

**修复方案**：

```java
@GetMapping("/order")
public Order getOrder(@RequestParam String orderId, Principal principal) {
    Order order = orderService.findById(orderId);
    
    // 验证订单归属
    if (!order.getUserId().equals(principal.getName())) {
        throw new AccessDeniedException("无权访问该订单");
    }
    
    return order;
}
```

### 6.2 案例二：支付系统审计

**审计目标**：支付回调接口

**发现漏洞**：

```java
@PostMapping("/payment/callback")
public void paymentCallback(HttpServletRequest request) {
    String orderId = request.getParameter("orderId");
    String status = request.getParameter("status");
    String amount = request.getParameter("amount");
    
    // 问题：未验证签名，直接更新订单状态
    orderService.updateStatus(orderId, status);
    
    // 问题：未验证金额是否一致
    // 攻击者可以伪造成功状态，修改任意订单
}
```

**漏洞分析**：
- 类型：业务逻辑漏洞
- 原因：未验证回调签名和金额
- 影响：攻击者可以伪造支付成功

**修复方案**：

```java
@PostMapping("/payment/callback")
public void paymentCallback(HttpServletRequest request) {
    String orderId = request.getParameter("orderId");
    String status = request.getParameter("status");
    String amount = request.getParameter("amount");
    String sign = request.getParameter("sign");
    
    // 验证签名
    String expectedSign = calculateSign(orderId, status, amount, SECRET_KEY);
    if (!expectedSign.equals(sign)) {
        throw new SecurityException("Invalid signature");
    }
    
    // 验证金额
    Order order = orderService.findById(orderId);
    if (!order.getAmount().equals(new BigDecimal(amount))) {
        throw new SecurityException("Amount mismatch");
    }
    
    // 验证状态
    if (!"success".equals(status)) {
        throw new SecurityException("Invalid status");
    }
    
    orderService.updateStatus(orderId, status);
}
```

### 6.3 案例三：文件上传审计

**发现漏洞**：

```java
@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) {
    String filename = file.getOriginalFilename();
    String suffix = filename.substring(filename.lastIndexOf("."));
    
    // 问题：仅检查后缀，可被绕过
    if (!Arrays.asList(".jpg", ".png").contains(suffix)) {
        return "error";
    }
    
    // 问题：文件名未处理，可能路径遍历
    File dest = new File("/uploads/" + filename);
    file.transferTo(dest);
    
    return "success";
}
```

**绕过方式**：
1. 双写后缀：`shell.php.jpg`
2. 空字节截断：`shell.php%00.jpg`
3. 路径遍历：`../../shell.php`
4. Content-Type伪造

**修复方案**：

```java
@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) {
    // 1. 验证文件类型（通过Magic Number）
    String contentType = detectFileType(file.getBytes());
    if (!Arrays.asList("image/jpeg", "image/png").contains(contentType)) {
        return "error";
    }
    
    // 2. 生成安全文件名
    String newFilename = UUID.randomUUID().toString() + 
        getExtensionByContentType(contentType);
    
    // 3. 确保存储路径安全
    Path uploadDir = Paths.get("/uploads").normalize().toAbsolutePath();
    Path destPath = uploadDir.resolve(newFilename).normalize();
    
    if (!destPath.startsWith(uploadDir)) {
        return "error";
    }
    
    // 4. 保存文件
    Files.copy(file.getInputStream(), destPath);
    
    return "success";
}
```

---

## 七、审计工作流

### 7.1 标准审计流程

```
代码审计工作流：
├── 准备阶段
│   ├── 获取源码
│   ├── 理解架构
│   ├── 识别技术栈
│   └── 配置审计工具
├── 自动扫描
│   ├── 静态分析工具扫描
│   ├── 依赖漏洞扫描
│   └── 配置安全检查
├── 人工审计
│   ├── 入口点分析
│   ├── 数据流追踪
│   ├── 业务逻辑审计
│   └── 敏感功能审计
├── 漏洞验证
│   ├── 构造PoC
│   ├── 环境验证
│   └── 影响评估
└── 报告输出
    ├── 漏洞描述
    ├── 修复建议
    └── 风险评级
```

### 7.2 审计检查清单

```markdown
## Java代码审计检查清单

### 输入验证
- [ ] 所有用户输入是否经过验证
- [ ] 输入长度是否限制
- [ ] 输入类型是否验证
- [ ] 特殊字符是否过滤

### SQL注入
- [ ] 是否使用PreparedStatement
- [ ] MyBatis是否使用#{}而非${}
- [ ] 动态SQL是否安全

### XSS
- [ ] 输出是否经过HTML编码
- [ ] JSP是否使用JSTL
- [ ] 响应头是否设置安全

### 认证授权
- [ ] 认证机制是否安全
- [ ] 会话管理是否正确
- [ ] 权限检查是否完整
- [ ] 越权访问是否防护

### 文件操作
- [ ] 文件路径是否验证
- [ ] 文件类型是否检查
- [ ] 文件大小是否限制

### 敏感数据
- [ ] 密码是否加密存储
- [ ] 敏感信息是否脱敏
- [ ] 传输是否加密

### 错误处理
- [ ] 错误信息是否泄露敏感信息
- [ ] 异常是否正确处理
- [ ] 日志是否安全

### 配置安全
- [ ] 配置文件是否安全
- [ ] 依赖是否存在漏洞
- [ ] 调试功能是否关闭
```

### 7.3 审计报告模板

```markdown
# 代码安全审计报告

## 项目信息
- 项目名称：
- 审计时间：
- 审计人员：
- 代码规模：

## 审计范围
- 审计模块：
- 技术栈：
- 审计工具：

## 漏洞汇总

| 编号 | 漏洞类型 | 风险等级 | 影响范围 | 状态 |
|------|----------|----------|----------|------|
| V-001 | SQL注入 | 高 | 用户模块 | 待修复 |
| V-002 | XSS | 中 | 评论功能 | 待修复 |
| V-003 | 越权访问 | 高 | 订单模块 | 待修复 |

## 漏洞详情

### V-001: SQL注入漏洞

**漏洞位置**：`src/main/java/com/example/UserDao.java:45`

**漏洞代码**：
```java
String sql = "SELECT * FROM users WHERE id = " + id;
```

**漏洞描述**：
用户输入直接拼接到SQL语句，存在SQL注入风险。

**风险等级**：高

**修复建议**：
使用PreparedStatement进行参数化查询。

**修复代码**：
```java
String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, id);
```

## 审计结论
- 发现高危漏洞：2个
- 发现中危漏洞：3个
- 发现低危漏洞：5个

## 修复建议
1. 优先修复高危漏洞
2. 加强输入验证
3. 完善权限检查
4. 定期进行安全审计
```

---

## 八、总结

### 8.1 审计要点回顾

1. **工具辅助**：善用静态分析工具提高效率
2. **人工审计**：工具无法替代人工审计复杂逻辑
3. **数据流追踪**：关注数据从输入到输出的完整路径
4. **业务逻辑**：特别关注业务逻辑漏洞
5. **持续审计**：代码变更后及时审计

### 8.2 学习建议

1. **掌握Java安全基础**：理解常见漏洞原理
2. **熟悉框架安全**：Spring、MyBatis等框架的安全实践
3. **练习审计工具**：Semgrep、CodeQL等工具的使用
4. **阅读漏洞案例**：学习真实漏洞案例
5. **实践审计**：对开源项目进行审计练习

### 8.3 进阶方向

- 学习更深入的污点分析技术
- 掌握更多静态分析工具
- 研究特定框架的安全特性
- 参与开源项目安全审计
- 获取安全认证（如OSCP、OSCE）

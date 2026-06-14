# Java安全基础

> 本章面向已掌握Java基础语法的读者，聚焦安全相关的编码实践与防护技术。

## 一、Java安全编码规范

### 1.1 输入验证原则

**永远不要信任用户输入**是安全编码的第一准则。所有来自外部的数据（HTTP请求、文件、数据库、网络等）都应视为不可信。

```java
// 错误示例：直接使用用户输入
String username = request.getParameter("username");
System.out.println("Hello " + username); // XSS风险

// 正确示例：输入验证与输出编码
String username = request.getParameter("username");
if (username != null && username.matches("^[a-zA-Z0-9_]{3,20}$")) {
    String safeOutput = htmlEncode(username);
    System.out.println("Hello " + safeOutput);
}
```

### 1.2 最小权限原则

- 应用程序应以最低必要权限运行
- 避免使用root/admin账户运行Java应用
- 文件操作仅申请必要权限
- 数据库账户仅授予必要权限

### 1.3 安全配置管理

```java
// 敏感信息不应硬编码
// 错误示例
String dbPassword = "admin123"; // 绝对禁止

// 正确示例：使用环境变量或配置中心
String dbPassword = System.getenv("DB_PASSWORD");
// 或使用Vault等密钥管理服务
```

## 二、常见安全漏洞模式

### 2.1 SQL注入

SQL注入是最危险的Web漏洞之一，攻击者可通过构造恶意输入篡改SQL语句。

```java
// 错误示例：字符串拼接SQL
String sql = "SELECT * FROM users WHERE username='" + username + 
             "' AND password='" + password + "'";
// 输入: username = "admin'--"
// 实际执行: SELECT * FROM users WHERE username='admin'--' AND password='...'

// 正确示例：使用预编译语句
String sql = "SELECT * FROM users WHERE username=? AND password=?";
PreparedStatement pstmt = connection.prepareStatement(sql);
pstmt.setString(1, username);
pstmt.setString(2, password);
ResultSet rs = pstmt.executeQuery();

// 使用JPA/Hibernate
@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);
```

**防护要点：**
- 所有SQL参数使用预编译语句（PreparedStatement）
- 使用ORM框架的参数绑定
- 对输入进行白名单验证
- 限制数据库账户权限

### 2.2 XSS（跨站脚本攻击）

XSS允许攻击者在受害者浏览器中执行恶意脚本。

```java
// 反射型XSS示例
// 错误：直接输出用户输入
response.getWriter().println("<div>" + userInput + "</div>");

// 正确：输出编码
import org.apache.commons.text.StringEscapeUtils;
response.getWriter().println("<div>" + 
    StringEscapeUtils.escapeHtml4(userInput) + "</div>");
```

**XSS类型：**
| 类型 | 触发方式 | 持久性 |
|------|----------|--------|
| 反射型 | URL参数直接输出 | 无 |
| 存储型 | 存入数据库后输出 | 有 |
| DOM型 | JavaScript动态渲染 | 无 |

**防护策略：**
- 输出时进行HTML编码
- 设置Content-Type响应头
- 使用CSP（内容安全策略）
- 使用现代框架的自动转义功能

### 2.3 命令注入

命令注入可让攻击者在服务器上执行任意系统命令。

```java
// 错误示例：直接拼接命令
String filename = request.getParameter("file");
Runtime.getRuntime().exec("cat " + filename);
// 输入: file = "test.txt; rm -rf /"

// 正确示例：使用参数化方式或白名单验证
String filename = request.getParameter("file");
if (filename.matches("^[a-zA-Z0-9_.-]+$")) {
    ProcessBuilder pb = new ProcessBuilder("cat", filename);
    Process p = pb.start();
}
```

### 2.4 路径遍历

路径遍历允许攻击者访问预期目录之外的文件。

```java
// 错误示例：直接使用用户提供的路径
String filePath = request.getParameter("path");
File file = new File("/app/data/" + filePath);
// 输入: path = "../../../etc/passwd"

// 正确示例：路径规范化与验证
String basePath = "/app/data/";
String userPath = request.getParameter("path");
File file = new File(basePath, userPath).getCanonicalFile();

if (!file.getPath().startsWith(new File(basePath).getCanonicalPath())) {
    throw new SecurityException("非法路径访问");
}
```

## 三、反序列化安全基础

Java反序列化漏洞是近年来最严重的安全问题之一。

### 3.1 漏洞原理

```java
// 危险的反序列化操作
ObjectInputStream ois = new ObjectInputStream(inputStream);
Object obj = ois.readObject(); // 可执行任意代码
```

攻击者可构造恶意序列化数据，在反序列化时触发任意代码执行。

### 3.2 防护措施

```java
// 方案1：使用安全的反序列化库
import com.fasterxml.jackson.databind.ObjectMapper;
ObjectMapper mapper = new ObjectMapper();
MyObject obj = mapper.readValue(json, MyObject.class);

// 方案2：重写ObjectInputStream的resolveClass方法
public class SafeObjectInputStream extends ObjectInputStream {
    private static final Set<String> ALLOWED_CLASSES = 
        Set.of("com.example.User", "com.example.Product");
    
    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) 
            throws IOException, ClassNotFoundException {
        if (!ALLOWED_CLASSES.contains(desc.getName())) {
            throw new InvalidClassException("Unauthorized deserialization attempt", 
                                            desc.getName());
        }
        return super.resolveClass(desc);
    }
}
```

**防护要点：**
- 避免使用Java原生序列化
- 使用JSON等安全的数据格式
- 实现白名单类过滤
- 升级依赖库修复已知漏洞

## 四、安全框架使用

### 4.1 Spring Security核心概念

Spring Security是Java生态最广泛使用的安全框架。

**核心组件：**

```java
// 1. 安全配置
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/home")
            )
            .csrf(csrf -> csrf.csrfTokenRepository(
                CSRFTokenRepository.withHttpOnlyFalse()
            ))
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'")
                )
            );
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}

// 2. 密码安全存储
@Service
public class UserService {
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    public void registerUser(String username, String rawPassword) {
        String encodedPassword = passwordEncoder.encode(rawPassword);
        // 存储encodedPassword到数据库
    }
    
    public boolean authenticate(String rawPassword, String encodedPassword) {
        return passwordEncoder.matches(rawPassword, encodedPassword);
    }
}
```

### 4.2 关键安全特性

| 特性 | 说明 |
|------|------|
| 认证 | 验证用户身份（表单登录、OAuth2、JWT等） |
| 授权 | 控制资源访问权限 |
| CSRF防护 | 防止跨站请求伪造 |
| 会话管理 | 会话固定保护、并发控制 |
| 安全头 | XSS防护、CSP、HSTS等 |

## 五、AI辅助代码安全审查

### 5.1 AI审查工作流

```
代码提交 → AI扫描 → 安全报告 → 人工复核 → 修复/合并
```

### 5.2 常用AI审查提示词

```
请审查以下Java代码的安全问题：
1. 检查SQL注入、XSS、命令注入漏洞
2. 检查敏感数据处理方式
3. 检查认证授权逻辑
4. 检查异常处理是否泄露敏感信息
5. 提供修复建议和代码示例

[粘贴代码]
```

### 5.3 AI审查最佳实践

- **不要完全依赖AI**：AI可能遗漏复杂漏洞
- **结合静态分析工具**：使用SonarQube、SpotBugs等
- **关注误报**：AI可能产生大量误报，需人工判断
- **持续学习**：将AI发现的问题纳入团队知识库

### 5.4 推荐工具组合

```
AI审查（ChatGPT/Claude） + 静态分析（SonarQube） + 依赖扫描（OWASP Dependency-Check）
```

---

## 小结

Java安全编码需要建立"零信任"思维，对所有外部输入保持警惕。核心要点：

1. **输入验证**：白名单优先，类型检查
2. **输出编码**：根据上下文选择编码方式
3. **安全API**：使用预编译语句、安全框架
4. **最小权限**：应用和数据库都应最小化权限
5. **持续审查**：结合AI和自动化工具进行代码审计

下一章我们将学习Linux安全操作，为渗透测试搭建基础环境。

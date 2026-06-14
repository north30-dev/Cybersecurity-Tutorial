# CI/CD安全

## 一、CI/CD安全概述

### 1.1 CI/CD安全的重要性

CI/CD（持续集成/持续部署）流水线是现代软件交付的核心环节，也是供应链攻击的主要目标。一旦CI/CD流水线被攻破，攻击者可以：

- 在所有部署环境中植入恶意代码
- 窃取生产环境凭证和密钥
- 篡改构建产物和容器镜像
- 获得基础设施的完全控制权

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD流水线安全风险点                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  代码提交 ──▶ 构建 ──▶ 测试 ──▶ 制品存储 ──▶ 部署               │
│     │          │        │          │           │                │
│     ▼          ▼        ▼          ▼           ▼                │
│  • 恶意PR    • 依赖投毒 • 环境泄露  • 镜像篡改  • 权限滥用      │
│  • 凭证泄露  • 构建注入 • 测试绕过  • 制品污染  • 配置错误      │
│  • 代码注入  • 缓存投毒 • 结果伪造  • 签名绕过  • 环境暴露      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 CI/CD安全模型

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD纵深防御体系                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  第1层：源代码安全                                       │   │
│  │  • 代码审查 • 签名验证 • 分支保护 • 提交验证             │   │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  第2层：构建环境安全                                     │   │
│  │  • 隔离构建 • 依赖锁定 • 缓存安全 • 环境清理             │   │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  第3层：制品安全                                         │   │
│  │  • 镜像扫描 • 签名验证 • SBOM生成 • 不变性保证           │   │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  第4层：部署安全                                         │   │
│  │  • 权限最小化 • 环境隔离 • 审批流程 • 回滚机制           │   │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、流水线安全

### 2.1 代码提交安全

#### 2.1.1 分支保护策略

**GitHub分支保护配置：**

```yaml
# 分支保护规则（推荐配置）
branches:
  - name: main
    protection:
      # 要求PR审查
      required_pull_request_reviews:
        dismiss_stale_reviews: true
        require_code_owner_reviews: true
        required_approving_review_count: 2
      
      # 要求状态检查通过
      required_status_checks:
        strict: true  # 要求分支是最新的
        contexts:
          - "security/scan"
          - "test/unit"
          - "test/integration"
      
      # 强制签名提交
      enforce_admins: true
      required_linear_history: true
      allow_force_pushes: false
      allow_deletions: false
      
      # 限制推送权限
      restrictions:
        users: []
        teams: ["maintainers", "security-team"]
```

**GitLab分支保护配置：**

```yaml
# .gitlab-ci.yml 分支保护
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
    - if: $CI_MERGE_REQUEST_IID
      when: always

# 保护分支设置（通过API或UI配置）
# Settings > Repository > Protected branches:
# - main: Maintainers only, require 2 approvals
# - develop: Developers + Maintainers, require 1 approval
```

#### 2.1.2 CODEOWNERS配置

```
# .github/CODEOWNERS 或 .gitlab/CODEOWNERS

# 默认所有者
* @org/dev-team

# 安全敏感文件需要安全团队审查
/security/ @org/security-team
**/secrets/** @org/security-team
**/*.key @org/security-team
**/Dockerfile* @org/security-team @org/devops-team

# 基础设施代码需要DevOps团队审查
/terraform/ @org/devops-team
/kubernetes/ @org/devops-team
/.github/workflows/ @org/devops-team @org/security-team

# 依赖文件需要审查
package.json @org/dev-team @org/security-team
package-lock.json @org/dev-team
go.mod @org/dev-team
requirements.txt @org/dev-team
```

#### 2.1.3 提交签名验证

```bash
# 配置GPG签名（开发者本地）
git config --global commit.gpgsign true
git config --global gpg.program gpg

# 生成GPG密钥
gpg --full-generate-key

# 获取GPG公钥
gpg --armor --export <email>

# 添加到GitHub/GitLab账户设置

# 验证提交签名
git log --show-signature

# GitHub要求签名提交（组织设置）
# Settings > Organizations > Repository defaults > Commit signing
```

### 2.2 构建安全

#### 2.2.1 构建环境隔离

```yaml
# GitHub Actions - 使用隔离的构建环境
name: Secure Build
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    # 限制权限
    permissions:
      contents: read
      packages: write
    
    # 使用特定版本的环境
    container:
      image: node:18.17.0-alpine
      options: --user root
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 完整历史用于验证
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '18.17.0'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci --ignore-scripts  # 忽略postinstall脚本
      
      - name: Build
        run: npm run build
        env:
          NODE_ENV: production
```

#### 2.2.2 依赖锁定与验证

```yaml
# 验证依赖完整性
name: Dependency Verification
on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Node.js - 验证package-lock.json
      - name: Verify npm dependencies
        run: |
          npm ci --ignore-scripts
          npm audit --audit-level=high
      
      # Python - 验证requirements.txt
      - name: Verify pip dependencies
        run: |
          pip install pip-audit
          pip-audit -r requirements.txt
      
      # Go - 验证go.sum
      - name: Verify Go dependencies
        run: |
          go mod verify
          go list -m all | nancy sleuth
```

#### 2.2.3 安全构建配置

```dockerfile
# 安全的构建Dockerfile
# 阶段1：构建（使用特定版本）
FROM node:18.17.0-alpine AS builder

# 安装构建依赖（指定版本）
RUN apk add --no-cache \
    build-base=0.5-r3 \
    python3=3.11.3-r0

WORKDIR /app

# 先复制依赖文件（利用缓存）
COPY package*.json ./

# 验证依赖完整性
RUN npm ci --ignore-scripts && \
    npm audit --audit-level=high

# 复制源代码
COPY . .

# 构建（非root用户）
RUN addgroup -S builder && adduser -S builder -G builder
USER builder
RUN npm run build

# 阶段2：运行时（最小化镜像）
FROM node:18.17.0-alpine

# 安全配置
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# 仅复制必要文件
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appuser /app/node_modules ./node_modules
COPY --chown=appuser:appgroup package*.json ./

# 切换用户
USER appuser

# 只读文件系统
# 运行时添加 --read-only 标志

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### 2.3 部署安全

#### 2.3.1 部署权限控制

```yaml
# GitHub Actions - 分环境权限控制
name: Deploy
on:
  push:
    branches: [main]

jobs:
  # 开发环境部署
  deploy-dev:
    runs-on: ubuntu-latest
    environment: development
    permissions:
      contents: read
      id-token: write  # OIDC认证
    steps:
      - name: Deploy to dev
        run: kubectl apply -f k8s/dev/

  # 生产环境部署（需要审批）
  deploy-prod:
    runs-on: ubuntu-latest
    environment: production  # 配置environment protection rules
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Deploy to prod
        run: |
          # 验证镜像签名
          cosign verify $IMAGE@$DIGEST --key $COSIGN_PUBLIC_KEY
          # 部署
          kubectl apply -f k8s/prod/
```

**GitHub Environment保护规则配置：**

```yaml
# Environment配置（通过API或UI）
environments:
  - name: production
    protection_rules:
      - type: required_reviewers
        reviewers:
          - security-team
          - ops-team
      - type: branch_policy
        branch: main
        required_contexts:
          - security/scan
          - test/all
    deployment_branch_policy:
      protected_branches: true
      custom_branch_patterns: []
```

#### 2.3.2 部署审计与回滚

```yaml
# GitLab CI - 部署审计
deploy:
  stage: deploy
  script:
    # 记录部署信息
    - |
      echo "Deployment Record:
        Time: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
        Commit: $CI_COMMIT_SHA
        User: $GITLAB_USER_LOGIN
        Environment: $DEPLOY_ENV
        Image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" >> deploy.log
    
    # 部署前备份
    - kubectl get all -n $KUBE_NAMESPACE -o yaml > backup.yaml
    
    # 部署
    - kubectl apply -f k8s/
    
    # 部署后验证
    - ./scripts/verify-deployment.sh
    
  after_script:
    # 发送审计日志
    - |
      curl -X POST $AUDIT_WEBHOOK \
        -H "Content-Type: application/json" \
        -d "{\"event\":\"deployment\",\"data\":\"$(cat deploy.log)\"}"
  
  # 自动回滚配置
  retry:
    max: 2
    when:
      - script_failure
```

---

## 三、供应链安全

### 3.1 依赖管理

#### 3.1.1 依赖安全策略

```yaml
# Dependabot配置 - .github/dependabot.yml
version: 2
updates:
  # npm依赖
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
    # 自动合并补丁版本
    versioning-strategy: increase-if-necessary
    ignore:
      # 忽略主版本更新
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]

  # Docker基础镜像
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    reviewers:
      - "security-team"
      - "devops-team"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

#### 3.1.2 依赖锁定策略

```json
// package.json - 强制锁定版本
{
  "name": "myapp",
  "version": "1.0.0",
  "dependencies": {
    "express": "4.18.2",        // 精确版本
    "lodash": "~4.17.21",       // 补丁版本
    "axios": "^1.4.0"           // 兼容版本（谨慎使用）
  },
  "devDependencies": {
    "typescript": "5.0.4"
  },
  // 禁止使用未锁定的依赖
  "scripts": {
    "preinstall": "node -e \"if(process.env.npm_package_lock_integrity===undefined) throw new Error('Use npm ci instead of npm install')\""
  }
}
```

```txt
# requirements.txt - Python依赖锁定
# 使用 pip-compile 生成
#
# This file is autogenerated by pip-compile
# To update, run:
#
#    pip-compile requirements.in
#
flask==2.3.2
    # via -r requirements.in
werkzeug==2.3.4
    # via flask
requests==2.31.0
    # via -r requirements.in
certifi==2023.5.7
    # via requests
```

### 3.2 SBOM（软件物料清单）

#### 3.2.1 生成SBOM

```yaml
# GitHub Actions - 生成SBOM
name: Generate SBOM
on: [push]

jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 使用Syft生成SBOM
      - name: Generate SBOM
        uses: anchore/sbom-action@v0.14
        with:
          image: ${{ steps.build.outputs.image }}
          format: spdx-json
          output-file: sbom.spdx.json
          artifact-name: sbom
      
      # 上传SBOM作为制品
      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          name: sbom
          path: sbom.spdx.json
      
      # 附加SBOM到镜像
      - name: Attach SBOM to image
        run: |
          cosign attach sbom --sbom sbom.spdx.json $IMAGE@$DIGEST
```

**SBOM示例（SPDX格式）：**

```json
{
  "spdxVersion": "SPDX-2.3",
  "dataLicense": "CC0-1.0",
  "SPDXID": "SPDXRef-DOCUMENT",
  "name": "myapp",
  "documentNamespace": "https://example.com/sbom/myapp@1.0.0",
  "creationInfo": {
    "created": "2024-01-15T10:00:00Z",
    "creators": ["Tool: syft-0.100.0"]
  },
  "packages": [
    {
      "SPDXID": "SPDXRef-Package-npm-express",
      "name": "express",
      "versionInfo": "4.18.2",
      "supplier": "Person: TJ Holowaychuk",
      "licenseConcluded": "MIT",
      "licenseDeclared": "MIT",
      "copyrightText": "Copyright (c) 2009-2014 TJ Holowaychuk",
      "filesAnalyzed": false
    }
  ],
  "relationships": [
    {
      "spdxElementId": "SPDXRef-DOCUMENT",
      "relatedSpdxElement": "SPDXRef-Package-npm-express",
      "relationshipType": "DEPENDS_ON"
    }
  ]
}
```

#### 3.2.2 SBOM验证与监控

```yaml
# 监控SBOM中的漏洞
name: SBOM Monitor
on:
  schedule:
    - cron: '0 6 * * *'  # 每天检查

jobs:
  monitor:
    runs-on: ubuntu-latest
    steps:
      - name: Download latest SBOM
        uses: actions/download-artifact@v3
        with:
          name: sbom
      
      # 使用Grype扫描SBOM
      - name: Scan SBOM for vulnerabilities
        uses: anchore/scan-action@v3
        with:
          sbom: sbom.spdx.json
          fail-build: true
          severity-cutoff: high
      
      # 发送通知
      - name: Notify on vulnerabilities
        if: failure()
        run: |
          curl -X POST $SLACK_WEBHOOK \
            -d '{"text":"New vulnerabilities found in SBOM!"}'
```

### 3.3 签名验证

#### 3.3.1 镜像签名流水线

```yaml
# 完整的镜像签名流程
name: Build and Sign Image
on:
  push:
    branches: [main]

jobs:
  build-sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write  # 用于OIDC
    
    steps:
      - uses: actions/checkout@v4
      
      # 构建镜像
      - name: Build image
        id: build
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }} .
          echo "digest=$(docker push ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }})" >> $GITHUB_OUTPUT
      
      # 安装cosign
      - name: Install cosign
        uses: sigstore/cosign-installer@v3
      
      # 使用OIDC签名（无需管理密钥）
      - name: Sign image
        env:
          DIGEST: ${{ steps.build.outputs.digest }}
        run: |
          cosign sign --yes ${{ env.REGISTRY }}/${{ env.IMAGE }}@${DIGEST}
      
      # 生成并附加SBOM
      - name: Attach SBOM
        uses: anchore/sbom-action@v0.14
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE }}@${{ steps.build.outputs.digest }}
          format: spdx-json
      
      # 签名SBOM
      - name: Sign SBOM
        run: |
          cosign attach sbom --sbom sbom.spdx.json $IMAGE@$DIGEST
          cosign sign --attachment sbom $IMAGE@$DIGEST

    env:
      REGISTRY: ghcr.io
      IMAGE: ${{ github.repository }}
```

#### 3.3.2 部署时验证签名

```yaml
# Kubernetes部署策略 - 验证镜像签名
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: enforce
  background: false
  rules:
  - name: verify-github-actions
    match:
      resources:
        kinds:
        - Pod
    verifyImages:
    - imageReferences:
      - "ghcr.io/myorg/*"
      attestors:
      - entries:
        - keyless:
            # 验证GitHub Actions OIDC签名
            issuer: https://token.actions.githubusercontent.com
            subject: "https://github.com/myorg/myrepo/.github/workflows/build.yml@refs/heads/main"
```

---

## 四、密钥管理

### 4.1 Secrets管理策略

#### 4.1.1 密钥存储最佳实践

```
┌─────────────────────────────────────────────────────────────────┐
│                    密钥管理策略对比                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ 不推荐：                                                     │
│  • 硬编码在代码中                                               │
│  • 存储在环境变量中                                             │
│  • 提交到版本控制系统                                           │
│  • 使用明文配置文件                                             │
│                                                                 │
│  ✅ 推荐：                                                       │
│  • 使用专业密钥管理服务（Vault、AWS SM、Azure KV）              │
│  • 使用CI/CD平台的Secrets管理                                   │
│  • 运行时注入密钥                                               │
│  • 定期轮换密钥                                                 │
│  • 审计密钥访问                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 GitHub Actions Secrets

```yaml
# 使用GitHub Secrets
name: Deploy with Secrets
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 使用Repository Secrets
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      # 使用Environment Secrets（生产环境）
      - name: Deploy to production
        if: github.ref == 'refs/heads/main'
        environment: production
        run: |
          kubectl create secret generic db-credentials \
            --from-literal=password=${{ secrets.DB_PASSWORD }} \
            --dry-run=client -o yaml | kubectl apply -f -
```

#### 4.1.3 GitLab CI Variables

```yaml
# .gitlab-ci.yml - 使用Masked和Protected Variables
variables:
  # 普通变量
  APP_NAME: "myapp"
  
  # 敏感变量（在GitLab UI中配置为Masked和Protected）
  # Settings > CI/CD > Variables:
  # - AWS_ACCESS_KEY_ID (masked, protected)
  # - AWS_SECRET_ACCESS_KEY (masked, protected)
  # - DEPLOY_TOKEN (masked, protected)

deploy:
  stage: deploy
  script:
    # 使用敏感变量
    - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
    - docker push $ECR_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
  # 仅在protected分支运行（main）
  only:
    - main
```

### 4.2 外部密钥管理集成

#### 4.2.1 HashiCorp Vault集成

```yaml
# GitHub Actions - Vault集成
name: Build with Vault
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 获取Vault令牌
      - name: Get Vault Token
        id: vault
        uses: hashicorp/vault-action@v2
        with:
          url: https://vault.example.com
          role: myapp-ci
          method: jwt  # 使用OIDC认证
      
      # 从Vault获取密钥
      - name: Get Secrets
        uses: hashicorp/vault-action@v2
        with:
          url: https://vault.example.com
          secrets: |
            secret/data/myapp/database username | DB_USERNAME ;
            secret/data/myapp/database password | DB_PASSWORD ;
            secret/data/myapp/api key | API_KEY
      
      - name: Build
        run: |
          # 使用获取的密钥
          echo "DB_USERNAME=$DB_USERNAME" >> .env
          npm run build
```

**Vault策略配置：**

```hcl
# Vault策略 - CI/CD服务账户
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

path "secret/data/myapp/database" {
  capabilities = ["read"]
}

# 限制IP访问
path "auth/token/renew-self" {
  capabilities = ["update"]
}
```

#### 4.2.2 AWS Secrets Manager集成

```yaml
# 使用AWS Secrets Manager
name: Deploy with AWS Secrets
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      secrets: read  # 需要secrets读取权限
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
      
      - name: Get Secrets
        id: secrets
        run: |
          # 获取JSON格式的密钥
          SECRETS=$(aws secretsmanager get-secret-value --secret-id prod/myapp/db --query SecretString --output text)
          echo "db_password=$(echo $SECRETS | jq -r .password)" >> $GITHUB_OUTPUT
      
      - name: Deploy
        env:
          DB_PASSWORD: ${{ steps.secrets.outputs.db_password }}
        run: |
          # 部署脚本
          ./deploy.sh
```

### 4.3 密钥轮换

#### 4.3.1 自动密钥轮换

```yaml
# 定期密钥轮换工作流
name: Rotate Secrets
on:
  schedule:
    - cron: '0 0 1 * *'  # 每月1日执行
  workflow_dispatch:      # 手动触发

jobs:
  rotate:
    runs-on: ubuntu-latest
    steps:
      - name: Generate new API key
        id: generate
        run: |
          NEW_KEY=$(openssl rand -hex 32)
          echo "new_key=$NEW_KEY" >> $GITHUB_OUTPUT
      
      - name: Update in Vault
        run: |
          vault kv put secret/myapp/api key=${{ steps.generate.outputs.new_key }}
      
      - name: Update in AWS Secrets Manager
        run: |
          aws secretsmanager put-secret-value \
            --secret-id prod/myapp/api \
            --secret-string '{"key":"${{ steps.generate.outputs.new_key }}"}'
      
      - name: Trigger redeploy
        run: |
          # 触发重新部署以使用新密钥
          curl -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/repos/${{ github.repository }}/dispatches \
            -d '{"event_type":"secrets-rotated"}'
      
      - name: Notify team
        run: |
          curl -X POST $SLACK_WEBHOOK \
            -d '{"text":"API key rotated successfully. Redeploy triggered."}'
```

#### 4.3.2 密钥泄露检测

```yaml
# 密钥泄露扫描
name: Secret Scan
on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 完整历史
      
      # Gitleaks扫描
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_CONFIG: .gitleaks.toml
      
      # TruffleHog扫描
      - name: Run TruffleHog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          extra_args: --only-verified
```

**Gitleaks配置：**

```toml
# .gitleaks.toml
title = "Gitleaks Config"

[extend]
useDefault = true

[[rules]]
id = "aws-access-key"
description = "AWS Access Key"
regex = '''AKIA[0-9A-Z]{16}'''
tags = ["key", "AWS"]

[[rules]]
id = "private-key"
description = "Private Key"
regex = '''-----BEGIN (?:RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----'''
tags = ["key", "private"]

[allowlist]
description = "Allowlist for test files"
paths = [
  '''^tests/''',
  '''^\.github/workflows/test-''',
]
```

---

## 五、制品安全

### 5.1 制品仓库安全

#### 5.1.1 容器镜像仓库配置

```yaml
# 推荐的镜像仓库安全配置

# AWS ECR配置
resource "aws_ecr_repository" "app" {
  name                 = "myapp"
  image_tag_mutability = "IMMUTABLE"  # 标签不可变
  
  encryption_configuration {
    encryption_type = "KMS"
    kms_key        = aws_kms_key.ecr.arn
  }
  
  image_scanning_configuration {
    scan_on_push = true  # 推送时自动扫描
  }
}

# 仓库策略 - 限制访问
resource "aws_ecr_repository_policy" "app" {
  repository = aws_ecr_repository.app.name
  policy     = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowPullFromCI"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.ci_role.arn
        }
        Action = [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability"
        ]
      },
      {
        Sid    = "AllowPushFromCI"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.ci_role.arn
        }
        Action = [
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload"
        ]
        Condition = {
          StringEquals = {
            "ecr:repositoryTagMutability" = "IMMUTABLE"
          }
        }
      }
    ]
  })
}
```

#### 5.1.2 制品签名策略

```yaml
# Kyverno策略 - 仅允许签名的镜像
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: signed-images-only
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: verify-signature
    match:
      resources:
        kinds:
        - Pod
    verifyImages:
    - imageReferences:
      - "ecr.example.com/*"
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
              -----END PUBLIC KEY-----
      - entries:
        - keyless:
            issuer: https://token.actions.githubusercontent.com
            subject: "https://github.com/myorg/myrepo/.github/workflows/build.yml@refs/heads/main"
```

### 5.2 制品扫描

```yaml
# 完整的制品安全扫描流水线
name: Artifact Security Scan
on:
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 构建镜像
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      
      # 漏洞扫描
      - name: Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      # 上传扫描结果
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      # CIS基准扫描
      - name: Docker CIS benchmark
        uses: docker://aquasec/kube-bench:latest
        with:
          args: run --targets node
      
      # 敏感信息扫描
      - name: Scan for secrets in image
        uses: trufflesecurity/trufflehog@main
        with:
          image: 'myapp:${{ github.sha }}'
```

---

## 六、常见CI/CD平台安全配置

### 6.1 Jenkins安全配置

#### 6.1.1 Jenkins安全设置

```groovy
// Jenkins安全配置
import jenkins.model.*
import hudson.security.*

def instance = Jenkins.getInstance()

// 安全域配置
def hudsonRealm = new HudsonPrivateSecurityRealm(false)
hudsonRealm.setAllowsSignup(false)  // 禁止注册
instance.setSecurityRealm(hudsonRealm)

// 授权策略 - 基于矩阵
def strategy = new GlobalMatrixAuthorizationStrategy()
// 管理员权限
strategy.add(Jenkins.ADMINISTER, "admin")
// 开发者权限
strategy.add(Jenkins.READ, "developers")
strategy.add(Item.READ, "developers")
strategy.add(Item.BUILD, "developers")
// 匿名无权限
strategy.add(Jenkins.READ, "anonymous")
instance.setAuthorizationStrategy(strategy)

// CSRF保护
instance.setCrumbIssuer(new DefaultCrumbIssuer(true))

// 禁用CLI
instance.getDescriptor("hudson.cli.CLI").setDisabled(true)

instance.save()
```

#### 6.1.2 Jenkins流水线安全

```groovy
// Jenkinsfile - 安全流水线
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: builder
    image: node:18-alpine
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      readOnlyRootFilesystem: true
'''
        }
    }
    
    environment {
        // 使用Credentials绑定，不暴露明文
        DB_PASSWORD = credentials('db-password')
        AWS_CREDS = credentials('aws-credentials')
    }
    
    options {
        // 构建超时
        timeout(time: 30, unit: 'MINUTES')
        // 禁止并发构建
        disableConcurrentBuilds()
        // 构建历史限制
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // 跳过默认checkout
        skipDefaultCheckout(true)
    }
    
    stages {
        stage('Checkout') {
            steps {
                // 安全checkout
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    extensions: [
                        // 强制清理工作区
                        [$class: 'CleanBeforeCheckout'],
                        // 禁用git config
                        [$class: 'DisableGitConfig']
                    ],
                    userRemoteConfigs: [[
                        url: 'https://github.com/org/repo.git',
                        credentialsId: 'github-token'
                    ]]
                ])
            }
        }
        
        stage('Security Scan') {
            steps {
                // 依赖检查
                sh 'npm audit --audit-level=high'
                // SAST扫描
                sh 'semgrep --config auto .'
                // 密钥扫描
                sh 'gitleaks detect --source . --no-git'
            }
        }
        
        stage('Build') {
            steps {
                // 忽略postinstall脚本
                sh 'npm ci --ignore-scripts'
                sh 'npm run build'
            }
        }
        
        stage('Container Scan') {
            steps {
                script {
                    // 构建镜像
                    sh "docker build -t myapp:${env.BUILD_NUMBER} ."
                    // 扫描镜像
                    sh "trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
                submitter "security-team,ops-team"
            }
            steps {
                script {
                    // 使用临时凭证
                    withAWS(credentials: 'aws-credentials', region: 'us-east-1') {
                        sh 'kubectl apply -f k8s/production/'
                    }
                }
            }
        }
    }
    
    post {
        always {
            // 清理工作区
            cleanWs()
        }
        failure {
            // 通知
            slackSend(channel: '#security-alerts', 
                     message: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
    }
}
```

### 6.2 GitLab CI安全配置

```yaml
# .gitlab-ci.yml - 安全配置
stages:
  - security
  - build
  - test
  - deploy

variables:
  # 安全变量配置
  FF_USE_FASTZIP: "true"  # 防止zip slip攻击
  GIT_DEPTH: "1"          # 浅克隆减少泄露风险
  SECURE_LOG_LEVEL: "info"

# 安全扫描阶段
security_scan:
  stage: security
  image: registry.gitlab.com/security-products/gitleaks:latest
  script:
    - gitleaks detect --source . --verbose
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# SAST扫描（GitLab内置）
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml

# SAST配置
sast:
  variables:
    SAST_EXCLUDED_ANALYZERS: "semgrep"  # 排除特定分析器
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# 安全构建
build:
  stage: build
  image: node:18-alpine
  before_script:
    # 验证依赖完整性
    - npm ci --ignore-scripts
    - npm audit --audit-level=high
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

# 部署到生产（需要审批）
deploy:production:
  stage: deploy
  environment:
    name: production
    url: https://app.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual  # 手动触发
  script:
    - kubectl apply -f k8s/production/
  # 配置环境保护（Settings > CI/CD > Protected environments）
```

### 6.3 GitHub Actions安全配置

```yaml
# 完整的安全GitHub Actions工作流
name: Secure CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# 最小权限原则
permissions:
  contents: read

jobs:
  # 安全扫描
  security:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write  # 用于上传SARIF
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false  # 不持久化凭证
      
      # CodeQL扫描
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript, python
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
      
      # 依赖审查
      - name: Dependency Review
        uses: actions/dependency-review-action@v3
        if: github.event_name == 'pull_request'
      
      # 密钥扫描
      - name: Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}

  # 构建
  build:
    needs: security
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write  # OIDC
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      
      # 漏洞扫描
      - name: Run Trivy
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      # 构建并推送
      - name: Build and Push
        id: build
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ghcr.io/${{ github.repository }}:${{ github.sha }} | cut -d@ -f2)
          echo "digest=$DIGEST" >> $GITHUB_OUTPUT
      
      # 签名
      - name: Sign Image
        uses: sigstore/cosign-installer@v3
      - run: |
          cosign sign --yes ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}

  # 部署到生产
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production  # 需要审批
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy
        run: |
          # 验证签名
          cosign verify ghcr.io/${{ github.repository }}@${{ needs.build.outputs.digest }} \
            --certificate-identity-regexp="https://github.com/${{ github.repository }}/.github/workflows/*" \
            --certificate-oidc-issuer="https://token.actions.githubusercontent.com"
          
          # 部署
          kubectl apply -f k8s/production/
```

---

## 七、CI/CD安全检查清单

### 7.1 源代码安全

- [ ] 启用分支保护规则
- [ ] 要求代码审查和审批
- [ ] 强制提交签名验证
- [ ] 配置CODEOWNERS
- [ ] 禁止强制推送
- [ ] 定期审计仓库权限

### 7.2 构建安全

- [ ] 使用隔离的构建环境
- [ ] 锁定依赖版本
- [ ] 验证依赖完整性
- [ ] 禁用postinstall脚本
- [ ] 清理构建缓存
- [ ] 限制构建超时

### 7.3 制品安全

- [ ] 启用镜像漏洞扫描
- [ ] 签名所有制品
- [ ] 生成并附加SBOM
- [ ] 使用不可变标签
- [ ] 限制仓库访问权限
- [ ] 启用仓库加密

### 7.4 部署安全

- [ ] 实施部署审批流程
- [ ] 验证制品签名
- [ ] 使用最小权限原则
- [ ] 分离环境权限
- [ ] 启用部署审计
- [ ] 配置自动回滚

### 7.5 密钥管理

- [ ] 使用外部密钥管理
- [ ] 禁止硬编码密钥
- [ ] 启用密钥轮换
- [ ] 审计密钥访问
- [ ] 使用临时凭证
- [ ] 扫描密钥泄露

---

## 参考资料

1. [OWASP CI/CD Security Top 10](https://owasp.org/www-project-top-10-ci-cd-security-risks/)
2. [GitHub Actions Security Hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
3. [GitLab Security Documentation](https://docs.gitlab.com/ee/user/application_security/)
4. [Jenkins Security Best Practices](https://www.jenkins.io/doc/book/security/)
5. [SLSA Framework](https://slsa.dev/)
6. [Sigstore](https://www.sigstore.dev/)

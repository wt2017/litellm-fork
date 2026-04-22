# LiteLLM Helm 部署指南（自定义 Alpine 镜像）

本指南基于 `deploy/charts/litellm-helm/` 官方 Helm Chart，使用自行编译的 `litellm-v1.83.3-stable:alpine` 容器镜像部署 LiteLLM Proxy。

---

## 前置条件

- Kubernetes 1.21+
- Helm 3.8.0+
- Docker 构建环境（用于编译镜像）
- 可访问的容器镜像仓库（Harbor、Docker Hub、自建 Registry 等）
- PV provisioner（如需 Chart 内嵌的 PostgreSQL）

---

## 1. 构建并推送自定义镜像

项目已提供 Alpine 版本的 Dockerfile，直接使用即可。

```bash
cd /path/to/litellm

# 构建镜像
docker build \
  -f docker/Dockerfile.alpine \
  -t <YOUR_REGISTRY>/litellm-v1.83.3-stable:alpine \
  .

# 推送到仓库
docker push <YOUR_REGISTRY>/litellm-v1.83.3-stable:alpine
```

---

## 2. 创建必需的 Secret

在部署前，先创建存放敏感信息的 Secret：

```bash
kubectl create namespace litellm

# 存放 LLM Provider 的 API Key（示例为 OpenAI）
kubectl -n litellm create secret generic litellm-env-secrets \
  --from-literal=OPENAI_API_KEY=sk-xxx

# 生产环境使用外部 PostgreSQL 时，创建数据库凭据 Secret
kubectl -n litellm create secret generic litellm-postgres-secret \
  --from-literal=username=litellm \
  --from-literal=password=YourStrongPassword

# 若镜像仓库需要拉取认证
kubectl -n litellm create secret docker-registry regcred \
  --docker-server=<YOUR_REGISTRY> \
  --docker-username=<USERNAME> \
  --docker-password=<PASSWORD>
```

---

## 3. 部署 Helm Chart

直接复用本地 `deploy/charts/litellm-helm/` 目录：

```bash
cd deploy/charts/litellm-helm

# 下载子 Chart 依赖（postgresql、redis 等）
helm dependency update

# 安装 / 升级（使用生产 values）
helm -n litellm upgrade --install litellm . \
  -f custom-values.production.yaml
```

---

## 4. 验证部署

```bash
# 查看 Pod 状态
kubectl -n litellm get pods

# 查看代理日志
kubectl -n litellm logs -f deployment/litellm

# 获取 Master Key（若使用自动生成）
kubectl -n litellm get secret litellm-masterkey \
  -o jsonpath="{.data.masterkey}" | base64 -d
```

---

## 5. 访问 Admin UI

浏览器打开 Ingress 配置的域名（如 `https://litellm.yourdomain.com`）。

- **Proxy Endpoint**: `http://litellm:4000`（Pod 内部访问 Service）
- **Proxy Key**: 上述命令获取的 master key

---

## 关键配置说明

| 配置项 | 说明 |
|--------|------|
| `image.repository` / `tag` | 指向自编译镜像，覆盖默认官方镜像 |
| `db.deployStandalone` | `true` 时由子 Chart 启动 PostgreSQL；**生产环境务必设为 `false` 并配置外部数据库** |
| `db.useExisting` | 使用外部已有 PostgreSQL，配合 `db.secret` 传入凭据 |
| `proxy_config` | 渲染为 `/etc/litellm/config.yaml`，支持所有 LiteLLM 代理配置项 |
| `migrationJob.enabled` | 升级时自动执行 Prisma 数据库迁移，建议保持启用 |
| `environmentSecrets` | 将 Secret 的键值对作为环境变量注入 Pod，用于传递 API Key |
| `separateHealthApp` | 将健康检查端口与业务端口分离，避免高负载时探针误杀 |
| `pdb.enabled` | 启用 PodDisruptionBudget，保证滚动更新/节点驱逐时至少保留指定副本 |
| `autoscaling.enabled` | 启用 HPA，根据 CPU/内存自动扩缩容 |

---

## 附录：生产环境 values 文件

以下提供一份完整的生产环境 `custom-values.production.yaml` 参考配置。

```yaml
# ==========================================
# 名称与镜像配置
# ==========================================
nameOverride: "litellm"

image:
  repository: <YOUR_REGISTRY>/litellm-v1.83.3-stable
  tag: alpine
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: regcred

# ==========================================
# 高可用与扩缩容
# ==========================================
replicaCount: 3

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

pdb:
  enabled: true
  minAvailable: 2

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1

# ==========================================
# 资源限制
# ==========================================
resources:
  limits:
    cpu: "4"
    memory: 4Gi
  requests:
    cpu: "2"
    memory: 2Gi

# ==========================================
# 健康检查分离（避免高负载误杀）
# ==========================================
separateHealthApp: true
separateHealthPort: 8081

livenessProbe:
  path: /health/liveliness
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  path: /health/readiness
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

startupProbe:
  path: /health/readiness
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 30

terminationGracePeriodSeconds: 120

# ==========================================
# 外部数据库配置（生产必选项）
# ==========================================
db:
  deployStandalone: false
  useExisting: true
  endpoint: postgres-prod.your-company.svc.cluster.local
  database: litellm
  url: postgresql://$(DATABASE_USERNAME):$(DATABASE_PASSWORD)@$(DATABASE_HOST)/$(DATABASE_NAME)?sslmode=require
  secret:
    name: litellm-postgres-secret
    usernameKey: username
    passwordKey: password
    endpointKey: ""

# ==========================================
# 可选：Redis 缓存
# ==========================================
redis:
  enabled: false
  architecture: standalone

# 如果使用外部 Redis，可通过 environmentSecrets 注入：
# REDIS_HOST / REDIS_PORT / REDIS_PASSWORD / REDIS_URL

# ==========================================
# LiteLLM 代理配置
# ==========================================
proxyConfigMap:
  create: true
  key: "config.yaml"

proxy_config:
  model_list:
    - model_name: gpt-4
      litellm_params:
        model: openai/gpt-4
        api_key: os.environ/OPENAI_API_KEY
    - model_name: gpt-4o
      litellm_params:
        model: openai/gpt-4o
        api_key: os.environ/OPENAI_API_KEY
  general_settings:
    master_key: os.environ/PROXY_MASTER_KEY
    # DATABASE_URL 环境变量已由 Deployment 自动注入，无需在此重复声明
    # 生产环境建议开启的使用限制示例：
    # max_parallel_requests: 100
    # request_timeout: 600

# ==========================================
# 环境变量注入（Provider API Keys 等）
# ==========================================
environmentSecrets:
  - litellm-env-secrets

# ==========================================
# Service / Ingress
# ==========================================
service:
  type: ClusterIP
  port: 4000

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
  hosts:
    - host: litellm.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: litellm-tls
      hosts:
        - litellm.yourdomain.com

# ==========================================
# Master Key 管理
# ==========================================
# 生产环境强烈建议预先创建 Secret 并指定：
# masterkeySecretName: "litellm-masterkey"
# masterkeySecretKey: "masterkey"

# ==========================================
# 安全与调度
# ==========================================
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: litellm

nodeSelector: {}
tolerations: []
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - litellm
          topologyKey: kubernetes.io/hostname

# ==========================================
# 数据库迁移 Job
# ==========================================
migrationJob:
  enabled: true
  backoffLimit: 4
  ttlSecondsAfterFinished: 300
  disableSchemaUpdate: false
  hooks:
    argocd:
      enabled: true
    helm:
      enabled: false
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

# ==========================================
# 可观测性
# ==========================================
serviceMonitor:
  enabled: false
  labels: {}
  interval: 15s
  scrapeTimeout: 10s

# ==========================================
# 额外挂载 / 容器（如有需要）
# ==========================================
volumes: []
volumeMounts: []
extraContainers: []
extraEnvVars: {}
extraResources: []
```

---

## 注意事项

1. **数据库**: 生产环境请使用托管 PostgreSQL（如 AWS RDS、Cloud SQL、Azure Database）或自行维护的高可用集群，不要使用 `db.deployStandalone`。Helm 子 Chart 部署的 PostgreSQL 不提供高可用和自动备份。
2. **Master Key**: 生产环境务必预先创建 `litellm-masterkey` Secret 并通过 `masterkeySecretName` 引用，避免每次部署生成随机密钥导致旧 API Key 失效。
3. **安全**: `readOnlyRootFilesystem: true` 需要配合 `securityContext` 中的 `emptyDir` 挂载。当前 Chart 模板已自动处理 `/tmp`、`/.cache`、`/.npm` 的临时卷挂载。
4. **Prisma 迁移**: `litellm-proxy-extras` 会在启动时执行 `prisma migrate deploy`。如果使用外部数据库，请确保数据库用户具备 `CREATE`、`ALTER` 权限；如果由 CI/CD 统一管理迁移，可将 `migrationJob.disableSchemaUpdate` 设为 `true`。

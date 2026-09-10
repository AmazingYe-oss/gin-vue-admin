# gin-vue-admin 云原生交付与可观测性实践

基于 [flipped-aurora/gin-vue-admin](https://github.com/flipped-aurora/gin-vue-admin)（MIT License）的个人全链路运维实战项目：以真实的 Gin + Vue3 全栈应用为载体，完整落地 **多阶段容器化 → GitHub Actions CI → 阿里云 ACR → ArgoCD GitOps 持续交付 → Prometheus & Alertmanager 监控告警** 的生产级云原生交付闭环。

本项目坚持“配置即代码”与手写验证原则，彻底清空上游原有容器配置，每一步独立完成构建、排错与工程沉淀。

---

## 🏗️ 全链路架构拓扑流转

```text
+----------------------------------------------------------------------------------------------------------------+
| 1. 源码与 CI 构建阶段 (App Repo: gin-vue-admin)                                                                 |
|   [开发代码提交/Tag] ---> [GitHub Actions 流水线]                                                               |
|                              |---> 多阶段构建瘦身 (Alpine 基础 + CGO=0 静态裁切, 164MB -> 57.7MB, 缩减 65%)       |
|                              |---> 推送镜像至 [阿里云 ACR 镜像仓库] (gva-server / gva-web)                     |
|                              +---> 自动提交并回写最新 Tag 至 GitOps 仓库 (values.yaml)                          |
+-------------------------------------------------------|--------------------------------------------------------+
                                                        | (自动回写 Git 声明)
                                                        v
+----------------------------------------------------------------------------------------------------------------+
| 2. 声明式持续交付阶段 (GitOps Repo: gin-vue-admin-gitops)                                                      |
|   [Helm Chart 参数化清单] <=== [ArgoCD 持续交付引擎]                                                             |
|                                 |---> Automated Sync: 监听 Git 变动自动触发发布                                 |
|                                 |---> Self-Heal: 3分钟内自动纠偏集群手动修改 (防配置漂移)                         |
|                                 +---> Prune: 自动级联清理被删除的孤儿资源                                       |
+-------------------------------------------------------|--------------------------------------------------------+
                                                        | (拉取配置并声明式渲染)
                                                        v
+----------------------------------------------------------------------------------------------------------------+
| 3. Kubernetes 运行时状态 (K3s 单节点集群)                                                                      |
|                                                                                                                |
|  [Namespace: gva] 业务负载栈                                                                                   |
|    +-- web: Deployment + NodePort Service (:30080)                                                             |
|    |     `-- Nginx 1.27 (try_files SPA 路由回退 + ^~ /api/ 前缀反代剥离)                                       |
|    |                                                                                                           |
|    `-- server: Deployment + ClusterIP Service (:8888)                                                          |
|          +-- Gin 原生集成 promhttp 暴露 /metrics 接口                                                           |
|          +-- 注入 ConfigMap 只读配置 (config.docker.yaml:ro)                                                   |
|          `-- 挂载 PVC (SQLite 数据文件) -> securityContext.fsGroup=1000 解决非 root 权限冲突                   |
|                                                                                                                |
|  [Namespace: monitoring] 可观测性基建 (kube-prometheus-stack)                                                  |
|    +-- ServiceMonitor: matchLabels 自动关联 app=gva-server，动态注册抓取目标                                    |
|    +-- Prometheus: 采集运行时与业务指标，执行 PrometheusRule 规则评估                                           |
|    |     |-- GvaServerDown: up{job="server"} == 0 (for: 1m, critical)                                          |
|    |     `-- GvaServerAbsent: absent(up{job="server"}) == 1 (for: 1m, labels: {namespace: gva})                |
|    +-- Alertmanager: AlertmanagerConfig 按 severity=critical 进行分组抑制与路由分发                            |
|    `-- Webhook Receiver: 接收 Firing (故障触发) 与 Resolved (故障恢复) POST 通知                               |
+----------------------------------------------------------------------------------------------------------------+
```

---

## 🌟 核心交付成果与工程亮点

### 1. 多阶段构建与极致镜像瘦身
- **后端（server/Dockerfile）**：采用 `golang:1.24-alpine` 编译（`CGO_ENABLED=0` + `-ldflags="-s -w"`）→ `alpine:3.20` 极简运行时；固定 `uid=1000` 非 root 运行对齐挂载属主；引入 `tini` 作为 PID 1 处理孤儿进程与信号转发。镜像体积由官方基础的 164MB 压缩至 **57.7MB（-65%）**。
- **前端（web/Dockerfile）**：采用 `node:20.19.0-alpine` 确定性构建 → `nginx:1.27-alpine` 托管静态资源，产物镜像仅 **24.3MB**。
- **Nginx 反代与路由分发**：配置 `try_files` 支持 SPA 路由回退，精确配置 `^~ /api/` 前缀剥离与长匹配分拣，解决生产环境 404/502 路径异常。

### 2. 自动化 CI/CD 流水线（GitHub Actions）
- 监听应用仓库 Tag/Release 触发流水线；
- 自动化执行前后端 Docker 镜像构建，并安全推流至阿里云 ACR 镜像仓库；
- 运行自动化发布脚本，利用 GitHub Personal Access Token 自动回写并提交最新镜像 Tag 至 GitOps 清单仓库，实现零人工介入的端到端自动化。

### 3. 原生声明式打点与白盒监控
- 后端服务内置集成 `promhttp` 暴露原生 Go Runtime 与 HTTP 业务指标接口（`/metrics`）；
- 剥离上游臃肿依赖，统一接口路由分拣，保障探针与采集探针的高可用访问。

---

## 🎯 核心面试高频深挖与自查速查

| 考察维度 | 面试官核心追问 | 你的标准答案与实战证据 |
|---|---|---|
| **1. 架构定位** | 为什么设计双仓库而不是单仓库？ | **应用仓**关注源码构建与单元测试；**GitOps 仓**声明集群期望状态并作为唯一事实源。隔离 CI 与 CD 权限，避免构建权限直接污染集群生产配置。 |
| **2. 镜像瘦身** | 怎么把后端镜像压到 57.7MB（-65%）的？ | 多阶段构建（`golang:1.24-alpine` 编译 + `alpine:3.20` 运行）；`CGO_ENABLED=0` 静态编译并加 `-ldflags="-s -w"` 剔除调试符号；`.dockerignore` 排除无用资源。 |
| **3. 权限与存储** | SQLite 挂载 PVC 报 `permission denied` 怎么解？ | 镜像非 root 运行（`uid=1000`），但 K8s 默认挂载卷属主为 root。在 Pod `securityContext` 注入 `fsGroup: 1000`，使挂载点目录属组归属当前容器用户。 |
| **4. 平滑接管** | 存量手写资源怎么切到 Helm 管理且零中断？ | 利用 Helm **Resource Adoption** 机制：在存量资源（Service/Deployment）打上 `app.kubernetes.io/managed-by: Helm` 及 Release 元数据注解，执行 `helm install` 直接接管。 |
| **5. 路由与代理** | Nginx 反代为什么用 `^~ /api/`？尾斜杠有什么坑？ | `^~` 提高前缀匹配优先级，防止被后续通用正则匹配截胡；`proxy_pass http://server:8888/` 结尾带斜杠会自动剥离 `/api/` 前缀，对齐后端原生路由。 |
| **6. GitOps 防漂移** | 有人在集群里手动改副本数会发生什么？ | ArgoCD 开启了 **Self-Heal** 机制，检测到集群实时状态与 Git 期望状态不一致时，3 分钟内自动触发同步，覆盖手动修改，强制回滚。 |
| **7. 监控发现** | ServiceMonitor 是怎么把目标注册进 Prometheus 的？ | ServiceMonitor 通过 `matchLabels: app=gva-server` 匹配对应的 Service，Prometheus Operator 监听该 CR 并自动在 Prometheus 生成 `job="server"` 抓取任务。 |
| **8. 告警盲区** | 为什么用了 `up == 0` 还要加 `absent()` 规则？ | 当 Pod 被彻底删除或副本缩容为 0 时，时序数据不复存在，`up == 0` 不会触发（因为没有数据可比）。必须补充 `absent(up{job="server"}) == 1` 覆盖时序消失场景。 |
| **9. 告警丢失** | `absent()` 产生的告警收不到通知，排查路径是什么？ | `absent()` 执行后会剥离原指标标签，导致匹配不到子路由匹配器被抛弃。修复方案是在 PrometheusRule 的 `labels` 中显式补齐 `namespace: gva`。 |
| **10. 故障演练** | 整个告警闭环是如何验证的？ | Git 提交 `replicas: 0` 模拟故障 → Prometheus 出现 Pending 并在 1 分钟后转为 Firing → Alertmanager 路由至 Webhook 收到 Firing POST → 改回 `replicas: 1` 恢复后收到 Resolved POST。 |

---

## 🔗 相关仓库与交付清单

- **应用源码仓库**：[AmazingYe-oss/gin-vue-admin](https://github.com/AmazingYe-oss/gin-vue-admin)
- **GitOps 配置仓库**：[AmazingYe-oss/gin-vue-admin-gitops](https://github.com/AmazingYe-oss/gin-vue-admin-gitops)

---

## 🚀 本地快速启动（Docker Compose）

```bash
# 克隆仓库
git clone https://github.com/AmazingYe-oss/gin-vue-admin.git
cd gin-vue-admin

# 本地快速构建并编排启动
cd server && docker build -t gva-server:v1 .
cd ../web && docker build -t gva-web:v1 .
cd .. && docker compose up -d

# 访问测试
# 前端访问：http://localhost:8080
# 后端健康检查：http://localhost:8888/metrics
```

---

## 📄 开源协议
本项目基于 [MIT License](LICENSE) 进行二次开发与运维交付实践，遵循开源社区规范。

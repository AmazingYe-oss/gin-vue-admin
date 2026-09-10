# gin-vue-admin 云原生交付与可观测性实践

基于 [flipped-aurora/gin-vue-admin](https://github.com/flipped-aurora/gin-vue-admin)（MIT License）的个人全链路运维实战项目：以真实的 Gin + Vue3 全栈应用为载体，完整落地 **多阶段容器化 → GitHub Actions CI → 阿里云 ACR → ArgoCD GitOps 持续交付 → Prometheus & Alertmanager 监控告警** 的生产级云原生交付闭环。

本项目坚持“配置即代码”与手写验证原则，彻底清空上游原有容器配置，每一步独立完成构建、排错与工程沉淀。

---

## 🏗️ 架构拓扑与技术栈

- **后端/前端**：Go 1.24 (Gin) + Vue3 (Vite) + Nginx 1.27
- **容器与编排**：Docker (多阶段构建) / k3s (Kubernetes v1.30+) / Helm v3
- **CI/CD & GitOps**：GitHub Actions + 阿里云容器镜像服务 (ACR) + ArgoCD
- **可观测性体系**：kube-prometheus-stack (Prometheus + Alertmanager + Grafana + ServiceMonitor)

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

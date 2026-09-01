# gin-vue-admin 云原生交付实践

基于 [flipped-aurora/gin-vue-admin](https://github.com/flipped-aurora/gin-vue-admin)（MIT License）的个人运维实战项目：以一个真实的 Gin + Vue3 全栈应用为载体，完整实践 **容器化 → 本地编排 → K8s 部署 → CI/CD → GitOps → 可观测性** 的运维交付链路。

与"跑通官方 docker 一键部署"不同，本项目刻意清空上游全部 Docker/K8s 相关文件，每一步手写并亲手验证（含挂载卷权限、代理路径、构建网络等真实故障的独立排查记录）。

## 技术栈

`Go 1.24 (Gin)` · `Vue3 + Vite` · `Docker / docker-compose` · `Nginx` · `k3s` · `GitHub Actions` · `阿里云 ACR` · `ArgoCD` · `Prometheus`

## 已完成

### 1. 后端镜像（server/Dockerfile）
- 多阶段构建：`golang:1.24-alpine` 编译（`CGO_ENABLED=0` + `-s -w` 静态裁剪）→ `alpine:3.20` 运行
- `go.mod` 先行 COPY 利用层缓存；GOPROXY 国内源
- 非 root 运行，且**固定 uid=1000**（对齐挂载卷属主，避免权限故障）
- `tini` 作为 PID 1 处理信号转发，保证优雅退出

### 2. 运行时配置与数据外置
- 镜像零状态化：`.dockerignore` 排除配置/数据/日志，同一镜像多环境复用
- `config.docker.yaml` 只读挂载注入（`:ro`），SQLite 数据与日志 volume 外置持久化

### 3. 前端镜像（web/Dockerfile + nginx.conf）
- 多阶段：`node:20.19.0-alpine` 内 `npm ci --replace-registry-host` 确定性构建 → `nginx:1.27-alpine` 仅托管 `dist` 静态产物
- nginx 反代：`try_files` SPA 路由回退、`^~ /api/` 前缀代理 + 尾斜杠剥前缀（经真实 404 故障验证分层定位法）

### 4. docker-compose 本地编排
- server + web 双服务，自定义网络内服务名 DNS 互访
- healthcheck 就绪探测、配置/数据/日志挂载、`depends_on` 启动顺序

## 进行中

- k3s 集群部署（K8s manifests）
- GitHub Actions 流水线：构建镜像推送 ACR → 回写 GitOps 仓库 → ArgoCD 自动同步
- Prometheus + AlertManager 监控告警链路

> GitOps 清单仓库：[AmazingYe-oss/gin-vue-admin-gitops](https://github.com/AmazingYe-oss/gin-vue-admin-gitops)

## 本地运行

```bash
cd server && docker build -t gva-server:v1 .
cd web    && docker build -t gva-web:v1 .
docker compose up -d          # 前端 http://localhost:8080（默认管理员账号见初始化向导）
```

## License

本项目基于 [MIT License](LICENSE) 的 gin-vue-admin 进行二次开发实践，上游版权见 LICENSE 文件。

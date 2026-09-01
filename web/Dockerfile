# ============ 第一阶段：构建阶段 ============
# 使用 Node 20 的 Alpine 镜像，并指定国内 DaoCloud 镜像源加速拉取
FROM docker.m.daocloud.io/node:20.19.0-alpine AS builder

WORKDIR /app

# 【关键】设置 npm 使用国内镜像源，解决国内构建超时问题
ENV NPM_CONFIG_REGISTRY=https://registry.npmmirror.com/

# 先复制依赖清单，利用 Docker 缓存机制（只要依赖没变，就不会重新下载）
COPY package.json package-lock.json* ./

# 使用 npm ci 进行确定性安装，速度更快且更稳定
RUN npm ci --replace-registry-host=npmjs


# 复制全部源码
COPY . .

# 执行构建命令（根据你的实际项目调整，如 npm run build）
RUN npm run build

# ============ 第二阶段：运行阶段 ============
# 使用轻量级 Nginx Alpine 镜像（同样使用国内加速源）
FROM docker.m.daocloud.io/nginx:1.27-alpine

# 将构建好的静态文件复制到 Nginx 的默认网站根目录
COPY --from=builder /app/dist /usr/share/nginx/html

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

# 以非守护进程模式启动 Nginx
CMD ["nginx", "-g", "daemon off;"]
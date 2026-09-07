# ============ 第一阶段：构建 ============
FROM golang:1.24-alpine AS builder

WORKDIR /app

ENV GOPROXY=https://goproxy.cn,https://proxy.golang.org,direct
ENV GOTOOLCHAIN=local

# 先复制依赖文件，利用 Docker 缓存层
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# 复制源码并编译为纯静态二进制
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64     go build -ldflags="-s -w -extldflags '-static'"     -o /out/gva-server .

# ============ 第二阶段：运行 ============
FROM alpine:3.20

# 换阿里云源（sed 替换官方源域名）
RUN sed -i 's#https://dl-cdn.alpinelinux.org#https://mirrors.aliyun.com#g' /etc/apk/repositories     && apk add --no-cache ca-certificates tzdata tini     && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime     && echo "Asia/Shanghai" > /etc/timezone

# 创建非 root 用户
RUN addgroup -g 1000 app && adduser -u 1000 -S -G app app

WORKDIR /app

COPY --from=builder /out/gva-server ./gva-server

RUN chown -R app:app /app
USER app

EXPOSE 8888

ENTRYPOINT ["tini", "--", "/app/gva-server"]
CMD ["-c", "/app/config/config.yaml"]

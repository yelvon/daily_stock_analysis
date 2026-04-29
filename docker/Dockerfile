# syntax=docker/dockerfile:1
# ===================================
# A股自选股智能分析系统 - Docker 镜像
# ===================================
# 多阶段构建：前端打包 + 后端运行
#
# 需要 BuildKit（Docker Desktop 默认开启；旧环境可设 DOCKER_BUILDKIT=1）
# 使用缓存挂载加速重复构建：npm / pip / apt 下载可跨构建复用

FROM node:20-slim AS web-builder

WORKDIR /app/apps/dsa-web

COPY apps/dsa-web/package.json apps/dsa-web/package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

COPY apps/dsa-web/ ./
RUN npm run build

# Pin to bookworm: wkhtmltopdf was removed from Debian testing (2025)
FROM python:3.11-slim-bookworm

# 国内等网络环境下 apt 访问 deb.debian.org 易失败：
#   - 默认 mirrors.aliyun.com（与 compose 默认一致）
#   - 使用官方源：--build-arg APT_MIRROR_HOST=official
ARG APT_MIRROR_HOST=mirrors.aliyun.com
# 设置工作目录
WORKDIR /app

# 设置时区为上海
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 安装系统依赖（wkhtmltopdf 含 wkhtmltoimage，用于 Markdown 转图片 Issue #289）
# apt 缓存挂载：重复 build 时不必反复下载数百 MB 的 .deb
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    set -eux; \
    if [ "$APT_MIRROR_HOST" != "official" ]; then \
      for f in /etc/apt/sources.list.d/debian.sources /etc/apt/sources.list; do \
        if [ -f "$f" ]; then \
          sed -i "s/deb.debian.org/${APT_MIRROR_HOST}/g" "$f"; \
          sed -i "s/security.debian.org/${APT_MIRROR_HOST}/g" "$f"; \
        fi; \
      done; \
    fi; \
    apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    curl \
    wkhtmltopdf \
    fontconfig \
    libjpeg62-turbo \
    libxrender1 \
    libxext6

# 未在层内 rm /var/lib/apt/lists：与 BuildKit 的 apt lists 缓存挂载配合，避免每次构建清空缓存

# 复制依赖文件
COPY requirements.txt .

# pip 缓存挂载：重复 build 时复用 wheel/下载（去掉 --no-cache-dir 才能写入缓存目录）
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# 复制应用代码
COPY *.py ./
COPY api/ ./api/
COPY data_provider/ ./data_provider/
COPY bot/ ./bot/
COPY patch/ ./patch/
COPY src/ ./src/
COPY strategies/ ./strategies/
COPY --from=web-builder /app/static ./static/

# 创建数据目录
RUN mkdir -p /app/data /app/logs /app/reports

# 设置环境变量默认值
ENV PYTHONUNBUFFERED=1
ENV LOG_DIR=/app/logs
ENV DATABASE_PATH=/app/data/stock_analysis.db
# Web/API service
ENV WEBUI_HOST=0.0.0.0
ENV API_PORT=8000

# 暴露 API 端口
EXPOSE 8000

# 数据卷（持久化数据）
VOLUME ["/app/data", "/app/logs", "/app/reports"]

# 健康检查（FastAPI 模式）
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/api/health || curl -f http://localhost:8000/health \
    || python -c "import sys; sys.exit(0)"

# 默认命令（可被覆盖）
CMD ["python", "main.py", "--schedule"]

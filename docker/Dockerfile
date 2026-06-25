# ===================================
# A股自选股智能分析系统 - Docker 镜像
# ===================================
# 多阶段构建：前端打包 + 后端运行

FROM node:20-slim AS web-builder

ARG NPM_REGISTRY=https://registry.npmmirror.com

WORKDIR /app/apps/dsa-web

COPY apps/dsa-web/package.json apps/dsa-web/package-lock.json ./
RUN npm config set registry ${NPM_REGISTRY} \
    && npm config set fetch-timeout 600000 \
    && npm config set fetch-retries 5 \
    && npm ci --no-audit --fund=false

COPY apps/dsa-web/ ./
RUN npm run build

# Pin to bookworm: wkhtmltopdf was removed from Debian testing (2025)
FROM python:3.11-slim-bookworm

ARG PIP_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple
ARG PIP_EXTRA_INDEX_URL=
ARG PIP_TRUSTED_HOST=pypi.tuna.tsinghua.edu.cn

# 设置工作目录
WORKDIR /app

# 设置时区为上海
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Use TUNA mirror for faster and more reliable production builds in China.
RUN sed -i 's|deb.debian.org|mirrors.tuna.tsinghua.edu.cn|g' /etc/apt/sources.list.d/debian.sources 2>/dev/null; \
    sed -i 's|deb.debian.org|mirrors.tuna.tsinghua.edu.cn|g' /etc/apt/sources.list 2>/dev/null; \
    true

# 安装系统依赖（wkhtmltopdf 含 wkhtmltoimage，用于 Markdown 转图片 Issue #289）
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    curl \
    git \
    gosu \
    wkhtmltopdf \
    fontconfig \
    libjpeg62-turbo \
    libxrender1 \
    libxext6 \
    && rm -rf /var/lib/apt/lists/*

# 创建非 root 用户 (UID 1000)
RUN groupadd -g 1000 dsa && \
    useradd -u 1000 -g dsa -m dsa

# 复制依赖文件
COPY requirements.txt .

# 安装 Python 依赖
# AlphaSift is installed from GitHub. In restricted production networks, pip's
# isolated build env may fail to fetch build deps such as setuptools even though
# the base image already ships a compatible setuptools version.
RUN set -eux; \
    pip_args="--index-url ${PIP_INDEX_URL} --trusted-host ${PIP_TRUSTED_HOST} --timeout 120 --retries 5"; \
    if [ -n "${PIP_EXTRA_INDEX_URL}" ]; then pip_args="$pip_args --extra-index-url ${PIP_EXTRA_INDEX_URL}"; fi; \
    pip install --no-cache-dir --no-build-isolation -r requirements.txt $pip_args

# 复制应用代码
COPY *.py ./
COPY api/ ./api/
COPY data_provider/ ./data_provider/
COPY bot/ ./bot/
COPY src/ ./src/
COPY strategies/ ./strategies/
COPY --from=web-builder /app/static ./static/
COPY docker/entrypoint.sh /usr/local/bin/docker-entrypoint.sh

# Verify the bundled AlphaSift adapter from requirements.
RUN python -c "import alphasift.dsa_adapter"

# 确保数据目录存在并授权给 non-root 用户
RUN mkdir -p /app/data /app/logs /app/reports && \
    chown -R dsa:dsa /app && \
    chmod +x /usr/local/bin/docker-entrypoint.sh

# efinance 将搜索缓存（search-cache.json）写入自身安装包内的 data 目录，
# 该目录默认归 root 所有；容器以非 root 用户 dsa 运行时写入会 PermissionError，
# 导致 efinance 数据源每次抓取 A 股数据都失败并降级。预置目录属主给 dsa 修复此问题。
# 动态解析路径，避免写死 Python 版本目录。
RUN EF_DATA_DIR="$(python -c 'import efinance, pathlib; print(pathlib.Path(efinance.__file__).parent / "data")')" && \
    mkdir -p "$EF_DATA_DIR" && \
    chown -R dsa:dsa "$EF_DATA_DIR"

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

# 启动入口先修复 bind mount 目录权限，再降权为 dsa 执行应用；
# 文档化 docker-compose exec 命令需显式使用 -u dsa，避免手动进程写入 root-owned 文件。
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

# 默认命令（可被覆盖）
CMD ["python", "main.py", "--schedule"]

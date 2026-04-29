# Docker 拉取镜像超时 / `auth.docker.io` 连接失败

若构建时出现类似报错：

- `failed to fetch oauth token: Post "https://auth.docker.io/token": dial tcp ... i/o timeout`
- `failed to resolve source metadata for docker.io/library/python:...`

说明本机到 **Docker Hub** 的网络不稳定或被阻断（国内网络较常见），需要在 **Docker Desktop** 侧配置镜像加速或代理，而不是改项目代码。

## 方式一：配置 registry mirror（推荐先试）

用于加速 **`docker.io`（Docker Hub）** 上的基础镜像，例如 `python:3.11-slim-bookworm`、`node:20-slim`。  
这与仓库里 Dockerfile 的 **`APT_MIRROR_HOST`（Debian apt 源）是两件事**：前者解决「拉镜像慢」，后者解决「容器里 `apt-get install` 失败」——在国内建议**两个都配置**。

1. 打开 **Docker Desktop** → **Settings**（设置）→ **Docker Engine**
2. 在 JSON 中增加或合并 `registry-mirrors`。国内建议**写多个地址**，Docker 会按顺序尝试，单站抽风时更容易成功：

```json
{
  "registry-mirrors": [
    "https://docker.xuanyuan.me",
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io"
  ]
}
```

`https://docker.m.daocloud.io` **并非错误**，仍可保留作备用；若你只配了一个站且经常超时，多半是**该镜像站当时不可用**，换成多源或换一个当下社区实测可用的地址即可（公开镜像站会随时间变化，需以你本机 `docker pull hello-world` 实测为准）。

3. 点击 **Apply & Restart**，再执行：

```bash
docker compose -f docker/docker-compose.yml build --no-cache server
```

## 方式二：HTTP/HTTPS 代理

若本机已能科学上网，可在同一 **Docker Engine** JSON 里配置（端口按你本地代理修改）：

```json
{
  "proxies": {
    "http-proxy": "http://127.0.0.1:7890",
    "https-proxy": "http://127.0.0.1:7890"
  }
}
```

或在 `docker/docker-compose.yml` 的 `x-common.environment` 中取消注释 `http_proxy` / `https_proxy`，指向 `host.docker.internal` 上的代理端口。

## 方式三：先在网络正常环境预拉基础镜像

在能访问 Docker Hub 的网络下执行：

```bash
docker pull python:3.11-slim-bookworm
docker pull node:20-slim
```

再在本机 `docker compose ... build`。

---

## apt 安装依赖失败 / `Failed to fetch ... deb.debian.org` / `Connection failed`

若构建在 **`apt-get install`（安装 gcc、wkhtmltopdf 等）** 阶段失败，日志中出现：

- `Err:145 ... gcc-12 ... Connection failed`
- `Unable to fetch some archives`

说明容器内访问 **Debian 官方源**（常经海外 CDN）不稳定。本项目 `docker/Dockerfile` 支持构建参数 **`APT_MIRROR_HOST`**，将 `deb.debian.org` / `security.debian.org` 替换为国内镜像主机名。

**默认行为（已修复）：** `docker/docker-compose.yml` 中 `APT_MIRROR_HOST` 默认 **`mirrors.aliyun.com`**，会替换 `deb.debian.org`，避免国内常见超时。此前若写成 `${APT_MIRROR_HOST:-}`，未配置 `.env` 时会传入**空串**，覆盖 Dockerfile 默认值，导致仍访问 `deb.debian.org`（日志里全是 `deb.debian.org` 即属此类）。

**在 `.env` 中**（仅当需要时）：

- 使用阿里云镜像：可省略不写，或显式 `APT_MIRROR_HOST=mirrors.aliyun.com`
- **强制使用 Debian 官方 CDN**（境外网络等）：`APT_MIRROR_HOST=official`

然后**重新构建**（建议无缓存）：

```bash
docker compose -f docker/docker-compose.yml build --no-cache server
docker compose -f docker/docker-compose.yml up -d server
```

也可单次指定：

```bash
docker compose -f docker/docker-compose.yml build --no-cache --build-arg APT_MIRROR_HOST=mirrors.aliyun.com server
```

---

## 加快构建速度（重复 `docker compose build`）

首次构建仍要拉基础镜像、装 `wkhtmltopdf` 依赖（体积大），**时间不会消失**；以下优化主要让**改代码后的重建**更快。

1. **BuildKit 缓存挂载（已写入 `docker/Dockerfile`）**  
   对 **npm / pip / apt** 使用缓存目录，重复构建时少下依赖包。需 **BuildKit**（Docker Desktop 默认开启；CLI 可设 `export DOCKER_BUILDKIT=1`）。

2. **尽量利用层缓存**  
   未改 `requirements.txt`、未改 `package.json` 时，不要加 `--no-cache`；只改业务代码时，pip/npm/apt 层会直接命中缓存。

3. **缩小构建上下文**  
   根目录 `.dockerignore` 已忽略 `.git/`、`tests/`、`apps/dsa-web/node_modules/`、`apps/dsa-desktop/` 等，减少传给 Docker 守护进程的数据量。

4. **国内网络**  
   同时配置 **Docker Hub** `registry-mirrors` 与 **`APT_MIRROR_HOST`**（见上文），避免拉镜像和 `apt-get` 长时间重试。

5. **首次构建仍慢**  
   `wkhtmltopdf` 会拉大量系统库，属预期；换更快的 Debian 源与 Docker 镜像站是唯一「从源头减负」的办法。

---

与项目无关但常见：终端若提示 `/opt/homebrew/bin/brew: no such file or directory`，说明 `~/.zprofile` 等里配置了 Homebrew 路径但本机未安装，可注释对应行或安装 Homebrew。

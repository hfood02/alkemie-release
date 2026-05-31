# Alkemie HPC 运行包

本运行包包含服务程序、runner、前端文件、配置模板和启动脚本。
Python 依赖由部署环境的 Conda 环境安装。

## 内容

```text
server/
  alkemie-server
  build/
  data/
  agents/knowledge_base/
  skills/
  shared/
bin/
  alkemie-runner
scripts/
  install-deps.sh
  check-env.sh
  run.sh
  start.sh
  start-nginx.sh
  stop-nginx.sh
  stop.sh
  healthcheck.sh
requirements.txt
environment.yml
.env.example
wheelhouse/   # 可选，离线 Python 依赖
model-cache/
logs/
workspaces/
```

## 系统要求

- Linux x86_64。
- glibc 2.17 或更高版本，通常对应 CentOS 7 及以上。
- Conda 环境，Python 主次版本需要与运行包一致。

查看服务器 glibc 版本：

```bash
ldd --version
```

## 快速启动

1. 解压运行包。

   ```bash
   unzip alkemie-hpc-nuitka-linux-x86_64.zip
   ```

   解压后再展开运行包：

   ```bash
   tar -xzf alkemie-hpc-nuitka-linux-x86_64-py3.12-latest.tar.gz
   cd alkemie-hpc-release
   ```

2. 创建并进入 Conda 环境。

   ```bash
   conda env create -f environment.yml
   conda activate alkemie
   ```

   `environment.yml` 会优先从 conda-forge 安装 NumPy、SciPy、OpenBLAS 等科学计算依赖。
   不要只执行 `pip install -r requirements.txt` 作为安装步骤。

3. 安装 Python 依赖。

   ```bash
   scripts/install-deps.sh
   ```

   如果现场不能联网，可以把预先下载好的 Python wheel 放到 `wheelhouse/`。
   `scripts/install-deps.sh` 会自动优先使用本地 `wheelhouse/`。

4. 创建配置文件。

   ```bash
   cp .env.example .env
   ```

5. 编辑 `.env`。

   至少修改：

   ```env
   LLM_API_KEY=replace-with-your-llm-api-key
   JWT_SECRET_KEY=replace-with-a-unique-32-plus-character-secret
   ALKEMIE_WORKSPACES_ROOT=/data1/alkemie-share/workspaces
   ```

   默认是单用户模式：

   ```env
   MULTI_USER_MODE=False
   ALKEMIE_EXECUTION_BACKEND=standalone
   ```

   如果启用多用户模式，改为：

   ```env
   MULTI_USER_MODE=True
   DATABASE_URL=postgresql://username:password@db-host:5432/alkemie
   ALKEMIE_ALLOWED_ORIGINS=https://your-domain.example.com
   AUTH_REFRESH_COOKIE_SECURE=true
   ```

6. 检查运行环境。

   ```bash
   scripts/check-env.sh
   ```

   如果启用了 `ALKEMIE_USE_NGINX=true`，检查脚本会确认当前环境可以执行 `nginx`。

7. 准备工作目录。

   ```bash
   mkdir -p /data1/alkemie-share/workspaces
   ```

   这个目录用于项目文件、上传文件、作业脚本、计算结果和日志。使用
   `cluster_runner` 时，平台服务账号和用户 runner 账号都必须能访问它。

8. 可选：放置离线 embedding 模型。

   默认模型是：

   ```text
   sentence-transformers/all-MiniLM-L6-v2
   ```

   如果模型包名为 `alkemie-model-cache-all-MiniLM-L6-v2.tar.gz`：

   ```bash
   mkdir -p model-cache/hub
   tar -xzf alkemie-model-cache-all-MiniLM-L6-v2.tar.gz -C model-cache/hub
   ```

   预期目录：

   ```text
   model-cache/hub/models--sentence-transformers--all-MiniLM-L6-v2
   ```

9. 启动服务。

   ```bash
   scripts/start.sh
   ```

   查看日志：

   ```bash
   tail -f logs/server.log
   ```

   停止服务：

   ```bash
   scripts/stop.sh
   ```

## 访问

默认监听：

```env
ALKEMIE_USE_NGINX=false
ALKEMIE_HOST=0.0.0.0
ALKEMIE_PORT=8443
ALKEMIE_NO_SSL=true
```

这种模式由应用直接提供 HTTP，浏览器访问：

```text
http://<服务节点地址>:8443
```

如果集群不允许从本地电脑直接访问该端口，使用 SSH 隧道：

```bash
ssh -L 8443:localhost:8443 username@hpc.example.edu
```

然后访问：

```text
http://localhost:8443
```

需要用户态反向代理时，修改 `.env`：

```env
ALKEMIE_USE_NGINX=true
ALKEMIE_PORT=8443
ALKEMIE_BACKEND_PORT=8442
ALKEMIE_NGINX_DIR=./run/nginx
ALKEMIE_NO_SSL=true
```

此时 nginx 监听 `ALKEMIE_PORT`，应用只监听
`127.0.0.1:ALKEMIE_BACKEND_PORT`。nginx 的配置、pid、日志和临时目录都在运行包目录下，
不需要 sudo 权限。

用户态 nginx 不能绕过集群网络限制：如果登录节点或服务节点端口没有对本地电脑开放，
仍然需要 SSH 隧道。站点已有 HTTPS 网关时，也可以让站点网关转发到 `ALKEMIE_PORT`。

## Runner 模式

如果需要让每个用户用自己的 HPC 账号提交作业，在 `.env` 启用：

```env
MULTI_USER_MODE=True
ALKEMIE_EXECUTION_BACKEND=cluster_runner
```

用户在自己的登录节点 shell 中运行：

```bash
bin/alkemie-runner bind --server-url https://your-domain.example.com --bind-code <code>
bin/alkemie-runner run --config ~/.alkemie/runner.json --poll-interval 10
```

runner 的 `--allowed-root` 必须覆盖 `.env` 中的 `ALKEMIE_WORKSPACES_ROOT`。

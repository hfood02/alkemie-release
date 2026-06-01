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
   HONPAS_PSEUDO_FOLDER=/data1/alkemie-share/workspaces/honpas_pseudo_library
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
   如果检查脚本提示 Python 版本不匹配，或日志出现 `No module named 'encodings'`，
   通常是没有激活运行包对应的 Conda 环境。先执行 `conda activate alkemie`，再重新检查。
   智能体工作台执行项目脚本时默认使用当前 Conda 环境里的 Python；如需指定其他解释器，
   在 `.env` 设置 `ALKEMIE_PROJECT_PYTHON=/path/to/python`。

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
   cp /path/to/alkemie-model-cache-all-MiniLM-L6-v2.tar.gz .
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

   启动后脚本会打印访问地址。若 `.env` 设置为：

   ```env
   ALKEMIE_USE_NGINX=true
   ALKEMIE_PORT=8443
   ALKEMIE_BACKEND_PORT=8442
   ALKEMIE_NO_SSL=false
   ```

   浏览器访问端口是 `8443`，不是后端内部端口 `8442`。`scripts/start.sh`
   会自动探测节点地址，并按实际协议打印 `Local URL`、`Node URL` 和 SSH 隧道示例。
   如果自动探测的节点地址不对，再设置 `ALKEMIE_PUBLIC_HOST=<服务节点地址或 IP>`。

   查看日志：

   ```bash
   tail -f logs/server.log
   ```

   如果日志出现：

   ```text
   Fatal Python error: init_fs_encoding
   ModuleNotFoundError: No module named 'encodings'
   ```

   先确认已激活 Conda 环境：

   ```bash
   conda activate alkemie
   scripts/check-env.sh
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
ALKEMIE_NO_SSL=false
ALKEMIE_SSL_CERT=./certs/cert.pem
ALKEMIE_SSL_KEY=./certs/key.pem
```

此时 nginx 监听 `ALKEMIE_PORT`，应用只监听
`127.0.0.1:ALKEMIE_BACKEND_PORT`。nginx 的配置、pid、日志和临时目录都在运行包目录下，
不需要 sudo 权限。

`ALKEMIE_NO_SSL=false` 时，nginx 对外提供 HTTPS；后端仍然只在本机 HTTP 端口运行。
如果 `ALKEMIE_SSL_CERT` 和 `ALKEMIE_SSL_KEY` 不存在，`scripts/start.sh` 会自动生成
自签名证书。浏览器首次访问自签名证书时会提示不受信任，测试环境可以手动接受；生产环境
应替换为站点签发的证书。

部分集群或代理会重置直连 HTTP 的 `PUT`/`DELETE` 请求，表现为页面能打开但保存、删除失败。
这种情况下使用上面的 HTTPS nginx 配置，或改用 SSH 隧道访问。用户态 nginx 不能绕过端口
完全不开放的网络限制；如果登录节点或服务节点端口没有对本地电脑开放，仍然需要 SSH 隧道。
站点已有 HTTPS 网关时，也可以让站点网关转发到 `ALKEMIE_PORT`。

`scripts/start.sh` 会自动探测 `Node URL`。只有自动探测的节点地址不对时才需要设置
`ALKEMIE_PUBLIC_HOST`；该值只填写主机名或 IP，不带 `http://`、`https://` 或端口。

`ALKEMIE_ALLOWED_ORIGINS` 在单用户运行包部署下可以留空；`scripts/run.sh` 会根据自动探测
或 `ALKEMIE_PUBLIC_HOST` 覆盖的节点地址、端口和协议，自动加入 localhost 与节点地址来源。
多用户模式必须显式配置，例如 `https://10.251.0.28:8443`，且不能使用 `*`。

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

# Alkemie HPC Nuitka 运行包

本运行包包含 Nuitka 编译后的后端程序、runner、前端构建产物、配置模板和启动脚本。
第三方 Python 依赖由部署环境的 Conda 环境安装。

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

## 快速启动

1. 解压运行包。

   如果拿到的是 GitHub Actions artifact zip：

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

3. 安装并检查 Python 依赖。

   ```bash
   scripts/install-deps.sh
   scripts/check-env.sh
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

6. 准备工作目录。

   ```bash
   mkdir -p /data1/alkemie-share/workspaces
   ```

   这个目录用于项目文件、上传文件、作业脚本、计算结果和日志。使用
   `cluster_runner` 时，平台服务账号和用户 runner 账号都必须能访问它。

7. 可选：放置离线 embedding 模型。

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

8. 启动服务。

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
ALKEMIE_HOST=0.0.0.0
ALKEMIE_PORT=8443
ALKEMIE_NO_SSL=true
```

如果服务部署在登录节点且浏览器无法直接访问，使用 SSH 隧道：

```bash
ssh -L 8443:localhost:8443 username@hpc.example.edu
```

然后访问：

```text
http://localhost:8443
```

运行包默认由应用直接提供普通 HTTP。生产环境建议放在站点已有的反向代理或网关后面，
由外部代理终止 HTTPS。

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

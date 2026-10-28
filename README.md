# Alkemie 2.0

Alkemie 是面向 HPC 的材料科学计算平台。通过浏览器组织计算工作流、管理结构与项目文件、跟踪 PBS/Slurm 作业，并在研究工作台中使用智能助手设计工作流、处理数据和分析结果。

[下载运行包](https://github.com/hfood02/alkemie-release/releases/latest/download/alkemie-hpc-nuitka-linux-x86_64.zip) · [下载离线模型](https://github.com/hfood02/alkemie-release/releases/latest/download/alkemie-model-cache-all-MiniLM-L6-v2.tar.gz) · [查看版本更新](https://github.com/hfood02/alkemie-release/releases/latest)

## 本次更新

- 修复运行包启动时提示缺少 `active_learning`、`autocorrection`、`gateway` 等模块的问题。
- 修复 FAISS 依赖版本检查误报，正确安装 `faiss-cpu 1.9.0.post1` 后可通过该项检查。
- 补充 Windows 下 WSL2 + Ubuntu 的安装、文件存放和浏览器访问说明。

## 系统要求

- Linux x86_64；服务程序以 glibc 2.17 为构建基线。
- Windows 用户通过 **WSL2 + Ubuntu 22.04/24.04 x86_64** 运行同一 Linux 包，再用 Windows 浏览器访问；当前不提供原生 Windows 可执行程序或 ARM64 运行包。
- Conda 和 Python 3.12。较新的 Linux 系统更适合安装当前科学计算依赖；CentOS 7 等旧系统需要使用兼容 glibc 2.17 的 Conda 安装器与依赖包。
- 可写的项目存储目录，以及集群允许使用的服务端口。
- 计算软件及赝势、轨道或势函数库由所在集群提供，并在平台中配置。
- `shared` 和 `cluster` 模式的智能助手执行命令需要管理员配置系统 bubblewrap 和 user namespace；`local` 模式无需额外安装系统 bubblewrap。

运行包已包含网页、服务程序、智能助手运行时、材料知识库和启动脚本，运行服务器无需安装 Node.js。Python 科学计算依赖通过下面的 Conda 环境和安装脚本准备。

## Windows 用户准备 WSL2

如果只访问集群上已部署的 Alkemie，直接使用 Windows 浏览器或下文的 SSH 隧道，无需安装 WSL。
需要在本机运行时，在管理员 PowerShell 中安装 Ubuntu；已有 WSL 的用户先检查发行版和版本：

```powershell
wsl --install -d Ubuntu-22.04
wsl --list --verbose
```

按安装提示重启并创建 Ubuntu 用户，确认列表中的 Ubuntu 的 `VERSION` 为 `2`。已有 WSL1 发行版可运行
`wsl --set-version Ubuntu-22.04 2`，其中发行版名称须与列表一致。安装条件和故障排查见
[Microsoft WSL 安装说明](https://learn.microsoft.com/windows/wsl/install)。

之后所有 `conda`、`tar` 和 `scripts/*.sh` 命令均在 **Ubuntu 终端**执行。Conda 也须安装 Linux x86_64 版本，
不能使用 Windows 的 Conda/Python。先确认 `uname -m` 输出 `x86_64`，再在 Linux 用户目录准备安装位置：

```bash
mkdir -p ~/Alkemie2.0 ~/alkemie-workspaces
cd ~/Alkemie2.0
explorer.exe .
```

将两个下载包复制到打开的目录，然后继续下方安装步骤，在 Ubuntu 内解压。项目和运行包建议保存在 Linux 用户目录，
不要在 Windows 解压后从 `/mnt/c` 运行；参见 [WSL 文件存储建议](https://learn.microsoft.com/windows/wsl/filesystems)。
编辑 `.env` 时使用 UTF-8、LF 换行和 Linux 绝对路径（例如 `/home/yourname/alkemie-workspaces`）。

启动并通过 `scripts/healthcheck.sh` 后，在 Windows 的 Edge/Chrome 打开打印的 `Local URL`，默认是
`https://localhost:8443`；设置 `ALKEMIE_NO_SSL=true` 时使用 `http://localhost:8443`。
WSL 支持从 Windows 经 localhost 访问 Linux 服务，参见 [WSL 网络说明](https://learn.microsoft.com/windows/wsl/networking)。
如果 Ubuntu 内健康检查通过而 Windows 无法打开页面，先检查 Windows 是否占用了该端口，以及 VPN/防火墙的本地访问设置。
关闭或重启 WSL 后，须重新激活 Conda 环境并运行 `scripts/start.sh`。

## 安装与启动

### 1. 下载

下载运行包和离线模型包，放在 Linux 服务器的同一目录。

### 2. 解压并创建环境

```bash
unzip -p alkemie-hpc-nuitka-linux-x86_64.zip '*.tar.gz' | tar -xz
cd alkemie-hpc-release
conda env create -f environment.yml
conda activate alkemie
scripts/install-deps.sh
```

`environment.yml` 和 `scripts/install-deps.sh` 会配合安装科学计算依赖。请完成这两个步骤。离线部署需要提前准备 Conda 包及 Python wheel；将 wheel 放入运行包的 `wheelhouse/` 后，安装脚本会优先使用本地文件。

### 3. 放置离线模型

默认使用 `sentence-transformers/all-MiniLM-L6-v2` 进行本地知识检索。配置模板默认启用离线模式，需要先解压模型缓存：

```bash
mkdir -p model-cache/hub
tar -xzf ../alkemie-model-cache-all-MiniLM-L6-v2.tar.gz -C model-cache/hub
```

解压后应存在 `model-cache/hub/models--sentence-transformers--all-MiniLM-L6-v2/`。离线模型用于知识检索；智能助手的对话仍需要能够访问所配置的模型服务。

### 4. 配置服务

```bash
cp .env.example .env
```

编辑 `.env`，填写模型服务密钥，并将存储路径改为当前账号可写的绝对路径。例如：

```env
LLM_PROVIDER=deepseek
LLM_API_KEY=your-api-key
LLM_BASE_URL=https://api.deepseek.com
LLM_MODEL=deepseek-v4-flash

ALKEMIE_DEPLOYMENT_PROFILE=local
ALKEMIE_WORKSPACES_ROOT=/path/to/alkemie-workspaces
ALKEMIE_BASE_DIR=/path/to/alkemie-workspaces/alkemie-projects
ALKEMIE_UPLOADS_DIR=/path/to/alkemie-workspaces/alkemie-uploads
ALKEMIE_LOGS_DIR=/path/to/alkemie-workspaces/alkemie-logs
ALKEMIE_POTCAR_DIR=/path/to/potcar-library
HONPAS_PSEUDO_FOLDER=/path/to/honpas-pseudo-library
```

使用 ABACUS 或 LASP 时，同时配置模板中的赝势、轨道和势函数路径。项目数据、数据库、密钥和计算结果应存放在运行包之外，便于升级时保留。

模板默认通过用户态 nginx 提供 HTTPS：

```env
ALKEMIE_USE_NGINX=true
ALKEMIE_PORT=8443
ALKEMIE_BACKEND_PORT=8442
ALKEMIE_NO_SSL=false
```

`8443` 是浏览器访问端口，`8442` 是本机后端端口。首次启动时若没有证书，会生成自签名证书；正式使用时可通过 `ALKEMIE_SSL_CERT` 和 `ALKEMIE_SSL_KEY` 配置站点证书。

### 5. 检查并启动

```bash
scripts/check-env.sh
scripts/start.sh
scripts/healthcheck.sh
```

启动脚本会打印浏览器访问地址和 SSH 隧道示例，并启动自动纠错工作进程。在浏览器中打开打印的 `Node URL`，进入平台后配置集群上的计算软件环境。

```bash
# 查看服务日志
tail -f logs/server.log
# 查看自动纠错日志
tail -f logs/correction-worker.log
# 停止服务与工作进程
scripts/stop.sh
```

## 浏览器访问

默认地址为 `https://<服务节点地址>:8443`。自签名证书会触发浏览器信任提示；使用站点证书后可正常验证。

集群端口不对外开放时，在本地电脑建立 SSH 隧道：

```bash
ssh -L 8443:127.0.0.1:8443 username@hpc.example.edu
```

然后访问 `https://localhost:8443`。SSH 目标应为实际运行 Alkemie 的节点；需要经过登录节点时按集群规定配置跳转。

如果启动脚本打印了错误的节点地址，在 `.env` 设置 `ALKEMIE_PUBLIC_HOST=<主机名或 IP>`，不带协议或端口。需要使用 HTTP 时设置 `ALKEMIE_NO_SSL=true`，并使用脚本打印的 HTTP 地址。部分集群网络会阻断直连 HTTP 的保存或删除请求，这种情况下使用 HTTPS 或 SSH 隧道。

## 多用户部署

| 模式 | 适用场景 | 作业执行身份 |
| --- | --- | --- |
| `local` | 单用户运行 | 当前服务账号 |
| `shared` | 多个平台用户共用服务 | 共用服务账号 |
| `cluster` | 平台用户对应独立 Unix 账号 | 通过 Gateway 使用映射账号 |

`shared` 和 `cluster` 模式需要 PostgreSQL，并在 `.env` 配置：

```env
DATABASE_URL=postgresql://username:password@db-host:5432/alkemie
JWT_SECRET_KEY=your-unique-32-plus-character-secret
AUTH_REFRESH_COOKIE_SECURE=true
```

选择 `cluster` 时，还需要显式指定调度器与 MPI 启动方式，例如：

```env
ALKEMIE_DEPLOYMENT_PROFILE=cluster
ALKEMIE_SCHEDULER_TYPE=slurm
ALKEMIE_MPI_LAUNCHER=srun
ALKEMIE_GATEWAY_SIGNING_KEY_FILE=./secrets/gateway-signing-key.pem
```

PBS 集群可设置 `ALKEMIE_SCHEDULER_TYPE=pbs`，并按集群实际环境选择 `mpirun` 或 `mpiexec`。

### Cluster Gateway 安装

由管理员生成签名密钥，配置 `config/privileged-helper.json.example` 中的允许目录、账号规则和项目 Python 路径。允许目录须覆盖 `.env` 中的项目存储目录；项目 Python 必须为所有映射账号可执行的绝对路径，并具备项目脚本所需依赖。

```bash
mkdir -p secrets
umask 077
openssl genpkey -algorithm ED25519 -out secrets/gateway-signing-key.pem
openssl pkey -in secrets/gateway-signing-key.pem -pubout \
  -out secrets/gateway-signing-key.pub
cp config/privileged-helper.json.example config/privileged-helper.json
```

审核配置后安装完整 helper 目录：

```bash
sudo env \
  ALKEMIE_GATEWAY_SERVICE_USER=alkemie-gateway \
  ALKEMIE_HELPER_BUNDLE_SOURCE="$PWD/libexec/alkemie-privileged-helper" \
  ALKEMIE_HELPER_CONFIG_SOURCE="$PWD/config/privileged-helper.json" \
  ALKEMIE_HELPER_INSTALL_DIR=/usr/local/libexec/alkemie-privileged-helper \
  ALKEMIE_HELPER_EXECUTABLE_NAME=alkemie-privileged-helper \
  ALKEMIE_HELPER_PUBLIC_KEY_SOURCE="$PWD/secrets/gateway-signing-key.pub" \
  ALKEMIE_HELPER_PUBLIC_KEY_INSTALL_PATH=/etc/alkemie/gateway-signing-key.pub \
  ALKEMIE_HELPER_SUDOERS_PATH=/etc/sudoers.d/alkemie-privileged-helper \
  scripts/install-helper.sh
```

在平台后台配置 Unix 账号映射并创建 Gateway 注册码，由 Gateway 服务账号完成注册：

```bash
bin/alkemie-cluster-gateway enroll \
  --server-url https://your-domain.example.com \
  --enrollment-code <code> \
  --allowed-root /path/to/alkemie-workspaces \
  --helper-path /usr/local/libexec/alkemie-privileged-helper/alkemie-privileged-helper \
  --sudo-path /usr/bin/sudo
bin/alkemie-cluster-gateway run --queue-system slurm
```

普通用户通过浏览器使用平台，无需安装客户端。

## 升级已有安装

1. 停止旧服务，备份 `.env`、数据库和项目数据。
2. 将新版本解压到新的目录，按安装步骤准备依赖和离线模型。
3. 将原有 `.env` 复制到新目录，对照新版 `.env.example` 补充配置，保留实际密钥与存储路径。
4. 运行环境检查并启动新服务，确认原有项目和计算结果可访问后再移除旧运行目录。

不要用 `.env.example` 覆盖已有 `.env`，也不要删除旧目录中的项目数据、模型缓存、证书或密钥。

## 常见问题

- **Python 版本不匹配或提示 `No module named 'encodings'`**：激活 Python 3.12 的 `alkemie` Conda 环境，再运行 `scripts/check-env.sh`。
- **提示 `GLIBC_x.y not found`**：确认运行包与依赖包都适用于目标系统；旧系统需要兼容的 Conda 包，或使用较新的 Linux 服务节点。
- **离线模式下无法加载检索模型**：检查模型是否解压到 `model-cache/hub/`，以及 `.env` 中的 `HF_HOME`、`HF_HUB_CACHE` 路径。
- **页面可打开但保存或删除失败**：优先使用 HTTPS 或 SSH 隧道，并确认请求到达同一服务地址。
- **智能助手无法执行项目脚本**：检查项目目录权限和当前 Conda 环境；`shared`/`cluster` 模式请由管理员检查沙箱和账号映射。需要使用其他 Python 时设置 `ALKEMIE_PROJECT_PYTHON=/absolute/path/to/python`。

[报告问题](https://github.com/hfood02/alkemie-release/issues)

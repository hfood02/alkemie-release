# Alkemie 下载

版本：v2.0.0-20261029-r2

| 文件 | 用途 | 校验文件 |
| --- | --- | --- |
| [alkemie-hpc-nuitka-linux-x86_64.zip](https://github.com/hfood02/alkemie-release/releases/latest/download/alkemie-hpc-nuitka-linux-x86_64.zip) | Linux x86_64、Python 3.12；Windows 通过 WSL2 + Ubuntu 使用 | [SHA-256](alkemie-hpc-nuitka-linux-x86_64.zip.sha256) |
| [alkemie-model-cache-all-MiniLM-L6-v2.tar.gz](https://github.com/hfood02/alkemie-release/releases/latest/download/alkemie-model-cache-all-MiniLM-L6-v2.tar.gz) | 本地知识检索的离线模型 | [SHA-256](alkemie-model-cache-all-MiniLM-L6-v2.tar.gz.sha256) |

运行包包含网页、服务程序和启动脚本；Python 科学计算依赖通过包内的 Conda 环境和安装脚本准备。默认离线检索需要同时下载模型缓存包。

[安装与升级说明](README.md) · [版本更新](https://github.com/hfood02/alkemie-release/releases/latest)

## 可选校验

需要确认下载文件完整性时，将两个下载文件与 [checksums.txt](checksums.txt) 放在同一目录。

Linux：

    sha256sum -c checksums.txt

macOS：

    shasum -a 256 -c checksums.txt

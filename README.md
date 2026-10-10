# MoviePilot V3 NAS 部署项目

独立维护的 MoviePilot V3 一键部署仓库，仓库地址为 [warptr/docker-mp-v3](https://github.com/warptr/docker-mp-v3)。基于旧项目 `warptr/docker` 的部署脚本重新初始化 Git，不继承旧仓库的提交历史和远程配置。

## 与 V2 的关系

- MoviePilot 镜像使用官方 `jxxghp/moviepilot-v3:latest`。
- MoviePilot 容器、配置及浏览器内核目录使用 `moviepilot-v3`；浏览器包发布到本仓库的 `v3-latest` Release。
- 保留 `dsm`、`fnos`、`fnos-arm`、`qnap`、`ugreen`、`zspace`、`zspace-arm`、`zspace-z4s` 平台分支及对应的路径、架构配置。
- 完整保留原项目 `docker/` 下的部署资源，将 MoviePilot 目录改为 `moviepilot-v3`，其余内容按原文件复制，包括三个服务的配置、数据库、插件与备份、缓存及日志。
- `media.zip` 沿用原项目的完整媒体预设包。三个服务的配置与媒体资源均随新仓库提供，安装时无需访问旧 V2 仓库或已有 V2 安装目录。
- MoviePilot 的 `app.env`、`category.yaml`、`plugins/`、`plugins_backup/` 和 `temp/plugin_backup/` 均已携带。插件备份不会自动变为已启用插件；请在 V3 中检查插件是否需要适配或重新安装。
- 不继承旧仓库的 `.git` 或 IDE 配置。浏览器内核通过本项目独立的 `v3-latest` Release 提供。

仓库中的数据库是安装预设，不是运行中的数据库。首次安装会复制该数据库到新的 `moviepilot-v3/config`，V3 启动时会执行数据库升级；原项目中的数据库保持原样。是否出现首次初始化页面以数据库中已有的账号数据为准。

重复安装只补齐缺失文件，不覆盖现有数据库、账号和插件配置。已有 V3 安装不会因更新仓库中的数据库预设而同步数据。

本项目不是原 NAS 实例的自动迁移工具。若需要原实例的最新状态，应先停止原 MoviePilot 并完整备份其配置，再将备份复制到新的 V3 配置目录后启动。不要同时让 V2 和 V3 使用同一个数据库。回退应恢复升级前的完整备份。

完整 Compose 还包含 qBittorrent 和 Emby，沿用原项目的容器名称、端口及数据路径；与旧完整项目并行运行会发生冲突。MoviePilot 默认 Web/API 端口仍为 13000/13001。如需同时运行多个实例，请先调整容器名、端口和数据映射。

## 发布前准备

1. 创建并推送新 GitHub 仓库与全部平台分支。
2. `main` 分支的内核打包脚本或工作流变更后自动运行内核发布，也支持定时同步及手动运行 `Sync MoviePilot Core Tarballs to GitHub Release`。等待 amd64 和 arm64 的包都发布到 `v3-latest`。
3. 在测试 NAS 上验证镜像拉取、账号登录、插件、订阅、下载和整理流程。

以下远程一键命令要求仓库公开且内核 Release 已准备好。私有仓库不支持这些匿名下载命令，请先准备认证下载方式再分发。

```bash
bash <(curl -fsSL "https://raw.githubusercontent.com/warptr/docker-mp-v3/main/mp-setup-choose.sh")
```

独立更新浏览器核心：

```bash
bash <(curl -fsSL "https://raw.githubusercontent.com/warptr/docker-mp-v3/main/update_mp_core.sh")
```

安装时读取所选平台分支的 `docker/moviepilot-v3/.env`，将其中的 `DOCKER_PATH` 和 `MEDIA_PATH` 作为交互默认值，可手动覆盖。`main` 不预设路径，必须手动填写。极空间的用户名、存储池和星号占位符必须按实际查询结果补全。

## 更新与恢复

- 安装脚本会备份已有 Compose 和 `.env` 到 `.nas-deploy-backups/`，只更新 `.env` 中的两个路径变量。
- 内核更新先下载、校验和解压，再替换 `core`。失败时保留或恢复旧内核，成功后保留 `core.backup.<时间>.<进程号>`。
- 新内核包附 `.sha256` 和 `.source.json`。校验文件格式无效或 SHA256 不匹配时停止更新；继承的历史兼容逻辑会在校验文件无法下载时明确提示仅检查压缩包结构。
- 独立更新内核后，在 `moviepilot-v3` 安装目录执行以下命令，使容器重新挂载新目录；安装脚本会自动执行：

```bash
docker compose up -d --no-deps --force-recreate moviepilot
```

- 三个脚本均支持 `GITHUB_PROXY`。设为空字符串即可直连：`GITHUB_PROXY='' bash ./update_mp_core.sh`。
- 容器检查最多等待 60 秒；无 HEALTHCHECK 时只能确认 running，不能证明应用初始化已完成。

## 本地验证

需要 Bash 4+、Python 3.12+、常见 Linux 文件工具及 unzip。Python 仅用于测试和 Actions 打包，NAS 安装脚本无需 Python。

```bash
bash -n mp-setup.sh && bash -n mp-setup-choose.sh && bash -n update_mp_core.sh
TMPDIR="$PWD" python3 -m unittest discover -s tests -v
```

回归测试使用临时目录、模拟网络和 Docker，不启动真实容器。完整安装测试包含所有真实配置文件；将临时目录放到当前磁盘目录可避免 NAS 的内存临时盘空间不足。浏览器内核打包使用 CloakHQ/CloakBrowser 的上游发布文件。

官方参考：[安装指引](https://github.com/jxxghp/MoviePilot-Wiki/blob/main/install.md)、[升级说明](https://github.com/jxxghp/MoviePilot-Wiki/blob/main/upgrade.md)。

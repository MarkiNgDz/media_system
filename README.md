# Media System

基于 Docker Compose 的媒体服务栈：组合媒体播放、媒体请求、影视整理、下载、索引器管理、字幕和弹幕服务。服务定义集中在 [`docker-compose.yml`](./docker-compose.yml)，主要用于自托管或家庭媒体库场景。

## 服务组成

| 服务 | 用途 | Web 访问地址 |
| --- | --- | --- |
| Jellyfin | 媒体库管理与播放 | `http://<主机地址>:8096` |
| Seerr | 媒体请求管理 | `http://<主机地址>:5055` |
| Radarr | 电影管理 | `http://<主机地址>:7878` |
| Sonarr | 电视剧及动画管理 | `http://<主机地址>:8989` |
| Prowlarr | 索引器管理 | `http://<主机地址>:9696` |
| Bazarr | 电影和电视剧字幕管理 | `http://<主机地址>:6767` |
| Aria2-Pro | 下载服务及 RPC 接口 | RPC：`http://<主机地址>:6800` |
| AriaNg | Aria2 Web 管理界面 | `http://<主机地址>:6880` |
| danmu-api | 弹幕 API | `http://<主机地址>:9321` |
| MetaTube | 元数据服务 | `http://<主机地址>:8080` |
| PostgreSQL | MetaTube 数据库 | 不对主机发布端口 |
| Watchtower | 定时更新容器 | 无 Web 界面 |

Jellyfin 还映射了 `8920/tcp`、`7359/udp` 和 `1900/udp`，用于 HTTPS 及客户端发现等功能。Aria2 还映射了 `6888/tcp` 和 `6888/udp`，用于 BitTorrent 连接。

> Compose 中发布的端口默认可从主机网络访问。部署到公网前请设置防火墙或反向代理访问策略，并为各服务配置强密码；不要直接公开 Aria2 RPC 端口。

## 环境要求

- Docker Engine
- Docker Compose v2（使用 `docker compose` 命令）
- 媒体目录和下载目录，以及 Docker 守护进程可访问这些目录的权限

## 配置

1. 在项目根目录创建 `.env`，复制 [`.env.example`](./.env.example) 后填写配置。请勿将真实凭据提交到版本库。

   Linux/macOS：

   ```sh
   cp .env.example .env
   ```

   Windows PowerShell：

   ```powershell
   Copy-Item .env.example .env
   ```

   如果 `.env` 已存在，请直接编辑，不要覆盖现有配置。

2. 填写 `.env` 中的变量：

   | 变量 | 说明 |
   | --- | --- |
   | `ARIA2_SECRET` | Aria2 RPC 密钥 |
   | `TRACKER_URL` | 自定义 Tracker 列表 URL；多个 URL 可按 Aria2 镜像所支持的格式填写 |
   | `SSD_MOVIE_PATH`、`SSD_TV_PATH`、`SSD_MUSIC_PATH`、`SSD_AV_PATH` | SSD 上对应媒体目录的主机绝对路径 |
   | `HDD_MOVIE_PATH`、`HDD_TV_PATH`、`HDD_ANIME_PATH` | HDD 上对应媒体目录的主机绝对路径 |
   | `DOWNLOAD_PATH` | 下载目录的主机绝对路径 |
   | `STGRES_USER`、`STGRES_PASSWORD`、`STGRES_DB` | PostgreSQL 初始化用户名、密码和数据库名 |

   `STGRES_*` 是 Compose 文件当前使用的变量名，请保持拼写一致。路径变量应填写 Docker 主机上的真实路径，并确保相应目录存在且容器有读写权限。Windows Docker Desktop 用户请使用 Docker 可识别的绝对路径，并确保所在磁盘已允许 Docker 访问。

3. 检查 `docker-compose.yml` 中 MetaTube 的 `command`：DSN 当前包含字面占位内容 `******`。首次部署前应将其替换为与 `STGRES_USER`、`STGRES_PASSWORD` 和 `STGRES_DB` 匹配的数据库连接配置；否则 MetaTube 可能无法连接 PostgreSQL。

   数据库使用 Compose 内部共享的 Unix socket 卷，不通过主机端口对外发布。

## 启动与维护

在项目根目录运行：

```sh
# 检查 Compose 配置及变量替换
docker compose config

# 拉取镜像并启动所有服务
docker compose pull
docker compose up -d

# 查看服务状态
docker compose ps

# 查看某个服务的实时日志，例如 Jellyfin
docker compose logs -f jellyfin

# 停止并移除容器和 Compose 网络
docker compose down
```

`docker compose down` 不会移除命名卷；不要轻易使用 `docker compose down -v`，因为它会删除命名卷中的持久化数据。

Watchtower 配置为每 `12600` 秒检查一次镜像，但**只监控 `danmu-api`**，并在更新后清理旧镜像。其他服务需要按需手动更新，例如：

```sh
docker compose pull jellyfin
docker compose up -d jellyfin
```

## 数据与目录

- `jellyfin/library`、`radarr/data`、`sonarr/data`、`prowlarr/data`、`bazarr/config`、`aria2/aria2-config`：通过绑定挂载保存服务配置或状态。
- `danmu-api/config`、`danmu-api/.cache`：挂载到弹幕服务容器。
- MetaTube、PostgreSQL 和 Seerr：使用 Compose 命名卷保存数据。
- 媒体库和下载目录：通过 `.env` 中的主机路径挂载到容器内的 `/data/...` 和 `/downloads`。
- Aria2 脚本配置设置下载完成目录为 `/downloads/completed`；下载脚本还配置了任务失败或删除时清理未完成文件。调整此行为前请先阅读 `aria2/aria2-config/script.conf`。

仓库的 `.gitignore` 默认忽略服务目录和运行数据；目前版本库只纳入 Compose 配置、环境变量示例、README 和 `.gitignore`。因此，克隆仓库本身不会完整恢复各服务已有的配置、数据库或媒体数据；升级或迁移前请自行备份绑定挂载目录和命名卷。

## 媒体路径映射

Compose 将主机目录挂载到固定容器路径。配置 Radarr、Sonarr、Bazarr、Jellyfin 及下载客户端之间的目录时，应使用这些**容器内路径**，并保持各服务的路径映射一致。

| 主机目录变量 | 容器内路径 | 使用服务 |
| --- | --- | --- |
| `SSD_MUSIC_PATH` | `/data/ssd/music` | Jellyfin |
| `SSD_MOVIE_PATH` | `/data/ssd/movie` | Jellyfin、Radarr、Bazarr |
| `SSD_TV_PATH` | `/data/ssd/tv` | Jellyfin、Sonarr、Bazarr |
| `SSD_AV_PATH` | `/data/ssd/av` | Jellyfin、Bazarr |
| `HDD_ANIME_PATH` | `/data/hdd/anime` | Jellyfin、Sonarr、Bazarr |
| `HDD_TV_PATH` | `/data/hdd/tv` | Jellyfin、Sonarr、Bazarr |
| `HDD_MOVIE_PATH` | `/data/hdd/movie` | Jellyfin、Radarr、Bazarr |
| `DOWNLOAD_PATH` | `/downloads` | Aria2-Pro、Radarr、Sonarr |

## 典型使用流程

1. 在 Prowlarr 配置索引器，并将其连接到 Radarr 和 Sonarr。
2. 在 Aria2-Pro 配置下载目录；Radarr、Sonarr 与下载服务应通过一致的容器路径访问下载文件。
3. 在 Radarr 管理电影，在 Sonarr 管理电视剧或动画，并配置媒体库目录。
4. 配置 Bazarr 的媒体路径和字幕服务。
5. 在 Jellyfin 添加相应的媒体库目录并播放媒体。
6. 按需配置 Seerr 的请求流程、danmu-api 和 Jellyfin 中的弹幕/元数据集成。

服务之间的 API 地址、认证信息、索引器、下载客户端和媒体目录需要在各 Web 管理界面中分别设置；Compose 只负责启动容器和挂载目录，不会自动完成这些应用内配置。

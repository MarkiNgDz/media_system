# Media System

基于 Docker Compose 的自托管媒体服务栈，包含媒体播放、媒体请求、影视管理、下载、索引器管理、字幕、弹幕和元数据服务。服务定义见 [`docker-compose.yml`](./docker-compose.yml)。

## 服务组成

| 服务 | 用途 | Web 访问地址 |
| --- | --- | --- |
| Jellyfin | 媒体库管理与播放 | `http://<主机地址>:8096` |
| Seerr | 媒体请求管理 | `http://<主机地址>:5055` |
| Radarr | 电影管理 | `http://<主机地址>:7878` |
| Sonarr | 电视剧及动画管理 | `http://<主机地址>:8989` |
| Prowlarr | 索引器管理 | `http://<主机地址>:9696` |
| Bazarr | 电影和电视剧字幕管理 | `http://<主机地址>:6767` |
| AriaNg | Aria2 Web 管理界面 | `http://<主机地址>:6880` |
| Aria2-Pro | 下载服务及 RPC 接口 | RPC：`http://<主机地址>:6800` |
| danmu-api | 弹幕 API | `http://<主机地址>:9321` |
| MetaTube | 元数据服务 | `http://<主机地址>:8080` |
| PostgreSQL | MetaTube 数据库 | 不发布主机端口 |
| Watchtower | 定时检查并更新容器镜像 | 无 Web 界面 |

Jellyfin 还发布 `8920/tcp`、`7359/udp` 和 `1900/udp`，用于 HTTPS 及客户端发现等功能。Aria2-Pro 还发布 `6888/tcp` 和 `6888/udp`，用于 BitTorrent 连接。

Compose 发布的端口默认可从主机网络访问。部署到公网前请配置防火墙或反向代理访问策略，并为各服务设置强密码；不要直接将 Aria2 RPC 端口暴露给不可信网络。

## 环境要求

- Docker Engine
- Docker Compose v2（使用 `docker compose` 命令）
- Docker 守护进程可访问的媒体目录和下载目录，并具有所需读写权限

## 配置

1. 在项目根目录创建 `.env`，从 [`.env.example`](./.env.example) 复制后填写配置。`.env` 已加入 `.gitignore`，请勿提交真实凭据。

   Linux/macOS：

   ```sh
   cp .env.example .env
   ```

   Windows PowerShell：

   ```powershell
   Copy-Item .env.example .env
   ```

   如果 `.env` 已存在，请直接编辑，不要覆盖已有配置。

2. 按实际环境填写 `.env`：

   | 变量 | 说明 |
   | --- | --- |
   | `ARIA2_SECRET` | Aria2 RPC 密钥 |
   | `TRACKER_URL` | Aria2-Pro 使用的自定义 Tracker 列表 URL |
   | `SSD_MOVIE_PATH`、`SSD_TV_PATH`、`SSD_MUSIC_PATH`、`SSD_JAV_PATH` | SSD 上对应媒体目录的主机路径 |
   | `HDD_MOVIE_PATH`、`HDD_TV_PATH`、`HDD_ANIME_PATH` | HDD 上对应媒体目录的主机路径 |
   | `DOWNLOAD_PATH` | 下载目录的主机路径 |
   | `POSTGRES_USER`、`POSTGRES_PASSWORD`、`POSTGRES_DB` | PostgreSQL 初始化用户名、密码和数据库名 |

   路径变量应使用 Docker 主机可访问的路径，并确保目录存在且容器具有读写权限。Windows Docker Desktop 用户还需确认 Docker 已获准访问对应磁盘。

3. 部署前检查 `docker-compose.yml` 中 MetaTube 的 `command`：当前 DSN 含有 `******` 占位内容。请按 MetaTube 支持的 DSN 格式将其替换为与 `POSTGRES_USER`、`POSTGRES_PASSWORD` 和 `POSTGRES_DB` 一致的数据库连接信息，否则 MetaTube 可能无法连接数据库。PostgreSQL 通过共享卷中的 Unix socket 与 MetaTube 通信，不发布主机端口。

## 启动与维护

完成 `.env` 配置及 MetaTube DSN 修改后，在项目根目录运行：

```sh
# 检查 Compose 配置及变量替换
docker compose config

# 拉取镜像并启动服务
docker compose pull
docker compose up -d

# 查看服务状态
docker compose ps

# 查看指定服务的实时日志，例如 Jellyfin
docker compose logs -f jellyfin

# 停止并移除容器和 Compose 网络
docker compose down
```

`docker compose down` 不会删除命名卷。不要轻易使用 `docker compose down -v`，因为这会删除命名卷中的持久化数据。

Watchtower 每 `12600` 秒检查一次镜像，但只监控 `danmu-api`，并在更新后清理旧镜像。其他服务可按需手动更新，例如：

```sh
docker compose pull jellyfin
docker compose up -d jellyfin
```

## 数据与目录

- Jellyfin、Seerr、Radarr、Sonarr、Prowlarr、Bazarr、Aria2-Pro、danmu-api、MetaTube 和 PostgreSQL 的配置或状态均通过 Compose 命名卷持久化。
- 媒体库和下载目录通过 `.env` 中的主机路径绑定挂载到容器内路径。
- 仓库仅包含 Compose 配置、环境变量示例、README 和 `.gitignore`；服务配置及数据库数据存放在 Docker 命名卷中，不会随克隆仓库一并恢复。迁移或升级前请备份相关命名卷及媒体数据。

## 媒体路径映射

在 Jellyfin、Radarr、Sonarr、Bazarr 和下载客户端中配置目录时，使用下表中的**容器内路径**，并确保相关服务对同一媒体或下载目录使用一致的路径。

| 主机目录变量 | 容器内路径 | 挂载到的服务 |
| --- | --- | --- |
| `SSD_MUSIC_PATH` | `/data/ssd/music` | Jellyfin |
| `SSD_MOVIE_PATH` | `/data/ssd/movie` | Jellyfin、Radarr、Bazarr |
| `SSD_TV_PATH` | `/data/ssd/tv` | Jellyfin、Sonarr、Bazarr |
| `SSD_JAV_PATH` | `/data/ssd/av` | Jellyfin、Bazarr |
| `HDD_ANIME_PATH` | `/data/hdd/anime` | Jellyfin、Sonarr、Bazarr |
| `HDD_TV_PATH` | `/data/hdd/tv` | Jellyfin、Sonarr、Bazarr |
| `HDD_MOVIE_PATH` | `/data/hdd/movie` | Jellyfin、Radarr、Bazarr |
| `DOWNLOAD_PATH` | `/downloads` | Aria2-Pro、Radarr、Sonarr |

## 典型使用流程

1. 在 Prowlarr 配置索引器，并将其连接到 Radarr 和 Sonarr。
2. 在 Aria2-Pro 配置下载目录；Radarr、Sonarr 与下载服务应通过相同的容器路径访问下载文件。
3. 在 Radarr 管理电影，在 Sonarr 管理电视剧或动画，并配置对应媒体库目录。
4. 在 Bazarr 配置媒体路径和字幕服务。
5. 在 Jellyfin 添加媒体库目录，并按需配置 Seerr 的请求流程、danmu-api 和 MetaTube 集成。

服务之间的 API 地址、认证信息、索引器、下载客户端和媒体目录需要在各 Web 管理界面中配置；Compose 负责启动容器和挂载目录，不会自动完成这些应用内设置。

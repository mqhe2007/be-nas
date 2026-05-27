# *arr 全家桶部署指南

## 架构

```
                    ┌─────────────┐
                    │   Prowlarr  │  索引器管理  :9696
                    └──────┬──────┘
                           │ 同步索引器
              ┌────────────┼────────────┐
              ▼            │            ▼
        ┌──────────┐       │     ┌──────────┐
        │  Radarr  │       │     │  Sonarr  │  电影/剧集管理
        │  :7878   │       │     │  :8989   │
        └────┬─────┘       │     └────┬─────┘
             │              │          │
             │ 搜索资源      │          │ 搜索资源
             ▼              │          ▼
        ┌──────────┐       │
        │qBittorrent│       │     下载引擎 :8080
        │  :8080    │◄──────┘
        └────┬─────┘
             │ 下载完成
             ▼
   /data/downloads/ ──硬链接──▶ /data/media/ ──▶ Jellyfin :8096
```

## 网络规划

所有 *arr 服务共用同一个 Docker bridge 网络（与 qBittorrent 一起），容器间通过**容器名**互访：

```yaml
networks:
  default:
    name: qbittorrent_net
    external: true
```

> ⚠️ 容器间互访必须用容器名（如 `http://radarr:7878`），不能用宿主机 IP，否则 hairpin NAT 不通。

## 代理策略

| 服务 | 代理 | 原因 |
|------|------|------|
| qBittorrent | ❌ | 国内 PT 站直连可达 |
| Prowlarr | ✅ | 索引器定义更新需翻墙 |
| Radarr | ✅ | TMDB 元数据需翻墙 |
| Sonarr | ✅ | TVDB/TMDB 需翻墙 |

Prowlarr 的 NO_PROXY 可排除 PT 站点域名：
```yaml
NO_PROXY: "localhost,127.0.0.1,10.10.1.0/24,*.m-team.cc"
```

## 目录规划

| 路径 | 介质 | 挂载到容器 |
|------|------|-----------|
| `~/docker/qbittorrent/` | SSD | `/config` |
| `~/docker/prowlarr/` | SSD | `/config` |
| `~/docker/radarr/` | SSD | `/config` |
| `~/docker/sonarr/` | SSD | `/config` |
| `/data/downloads/` | HDD | `/downloads`（qBit, Radarr, Sonarr 都挂载） |
| `/data/media/` | HDD | `/media`（Radarr, Sonarr, Jellyfin 都挂载） |

**硬链接前提**：`/data/downloads/` 和 `/data/media/` 必须在同一文件系统，且容器内外路径一致。

## Compose 模板

### qBittorrent（先部署，创建网络）

```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    volumes:
      - /home/qingjin/docker/qbittorrent/config:/config
      - /data/downloads:/downloads
    environment:
      PUID: "1000"
      PGID: "1000"
      TZ: Asia/Shanghai
      WEBUI_PORT: "8080"
    restart: unless-stopped

networks:
  default:
    name: qbittorrent_net
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### Prowlarr / Radarr / Sonarr（后续部署，加入已有网络）

```yaml
services:
  prowlarr:        # 或 radarr / sonarr
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    ports:
      - "9696:9696"   # Radarr: 7878, Sonarr: 8989
    volumes:
      - /home/qingjin/docker/prowlarr/config:/config
      - /data/downloads:/downloads       # Radarr/Sonarr 需要
      - /data/media:/media               # Radarr/Sonarr 需要
    environment:
      PUID: "1000"
      PGID: "1000"
      TZ: Asia/Shanghai
      HTTP_PROXY: "http://172.20.0.1:7897"
      HTTPS_PROXY: "http://172.20.0.1:7897"
      NO_PROXY: "localhost,127.0.0.1,10.10.1.0/24"
    restart: unless-stopped

networks:
  default:
    name: qbittorrent_net
    external: true
```

## 串联配置（WebUI 操作）

### 1. Prowlarr 添加索引器
Settings → Indexers → Add → 搜 PT 站名 → 填账号密码

### 2. Prowlarr → Radarr/Sonarr 同步
Settings → Apps → + → Radarr：
- Prowlarr Server: `http://radarr:7878`
- API Key: Radarr → Settings → General 获取

Sonarr 同理，地址 `http://sonarr:8989`

### 3. Radarr/Sonarr 添加下载客户端
Settings → Download Clients → + → qBittorrent：
- Host: `qbittorrent`
- Port: `8080`
- 用户名密码: qBit WebUI 凭据

### 4. 设置媒体库根目录
Settings → Media Management → Root Folders：
- Radarr: `/media/影视/电影`
- Sonarr: `/media/影视/剧集`

## 下载分类

qBittorrent WebUI 左侧「分类」→ 添加：

| 分类 | 路径 |
|------|------|
| movies | `/downloads/movies/` |
| tv | `/downloads/tv/` |

非媒体资源（软件/游戏等）不添加分类，Radarr/Sonarr 会自动忽略。

## 硬链接验证

```bash
# 下载一个文件后，在 Radarr 中手动导入
# 确认两个路径指向同一 inode：
ls -li /data/downloads/movies/<file> /data/media/影视/电影/<file>
# 两行第一列（inode 号）相同 = 硬链接成功
```

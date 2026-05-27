# qBittorrent 部署备忘

## Compose 模板

```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    ports:
      - "8080:8080"      # Web UI
      - "6881:6881"      # BT TCP
      - "6881:6881/udp"  # BT UDP (DHT)
    volumes:
      - ~/docker/qbittorrent/config:/config
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

## 首次登录

默认账号 `admin`，临时密码在日志中：

```bash
docker logs qbittorrent | grep password
```

登录后立即修改密码。

## 代理策略：按需判断，不要一刀切

**qBittorrent 通常不需要代理**。如果用户的 PT 站是国内站（如 M-Team、HDSky、馒头），直连即可。

判断标准：
- 用户能直接从浏览器访问 PT 站 → qBittorrent 不需要代理
- 只有 Jellyfin（TMDB）、Radarr/Sonarr（元数据 API）、Prowlarr（索引器定义）等访问被墙服务的才需要代理

若确需代理（国外 tracker），在 WebUI 中配置：
- 工具 → 选项 → 连接 → 代理服务器
- 类型 HTTP，主机 `172.20.0.1`，端口 `7897`
- ⚠️ **必须勾选「仅对 tracker 使用代理」**，否则 P2P 流量也走代理

## 下载分类

| 分类 | 保存路径 |
|------|---------|
| movies | `/downloads/movies/` |
| tv | `/downloads/tv/` |
| software | `/downloads/software/` |
| games | `/downloads/games/` |
| books | `/downloads/books/` |

## 与 Jellyfin 配合

下载完成后手动将文件移到 `/data/media/影视/电影/` 或 `电视/`，Jellyfin 自动扫描入库。

若部署了 Radarr/Sonarr，它们会通过硬链接自动完成这一步；手动模式下用户自己移动即可。

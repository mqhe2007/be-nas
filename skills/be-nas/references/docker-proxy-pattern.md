# Docker 容器代理配置模式

## 问题背景

在中国大陆部署 Docker 服务时，许多服务需要代理才能正常访问外网（TMDB 刮削、模型下载等）。但 Docker bridge 网络与宿主机代理之间存在多层网络隔离问题。

## 核心问题链

```
容器 → host.docker.internal → 默认 bridge 网关 (172.17.0.1)
        ↕ Docker 跨 bridge 网络不通！
容器在自定义 bridge (172.18.x / 172.19.x)

容器 → 宿主机 LAN IP (10.10.1.10)
        ↕ hairpin NAT 不通！
Docker bridge 流量无法回连宿主机自身 IP

容器 → bridge 网关 IP (172.18.0.1 / 172.19.0.1)
        ↕ UFW 默认 DROP！
UFW 拦截非标准端口
```

## 最终方案：Bridge 网关 + UFW 白名单

### 1. 确定容器的 bridge 网关

```bash
docker inspect <container> --format '{{range .NetworkSettings.Networks}}{{.Gateway}}{{end}}'
```

### 2. 添加 UFW 规则

```bash
sudo ufw allow from <bridge_subnet> to any port 7897 proto tcp comment '<service> proxy access'
```

例如：
```bash
sudo ufw allow from 172.18.0.0/16 to any port 7897 proto tcp comment 'Jellyfin proxy'
sudo ufw allow from 172.19.0.0/16 to any port 7897 proto tcp comment 'Immich proxy'
```

### 3. 配置 Compose

```yaml
environment:
  HTTP_PROXY: "http://172.18.0.1:7897"   # 使用 bridge 网关 IP
  HTTPS_PROXY: "http://172.18.0.1:7897"
  NO_PROXY: "localhost,127.0.0.1,10.10.1.0/24,<container1>,<container2>"  # 含同网容器名
```

### 4. 锁定网络子网（防重启漂移）

```yaml
networks:
  default:
    name: <service>_net
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/16
```

## 验证方法

```bash
# 从容器内测试代理
docker exec <container> curl -s --max-time 10 -x http://<bridge_gw>:7897 https://api.themoviedb.org/3/configuration

# 预期返回：{"status_code":7,"status_message":"Invalid API key: ..."}
# 说明 TMDB 可访问
```

## 错误排查

| 症状 | 原因 | 排查命令 |
|------|------|---------|
| `ECONNREFUSED` | UFW 拦截 | `sudo ufw status numbered` |
| `Connection timed out` | hairpin NAT / 跨 bridge | 测试其他端口如 22 |
| `EAI_AGAIN` / DNS | 容器 DNS 配置 | `docker exec <c> cat /etc/resolv.conf` |

## 容器间互访：用容器名，不用宿主机 IP

同一 Docker 网络的容器之间，**使用容器名作为 DNS**，不要用宿主机 IP：

```
❌ http://10.10.1.10:7878   → hairpin NAT 不通
✅ http://radarr:7878        → 容器名即 DNS
```

hairpin NAT 对容器之间的互访同样有效——容器发出的包无法通过宿主 IP 回到同一宿主上的另一个容器。所有 \\*arr 系列服务（Prowlarr ↔ Radarr/Sonarr ↔ qBittorrent）互配时都使用容器名。

> ⚠️ **NO_PROXY 必须包含所有容器名**：如果 `HTTP_PROXY` 已设但 NO_PROXY 里没有容器名，容器间互访请求会被代理拦截 → 502。正确写法：
> ```yaml
> NO_PROXY: "localhost,127.0.0.1,10.10.1.0/24,radarr,sonarr,qbittorrent,prowlarr"
> ```

## 代理策略：按服务实际需要决定

不是所有服务都需要代理，按需判断：

| 需要代理 | 不需要代理 |
|---------|-----------|
| Jellyfin（TMDB 刮削） | qBittorrent（国内 PT 站直连） |
| Radarr/Sonarr（TMDB 元数据） | Samba、Minecraft |
| Prowlarr（索引器定义更新） | |
| Immich ML（模型下载） | |

> 用户能直连的国内 PT 站（如 M-Team），qBittorrent 无需配代理；Prowlarr 中可为特定站点配置直连。

## 为什么不用 host 网络模式

`network_mode: host` 虽然简单，但：
- 需要额外配置 UFW 放行业务端口（如 8096）
- 失去了 Docker 的端口映射和网络隔离
- 端口冲突风险

仅在以下场景考虑 host 模式：
- 高性能网络需求（如 Minecraft 游戏服务器）
- 服务端口不会暴露到公网
- 防火墙配置可控

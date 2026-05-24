# Immich 部署备忘

## 镜像选择（重要！）

Immich 官方提供了自己的 PostgreSQL 镜像，**不要**使用 `tensorchord/pgvecto-rs`！

```yaml
# ✅ 正确
database:
  image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0

# ❌ 错误（标签不存在）
database:
  image: tensorchord/pgvecto-rs:pg16
```

Redis 使用 Valkey（Immich 官方推荐）：
```yaml
valkey:
  image: docker.io/valkey/valkey:9
```

## 关键环境变量

```env
IMMICH_VERSION=v2
UPLOAD_LOCATION=/data/immich/upload
DB_DATA_LOCATION=/home/qingjin/docker/immich/postgres
DB_USERNAME=postgres
DB_PASSWORD=<secure-password>
DB_DATABASE_NAME=immich
TZ=Asia/Shanghai
REDIS_HOSTNAME=valkey        # ⚠️ 必须设置！默认是 redis
```

**`REDIS_HOSTNAME=valkey` 是必须的**，没有这个变量 Immich 会尝试连接 `redis` 主机名导致 `EAI_AGAIN` 错误。

## 代理配置

ML 容器需要代理下载模型：
```yaml
environment:
  HTTP_PROXY: "http://172.19.0.1:7897"   # bridge 网关 IP
  HTTPS_PROXY: "http://172.19.0.1:7897"
  NO_PROXY: "localhost,127.0.0.1,10.10.1.0/24"
```

详见 `references/docker-proxy-pattern.md`。

## CLI 批量导入

```bash
npm install -g @immich/cli
immich login http://<host>:2283/api <api-key>
immich upload --recursive --album "相册名" /path/to/photos/
```

- `--album` 自动创建相册，目录名即为相册名
- `--recursive` 递归上传子目录
- CLI 自动去重，重复文件不会再次上传
- 建议后台运行：33K 张照片约需 4 小时

## 容器目录规划

| 用途 | 路径 | 介质 |
|------|------|------|
| 照片存储 | `/data/immich/upload/` | HDD |
| 数据库 | `~/docker/immich/postgres/` | SSD |
| ML 缓存 | `/data/immich/ml-cache/` | HDD |
| Compose 配置 | `~/docker/immich/` | SSD |

# baidupan-cli 百度网盘命令行客户端

## 概述

baidupan-cli 是用户自开发的百度网盘 CLI 工具，用于备份 NAS 数据到百度网盘。
GitHub: https://github.com/mqhe2007/baidupan-cli
当前版本：v0.3.0

> ⚠️ v0.3.0 重大变更：移除 mengqinghe.com 认证后端，改用百度 OAuth 直连。需要手动设置 `BAIDUPAN_APP_KEY`/`BAIDUPAN_APP_SECRET`/`BAIDUPAN_APP_NAME` 环境变量。

## 安装

从 GitHub Releases 下载预编译二进制：

```bash
# 确认架构
uname -m   # x86_64 / aarch64

# 下载最新版（国内需走代理）
curl -x http://127.0.0.1:7897 -sL \
  "$(curl -sL https://api.github.com/repos/mqhe2007/baidupan-cli/releases/latest \
    | jq -r '.assets[] | select(.name | test("linux-x86_64")) | .browser_download_url')" \
  -o /tmp/baidupan.tar.gz

tar xzf /tmp/baidupan.tar.gz
sudo install -m 755 baidupan-cli /usr/local/bin/
```

> 注意：`--version` 可能显示旧版本号，用 `upload --help | grep force` 验证是否为 v0.3.0+。

## 环境变量

v0.3.0 需要以下环境变量：

```bash
export BAIDUPAN_APP_NAME=baidupan-cli
export BAIDUPAN_APP_KEY=<your_app_key>
export BAIDUPAN_APP_SECRET=<your_secret>
export BAIDUPAN_CRYPTO_PASSPHRASE=<your_passphrase>   # 加密/解密时需要
```

所有命令都需要前三项；`--encrypt`/`--decrypt` 还需 `BAIDUPAN_CRYPTO_PASSPHRASE`。

## 认证流程

v0.3.0 使用百度 OAuth 设备码直连（`https://openapi.baidu.com/device`），不再依赖外部认证后端。

### 设备码授权（唯一方式）

`baidupan-cli login` 生成设备码后进入轮询，需要在**另一个终端或浏览器**输入设备码授权。

**关键技巧：并行处理**——后台启动 login 轮询，同时把设备码发给用户：

```bash
# 后台启动（需要 PTY 才能实时看到输出）
script -q -c "baidupan-cli login" /tmp/baidupan-login.log &

# 等 3 秒后提取设备码并发给用户
sleep 3 && cat /tmp/baidupan-login.log
```

输出示例：
```
App key: VImW...AWDJ
Open this URL: https://openapi.baidu.com/device
User code: a8z6qyf2
QR URL: https://openapi.baidu.com/device/qrcode/...
Waiting for authorization, polling every 5 seconds...
```

用户打开 `https://openapi.baidu.com/device` 输入用户码即可。login 进程在后台每 5 秒轮询，授权成功后自动完成。

> ⚠️ 每次 `login` 生成**新设备码**，旧的立即失效。必须并行操作，不要先 login → kill → 再 login。

## 命令速览

| 命令 | 说明 |
|------|------|
| `login` | OAuth 设备码登录 |
| `logout` | 删除本地 token |
| `whoami` | 查看 token scope 和过期时间 |
| `ls <path>` | 列出远程目录 |
| `mkdir <path>` | 创建远程目录 |
| `rm <path>` | 删除远程文件/目录 |
| `mv <old> <new>` | 移动/重命名 |
| `cp <src> <dst>` | 复制 |
| `upload [--encrypt] [--force] [--verbose] <local> <remote>` | 上传文件（流式分块加密，支持断点续传） |
| `download [--decrypt] <remote> <local>` | 下载文件（支持断点续传） |
| `batch <manifest.json>` | 批量执行 |
| `--json` | 输出 JSON 格式 |

## 沙箱路径

百度网盘开放平台应用限定在沙箱目录下，**所有路径自动映射到 `/apps/baidupan-cli/`**。

```bash
# 你写的路径           实际网盘路径
/backup/private/  →  /apps/baidupan-cli/backup/private/
```

- ls 时**不要**手动加 `/apps/baidupan-cli/` 前缀（会报错）
- 所有命令的路径参数都是相对于沙箱根目录

## 加密

`upload --encrypt` 和 `download --decrypt` 支持端到端加密，加密在本地流式分块完成后再上传。

- 需要设置 `BAIDUPAN_CRYPTO_PASSPHRASE` 环境变量
- 上传完成时输出 `remote payload is encrypted` 确认加密
- 下载完成时输出 `local payload has been decrypted` 确认解密

### 验证加密完整性

部署前必须验证加密→解密链路（用一个小文件即可）：

```bash
SRC="/path/to/test.jpg"
REMOTE="/backup/_test/test.jpg"
DOWNLOAD="/tmp/test-download.jpg"

# 上传加密
baidupan-cli mkdir /backup/_test
baidupan-cli upload --encrypt "$SRC" "$REMOTE"
# 云端确认：输出会显示 "remote payload is encrypted"

# 下载解密
baidupan-cli download --decrypt "$REMOTE" "$DOWNLOAD" --force
# 本地确认：输出会显示 "local payload has been decrypted"

# MD5 对比
md5sum "$SRC" "$DOWNLOAD"  # 必须完全一致
```

## 备份脚本集成

备份脚本 `~/.hermes/scripts/baidupan-backup.py` 通过 `subprocess` 调用 `baidupan-cli`。

### 架构

```
SQLite 状态追踪 → mtime+size 增量比对 → 加密上传 → 更新状态
```

- **状态数据库**: `~/.hermes/data/backup.db`（SQLite，百万级文件毫秒响应）
- **调度**: Hermes Cron 每天北京时间 02:00
- **并发**: `ThreadPoolExecutor(max_workers=1)`（串行上传，避免多文件同时吃内存触发 OOM）

### 备份语义（单向备份，非双向同步）

| 本地操作 | 远程行为 | 实现 |
|---------|---------|------|
| 新增文件 | upload --encrypt | mtime/size 不匹配 |
| 修改文件 | upload --encrypt 覆盖 | mtime 变化 |
| 删除文件 | **不动**，标记 deleted（墓碑） | 云端保留，可恢复 |
| 移动/重命名 | `baidupan-cli mv` | size 相同 + mtime 相近（<5s）→ 判定 rename |

### 排除策略

**核心原则：公共资源刮削产物不备份**——Jellyfin 元数据、世界编辑语言包、AI 缓存等可通过服务重扫/重建恢复的数据一律排除，只备份用户的真实数据和服务核心配置。

| 服务 | 排除项 | 原因 |
|------|--------|------|
| Jellyfin | `metadata`, `cache`, `log`, `transcoding-temp` | TMDB 刮削可重扫 |
| Immich | `ml-cache`, `library`, `thumbs`, `encoded-video`, `profile` | AI 缓存/缩略图可重建 |
| Minecraft | `logs`, `*.log`, `*.log.gz`, `crash-reports`, `libraries`, `versions`, `.archive-unpack` | 游戏库启动自下载 + 插件解包产物 |
| qBittorrent | `cache`, `log`, `GeoDB`, `lockfile` | 缓存+GeoIP 自动下载+运行时锁 |
| Hermes | `logs`, `cache`, `sessions`, `sandboxes`, `node`, `*.pid`, `*.lock` | 运行时文件 |
| 全局 | `*.tmp`, `*.temp`, `Thumbs.db`, `.DS_Store`, `__pycache__` | 系统垃圾文件 |

> ⚠️ `*.log` 不匹配 `*.log.gz`，需单独加 `*.log.gz` 规则。

保留的核心数据：Jellyfin 数据库（`jellyfin.db`、播放记录、用户设置）、插件配置文件、qBittorrent 种子文件（`BT_backup/*.torrent`、`*.fastresume`）、compose 文件、环境变量文件。

### 定时调度（零 Token）

使用 Hermes Cron 的 `no_agent` 模式，调度器直接运行 Python 脚本，**完全绕过 LLM，零 token 消耗**：

```bash
hermes cron create \
  --name "小宇宙备份" \
  --schedule "0 18 * * *" \     # UTC 18:00 = 北京时间 02:00
  --no-agent \
  --script baidupan-backup.py
```

关键配置：
- `no_agent=true` → 调度器直接 exec 脚本，不启动 Agent
- `script` → 脚本必须在 `~/.hermes/scripts/` 下，只用文件名
- 脚本 `print()` 到 stdout 的内容**自动投递**到飞书作为完成通知
- stdout **为空时不发送任何消息**（静默成功）

### 数据库 Schema

```sql
CREATE TABLE files (
    source      TEXT NOT NULL,    -- 备份任务名 (private/immich/docker/...)
    rel_path    TEXT NOT NULL,    -- 相对于备份根目录
    size        INTEGER NOT NULL,
    mtime       REAL NOT NULL,
    status      TEXT NOT NULL DEFAULT 'pending',  -- pending/synced/modified/deleted
    uploaded_at REAL,
    remote_path TEXT,
    PRIMARY KEY (source, rel_path)
);

## 已知陷阱

### OOM Killer — 大文件内存爆炸

`baidupan-cli` 在上传大文件（尤其是视频 .avi 等）时内存占用极高，单个进程可达 **13GB+**。14GB 内存的机器上 4 并发会立即触发 OOM Killer。

**症状**：进程突然消失，`dmesg` 可见 `Out of memory: Killed process (baidupan-cli)`。

**缓解**：
- 并发降到 1（`CONCURRENCY=1`），串行上传避免多进程叠加
- 依赖 SQLite 状态数据库实现断点续传——即使被 OOM，重启后已上传文件会被跳过
- 如果单文件仍然 OOM（如 >2GB 视频），需要在脚本中加入文件大小阈值，超限文件跳过并记录

### 远程路径长度限制

百度网盘 API 对远程路径有限制，**限制的是 URL 编码后的总长度**（而非 UTF-8 字符数）。中文路径经 URL 编码后每个字符膨胀为 9 字节（`%E6%88%91`），因此同样字符数下中文路径更容易超限。

实测数据：
| 路径 | 字符数 | UTF-8字节 | 结果 |
|------|:---:|:---:|:---:|
| 纯英文 196 字节 | 196 | 196 | ✅ |
| 中文 avi 路径 | 75 | 117 | ✅ |
| 中文 JPG 深层路径 | 97 | **141** | ❌ |

完整远程路径格式：`/apps/baidupan-cli/backup/<source>/<rel_path>`

**症状**：`upload` 返回 `API error: create failed`，错误信息被截断。同名文件用短路径可成功。

**应对**：将超长路径的文件标记为 `skipped` 并记录日志；恢复时需手动处理或用短名重传。

### `*.log` 不匹配 `*.log.gz`

`fnmatch` 的 `*.log` 模式只匹配 `.log` 结尾，**不会**匹配 `.log.gz`。排除规则中需单独加 `*.log.gz`（或直接用 `*.log*`）。

### 路径前缀不要手动加

所有远程路径自动限定在 `/apps/baidupan-cli/` 沙箱内。`ls /backup` 即可，**不要**写成 `ls /apps/baidupan-cli/backup`（会报错 `do not include the /apps/<app_name> prefix`）。

### 环境变量缺失

v0.3.0 的每个命令都需要 `BAIDUPAN_APP_KEY`/`BAIDUPAN_APP_SECRET`/`BAIDUPAN_APP_NAME`。加密操作还需 `BAIDUPAN_CRYPTO_PASSPHRASE`。缺少任何一项都会报 `missing environment variable`。
```

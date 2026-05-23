# 磁盘识别与盘点

本参考用于在执行分区、格式化、建池、替换硬盘等高风险操作前，稳定识别目标磁盘。不得只依赖 `/dev/sdX`、`/dev/nvmeXnY` 这类启动后可能变化的临时名称。

## 安全原则

- 只读盘点优先，先收集信息，不修改磁盘。
- 面向用户展示目标磁盘时，同时列出路径、容量、型号、序列号或 WWN、现有文件系统和挂载点。
- 用户分享盘点结果时，提醒其可打码序列号、WWN、主机名、公网 IP 等可识别信息。
- 执行破坏性命令前，要求用户明确确认目标磁盘的稳定路径，例如 `/dev/disk/by-id/<disk-id>`。

## Linux 常用只读盘点命令

```bash
lsblk -o NAME,PATH,SIZE,TYPE,TRAN,MODEL,SERIAL,WWN,FSTYPE,FSVER,LABEL,UUID,MOUNTPOINTS
blkid
findmnt
ls -l /dev/disk/by-id/
```

SATA/SAS/USB 磁盘可使用：

```bash
sudo smartctl -a /dev/disk/by-id/<disk-id>
```

NVMe 磁盘可使用：

```bash
sudo nvme list
sudo nvme smart-log /dev/disk/by-id/<nvme-disk-id>
```

USB 硬盘盒可能隐藏真实序列号或 WWN。遇到多个同型号同容量磁盘时，应结合盘位标签、热插拔前后 `dmesg`、硬盘盒端口、SMART 信息和物理贴纸确认，不要仅凭 `/dev/sdX` 顺序操作。

## 推荐输出格式

在给用户方案前，先整理为表格：

| 稳定路径                    | 临时设备名 | 容量     | 型号      | 序列号/WWN   | 文件系统   | 挂载点         | 计划用途 |
| --------------------------- | ---------- | -------- | --------- | ------------ | ---------- | -------------- | -------- |
| `/dev/disk/by-id/<disk-id>` | `/dev/sdX` | `<size>` | `<model>` | `<redacted>` | `<fstype>` | `<mountpoint>` | `<role>` |

## 破坏性操作前确认语句

在分区、格式化、创建 PV、重建阵列、擦除签名或替换磁盘前，要求用户确认类似内容：

```text
我确认将对 /dev/disk/by-id/<disk-id> 执行 <operation>，该磁盘容量为 <size>，型号为 <model>，现有数据可以清除或已完成备份。
```

没有得到明确确认前，只能给出检查命令、方案解释和备份建议，不能继续执行破坏性步骤。

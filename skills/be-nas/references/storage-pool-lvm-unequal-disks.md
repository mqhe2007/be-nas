# 不等容量磁盘的 LVM 顺序建池

本参考用于把多块不等容量磁盘合并为一个逻辑卷。该方案重在容量整合和后续扩容，不提供冗余；任一数据盘故障都可能导致逻辑卷损坏。重要数据必须先设计备份，不能把 LVM 合池当作 RAID 或备份。

## 适用场景

- 家庭媒体库、下载暂存、可重新获取的数据。
- 多块容量不同的磁盘，需要形成统一挂载点。
- 用户接受无冗余风险，并已有备份或可恢复来源。

如果环境中同时存在可用 SSD 和 HDD，优先将 SSD 留给系统、应用安装位置、数据库和其他高频读写目录；本参考更适合作为 HDD 数据池或其他容量优先场景，而不是应用层默认落盘方案。

不适合：唯一照片库、唯一文档库、业务数据库、没有备份的珍贵数据。此类数据优先考虑 ZFS mirror、mdadm RAID、Btrfs RAID1、SnapRAID + 定期校验，或直接使用独立磁盘加 3-2-1 备份。

## 操作前检查

先按 `references/disk-identification.md` 完成盘点，确认目标磁盘稳定路径。所有示例都使用占位符，实际执行前必须替换为 `/dev/disk/by-id/<disk-id>` 形式。

```bash
lsblk -o NAME,PATH,SIZE,TYPE,ROTA,MODEL,SERIAL,WWN,FSTYPE,LABEL,UUID,MOUNTPOINTS
findmnt
sudo pvs
sudo vgs
sudo lvs
```

确认事项：

- 目标磁盘没有仍需保留的数据，或已经完成备份。
- 目标磁盘没有被挂载，没有正在被服务使用。
- 使用稳定磁盘路径，不使用可能漂移的 `/dev/sdX` 顺序。
- 已规划文件系统、挂载点、备份策略和扩容方式。

## 新建存储池流程

以下命令会破坏目标磁盘上的现有数据。只有在用户明确确认后才能执行。

```bash
sudo pvcreate /dev/disk/by-id/<disk-1-id> /dev/disk/by-id/<disk-2-id>
sudo vgcreate vg_nas_data /dev/disk/by-id/<disk-1-id> /dev/disk/by-id/<disk-2-id>
sudo lvcreate -n lv_data -l 100%FREE vg_nas_data
sudo mkfs.ext4 -L nas_data /dev/vg_nas_data/lv_data
sudo mkdir -p /data
sudo mount /dev/vg_nas_data/lv_data /data
```

验证：

```bash
sudo pvs
sudo vgs
sudo lvs
df -hT /data
findmnt /data
```

写入开机挂载前，优先使用文件系统 UUID：

```bash
sudo blkid /dev/vg_nas_data/lv_data
```

将 UUID 写入 `/etc/fstab` 后，必须验证：

```bash
sudo findmnt --verify
sudo umount /data
sudo mount /data
df -hT /data
```

## 后续追加磁盘

追加磁盘同样会清空新磁盘上的数据。先确认新磁盘稳定路径和备份状态，再执行：

```bash
sudo pvcreate /dev/disk/by-id/<new-disk-id>
sudo vgextend vg_nas_data /dev/disk/by-id/<new-disk-id>
sudo lvextend -l +100%FREE /dev/vg_nas_data/lv_data
sudo resize2fs /dev/vg_nas_data/lv_data
```

如果使用 XFS，扩容文件系统命令改为：

```bash
sudo xfs_growfs /data
```

## 运维与风险提示

- LVM 合池不是备份，也不是冗余。
- 定期检查 `pvs`、`vgs`、`lvs`、`df -hT`、SMART 健康状态和备份任务。
- 避免把数据库、相册唯一副本和文档唯一副本放在无冗余 LVM 合池中。
- 扩容、迁移或移除 PV 前，先确认完整备份和恢复演练，不要直接删除 PV 或清理签名。

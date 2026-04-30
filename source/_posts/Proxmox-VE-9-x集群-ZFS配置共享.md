---
title: Proxmox VE 9.x集群环境下配置ZFS共享
date: 2026-04-30 09:28:55
tags: PVE
categories: Proxmox VE
top: 39
---

## Proxmox VE 9.x集群环境下配置ZFS共享

## 前提

```bash
我这里是6块4T容量大硬盘做RAID-Z2
RAID-Z1 = 1 块校验盘（最多坏 1 块）
RAID-Z2 = 2 块校验盘（最多坏 2 块）✅ 
RAID-Z3 = 3 块校验盘

一组硬盘里，固定拿出 N 块当「全局校验 / 冗余」，剩下全是纯存储空间

6 块 × 4T 硬盘 RAID-Z2 计算
总盘数：6 块
校验盘：固定 2 块
数据盘：6 - 2 = 4块
容量公式：可用容量=(总盘数−校验数)×单盘容量
代入：(6−2)×4T=16T

理论可用：16T（实际扣一点 ZFS 元数据、预留，实际可用 15.2T 左右）

容错原理：当任意 1 块 / 2 块硬盘彻底损坏，ZFS 依靠剩下 4 块完好盘 + 双重校验码，自动反向计算、还原出丢失的所有数据。
```

### 创建ZFS
```bash
#假设磁盘 sda|sdb|sde|sdf|sdg|sdh，建议用 /dev/disk/by-id/xxx 建池，不建议直接用 /dev/sda，因为重启后盘符可能变化
#查看磁盘id
lsblk -o NAME,SIZE,MODEL,SERIAL,TYPE,MOUNTPOINT
ls -l /dev/disk/by-id/ | grep -E "sda|sdb|sde|sdf|sdg|sdh"

# 先停用可能的 LVM（避免占用）
lvremove -f $(lvs --noheadings -o lv_path 2>/dev/null) || true
vgremove -f $(vgs --noheadings -o vg_name 2>/dev/null) || true
pvremove -ff /dev/sda /dev/sdb /dev/sde /dev/sdf /dev/sdg /dev/sdh || true

#清空磁盘wipe 文件系统签名
wipefs -a /dev/sda 
wipefs -a /dev/sdb 
wipefs -a /dev/sdc 
wipefs -a /dev/sde 
wipefs -a /dev/sdg 
wipefs -a /dev/sdh

#清 GPT/分区表（非常关键）
sgdisk --zap-all /dev/sda 
sgdisk --zap-all /dev/sdb 
sgdisk --zap-all /dev/sdc
sgdisk --zap-all /dev/sde 
sgdisk --zap-all /dev/sdg 
sgdisk --zap-all /dev/sdh

wipefs -n /dev/sda /dev/sdb /dev/sdc /dev/sde /dev/sdg /dev/sdh

lsblk

#创建 6 盘 RAIDZ2
zpool create -f \
  -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O xattr=sa \
  -O acltype=posixacl \
  -O recordsize=1M \
  tank raidz2 \
  /dev/disk/by-id/ata-ST4000VX005-2LY104_ZFN4T9DP \
  /dev/disk/by-id/ata-ST4000VX000-1F4168_Z305GS8J \
  /dev/disk/by-id/ata-ST4000NM0033-9ZM170_ZADRW4C1 \
  /dev/disk/by-id/ata-ST4000NM0033-9ZM170_Z1Z4AN8B \
  /dev/disk/by-id/ata-ST4000NM0035_ZC1XJSB7 \
  /dev/disk/by-id/ata-ST4000NM0033-9ZM170_Z1Z480BY
  
#检查
zpool status && zfs list

#ZFS 池信息
pool: tank
type: raidz2
state: ONLINE
mountpoint: /tank
```
![alt text](/images/pve/6.png)

### 1、创建 PVE 备份目录

```bash
zfs create tank/pve-backup

zfs set mountpoint=/backup/pve-vzdump tank/pve-backup

zfs set compression=lz4 tank/pve-backup

zfs set atime=off tank/pve-backup

zfs set recordsize=1M tank/pve-backup
```

#### 检查

```bash
zfs list
df -h /backup/pve-vzdump
```

### 安装 NFS

```bash
apt update
apt install -y nfs-kernel-server
```

### 配置 NFS 共享

```bash
nano /etc/exports
```
###### 加入一行，假设你的 PVE 管理网段是 192.167.8.0/24：

```
/backup/pve-vzdump 192.167.8.0/24(rw,sync,no_subtree_check,no_root_squash)
```

###### 生效
```bash
exportfs -ra
systemctl enable --now nfs-server
systemctl status nfs-server
```

##### 检查共享

```bash
exportfs -v
```

### 把 NFS 加到 PVE 集群 (可选，如果是集群环境)

```bash
pvesm add nfs pve04-backup \
  --server 192.167.8.6 \
  --export /backup/pve-vzdump \
  --path /mnt/pve/pve04-backup \
  --content backup \
  --options vers=4
  
#查看状态  
pvesm status
```
看到 pve04-backup 就成功。
![alt text](/images/pve/7.png)
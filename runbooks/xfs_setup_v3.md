# XFS Setup v3 - Runbook

## Overview

This runbook describes the XFS filesystem setup process for KeyBuzz v3 infrastructure servers.

## Purpose

XFS is used for:
- Data disks (databases, storage, etc.)
- High-performance I/O requirements
- Large file support
- Better performance than ext4 for data workloads

## Architecture

- **Filesystem**: XFS
- **Mount Points**: `/data/<role_v3>/<disk_name>/`
- **Persistence**: Entries in `/etc/fstab`
- **Permissions**: Root-owned, 755 directories

## Ansible Role

The XFS setup is automated via Ansible role:
- **Location**: `ansible/roles/xfs/`
- **Main tasks**: `ansible/roles/xfs/tasks/main.yml`

## Role Functionality

The XFS role performs:

1. **Disk Detection**
   - Scans for data disks
   - Excludes root disk (sda, vda, nvme0n1)
   - Excludes swap partitions

2. **Filesystem Formatting**
   - Checks if disk is already XFS
   - Formats with XFS if needed
   - Idempotent (skips if already formatted)

3. **Mount Point Creation**
   - Creates `/data/<role_v3>/` directory
   - Creates subdirectories per disk
   - Sets proper permissions

4. **Mounting**
   - Mounts disks using UUID
   - Adds to `/etc/fstab` for persistence
   - Uses `defaults,noatime` options

5. **Verification**
   - Tests write operations
   - Verifies mount status
   - Confirms fstab entries

## Usage

### Apply to Single Server

```bash
cd /opt/keybuzz/keybuzz-infra
ansible-playbook -i ansible/inventory/hosts.yml \
  -e "hosts=db-master-01" \
  -e "role_v3=db_postgres" \
  ansible/playbooks/xfs-setup.yml
```

### Apply to Group

```bash
ansible-playbook -i ansible/inventory/hosts.yml \
  -e "hosts=db_postgres" \
  ansible/playbooks/xfs-setup.yml
```

### Direct Role Application

```yaml
- hosts: db_postgres
  roles:
    - role: xfs
      vars:
        role_v3: "db_postgres"
```

## Mount Point Structure

### Example for db_postgres

```
/data/db_postgres/
├── sdb/          # First data disk
├── sdc/          # Second data disk
└── nvme1n1/      # NVMe disk
```

### Example for k8s_worker

```
/data/k8s_worker/
├── sdb/          # Data disk
└── sdc/          # Additional disk
```

## Manual Setup (if needed)

### 1. Detect Disks

```bash
lsblk -dno NAME,TYPE,SIZE | grep disk
```

### 2. Format Disk

```bash
# WARNING: This will destroy all data on the disk
mkfs.xfs /dev/sdb
```

### 3. Get UUID

```bash
blkid -s UUID -o value /dev/sdb
```

### 4. Create Mount Point

```bash
mkdir -p /data/db_postgres/sdb
```

### 5. Mount Disk

```bash
mount -t xfs UUID=<uuid> /data/db_postgres/sdb
```

### 6. Add to /etc/fstab

```bash
echo "UUID=<uuid> /data/db_postgres/sdb xfs defaults,noatime 0 2" >> /etc/fstab
```

### 7. Test

```bash
mount -a
df -h | grep xfs
```

## Verification

### Check Mount Status

```bash
mount | grep xfs
df -h | grep xfs
```

### Verify fstab

```bash
cat /etc/fstab | grep xfs
```

### Test Write

```bash
echo "test" > /data/db_postgres/sdb/.test_write
cat /data/db_postgres/sdb/.test_write
rm /data/db_postgres/sdb/.test_write
```

### Check Disk Usage

```bash
lsblk
df -h
```

## Troubleshooting

### Disk Not Detected

1. Check if disk is attached:
   ```bash
   lsblk
   ```

2. Check if disk is visible to OS:
   ```bash
   fdisk -l
   ```

3. Rescan for new disks:
   ```bash
   echo "- - -" > /sys/class/scsi_host/host0/scan
   ```

### Format Failed

1. Check if disk is in use:
   ```bash
   lsof | grep /dev/sdb
   ```

2. Unmount if mounted:
   ```bash
   umount /dev/sdb
   ```

3. Check disk for errors:
   ```bash
   fsck.xfs /dev/sdb
   ```

### Mount Failed

1. Check UUID:
   ```bash
   blkid /dev/sdb
   ```

2. Verify mount point exists:
   ```bash
   ls -la /data/db_postgres/sdb
   ```

3. Check fstab syntax:
   ```bash
   mount -a
   ```

### Permission Denied

1. Check directory permissions:
   ```bash
   ls -la /data/
   ```

2. Fix permissions if needed:
   ```bash
   chmod 755 /data/db_postgres/sdb
   chown root:root /data/db_postgres/sdb
   ```

## Performance Tuning

### XFS Options

Default options: `defaults,noatime`

Additional options for performance:
- `noatime`: Don't update access times
- `nodiratime`: Don't update directory access times
- `largeio`: Use large I/O sizes
- `swalloc`: Allocate swap space

Example:
```bash
UUID=<uuid> /data/db_postgres/sdb xfs defaults,noatime,nodiratime,largeio 0 2
```

### I/O Scheduler

For better performance with XFS:
```bash
echo deadline > /sys/block/sdb/queue/scheduler
```

## Maintenance

### Regular Checks

```bash
# Check all XFS mounts
ansible all -a "df -h | grep xfs" -i ansible/inventory/hosts.yml

# Check disk health
ansible all -a "lsblk" -i ansible/inventory/hosts.yml
```

### Disk Space Monitoring

```bash
ansible all -a "df -h /data" -i ansible/inventory/hosts.yml
```

## Best Practices

1. **Always use UUID** in fstab (not /dev/sdX)
2. **Test mounts** after adding to fstab
3. **Monitor disk space** regularly
4. **Backup critical data** before formatting
5. **Use role_v3** for organization

## Related Documentation

- `ssh_mesh_v3.md`: SSH connectivity
- `reset_hetzner_v3.md`: Server reset process


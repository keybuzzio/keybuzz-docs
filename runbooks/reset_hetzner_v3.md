# Reset Hetzner Servers v3 - Runbook

## Overview

This runbook describes the process for resetting Hetzner Cloud servers in batches, including volume management and XFS filesystem setup.

## Prerequisites

- Access to Hetzner Cloud API (token required)
- install-v3 bastion configured and accessible
- SSH mesh established (see `ssh_mesh_v3.md`)
- Ansible inventory configured
- XFS role available

## Process Overview

The reset process is performed in batches of 5 servers to minimize impact:

1. Read `servers_v3.tsv` to get server list
2. Create batches of 5 servers
3. For each server in batch:
   - Delete existing volumes
   - Recreate volumes with XFS filesystem
   - Reattach volumes to server
   - Reboot server
   - Verify SSH connectivity
   - Apply XFS role to format and mount disks

## Playbook Location

```
/opt/keybuzz/keybuzz-infra/ansible/playbooks/reset_hetzner.yml
```

## Execution

### 1. Set Hetzner API Token

```bash
export HETZNER_API_TOKEN="your-api-token-here"
```

### 2. Review Servers to Reset

The playbook automatically:
- Reads `servers_v3.tsv`
- Excludes bastions (install-01, install-v3)
- Creates batches of 5 servers

### 3. Run Playbook

```bash
cd /opt/keybuzz/keybuzz-infra
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/reset_hetzner.yml
```

### 4. Monitor Progress

The playbook will:
- Display batch information
- Show progress for each server
- Wait for servers to come back online
- Test SSH connectivity
- Apply XFS role

## Batch Processing

### Batch Size

Default batch size: **5 servers**

To change batch size, modify the playbook variable:
```yaml
batch_size: 5
```

### Batch Sequence

Batches are processed sequentially:
- Batch 1: Servers 1-5
- Batch 2: Servers 6-10
- Batch 3: Servers 11-15
- etc.

## Server Reset Steps

For each server:

1. **Delete Volumes**
   - List volumes attached to server
   - Detach volumes
   - Delete volumes

2. **Recreate Volumes**
   - Create new volumes (same size)
   - Format with XFS (handled by XFS role)

3. **Reattach Volumes**
   - Attach volumes to server
   - Verify attachment

4. **Reboot Server**
   - Graceful shutdown
   - Wait for server to be offline
   - Power on server
   - Wait for server to be online

5. **Verify Connectivity**
   - Wait for SSH port (22) to be available
   - Test SSH connection
   - Verify server is responsive

6. **Apply XFS Role**
   - Format disks with XFS
   - Create mount points
   - Mount disks
   - Add to /etc/fstab

## Safety Measures

### Exclusions

The following servers are **never** reset:
- `install-01` (legacy bastion)
- `install-v3` (current bastion)

### Verification Steps

Before proceeding with next batch:
- All servers in current batch must be online
- SSH connectivity verified
- XFS role applied successfully

## Troubleshooting

### Server Not Coming Online

1. Check Hetzner Cloud console
2. Verify server status via API
3. Check network connectivity
4. Review server logs

### Volume Attachment Failed

1. Verify volume exists
2. Check server status
3. Retry attachment
4. Check Hetzner API limits

### SSH Connection Failed After Reboot

1. Wait longer (up to 5 minutes)
2. Check server console
3. Verify network configuration
4. Check firewall rules

### XFS Role Failed

1. Check disk detection
2. Verify disk permissions
3. Review role logs
4. Manually apply XFS role if needed

## Rollback Procedure

If a batch fails:

1. **Stop playbook execution**
2. **Assess damage**: Check which servers were affected
3. **Manual intervention**: Fix affected servers individually
4. **Resume**: Continue with next batch after fixing issues

## Post-Reset Verification

After all batches complete:

```bash
# Test all servers
ansible all -m ping -i ansible/inventory/hosts.yml

# Verify XFS mounts
ansible all -a "df -h | grep xfs" -i ansible/inventory/hosts.yml

# Check disk usage
ansible all -a "lsblk" -i ansible/inventory/hosts.yml
```

## Best Practices

1. **Test on non-production first**: Use staging/dev servers
2. **Backup critical data**: Before resetting production
3. **Monitor during execution**: Watch for errors
4. **Document issues**: Note any problems encountered
5. **Verify after each batch**: Don't proceed if batch fails

## API Rate Limits

Hetzner Cloud API has rate limits:
- Monitor API usage
- Add delays if needed
- Batch processing helps stay within limits

## Maintenance Window

Recommended:
- **Off-peak hours**: Low traffic periods
- **Maintenance window**: Communicate with team
- **Rollback plan**: Always have a plan B

## Related Documentation

- `ssh_mesh_v3.md`: SSH connectivity setup
- `xfs_setup_v3.md`: XFS filesystem configuration


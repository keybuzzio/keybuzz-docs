# Rebuild Servers v3 - Runbook

## Overview

This runbook describes the complete process for rebuilding Hetzner Cloud servers for KeyBuzz v3 infrastructure. This is **PHASE 1** and must be completed **BEFORE** SSH key deployment.

## ⚠️ CRITICAL WARNING

**DO NOT deploy SSH keys until all servers have been successfully rebuilt.**

The rebuild process:
1. Detaches and deletes existing volumes
2. Rebuilds servers with Ubuntu 24.04
3. Servers are accessible via root password (temporary)
4. SSH keys are deployed in PHASE 2 (after rebuild)

## Prerequisites

- Hetzner Cloud API token with appropriate permissions
- Root password for temporary SSH access (set via Hetzner Cloud)
- Ansible installed on install-v3
- Access to `servers_v3.tsv` and `rebuild_order_v3.json`

## File Sources

- **`servers.tsv`**: Source of truth from v2 infrastructure (legacy)
- **`servers_v3.tsv`**: Complete v3 inventory with all servers + role_v3 and logical_name_v3 columns
- **`rebuild_order_v3.json`**: Ordered list of **all rebuildable servers** (47 servers, excluding install-01 and install-v3)

## Process Overview

### Batch Processing

Servers are processed in **batches of 5** to:
- Respect Hetzner API rate limits
- Minimize impact on infrastructure
- Allow for error recovery

### Rebuild Order

The rebuild order is defined in `rebuild_order_v3.json`:
- **Total servers to rebuild**: 47 (all servers except install-01 and install-v3)
- **Total batches**: 10 (batches of 5 servers each, last batch may have fewer)
- **Batch size**: 5 servers per batch

The complete batch list is available in `servers/rebuild_order_v3.json` under the `batches` key. Each batch contains 5 servers (except the last batch which may have fewer).

**Note**: The batch order is optimized to balance server roles and minimize infrastructure impact.

### Excluded Servers

The following servers are **NEVER** rebuilt:
- `install-01` (legacy bastion)
- `install-v3` (current bastion)

## Execution

### 1. Set Environment Variables

```bash
export HETZNER_API_TOKEN="your-hetzner-api-token"
export HETZNER_ROOT_PASSWORD="temporary-root-password"
```

### 2. Verify Files

```bash
cd /opt/keybuzz/keybuzz-infra
ls -la servers/servers_v3.tsv
ls -la servers/rebuild_order_v3.json
ls -la ansible/playbooks/reset_hetzner.yml
```

### 3. Run Playbook

```bash
cd /opt/keybuzz/keybuzz-infra
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/reset_hetzner.yml
```

## Per-Server Process

For each server in a batch:

1. **List Volumes**
   - Query Hetzner API for volumes attached to server
   - Identify volumes by server hostname

2. **Detach Volumes**
   - Detach all volumes from server
   - Wait for detachment to complete (5 seconds)

3. **Delete Volumes**
   - Delete all detached volumes
   - This is **destructive** - ensure backups are taken

4. **Find Server ID**
   - Query Hetzner API for server by public IP
   - Extract server ID

5. **Rebuild Server**
   - Execute rebuild action with Ubuntu 24.04 image
   - Wait for rebuild action to complete (up to 5 minutes)

6. **Wait for Running Status**
   - Poll server status until "running"
   - Maximum wait: 5 minutes

7. **Wait for SSH**
   - Wait for SSH port (22) to be accessible on public IP
   - Maximum wait: 5 minutes

8. **Test SSH Connection**
   - Test SSH with root password
   - Verify server is responsive

## Hetzner API Rate Limits

Hetzner Cloud API has rate limits:
- **Read operations**: 3600 requests/hour
- **Write operations**: 1800 requests/hour

### Batch Processing Benefits

Processing in batches of 5:
- Reduces API calls per minute
- Allows for delays between batches
- Prevents hitting rate limits

### Recommended Delays

- Between servers in batch: 5 seconds
- Between batches: 30 seconds
- After volume operations: 10 seconds

## Verification

### After Each Batch

1. Check server status in Hetzner Cloud console
2. Verify SSH access with password
3. Check server logs for errors

### After All Batches

```bash
# Test connectivity to all rebuilt servers
for ip in $(cat servers/servers_v3.tsv | grep -v "^ENV\|install-01\|install-v3" | cut -f2); do
  echo "Testing $ip..."
  sshpass -p "$HETZNER_ROOT_PASSWORD" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 root@$ip "echo 'OK'"
done
```

## Rollback Procedure

If a batch fails:

1. **Stop playbook execution** (Ctrl+C)
2. **Assess damage**:
   - Check which servers were affected
   - Review Hetzner Cloud console
   - Check API logs

3. **Manual intervention**:
   - Rebuild failed servers individually via Hetzner console
   - Or retry failed batch after fixing issues

4. **Resume**:
   - Continue with next batch after fixing issues
   - Or restart from failed batch

## Troubleshooting

### Volume Detachment Failed

**Symptoms**: Volume remains attached after detach action

**Solutions**:
1. Check volume status in Hetzner console
2. Manually detach via console
3. Retry playbook

### Rebuild Action Failed

**Symptoms**: Rebuild action returns error

**Solutions**:
1. Check server status (may already be rebuilding)
2. Verify image name (ubuntu-24.04)
3. Check API token permissions
4. Retry after delay

### Server Not Coming Online

**Symptoms**: Server status stuck in "starting" or "rebuilding"

**Solutions**:
1. Wait longer (up to 10 minutes)
2. Check Hetzner Cloud console
3. Review server logs
4. Contact Hetzner support if stuck > 15 minutes

### SSH Not Accessible

**Symptoms**: SSH port 22 not responding after rebuild

**Solutions**:
1. Wait longer (up to 5 minutes)
2. Check firewall rules
3. Verify root password is correct
4. Check server console in Hetzner

## Post-Rebuild Checklist

Before proceeding to PHASE 2 (SSH deployment):

- [ ] All servers show "running" status
- [ ] All servers respond to ping (public IP)
- [ ] SSH accessible with root password
- [ ] No volumes attached (will be recreated in PHASE 3)
- [ ] All batches completed successfully

## Next Steps

After successful rebuild:

1. **PHASE 2**: Deploy SSH keys (see `ssh_mesh_v3.md`)
2. **PHASE 3**: Create and attach volumes with XFS (see `xfs_setup_v3.md`)

## Related Documentation

- `ssh_mesh_v3.md`: SSH key deployment (PHASE 2)
- `xfs_setup_v3.md`: XFS volume setup (PHASE 3)
- `reset_hetzner_v3.md`: General reset procedures

## Best Practices

1. **Test on non-production first**: Use staging/dev servers
2. **Backup critical data**: Before rebuilding production
3. **Monitor during execution**: Watch for errors
4. **Document issues**: Note any problems encountered
5. **Verify after each batch**: Don't proceed if batch fails
6. **Maintain batch order**: Don't skip batches

## Maintenance Window

Recommended:
- **Off-peak hours**: Low traffic periods
- **Maintenance window**: Communicate with team
- **Rollback plan**: Always have a plan B
- **Estimated time**: 2-3 hours for all batches


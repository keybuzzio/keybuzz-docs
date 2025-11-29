# Rebuild Complete Summary v3

## Overview

This document summarizes the complete rebuild process for KeyBuzz v3 infrastructure, including all phases and verification steps.

## File Sources

- **`servers.tsv`**: Source of truth from v2 infrastructure (legacy, imported from Infra-v2-legacy)
- **`servers_v3.tsv`**: Complete v3 inventory with all 49 servers, including role_v3 and logical_name_v3 columns
- **`rebuild_order_v3.json`**: Ordered list of **all rebuildable servers** (47 servers, excluding install-01 and install-v3)

## Execution Phases

### ✅ PHASE 0: Preparation and Verification

**Status**: Complete

**Actions**:
- Verified all required files present
- Synchronized repositories (keybuzz-infra, keybuzz-docs)
- Confirmed playbooks and roles available

**Files Verified**:
- `keybuzz-infra/scripts/bootstrap-install-v3.sh`
- `keybuzz-infra/ansible/inventory/hosts.yml`
- `keybuzz-infra/ansible/playbooks/reset_hetzner.yml`
- `keybuzz-infra/ansible/roles/xfs/tasks/main.yml`

### ✅ PHASE 1: Server Rebuild (BEFORE SSH)

**Status**: Ready for execution

**Actions**:
- Created `rebuild_order_v3.json` with 28 servers in 6 batches
- Updated `reset_hetzner.yml` with Hetzner API integration
- Documented process in `rebuild_servers_v3.md`

**Servers to Rebuild**: 47 servers (excluding install-01 and install-v3)

**Batches**: 10 batches of 5 servers each (last batch may have fewer)

The complete batch list is defined in `servers/rebuild_order_v3.json`. Each batch contains up to 5 servers, processed sequentially to respect Hetzner API rate limits.

**Execution**:
```bash
export HETZNER_API_TOKEN="your-token"
export HETZNER_ROOT_PASSWORD="temporary-password"
cd /opt/keybuzz/keybuzz-infra
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/reset_hetzner.yml
```

### ✅ PHASE 2: SSH Mesh Deployment (AFTER REBUILD)

**Status**: Ready for execution

**Actions**:
- Updated `bootstrap-install-v3.sh` to deploy SSH keys using root password
- Added warnings to prevent premature execution
- Updated `ssh_mesh_v3.md` with PHASE 2 instructions

**Execution**:
```bash
export HETZNER_ROOT_PASSWORD="temporary-password"
cd /opt/keybuzz/keybuzz-infra/scripts
bash bootstrap-install-v3.sh
```

**Verification**:
```bash
ansible all -m ping -i /opt/keybuzz/keybuzz-infra/ansible/inventory/hosts.yml
```

### ✅ PHASE 3: XFS Volume Creation (AFTER SSH)

**Status**: Ready for execution

**Actions**:
- Added volume creation section to `reset_hetzner.yml`
- Volumes created from `rebuild_order_v3.json` specifications
- XFS role applied automatically after volume attachment
- Updated `xfs_setup_v3.md` with PHASE 3 instructions

**Execution**:
The volume creation is part of `reset_hetzner.yml` (PHASE 3 section) or can be run separately:

```bash
export HETZNER_API_TOKEN="your-token"
cd /opt/keybuzz/keybuzz-infra
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/reset_hetzner.yml
```

**Verification**:
```bash
ansible all -a "df -h | grep xfs" -i ansible/inventory/hosts.yml
ansible all -a "mount | grep xfs" -i ansible/inventory/hosts.yml
```

## Files Created/Modified

### keybuzz-infra Repository

**Created**:
- `servers/rebuild_order_v3.json` - Server rebuild order and volume specifications
- `ansible/roles/xfs/tasks/main.yml` - XFS role tasks
- `ansible/roles/xfs/README.md` - XFS role documentation
- `ansible/playbooks/reset_hetzner.yml` - Complete rebuild playbook (PHASE 1 + 3)

**Modified**:
- `scripts/bootstrap-install-v3.sh` - SSH deployment with password support
- `ansible/inventory/hosts.yml` - Updated to use private IPs and SSH key

### keybuzz-docs Repository

**Created**:
- `runbooks/rebuild_servers_v3.md` - PHASE 1 documentation
- `runbooks/rebuild_complete_summary_v3.md` - This summary

**Modified**:
- `runbooks/ssh_mesh_v3.md` - Added PHASE 2 warnings and instructions
- `runbooks/xfs_setup_v3.md` - Added PHASE 3 prerequisites

## Verification Checklist

### After PHASE 1 (Rebuild)

- [ ] All 47 servers show "running" status in Hetzner Cloud
- [ ] All servers respond to ping (public IP)
- [ ] SSH accessible with root password on all servers
- [ ] No volumes attached (will be created in PHASE 3)

### After PHASE 2 (SSH Deployment)

- [ ] SSH key generated at `/root/.ssh/id_rsa_keybuzz_v3`
- [ ] All servers accessible via SSH key (public IP)
- [ ] All servers accessible via SSH key (private IP)
- [ ] Ansible ping successful: `ansible all -m ping`

### After PHASE 3 (XFS Volumes)

- [ ] All volumes created and attached (check Hetzner console)
- [ ] XFS filesystems formatted on all data disks
- [ ] Mount points created: `/data/<role_v3>/<disk>/`
- [ ] Mounts persistent in `/etc/fstab`
- [ ] Write tests successful on all mounts
- [ ] Disk usage verified: `df -h | grep xfs`

## Server Status Summary

### Expected Server Counts

- **Total servers**: 49 (from servers_v3.tsv)
- **Excluded from rebuild**: 2 (install-01, install-v3)
- **Servers to rebuild**: 47
- **Batches**: 10 (batches of 5, last batch may have fewer)

### Volume Specifications

Volumes are created with sizes from `rebuild_order_v3.json`:
- **k8s_master**: 20GB
- **k8s_worker**: 50GB
- **db_postgres**: 100GB
- **db_mariadb**: 100GB
- **redis**: 20GB
- **rabbitmq**: 30GB
- **minio**: 200GB
- **backup**: 500GB
- **monitoring**: 50GB
- **vault**: 20GB
- **builder**: 100GB

## SSH Mesh Status

### Configuration

- **SSH Key**: `/root/.ssh/id_rsa_keybuzz_v3`
- **Connection Method**: Private IPs (10.0.0.x)
- **Authentication**: SSH key-based
- **Ansible Inventory**: Uses private IPs for `ansible_host`

### Connectivity

After PHASE 2, verify:
```bash
# Test all servers
ansible all -m ping -i ansible/inventory/hosts.yml

# Test specific groups
ansible k8s_masters -m ping
ansible db_postgres -m ping
```

## XFS Mount Points

### Expected Mount Structure

After PHASE 3, each server should have:
```
/data/<role_v3>/<volume_name>/
```

Examples:
- `/data/db_postgres/db-master-01-data/`
- `/data/k8s_worker/k8s-worker-01-data/`
- `/data/minio/minio-01-data/`

### Verification Commands

```bash
# Check all XFS mounts
ansible all -a "df -h | grep xfs" -i ansible/inventory/hosts.yml

# Check mount points
ansible all -a "mount | grep xfs" -i ansible/inventory/hosts.yml

# Check fstab entries
ansible all -a "cat /etc/fstab | grep xfs" -i ansible/inventory/hosts.yml
```

## Next Steps

### Immediate Next Steps

1. **Execute PHASE 1**: Rebuild all servers
   ```bash
   export HETZNER_API_TOKEN="..."
   export HETZNER_ROOT_PASSWORD="..."
   ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/reset_hetzner.yml
   ```

2. **Execute PHASE 2**: Deploy SSH keys
   ```bash
   export HETZNER_ROOT_PASSWORD="..."
   bash /opt/keybuzz/keybuzz-infra/scripts/bootstrap-install-v3.sh
   ```

3. **Execute PHASE 3**: Create volumes and apply XFS
   ```bash
   # Volume creation is in reset_hetzner.yml PHASE 3 section
   # Or run separately after verifying SSH mesh
   ```

### Recommended Next Steps (kubeadm)

After all phases complete:

1. **Prepare kubeadm cluster**:
   - Install kubeadm on k8s_masters
   - Initialize first master
   - Join other masters and workers

2. **Deploy ArgoCD**:
   - Install ArgoCD in cluster
   - Configure GitOps repositories
   - Deploy applications

3. **Setup PostgreSQL 17 + Patroni**:
   - Deploy Patroni cluster
   - Configure pgvector extension
   - Setup HAProxy for database load balancing

4. **Deploy monitoring**:
   - Install Prometheus/Grafana
   - Configure alerting
   - Setup log aggregation

## Troubleshooting

### Common Issues

1. **Server rebuild fails**: Check Hetzner API token and permissions
2. **SSH deployment fails**: Verify root password and server accessibility
3. **Volume creation fails**: Check API rate limits and volume quotas
4. **XFS role fails**: Verify disk detection and permissions

### Support Documentation

- `rebuild_servers_v3.md` - PHASE 1 troubleshooting
- `ssh_mesh_v3.md` - PHASE 2 troubleshooting
- `xfs_setup_v3.md` - PHASE 3 troubleshooting

## Success Criteria

All phases are successful when:

✅ All servers rebuilt and running
✅ SSH mesh operational (all servers accessible via private IP)
✅ All volumes created and attached
✅ All XFS filesystems formatted and mounted
✅ All mounts persistent in /etc/fstab
✅ Write tests successful on all mounts
✅ Ansible can connect to all servers
✅ Ready for kubeadm cluster setup


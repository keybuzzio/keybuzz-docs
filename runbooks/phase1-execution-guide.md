# PHASE 1 Execution Guide

## Overview

This guide describes how to execute PHASE 1 (complete server rebuild) on install-v3.

## Prerequisites

- Access to install-v3 (46.62.171.61)
- KeyBuzz v3 repositories cloned on install-v3
- hcloud CLI installed
- Ansible installed with `community.general` collection

## Quick Start

On install-v3, execute:

```bash
cd /opt/keybuzz/keybuzz-infra
bash scripts/execute-phase1.sh
```

This script will:
1. Setup Hetzner Cloud token
2. Rename PostgreSQL servers
3. Launch PHASE 1 rebuild

## Manual Steps

### 1. Setup Hetzner Token

```bash
cd /opt/keybuzz/keybuzz-infra
bash scripts/setup-hetzner-token.sh
```

This creates:
- `/opt/keybuzz/credentials/hcloud.env` (chmod 600)
- `~/.config/hcloud/cli.toml` (chmod 600)

### 2. Rename PostgreSQL Servers

```bash
source /opt/keybuzz/credentials/hcloud.env
export HETZNER_API_TOKEN
bash scripts/rename-postgres-servers.sh
```

This renames:
- `db-master-01` → `db-postgres-01`
- `db-slave-01` → `db-postgres-02`
- `db-slave-02` → `db-postgres-03`

### 3. Verify Inventory

```bash
cd /opt/keybuzz/keybuzz-infra
python3 scripts/generate_inventory.py > ansible/inventory/hosts.yml
python3 scripts/generate_rebuild_order.py
```

### 4. Launch PHASE 1 Rebuild

```bash
cd /opt/keybuzz/keybuzz-infra
source /opt/keybuzz/credentials/hcloud.env
export HETZNER_API_TOKEN
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/reset_hetzner.yml
```

## Expected Results

- **47 servers** rebuilt (excluding install-01 and install-v3)
- **10 batches** processed
- All servers in "running" status
- Port 22 open on all servers
- Volumes detached and deleted (will be recreated in PHASE 3)

## Verification

After rebuild, generate a report:

```bash
bash /opt/keybuzz/keybuzz-infra/scripts/phase1-report.sh
```

## Troubleshooting

### Token Issues

If hcloud fails:
```bash
source /opt/keybuzz/credentials/hcloud.env
export HETZNER_API_TOKEN
hcloud server list
```

### Playbook Errors

Check Ansible logs and verify:
- Token is accessible
- hcloud modules are available
- Inventory is correct

### Server Not Rebuilding

Check Hetzner Cloud console and verify:
- Server exists
- API token has permissions
- No rate limiting issues

## Next Steps

After PHASE 1 completes successfully:
- Review the report
- Verify all servers are running
- Proceed to PHASE 2 (SSH mesh deployment)


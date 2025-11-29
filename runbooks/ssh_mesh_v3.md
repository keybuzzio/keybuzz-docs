# SSH Mesh v3 - Runbook

## Overview

The SSH mesh for KeyBuzz v3 enables secure communication between install-v3 bastion and all infrastructure servers using private IP addresses.

## Architecture

- **Bastion**: install-v3 (10.0.0.251)
- **SSH Key**: `/root/.ssh/id_rsa_keybuzz_v3`
- **Connection Method**: Private IP addresses (10.0.0.x)
- **Authentication**: SSH key-based (no passwords)

## Initial Setup

### 1. Bootstrap install-v3

Run the bootstrap script on install-v3:

```bash
cd /opt/keybuzz/keybuzz-infra/scripts
bash bootstrap-install-v3.sh
```

This script will:
- Generate SSH key at `/root/.ssh/id_rsa_keybuzz_v3`
- Deploy the public key to all servers via public IP
- Test connectivity via private IP

### 2. Verify SSH Key Generation

```bash
ls -la /root/.ssh/id_rsa_keybuzz_v3*
```

Expected output:
- `id_rsa_keybuzz_v3` (private key, 600 permissions)
- `id_rsa_keybuzz_v3.pub` (public key, 644 permissions)

### 3. Manual Key Deployment (if automated fails)

If automated deployment fails for some servers:

```bash
# Display public key
cat /root/.ssh/id_rsa_keybuzz_v3.pub

# Copy to target server (replace with actual IP and user)
ssh-copy-id -i /root/.ssh/id_rsa_keybuzz_v3.pub root@<PUBLIC_IP>
```

## Ansible Inventory Configuration

The inventory file (`ansible/inventory/hosts.yml`) is configured to:
- Use private IPs for `ansible_host`
- Use SSH key: `/root/.ssh/id_rsa_keybuzz_v3`
- Store public IPs in `ip_public` variable

## Testing Connectivity

### Test Single Server

```bash
ssh -i /root/.ssh/id_rsa_keybuzz_v3 -o StrictHostKeyChecking=no root@10.0.0.100 "echo 'Connection successful'"
```

### Test All Servers via Ansible

```bash
cd /opt/keybuzz/keybuzz-infra
ansible all -m ping -i ansible/inventory/hosts.yml
```

Expected: All servers should return `pong`

### Test Specific Group

```bash
ansible k8s_masters -m ping -i ansible/inventory/hosts.yml
ansible db_postgres -m ping -i ansible/inventory/hosts.yml
```

## Troubleshooting

### Connection Refused

1. Check if server is online:
   ```bash
   ping 10.0.0.100
   ```

2. Check SSH service:
   ```bash
   ssh -i /root/.ssh/id_rsa_keybuzz_v3 root@10.0.0.100 "systemctl status ssh"
   ```

3. Verify key permissions:
   ```bash
   ls -la /root/.ssh/id_rsa_keybuzz_v3
   # Should be 600
   ```

### Permission Denied

1. Verify public key is in authorized_keys:
   ```bash
   ssh -i /root/.ssh/id_rsa_keybuzz_v3 root@10.0.0.100 "cat ~/.ssh/authorized_keys | grep install-v3"
   ```

2. Re-deploy key:
   ```bash
   ssh-copy-id -i /root/.ssh/id_rsa_keybuzz_v3.pub root@<PUBLIC_IP>
   ```

### Private IP Not Reachable

1. Check network connectivity:
   ```bash
   ping 10.0.0.100
   ```

2. Verify routing:
   ```bash
   ip route show
   ```

3. Check firewall rules (if applicable)

## Security Notes

- SSH key is stored in `/root/.ssh/` with restricted permissions
- Private IPs are used for all internal communication
- Public IPs are only used for initial key deployment
- Key rotation should be performed periodically

## Key Rotation

To rotate SSH keys:

1. Generate new key:
   ```bash
   ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa_keybuzz_v3_new -N "" -C "install-v3-keybuzz-v3"
   ```

2. Deploy new key to all servers

3. Update inventory to use new key

4. Remove old key from servers

5. Rename new key to replace old one

## Maintenance

- Regularly test connectivity: `ansible all -m ping`
- Monitor SSH connection logs
- Keep SSH key secure and backed up (encrypted)


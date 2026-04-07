# Ansible Nginx Version Update Playbook

Zero-downtime Ansible playbook for updating Nginx on production application servers.

## Overview

This playbook safely updates Nginx to a specific version on live servers with:

- **Pre-flight checks** - Verify current version, backup configs
- **Health verification** - Ensure servers are healthy before update
- **Staged rollout** - Update one server at a time (serial execution)
- **Config backup** - Automatic backup of nginx.conf before changes
- **Service validation** - Test Nginx after update before moving to next server
- **Rollback capability** - Restore from backup if update fails

## Use Case

**Scenario**: Your application is running behind Nginx reverse proxy across multiple servers. A security patch requires updating Nginx from 1.18.x to 1.24.x without downtime.

**This playbook handles**:
- Update Nginx package
- Preserve custom configurations
- Restart service safely
- Verify each server before proceeding to next

## Prerequisites

- Ansible >= 2.10 installed on control machine
- SSH access to target servers
- Sudo privileges on target servers
- Servers running Ubuntu/Debian (apt-based)

## Quick Start

### 1. Configure Inventory

Edit `inventory.ini` with your server IPs:

```ini
[webservers]
web1.example.com ansible_host=10.0.1.10
web2.example.com ansible_host=10.0.1.11
web3.example.com ansible_host=10.0.1.12

[webservers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/prod-key.pem
```

### 2. Test Connection

```bash
ansible -i inventory.ini webservers -m ping
```

### 3. Dry Run (Check Mode)

```bash
ansible-playbook -i inventory.ini nginx-update.yml --check
```

### 4. Execute Update

```bash
ansible-playbook -i inventory.ini nginx-update.yml
```

## Playbook Flow

```
1. Pre-Flight Checks
   ├─ Gather current Nginx version
   ├─ Check if Nginx is running
   └─ Verify backup directory exists

2. Backup Phase
   ├─ Backup /etc/nginx/nginx.conf
   ├─ Backup /etc/nginx/sites-available/
   └─ Create timestamped backup archive

3. Update Phase (Serial: 1 server at a time)
   ├─ Update apt cache
   ├─ Install/upgrade Nginx package
   ├─ Validate configuration syntax
   └─ Reload Nginx service

4. Verification Phase
   ├─ Check Nginx service status
   ├─ Verify HTTP response (curl localhost)
   ├─ Compare versions (before/after)
   └─ Report success/failure

5. Post-Update Tasks
   └─ Display summary across all servers
```


## Files

- `nginx-update.yml` - Main update playbook
- `nginx-rollback.yml` - Rollback playbook (restores from backup)
- `inventory.ini` - Server inventory (add your IPs)
- `ansible.cfg` - Ansible configuration
- `README.md` - This file

## Example Output

```
PLAY [Update Nginx on web servers] *******************************************

TASK [Check current Nginx version] *******************************************
ok: [web1.example.com] => (item=nginx version: nginx/1.18.0)

TASK [Backup Nginx configuration] ********************************************
changed: [web1.example.com]

TASK [Update Nginx package] **************************************************
changed: [web1.example.com]

TASK [Validate Nginx config] *************************************************
ok: [web1.example.com] => (item=nginx: configuration file /etc/nginx/nginx.conf test is successful)

TASK [Reload Nginx service] **************************************************
changed: [web1.example.com]

PLAY RECAP *******************************************************************
web1.example.com           : ok=8    changed=3    unreachable=0    failed=0
web2.example.com           : ok=8    changed=3    unreachable=0    failed=0
web3.example.com           : ok=8    changed=3    unreachable=0    failed=0
```

## Integration with CI/CD

This playbook can be triggered from:
- Jenkins pipeline
- GitLab CI/CD
- Bitbucket Pipelines
- GitHub Actions

Example Jenkins stage:
```groovy
stage('Update Nginx') {
    steps {
        sh 'ansible-playbook -i inventory.ini nginx-update.yml'
    }
}
```

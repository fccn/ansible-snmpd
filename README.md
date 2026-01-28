# Ansible Role: snmpd

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

An Ansible role to install and configure Net-SNMP daemon (snmpd) for monitoring Linux servers with SNMP-based monitoring tools like Icinga, Nagios, and Cacti.

## Overview

This role automates the deployment and configuration of snmpd across Debian/Ubuntu and RedHat/CentOS systems. It provides a secure, configurable SNMP agent that enables network monitoring systems to collect system metrics, disk usage, process status, and custom monitoring data.

SNMP (Simple Network Management Protocol) allows monitoring systems to:
- **Nagios/Icinga**: Send alerts when thresholds are exceeded or services fail
- **Cacti**: Generate time-series graphs for performance trending
- **Custom Monitoring**: Execute custom scripts and expose metrics via SNMP

## Use Cases

### Infrastructure Monitoring
Deploy snmpd across your server fleet to enable centralized monitoring of system health, resource usage, and service availability.

### Disk Space Monitoring
Monitor disk usage on critical mount points with configurable thresholds to prevent storage-related outages.

### Custom Metrics
Extend SNMP with custom scripts to expose application-specific metrics through the standard SNMP protocol.

### Multi-Environment Monitoring
Use per-host or per-environment variables to customize monitoring parameters (contact info, location, disk paths) while maintaining consistent base configuration.

## Requirements

- Ansible 2.9 or higher
- Linux-based systems (Ubuntu, Debian, CentOS, RHEL)
- Root or sudo access on target hosts
- Network access from monitoring systems to target hosts on UDP port 161

## Role Variables

### Security Variables

**Required - Should be overridden with vault-encrypted values:**

- `snmpd_community_string` - SNMP community string (password) for authentication
  - Default: `"public"`
  - **Security Warning**: Override this with a strong secret and use Ansible Vault

### Network Variables

- `snmpd_network` - Network CIDR allowed to access SNMP
  - Default: `"193.136.7.0/24"`
  - Example: `"10.0.0.0/8"` or `"192.168.1.0/24"`

### System Information Variables

- `snmpd_syscontact` - System contact email address
  - Default: `"admin@example.com"`
  
- `snmpd_syslocation` - System location description
  - Default: `"{{ inventory_hostname }}"`
  
- `snmpd_sysdescr` - System description
  - Default: `"{{ ansible_distribution }} {{ ansible_distribution_version }}"`

### Access Control Variables

- `snmpd_group_name` - SNMP access group name
  - Default: `"MyROGroup"`

### Monitoring Variables

- `snmpd_disk_path` - Disk mount point to monitor
  - Default: `"/"`
  - For multiple disks, see "Monitoring Multiple Disks" section below
  
- `snmpd_disk_threshold` - Disk usage warning threshold
  - Default: `"80%"`

- `snmpd_configs` - List of additional monitoring configurations
  - Optional: Add custom monitoring scripts or OID extensions

### Package Variables

- `snmpd_packages_debian` - Packages for Debian/Ubuntu
  - Default: `["snmpd", "snmp", "snmp-mibs-downloader"]`
  
- `snmpd_packages_redhat` - Packages for RedHat/CentOS
  - Default: `["net-snmp"]`

### Post-Configuration Variables (Debian Only)

- `snmpd_post_debian_snmpd_user` - SNMP daemon user
  - Default: `"Debian-snmp"`
  
- `snmpd_post_debian_snmpd_additional_group` - Additional group for SNMP user
  - Default: `"docker"`
  
- `snmpd_post_debian_configure_user_docker` - Whether to add user to docker group
  - Default: `true` (for Ubuntu 20.04+)

See [defaults/main.yml](defaults/main.yml) for all available variables.

## Usage

### Basic Playbook Example

```yaml
---
- name: Configure SNMP monitoring
  hosts: monitored_servers
  become: true
  roles:
    - snmpd
  vars:
    snmpd_community_string: "{{ vault_snmpd_community_string }}"
    snmpd_network: "10.0.0.0/8"
    snmpd_syscontact: "ops@example.com"
```

### Environment-Specific Configuration

Use group_vars to customize per environment:

**group_vars/production/snmpd.yml:**
```yaml
snmpd_community_string: "{{ vault_snmpd_community_string }}"
snmpd_network: "10.100.0.0/16"
snmpd_syscontact: "ops-prod@example.com"
snmpd_disk_threshold: "85%"
```

**group_vars/all/vault.yml** (encrypted with `ansible-vault`):
```yaml
vault_snmpd_community_string: "your-secure-community-string-here"
```

### Monitoring Multiple Disks

To monitor multiple disk mount points, use the `snmpd_configs` list:

```yaml
snmpd_configs:
  - "disk /data 90%"
  - "disk /backup 95%"
  - "disk /var/log 80%"
```

Or use dynamic configuration based on mounted filesystems:

```yaml
# In your playbook or group_vars
snmpd_configs: |
  {% set disks = [] %}
  {% for mount in ansible_mounts %}
  {%   if mount.mount != '/' and mount.mount.startswith('/') and not mount.mount.startswith('/snap') %}
  {%     set _ = disks.append('disk ' + mount.mount + ' 80%') %}
  {%   endif %}
  {% endfor %}
  {{ disks }}
```

### Custom Monitoring Scripts

Add custom SNMP extensions for application-specific monitoring:

```yaml
snmpd_configs:
  - "extend redis_health /usr/local/bin/check_redis.sh"
  - "extend app_status /usr/local/bin/check_app_status.sh"
  - "extend backup_age /usr/local/bin/check_backup_age.sh"
```

## Configuration Examples

### Simple Monitoring Setup

```yaml
---
- name: Deploy SNMP monitoring
  hosts: web_servers
  become: true
  roles:
    - snmpd
  vars:
    snmpd_community_string: "{{ vault_snmpd_community_string }}"
    snmpd_network: "192.168.100.0/24"
    snmpd_syscontact: "webteam@example.com"
    snmpd_syslocation: "Datacenter A - Web Cluster"
```

### Database Servers with Custom Monitoring

```yaml
---
- name: Configure database server monitoring
  hosts: db_servers
  become: true
  roles:
    - snmpd
  vars:
    snmpd_community_string: "{{ vault_snmpd_community_string }}"
    snmpd_network: "10.50.0.0/16"
    snmpd_syscontact: "dba@example.com"
    snmpd_syslocation: "{{ datacenter_name }} - {{ inventory_hostname }}"
    snmpd_disk_threshold: "90%"
    snmpd_configs:
      - "disk /data 85%"
      - "disk /backups 95%"
      - "extend mysql_status /usr/local/bin/check_mysql.sh"
      - "extend replication_lag /usr/local/bin/check_replication.sh"
```

### Per-Host Customization

For servers with unique monitoring requirements, use host_vars:

**host_vars/storage01.example.com.yml:**
```yaml
snmpd_syslocation: "Storage Array - Primary"
snmpd_configs:
  - "disk / 95%"
  - "disk /storage1 98%"
  - "disk /storage2 98%"
  - "disk /storage3 98%"
  - "extend raid_status /bin/cat /proc/mdstat"
```

## Testing

### Test SNMP Access

Test SNMP connectivity from your monitoring server:

```bash
# Basic connectivity test
snmpwalk -v 2c -c YOUR_COMMUNITY_STRING target_host SNMPv2-MIB::system

# Test disk monitoring
snmpwalk -v 2c -c YOUR_COMMUNITY_STRING target_host UCD-SNMP-MIB::dskTable

# Test custom extensions
snmpwalk -v 2c -c YOUR_COMMUNITY_STRING target_host NET-SNMP-EXTEND-MIB::nsExtendOutputFull

# Get specific disk usage
snmpget -v 2c -c YOUR_COMMUNITY_STRING target_host UCD-SNMP-MIB::dskPercent.1
```

### Test from Localhost

On the target server itself:

```bash
# Test local SNMP agent
snmpwalk -v 2c -c YOUR_COMMUNITY_STRING localhost SNMPv2-MIB::system

# Check disk monitoring configuration
snmpwalk -v 2c -c YOUR_COMMUNITY_STRING localhost UCD-SNMP-MIB::dskPath
```

### Verify Configuration

Check the generated configuration file:

```bash
# View snmpd configuration
sudo cat /etc/snmp/snmpd.conf

# Check snmpd service status
sudo systemctl status snmpd

# View snmpd logs
sudo journalctl -u snmpd -n 50
```

## Security Best Practices

### 1. Use Strong Community Strings

Never use default community strings like "public" or "private":

```yaml
# Bad
snmpd_community_string: "public"

# Good - Use ansible-vault
snmpd_community_string: "{{ vault_snmpd_community_string }}"
```

Create encrypted vault file:

```bash
ansible-vault create group_vars/all/vault.yml
```

Add:
```yaml
vault_snmpd_community_string: "aB9$xK2m@pL7qR5"
```

### 2. Restrict Network Access

Limit SNMP access to your monitoring network:

```yaml
# Restrict to monitoring subnet
snmpd_network: "10.200.0.0/24"

# Or use firewall rules in addition
```

### 3. Read-Only Access

This role configures read-only access by default (`none` for write permissions in access rules). Never grant write access unless absolutely necessary.

### 4. Monitor the Monitors

Ensure your monitoring systems themselves are monitored and their credentials are rotated regularly.

## Important Notes

### OS Family Detection

This role uses `ansible_os_family` to automatically select the correct package list and configuration:
- **Debian** family: Uses `setup-Debian.yml` and `post-Debian.yml`
- **RedHat** family: Uses `setup-RedHat.yml` and `post-RedHat.yml`

### Backup Configuration

The role automatically backs up the existing snmpd configuration before making changes (`backup: yes` in template task).

### Service Restart

The role includes a handler to restart snmpd when configuration changes. Ensure your monitoring systems can handle brief service interruptions.

## Dependencies

None.

## Installation

### Using ansible-galaxy with requirements.yml

Add to your `requirements.yml`:

```yaml
roles:
  - src: git+https://github.com/fccn/ansible-snmpd.git
    version: main
    name: snmpd
```

Install:
```bash
ansible-galaxy install -p vendor/roles -r requirements.yml
```

### Using git submodules

```bash
cd your-ansible-project
git submodule add https://github.com/fccn/ansible-snmpd.git roles/snmpd
```

## Troubleshooting

### SNMP Timeout Errors

If `snmpwalk` times out:
1. Check snmpd service is running: `systemctl status snmpd`
2. Verify firewall allows UDP 161: `sudo ufw status` or `sudo firewall-cmd --list-all`
3. Check community string matches
4. Verify network access in snmpd.conf

### Permission Denied Errors

If SNMP queries return "noSuchObject" or permission errors:
1. Verify the MIB/OID is included in the view definition
2. Check community string has proper access rights
3. Ensure the queried OID exists: `snmpwalk -v 2c -c COMMUNITY localhost .1`

### Custom Scripts Not Working

If custom extensions don't execute:
1. Verify script has execute permissions: `chmod +x /path/to/script`
2. Test script manually as snmpd user: `sudo -u Debian-snmp /path/to/script`
3. Check script output is not too large (snmpd has size limits)
4. Review snmpd logs: `journalctl -u snmpd -f`

## License

[GNU General Public License v3.0](LICENSE)

## Author Information

Created and maintained by **Ivo Branco** at [FCCN](https://www.fccn.pt/).

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

---

**Repository**: [fccn/ansible-snmpd](https://github.com/fccn/ansible-snmpd)

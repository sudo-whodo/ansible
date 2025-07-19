# Golden Path Systemd Role

This Ansible role provides a standardized way to manage systemd services with local user/group management and drop-in configurations. It follows best practices for security and service management.

## Features

- Creates local system users and groups for service ownership
- Manages systemd service files with comprehensive security settings
- Supports drop-in configuration files for service customization
- Creates necessary directories with proper ownership and permissions
- Validates systemd service configuration syntax
- Provides comprehensive logging and monitoring capabilities

## Requirements

- Ansible 2.9 or higher
- Target systems with systemd (RHEL/CentOS 7+, Ubuntu 18.04+, Debian 10+)
- Root or sudo access on target systems

## Role Variables

### Service Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `systemd_service_name` | `"app"` | Name of the systemd service |
| `systemd_service_description` | `"Default Application Service"` | Service description |
| `systemd_service_type` | `"simple"` | Systemd service type |
| `systemd_service_restart` | `"always"` | Restart policy |
| `systemd_service_restart_sec` | `"10"` | Restart delay in seconds |
| `systemd_service_exec_start` | `/opt/{{ systemd_service_name }}/bin/{{ systemd_service_name }}` | Command to start the service |
| `systemd_service_exec_reload` | `"/bin/kill -HUP $MAINPID"` | Command to reload the service |
| `systemd_service_wanted_by` | `"multi-user.target"` | Systemd target for service |

### User and Group Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `systemd_service_user` | `"{{ systemd_service_name }}"` | Service user name |
| `systemd_service_group` | `"{{ systemd_service_name }}"` | Service group name |
| `systemd_service_user_shell` | `"/sbin/nologin"` | User shell |
| `systemd_service_user_home` | `/opt/{{ systemd_service_name }}` | User home directory |
| `systemd_service_user_system` | `true` | Create as system user |
| `systemd_service_group_system` | `true` | Create as system group |

### Directory Configuration

The role creates the following directories by default with setgid permissions for group inheritance:
- `/opt/{{ systemd_service_name }}` - Main application directory (mode: 2755)
- `/opt/{{ systemd_service_name }}/bin` - Binary files (mode: 2755)
- `/opt/{{ systemd_service_name }}/config` - Configuration files (mode: 2750)
- `/var/log/{{ systemd_service_name }}` - Log files (mode: 2755)
- `/var/lib/{{ systemd_service_name }}` - Data files (mode: 2750)
- `/etc/systemd/system/{{ systemd_service_name }}.service.d/` - Drop-in directory (mode: 2755)

**Note**: The setgid bit (2xxx permissions) ensures that new files and directories created within these directories will automatically inherit the group ownership of the parent directory.

### Drop-in Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `systemd_service_drop_in_enabled` | `true` | Enable drop-in configuration |
| `systemd_service_drop_in_name` | `"override"` | Drop-in file name |
| `systemd_service_drop_in_config` | See defaults | Drop-in configuration content |

### Logging Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `systemd_service_standard_output` | `"journal"` | Standard output destination |
| `systemd_service_standard_error` | `"journal"` | Standard error destination |
| `systemd_service_syslog_identifier` | `"{{ systemd_service_name }}"` | Legacy syslog identifier |
| `systemd_service_journalctl_log_tag` | `"customer"` | Journalctl log tag for filtering logs |

### Timer Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `systemd_service_timer_enabled` | `false` | Enable systemd timer creation for scheduled tasks |

**Note**: Timers are disabled by default. Set `systemd_service_timer_enabled: true` to create a systemd timer for scheduled execution of the service.

### Sudo and Polkit Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `systemd_service_sudo_enabled` | `true` | Enable sudo access for service group (mutually exclusive with polkit) |
| `systemd_service_polkit_enabled` | `false` | Enable polkit rules for service group (mutually exclusive with sudo) |

**Note**: Only one of `systemd_service_sudo_enabled` or `systemd_service_polkit_enabled` should be set to `true`. They are mutually exclusive.

#### Sudo Privileges (when enabled)

When `systemd_service_sudo_enabled` is `true`, the service group gets passwordless sudo access to:

**Service Management:**
- `systemctl start/stop/restart/reload/status/enable/disable {{ systemd_service_name }}.service`
- `systemctl daemon-reload` and `systemctl daemon-reexec`

**User Switching:**
- `su - {{ systemd_service_user }}` and `su {{ systemd_service_user }}`
- Full access as the service user: `sudo -u {{ systemd_service_user }} <command>`

**Log Access:**
- `journalctl -u {{ systemd_service_name }}.service*`
- `journalctl -t {{ systemd_service_journalctl_log_tag }}*`

**Network Monitoring:**
- `ss` and `netstat` commands (both `/bin` and `/usr/bin` locations)

**Firewall Management:**
- `iptables` commands (RHEL/CentOS/Ubuntu)
- `ufw` commands (Ubuntu/Debian)
- `firewall-cmd` commands (RHEL/CentOS with firewalld)

#### Polkit Rules (when enabled)

When `systemd_service_polkit_enabled` is `true`, the service group gets passwordless access to:

**Service Management:**
- Manage systemd units for the specific service
- Reload systemd daemon
- Manage systemd socket for the service

**Network Monitoring (Debian/Ubuntu only):**
- NetworkManager network control actions

### Security Settings

The role includes comprehensive security settings:
- `NoNewPrivileges=true`
- `ProtectSystem=strict`
- `ProtectHome=true`
- `PrivateTmp=true`
- `PrivateDevices=true`
- And many more security hardening options

## Dependencies

None.

## Example Playbook

### Basic Usage

```yaml
---
- hosts: servers
  become: true
  roles:
    - golden_path_systemd
```

### Custom Service Configuration

```yaml
---
- hosts: servers
  become: true
  roles:
    - role: golden_path_systemd
      vars:
        systemd_service_name: "myapp"
        systemd_service_description: "My Custom Application"
        systemd_service_exec_start: "/opt/myapp/bin/myapp --config /opt/myapp/config/app.conf"
        systemd_service_environment_vars:
          APP_ENV: "production"
          LOG_LEVEL: "info"
```

### Custom Drop-in Configuration

```yaml
---
- hosts: servers
  become: true
  roles:
    - role: golden_path_systemd
      vars:
        systemd_service_name: "webapp"
        systemd_service_drop_in_config: |
          [Service]
          # Custom resource limits
          LimitNOFILE=100000
          LimitNPROC=8192

          # Custom environment
          Environment="JAVA_OPTS=-Xmx2g -Xms1g"

          [Unit]
          # Additional dependencies
          After=postgresql.service
          Wants=postgresql.service
```

### Disable Drop-in Configuration

```yaml
---
- hosts: servers
  become: true
  roles:
    - role: golden_path_systemd
      vars:
        systemd_service_name: "simpleapp"
        systemd_service_drop_in_enabled: false
```

## Directory Structure

After running this role, the following structure will be created:

```
/opt/{{ systemd_service_name }}/
├── bin/                    # Executable files
├── config/                 # Configuration files
/var/log/{{ systemd_service_name }}/  # Log files
/var/lib/{{ systemd_service_name }}/  # Data files
/etc/systemd/system/
├── {{ systemd_service_name }}.service
└── {{ systemd_service_name }}.service.d/
    └── override.conf       # Drop-in configuration
```

## Service Management

After the role completes, you can manage the service using standard systemctl commands:

```bash
# Check service status
sudo systemctl status {{ systemd_service_name }}

# Start/stop/restart service
sudo systemctl start {{ systemd_service_name }}
sudo systemctl stop {{ systemd_service_name }}
sudo systemctl restart {{ systemd_service_name }}

# Enable/disable service
sudo systemctl enable {{ systemd_service_name }}
sudo systemctl disable {{ systemd_service_name }}

# View logs
sudo journalctl -u {{ systemd_service_name }} -f

# View logs filtered by custom tag
sudo journalctl -t {{ systemd_service_journalctl_log_tag }} -f
```

## Security Considerations

This role implements several security best practices:

1. **Principle of Least Privilege**: Services run as dedicated system users with minimal permissions
2. **Filesystem Protection**: Uses systemd security features to restrict filesystem access
3. **Resource Limits**: Configurable resource limits to prevent resource exhaustion
4. **Network Restrictions**: Limits network access to necessary address families
5. **Capability Restrictions**: Removes unnecessary capabilities from the service

## Validation

The role includes validation steps:
- Systemd service file syntax validation using `systemd-analyze verify`
- Drop-in configuration validation
- Directory and file permission verification

## License

MIT

## Author Information

This role was created as part of the Golden Path Systemd project for standardized service management.

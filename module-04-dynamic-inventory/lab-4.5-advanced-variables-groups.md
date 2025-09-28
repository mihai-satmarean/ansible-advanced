# Lab 4.5: Advanced Variables and Group Management

## Objective
Master advanced variable inheritance and dynamic group creation for complex enterprise inventory management.

## Duration
20 minutes

## Prerequisites
- Completed Labs 4.1-4.4
- Understanding of Ansible variable precedence
- Knowledge of group and host variable concepts

## Lab Setup

```bash
cd ~/ansible-labs/module-04
mkdir -p lab-4.5
cd lab-4.5

# Create directory structure
mkdir -p {group_vars,host_vars,inventories,configs,scripts}
```

## Exercise 1: Advanced Variable Inheritance (8 minutes)

### Task: Implement complex variable inheritance patterns

Create hierarchical group structure:

```bash
# Create base inventory
cat > inventories/enterprise.ini << 'EOF'
# Enterprise Hierarchical Inventory

# Geographic regions
[europe]
[north_america]
[asia_pacific]

# Environments within regions
[europe_production]
eu-prod-web-01 ansible_host=10.1.1.10
eu-prod-web-02 ansible_host=10.1.1.11
eu-prod-db-01 ansible_host=10.1.2.10

[europe_staging]
eu-stage-web-01 ansible_host=10.1.3.10
eu-stage-db-01 ansible_host=10.1.4.10

[north_america_production]
na-prod-web-01 ansible_host=10.2.1.10
na-prod-web-02 ansible_host=10.2.1.11
na-prod-db-01 ansible_host=10.2.2.10

[north_america_staging]
na-stage-web-01 ansible_host=10.2.3.10

[asia_pacific_production]
ap-prod-web-01 ansible_host=10.3.1.10
ap-prod-db-01 ansible_host=10.3.2.10

# Application tiers
[web_servers]
eu-prod-web-01
eu-prod-web-02
eu-stage-web-01
na-prod-web-01
na-prod-web-02
na-stage-web-01
ap-prod-web-01

[database_servers]
eu-prod-db-01
eu-stage-db-01
na-prod-db-01
ap-prod-db-01

# Environment aggregation
[production:children]
europe_production
north_america_production
asia_pacific_production

[staging:children]
europe_staging
north_america_staging

# Regional aggregation
[europe:children]
europe_production
europe_staging

[north_america:children]
north_america_production
north_america_staging

[asia_pacific:children]
asia_pacific_production

# Global groups
[all_web:children]
web_servers

[all_databases:children]
database_servers
EOF
```

Create hierarchical group variables:

```bash
# Global variables
cat > group_vars/all.yml << 'EOF'
---
# Global configuration for all hosts
company_name: "Enterprise Corp"
ansible_user: "ansible"
ansible_ssh_private_key_file: "~/.ssh/enterprise_key"

# Global security settings
security_baseline:
  ssh_port: 22
  disable_root_login: true
  password_authentication: false
  fail2ban_enabled: true

# Global monitoring
monitoring:
  enabled: true
  agent: "datadog"
  log_level: "INFO"

# Global backup settings
backup:
  enabled: false  # Override in specific groups
  retention_days: 7
  schedule: "0 2 * * *"

# Network settings
dns_servers:
  - "8.8.8.8"
  - "8.8.4.4"

ntp_servers:
  - "pool.ntp.org"
EOF

# Regional variables
cat > group_vars/europe.yml << 'EOF'
---
# Europe regional configuration
region: "europe"
timezone: "Europe/London"
locale: "en_GB.UTF-8"

# Regional compliance
gdpr_compliance: true
data_residency: "EU"

# Regional DNS
dns_servers:
  - "1.1.1.1"
  - "1.0.0.1"

# Regional NTP
ntp_servers:
  - "europe.pool.ntp.org"
  - "uk.pool.ntp.org"

# Regional backup
backup:
  enabled: true
  retention_days: 30
  location: "eu-backup-vault"
  encryption: true

# Regional monitoring
monitoring:
  datacenter: "eu-west-1"
  compliance_logging: true
EOF

cat > group_vars/north_america.yml << 'EOF'
---
# North America regional configuration
region: "north_america"
timezone: "America/New_York"
locale: "en_US.UTF-8"

# Regional compliance
sox_compliance: true
data_residency: "US"

# Regional DNS
dns_servers:
  - "8.8.8.8"
  - "8.8.4.4"

# Regional NTP
ntp_servers:
  - "north-america.pool.ntp.org"
  - "us.pool.ntp.org"

# Regional backup
backup:
  enabled: true
  retention_days: 90  # Longer retention for SOX
  location: "us-backup-vault"
  encryption: true

# Regional monitoring
monitoring:
  datacenter: "us-east-1"
  compliance_logging: true
EOF

cat > group_vars/asia_pacific.yml << 'EOF'
---
# Asia Pacific regional configuration
region: "asia_pacific"
timezone: "Asia/Tokyo"
locale: "en_US.UTF-8"

# Regional compliance
data_residency: "APAC"

# Regional DNS
dns_servers:
  - "1.1.1.1"
  - "1.0.0.1"

# Regional NTP
ntp_servers:
  - "asia.pool.ntp.org"
  - "jp.pool.ntp.org"

# Regional backup
backup:
  enabled: true
  retention_days: 14
  location: "ap-backup-vault"
  encryption: true

# Regional monitoring
monitoring:
  datacenter: "ap-northeast-1"
  compliance_logging: false
EOF

# Environment variables
cat > group_vars/production.yml << 'EOF'
---
# Production environment configuration
environment: "production"
environment_tier: "prod"

# Production security
security_baseline:
  ssh_port: 2222  # Override global setting
  disable_root_login: true
  password_authentication: false
  fail2ban_enabled: true
  selinux_enabled: true
  firewall_enabled: true

# Production monitoring
monitoring:
  enabled: true
  log_level: "WARN"
  alerting: true
  retention_days: 90
  detailed_metrics: true

# Production backup
backup:
  enabled: true
  frequency: "hourly"
  retention_days: 365
  offsite_replication: true
  encryption: true
  compression: true

# Production performance
performance:
  cpu_governor: "performance"
  swappiness: 10
  transparent_hugepages: "never"

# Production maintenance
maintenance:
  window: "Sunday 02:00-04:00"
  auto_patching: false
  reboot_required_notification: true
EOF

cat > group_vars/staging.yml << 'EOF'
---
# Staging environment configuration
environment: "staging"
environment_tier: "stage"

# Staging security (relaxed)
security_baseline:
  ssh_port: 22
  disable_root_login: true
  password_authentication: true  # Allow for testing
  fail2ban_enabled: false
  selinux_enabled: false
  firewall_enabled: false

# Staging monitoring
monitoring:
  enabled: true
  log_level: "DEBUG"
  alerting: false
  retention_days: 7
  detailed_metrics: false

# Staging backup
backup:
  enabled: false  # Override regional settings
  frequency: "daily"
  retention_days: 7

# Staging performance
performance:
  cpu_governor: "ondemand"
  swappiness: 60

# Staging maintenance
maintenance:
  window: "Any time"
  auto_patching: true
  reboot_required_notification: false
EOF

# Application tier variables
cat > group_vars/web_servers.yml << 'EOF'
---
# Web servers configuration
server_role: "web"
application_type: "frontend"

# Web server software
web_server:
  software: "nginx"
  version: "1.20"
  worker_processes: "auto"
  worker_connections: 1024
  keepalive_timeout: 65

# Application settings
application:
  name: "enterprise-web"
  port: 8080
  ssl_enabled: true
  session_timeout: 3600

# Web server monitoring
monitoring:
  web_metrics: true
  response_time_monitoring: true
  error_rate_alerts: true

# Web server backup
backup:
  config_backup: true
  log_backup: false  # Logs are ephemeral
EOF

cat > group_vars/database_servers.yml << 'EOF'
---
# Database servers configuration
server_role: "database"
application_type: "backend"

# Database software
database:
  engine: "postgresql"
  version: "13"
  max_connections: 200
  shared_buffers: "256MB"
  effective_cache_size: "1GB"
  checkpoint_completion_target: 0.9

# Database monitoring
monitoring:
  db_metrics: true
  slow_query_monitoring: true
  connection_monitoring: true
  replication_monitoring: true

# Database backup
backup:
  enabled: true  # Always backup databases
  frequency: "every_4_hours"
  full_backup_schedule: "0 1 * * 0"  # Weekly full backup
  point_in_time_recovery: true
  backup_compression: true
  backup_encryption: true
EOF
```

Create host-specific variables:

```bash
# High-performance production web server
cat > host_vars/eu-prod-web-01.yml << 'EOF'
---
# EU Production Web Server 01 - High Performance
server_spec: "high_performance"

# Override web server settings for high traffic
web_server:
  worker_processes: 8
  worker_connections: 2048
  keepalive_timeout: 30

# Specific SSL configuration
ssl_config:
  certificate: "/etc/ssl/certs/eu-prod-web-01.crt"
  private_key: "/etc/ssl/private/eu-prod-web-01.key"
  protocols: ["TLSv1.2", "TLSv1.3"]

# Load balancer configuration
load_balancer:
  primary: true
  weight: 10
  health_check_url: "/health"

# Monitoring overrides
monitoring:
  detailed_metrics: true
  custom_dashboards: true
  alert_threshold_cpu: 80
  alert_threshold_memory: 85
EOF

# Database master server
cat > host_vars/eu-prod-db-01.yml << 'EOF'
---
# EU Production Database 01 - Master
server_spec: "database_master"

# Database master configuration
database:
  role: "master"
  max_connections: 500
  shared_buffers: "1GB"
  effective_cache_size: "4GB"
  wal_level: "replica"
  max_wal_senders: 3
  archive_mode: "on"
  archive_command: "cp %p /backup/wal/%f"

# Replication settings
replication:
  enabled: true
  synchronous_standby_names: "eu-prod-db-02"
  hot_standby: true

# Enhanced monitoring for master
monitoring:
  replication_lag_alerts: true
  wal_monitoring: true
  lock_monitoring: true
  vacuum_monitoring: true

# Critical backup settings
backup:
  frequency: "every_2_hours"
  full_backup_schedule: "0 0 * * *"  # Daily full backup
  wal_archiving: true
  backup_verification: true
EOF
```

Test variable inheritance:

```yaml
# Create test-variable-inheritance.yml
cat > test-variable-inheritance.yml << 'EOF'
---
- name: Test advanced variable inheritance
  hosts: all
  gather_facts: no
  tasks:
    - name: Display comprehensive variable inheritance
      debug:
        msg: |
          === Host: {{ inventory_hostname }} ===
          
          Geographic Information:
          - Region: {{ region | default('unknown') }}
          - Timezone: {{ timezone | default('unknown') }}
          - Data Residency: {{ data_residency | default('unknown') }}
          
          Environment Configuration:
          - Environment: {{ environment | default('unknown') }}
          - Environment Tier: {{ environment_tier | default('unknown') }}
          
          Server Role:
          - Server Role: {{ server_role | default('unknown') }}
          - Application Type: {{ application_type | default('unknown') }}
          - Server Spec: {{ server_spec | default('standard') }}
          
          Security Settings:
          - SSH Port: {{ security_baseline.ssh_port }}
          - Root Login Disabled: {{ security_baseline.disable_root_login }}
          - Fail2ban Enabled: {{ security_baseline.fail2ban_enabled }}
          
          Monitoring Configuration:
          - Enabled: {{ monitoring.enabled }}
          - Log Level: {{ monitoring.log_level }}
          - Alerting: {{ monitoring.alerting | default(false) }}
          - Datacenter: {{ monitoring.datacenter | default('unknown') }}
          
          Backup Configuration:
          - Enabled: {{ backup.enabled }}
          - Frequency: {{ backup.frequency | default('daily') }}
          - Retention Days: {{ backup.retention_days }}
          - Encryption: {{ backup.encryption | default(false) }}
          
          {% if web_server is defined %}
          Web Server Settings:
          - Software: {{ web_server.software }}
          - Worker Processes: {{ web_server.worker_processes }}
          - Worker Connections: {{ web_server.worker_connections }}
          {% endif %}
          
          {% if database is defined %}
          Database Settings:
          - Engine: {{ database.engine }}
          - Max Connections: {{ database.max_connections }}
          - Role: {{ database.role | default('standalone') }}
          {% endif %}
    
    - name: Variable inheritance analysis
      debug:
        msg: |
          Variable Inheritance Analysis:
          
          Regional Overrides:
          {% for host in groups['europe'] | default([]) %}
          - {{ host }}: Europe region (GDPR: {{ hostvars[host].gdpr_compliance | default(false) }})
          {% endfor %}
          {% for host in groups['north_america'] | default([]) %}
          - {{ host }}: North America region (SOX: {{ hostvars[host].sox_compliance | default(false) }})
          {% endfor %}
          
          Environment Overrides:
          {% for host in groups['production'] | default([]) %}
          - {{ host }}: Production (SSH Port: {{ hostvars[host].security_baseline.ssh_port }})
          {% endfor %}
          {% for host in groups['staging'] | default([]) %}
          - {{ host }}: Staging (SSH Port: {{ hostvars[host].security_baseline.ssh_port }})
          {% endfor %}
          
          Application Overrides:
          {% for host in groups['web_servers'] | default([]) %}
          - {{ host }}: Web Server (Workers: {{ hostvars[host].web_server.worker_processes }})
          {% endfor %}
          {% for host in groups['database_servers'] | default([]) %}
          - {{ host }}: Database (Max Conn: {{ hostvars[host].database.max_connections }})
          {% endfor %}
      run_once: true
EOF
```

Run variable inheritance test:

```bash
# Test variable inheritance
ansible-playbook -i inventories/enterprise.ini test-variable-inheritance.yml

# Test specific host variables
ansible-inventory -i inventories/enterprise.ini --host eu-prod-web-01 --yaml
ansible-inventory -i inventories/enterprise.ini --host eu-prod-db-01 --yaml
```

## Exercise 2: Dynamic Group Creation (7 minutes)

### Task: Create dynamic groups based on host properties and variables

Create dynamic group configuration:

```yaml
# Create configs/dynamic-groups.yml
cat > configs/dynamic-groups.yml << 'EOF'
# Dynamic Group Creation Configuration
plugin: constructed
strict: False

# Create dynamic groups based on host properties
groups:
    # Performance-based groups
    high_performance_servers: server_spec | default('standard') == 'high_performance'
    database_masters: (database.role | default('')) == 'master'
    database_slaves: (database.role | default('')) == 'slave'
    
    # Compliance-based groups
    gdpr_compliant: gdpr_compliance | default(false)
    sox_compliant: sox_compliance | default(false)
    
    # Security level groups
    high_security: >-
        security_baseline.ssh_port != 22 and
        security_baseline.selinux_enabled | default(false) and
        security_baseline.firewall_enabled | default(false)
    
    standard_security: >-
        security_baseline.ssh_port == 22 and
        not (security_baseline.selinux_enabled | default(false))
    
    # Monitoring level groups
    detailed_monitoring: >-
        monitoring.detailed_metrics | default(false) and
        monitoring.alerting | default(false)
    
    basic_monitoring: >-
        monitoring.enabled | default(false) and
        not (monitoring.detailed_metrics | default(false))
    
    # Backup strategy groups
    critical_backup: >-
        backup.enabled | default(false) and
        backup.frequency in ['hourly', 'every_2_hours', 'every_4_hours']
    
    standard_backup: >-
        backup.enabled | default(false) and
        backup.frequency == 'daily'
    
    no_backup: not (backup.enabled | default(false))
    
    # Maintenance window groups
    weekend_maintenance: >-
        'Sunday' in (maintenance.window | default(''))
    
    anytime_maintenance: >-
        maintenance.window | default('') == 'Any time'
    
    # Resource tier groups
    high_resource: >-
        (web_server.worker_processes | default(0) | int > 4) or
        (database.max_connections | default(0) | int > 300)
    
    medium_resource: >-
        (web_server.worker_processes | default(0) | int > 2 and web_server.worker_processes | default(0) | int <= 4) or
        (database.max_connections | default(0) | int > 100 and database.max_connections | default(0) | int <= 300)
    
    low_resource: >-
        (web_server.worker_processes | default(0) | int <= 2 and web_server.worker_processes | default(0) | int > 0) or
        (database.max_connections | default(0) | int <= 100 and database.max_connections | default(0) | int > 0)

# Create keyed groups for complex categorization
keyed_groups:
    # Environment and region combination
    - key: region + '_' + environment
      separator: ""
    
    # Server role and environment
    - key: server_role + '_' + environment
      prefix: role
      separator: "_"
    
    # Compliance requirements
    - key: >-
        'gdpr' if gdpr_compliance | default(false) else
        'sox' if sox_compliance | default(false) else
        'standard'
      prefix: compliance
      separator: "_"
    
    # Resource and environment combination
    - key: >-
        ('high' if (
            (web_server.worker_processes | default(0) | int > 4) or
            (database.max_connections | default(0) | int > 300)
        ) else 'medium' if (
            (web_server.worker_processes | default(0) | int > 2) or
            (database.max_connections | default(0) | int > 100)
        ) else 'low') + '_' + environment
      prefix: resource
      separator: "_"

# Compose additional variables for grouping
compose:
    # Calculate total backup frequency score
    backup_frequency_score: >-
        10 if backup.frequency | default('') == 'hourly' else
        8 if backup.frequency | default('') == 'every_2_hours' else
        6 if backup.frequency | default('') == 'every_4_hours' else
        4 if backup.frequency | default('') == 'daily' else
        0
    
    # Calculate security score
    security_score: >-
        (5 if security_baseline.ssh_port != 22 else 0) +
        (3 if security_baseline.selinux_enabled | default(false) else 0) +
        (3 if security_baseline.firewall_enabled | default(false) else 0) +
        (2 if security_baseline.fail2ban_enabled | default(false) else 0) +
        (2 if not security_baseline.password_authentication | default(true) else 0)
    
    # Calculate monitoring score
    monitoring_score: >-
        (5 if monitoring.detailed_metrics | default(false) else 0) +
        (3 if monitoring.alerting | default(false) else 0) +
        (2 if monitoring.enabled | default(false) else 0) +
        (1 if monitoring.compliance_logging | default(false) else 0)
    
    # Overall criticality score
    criticality_score: >-
        backup_frequency_score + security_score + monitoring_score
EOF
```

Create dynamic group test:

```yaml
# Create test-dynamic-groups.yml
cat > test-dynamic-groups.yml << 'EOF'
---
- name: Test dynamic group creation
  hosts: all
  gather_facts: no
  tasks:
    - name: Display dynamic group membership
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Dynamic Groups: {{ group_names | select('match', '^(high_|standard_|basic_|critical_|compliance_|resource_|role_).*') | list | join(', ') }}
          
          Calculated Scores:
          - Backup Frequency Score: {{ backup_frequency_score | default(0) }}
          - Security Score: {{ security_score | default(0) }}
          - Monitoring Score: {{ monitoring_score | default(0) }}
          - Overall Criticality: {{ criticality_score | default(0) }}
    
    - name: Dynamic group analysis
      debug:
        msg: |
          Dynamic Group Analysis:
          
          Performance Groups:
          - High Performance Servers: {{ groups['high_performance_servers'] | default([]) | length }}
          - Database Masters: {{ groups['database_masters'] | default([]) | length }}
          
          Security Groups:
          - High Security: {{ groups['high_security'] | default([]) | length }}
          - Standard Security: {{ groups['standard_security'] | default([]) | length }}
          
          Compliance Groups:
          - GDPR Compliant: {{ groups['gdpr_compliant'] | default([]) | length }}
          - SOX Compliant: {{ groups['sox_compliant'] | default([]) | length }}
          
          Backup Strategy Groups:
          - Critical Backup: {{ groups['critical_backup'] | default([]) | length }}
          - Standard Backup: {{ groups['standard_backup'] | default([]) | length }}
          - No Backup: {{ groups['no_backup'] | default([]) | length }}
          
          Resource Tier Groups:
          - High Resource: {{ groups['high_resource'] | default([]) | length }}
          - Medium Resource: {{ groups['medium_resource'] | default([]) | length }}
          - Low Resource: {{ groups['low_resource'] | default([]) | length }}
          
          Monitoring Groups:
          - Detailed Monitoring: {{ groups['detailed_monitoring'] | default([]) | length }}
          - Basic Monitoring: {{ groups['basic_monitoring'] | default([]) | length }}
      run_once: true
    
    - name: Criticality analysis
      debug:
        msg: |
          Host Criticality Analysis:
          {% set critical_hosts = [] %}
          {% set important_hosts = [] %}
          {% set standard_hosts = [] %}
          {% for host in groups['all'] %}
          {% set score = hostvars[host].criticality_score | default(0) | int %}
          {% if score >= 15 %}
          {% set _ = critical_hosts.append(host) %}
          {% elif score >= 10 %}
          {% set _ = important_hosts.append(host) %}
          {% else %}
          {% set _ = standard_hosts.append(host) %}
          {% endif %}
          {% endfor %}
          
          Critical Hosts (Score >= 15): {{ critical_hosts | join(', ') if critical_hosts else 'None' }}
          Important Hosts (Score 10-14): {{ important_hosts | join(', ') if important_hosts else 'None' }}
          Standard Hosts (Score < 10): {{ standard_hosts | join(', ') if standard_hosts else 'None' }}
      run_once: true
EOF
```

Test dynamic groups:

```bash
# Test dynamic group creation
ansible-inventory -i inventories/enterprise.ini -i configs/dynamic-groups.yml --graph
ansible-inventory -i inventories/enterprise.ini -i configs/dynamic-groups.yml --list --yaml

# Run dynamic group test
ansible-playbook -i inventories/enterprise.ini -i configs/dynamic-groups.yml test-dynamic-groups.yml
```

## Exercise 3: Variable Management Best Practices (5 minutes)

### Task: Implement variable management best practices and validation

Create variable validation script:

```python
# Create scripts/validate-variables.py
cat > scripts/validate-variables.py << 'EOF'
#!/usr/bin/env python3
"""
Variable Validation Script
Validates variable consistency and inheritance
"""

import json
import subprocess
import sys
from collections import defaultdict

def get_host_vars(inventory_path, host):
    """Get variables for a specific host"""
    try:
        result = subprocess.run([
            'ansible-inventory',
            '-i', inventory_path,
            '--host', host
        ], capture_output=True, text=True, check=True)
        return json.loads(result.stdout)
    except (subprocess.CalledProcessError, json.JSONDecodeError):
        return {}

def get_all_hosts(inventory_path):
    """Get all hosts from inventory"""
    try:
        result = subprocess.run([
            'ansible-inventory',
            '-i', inventory_path,
            '--list'
        ], capture_output=True, text=True, check=True)
        inventory = json.loads(result.stdout)
        return list(inventory.get('_meta', {}).get('hostvars', {}).keys())
    except (subprocess.CalledProcessError, json.JSONDecodeError):
        return []

def validate_variables(inventory_path):
    """Validate variable consistency across hosts"""
    hosts = get_all_hosts(inventory_path)
    issues = []
    warnings = []
    
    # Required variables for all hosts
    required_vars = [
        'ansible_host', 'region', 'environment', 'monitoring.enabled',
        'backup.enabled', 'security_baseline.ssh_port'
    ]
    
    # Collect all host variables
    all_host_vars = {}
    for host in hosts:
        all_host_vars[host] = get_host_vars(inventory_path, host)
    
    # Check required variables
    for host in hosts:
        host_vars = all_host_vars[host]
        for req_var in required_vars:
            if '.' in req_var:
                # Nested variable
                keys = req_var.split('.')
                value = host_vars
                for key in keys:
                    if isinstance(value, dict) and key in value:
                        value = value[key]
                    else:
                        value = None
                        break
                if value is None:
                    issues.append(f"Host {host} missing required variable: {req_var}")
            else:
                # Simple variable
                if req_var not in host_vars:
                    issues.append(f"Host {host} missing required variable: {req_var}")
    
    # Check variable consistency within environments
    env_vars = defaultdict(lambda: defaultdict(set))
    for host in hosts:
        host_vars = all_host_vars[host]
        env = host_vars.get('environment', 'unknown')
        
        # Check SSH port consistency within environment
        ssh_port = host_vars.get('security_baseline', {}).get('ssh_port')
        if ssh_port:
            env_vars[env]['ssh_port'].add(ssh_port)
        
        # Check monitoring level consistency
        monitoring_level = host_vars.get('monitoring', {}).get('log_level')
        if monitoring_level:
            env_vars[env]['monitoring_level'].add(monitoring_level)
    
    # Report inconsistencies
    for env, vars_dict in env_vars.items():
        for var_name, values in vars_dict.items():
            if len(values) > 1:
                warnings.append(f"Environment {env} has inconsistent {var_name}: {list(values)}")
    
    # Check for potential security issues
    for host in hosts:
        host_vars = all_host_vars[host]
        env = host_vars.get('environment', 'unknown')
        
        # Production hosts should have enhanced security
        if env == 'production':
            ssh_port = host_vars.get('security_baseline', {}).get('ssh_port', 22)
            if ssh_port == 22:
                warnings.append(f"Production host {host} using default SSH port")
            
            password_auth = host_vars.get('security_baseline', {}).get('password_authentication', True)
            if password_auth:
                issues.append(f"Production host {host} allows password authentication")
    
    return issues, warnings, len(hosts)

def main():
    if len(sys.argv) < 2:
        print("Usage: validate-variables.py <inventory_path>")
        sys.exit(1)
    
    inventory_path = sys.argv[1]
    print(f"Validating variables for inventory: {inventory_path}")
    
    issues, warnings, host_count = validate_variables(inventory_path)
    
    print(f"\nValidation Results:")
    print(f"- Hosts validated: {host_count}")
    print(f"- Issues found: {len(issues)}")
    print(f"- Warnings: {len(warnings)}")
    
    if issues:
        print("\nISSUES (must be resolved):")
        for issue in issues:
            print(f"  ❌ {issue}")
    
    if warnings:
        print("\nWARNINGS (should be reviewed):")
        for warning in warnings:
            print(f"  ⚠️  {warning}")
    
    if not issues and not warnings:
        print("\n✅ Variable validation passed!")
    
    return len(issues)

if __name__ == '__main__':
    sys.exit(main())
EOF

chmod +x scripts/validate-variables.py
```

Create variable documentation generator:

```python
# Create scripts/document-variables.py
cat > scripts/document-variables.py << 'EOF'
#!/usr/bin/env python3
"""
Variable Documentation Generator
Creates documentation for variable inheritance
"""

import json
import subprocess
import sys
from collections import defaultdict

def get_inventory_structure(inventory_path):
    """Get complete inventory structure"""
    try:
        result = subprocess.run([
            'ansible-inventory',
            '-i', inventory_path,
            '--list'
        ], capture_output=True, text=True, check=True)
        return json.loads(result.stdout)
    except (subprocess.CalledProcessError, json.JSONDecodeError):
        return {}

def generate_documentation(inventory_path):
    """Generate variable inheritance documentation"""
    inventory = get_inventory_structure(inventory_path)
    
    doc = []
    doc.append("# Variable Inheritance Documentation")
    doc.append(f"Generated from: {inventory_path}")
    doc.append("")
    
    # Group structure
    doc.append("## Group Structure")
    doc.append("")
    groups = {k: v for k, v in inventory.items() if k != '_meta'}
    
    for group_name, group_data in sorted(groups.items()):
        if 'children' in group_data:
            doc.append(f"### {group_name} (Parent Group)")
            doc.append(f"Children: {', '.join(group_data['children'])}")
        elif 'hosts' in group_data:
            doc.append(f"### {group_name} (Host Group)")
            doc.append(f"Hosts: {', '.join(group_data['hosts'])}")
        
        if 'vars' in group_data:
            doc.append("Variables:")
            for var_name, var_value in sorted(group_data['vars'].items()):
                doc.append(f"- `{var_name}`: {var_value}")
        doc.append("")
    
    # Host variables summary
    doc.append("## Host Variables Summary")
    doc.append("")
    
    hostvars = inventory.get('_meta', {}).get('hostvars', {})
    for host_name in sorted(hostvars.keys()):
        host_vars = hostvars[host_name]
        doc.append(f"### {host_name}")
        
        # Key variables
        key_vars = [
            'region', 'environment', 'server_role', 'ansible_host',
            'monitoring', 'backup', 'security_baseline'
        ]
        
        for var_name in key_vars:
            if var_name in host_vars:
                value = host_vars[var_name]
                if isinstance(value, dict):
                    doc.append(f"- `{var_name}`: (complex object)")
                    for sub_key, sub_value in value.items():
                        doc.append(f"  - `{sub_key}`: {sub_value}")
                else:
                    doc.append(f"- `{var_name}`: {value}")
        doc.append("")
    
    return "\n".join(doc)

def main():
    if len(sys.argv) < 2:
        print("Usage: document-variables.py <inventory_path>")
        sys.exit(1)
    
    inventory_path = sys.argv[1]
    documentation = generate_documentation(inventory_path)
    
    output_file = "variable-inheritance-documentation.md"
    with open(output_file, 'w') as f:
        f.write(documentation)
    
    print(f"Documentation generated: {output_file}")

if __name__ == '__main__':
    main()
EOF

chmod +x scripts/document-variables.py
```

Test variable management:

```bash
# Validate variables
./scripts/validate-variables.py "inventories/enterprise.ini"

# Generate documentation
./scripts/document-variables.py "inventories/enterprise.ini"
cat variable-inheritance-documentation.md

# Test with dynamic groups
./scripts/validate-variables.py "inventories/enterprise.ini -i configs/dynamic-groups.yml"
```

## Verification and Discussion

### 1. Check Results
```bash
# Test variable inheritance
echo "=== Variable Inheritance Test ==="
ansible-playbook -i inventories/enterprise.ini test-variable-inheritance.yml

# Test dynamic groups
echo "=== Dynamic Groups Test ==="
ansible-playbook -i inventories/enterprise.ini -i configs/dynamic-groups.yml test-dynamic-groups.yml

# Validate variables
echo "=== Variable Validation ==="
./scripts/validate-variables.py "inventories/enterprise.ini"

# Check documentation
echo "=== Generated Documentation ==="
head -50 variable-inheritance-documentation.md
```

### 2. Discussion Points
- How do you manage variable inheritance complexity in large environments?
- What strategies work best for maintaining variable consistency?
- How do you implement variable validation in your CI/CD pipelines?
- What are the performance implications of complex dynamic groups?

### 3. Clean Up
```bash
# Remove demo files
rm -rf group_vars host_vars inventories configs scripts
rm -f *.yml *.py *.md
```

## Key Takeaways
- Hierarchical variable inheritance enables flexible configuration management
- Dynamic groups provide powerful categorization based on host properties
- Variable validation prevents configuration drift and inconsistencies
- Documentation generation helps maintain complex inventory structures
- Performance considerations are important with complex variable calculations
- Best practices include validation, documentation, and consistent naming

## Module 4 Summary

### What We Covered
1. **Static vs Dynamic Inventory Fundamentals** - Understanding inventory approaches and migration strategies
2. **Custom Inventory Scripts Development** - Creating enterprise-specific inventory solutions
3. **Inventory Plugins and Cloud Integration** - Modern plugin architecture and cloud provider integration
4. **Multiple Inventory Sources and Merging** - Combining sources and handling conflicts
5. **Advanced Variables and Group Management** - Complex inheritance and dynamic grouping

### Key Skills Developed
- Transitioning from static to dynamic inventory approaches
- Developing custom inventory scripts and plugins
- Integrating multiple inventory sources effectively
- Implementing advanced variable inheritance patterns
- Creating dynamic groups based on host properties
- Validating and documenting complex inventory structures

### Next Steps
You now have comprehensive knowledge of dynamic inventory management for enterprise environments. These skills will be essential for managing large-scale, multi-cloud infrastructure in the subsequent modules.

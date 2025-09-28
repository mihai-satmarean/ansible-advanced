# Lab 5.2: Role Dependencies and Relationships

## Objective
Implement complex role dependencies and inter-role communication patterns for enterprise automation scenarios.

## Duration
30 minutes

## Prerequisites
- Completed Lab 5.1
- Understanding of role structure and organization

## Lab Setup

```bash
cd ~/ansible-labs/module-05
mkdir -p lab-5.2
cd lab-5.2

# Create inventory for testing
cat > inventory.ini << 'EOF'
[web_servers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Basic Role Dependencies (10 minutes)

### Task: Create roles with simple dependencies

Create a security baseline role:

```bash
# Create security baseline role
ansible-galaxy init roles/security_baseline

# Update security baseline metadata
cat > roles/security_baseline/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: security_baseline
  namespace: company
  author: Security Team
  description: Enterprise security baseline configuration
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
    - name: EL
      versions: [8, 9]
  galaxy_tags:
    - security
    - baseline
    - hardening

dependencies: []
EOF
```

Create security baseline tasks:

```yaml
# Create roles/security_baseline/tasks/main.yml
cat > roles/security_baseline/tasks/main.yml << 'EOF'
---
# Security Baseline Role - Main Tasks

- name: Display security baseline information
  debug:
    msg: |
      Applying security baseline configuration
      SSH Port: {{ security_ssh_port }}
      Firewall Enabled: {{ security_firewall_enabled }}
      Fail2ban Enabled: {{ security_fail2ban_enabled }}

- name: Update system packages
  package:
    name: "*"
    state: latest
  when: security_update_packages | default(true)
  tags: security_updates

- name: Install security packages
  package:
    name: "{{ security_packages }}"
    state: present
  tags: security_packages

- name: Configure SSH security
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
    backup: yes
  loop:
    - regexp: '^#?Port'
      line: "Port {{ security_ssh_port }}"
    - regexp: '^#?PermitRootLogin'
      line: "PermitRootLogin {{ 'no' if security_disable_root_login else 'yes' }}"
    - regexp: '^#?PasswordAuthentication'
      line: "PasswordAuthentication {{ 'no' if security_disable_password_auth else 'yes' }}"
    - regexp: '^#?PubkeyAuthentication'
      line: "PubkeyAuthentication yes"
  notify: restart sshd
  tags: ssh_security

- name: Configure firewall
  include_tasks: firewall.yml
  when: security_firewall_enabled | default(true)
  tags: firewall

- name: Configure fail2ban
  include_tasks: fail2ban.yml
  when: security_fail2ban_enabled | default(true)
  tags: fail2ban

- name: Set security baseline completion flag
  set_fact:
    security_baseline_applied: true
    security_baseline_timestamp: "{{ ansible_date_time.iso8601 }}"
EOF
```

Create security baseline defaults:

```yaml
# Create roles/security_baseline/defaults/main.yml
cat > roles/security_baseline/defaults/main.yml << 'EOF'
---
# Security Baseline Default Variables

# SSH Configuration
security_ssh_port: 2222
security_disable_root_login: true
security_disable_password_auth: true

# Firewall Configuration
security_firewall_enabled: true
security_firewall_default_policy: "deny"
security_allowed_ports:
  - "{{ security_ssh_port }}/tcp"
  - "80/tcp"
  - "443/tcp"

# Fail2ban Configuration
security_fail2ban_enabled: true
security_fail2ban_bantime: 3600
security_fail2ban_maxretry: 3

# Package Management
security_update_packages: true
security_packages:
  - ufw
  - fail2ban
  - unattended-upgrades
  - logwatch

# Audit Configuration
security_audit_enabled: true
security_audit_rules:
  - "-w /etc/passwd -p wa -k identity"
  - "-w /etc/group -p wa -k identity"
  - "-w /etc/shadow -p wa -k identity"
EOF
```

Create firewall tasks:

```yaml
# Create roles/security_baseline/tasks/firewall.yml
cat > roles/security_baseline/tasks/firewall.yml << 'EOF'
---
# Firewall configuration tasks

- name: Enable UFW firewall
  ufw:
    state: enabled
    policy: "{{ security_firewall_default_policy }}"
  when: ansible_os_family == "Debian"

- name: Configure UFW rules
  ufw:
    rule: allow
    port: "{{ item.split('/')[0] }}"
    proto: "{{ item.split('/')[1] }}"
  loop: "{{ security_allowed_ports }}"
  when: ansible_os_family == "Debian"

- name: Start and enable firewalld (RHEL/CentOS)
  service:
    name: firewalld
    state: started
    enabled: yes
  when: ansible_os_family == "RedHat"

- name: Configure firewalld rules (RHEL/CentOS)
  firewalld:
    port: "{{ item }}"
    permanent: yes
    state: enabled
    immediate: yes
  loop: "{{ security_allowed_ports }}"
  when: ansible_os_family == "RedHat"
EOF
```

Create fail2ban tasks:

```yaml
# Create roles/security_baseline/tasks/fail2ban.yml
cat > roles/security_baseline/tasks/fail2ban.yml << 'EOF'
---
# Fail2ban configuration tasks

- name: Configure fail2ban jail
  template:
    src: jail.local.j2
    dest: /etc/fail2ban/jail.local
    backup: yes
  notify: restart fail2ban

- name: Start and enable fail2ban
  service:
    name: fail2ban
    state: started
    enabled: yes
EOF
```

Create fail2ban template:

```jinja2
# Create roles/security_baseline/templates/jail.local.j2
mkdir -p roles/security_baseline/templates
cat > roles/security_baseline/templates/jail.local.j2 << 'EOF'
# {{ ansible_managed }}
# Fail2ban jail configuration

[DEFAULT]
bantime = {{ security_fail2ban_bantime }}
maxretry = {{ security_fail2ban_maxretry }}
findtime = 600

[sshd]
enabled = true
port = {{ security_ssh_port }}
filter = sshd
logpath = /var/log/auth.log
maxretry = {{ security_fail2ban_maxretry }}

[nginx-http-auth]
enabled = true
filter = nginx-http-auth
port = http,https
logpath = /var/log/nginx/error.log

[nginx-limit-req]
enabled = true
filter = nginx-limit-req
port = http,https
logpath = /var/log/nginx/error.log
EOF
```

Create security baseline handlers:

```yaml
# Create roles/security_baseline/handlers/main.yml
cat > roles/security_baseline/handlers/main.yml << 'EOF'
---
# Security Baseline Handlers

- name: restart sshd
  service:
    name: "{{ 'ssh' if ansible_os_family == 'Debian' else 'sshd' }}"
    state: restarted

- name: restart fail2ban
  service:
    name: fail2ban
    state: restarted
EOF
```

Create monitoring agent role:

```bash
# Create monitoring agent role
ansible-galaxy init roles/monitoring_agent

# Update monitoring agent metadata
cat > roles/monitoring_agent/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: monitoring_agent
  namespace: company
  author: Monitoring Team
  description: Enterprise monitoring agent deployment
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
    - name: EL
      versions: [8, 9]
  galaxy_tags:
    - monitoring
    - observability
    - metrics

dependencies: []
EOF
```

Create monitoring agent tasks:

```yaml
# Create roles/monitoring_agent/tasks/main.yml
cat > roles/monitoring_agent/tasks/main.yml << 'EOF'
---
# Monitoring Agent Role - Main Tasks

- name: Display monitoring agent information
  debug:
    msg: |
      Installing monitoring agent: {{ monitoring_agent_type }}
      API Key configured: {{ monitoring_api_key is defined }}
      Tags: {{ monitoring_tags | join(', ') if monitoring_tags is defined else 'None' }}

- name: Include agent-specific tasks
  include_tasks: "{{ monitoring_agent_type }}.yml"
  when: monitoring_agent_type in ['datadog', 'newrelic', 'prometheus']

- name: Configure monitoring agent
  template:
    src: "{{ monitoring_agent_type }}.conf.j2"
    dest: "{{ monitoring_config_path }}/{{ monitoring_agent_type }}.conf"
    backup: yes
  notify: restart monitoring agent
  when: monitoring_agent_type in ['datadog', 'newrelic']

- name: Start and enable monitoring agent
  service:
    name: "{{ monitoring_service_name }}"
    state: started
    enabled: yes

- name: Set monitoring agent completion flag
  set_fact:
    monitoring_agent_installed: true
    monitoring_agent_version: "{{ monitoring_agent_version | default('latest') }}"
    monitoring_agent_timestamp: "{{ ansible_date_time.iso8601 }}"
EOF
```

Create monitoring agent defaults:

```yaml
# Create roles/monitoring_agent/defaults/main.yml
cat > roles/monitoring_agent/defaults/main.yml << 'EOF'
---
# Monitoring Agent Default Variables

# Agent Configuration
monitoring_agent_type: "datadog"
monitoring_agent_version: "latest"
monitoring_api_key: ""

# Service Configuration
monitoring_service_name: "{{ monitoring_agent_type }}-agent"
monitoring_config_path: "/etc/{{ monitoring_agent_type }}-agent"

# Monitoring Tags
monitoring_tags:
  - "environment:{{ environment | default('production') }}"
  - "role:{{ server_role | default('unknown') }}"
  - "region:{{ region | default('unknown') }}"

# Metrics Collection
monitoring_collect_system_metrics: true
monitoring_collect_process_metrics: true
monitoring_collect_network_metrics: true

# Log Collection
monitoring_collect_logs: true
monitoring_log_paths:
  - "/var/log/syslog"
  - "/var/log/auth.log"
  - "/var/log/nginx/*.log"
EOF
```

Update the web server role to use dependencies:

```yaml
# Update roles/enterprise_webserver/meta/main.yml (from Lab 5.1)
cat > roles/enterprise_webserver/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: enterprise_webserver
  namespace: company
  author: Enterprise DevOps Team
  description: Enterprise-grade web server role with security and monitoring
  company: Enterprise Corp
  license: MIT
  min_ansible_version: 2.9
  
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
    - name: EL
      versions: [8, 9]
  
  galaxy_tags:
    - web
    - nginx
    - security
    - monitoring
    - enterprise

# Role Dependencies
dependencies:
  - role: company.security_baseline
    when: security_hardening_enabled | default(true)
    vars:
      security_allowed_ports:
        - "{{ webserver_port }}/tcp"
        - "{{ webserver_ssl_port }}/tcp"
        - "{{ security_ssh_port | default(2222) }}/tcp"
  
  - role: company.monitoring_agent
    when: monitoring_enabled | default(true)
    vars:
      monitoring_tags:
        - "service:webserver"
        - "domain:{{ webserver_domain }}"
        - "environment:{{ webserver_environment | default('production') }}"

collections:
  - community.general
  - ansible.posix
EOF
```

Test basic dependencies:

```yaml
# Create test-dependencies.yml
cat > test-dependencies.yml << 'EOF'
---
- name: Test role dependencies
  hosts: web_servers
  become: yes
  vars:
    webserver_domain: "test.local"
    webserver_admin_email: "admin@test.local"
    security_hardening_enabled: true
    monitoring_enabled: true
    monitoring_api_key: "test-api-key"
  roles:
    - role: enterprise_webserver
EOF

# Test dependency resolution
ansible-playbook --list-tasks test-dependencies.yml -i inventory.ini
```

## Exercise 2: Conditional Dependencies and Complex Relationships (12 minutes)

### Task: Implement conditional dependencies and inter-role communication

Create a database role with conditional dependencies:

```bash
# Create database role
ansible-galaxy init roles/enterprise_database

# Update database metadata with conditional dependencies
cat > roles/enterprise_database/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: enterprise_database
  namespace: company
  author: Database Team
  description: Enterprise database role with backup and monitoring
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
    - name: EL
      versions: [8, 9]
  galaxy_tags:
    - database
    - postgresql
    - mysql
    - backup

# Conditional Dependencies
dependencies:
  # Always apply security baseline
  - role: company.security_baseline
    vars:
      security_allowed_ports:
        - "{{ database_port }}/tcp"
        - "{{ security_ssh_port | default(2222) }}/tcp"
  
  # Install monitoring if enabled
  - role: company.monitoring_agent
    when: monitoring_enabled | default(true)
    vars:
      monitoring_tags:
        - "service:database"
        - "engine:{{ database_engine }}"
        - "environment:{{ database_environment | default('production') }}"
  
  # Install backup agent if backup is enabled
  - role: company.backup_agent
    when: backup_enabled | default(true)
    vars:
      backup_type: "database"
      backup_schedule: "{{ database_backup_schedule | default('0 2 * * *') }}"
EOF
```

Create backup agent role:

```bash
# Create backup agent role
ansible-galaxy init roles/backup_agent

cat > roles/backup_agent/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: backup_agent
  namespace: company
  author: Backup Team
  description: Enterprise backup agent for various services
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
    - name: EL
      versions: [8, 9]
  galaxy_tags:
    - backup
    - recovery
    - enterprise

dependencies: []
EOF

cat > roles/backup_agent/defaults/main.yml << 'EOF'
---
# Backup Agent Default Variables

backup_type: "filesystem"  # filesystem, database, application
backup_schedule: "0 2 * * *"
backup_retention_days: 30
backup_destination: "/backup"
backup_compression: true
backup_encryption: false

# Database-specific backup settings
database_backup_full_schedule: "0 1 * * 0"  # Weekly full backup
database_backup_incremental_schedule: "0 2 * * 1-6"  # Daily incremental

# Application-specific backup settings
application_backup_config: true
application_backup_data: true
application_backup_logs: false
EOF

cat > roles/backup_agent/tasks/main.yml << 'EOF'
---
# Backup Agent Role - Main Tasks

- name: Display backup agent information
  debug:
    msg: |
      Configuring backup agent
      Type: {{ backup_type }}
      Schedule: {{ backup_schedule }}
      Retention: {{ backup_retention_days }} days
      Destination: {{ backup_destination }}

- name: Create backup directories
  file:
    path: "{{ item }}"
    state: directory
    mode: '0750'
  loop:
    - "{{ backup_destination }}"
    - "{{ backup_destination }}/{{ backup_type }}"
    - "/var/log/backup"

- name: Install backup tools
  package:
    name: "{{ backup_packages }}"
    state: present

- name: Configure backup scripts
  template:
    src: "backup-{{ backup_type }}.sh.j2"
    dest: "/usr/local/bin/backup-{{ backup_type }}.sh"
    mode: '0755'

- name: Configure backup cron job
  cron:
    name: "{{ backup_type }} backup"
    job: "/usr/local/bin/backup-{{ backup_type }}.sh"
    minute: "{{ backup_schedule.split()[1] }}"
    hour: "{{ backup_schedule.split()[2] }}"
    day: "{{ backup_schedule.split()[3] }}"
    month: "{{ backup_schedule.split()[4] }}"
    weekday: "{{ backup_schedule.split()[5] }}"

- name: Set backup agent completion flag
  set_fact:
    backup_agent_configured: true
    backup_agent_type: "{{ backup_type }}"
    backup_agent_timestamp: "{{ ansible_date_time.iso8601 }}"
EOF

# Create backup script template
mkdir -p roles/backup_agent/templates
cat > roles/backup_agent/templates/backup-database.sh.j2 << 'EOF'
#!/bin/bash
# {{ ansible_managed }}
# Database Backup Script

BACKUP_DIR="{{ backup_destination }}/{{ backup_type }}"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/backup/database-backup-$DATE.log"

echo "Starting database backup at $(date)" | tee -a $LOG_FILE

{% if database_engine == 'postgresql' %}
# PostgreSQL backup
pg_dump -h localhost -U postgres {{ database_name }} > $BACKUP_DIR/{{ database_name }}-$DATE.sql
{% elif database_engine == 'mysql' %}
# MySQL backup
mysqldump -u root -p{{ database_password }} {{ database_name }} > $BACKUP_DIR/{{ database_name }}-$DATE.sql
{% endif %}

{% if backup_compression %}
# Compress backup
gzip $BACKUP_DIR/{{ database_name }}-$DATE.sql
{% endif %}

# Clean up old backups
find $BACKUP_DIR -name "{{ database_name }}-*.sql*" -mtime +{{ backup_retention_days }} -delete

echo "Database backup completed at $(date)" | tee -a $LOG_FILE
EOF

# Set backup packages variable
cat > roles/backup_agent/vars/main.yml << 'EOF'
---
backup_packages:
  - rsync
  - gzip
  - tar
  - cron
EOF
```

Create load balancer role with complex dependencies:

```bash
# Create load balancer role
ansible-galaxy init roles/load_balancer

cat > roles/load_balancer/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: load_balancer
  namespace: company
  author: Infrastructure Team
  description: Enterprise load balancer with health checks
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
  galaxy_tags:
    - loadbalancer
    - haproxy
    - nginx

# Complex Dependencies
dependencies:
  # Security baseline (always)
  - role: company.security_baseline
    vars:
      security_allowed_ports:
        - "80/tcp"
        - "443/tcp"
        - "8080/tcp"  # Stats port
        - "{{ security_ssh_port | default(2222) }}/tcp"
  
  # Monitoring (conditional)
  - role: company.monitoring_agent
    when: monitoring_enabled | default(true)
    vars:
      monitoring_tags:
        - "service:loadbalancer"
        - "type:{{ lb_type | default('haproxy') }}"
        - "environment:{{ lb_environment | default('production') }}"
  
  # Backup for configuration (conditional)
  - role: company.backup_agent
    when: backup_enabled | default(true)
    vars:
      backup_type: "application"
      backup_schedule: "0 3 * * *"
EOF

cat > roles/load_balancer/defaults/main.yml << 'EOF'
---
# Load Balancer Default Variables

lb_type: "haproxy"  # haproxy or nginx
lb_stats_enabled: true
lb_stats_port: 8080
lb_stats_user: "admin"
lb_stats_password: "secure_password"

# Backend servers
lb_backend_servers: []
# Example:
# lb_backend_servers:
#   - name: "web1"
#     address: "192.168.1.10"
#     port: 80
#     check: true
#   - name: "web2"
#     address: "192.168.1.11"
#     port: 80
#     check: true

# Health check configuration
lb_health_check_enabled: true
lb_health_check_interval: "5s"
lb_health_check_timeout: "3s"
lb_health_check_retries: 3

# SSL Configuration
lb_ssl_enabled: false
lb_ssl_certificate: "/etc/ssl/certs/loadbalancer.crt"
lb_ssl_private_key: "/etc/ssl/private/loadbalancer.key"
EOF

cat > roles/load_balancer/tasks/main.yml << 'EOF'
---
# Load Balancer Role - Main Tasks

- name: Display load balancer information
  debug:
    msg: |
      Configuring load balancer
      Type: {{ lb_type }}
      Backend servers: {{ lb_backend_servers | length }}
      Stats enabled: {{ lb_stats_enabled }}
      SSL enabled: {{ lb_ssl_enabled }}

- name: Validate backend servers configuration
  assert:
    that:
      - lb_backend_servers is defined
      - lb_backend_servers | length > 0
    fail_msg: "At least one backend server must be configured"
    success_msg: "Backend servers configuration is valid"

- name: Install load balancer package
  package:
    name: "{{ lb_type }}"
    state: present

- name: Configure load balancer
  template:
    src: "{{ lb_type }}.cfg.j2"
    dest: "/etc/{{ lb_type }}/{{ lb_type }}.cfg"
    backup: yes
  notify: restart load balancer

- name: Start and enable load balancer
  service:
    name: "{{ lb_type }}"
    state: started
    enabled: yes

- name: Set load balancer completion flag
  set_fact:
    load_balancer_configured: true
    load_balancer_type: "{{ lb_type }}"
    load_balancer_backend_count: "{{ lb_backend_servers | length }}"
    load_balancer_timestamp: "{{ ansible_date_time.iso8601 }}"
EOF

# Create HAProxy configuration template
mkdir -p roles/load_balancer/templates
cat > roles/load_balancer/templates/haproxy.cfg.j2 << 'EOF'
# {{ ansible_managed }}
# HAProxy Configuration

global
    daemon
    chroot /var/lib/haproxy
    user haproxy
    group haproxy
    pidfile /var/run/haproxy.pid

defaults
    mode http
    timeout connect {{ lb_health_check_timeout }}
    timeout client 30s
    timeout server 30s
    option httplog

{% if lb_stats_enabled %}
# Statistics
listen stats
    bind *:{{ lb_stats_port }}
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE
    stats auth {{ lb_stats_user }}:{{ lb_stats_password }}
{% endif %}

# Frontend
frontend web_frontend
    bind *:80
    {% if lb_ssl_enabled %}
    bind *:443 ssl crt {{ lb_ssl_certificate }}
    {% endif %}
    default_backend web_servers

# Backend
backend web_servers
    balance roundrobin
    {% if lb_health_check_enabled %}
    option httpchk GET /health
    {% endif %}
    
    {% for server in lb_backend_servers %}
    server {{ server.name }} {{ server.address }}:{{ server.port }} {% if server.check | default(lb_health_check_enabled) %}check inter {{ lb_health_check_interval }}{% endif %}
    {% endfor %}
EOF

cat > roles/load_balancer/handlers/main.yml << 'EOF'
---
# Load Balancer Handlers

- name: restart load balancer
  service:
    name: "{{ lb_type }}"
    state: restarted
EOF
```

Create a comprehensive test with complex dependencies:

```yaml
# Create test-complex-dependencies.yml
cat > test-complex-dependencies.yml << 'EOF'
---
- name: Test complex role dependencies
  hosts: web_servers
  become: yes
  vars:
    # Global settings
    environment: "production"
    region: "us-east-1"
    
    # Security settings
    security_hardening_enabled: true
    security_ssh_port: 2222
    
    # Monitoring settings
    monitoring_enabled: true
    monitoring_api_key: "test-api-key-12345"
    
    # Backup settings
    backup_enabled: true
    
    # Database settings
    database_engine: "postgresql"
    database_name: "app_db"
    database_port: 5432
    database_backup_schedule: "0 1 * * *"
    
    # Load balancer settings
    lb_backend_servers:
      - name: "web1"
        address: "192.168.1.10"
        port: 80
        check: true
      - name: "web2"
        address: "192.168.1.11"
        port: 80
        check: true
    
    # Web server settings
    webserver_domain: "app.company.com"
    webserver_admin_email: "devops@company.com"
    
  roles:
    - role: enterprise_database
    - role: enterprise_webserver
    - role: load_balancer

- name: Display role execution summary
  hosts: web_servers
  become: yes
  tasks:
    - name: Show applied configurations
      debug:
        msg: |
          Role Execution Summary:
          
          Security Baseline:
          - Applied: {{ security_baseline_applied | default(false) }}
          - Timestamp: {{ security_baseline_timestamp | default('N/A') }}
          
          Monitoring Agent:
          - Installed: {{ monitoring_agent_installed | default(false) }}
          - Version: {{ monitoring_agent_version | default('N/A') }}
          - Timestamp: {{ monitoring_agent_timestamp | default('N/A') }}
          
          Backup Agent:
          - Configured: {{ backup_agent_configured | default(false) }}
          - Type: {{ backup_agent_type | default('N/A') }}
          - Timestamp: {{ backup_agent_timestamp | default('N/A') }}
          
          Load Balancer:
          - Configured: {{ load_balancer_configured | default(false) }}
          - Type: {{ load_balancer_type | default('N/A') }}
          - Backend Count: {{ load_balancer_backend_count | default('N/A') }}
          - Timestamp: {{ load_balancer_timestamp | default('N/A') }}
EOF

# Test dependency resolution and execution order
ansible-playbook --list-tasks test-complex-dependencies.yml -i inventory.ini
```

## Exercise 3: Inter-Role Communication and Data Sharing (8 minutes)

### Task: Implement communication patterns between roles

Create a role that consumes data from other roles:

```bash
# Create application deployment role
ansible-galaxy init roles/application_stack

cat > roles/application_stack/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: application_stack
  namespace: company
  author: Application Team
  description: Complete application stack orchestrator
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
  galaxy_tags:
    - application
    - stack
    - orchestration

# Dependencies that provide data to this role
dependencies:
  - role: company.enterprise_database
    vars:
      database_engine: "{{ app_database_engine | default('postgresql') }}"
      database_name: "{{ app_name }}_db"
      database_port: "{{ app_database_port | default(5432) }}"
  
  - role: company.enterprise_webserver
    vars:
      webserver_domain: "{{ app_domain }}"
      webserver_admin_email: "{{ app_admin_email }}"
      virtual_hosts:
        - name: "{{ app_domain }}"
          document_root: "/var/www/{{ app_name }}"
          ssl_enabled: "{{ app_ssl_enabled | default(true) }}"
          upstream: "{{ app_name }}_backend"
      upstream_servers:
        - name: "{{ app_name }}_backend"
          servers: "{{ app_backend_servers | default(['127.0.0.1:8080']) }}"
  
  - role: company.load_balancer
    when: app_load_balancer_enabled | default(false)
    vars:
      lb_backend_servers: "{{ app_backend_servers_config | default([]) }}"
EOF

cat > roles/application_stack/defaults/main.yml << 'EOF'
---
# Application Stack Default Variables

app_name: "myapp"
app_version: "1.0.0"
app_domain: "{{ app_name }}.company.com"
app_admin_email: "admin@company.com"

# Database configuration
app_database_engine: "postgresql"
app_database_port: 5432
app_database_user: "{{ app_name }}_user"
app_database_password: "{{ vault_app_database_password | default('changeme') }}"

# Web server configuration
app_ssl_enabled: true
app_load_balancer_enabled: false

# Backend servers for load balancing
app_backend_servers:
  - "127.0.0.1:8080"

app_backend_servers_config:
  - name: "app1"
    address: "127.0.0.1"
    port: 8080
    check: true

# Application-specific settings
app_environment: "production"
app_debug_enabled: false
app_log_level: "INFO"

# Integration settings
app_monitoring_enabled: true
app_backup_enabled: true
app_security_hardening: true
EOF

cat > roles/application_stack/tasks/main.yml << 'EOF'
---
# Application Stack Role - Main Tasks

- name: Display application stack information
  debug:
    msg: |
      Deploying application stack: {{ app_name }}
      Version: {{ app_version }}
      Domain: {{ app_domain }}
      Environment: {{ app_environment }}
      
      Database Configuration:
      - Engine: {{ app_database_engine }}
      - Port: {{ app_database_port }}
      
      Load Balancer: {{ 'Enabled' if app_load_balancer_enabled else 'Disabled' }}
      Backend Servers: {{ app_backend_servers | length }}

- name: Validate role dependencies were executed
  assert:
    that:
      - security_baseline_applied | default(false)
      - monitoring_agent_installed | default(false) or not app_monitoring_enabled
      - backup_agent_configured | default(false) or not app_backup_enabled
    fail_msg: "Required role dependencies were not properly executed"
    success_msg: "All role dependencies executed successfully"

- name: Create application directory structure
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ webserver_user | default('www-data') }}"
    group: "{{ webserver_group | default('www-data') }}"
    mode: '0755'
  loop:
    - "/var/www/{{ app_name }}"
    - "/var/www/{{ app_name }}/config"
    - "/var/www/{{ app_name }}/logs"
    - "/var/www/{{ app_name }}/uploads"

- name: Generate application configuration
  template:
    src: app_config.yml.j2
    dest: "/var/www/{{ app_name }}/config/config.yml"
    owner: "{{ webserver_user | default('www-data') }}"
    group: "{{ webserver_group | default('www-data') }}"
    mode: '0640'

- name: Create database connection configuration
  template:
    src: database.yml.j2
    dest: "/var/www/{{ app_name }}/config/database.yml"
    owner: "{{ webserver_user | default('www-data') }}"
    group: "{{ webserver_group | default('www-data') }}"
    mode: '0600'  # Sensitive database credentials

- name: Deploy application files
  copy:
    src: "{{ item }}"
    dest: "/var/www/{{ app_name }}/"
    owner: "{{ webserver_user | default('www-data') }}"
    group: "{{ webserver_group | default('www-data') }}"
    mode: '0644'
  with_fileglob:
    - "files/app/*"
  when: app_deploy_files | default(true)

- name: Configure application service
  template:
    src: app.service.j2
    dest: "/etc/systemd/system/{{ app_name }}.service"
    mode: '0644'
  notify: 
    - reload systemd
    - restart application

- name: Start and enable application service
  service:
    name: "{{ app_name }}"
    state: started
    enabled: yes

- name: Collect role execution data
  set_fact:
    application_stack_info:
      name: "{{ app_name }}"
      version: "{{ app_version }}"
      domain: "{{ app_domain }}"
      database_engine: "{{ app_database_engine }}"
      security_applied: "{{ security_baseline_applied | default(false) }}"
      monitoring_installed: "{{ monitoring_agent_installed | default(false) }}"
      backup_configured: "{{ backup_agent_configured | default(false) }}"
      load_balancer_enabled: "{{ app_load_balancer_enabled }}"
      deployment_timestamp: "{{ ansible_date_time.iso8601 }}"

- name: Display final application stack status
  debug:
    msg: |
      Application Stack Deployment Complete:
      {{ application_stack_info | to_nice_yaml }}
EOF

# Create application configuration templates
mkdir -p roles/application_stack/templates
cat > roles/application_stack/templates/app_config.yml.j2 << 'EOF'
# {{ ansible_managed }}
# Application Configuration for {{ app_name }}

application:
  name: {{ app_name }}
  version: {{ app_version }}
  environment: {{ app_environment }}
  debug: {{ app_debug_enabled | lower }}
  log_level: {{ app_log_level }}

server:
  host: 0.0.0.0
  port: 8080
  domain: {{ app_domain }}

security:
  baseline_applied: {{ security_baseline_applied | default(false) | lower }}
  ssh_port: {{ security_ssh_port | default(22) }}
  ssl_enabled: {{ app_ssl_enabled | lower }}

monitoring:
  enabled: {{ monitoring_agent_installed | default(false) | lower }}
  agent_type: {{ monitoring_agent_type | default('none') }}

backup:
  enabled: {{ backup_agent_configured | default(false) | lower }}
  type: {{ backup_agent_type | default('none') }}

load_balancer:
  enabled: {{ app_load_balancer_enabled | lower }}
  {% if app_load_balancer_enabled %}
  type: {{ load_balancer_type | default('none') }}
  backend_count: {{ load_balancer_backend_count | default(0) }}
  {% endif %}

deployment:
  timestamp: {{ ansible_date_time.iso8601 }}
  deployed_by: {{ ansible_user_id }}
EOF

cat > roles/application_stack/templates/database.yml.j2 << 'EOF'
# {{ ansible_managed }}
# Database Configuration for {{ app_name }}

database:
  engine: {{ app_database_engine }}
  host: localhost
  port: {{ app_database_port }}
  name: {{ app_name }}_db
  user: {{ app_database_user }}
  password: {{ app_database_password }}
  
  # Connection pool settings
  pool_size: 10
  max_overflow: 20
  pool_timeout: 30
  pool_recycle: 3600

  # SSL settings
  ssl_enabled: {{ app_ssl_enabled | lower }}
  
backup:
  enabled: {{ backup_agent_configured | default(false) | lower }}
  schedule: {{ database_backup_schedule | default('0 2 * * *') }}
EOF

cat > roles/application_stack/templates/app.service.j2 << 'EOF'
# {{ ansible_managed }}
[Unit]
Description={{ app_name | title }} Application
After=network.target{% if app_database_engine == 'postgresql' %} postgresql.service{% elif app_database_engine == 'mysql' %} mysql.service{% endif %}

Requires={% if app_database_engine == 'postgresql' %}postgresql.service{% elif app_database_engine == 'mysql' %}mysql.service{% endif %}

[Service]
Type=simple
User={{ webserver_user | default('www-data') }}
Group={{ webserver_group | default('www-data') }}
WorkingDirectory=/var/www/{{ app_name }}
ExecStart=/usr/bin/python3 -m http.server 8080
Restart=always
RestartSec=10

Environment=APP_ENV={{ app_environment }}
Environment=APP_DEBUG={{ app_debug_enabled | lower }}
Environment=DATABASE_URL={{ app_database_engine }}://{{ app_database_user }}:{{ app_database_password }}@localhost:{{ app_database_port }}/{{ app_name }}_db

[Install]
WantedBy=multi-user.target
EOF

cat > roles/application_stack/handlers/main.yml << 'EOF'
---
# Application Stack Handlers

- name: reload systemd
  systemd:
    daemon_reload: yes

- name: restart application
  service:
    name: "{{ app_name }}"
    state: restarted
EOF
```

Test inter-role communication:

```yaml
# Create test-inter-role-communication.yml
cat > test-inter-role-communication.yml << 'EOF'
---
- name: Test inter-role communication
  hosts: web_servers
  become: yes
  vars:
    # Application configuration
    app_name: "enterprise_app"
    app_version: "2.1.0"
    app_domain: "app.enterprise.local"
    app_admin_email: "devops@enterprise.local"
    app_environment: "production"
    
    # Enable all features for testing
    app_load_balancer_enabled: true
    app_monitoring_enabled: true
    app_backup_enabled: true
    app_security_hardening: true
    
    # Monitoring configuration
    monitoring_api_key: "test-monitoring-key"
    
    # Database configuration
    app_database_engine: "postgresql"
    vault_app_database_password: "secure_db_password"
    
  roles:
    - role: application_stack

- name: Verify inter-role data sharing
  hosts: web_servers
  become: yes
  tasks:
    - name: Display shared data between roles
      debug:
        msg: |
          Inter-Role Communication Verification:
          
          Application Stack Info:
          {{ application_stack_info | to_nice_yaml }}
          
          Data Shared from Dependencies:
          - Security Baseline Applied: {{ security_baseline_applied | default('Not Set') }}
          - Monitoring Agent Installed: {{ monitoring_agent_installed | default('Not Set') }}
          - Backup Agent Configured: {{ backup_agent_configured | default('Not Set') }}
          - Load Balancer Configured: {{ load_balancer_configured | default('Not Set') }}
          
          Configuration Files Generated:
          - App Config: /var/www/{{ app_name }}/config/config.yml
          - Database Config: /var/www/{{ app_name }}/config/database.yml
          - Service File: /etc/systemd/system/{{ app_name }}.service
    
    - name: Verify configuration file contents
      slurp:
        src: "/var/www/{{ app_name }}/config/config.yml"
      register: app_config_content
    
    - name: Display application configuration
      debug:
        msg: |
          Application Configuration Content:
          {{ app_config_content.content | b64decode }}
EOF

# Test the complete inter-role communication
ansible-playbook --list-tasks test-inter-role-communication.yml -i inventory.ini
```

## Verification and Discussion

### 1. Check Results
```bash
# Test basic dependencies
echo "=== Testing Basic Dependencies ==="
ansible-playbook --list-tasks test-dependencies.yml -i inventory.ini

# Test complex dependencies
echo "=== Testing Complex Dependencies ==="
ansible-playbook --list-tasks test-complex-dependencies.yml -i inventory.ini

# Test inter-role communication
echo "=== Testing Inter-Role Communication ==="
ansible-playbook --list-tasks test-inter-role-communication.yml -i inventory.ini

# Check role dependency tree
ansible-galaxy list
```

### 2. Discussion Points
- How do you manage role dependencies in your current automation?
- What strategies do you use for sharing data between roles?
- How do you handle conditional dependencies based on environment or configuration?
- What are the performance implications of complex role dependencies?

### 3. Clean Up
```bash
# Keep roles for next exercises
# rm -rf roles/ *.yml
```

## Key Takeaways
- Role dependencies enable modular and reusable automation components
- Conditional dependencies provide flexibility for different deployment scenarios
- Inter-role communication through facts enables sophisticated orchestration
- Proper dependency management prevents circular dependencies and conflicts
- Role execution order is determined by dependency relationships
- Data sharing between roles enables complex application stack deployments

## Next Steps
Proceed to Lab 5.3: Role Reusability and Parameterization

# Lab 5.3: Role Reusability and Parameterization

## Objective
Create highly reusable roles with advanced parameterization patterns for maximum flexibility across different environments and use cases.

## Duration
25 minutes

## Prerequisites
- Completed Labs 5.1 and 5.2
- Understanding of role dependencies and structure

## Lab Setup

```bash
cd ~/ansible-labs/module-05
mkdir -p lab-5.3
cd lab-5.3

# Create inventory for testing
cat > inventory.ini << 'EOF'
[web_servers]
web1 ansible_host=localhost ansible_connection=local
web2 ansible_host=localhost ansible_connection=local

[database_servers]
db1 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Advanced Parameterization Patterns (10 minutes)

### Task: Create a highly parameterized service role

Create a generic service management role:

```bash
# Create generic service role
ansible-galaxy init roles/generic_service

# Update metadata for maximum reusability
cat > roles/generic_service/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: generic_service
  namespace: company
  author: Platform Team
  description: Highly reusable generic service management role
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
    - name: EL
      versions: [8, 9]
  galaxy_tags:
    - service
    - generic
    - reusable
    - parameterized

dependencies: []
EOF
```

Create comprehensive default variables with multiple parameterization levels:

```yaml
# Create roles/generic_service/defaults/main.yml
cat > roles/generic_service/defaults/main.yml << 'EOF'
---
# Generic Service Role - Comprehensive Default Variables

# === BASIC SERVICE CONFIGURATION ===
service_name: "myservice"
service_version: "latest"
service_description: "{{ service_name | title }} Service"
service_user: "{{ service_name }}"
service_group: "{{ service_name }}"
service_home: "/opt/{{ service_name }}"

# === INSTALLATION CONFIGURATION ===
service_install_method: "package"  # package, binary, source, container
service_package_name: "{{ service_name }}"
service_package_state: "present"
service_repository_url: ""
service_binary_url: ""
service_source_url: ""

# === DIRECTORY STRUCTURE ===
service_directories:
  base: "{{ service_home }}"
  config: "{{ service_home }}/config"
  data: "{{ service_home }}/data"
  logs: "{{ service_home }}/logs"
  temp: "{{ service_home }}/tmp"
  bin: "{{ service_home }}/bin"

# === CONFIGURATION MANAGEMENT ===
service_config_template: "{{ service_name }}.conf.j2"
service_config_file: "{{ service_directories.config }}/{{ service_name }}.conf"
service_config_mode: "0644"
service_config_backup: true

# Additional configuration files
service_additional_configs: []
# Example:
# service_additional_configs:
#   - src: "logging.conf.j2"
#     dest: "{{ service_directories.config }}/logging.conf"
#     mode: "0644"
#   - src: "database.yml.j2"
#     dest: "{{ service_directories.config }}/database.yml"
#     mode: "0600"

# === SERVICE MANAGEMENT ===
service_systemd_enabled: true
service_systemd_template: "generic.service.j2"
service_systemd_file: "/etc/systemd/system/{{ service_name }}.service"

# Service startup configuration
service_start_command: ""
service_stop_command: ""
service_reload_command: ""
service_restart_command: ""
service_status_command: ""

# Service runtime configuration
service_environment_vars: {}
# Example:
# service_environment_vars:
#   JAVA_OPTS: "-Xmx2g -Xms1g"
#   APP_ENV: "production"
#   LOG_LEVEL: "INFO"

service_command_line_args: []
service_working_directory: "{{ service_home }}"
service_pid_file: "/var/run/{{ service_name }}.pid"

# === NETWORK CONFIGURATION ===
service_ports: []
# Example:
# service_ports:
#   - port: 8080
#     protocol: tcp
#     description: "HTTP API"
#   - port: 8443
#     protocol: tcp
#     description: "HTTPS API"

service_bind_address: "0.0.0.0"
service_listen_port: 8080

# === SECURITY CONFIGURATION ===
service_security_enabled: true
service_firewall_rules: []
# Automatically generated from service_ports if empty

service_ssl_enabled: false
service_ssl_certificate: "/etc/ssl/certs/{{ service_name }}.crt"
service_ssl_private_key: "/etc/ssl/private/{{ service_name }}.key"

# === MONITORING AND HEALTH CHECKS ===
service_monitoring_enabled: true
service_health_check_enabled: true
service_health_check_url: "http://{{ service_bind_address }}:{{ service_listen_port }}/health"
service_health_check_interval: 30
service_health_check_timeout: 5
service_health_check_retries: 3

# Metrics configuration
service_metrics_enabled: false
service_metrics_port: 9090
service_metrics_path: "/metrics"

# === LOGGING CONFIGURATION ===
service_logging_enabled: true
service_log_level: "INFO"
service_log_file: "{{ service_directories.logs }}/{{ service_name }}.log"
service_log_rotation_enabled: true
service_log_rotation_size: "100M"
service_log_rotation_count: 10

# === BACKUP CONFIGURATION ===
service_backup_enabled: false
service_backup_directories:
  - "{{ service_directories.config }}"
  - "{{ service_directories.data }}"
service_backup_schedule: "0 2 * * *"
service_backup_retention: 30

# === PERFORMANCE TUNING ===
service_performance_profile: "default"  # default, high_performance, low_resource
service_memory_limit: ""
service_cpu_limit: ""
service_file_descriptor_limit: 65536

# Performance profiles
service_performance_profiles:
  default:
    memory_limit: "1G"
    cpu_limit: "1.0"
    worker_processes: 2
  high_performance:
    memory_limit: "4G"
    cpu_limit: "4.0"
    worker_processes: 8
  low_resource:
    memory_limit: "512M"
    cpu_limit: "0.5"
    worker_processes: 1

# === ENVIRONMENT-SPECIFIC OVERRIDES ===
service_environment: "{{ environment | default('production') }}"
service_environment_configs:
  development:
    log_level: "DEBUG"
    health_check_interval: 10
    backup_enabled: false
  staging:
    log_level: "INFO"
    health_check_interval: 20
    backup_enabled: false
  production:
    log_level: "WARN"
    health_check_interval: 30
    backup_enabled: true

# === INTEGRATION HOOKS ===
service_pre_install_tasks: []
service_post_install_tasks: []
service_pre_configure_tasks: []
service_post_configure_tasks: []
service_pre_start_tasks: []
service_post_start_tasks: []

# === VALIDATION CONFIGURATION ===
service_validation_enabled: true
service_validation_tests: []
# Example:
# service_validation_tests:
#   - name: "Check service port"
#     uri:
#       url: "http://localhost:{{ service_listen_port }}/health"
#       method: GET
#       status_code: 200
#   - name: "Check log file exists"
#     stat:
#       path: "{{ service_log_file }}"
EOF
```

Create the main tasks with advanced parameterization:

```yaml
# Create roles/generic_service/tasks/main.yml
cat > roles/generic_service/tasks/main.yml << 'EOF'
---
# Generic Service Role - Main Tasks

- name: Load environment-specific configuration
  include_vars: "environments/{{ service_environment }}.yml"
  when: 
    - service_environment is defined
    - service_environment != ""
  ignore_errors: yes
  tags: always

- name: Merge environment-specific configurations
  set_fact:
    service_log_level: "{{ service_environment_configs[service_environment].log_level | default(service_log_level) }}"
    service_health_check_interval: "{{ service_environment_configs[service_environment].health_check_interval | default(service_health_check_interval) }}"
    service_backup_enabled: "{{ service_environment_configs[service_environment].backup_enabled | default(service_backup_enabled) }}"
  when: service_environment in service_environment_configs
  tags: always

- name: Apply performance profile settings
  set_fact:
    service_memory_limit: "{{ service_performance_profiles[service_performance_profile].memory_limit | default(service_memory_limit) }}"
    service_cpu_limit: "{{ service_performance_profiles[service_performance_profile].cpu_limit | default(service_cpu_limit) }}"
    service_worker_processes: "{{ service_performance_profiles[service_performance_profile].worker_processes | default(2) }}"
  when: service_performance_profile in service_performance_profiles
  tags: always

- name: Display service configuration summary
  debug:
    msg: |
      Configuring Service: {{ service_name }}
      Version: {{ service_version }}
      Environment: {{ service_environment }}
      Performance Profile: {{ service_performance_profile }}
      Install Method: {{ service_install_method }}
      Monitoring Enabled: {{ service_monitoring_enabled }}
      Backup Enabled: {{ service_backup_enabled }}
      SSL Enabled: {{ service_ssl_enabled }}
  tags: always

- name: Execute pre-install tasks
  include_tasks: "{{ item }}"
  loop: "{{ service_pre_install_tasks }}"
  when: service_pre_install_tasks | length > 0
  tags: [install, pre_install]

- name: Create service user and group
  group:
    name: "{{ service_group }}"
    state: present
  tags: install

- name: Create service user
  user:
    name: "{{ service_user }}"
    group: "{{ service_group }}"
    home: "{{ service_home }}"
    shell: /bin/false
    system: yes
    create_home: no
  tags: install

- name: Create service directories
  file:
    path: "{{ item.value }}"
    state: directory
    owner: "{{ service_user }}"
    group: "{{ service_group }}"
    mode: '0755'
  loop: "{{ service_directories | dict2items }}"
  tags: install

- name: Install service based on method
  include_tasks: "install/{{ service_install_method }}.yml"
  tags: install

- name: Execute post-install tasks
  include_tasks: "{{ item }}"
  loop: "{{ service_post_install_tasks }}"
  when: service_post_install_tasks | length > 0
  tags: [install, post_install]

- name: Execute pre-configure tasks
  include_tasks: "{{ item }}"
  loop: "{{ service_pre_configure_tasks }}"
  when: service_pre_configure_tasks | length > 0
  tags: [configure, pre_configure]

- name: Generate main configuration file
  template:
    src: "{{ service_config_template }}"
    dest: "{{ service_config_file }}"
    owner: "{{ service_user }}"
    group: "{{ service_group }}"
    mode: "{{ service_config_mode }}"
    backup: "{{ service_config_backup }}"
  notify: restart service
  tags: configure

- name: Generate additional configuration files
  template:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    owner: "{{ service_user }}"
    group: "{{ service_group }}"
    mode: "{{ item.mode | default('0644') }}"
    backup: "{{ item.backup | default(true) }}"
  loop: "{{ service_additional_configs }}"
  when: service_additional_configs | length > 0
  notify: restart service
  tags: configure

- name: Generate systemd service file
  template:
    src: "{{ service_systemd_template }}"
    dest: "{{ service_systemd_file }}"
    mode: '0644'
  notify:
    - reload systemd
    - restart service
  when: service_systemd_enabled
  tags: configure

- name: Configure firewall rules
  include_tasks: firewall.yml
  when: service_security_enabled and (service_ports | length > 0 or service_firewall_rules | length > 0)
  tags: [configure, security]

- name: Execute post-configure tasks
  include_tasks: "{{ item }}"
  loop: "{{ service_post_configure_tasks }}"
  when: service_post_configure_tasks | length > 0
  tags: [configure, post_configure]

- name: Execute pre-start tasks
  include_tasks: "{{ item }}"
  loop: "{{ service_pre_start_tasks }}"
  when: service_pre_start_tasks | length > 0
  tags: [service, pre_start]

- name: Start and enable service
  service:
    name: "{{ service_name }}"
    state: started
    enabled: yes
  when: service_systemd_enabled
  tags: service

- name: Execute post-start tasks
  include_tasks: "{{ item }}"
  loop: "{{ service_post_start_tasks }}"
  when: service_post_start_tasks | length > 0
  tags: [service, post_start]

- name: Configure monitoring
  include_tasks: monitoring.yml
  when: service_monitoring_enabled
  tags: [monitoring]

- name: Configure backup
  include_tasks: backup.yml
  when: service_backup_enabled
  tags: [backup]

- name: Run service validation
  include_tasks: validate.yml
  when: service_validation_enabled
  tags: [validate]

- name: Set service deployment facts
  set_fact:
    "{{ service_name }}_service_info":
      name: "{{ service_name }}"
      version: "{{ service_version }}"
      user: "{{ service_user }}"
      home: "{{ service_home }}"
      ports: "{{ service_ports }}"
      environment: "{{ service_environment }}"
      performance_profile: "{{ service_performance_profile }}"
      monitoring_enabled: "{{ service_monitoring_enabled }}"
      backup_enabled: "{{ service_backup_enabled }}"
      ssl_enabled: "{{ service_ssl_enabled }}"
      deployment_timestamp: "{{ ansible_date_time.iso8601 }}"
  tags: always
EOF
```

Create installation method tasks:

```bash
# Create installation method directories
mkdir -p roles/generic_service/tasks/install

# Package installation method
cat > roles/generic_service/tasks/install/package.yml << 'EOF'
---
# Package installation method

- name: Update package cache
  package:
    update_cache: yes
  when: ansible_os_family in ['Debian', 'RedHat']

- name: Install service package
  package:
    name: "{{ service_package_name }}"
    state: "{{ service_package_state }}"

- name: Verify package installation
  package_facts:

- name: Assert package is installed
  assert:
    that:
      - service_package_name in ansible_facts.packages
    fail_msg: "Package {{ service_package_name }} was not installed successfully"
    success_msg: "Package {{ service_package_name }} installed successfully"
EOF

# Binary installation method
cat > roles/generic_service/tasks/install/binary.yml << 'EOF'
---
# Binary installation method

- name: Download service binary
  get_url:
    url: "{{ service_binary_url }}"
    dest: "{{ service_directories.bin }}/{{ service_name }}"
    owner: "{{ service_user }}"
    group: "{{ service_group }}"
    mode: '0755'
  when: service_binary_url != ""

- name: Create symlink to binary
  file:
    src: "{{ service_directories.bin }}/{{ service_name }}"
    dest: "/usr/local/bin/{{ service_name }}"
    state: link
  when: service_binary_url != ""
EOF

# Container installation method
cat > roles/generic_service/tasks/install/container.yml << 'EOF'
---
# Container installation method

- name: Install container runtime
  package:
    name: docker.io
    state: present

- name: Start and enable Docker
  service:
    name: docker
    state: started
    enabled: yes

- name: Pull service container image
  docker_image:
    name: "{{ service_container_image | default(service_name + ':' + service_version) }}"
    source: pull

- name: Create container service script
  template:
    src: container-service.sh.j2
    dest: "{{ service_directories.bin }}/{{ service_name }}-container.sh"
    mode: '0755'
    owner: "{{ service_user }}"
    group: "{{ service_group }}"
EOF
```

Create generic templates:

```bash
# Create templates directory
mkdir -p roles/generic_service/templates

# Generic systemd service template
cat > roles/generic_service/templates/generic.service.j2 << 'EOF'
# {{ ansible_managed }}
[Unit]
Description={{ service_description }}
After=network.target
{% if service_dependencies is defined %}
{% for dep in service_dependencies %}
Requires={{ dep }}
After={{ dep }}
{% endfor %}
{% endif %}

[Service]
Type=simple
User={{ service_user }}
Group={{ service_group }}
WorkingDirectory={{ service_working_directory }}

{% if service_start_command != "" %}
ExecStart={{ service_start_command }}
{% else %}
ExecStart={{ service_directories.bin }}/{{ service_name }}{% if service_command_line_args | length > 0 %} {{ service_command_line_args | join(' ') }}{% endif %}
{% endif %}

{% if service_stop_command != "" %}
ExecStop={{ service_stop_command }}
{% endif %}

{% if service_reload_command != "" %}
ExecReload={{ service_reload_command }}
{% endif %}

{% if service_pid_file != "" %}
PIDFile={{ service_pid_file }}
{% endif %}

Restart=always
RestartSec=10

# Environment variables
{% for key, value in service_environment_vars.items() %}
Environment={{ key }}={{ value }}
{% endfor %}

# Resource limits
{% if service_memory_limit != "" %}
MemoryLimit={{ service_memory_limit }}
{% endif %}

{% if service_cpu_limit != "" %}
CPUQuota={{ (service_cpu_limit | float * 100) | int }}%
{% endif %}

LimitNOFILE={{ service_file_descriptor_limit }}

[Install]
WantedBy=multi-user.target
EOF

# Generic configuration template
cat > roles/generic_service/templates/generic.conf.j2 << 'EOF'
# {{ ansible_managed }}
# Generic Service Configuration for {{ service_name }}

[server]
bind_address = {{ service_bind_address }}
listen_port = {{ service_listen_port }}
worker_processes = {{ service_worker_processes | default(2) }}

[logging]
enabled = {{ service_logging_enabled | lower }}
level = {{ service_log_level }}
file = {{ service_log_file }}

{% if service_log_rotation_enabled %}
[log_rotation]
enabled = true
size = {{ service_log_rotation_size }}
count = {{ service_log_rotation_count }}
{% endif %}

{% if service_ssl_enabled %}
[ssl]
enabled = true
certificate = {{ service_ssl_certificate }}
private_key = {{ service_ssl_private_key }}
{% endif %}

{% if service_health_check_enabled %}
[health_check]
enabled = true
interval = {{ service_health_check_interval }}
timeout = {{ service_health_check_timeout }}
retries = {{ service_health_check_retries }}
{% endif %}

{% if service_metrics_enabled %}
[metrics]
enabled = true
port = {{ service_metrics_port }}
path = {{ service_metrics_path }}
{% endif %}

[environment]
name = {{ service_environment }}
performance_profile = {{ service_performance_profile }}

# Custom configuration sections
{% for key, value in service_custom_config.items() %}
[{{ key }}]
{% if value is mapping %}
{% for subkey, subvalue in value.items() %}
{{ subkey }} = {{ subvalue }}
{% endfor %}
{% else %}
value = {{ value }}
{% endif %}
{% endfor %}
EOF
```

Create supporting task files:

```yaml
# Create roles/generic_service/tasks/firewall.yml
cat > roles/generic_service/tasks/firewall.yml << 'EOF'
---
# Firewall configuration for service

- name: Generate firewall rules from service ports
  set_fact:
    computed_firewall_rules: "{{ computed_firewall_rules | default([]) + [{'port': item.port, 'protocol': item.protocol, 'description': item.description | default('Service port')}] }}"
  loop: "{{ service_ports }}"
  when: service_firewall_rules | length == 0

- name: Use provided firewall rules
  set_fact:
    computed_firewall_rules: "{{ service_firewall_rules }}"
  when: service_firewall_rules | length > 0

- name: Configure UFW firewall rules (Debian/Ubuntu)
  ufw:
    rule: allow
    port: "{{ item.port }}"
    proto: "{{ item.protocol }}"
    comment: "{{ item.description | default('Service port') }}"
  loop: "{{ computed_firewall_rules }}"
  when: ansible_os_family == "Debian"

- name: Configure firewalld rules (RHEL/CentOS)
  firewalld:
    port: "{{ item.port }}/{{ item.protocol }}"
    permanent: yes
    state: enabled
    immediate: yes
  loop: "{{ computed_firewall_rules }}"
  when: ansible_os_family == "RedHat"
EOF

# Create roles/generic_service/tasks/monitoring.yml
cat > roles/generic_service/tasks/monitoring.yml << 'EOF'
---
# Monitoring configuration for service

- name: Create monitoring configuration
  template:
    src: monitoring.yml.j2
    dest: "{{ service_directories.config }}/monitoring.yml"
    owner: "{{ service_user }}"
    group: "{{ service_group }}"
    mode: '0644'

- name: Configure health check endpoint
  uri:
    url: "{{ service_health_check_url }}"
    method: GET
    status_code: 200
    timeout: "{{ service_health_check_timeout }}"
  register: health_check_result
  retries: "{{ service_health_check_retries }}"
  delay: 5
  when: service_health_check_enabled
  ignore_errors: yes

- name: Display health check result
  debug:
    msg: |
      Health Check Result:
      URL: {{ service_health_check_url }}
      Status: {{ 'PASSED' if health_check_result.status == 200 else 'FAILED' }}
  when: service_health_check_enabled
EOF

# Create roles/generic_service/tasks/backup.yml
cat > roles/generic_service/tasks/backup.yml << 'EOF'
---
# Backup configuration for service

- name: Create backup directory
  file:
    path: "/backup/{{ service_name }}"
    state: directory
    owner: "{{ service_user }}"
    group: "{{ service_group }}"
    mode: '0750'

- name: Create backup script
  template:
    src: backup.sh.j2
    dest: "/usr/local/bin/backup-{{ service_name }}.sh"
    mode: '0755'

- name: Configure backup cron job
  cron:
    name: "{{ service_name }} backup"
    job: "/usr/local/bin/backup-{{ service_name }}.sh"
    minute: "{{ service_backup_schedule.split()[1] }}"
    hour: "{{ service_backup_schedule.split()[2] }}"
    day: "{{ service_backup_schedule.split()[3] }}"
    month: "{{ service_backup_schedule.split()[4] }}"
    weekday: "{{ service_backup_schedule.split()[5] }}"
    user: "{{ service_user }}"
EOF

# Create roles/generic_service/tasks/validate.yml
cat > roles/generic_service/tasks/validate.yml << 'EOF'
---
# Service validation tasks

- name: Check if service is running
  service_facts:

- name: Validate service is active
  assert:
    that:
      - ansible_facts.services[service_name + '.service'].state == 'running'
    fail_msg: "Service {{ service_name }} is not running"
    success_msg: "Service {{ service_name }} is running successfully"

- name: Validate service ports are listening
  wait_for:
    port: "{{ item.port }}"
    host: "{{ service_bind_address }}"
    timeout: 10
  loop: "{{ service_ports }}"
  when: service_ports | length > 0

- name: Run custom validation tests
  include_tasks: "{{ item }}"
  loop: "{{ service_validation_tests }}"
  when: service_validation_tests | length > 0

- name: Display validation summary
  debug:
    msg: |
      Service Validation Summary:
      - Service Name: {{ service_name }}
      - Service Status: {{ ansible_facts.services[service_name + '.service'].state }}
      - Ports Validated: {{ service_ports | length }}
      - Custom Tests: {{ service_validation_tests | length }}
EOF
```

Create handlers:

```yaml
# Create roles/generic_service/handlers/main.yml
cat > roles/generic_service/handlers/main.yml << 'EOF'
---
# Generic Service Handlers

- name: reload systemd
  systemd:
    daemon_reload: yes

- name: restart service
  service:
    name: "{{ service_name }}"
    state: restarted
  when: service_systemd_enabled

- name: reload service
  service:
    name: "{{ service_name }}"
    state: reloaded
  when: service_systemd_enabled
EOF
```

## Exercise 2: Multi-Environment Configuration (8 minutes)

### Task: Create environment-specific configurations and overrides

Create environment-specific variable files:

```bash
# Create environment-specific directories
mkdir -p roles/generic_service/vars/environments

# Development environment
cat > roles/generic_service/vars/environments/development.yml << 'EOF'
---
# Development Environment Configuration

service_log_level: "DEBUG"
service_health_check_interval: 10
service_backup_enabled: false
service_ssl_enabled: false
service_performance_profile: "low_resource"

# Development-specific ports
service_ports:
  - port: 8080
    protocol: tcp
    description: "Development HTTP API"
  - port: 9090
    protocol: tcp
    description: "Development Metrics"

# Development environment variables
service_environment_vars:
  APP_ENV: "development"
  DEBUG: "true"
  LOG_LEVEL: "DEBUG"
  CACHE_ENABLED: "false"

# Development custom configuration
service_custom_config:
  database:
    host: "localhost"
    port: 5432
    name: "{{ service_name }}_dev"
    pool_size: 5
  cache:
    enabled: false
  features:
    debug_mode: true
    profiling: true
    hot_reload: true
EOF

# Staging environment
cat > roles/generic_service/vars/environments/staging.yml << 'EOF'
---
# Staging Environment Configuration

service_log_level: "INFO"
service_health_check_interval: 20
service_backup_enabled: false
service_ssl_enabled: true
service_performance_profile: "default"

# Staging-specific ports
service_ports:
  - port: 8080
    protocol: tcp
    description: "Staging HTTP API"
  - port: 8443
    protocol: tcp
    description: "Staging HTTPS API"
  - port: 9090
    protocol: tcp
    description: "Staging Metrics"

# Staging environment variables
service_environment_vars:
  APP_ENV: "staging"
  DEBUG: "false"
  LOG_LEVEL: "INFO"
  CACHE_ENABLED: "true"
  SSL_VERIFY: "false"

# Staging custom configuration
service_custom_config:
  database:
    host: "staging-db.internal"
    port: 5432
    name: "{{ service_name }}_staging"
    pool_size: 10
    ssl_mode: "prefer"
  cache:
    enabled: true
    ttl: 300
    type: "redis"
    host: "staging-redis.internal"
  features:
    debug_mode: false
    profiling: true
    hot_reload: false
    rate_limiting: true
EOF

# Production environment
cat > roles/generic_service/vars/environments/production.yml << 'EOF'
---
# Production Environment Configuration

service_log_level: "WARN"
service_health_check_interval: 30
service_backup_enabled: true
service_ssl_enabled: true
service_performance_profile: "high_performance"

# Production-specific ports
service_ports:
  - port: 8080
    protocol: tcp
    description: "Production HTTP API"
  - port: 8443
    protocol: tcp
    description: "Production HTTPS API"

# Production environment variables
service_environment_vars:
  APP_ENV: "production"
  DEBUG: "false"
  LOG_LEVEL: "WARN"
  CACHE_ENABLED: "true"
  SSL_VERIFY: "true"
  MONITORING_ENABLED: "true"

# Production custom configuration
service_custom_config:
  database:
    host: "prod-db-cluster.internal"
    port: 5432
    name: "{{ service_name }}_prod"
    pool_size: 20
    ssl_mode: "require"
    connection_timeout: 30
  cache:
    enabled: true
    ttl: 3600
    type: "redis"
    host: "prod-redis-cluster.internal"
    cluster_mode: true
  features:
    debug_mode: false
    profiling: false
    hot_reload: false
    rate_limiting: true
    circuit_breaker: true
  security:
    encryption_at_rest: true
    audit_logging: true
    access_control: "strict"
EOF
```

Create application-specific role variants:

```bash
# Create web API service variant
ansible-galaxy init roles/web_api_service

cat > roles/web_api_service/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: web_api_service
  namespace: company
  author: API Team
  description: Web API service based on generic service role
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
  galaxy_tags:
    - api
    - web
    - service

dependencies:
  - role: company.generic_service
    vars:
      service_name: "{{ api_service_name | default('api') }}"
      service_version: "{{ api_service_version | default('latest') }}"
      service_install_method: "{{ api_install_method | default('package') }}"
      service_ports:
        - port: "{{ api_http_port | default(8080) }}"
          protocol: tcp
          description: "HTTP API"
        - port: "{{ api_https_port | default(8443) }}"
          protocol: tcp
          description: "HTTPS API"
        - port: "{{ api_metrics_port | default(9090) }}"
          protocol: tcp
          description: "Metrics endpoint"
      service_ssl_enabled: "{{ api_ssl_enabled | default(true) }}"
      service_monitoring_enabled: "{{ api_monitoring_enabled | default(true) }}"
      service_backup_enabled: "{{ api_backup_enabled | default(true) }}"
      service_health_check_url: "http://{{ service_bind_address }}:{{ api_http_port | default(8080) }}/health"
      service_additional_configs:
        - src: "api-routes.yml.j2"
          dest: "{{ service_directories.config }}/routes.yml"
          mode: "0644"
        - src: "api-middleware.yml.j2"
          dest: "{{ service_directories.config }}/middleware.yml"
          mode: "0644"
      service_environment_vars:
        API_VERSION: "{{ api_version | default('v1') }}"
        MAX_REQUEST_SIZE: "{{ api_max_request_size | default('10MB') }}"
        RATE_LIMIT: "{{ api_rate_limit | default('1000/hour') }}"
        CORS_ENABLED: "{{ api_cors_enabled | default('true') }}"
EOF

cat > roles/web_api_service/defaults/main.yml << 'EOF'
---
# Web API Service Default Variables

api_service_name: "web-api"
api_service_version: "1.0.0"
api_version: "v1"

# Port configuration
api_http_port: 8080
api_https_port: 8443
api_metrics_port: 9090

# API-specific settings
api_max_request_size: "10MB"
api_rate_limit: "1000/hour"
api_cors_enabled: true
api_cors_origins: ["*"]

# Authentication
api_auth_enabled: true
api_auth_method: "jwt"  # jwt, oauth, basic
api_jwt_secret: "{{ vault_api_jwt_secret | default('changeme') }}"

# Database connection
api_database_enabled: true
api_database_url: "postgresql://{{ api_db_user }}:{{ api_db_password }}@localhost:5432/{{ api_service_name }}"

# Caching
api_cache_enabled: true
api_cache_ttl: 300

# Routes configuration
api_routes:
  - path: "/health"
    method: "GET"
    handler: "health_check"
  - path: "/api/{{ api_version }}/users"
    method: "GET"
    handler: "list_users"
    auth_required: true
  - path: "/api/{{ api_version }}/users"
    method: "POST"
    handler: "create_user"
    auth_required: true

# Middleware configuration
api_middleware:
  - name: "cors"
    enabled: "{{ api_cors_enabled }}"
    config:
      origins: "{{ api_cors_origins }}"
  - name: "rate_limiter"
    enabled: true
    config:
      limit: "{{ api_rate_limit }}"
  - name: "auth"
    enabled: "{{ api_auth_enabled }}"
    config:
      method: "{{ api_auth_method }}"
EOF

# Create API-specific templates
mkdir -p roles/web_api_service/templates
cat > roles/web_api_service/templates/api-routes.yml.j2 << 'EOF'
# {{ ansible_managed }}
# API Routes Configuration

routes:
{% for route in api_routes %}
  - path: {{ route.path }}
    method: {{ route.method }}
    handler: {{ route.handler }}
    {% if route.auth_required | default(false) %}
    auth_required: true
    {% endif %}
    {% if route.rate_limit is defined %}
    rate_limit: {{ route.rate_limit }}
    {% endif %}
{% endfor %}

# Global route settings
global:
  prefix: "/api/{{ api_version }}"
  timeout: 30
  max_request_size: {{ api_max_request_size }}
EOF

cat > roles/web_api_service/templates/api-middleware.yml.j2 << 'EOF'
# {{ ansible_managed }}
# API Middleware Configuration

middleware:
{% for middleware in api_middleware %}
  - name: {{ middleware.name }}
    enabled: {{ middleware.enabled | lower }}
    {% if middleware.config is defined %}
    config:
      {% for key, value in middleware.config.items() %}
      {{ key }}: {{ value }}
      {% endfor %}
    {% endif %}
{% endfor %}
EOF

cat > roles/web_api_service/tasks/main.yml << 'EOF'
---
# Web API Service Tasks

- name: Display API service information
  debug:
    msg: |
      Deploying Web API Service: {{ api_service_name }}
      Version: {{ api_service_version }}
      API Version: {{ api_version }}
      HTTP Port: {{ api_http_port }}
      HTTPS Port: {{ api_https_port }}
      Authentication: {{ api_auth_enabled }}
      Database: {{ api_database_enabled }}
      Cache: {{ api_cache_enabled }}

- name: Validate API configuration
  assert:
    that:
      - api_service_name is defined
      - api_service_version is defined
      - api_routes | length > 0
    fail_msg: "API service configuration is incomplete"
    success_msg: "API service configuration is valid"

# The generic_service role will be executed via dependency
# Additional API-specific tasks can be added here

- name: Create API documentation
  template:
    src: api-docs.md.j2
    dest: "{{ service_directories.base }}/README.md"
    owner: "{{ service_user }}"
    group: "{{ service_group }}"
    mode: '0644'

- name: Set API service facts
  set_fact:
    api_service_info:
      name: "{{ api_service_name }}"
      version: "{{ api_service_version }}"
      api_version: "{{ api_version }}"
      endpoints: "{{ api_routes | length }}"
      auth_enabled: "{{ api_auth_enabled }}"
      deployment_timestamp: "{{ ansible_date_time.iso8601 }}"
EOF
```

## Exercise 3: Role Composition and Inheritance (7 minutes)

### Task: Create role composition patterns for complex scenarios

Create a microservices stack role that composes multiple service roles:

```bash
# Create microservices stack role
ansible-galaxy init roles/microservices_stack

cat > roles/microservices_stack/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: microservices_stack
  namespace: company
  author: Platform Team
  description: Complete microservices stack orchestrator
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
  galaxy_tags:
    - microservices
    - stack
    - orchestration

dependencies: []  # Dependencies will be managed dynamically
EOF

cat > roles/microservices_stack/defaults/main.yml << 'EOF'
---
# Microservices Stack Default Variables

stack_name: "microservices"
stack_version: "1.0.0"
stack_environment: "production"

# Service definitions
stack_services: []
# Example:
# stack_services:
#   - name: "user-service"
#     type: "web_api_service"
#     version: "1.2.0"
#     port: 8081
#     database_required: true
#   - name: "order-service"
#     type: "web_api_service"
#     version: "1.1.0"
#     port: 8082
#     database_required: true
#   - name: "notification-service"
#     type: "generic_service"
#     version: "1.0.0"
#     port: 8083
#     database_required: false

# Global stack configuration
stack_load_balancer_enabled: true
stack_monitoring_enabled: true
stack_backup_enabled: true
stack_ssl_enabled: true

# Service discovery configuration
stack_service_discovery_enabled: true
stack_service_registry: "consul"  # consul, etcd, zookeeper

# Inter-service communication
stack_service_mesh_enabled: false
stack_service_mesh_type: "istio"  # istio, linkerd, consul-connect

# Database configuration
stack_database_shared: false
stack_database_per_service: true

# Monitoring and observability
stack_tracing_enabled: true
stack_metrics_aggregation: true
stack_log_aggregation: true

# Security
stack_mutual_tls: false
stack_api_gateway_enabled: true
EOF

cat > roles/microservices_stack/tasks/main.yml << 'EOF'
---
# Microservices Stack Tasks

- name: Display microservices stack information
  debug:
    msg: |
      Deploying Microservices Stack: {{ stack_name }}
      Version: {{ stack_version }}
      Environment: {{ stack_environment }}
      Services: {{ stack_services | length }}
      Load Balancer: {{ stack_load_balancer_enabled }}
      Service Discovery: {{ stack_service_discovery_enabled }}
      Service Mesh: {{ stack_service_mesh_enabled }}

- name: Validate stack configuration
  assert:
    that:
      - stack_services | length > 0
      - stack_name is defined
      - stack_environment in ['development', 'staging', 'production']
    fail_msg: "Microservices stack configuration is invalid"
    success_msg: "Microservices stack configuration is valid"

- name: Deploy infrastructure services
  include_tasks: infrastructure.yml
  tags: infrastructure

- name: Deploy application services
  include_tasks: deploy_service.yml
  loop: "{{ stack_services }}"
  loop_control:
    loop_var: service_config
  tags: services

- name: Configure service discovery
  include_tasks: service_discovery.yml
  when: stack_service_discovery_enabled
  tags: service_discovery

- name: Configure load balancer
  include_tasks: load_balancer.yml
  when: stack_load_balancer_enabled
  tags: load_balancer

- name: Configure monitoring
  include_tasks: monitoring.yml
  when: stack_monitoring_enabled
  tags: monitoring

- name: Generate stack documentation
  template:
    src: stack-documentation.md.j2
    dest: "/opt/{{ stack_name }}/README.md"
    mode: '0644'

- name: Set stack deployment facts
  set_fact:
    microservices_stack_info:
      name: "{{ stack_name }}"
      version: "{{ stack_version }}"
      environment: "{{ stack_environment }}"
      services: "{{ stack_services | map(attribute='name') | list }}"
      service_count: "{{ stack_services | length }}"
      load_balancer_enabled: "{{ stack_load_balancer_enabled }}"
      service_discovery_enabled: "{{ stack_service_discovery_enabled }}"
      deployment_timestamp: "{{ ansible_date_time.iso8601 }}"
EOF

# Create service deployment task
cat > roles/microservices_stack/tasks/deploy_service.yml << 'EOF'
---
# Deploy individual service

- name: Display service deployment info
  debug:
    msg: |
      Deploying service: {{ service_config.name }}
      Type: {{ service_config.type }}
      Version: {{ service_config.version }}
      Port: {{ service_config.port }}

- name: Deploy web API service
  include_role:
    name: company.web_api_service
  vars:
    api_service_name: "{{ service_config.name }}"
    api_service_version: "{{ service_config.version }}"
    api_http_port: "{{ service_config.port }}"
    api_https_port: "{{ service_config.port + 1000 }}"
    api_database_enabled: "{{ service_config.database_required | default(false) }}"
    service_environment: "{{ stack_environment }}"
    service_monitoring_enabled: "{{ stack_monitoring_enabled }}"
    service_backup_enabled: "{{ stack_backup_enabled }}"
    service_ssl_enabled: "{{ stack_ssl_enabled }}"
  when: service_config.type == "web_api_service"

- name: Deploy generic service
  include_role:
    name: company.generic_service
  vars:
    service_name: "{{ service_config.name }}"
    service_version: "{{ service_config.version }}"
    service_listen_port: "{{ service_config.port }}"
    service_environment: "{{ stack_environment }}"
    service_monitoring_enabled: "{{ stack_monitoring_enabled }}"
    service_backup_enabled: "{{ stack_backup_enabled }}"
    service_ssl_enabled: "{{ stack_ssl_enabled }}"
    service_ports:
      - port: "{{ service_config.port }}"
        protocol: tcp
        description: "{{ service_config.name }} service port"
  when: service_config.type == "generic_service"

- name: Register service in discovery
  uri:
    url: "http://{{ stack_service_registry_host | default('localhost') }}:8500/v1/agent/service/register"
    method: PUT
    body_format: json
    body:
      ID: "{{ service_config.name }}"
      Name: "{{ service_config.name }}"
      Port: "{{ service_config.port }}"
      Check:
        HTTP: "http://localhost:{{ service_config.port }}/health"
        Interval: "10s"
  when: stack_service_discovery_enabled and stack_service_registry == "consul"
EOF

# Create infrastructure tasks
cat > roles/microservices_stack/tasks/infrastructure.yml << 'EOF'
---
# Deploy infrastructure services

- name: Create stack directory
  file:
    path: "/opt/{{ stack_name }}"
    state: directory
    mode: '0755'

- name: Deploy service registry (Consul)
  include_role:
    name: company.generic_service
  vars:
    service_name: "consul"
    service_version: "1.15.0"
    service_listen_port: 8500
    service_install_method: "binary"
    service_binary_url: "https://releases.hashicorp.com/consul/1.15.0/consul_1.15.0_linux_amd64.zip"
    service_environment: "{{ stack_environment }}"
    service_ports:
      - port: 8500
        protocol: tcp
        description: "Consul HTTP API"
      - port: 8600
        protocol: tcp
        description: "Consul DNS"
  when: stack_service_discovery_enabled and stack_service_registry == "consul"

- name: Deploy metrics collector (Prometheus)
  include_role:
    name: company.generic_service
  vars:
    service_name: "prometheus"
    service_version: "2.40.0"
    service_listen_port: 9090
    service_install_method: "binary"
    service_environment: "{{ stack_environment }}"
    service_ports:
      - port: 9090
        protocol: tcp
        description: "Prometheus metrics"
  when: stack_monitoring_enabled and stack_metrics_aggregation
EOF
```

Create comprehensive test scenarios:

```yaml
# Create test-reusable-roles.yml
cat > test-reusable-roles.yml << 'EOF'
---
- name: Test highly reusable roles
  hosts: web_servers
  become: yes
  vars:
    # Test different service configurations
    test_services:
      - name: "simple-web-service"
        type: "generic_service"
        environment: "development"
        performance_profile: "low_resource"
        ports: [8080]
      - name: "api-service"
        type: "web_api_service"
        environment: "staging"
        performance_profile: "default"
        ssl_enabled: true
      - name: "background-worker"
        type: "generic_service"
        environment: "production"
        performance_profile: "high_performance"
        monitoring_enabled: true
        backup_enabled: true

  tasks:
    - name: Deploy simple web service
      include_role:
        name: generic_service
      vars:
        service_name: "simple-web"
        service_version: "1.0.0"
        service_environment: "development"
        service_performance_profile: "low_resource"
        service_ports:
          - port: 8080
            protocol: tcp
            description: "Simple web service"
        service_install_method: "package"
        service_package_name: "nginx"
        service_monitoring_enabled: false
        service_backup_enabled: false

    - name: Deploy API service
      include_role:
        name: web_api_service
      vars:
        api_service_name: "test-api"
        api_service_version: "2.1.0"
        api_http_port: 8081
        api_https_port: 8444
        api_ssl_enabled: true
        api_auth_enabled: true
        api_database_enabled: true
        service_environment: "staging"

    - name: Deploy microservices stack
      include_role:
        name: microservices_stack
      vars:
        stack_name: "test-stack"
        stack_environment: "development"
        stack_services:
          - name: "user-api"
            type: "web_api_service"
            version: "1.0.0"
            port: 8090
            database_required: true
          - name: "notification-service"
            type: "generic_service"
            version: "1.0.0"
            port: 8091
            database_required: false
        stack_load_balancer_enabled: false
        stack_service_discovery_enabled: false

    - name: Display deployment summary
      debug:
        msg: |
          Role Reusability Test Summary:
          
          Simple Web Service:
          {{ simple_web_service_info | default('Not deployed') | to_nice_yaml }}
          
          API Service:
          {{ api_service_info | default('Not deployed') | to_nice_yaml }}
          
          Microservices Stack:
          {{ microservices_stack_info | default('Not deployed') | to_nice_yaml }}
EOF

# Test role parameterization
ansible-playbook --list-tasks test-reusable-roles.yml -i inventory.ini
```

## Verification and Discussion

### 1. Check Results
```bash
# Test generic service role with different configurations
echo "=== Testing Generic Service Role ==="
ansible-playbook --list-tasks test-reusable-roles.yml -i inventory.ini

# Check role structure
echo "=== Role Structure ==="
find roles/ -name "*.yml" | head -20

# Validate role templates
echo "=== Template Validation ==="
find roles/ -name "*.j2" | head -10
```

### 2. Discussion Points
- How do you balance flexibility with simplicity in role design?
- What parameterization patterns work best in your environment?
- How do you manage role complexity while maintaining reusability?
- What strategies do you use for role composition and inheritance?

### 3. Clean Up
```bash
# Keep roles for next exercises
# rm -rf roles/ *.yml
```

## Key Takeaways
- Advanced parameterization enables maximum role reusability
- Environment-specific configurations provide deployment flexibility
- Role composition patterns enable complex application stacks
- Template-driven configuration supports diverse use cases
- Performance profiles optimize resource usage across environments
- Validation and testing ensure role reliability across scenarios

## Next Steps
Proceed to Lab 5.4: Enterprise Role Development Patterns

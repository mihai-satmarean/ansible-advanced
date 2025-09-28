# Lab 5.4: Enterprise Role Development Patterns

## Objective
Develop enterprise-grade roles with proper testing, documentation, CI/CD integration, and governance patterns.

## Duration
30 minutes

## Prerequisites
- Completed Labs 5.1-5.3
- Understanding of testing frameworks and CI/CD concepts

## Lab Setup

```bash
cd ~/ansible-labs/module-05
mkdir -p lab-5.4
cd lab-5.4

# Create inventory for testing
cat > inventory.ini << 'EOF'
[test_servers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Role Testing with Molecule (12 minutes)

### Task: Implement comprehensive role testing using Molecule framework

Create an enterprise role with Molecule testing:

```bash
# Create enterprise application role
ansible-galaxy init roles/enterprise_application

# Initialize Molecule for the role
cd roles/enterprise_application
molecule init scenario default --driver-name docker
cd ../..
```

Update the role with enterprise patterns:

```yaml
# Update roles/enterprise_application/meta/main.yml
cat > roles/enterprise_application/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: enterprise_application
  namespace: company
  author: Enterprise Platform Team
  description: Enterprise application deployment with full lifecycle management
  company: Enterprise Corp
  license: MIT
  min_ansible_version: 2.9
  
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
    - name: EL
      versions: [8, 9]
  
  galaxy_tags:
    - enterprise
    - application
    - lifecycle
    - testing

dependencies:
  - role: company.security_baseline
    when: app_security_enabled | default(true)
  - role: company.monitoring_agent
    when: app_monitoring_enabled | default(true)

collections:
  - community.general
  - ansible.posix
  - community.docker
EOF
```

Create comprehensive role defaults:

```yaml
# Create roles/enterprise_application/defaults/main.yml
cat > roles/enterprise_application/defaults/main.yml << 'EOF'
---
# Enterprise Application Role - Default Variables

# Application identification
app_name: "enterprise-app"
app_version: "1.0.0"
app_description: "{{ app_name | title }} Enterprise Application"
app_owner: "platform-team"
app_environment: "production"

# Application lifecycle
app_lifecycle_stage: "deploy"  # deploy, update, rollback, retire
app_deployment_strategy: "blue_green"  # rolling, blue_green, canary
app_rollback_enabled: true
app_health_check_required: true

# Installation and deployment
app_install_method: "package"  # package, binary, container, source
app_package_name: "{{ app_name }}"
app_binary_url: ""
app_container_image: "{{ app_name }}:{{ app_version }}"
app_source_repo: ""

# Directory structure
app_base_dir: "/opt/{{ app_name }}"
app_config_dir: "{{ app_base_dir }}/config"
app_data_dir: "{{ app_base_dir }}/data"
app_logs_dir: "{{ app_base_dir }}/logs"
app_backup_dir: "/backup/{{ app_name }}"

# Service configuration
app_service_user: "{{ app_name }}"
app_service_group: "{{ app_name }}"
app_service_port: 8080
app_service_bind_address: "0.0.0.0"

# Security configuration
app_security_enabled: true
app_ssl_enabled: true
app_ssl_certificate: "/etc/ssl/certs/{{ app_name }}.crt"
app_ssl_private_key: "/etc/ssl/private/{{ app_name }}.key"

# Monitoring and observability
app_monitoring_enabled: true
app_metrics_enabled: true
app_metrics_port: 9090
app_logging_enabled: true
app_log_level: "INFO"

# High availability
app_ha_enabled: false
app_cluster_size: 3
app_load_balancer_enabled: false

# Backup and recovery
app_backup_enabled: true
app_backup_schedule: "0 2 * * *"
app_backup_retention_days: 30

# Performance tuning
app_performance_profile: "default"  # default, high_performance, low_resource
app_jvm_heap_size: "1g"
app_worker_processes: 2
app_connection_pool_size: 10

# Integration settings
app_database_enabled: false
app_database_url: ""
app_cache_enabled: false
app_cache_url: ""
app_message_queue_enabled: false
app_message_queue_url: ""

# Compliance and governance
app_compliance_required: true
app_audit_logging: true
app_data_classification: "internal"  # public, internal, confidential, restricted
app_retention_policy: "7_years"

# Testing configuration
app_testing_enabled: true
app_unit_tests_enabled: true
app_integration_tests_enabled: true
app_performance_tests_enabled: false
app_security_tests_enabled: true
EOF
```

Create comprehensive Molecule configuration:

```yaml
# Update roles/enterprise_application/molecule/default/molecule.yml
cat > roles/enterprise_application/molecule/default/molecule.yml << 'EOF'
---
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yml

driver:
  name: docker

platforms:
  - name: ubuntu-focal
    image: ubuntu:20.04
    pre_build_image: true
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    command: "/lib/systemd/systemd"
    tmpfs:
      - /run
      - /tmp
    capabilities:
      - SYS_ADMIN
  
  - name: ubuntu-jammy
    image: ubuntu:22.04
    pre_build_image: true
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    command: "/lib/systemd/systemd"
    tmpfs:
      - /run
      - /tmp
    capabilities:
      - SYS_ADMIN

  - name: centos-8
    image: quay.io/centos/centos:stream8
    pre_build_image: true
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    command: "/usr/sbin/init"
    tmpfs:
      - /run
      - /tmp
    capabilities:
      - SYS_ADMIN

provisioner:
  name: ansible
  config_options:
    defaults:
      callbacks_enabled: profile_tasks,timer
      stdout_callback: yaml
      bin_ansible_callbacks: true
  inventory:
    host_vars:
      ubuntu-focal:
        app_environment: "development"
        app_performance_profile: "low_resource"
      ubuntu-jammy:
        app_environment: "staging"
        app_performance_profile: "default"
      centos-8:
        app_environment: "production"
        app_performance_profile: "high_performance"

verifier:
  name: ansible

scenario:
  test_sequence:
    - dependency
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - side_effect
    - verify
    - cleanup
    - destroy
EOF
```

Create Molecule test requirements:

```yaml
# Create roles/enterprise_application/molecule/default/requirements.yml
cat > roles/enterprise_application/molecule/default/requirements.yml << 'EOF'
---
# Molecule test requirements
roles:
  - name: company.security_baseline
    src: ../../security_baseline
  - name: company.monitoring_agent
    src: ../../monitoring_agent

collections:
  - community.general
  - ansible.posix
  - community.docker
EOF
```

Create preparation playbook:

```yaml
# Create roles/enterprise_application/molecule/default/prepare.yml
cat > roles/enterprise_application/molecule/default/prepare.yml << 'EOF'
---
- name: Prepare test environment
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: Update package cache (Debian/Ubuntu)
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Update package cache (RHEL/CentOS)
      yum:
        update_cache: yes
      when: ansible_os_family == "RedHat"

    - name: Install required packages
      package:
        name:
          - curl
          - wget
          - unzip
          - systemd
        state: present

    - name: Start systemd (if not running)
      command: systemctl daemon-reload
      changed_when: false
      failed_when: false

    - name: Create test directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /opt
        - /backup
        - /var/log
EOF
```

Create comprehensive verification tests:

```yaml
# Create roles/enterprise_application/molecule/default/verify.yml
cat > roles/enterprise_application/molecule/default/verify.yml << 'EOF'
---
- name: Verify enterprise application deployment
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: Verify application user exists
      user:
        name: "{{ app_name }}"
        state: present
      check_mode: yes
      register: user_check
      failed_when: user_check.changed

    - name: Verify application directories exist
      stat:
        path: "{{ item }}"
      register: dir_check
      failed_when: not dir_check.stat.exists or not dir_check.stat.isdir
      loop:
        - "{{ app_base_dir }}"
        - "{{ app_config_dir }}"
        - "{{ app_data_dir }}"
        - "{{ app_logs_dir }}"

    - name: Verify application service is running
      service_facts:

    - name: Check service status
      assert:
        that:
          - ansible_facts.services[app_name + '.service'] is defined
          - ansible_facts.services[app_name + '.service'].state == 'running'
        fail_msg: "Application service {{ app_name }} is not running"
        success_msg: "Application service {{ app_name }} is running"

    - name: Verify application port is listening
      wait_for:
        port: "{{ app_service_port }}"
        host: "{{ app_service_bind_address }}"
        timeout: 30
      when: app_service_bind_address != "0.0.0.0"

    - name: Test application health endpoint
      uri:
        url: "http://localhost:{{ app_service_port }}/health"
        method: GET
        status_code: 200
        timeout: 10
      register: health_check
      retries: 3
      delay: 5
      when: app_health_check_required

    - name: Verify configuration files exist
      stat:
        path: "{{ app_config_dir }}/{{ app_name }}.conf"
      register: config_check
      failed_when: not config_check.stat.exists

    - name: Verify log file exists
      stat:
        path: "{{ app_logs_dir }}/{{ app_name }}.log"
      register: log_check
      when: app_logging_enabled

    - name: Check SSL certificates (if SSL enabled)
      stat:
        path: "{{ item }}"
      register: ssl_check
      failed_when: not ssl_check.stat.exists
      loop:
        - "{{ app_ssl_certificate }}"
        - "{{ app_ssl_private_key }}"
      when: app_ssl_enabled

    - name: Verify backup directory exists
      stat:
        path: "{{ app_backup_dir }}"
      register: backup_check
      failed_when: not backup_check.stat.exists or not backup_check.stat.isdir
      when: app_backup_enabled

    - name: Test application metrics endpoint
      uri:
        url: "http://localhost:{{ app_metrics_port }}/metrics"
        method: GET
        status_code: 200
        timeout: 10
      when: app_metrics_enabled
      ignore_errors: yes

    - name: Verify systemd service file
      stat:
        path: "/etc/systemd/system/{{ app_name }}.service"
      register: systemd_check
      failed_when: not systemd_check.stat.exists

    - name: Display verification summary
      debug:
        msg: |
          Verification Summary for {{ app_name }}:
          - User created: ✓
          - Directories created: ✓
          - Service running: ✓
          - Port listening: {{ '✓' if app_service_bind_address != '0.0.0.0' else 'N/A' }}
          - Health check: {{ '✓' if app_health_check_required and health_check.status == 200 else 'N/A' }}
          - Configuration: ✓
          - SSL certificates: {{ '✓' if app_ssl_enabled else 'N/A' }}
          - Backup directory: {{ '✓' if app_backup_enabled else 'N/A' }}
          - Metrics endpoint: {{ '✓' if app_metrics_enabled else 'N/A' }}
EOF
```

Create the main role tasks:

```yaml
# Create roles/enterprise_application/tasks/main.yml
cat > roles/enterprise_application/tasks/main.yml << 'EOF'
---
# Enterprise Application Role - Main Tasks

- name: Display enterprise application deployment information
  debug:
    msg: |
      Deploying Enterprise Application: {{ app_name }}
      Version: {{ app_version }}
      Environment: {{ app_environment }}
      Lifecycle Stage: {{ app_lifecycle_stage }}
      Deployment Strategy: {{ app_deployment_strategy }}
      Owner: {{ app_owner }}

- name: Validate application configuration
  assert:
    that:
      - app_name is defined and app_name != ""
      - app_version is defined and app_version != ""
      - app_environment in ['development', 'staging', 'production']
      - app_lifecycle_stage in ['deploy', 'update', 'rollback', 'retire']
    fail_msg: "Application configuration validation failed"
    success_msg: "Application configuration is valid"

- name: Include lifecycle-specific tasks
  include_tasks: "lifecycle/{{ app_lifecycle_stage }}.yml"
  tags: lifecycle

- name: Include environment-specific configuration
  include_vars: "environments/{{ app_environment }}.yml"
  when: app_environment is defined
  ignore_errors: yes

- name: Create application user and group
  group:
    name: "{{ app_service_group }}"
    state: present

- name: Create application user
  user:
    name: "{{ app_service_user }}"
    group: "{{ app_service_group }}"
    home: "{{ app_base_dir }}"
    shell: /bin/false
    system: yes
    create_home: no

- name: Create application directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ app_service_user }}"
    group: "{{ app_service_group }}"
    mode: '0755'
  loop:
    - "{{ app_base_dir }}"
    - "{{ app_config_dir }}"
    - "{{ app_data_dir }}"
    - "{{ app_logs_dir }}"
    - "{{ app_backup_dir }}"

- name: Install application
  include_tasks: "install/{{ app_install_method }}.yml"
  tags: install

- name: Configure application
  template:
    src: application.conf.j2
    dest: "{{ app_config_dir }}/{{ app_name }}.conf"
    owner: "{{ app_service_user }}"
    group: "{{ app_service_group }}"
    mode: '0644'
    backup: yes
  notify: restart application

- name: Configure SSL certificates
  include_tasks: ssl.yml
  when: app_ssl_enabled
  tags: ssl

- name: Create systemd service
  template:
    src: application.service.j2
    dest: "/etc/systemd/system/{{ app_name }}.service"
    mode: '0644'
  notify:
    - reload systemd
    - restart application

- name: Configure monitoring
  include_tasks: monitoring.yml
  when: app_monitoring_enabled
  tags: monitoring

- name: Configure backup
  include_tasks: backup.yml
  when: app_backup_enabled
  tags: backup

- name: Start and enable application service
  service:
    name: "{{ app_name }}"
    state: started
    enabled: yes

- name: Run health checks
  include_tasks: health_check.yml
  when: app_health_check_required
  tags: health_check

- name: Run compliance checks
  include_tasks: compliance.yml
  when: app_compliance_required
  tags: compliance

- name: Set application deployment facts
  set_fact:
    "{{ app_name }}_deployment_info":
      name: "{{ app_name }}"
      version: "{{ app_version }}"
      environment: "{{ app_environment }}"
      lifecycle_stage: "{{ app_lifecycle_stage }}"
      deployment_strategy: "{{ app_deployment_strategy }}"
      owner: "{{ app_owner }}"
      service_user: "{{ app_service_user }}"
      base_directory: "{{ app_base_dir }}"
      service_port: "{{ app_service_port }}"
      ssl_enabled: "{{ app_ssl_enabled }}"
      monitoring_enabled: "{{ app_monitoring_enabled }}"
      backup_enabled: "{{ app_backup_enabled }}"
      deployment_timestamp: "{{ ansible_date_time.iso8601 }}"
      compliance_status: "{{ 'compliant' if app_compliance_required else 'not_required' }}"
EOF
```

Create lifecycle management tasks:

```bash
# Create lifecycle task directories
mkdir -p roles/enterprise_application/tasks/{lifecycle,install}

# Deploy lifecycle task
cat > roles/enterprise_application/tasks/lifecycle/deploy.yml << 'EOF'
---
# Application deployment lifecycle

- name: Display deployment information
  debug:
    msg: "Performing initial deployment of {{ app_name }} version {{ app_version }}"

- name: Check if application is already deployed
  stat:
    path: "{{ app_base_dir }}/VERSION"
  register: version_file

- name: Read current version
  slurp:
    src: "{{ app_base_dir }}/VERSION"
  register: current_version_content
  when: version_file.stat.exists

- name: Set current version fact
  set_fact:
    current_app_version: "{{ current_version_content.content | b64decode | trim }}"
  when: version_file.stat.exists

- name: Display version information
  debug:
    msg: |
      Current version: {{ current_app_version | default('None (new deployment)') }}
      Target version: {{ app_version }}
      Deployment type: {{ 'Upgrade' if current_app_version is defined else 'New deployment' }}

- name: Create version file
  copy:
    content: "{{ app_version }}"
    dest: "{{ app_base_dir }}/VERSION"
    owner: "{{ app_service_user }}"
    group: "{{ app_service_group }}"
    mode: '0644'
EOF

# Update lifecycle task
cat > roles/enterprise_application/tasks/lifecycle/update.yml << 'EOF'
---
# Application update lifecycle

- name: Display update information
  debug:
    msg: "Performing update of {{ app_name }} to version {{ app_version }}"

- name: Stop application service for update
  service:
    name: "{{ app_name }}"
    state: stopped
  when: app_deployment_strategy != "blue_green"

- name: Create backup before update
  include_tasks: ../backup.yml
  when: app_backup_enabled

- name: Perform rolling update
  include_tasks: rolling_update.yml
  when: app_deployment_strategy == "rolling"

- name: Perform blue-green update
  include_tasks: blue_green_update.yml
  when: app_deployment_strategy == "blue_green"

- name: Update version file
  copy:
    content: "{{ app_version }}"
    dest: "{{ app_base_dir }}/VERSION"
    owner: "{{ app_service_user }}"
    group: "{{ app_service_group }}"
    mode: '0644'
EOF

# Rollback lifecycle task
cat > roles/enterprise_application/tasks/lifecycle/rollback.yml << 'EOF'
---
# Application rollback lifecycle

- name: Display rollback information
  debug:
    msg: "Performing rollback of {{ app_name }}"
  when: app_rollback_enabled

- name: Fail if rollback is not enabled
  fail:
    msg: "Rollback is not enabled for this application"
  when: not app_rollback_enabled

- name: Find previous version
  find:
    paths: "{{ app_backup_dir }}"
    patterns: "version-*"
    file_type: directory
  register: backup_versions

- name: Determine rollback version
  set_fact:
    rollback_version: "{{ backup_versions.files | sort(attribute='mtime', reverse=true) | first | regex_replace('.*version-(.*)$', '\\1') }}"
  when: backup_versions.files | length > 0

- name: Perform rollback
  include_tasks: restore_backup.yml
  when: rollback_version is defined
EOF
```

Create supporting task files:

```yaml
# Create roles/enterprise_application/tasks/health_check.yml
cat > roles/enterprise_application/tasks/health_check.yml << 'EOF'
---
# Application health check tasks

- name: Wait for application to start
  wait_for:
    port: "{{ app_service_port }}"
    host: "{{ app_service_bind_address }}"
    timeout: 60
    delay: 5

- name: Perform health check
  uri:
    url: "http://{{ app_service_bind_address }}:{{ app_service_port }}/health"
    method: GET
    status_code: 200
    timeout: 10
  register: health_check_result
  retries: 5
  delay: 10

- name: Display health check result
  debug:
    msg: |
      Health Check Result:
      Status: {{ health_check_result.status }}
      Response: {{ health_check_result.json | default('No JSON response') }}
      
- name: Fail if health check fails
  fail:
    msg: "Application health check failed"
  when: health_check_result.status != 200
EOF

# Create roles/enterprise_application/tasks/compliance.yml
cat > roles/enterprise_application/tasks/compliance.yml << 'EOF'
---
# Compliance and governance checks

- name: Check data classification compliance
  assert:
    that:
      - app_data_classification in ['public', 'internal', 'confidential', 'restricted']
    fail_msg: "Invalid data classification: {{ app_data_classification }}"
    success_msg: "Data classification is compliant: {{ app_data_classification }}"

- name: Verify audit logging is enabled
  assert:
    that:
      - app_audit_logging | bool
    fail_msg: "Audit logging must be enabled for compliance"
    success_msg: "Audit logging is enabled"
  when: app_data_classification in ['confidential', 'restricted']

- name: Check SSL/TLS compliance
  assert:
    that:
      - app_ssl_enabled | bool
    fail_msg: "SSL/TLS must be enabled for {{ app_data_classification }} data"
    success_msg: "SSL/TLS compliance verified"
  when: app_data_classification in ['confidential', 'restricted']

- name: Verify backup compliance
  assert:
    that:
      - app_backup_enabled | bool
    fail_msg: "Backup must be enabled for compliance"
    success_msg: "Backup compliance verified"
  when: app_data_classification in ['internal', 'confidential', 'restricted']

- name: Create compliance report
  template:
    src: compliance_report.json.j2
    dest: "{{ app_config_dir }}/compliance_report.json"
    owner: "{{ app_service_user }}"
    group: "{{ app_service_group }}"
    mode: '0644'

- name: Set compliance status
  set_fact:
    app_compliance_status: "compliant"
    app_compliance_timestamp: "{{ ansible_date_time.iso8601 }}"
EOF
```

Create templates:

```bash
# Create templates directory
mkdir -p roles/enterprise_application/templates

# Application configuration template
cat > roles/enterprise_application/templates/application.conf.j2 << 'EOF'
# {{ ansible_managed }}
# Enterprise Application Configuration for {{ app_name }}

[application]
name = {{ app_name }}
version = {{ app_version }}
description = {{ app_description }}
environment = {{ app_environment }}
owner = {{ app_owner }}

[server]
bind_address = {{ app_service_bind_address }}
port = {{ app_service_port }}
{% if app_ssl_enabled %}
ssl_enabled = true
ssl_certificate = {{ app_ssl_certificate }}
ssl_private_key = {{ app_ssl_private_key }}
{% endif %}

[logging]
enabled = {{ app_logging_enabled | lower }}
level = {{ app_log_level }}
file = {{ app_logs_dir }}/{{ app_name }}.log
audit_logging = {{ app_audit_logging | lower }}

[monitoring]
enabled = {{ app_monitoring_enabled | lower }}
{% if app_metrics_enabled %}
metrics_enabled = true
metrics_port = {{ app_metrics_port }}
{% endif %}

[performance]
profile = {{ app_performance_profile }}
{% if app_performance_profile == "high_performance" %}
jvm_heap_size = {{ app_jvm_heap_size }}
worker_processes = {{ app_worker_processes }}
connection_pool_size = {{ app_connection_pool_size }}
{% endif %}

[compliance]
data_classification = {{ app_data_classification }}
retention_policy = {{ app_retention_policy }}
compliance_required = {{ app_compliance_required | lower }}

{% if app_database_enabled %}
[database]
enabled = true
url = {{ app_database_url }}
{% endif %}

{% if app_cache_enabled %}
[cache]
enabled = true
url = {{ app_cache_url }}
{% endif %}

{% if app_message_queue_enabled %}
[message_queue]
enabled = true
url = {{ app_message_queue_url }}
{% endif %}
EOF

# Systemd service template
cat > roles/enterprise_application/templates/application.service.j2 << 'EOF'
# {{ ansible_managed }}
[Unit]
Description={{ app_description }}
After=network.target
{% if app_database_enabled %}
Requires=postgresql.service
After=postgresql.service
{% endif %}

[Service]
Type=simple
User={{ app_service_user }}
Group={{ app_service_group }}
WorkingDirectory={{ app_base_dir }}
ExecStart=/usr/bin/python3 -m http.server {{ app_service_port }}
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=10

# Environment
Environment=APP_NAME={{ app_name }}
Environment=APP_VERSION={{ app_version }}
Environment=APP_ENV={{ app_environment }}
Environment=APP_CONFIG={{ app_config_dir }}/{{ app_name }}.conf

# Resource limits
{% if app_performance_profile == "high_performance" %}
MemoryLimit=4G
CPUQuota=400%
{% elif app_performance_profile == "low_resource" %}
MemoryLimit=512M
CPUQuota=50%
{% else %}
MemoryLimit=1G
CPUQuota=100%
{% endif %}

[Install]
WantedBy=multi-user.target
EOF

# Compliance report template
cat > roles/enterprise_application/templates/compliance_report.json.j2 << 'EOF'
{
  "application": {
    "name": "{{ app_name }}",
    "version": "{{ app_version }}",
    "environment": "{{ app_environment }}",
    "owner": "{{ app_owner }}"
  },
  "compliance": {
    "status": "{{ app_compliance_status | default('pending') }}",
    "timestamp": "{{ app_compliance_timestamp | default(ansible_date_time.iso8601) }}",
    "data_classification": "{{ app_data_classification }}",
    "retention_policy": "{{ app_retention_policy }}",
    "requirements": {
      "ssl_enabled": {{ app_ssl_enabled | lower }},
      "audit_logging": {{ app_audit_logging | lower }},
      "backup_enabled": {{ app_backup_enabled | lower }},
      "monitoring_enabled": {{ app_monitoring_enabled | lower }}
    }
  },
  "security": {
    "ssl_enabled": {{ app_ssl_enabled | lower }},
    "certificate_path": "{{ app_ssl_certificate if app_ssl_enabled else 'N/A' }}"
  },
  "backup": {
    "enabled": {{ app_backup_enabled | lower }},
    "schedule": "{{ app_backup_schedule if app_backup_enabled else 'N/A' }}",
    "retention_days": {{ app_backup_retention_days if app_backup_enabled else 0 }}
  },
  "monitoring": {
    "enabled": {{ app_monitoring_enabled | lower }},
    "metrics_enabled": {{ app_metrics_enabled | lower }},
    "metrics_port": {{ app_metrics_port if app_metrics_enabled else 0 }}
  }
}
EOF
```

Create handlers:

```yaml
# Create roles/enterprise_application/handlers/main.yml
cat > roles/enterprise_application/handlers/main.yml << 'EOF'
---
# Enterprise Application Handlers

- name: reload systemd
  systemd:
    daemon_reload: yes

- name: restart application
  service:
    name: "{{ app_name }}"
    state: restarted

- name: reload application
  service:
    name: "{{ app_name }}"
    state: reloaded
EOF
```

Run Molecule tests:

```bash
# Navigate to role directory and run tests
cd roles/enterprise_application

# Run molecule tests (if Docker is available)
# molecule test

# Run syntax check
molecule syntax

# Create and converge (if Docker is available)
# molecule create
# molecule converge

# Go back to lab directory
cd ../../
```

## Exercise 2: CI/CD Integration and Automation (10 minutes)

### Task: Create CI/CD pipeline integration for role testing and deployment

Create GitHub Actions workflow:

```bash
# Create GitHub Actions directory
mkdir -p .github/workflows

# Create role testing workflow
cat > .github/workflows/role-testing.yml << 'EOF'
name: Ansible Role Testing

on:
  push:
    branches: [ main, develop ]
    paths:
      - 'roles/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'roles/**'

jobs:
  lint:
    name: Lint Ansible Roles
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          pip install ansible ansible-lint yamllint molecule[docker]

      - name: Run yamllint
        run: |
          yamllint roles/

      - name: Run ansible-lint
        run: |
          ansible-lint roles/

  molecule:
    name: Molecule Testing
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      matrix:
        role: 
          - enterprise_application
          - generic_service
          - web_api_service
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          pip install ansible molecule[docker] docker

      - name: Run Molecule tests
        run: |
          cd roles/${{ matrix.role }}
          molecule test
        env:
          MOLECULE_NO_LOG: false

  integration:
    name: Integration Testing
    runs-on: ubuntu-latest
    needs: molecule
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install Ansible
        run: |
          pip install ansible

      - name: Run integration tests
        run: |
          ansible-playbook tests/integration/test-all-roles.yml --check

  security:
    name: Security Scanning
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run security scan
        uses: securecodewarrior/github-action-add-sarif@v1
        with:
          sarif-file: 'security-scan-results.sarif'

  publish:
    name: Publish to Galaxy
    runs-on: ubuntu-latest
    needs: [molecule, integration, security]
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install ansible-galaxy
        run: |
          pip install ansible

      - name: Publish to Ansible Galaxy
        run: |
          ansible-galaxy role import --api-key ${{ secrets.GALAXY_API_KEY }} \
            ${{ github.repository_owner }} ${{ github.event.repository.name }}
        env:
          GALAXY_API_KEY: ${{ secrets.GALAXY_API_KEY }}
EOF
```

Create role quality gates:

```yaml
# Create quality gates configuration
cat > .ansible-lint << 'EOF'
---
# Ansible Lint Configuration

exclude_paths:
  - .cache/
  - .github/
  - molecule/
  - .molecule/

use_default_rules: true

rules:
  # Disable some rules for development
  line-length: disable
  truthy: disable
  
  # Enable additional rules
  yaml[line-length]: enable
  yaml[truthy]: enable

# Skip certain files
skip_list:
  - yaml[line-length]
  - name[casing]

# Warn on these rules instead of error
warn_list:
  - experimental
  - ignore-errors
  - no-handler
  - unnamed-task

# Set severity levels
severity:
  - error
  - warning
  - info
EOF

# Create yamllint configuration
cat > .yamllint << 'EOF'
---
extends: default

rules:
  line-length:
    max: 120
    level: warning
  
  truthy:
    allowed-values: ['true', 'false', 'yes', 'no']
    check-keys: false
  
  comments:
    min-spaces-from-content: 1
  
  indentation:
    spaces: 2
    indent-sequences: true
    check-multi-line-strings: false

ignore: |
  .cache/
  .github/
  molecule/
  .molecule/
EOF
```

Create role versioning and changelog:

```bash
# Create role versioning script
cat > scripts/version-role.sh << 'EOF'
#!/bin/bash
# Role versioning script

set -e

ROLE_NAME=$1
VERSION_TYPE=$2  # major, minor, patch

if [ -z "$ROLE_NAME" ] || [ -z "$VERSION_TYPE" ]; then
    echo "Usage: $0 <role_name> <version_type>"
    echo "Version types: major, minor, patch"
    exit 1
fi

ROLE_DIR="roles/$ROLE_NAME"
META_FILE="$ROLE_DIR/meta/main.yml"

if [ ! -f "$META_FILE" ]; then
    echo "Error: Role meta file not found: $META_FILE"
    exit 1
fi

# Extract current version
CURRENT_VERSION=$(grep -E "^\s*version:" "$META_FILE" | sed 's/.*version:\s*//' | tr -d '"' | tr -d "'")

if [ -z "$CURRENT_VERSION" ]; then
    echo "Warning: No version found in meta file, starting with 1.0.0"
    CURRENT_VERSION="1.0.0"
fi

# Parse version components
IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT_VERSION"

# Increment version based on type
case $VERSION_TYPE in
    major)
        MAJOR=$((MAJOR + 1))
        MINOR=0
        PATCH=0
        ;;
    minor)
        MINOR=$((MINOR + 1))
        PATCH=0
        ;;
    patch)
        PATCH=$((PATCH + 1))
        ;;
    *)
        echo "Error: Invalid version type: $VERSION_TYPE"
        exit 1
        ;;
esac

NEW_VERSION="$MAJOR.$MINOR.$PATCH"

# Update meta file
sed -i.bak "s/version:.*/version: \"$NEW_VERSION\"/" "$META_FILE"

# Update changelog
CHANGELOG_FILE="$ROLE_DIR/CHANGELOG.md"
if [ ! -f "$CHANGELOG_FILE" ]; then
    cat > "$CHANGELOG_FILE" << EOL
# Changelog

All notable changes to this role will be documented in this file.

## [Unreleased]

EOL
fi

# Add new version to changelog
DATE=$(date +%Y-%m-%d)
sed -i.bak "s/## \[Unreleased\]/## [Unreleased]\n\n## [$NEW_VERSION] - $DATE/" "$CHANGELOG_FILE"

echo "Updated $ROLE_NAME from $CURRENT_VERSION to $NEW_VERSION"
echo "Don't forget to update the changelog with your changes!"
EOF

chmod +x scripts/version-role.sh

# Create changelog template
mkdir -p roles/enterprise_application
cat > roles/enterprise_application/CHANGELOG.md << 'EOF'
# Changelog

All notable changes to the Enterprise Application role will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial enterprise application role implementation
- Comprehensive Molecule testing framework
- CI/CD pipeline integration
- Compliance and governance checks
- Multi-environment support
- Lifecycle management (deploy, update, rollback)
- Health check and monitoring integration

### Changed
- N/A

### Deprecated
- N/A

### Removed
- N/A

### Fixed
- N/A

### Security
- SSL/TLS configuration
- Audit logging implementation
- Security baseline integration

## [1.0.0] - 2024-01-15

### Added
- Initial release of enterprise application role
- Support for Ubuntu 20.04, 22.04 and CentOS 8
- Docker-based testing with Molecule
- Comprehensive documentation
- Example playbooks and configurations
EOF
```

## Exercise 3: Role Governance and Documentation (8 minutes)

### Task: Implement role governance, documentation, and approval processes

Create role governance documentation:

```markdown
# Create ROLE_GOVERNANCE.md
cat > ROLE_GOVERNANCE.md << 'EOF'
# Ansible Role Governance

## Overview

This document defines the governance model, standards, and processes for Ansible role development within the enterprise.

## Role Standards

### Naming Conventions

- **Role Names**: Use lowercase with underscores (e.g., `web_server`, `database_cluster`)
- **Variable Names**: Use descriptive names with role prefix (e.g., `webserver_port`, `db_connection_pool_size`)
- **Tag Names**: Use consistent tagging (e.g., `install`, `configure`, `service`, `security`)

### Directory Structure

All roles must follow the standard Ansible role directory structure:

```
role_name/
├── defaults/main.yml          # Default variables
├── vars/main.yml              # Role variables
├── tasks/main.yml             # Main task file
├── handlers/main.yml          # Handlers
├── templates/                 # Jinja2 templates
├── files/                     # Static files
├── meta/main.yml              # Role metadata
├── molecule/                  # Testing scenarios
├── docs/                      # Additional documentation
├── examples/                  # Usage examples
├── CHANGELOG.md               # Version history
└── README.md                  # Role documentation
```

### Quality Requirements

#### Code Quality
- All roles must pass `ansible-lint` without errors
- YAML files must pass `yamllint` validation
- Jinja2 templates must be syntactically correct
- All tasks must be idempotent

#### Testing Requirements
- **Unit Tests**: Molecule tests for all supported platforms
- **Integration Tests**: End-to-end testing with dependent roles
- **Security Tests**: Security scanning and compliance checks
- **Performance Tests**: Resource usage and performance validation

#### Documentation Requirements
- **README.md**: Comprehensive role documentation
- **CHANGELOG.md**: Version history and changes
- **Examples**: Working example playbooks
- **API Documentation**: Variable and parameter documentation

## Development Process

### 1. Planning Phase
- Create GitHub issue describing the role requirements
- Define acceptance criteria and success metrics
- Identify dependencies and integration points
- Review with architecture team

### 2. Development Phase
- Create feature branch from `develop`
- Implement role following standards
- Write comprehensive tests
- Update documentation

### 3. Review Phase
- Create pull request with detailed description
- Automated testing (CI/CD pipeline)
- Peer code review (minimum 2 approvers)
- Security review for sensitive roles
- Architecture review for complex roles

### 4. Testing Phase
- Integration testing in development environment
- User acceptance testing
- Performance and security validation
- Rollback testing

### 5. Release Phase
- Merge to `main` branch
- Tag release with semantic version
- Publish to Ansible Galaxy
- Update role registry
- Notify stakeholders

## Role Categories

### Tier 1 - Foundation Roles
- **Examples**: `security_baseline`, `monitoring_agent`, `backup_agent`
- **Requirements**: Highest quality standards, extensive testing
- **Approval**: Architecture team + Security team
- **Maintenance**: Platform team

### Tier 2 - Application Roles
- **Examples**: `web_server`, `database`, `load_balancer`
- **Requirements**: Standard quality requirements
- **Approval**: Senior developers + Platform team
- **Maintenance**: Application teams

### Tier 3 - Utility Roles
- **Examples**: `file_manager`, `log_rotator`, `certificate_manager`
- **Requirements**: Basic quality requirements
- **Approval**: Peer review
- **Maintenance**: Individual contributors

## Approval Matrix

| Role Tier | Code Review | Security Review | Architecture Review | Final Approval |
|-----------|-------------|-----------------|-------------------|----------------|
| Tier 1    | 2 Senior    | Required        | Required          | Platform Lead  |
| Tier 2    | 2 Peers     | If applicable   | If complex        | Team Lead      |
| Tier 3    | 1 Peer      | If applicable   | Not required      | Any Senior     |

## Versioning Strategy

### Semantic Versioning
- **MAJOR**: Breaking changes, incompatible API changes
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, backward compatible

### Version Lifecycle
- **Alpha**: Early development, unstable
- **Beta**: Feature complete, testing phase
- **RC**: Release candidate, final testing
- **Stable**: Production ready

### Support Policy
- **Current**: Full support, active development
- **Maintenance**: Bug fixes only, 12 months
- **End of Life**: No support, security fixes only

## Security Requirements

### Security Standards
- All roles must integrate with security baseline
- Sensitive data must use Ansible Vault
- Network communications must use encryption
- Access controls must follow least privilege principle

### Compliance Requirements
- Data classification handling
- Audit logging implementation
- Regulatory compliance (GDPR, SOX, etc.)
- Security scanning integration

### Security Review Process
- Static code analysis
- Dependency vulnerability scanning
- Configuration security assessment
- Runtime security testing

## Monitoring and Metrics

### Role Performance Metrics
- Execution time and resource usage
- Success/failure rates
- Adoption and usage statistics
- User satisfaction scores

### Quality Metrics
- Test coverage percentage
- Code quality scores
- Documentation completeness
- Issue resolution time

### Compliance Metrics
- Security scan results
- Compliance check status
- Audit trail completeness
- Policy adherence rates

## Role Registry

### Central Registry
- All approved roles must be registered
- Metadata includes version, dependencies, compatibility
- Usage examples and documentation links
- Deprecation and migration notices

### Discovery and Search
- Role catalog with search capabilities
- Dependency visualization
- Usage recommendations
- Community ratings and feedback

## Training and Support

### Developer Training
- Role development best practices
- Testing framework usage
- Security and compliance requirements
- Governance process overview

### Support Channels
- Internal documentation wiki
- Developer forums and chat
- Office hours with platform team
- Escalation procedures

## Continuous Improvement

### Regular Reviews
- Quarterly governance process review
- Annual standards update
- Role portfolio assessment
- Technology stack evaluation

### Feedback Mechanisms
- Developer surveys
- Role usage analytics
- Incident post-mortems
- Community feedback

---

**Document Version**: 1.0  
**Last Updated**: 2024-01-15  
**Next Review**: 2024-04-15  
**Owner**: Platform Engineering Team
EOF
```

Create role documentation template:

```markdown
# Create ROLE_TEMPLATE.md
cat > ROLE_TEMPLATE.md << 'EOF'
# Role Name

Brief description of what this role does and its primary purpose.

## Requirements

- Ansible >= 2.9
- Target OS: Ubuntu 20.04+, CentOS 8+
- Python >= 3.6
- Required privileges: sudo/root

## Role Variables

### Required Variables

```yaml
# Required variable with no default
required_variable: "value"
```

### Optional Variables

```yaml
# Optional variable with default value
optional_variable: "default_value"

# Complex variable structure
complex_variable:
  key1: "value1"
  key2: "value2"
  nested:
    subkey: "subvalue"
```

### Variable Descriptions

| Variable | Default | Description |
|----------|---------|-------------|
| `required_variable` | N/A | Description of required variable |
| `optional_variable` | `default_value` | Description of optional variable |

## Dependencies

List of role dependencies:

- `namespace.role_name`: Description of dependency
- `another.dependency`: Another dependency description

## Example Playbook

### Basic Usage

```yaml
- hosts: servers
  roles:
    - role: namespace.role_name
      vars:
        required_variable: "custom_value"
```

### Advanced Usage

```yaml
- hosts: servers
  roles:
    - role: namespace.role_name
      vars:
        required_variable: "custom_value"
        complex_variable:
          key1: "custom_value1"
          key2: "custom_value2"
```

## Testing

### Running Tests

```bash
# Install test dependencies
pip install molecule[docker] ansible-lint yamllint

# Run all tests
molecule test

# Run specific test scenarios
molecule test -s default
molecule test -s ubuntu
molecule test -s centos
```

### Test Scenarios

- **default**: Basic functionality test
- **ubuntu**: Ubuntu-specific testing
- **centos**: CentOS-specific testing
- **integration**: Integration with other roles

## Supported Platforms

| Platform | Version | Status |
|----------|---------|--------|
| Ubuntu | 20.04 | ✅ Supported |
| Ubuntu | 22.04 | ✅ Supported |
| CentOS | 8 | ✅ Supported |
| CentOS | 9 | 🧪 Testing |
| RHEL | 8 | ✅ Supported |

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests for new functionality
5. Run the test suite
6. Submit a pull request

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Author Information

- **Author**: Your Name
- **Email**: your.email@company.com
- **Team**: Platform Engineering
- **Maintained Since**: 2024

## Support

- **Documentation**: [Internal Wiki Link]
- **Issues**: [GitHub Issues Link]
- **Chat**: #ansible-roles Slack channel
- **Office Hours**: Tuesdays 2-3 PM EST

---

**Role Version**: 1.0.0  
**Last Updated**: 2024-01-15  
**Compatibility**: Ansible 2.9+
EOF
```

Create role approval checklist:

```markdown
# Create ROLE_APPROVAL_CHECKLIST.md
cat > ROLE_APPROVAL_CHECKLIST.md << 'EOF'
# Role Approval Checklist

Use this checklist to ensure roles meet enterprise standards before approval.

## Code Quality

### Structure and Organization
- [ ] Role follows standard directory structure
- [ ] All required files are present (meta/main.yml, README.md, etc.)
- [ ] File and directory naming follows conventions
- [ ] Code is properly organized and modular

### Ansible Best Practices
- [ ] All tasks are idempotent
- [ ] Tasks have descriptive names
- [ ] Variables are properly scoped and named
- [ ] Handlers are used appropriately
- [ ] Tags are applied consistently

### Code Standards
- [ ] Passes `ansible-lint` without errors
- [ ] Passes `yamllint` validation
- [ ] Jinja2 templates are syntactically correct
- [ ] No hardcoded values (use variables)
- [ ] Proper error handling implemented

## Testing

### Test Coverage
- [ ] Molecule tests implemented
- [ ] All supported platforms tested
- [ ] Integration tests included
- [ ] Edge cases covered
- [ ] Rollback scenarios tested

### Test Quality
- [ ] Tests are comprehensive and meaningful
- [ ] Test data is realistic
- [ ] Assertions verify expected outcomes
- [ ] Performance tests included (if applicable)
- [ ] Security tests implemented

## Documentation

### Role Documentation
- [ ] README.md is comprehensive and accurate
- [ ] All variables are documented
- [ ] Examples are provided and working
- [ ] Dependencies are clearly listed
- [ ] Platform support is documented

### Code Documentation
- [ ] Complex logic is commented
- [ ] Template variables are documented
- [ ] Task purposes are clear
- [ ] Configuration options explained

### Change Documentation
- [ ] CHANGELOG.md is updated
- [ ] Version number is incremented
- [ ] Breaking changes are highlighted
- [ ] Migration guide provided (if needed)

## Security

### Security Implementation
- [ ] Security baseline integration
- [ ] Sensitive data uses Ansible Vault
- [ ] Network communications encrypted
- [ ] Access controls implemented
- [ ] Audit logging configured

### Security Review
- [ ] Static code analysis passed
- [ ] Dependency vulnerability scan clean
- [ ] Configuration security assessed
- [ ] Runtime security tested
- [ ] Compliance requirements met

## Operational Readiness

### Monitoring and Observability
- [ ] Health checks implemented
- [ ] Monitoring integration configured
- [ ] Logging properly configured
- [ ] Metrics collection enabled
- [ ] Alerting rules defined

### Backup and Recovery
- [ ] Backup procedures implemented
- [ ] Recovery procedures tested
- [ ] Data retention policies applied
- [ ] Disaster recovery considered

### Performance
- [ ] Resource usage optimized
- [ ] Performance benchmarks established
- [ ] Scalability considerations addressed
- [ ] Capacity planning documented

## Governance

### Process Compliance
- [ ] Development process followed
- [ ] Required approvals obtained
- [ ] Security review completed (if required)
- [ ] Architecture review completed (if required)

### Role Classification
- [ ] Role tier assigned correctly
- [ ] Approval matrix followed
- [ ] Maintenance responsibility assigned
- [ ] Support model defined

### Registry and Catalog
- [ ] Role registered in catalog
- [ ] Metadata is complete and accurate
- [ ] Dependencies properly declared
- [ ] Usage examples provided

## Deployment Readiness

### Environment Compatibility
- [ ] Development environment tested
- [ ] Staging environment validated
- [ ] Production readiness assessed
- [ ] Rollback procedures verified

### Integration Testing
- [ ] Integration with existing roles tested
- [ ] Dependency compatibility verified
- [ ] End-to-end scenarios validated
- [ ] Performance impact assessed

### Release Preparation
- [ ] Release notes prepared
- [ ] Migration documentation ready
- [ ] Training materials updated
- [ ] Support procedures documented

## Final Approval

### Review Sign-offs

| Review Type | Reviewer | Date | Status |
|-------------|----------|------|--------|
| Code Review | | | ⏳ Pending |
| Security Review | | | ⏳ Pending |
| Architecture Review | | | ⏳ Pending |
| Final Approval | | | ⏳ Pending |

### Approval Criteria Met
- [ ] All checklist items completed
- [ ] Required reviews obtained
- [ ] Tests passing in CI/CD
- [ ] Documentation complete
- [ ] Security requirements satisfied

### Post-Approval Actions
- [ ] Role published to Galaxy
- [ ] Registry updated
- [ ] Team notifications sent
- [ ] Training materials updated
- [ ] Support procedures activated

---

**Checklist Version**: 1.0  
**Role Name**: _______________  
**Role Version**: _______________  
**Reviewer**: _______________  
**Date**: _______________
EOF
```

## Verification and Discussion

### 1. Check Results
```bash
# Test role structure and quality
echo "=== Role Structure Validation ==="
find roles/enterprise_application -type f | sort

# Test Molecule configuration (if available)
echo "=== Molecule Configuration ==="
cd roles/enterprise_application
molecule --version 2>/dev/null || echo "Molecule not available"
cd ../..

# Validate governance documentation
echo "=== Governance Documentation ==="
ls -la *.md

# Check CI/CD configuration
echo "=== CI/CD Configuration ==="
ls -la .github/workflows/
```

### 2. Discussion Points
- How do you implement role governance in your current environment?
- What testing strategies work best for enterprise role development?
- How do you balance automation with manual approval processes?
- What metrics do you use to measure role quality and adoption?

### 3. Clean Up
```bash
# Keep files for next exercise
# rm -rf roles/ .github/ scripts/ *.md
```

## Key Takeaways
- Comprehensive testing with Molecule ensures role reliability
- CI/CD integration automates quality gates and deployment
- Role governance provides structure and consistency
- Documentation standards improve adoption and maintenance
- Security integration is essential for enterprise environments
- Approval processes ensure quality while enabling innovation

## Next Steps
Proceed to Lab 5.5: Role Distribution and Galaxy Integration

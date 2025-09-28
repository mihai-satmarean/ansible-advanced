# Lab 6.3: Template Inheritance and Macros

## Objective
Implement template inheritance patterns and reusable macros for enterprise-scale template management and code reuse.

## Duration
25 minutes

## Prerequisites
- Completed Labs 6.1 and 6.2
- Understanding of template inheritance concepts
- Knowledge of macro programming patterns

## Lab Setup

```bash
cd ~/ansible-labs/module-06
mkdir -p lab-6.3
cd lab-6.3

# Create inventory for testing
cat > inventory.ini << 'EOF'
[web_servers]
web1 ansible_host=localhost ansible_connection=local service_type=nginx
web2 ansible_host=localhost ansible_connection=local service_type=apache

[database_servers]
db1 ansible_host=localhost ansible_connection=local service_type=postgresql

[all:vars]
ansible_python_interpreter=/usr/bin/python3
environment=production
EOF
```

## Exercise 1: Template Inheritance Patterns (10 minutes)

### Task: Create hierarchical template inheritance for configuration management

Create base template structure:

```bash
# Create template hierarchy
mkdir -p templates/{base,services,environments,components}
```

Create master base template:

```jinja2
# Create templates/base/master.j2
cat > templates/base/master.j2 << 'EOF'
{# Master Base Template - Root of inheritance hierarchy #}
{# {{ ansible_managed }} #}

{# Template metadata #}
{%- set template_info = {
    'name': template_name | default('unknown'),
    'version': template_version | default('1.0.0'),
    'generated': ansible_date_time.iso8601,
    'environment': environment | default('unknown'),
    'host': ansible_hostname,
    'inheritance_level': 'master'
} -%}

{# Configuration header block - can be overridden #}
{% block config_header %}
# =============================================================================
# {{ template_info.name | upper }} CONFIGURATION
# =============================================================================
# Template: {{ template_info.name }}
# Version: {{ template_info.version }}
# Environment: {{ template_info.environment | upper }}
# Generated: {{ template_info.generated }}
# Host: {{ template_info.host }}
# Managed by: Ansible
# =============================================================================
{% endblock config_header %}

{# Global variables block - can be extended #}
{% block global_variables %}
# Global Configuration Variables
ENVIRONMENT={{ template_info.environment | upper }}
HOSTNAME={{ ansible_hostname }}
GENERATED_DATE={{ template_info.generated }}
CONFIG_VERSION={{ template_info.version }}
{% endblock global_variables %}

{# Service-specific configuration block - must be implemented by children #}
{% block service_config %}
# Service configuration should be implemented in child templates
{% endblock service_config %}

{# Security configuration block - can be extended #}
{% block security_config %}
# Security Configuration
{% if security is defined %}
{% for key, value in security.items() %}
SECURITY_{{ key | upper }}={{ value }}
{% endfor %}
{% endif %}
{% endblock security_config %}

{# Monitoring configuration block - can be extended #}
{% block monitoring_config %}
# Monitoring Configuration
{% if monitoring is defined %}
MONITORING_ENABLED={{ monitoring.enabled | default(true) | upper }}
{% if monitoring.port is defined %}
MONITORING_PORT={{ monitoring.port }}
{% endif %}
{% if monitoring.endpoint is defined %}
MONITORING_ENDPOINT={{ monitoring.endpoint }}
{% endif %}
{% endif %}
{% endblock monitoring_config %}

{# Logging configuration block - can be extended #}
{% block logging_config %}
# Logging Configuration
{% if logging is defined %}
LOG_LEVEL={{ logging.level | default('INFO') | upper }}
LOG_FORMAT={{ logging.format | default('json') | upper }}
{% if logging.file is defined %}
LOG_FILE={{ logging.file }}
{% endif %}
{% endif %}
{% endblock logging_config %}

{# Environment-specific configuration block - can be overridden #}
{% block environment_config %}
{% if environment == 'development' %}
# Development Environment Settings
DEBUG=true
HOT_RELOAD=true
{% elif environment == 'staging' %}
# Staging Environment Settings
DEBUG=false
PERFORMANCE_TESTING=true
{% elif environment == 'production' %}
# Production Environment Settings
DEBUG=false
HIGH_AVAILABILITY=true
{% endif %}
{% endblock environment_config %}

{# Custom configuration block - for additional configurations #}
{% block custom_config %}
{# Override this block for custom configurations #}
{% endblock custom_config %}

{# Configuration footer block - can be overridden #}
{% block config_footer %}
# =============================================================================
# END OF CONFIGURATION
# Template inheritance level: {{ template_info.inheritance_level }}
# =============================================================================
{% endblock config_footer %}
EOF
```

Create service-specific base templates:

```jinja2
# Create templates/base/web-service.j2
cat > templates/base/web-service.j2 << 'EOF'
{# Web Service Base Template #}
{% extends "base/master.j2" %}

{%- set template_info = template_info | combine({
    'inheritance_level': 'web-service',
    'service_type': 'web'
}) -%}

{# Override service configuration for web services #}
{% block service_config %}
# Web Service Configuration
SERVICE_TYPE=web
SERVICE_PORT={{ service_port | default(80) }}
SERVICE_BIND_ADDRESS={{ service_bind_address | default('0.0.0.0') }}

# Web server specific settings
{% if web_server is defined %}
WEB_SERVER_SOFTWARE={{ web_server.software | default('nginx') }}
WEB_SERVER_WORKERS={{ web_server.workers | default('auto') }}
WEB_SERVER_CONNECTIONS={{ web_server.connections | default(1024) }}
{% endif %}

# SSL Configuration
{% if ssl is defined and ssl.enabled | default(false) %}
SSL_ENABLED=true
SSL_CERTIFICATE={{ ssl.certificate | default('/etc/ssl/certs/server.crt') }}
SSL_PRIVATE_KEY={{ ssl.private_key | default('/etc/ssl/private/server.key') }}
SSL_PROTOCOLS={{ ssl.protocols | default(['TLSv1.2', 'TLSv1.3']) | join(',') }}
{% else %}
SSL_ENABLED=false
{% endif %}

# Load balancing configuration
{% if load_balancer is defined %}
LOAD_BALANCER_ENABLED=true
LOAD_BALANCER_METHOD={{ load_balancer.method | default('round_robin') }}
{% if load_balancer.upstream_servers is defined %}
UPSTREAM_SERVERS={{ load_balancer.upstream_servers | join(',') }}
{% endif %}
{% endif %}
{% endblock service_config %}

{# Extend monitoring for web services #}
{% block monitoring_config %}
{{ super() }}
# Web Service Monitoring
WEB_METRICS_ENABLED={{ web_metrics_enabled | default(true) | upper }}
HEALTH_CHECK_ENDPOINT={{ health_check_endpoint | default('/health') }}
STATUS_PAGE_ENABLED={{ status_page_enabled | default(false) | upper }}
{% endblock monitoring_config %}

{# Web service specific logging #}
{% block logging_config %}
{{ super() }}
# Web Service Logging
ACCESS_LOG_ENABLED={{ access_log_enabled | default(true) | upper }}
{% if access_log_format is defined %}
ACCESS_LOG_FORMAT={{ access_log_format }}
{% endif %}
ERROR_LOG_ENABLED={{ error_log_enabled | default(true) | upper }}
{% endblock logging_config %}
EOF

# Create templates/base/database-service.j2
cat > templates/base/database-service.j2 << 'EOF'
{# Database Service Base Template #}
{% extends "base/master.j2" %}

{%- set template_info = template_info | combine({
    'inheritance_level': 'database-service',
    'service_type': 'database'
}) -%}

{# Override service configuration for database services #}
{% block service_config %}
# Database Service Configuration
SERVICE_TYPE=database
SERVICE_PORT={{ service_port | default(5432) }}
SERVICE_BIND_ADDRESS={{ service_bind_address | default('127.0.0.1') }}

# Database engine specific settings
{% if database is defined %}
DATABASE_ENGINE={{ database.engine | default('postgresql') }}
DATABASE_NAME={{ database.name | default('app_db') }}
DATABASE_USER={{ database.user | default('app_user') }}
DATABASE_PASSWORD={{ database.password | default('changeme') }}

# Connection pool settings
DATABASE_POOL_SIZE={{ database.pool_size | default(20) }}
DATABASE_POOL_TIMEOUT={{ database.pool_timeout | default(30) }}
DATABASE_MAX_CONNECTIONS={{ database.max_connections | default(100) }}

# Performance settings
{% if database.shared_buffers is defined %}
DATABASE_SHARED_BUFFERS={{ database.shared_buffers }}
{% endif %}
{% if database.effective_cache_size is defined %}
DATABASE_EFFECTIVE_CACHE_SIZE={{ database.effective_cache_size }}
{% endif %}
{% endif %}

# Backup configuration
{% if backup is defined %}
BACKUP_ENABLED={{ backup.enabled | default(true) | upper }}
BACKUP_SCHEDULE={{ backup.schedule | default('0 2 * * *') }}
BACKUP_RETENTION_DAYS={{ backup.retention_days | default(30) }}
BACKUP_COMPRESSION={{ backup.compression | default(true) | upper }}
{% endif %}

# Replication configuration
{% if replication is defined %}
REPLICATION_ENABLED={{ replication.enabled | default(false) | upper }}
{% if replication.master_host is defined %}
REPLICATION_MASTER_HOST={{ replication.master_host }}
{% endif %}
{% if replication.slave_hosts is defined %}
REPLICATION_SLAVE_HOSTS={{ replication.slave_hosts | join(',') }}
{% endif %}
{% endif %}
{% endblock service_config %}

{# Extend monitoring for database services #}
{% block monitoring_config %}
{{ super() }}
# Database Service Monitoring
DATABASE_METRICS_ENABLED={{ database_metrics_enabled | default(true) | upper }}
SLOW_QUERY_LOG_ENABLED={{ slow_query_log_enabled | default(true) | upper }}
{% if slow_query_threshold is defined %}
SLOW_QUERY_THRESHOLD={{ slow_query_threshold }}
{% endif %}
CONNECTION_MONITORING_ENABLED={{ connection_monitoring_enabled | default(true) | upper }}
{% endblock monitoring_config %}

{# Database specific security #}
{% block security_config %}
{{ super() }}
# Database Security
DATABASE_SSL_ENABLED={{ database_ssl_enabled | default(false) | upper }}
{% if database_ssl_enabled | default(false) %}
DATABASE_SSL_CERT={{ database_ssl_cert | default('/etc/ssl/certs/database.crt') }}
DATABASE_SSL_KEY={{ database_ssl_key | default('/etc/ssl/private/database.key') }}
{% endif %}
PASSWORD_ENCRYPTION={{ password_encryption | default('scram-sha-256') }}
{% endblock security_config %}
EOF
```

Create specific service templates:

```jinja2
# Create templates/services/nginx.j2
cat > templates/services/nginx.j2 << 'EOF'
{# Nginx Service Template #}
{% extends "base/web-service.j2" %}

{%- set template_info = template_info | combine({
    'inheritance_level': 'nginx-service',
    'service_name': 'nginx'
}) -%}

{# Nginx-specific configuration #}
{% block service_config %}
{{ super() }}

# Nginx Specific Configuration
NGINX_VERSION={{ nginx_version | default('latest') }}
NGINX_USER={{ nginx_user | default('www-data') }}
NGINX_WORKER_PROCESSES={{ nginx_worker_processes | default('auto') }}
NGINX_WORKER_CONNECTIONS={{ nginx_worker_connections | default(1024) }}
NGINX_KEEPALIVE_TIMEOUT={{ nginx_keepalive_timeout | default(65) }}
NGINX_CLIENT_MAX_BODY_SIZE={{ nginx_client_max_body_size | default('64m') }}

# Nginx modules
{% if nginx_modules is defined %}
NGINX_MODULES={{ nginx_modules | join(',') }}
{% endif %}

# Gzip configuration
NGINX_GZIP_ENABLED={{ nginx_gzip_enabled | default(true) | upper }}
{% if nginx_gzip_enabled | default(true) %}
NGINX_GZIP_LEVEL={{ nginx_gzip_level | default(6) }}
NGINX_GZIP_TYPES={{ nginx_gzip_types | default(['text/plain', 'text/css', 'application/json']) | join(',') }}
{% endif %}

# Rate limiting
{% if nginx_rate_limiting is defined %}
NGINX_RATE_LIMITING_ENABLED=true
NGINX_RATE_LIMIT={{ nginx_rate_limiting.limit | default('10r/s') }}
NGINX_RATE_LIMIT_BURST={{ nginx_rate_limiting.burst | default(20) }}
{% endif %}
{% endblock service_config %}

{# Nginx-specific custom configuration #}
{% block custom_config %}
# Nginx Virtual Hosts
{% if nginx_vhosts is defined %}
{% for vhost in nginx_vhosts %}
# Virtual Host: {{ vhost.server_name }}
VHOST_{{ loop.index }}_SERVER_NAME={{ vhost.server_name }}
VHOST_{{ loop.index }}_DOCUMENT_ROOT={{ vhost.document_root | default('/var/www/html') }}
{% if vhost.ssl_enabled | default(false) %}
VHOST_{{ loop.index }}_SSL_ENABLED=true
{% endif %}
{% if vhost.proxy_pass is defined %}
VHOST_{{ loop.index }}_PROXY_PASS={{ vhost.proxy_pass }}
{% endif %}
{% endfor %}
{% endif %}

# Nginx upstream servers
{% if nginx_upstreams is defined %}
{% for upstream in nginx_upstreams %}
# Upstream: {{ upstream.name }}
UPSTREAM_{{ upstream.name | upper }}_SERVERS={{ upstream.servers | join(',') }}
UPSTREAM_{{ upstream.name | upper }}_METHOD={{ upstream.method | default('round_robin') }}
{% endfor %}
{% endif %}
{% endblock custom_config %}
EOF

# Create templates/services/postgresql.j2
cat > templates/services/postgresql.j2 << 'EOF'
{# PostgreSQL Service Template #}
{% extends "base/database-service.j2" %}

{%- set template_info = template_info | combine({
    'inheritance_level': 'postgresql-service',
    'service_name': 'postgresql'
}) -%}

{# PostgreSQL-specific configuration #}
{% block service_config %}
{{ super() }}

# PostgreSQL Specific Configuration
POSTGRESQL_VERSION={{ postgresql_version | default('13') }}
POSTGRESQL_DATA_DIR={{ postgresql_data_dir | default('/var/lib/postgresql/data') }}
POSTGRESQL_CONFIG_DIR={{ postgresql_config_dir | default('/etc/postgresql') }}

# PostgreSQL performance tuning
POSTGRESQL_SHARED_BUFFERS={{ postgresql_shared_buffers | default('256MB') }}
POSTGRESQL_EFFECTIVE_CACHE_SIZE={{ postgresql_effective_cache_size | default('1GB') }}
POSTGRESQL_WORK_MEM={{ postgresql_work_mem | default('4MB') }}
POSTGRESQL_MAINTENANCE_WORK_MEM={{ postgresql_maintenance_work_mem | default('64MB') }}
POSTGRESQL_CHECKPOINT_COMPLETION_TARGET={{ postgresql_checkpoint_completion_target | default(0.9) }}
POSTGRESQL_WAL_BUFFERS={{ postgresql_wal_buffers | default('16MB') }}
POSTGRESQL_DEFAULT_STATISTICS_TARGET={{ postgresql_default_statistics_target | default(100) }}

# PostgreSQL connection settings
POSTGRESQL_MAX_CONNECTIONS={{ postgresql_max_connections | default(200) }}
POSTGRESQL_SUPERUSER_RESERVED_CONNECTIONS={{ postgresql_superuser_reserved_connections | default(3) }}

# PostgreSQL logging
POSTGRESQL_LOG_DESTINATION={{ postgresql_log_destination | default('stderr') }}
POSTGRESQL_LOG_COLLECTOR={{ postgresql_log_collector | default('on') }}
POSTGRESQL_LOG_DIRECTORY={{ postgresql_log_directory | default('log') }}
POSTGRESQL_LOG_FILENAME={{ postgresql_log_filename | default('postgresql-%Y-%m-%d_%H%M%S.log') }}
POSTGRESQL_LOG_ROTATION_AGE={{ postgresql_log_rotation_age | default('1d') }}
POSTGRESQL_LOG_ROTATION_SIZE={{ postgresql_log_rotation_size | default('10MB') }}
{% endblock service_config %}

{# PostgreSQL-specific custom configuration #}
{% block custom_config %}
# PostgreSQL Extensions
{% if postgresql_extensions is defined %}
POSTGRESQL_EXTENSIONS={{ postgresql_extensions | join(',') }}
{% endif %}

# PostgreSQL databases
{% if postgresql_databases is defined %}
{% for db in postgresql_databases %}
# Database: {{ db.name }}
DATABASE_{{ loop.index }}_NAME={{ db.name }}
DATABASE_{{ loop.index }}_OWNER={{ db.owner | default('postgres') }}
DATABASE_{{ loop.index }}_ENCODING={{ db.encoding | default('UTF8') }}
DATABASE_{{ loop.index }}_LOCALE={{ db.locale | default('en_US.UTF-8') }}
{% endfor %}
{% endif %}

# PostgreSQL users
{% if postgresql_users is defined %}
{% for user in postgresql_users %}
# User: {{ user.name }}
USER_{{ loop.index }}_NAME={{ user.name }}
USER_{{ loop.index }}_PASSWORD={{ user.password }}
USER_{{ loop.index }}_PRIVILEGES={{ user.privileges | default([]) | join(',') }}
{% if user.databases is defined %}
USER_{{ loop.index }}_DATABASES={{ user.databases | join(',') }}
{% endif %}
{% endfor %}
{% endif %}
{% endblock custom_config %}
EOF
```

Create environment-specific templates:

```jinja2
# Create templates/environments/production.j2
cat > templates/environments/production.j2 << 'EOF'
{# Production Environment Template #}
{% extends template_base | default("base/master.j2") %}

{%- set template_info = template_info | combine({
    'inheritance_level': template_info.inheritance_level + '-production',
    'environment': 'production'
}) -%}

{# Override environment configuration for production #}
{% block environment_config %}
# Production Environment Configuration
ENVIRONMENT=PRODUCTION
DEBUG=false
HIGH_AVAILABILITY=true
DISASTER_RECOVERY=true
COMPLIANCE_LOGGING=true
AUDIT_TRAIL=true

# Production performance settings
PERFORMANCE_OPTIMIZATION=true
CACHING_ENABLED=true
CDN_ENABLED=true
COMPRESSION_ENABLED=true

# Production scaling
AUTO_SCALING=true
MIN_INSTANCES={{ min_instances | default(3) }}
MAX_INSTANCES={{ max_instances | default(50) }}
SCALE_UP_THRESHOLD={{ scale_up_threshold | default(70) }}
SCALE_DOWN_THRESHOLD={{ scale_down_threshold | default(30) }}
{% endblock environment_config %}

{# Override security for production #}
{% block security_config %}
{{ super() }}
# Production Security Hardening
SECURITY_HARDENING=true
SSL_ONLY=true
HSTS_ENABLED=true
CSRF_PROTECTION=true
XSS_PROTECTION=true
CLICKJACKING_PROTECTION=true
CONTENT_TYPE_NOSNIFF=true

# Production authentication
STRONG_AUTHENTICATION=true
MFA_REQUIRED=true
SESSION_TIMEOUT={{ session_timeout | default(1800) }}
PASSWORD_COMPLEXITY=high
{% endblock security_config %}

{# Override monitoring for production #}
{% block monitoring_config %}
{{ super() }}
# Production Monitoring
REAL_TIME_MONITORING=true
ALERTING_ENABLED=true
SLA_MONITORING=true
BUSINESS_METRICS=true
PERFORMANCE_MONITORING=true
SECURITY_MONITORING=true

# Production alerting
ALERT_CHANNELS={{ alert_channels | default(['email', 'slack', 'pagerduty']) | join(',') }}
CRITICAL_ALERT_THRESHOLD={{ critical_alert_threshold | default(95) }}
WARNING_ALERT_THRESHOLD={{ warning_alert_threshold | default(80) }}
{% endblock monitoring_config %}
EOF
```

Create test for template inheritance:

```yaml
# Create test-inheritance.yml
cat > test-inheritance.yml << 'EOF'
---
- name: Test template inheritance patterns
  hosts: all
  gather_facts: yes
  vars:
    template_name: "{{ service_type }}-config"
    template_version: "2.1.0"
    
    # Service-specific variables
    nginx_worker_processes: 4
    nginx_worker_connections: 2048
    nginx_gzip_enabled: true
    nginx_vhosts:
      - server_name: "app.example.com"
        document_root: "/var/www/app"
        ssl_enabled: true
        proxy_pass: "http://backend"
    
    postgresql_version: "13"
    postgresql_max_connections: 200
    postgresql_shared_buffers: "512MB"
    postgresql_databases:
      - name: "app_db"
        owner: "app_user"
        encoding: "UTF8"
    
    # Common variables
    monitoring:
      enabled: true
      port: 9090
      endpoint: "/metrics"
    
    logging:
      level: "INFO"
      format: "json"
      file: "/var/log/{{ service_type }}/app.log"
    
    security:
      ssl_enabled: true
      encryption: "AES-256"
      authentication: "strong"
  
  tasks:
    - name: Generate service-specific configuration using inheritance
      template:
        src: "services/{{ service_type }}.j2"
        dest: "/tmp/{{ service_type }}-config-{{ environment }}.conf"
        mode: '0644'
      when: service_type in ['nginx', 'postgresql']
    
    - name: Generate production environment configuration
      template:
        src: "environments/production.j2"
        dest: "/tmp/production-{{ service_type }}-config.conf"
        mode: '0644'
      vars:
        template_base: "services/{{ service_type }}.j2"
      when: service_type in ['nginx', 'postgresql']
    
    - name: Display inheritance chain information
      debug:
        msg: |
          Template Inheritance Chain for {{ service_type | upper }}:
          1. Master Base Template (base/master.j2)
          2. Service Base Template (base/{{ 'web-service' if service_type == 'nginx' else 'database-service' }}.j2)
          3. Specific Service Template (services/{{ service_type }}.j2)
          4. Environment Override (environments/{{ environment }}.j2)
          
          Generated Files:
          - Service Config: /tmp/{{ service_type }}-config-{{ environment }}.conf
          - Production Config: /tmp/production-{{ service_type }}-config.conf
EOF

# Run inheritance test
ansible-playbook -i inventory.ini test-inheritance.yml
```

## Exercise 2: Reusable Macros and Functions (8 minutes)

### Task: Create reusable macros for common configuration patterns

Create macro library:

```jinja2
# Create templates/macros/common.j2
cat > templates/macros/common.j2 << 'EOF'
{# Common Macros Library #}

{# Macro for generating configuration section headers #}
{% macro section_header(title, level=1, width=80) -%}
{% set char = '#' if level == 1 else '=' if level == 2 else '-' %}
{% set padding = (width - title|length - 2) // 2 %}
{{ char * width }}
{{ char }}{{ ' ' * padding }}{{ title | upper }}{{ ' ' * (width - title|length - padding - 2) }}{{ char }}
{{ char * width }}
{%- endmacro %}

{# Macro for generating key-value pairs with validation #}
{% macro config_pair(key, value, required=false, default=none, comment=none) -%}
{% if required and value is not defined and default is none %}
# ERROR: {{ key }} is required but not defined
{% elif value is defined %}
{% if comment %}# {{ comment }}{% endif %}
{{ key | upper }}={{ value }}
{% elif default is not none %}
{% if comment %}# {{ comment }} (default){% endif %}
{{ key | upper }}={{ default }}
{% else %}
# {{ key | upper }}=<not_set>
{% endif %}
{%- endmacro %}

{# Macro for generating boolean configuration #}
{% macro bool_config(key, value, true_val='true', false_val='false', comment=none) -%}
{% if comment %}# {{ comment }}{% endif %}
{{ key | upper }}={{ true_val if value else false_val }}
{%- endmacro %}

{# Macro for generating list configuration #}
{% macro list_config(key, items, separator=',', comment=none) -%}
{% if items is defined and items|length > 0 %}
{% if comment %}# {{ comment }}{% endif %}
{{ key | upper }}={{ items | join(separator) }}
{% else %}
# {{ key | upper }}=<empty_list>
{% endif %}
{%- endmacro %}

{# Macro for generating database connection string #}
{% macro db_connection_string(engine, host, port, database, username, password, ssl=false) -%}
{% if engine == 'postgresql' %}
postgresql://{{ username }}:{{ password }}@{{ host }}:{{ port }}/{{ database }}{% if ssl %}?sslmode=require{% endif %}
{% elif engine == 'mysql' %}
mysql://{{ username }}:{{ password }}@{{ host }}:{{ port }}/{{ database }}{% if ssl %}?ssl=true{% endif %}
{% elif engine == 'sqlite' %}
sqlite:///{{ database }}
{% else %}
# Unsupported database engine: {{ engine }}
{% endif %}
{%- endmacro %}

{# Macro for generating SSL configuration block #}
{% macro ssl_config_block(enabled, certificate=none, private_key=none, protocols=none, ciphers=none) -%}
{{ section_header('SSL Configuration', 2) }}
{{ bool_config('ssl_enabled', enabled, comment='Enable/disable SSL') }}
{% if enabled %}
{{ config_pair('ssl_certificate', certificate, required=true, comment='SSL certificate file path') }}
{{ config_pair('ssl_private_key', private_key, required=true, comment='SSL private key file path') }}
{{ list_config('ssl_protocols', protocols, comment='Supported SSL/TLS protocols') }}
{{ config_pair('ssl_ciphers', ciphers, comment='SSL cipher suite') }}
{% endif %}
{%- endmacro %}

{# Macro for generating monitoring configuration block #}
{% macro monitoring_config_block(enabled, port=none, endpoint=none, metrics=none, alerts=none) -%}
{{ section_header('Monitoring Configuration', 2) }}
{{ bool_config('monitoring_enabled', enabled, comment='Enable/disable monitoring') }}
{% if enabled %}
{{ config_pair('monitoring_port', port, default=9090, comment='Monitoring service port') }}
{{ config_pair('monitoring_endpoint', endpoint, default='/metrics', comment='Metrics endpoint path') }}
{% if metrics is defined %}
{% for metric, config in metrics.items() %}
{{ config_pair('metric_' + metric + '_enabled', config.enabled, default=true) }}
{% if config.interval is defined %}
{{ config_pair('metric_' + metric + '_interval', config.interval) }}
{% endif %}
{% endfor %}
{% endif %}
{% if alerts is defined %}
{{ bool_config('alerts_enabled', alerts.enabled, default=false) }}
{% if alerts.enabled %}
{{ list_config('alert_channels', alerts.channels, comment='Alert notification channels') }}
{{ config_pair('alert_threshold_critical', alerts.thresholds.critical, default=90) }}
{{ config_pair('alert_threshold_warning', alerts.thresholds.warning, default=75) }}
{% endif %}
{% endif %}
{% endif %}
{%- endmacro %}

{# Macro for generating logging configuration block #}
{% macro logging_config_block(level, format=none, file=none, rotation=none, syslog=none) -%}
{{ section_header('Logging Configuration', 2) }}
{{ config_pair('log_level', level, required=true, comment='Logging level (DEBUG, INFO, WARN, ERROR)') }}
{{ config_pair('log_format', format, default='json', comment='Log format (json, text)') }}
{% if file is defined %}
{{ config_pair('log_file', file.path, comment='Log file path') }}
{{ config_pair('log_file_max_size', file.max_size, default='100MB', comment='Maximum log file size') }}
{{ config_pair('log_file_max_count', file.max_count, default=10, comment='Maximum number of log files') }}
{% endif %}
{% if rotation is defined %}
{{ bool_config('log_rotation_enabled', rotation.enabled, default=true) }}
{{ config_pair('log_rotation_schedule', rotation.schedule, default='daily') }}
{% endif %}
{% if syslog is defined %}
{{ bool_config('syslog_enabled', syslog.enabled, default=false) }}
{% if syslog.enabled %}
{{ config_pair('syslog_facility', syslog.facility, default='local0') }}
{{ config_pair('syslog_tag', syslog.tag, default=ansible_hostname) }}
{% endif %}
{% endif %}
{%- endmacro %}

{# Macro for generating performance tuning block #}
{% macro performance_config_block(profile, custom_settings=none) -%}
{{ section_header('Performance Configuration', 2) }}
{{ config_pair('performance_profile', profile, default='default', comment='Performance profile (low, default, high)') }}

{% if profile == 'low' %}
# Low resource profile
WORKER_PROCESSES=1
WORKER_CONNECTIONS=512
MEMORY_LIMIT=256MB
CACHE_SIZE=64MB
{% elif profile == 'high' %}
# High performance profile
WORKER_PROCESSES=auto
WORKER_CONNECTIONS=4096
MEMORY_LIMIT=2GB
CACHE_SIZE=512MB
{% else %}
# Default performance profile
WORKER_PROCESSES=2
WORKER_CONNECTIONS=1024
MEMORY_LIMIT=1GB
CACHE_SIZE=256MB
{% endif %}

{% if custom_settings is defined %}
# Custom performance settings
{% for key, value in custom_settings.items() %}
{{ config_pair(key, value) }}
{% endfor %}
{% endif %}
{%- endmacro %}

{# Macro for generating backup configuration block #}
{% macro backup_config_block(enabled, schedule=none, retention=none, compression=none, encryption=none) -%}
{{ section_header('Backup Configuration', 2) }}
{{ bool_config('backup_enabled', enabled, comment='Enable/disable backups') }}
{% if enabled %}
{{ config_pair('backup_schedule', schedule, default='0 2 * * *', comment='Backup schedule (cron format)') }}
{{ config_pair('backup_retention_days', retention, default=30, comment='Backup retention period in days') }}
{{ bool_config('backup_compression', compression, default=true, comment='Enable backup compression') }}
{{ bool_config('backup_encryption', encryption, default=false, comment='Enable backup encryption') }}
{% endif %}
{%- endmacro %}

{# Macro for generating environment-specific overrides #}
{% macro environment_overrides(environment, overrides) -%}
{{ section_header('Environment Overrides: ' + environment | upper, 2) }}
{% if overrides is defined %}
{% for key, value in overrides.items() %}
{{ config_pair(key, value, comment='Environment-specific override') }}
{% endfor %}
{% else %}
# No environment-specific overrides defined
{% endif %}
{%- endmacro %}
EOF
```

Create service-specific macro libraries:

```jinja2
# Create templates/macros/web.j2
cat > templates/macros/web.j2 << 'EOF'
{# Web Service Macros #}

{# Import common macros #}
{% from 'macros/common.j2' import section_header, config_pair, bool_config, list_config %}

{# Macro for generating virtual host configuration #}
{% macro virtual_host_config(vhost, index) -%}
{{ section_header('Virtual Host: ' + vhost.server_name, 3) }}
VHOST_{{ index }}_SERVER_NAME={{ vhost.server_name }}
VHOST_{{ index }}_DOCUMENT_ROOT={{ vhost.document_root | default('/var/www/html') }}
{{ bool_config('vhost_' + index|string + '_ssl_enabled', vhost.ssl_enabled | default(false)) }}
{% if vhost.ssl_enabled | default(false) %}
VHOST_{{ index }}_SSL_CERTIFICATE={{ vhost.ssl_certificate | default('/etc/ssl/certs/' + vhost.server_name + '.crt') }}
VHOST_{{ index }}_SSL_PRIVATE_KEY={{ vhost.ssl_private_key | default('/etc/ssl/private/' + vhost.server_name + '.key') }}
{% endif %}
{% if vhost.proxy_pass is defined %}
VHOST_{{ index }}_PROXY_PASS={{ vhost.proxy_pass }}
{% endif %}
{% if vhost.aliases is defined %}
{{ list_config('vhost_' + index|string + '_aliases', vhost.aliases) }}
{% endif %}
{% if vhost.custom_config is defined %}
{% for key, value in vhost.custom_config.items() %}
VHOST_{{ index }}_{{ key | upper }}={{ value }}
{% endfor %}
{% endif %}
{%- endmacro %}

{# Macro for generating upstream server configuration #}
{% macro upstream_config(upstream, index) -%}
{{ section_header('Upstream: ' + upstream.name, 3) }}
UPSTREAM_{{ index }}_NAME={{ upstream.name }}
{{ list_config('upstream_' + index|string + '_servers', upstream.servers) }}
{{ config_pair('upstream_' + index|string + '_method', upstream.method, default='round_robin') }}
{{ config_pair('upstream_' + index|string + '_max_fails', upstream.max_fails, default=3) }}
{{ config_pair('upstream_' + index|string + '_fail_timeout', upstream.fail_timeout, default='30s') }}
{% if upstream.health_check is defined %}
{{ bool_config('upstream_' + index|string + '_health_check', upstream.health_check.enabled, default=false) }}
{% if upstream.health_check.enabled %}
{{ config_pair('upstream_' + index|string + '_health_check_uri', upstream.health_check.uri, default='/health') }}
{{ config_pair('upstream_' + index|string + '_health_check_interval', upstream.health_check.interval, default='5s') }}
{% endif %}
{% endif %}
{%- endmacro %}

{# Macro for generating rate limiting configuration #}
{% macro rate_limiting_config(zones) -%}
{{ section_header('Rate Limiting Configuration', 3) }}
{% if zones is defined %}
{% for zone in zones %}
RATE_LIMIT_ZONE_{{ loop.index }}_NAME={{ zone.name }}
RATE_LIMIT_ZONE_{{ loop.index }}_KEY={{ zone.key | default('$binary_remote_addr') }}
RATE_LIMIT_ZONE_{{ loop.index }}_SIZE={{ zone.size | default('10m') }}
RATE_LIMIT_ZONE_{{ loop.index }}_RATE={{ zone.rate | default('10r/s') }}
RATE_LIMIT_ZONE_{{ loop.index }}_BURST={{ zone.burst | default(20) }}
{% endfor %}
{% else %}
# No rate limiting zones configured
{% endif %}
{%- endmacro %}

{# Macro for generating cache configuration #}
{% macro cache_config(cache_zones) -%}
{{ section_header('Cache Configuration', 3) }}
{% if cache_zones is defined %}
{% for zone in cache_zones %}
CACHE_ZONE_{{ loop.index }}_NAME={{ zone.name }}
CACHE_ZONE_{{ loop.index }}_PATH={{ zone.path }}
CACHE_ZONE_{{ loop.index }}_LEVELS={{ zone.levels | default('1:2') }}
CACHE_ZONE_{{ loop.index }}_KEYS_ZONE={{ zone.keys_zone | default(zone.name + ':10m') }}
CACHE_ZONE_{{ loop.index }}_MAX_SIZE={{ zone.max_size | default('1g') }}
CACHE_ZONE_{{ loop.index }}_INACTIVE={{ zone.inactive | default('60m') }}
{% endfor %}
{% else %}
# No cache zones configured
{% endif %}
{%- endmacro %}
EOF

# Create templates/macros/database.j2
cat > templates/macros/database.j2 << 'EOF'
{# Database Service Macros #}

{# Import common macros #}
{% from 'macros/common.j2' import section_header, config_pair, bool_config, list_config %}

{# Macro for generating database connection pool configuration #}
{% macro connection_pool_config(pool_config) -%}
{{ section_header('Connection Pool Configuration', 3) }}
{{ config_pair('pool_size', pool_config.size, default=20, comment='Maximum number of connections in pool') }}
{{ config_pair('pool_min_size', pool_config.min_size, default=5, comment='Minimum number of connections in pool') }}
{{ config_pair('pool_timeout', pool_config.timeout, default=30, comment='Connection timeout in seconds') }}
{{ config_pair('pool_max_overflow', pool_config.max_overflow, default=10, comment='Maximum overflow connections') }}
{{ config_pair('pool_recycle', pool_config.recycle, default=3600, comment='Connection recycle time in seconds') }}
{{ bool_config('pool_pre_ping', pool_config.pre_ping, default=true, comment='Enable connection pre-ping') }}
{%- endmacro %}

{# Macro for generating database user configuration #}
{% macro database_user_config(user, index) -%}
{{ section_header('Database User: ' + user.name, 3) }}
DATABASE_USER_{{ index }}_NAME={{ user.name }}
DATABASE_USER_{{ index }}_PASSWORD={{ user.password }}
{% if user.databases is defined %}
{{ list_config('database_user_' + index|string + '_databases', user.databases) }}
{% endif %}
{% if user.privileges is defined %}
{{ list_config('database_user_' + index|string + '_privileges', user.privileges) }}
{% endif %}
{{ bool_config('database_user_' + index|string + '_superuser', user.superuser | default(false)) }}
{{ bool_config('database_user_' + index|string + '_createdb', user.createdb | default(false)) }}
{{ bool_config('database_user_' + index|string + '_createrole', user.createrole | default(false)) }}
{% if user.valid_until is defined %}
DATABASE_USER_{{ index }}_VALID_UNTIL={{ user.valid_until }}
{% endif %}
{%- endmacro %}

{# Macro for generating database configuration #}
{% macro database_config(database, index) -%}
{{ section_header('Database: ' + database.name, 3) }}
DATABASE_{{ index }}_NAME={{ database.name }}
DATABASE_{{ index }}_OWNER={{ database.owner | default('postgres') }}
DATABASE_{{ index }}_ENCODING={{ database.encoding | default('UTF8') }}
DATABASE_{{ index }}_LOCALE={{ database.locale | default('en_US.UTF-8') }}
DATABASE_{{ index }}_COLLATE={{ database.collate | default(database.locale | default('en_US.UTF-8')) }}
DATABASE_{{ index }}_CTYPE={{ database.ctype | default(database.locale | default('en_US.UTF-8')) }}
{% if database.template is defined %}
DATABASE_{{ index }}_TEMPLATE={{ database.template }}
{% endif %}
{% if database.tablespace is defined %}
DATABASE_{{ index }}_TABLESPACE={{ database.tablespace }}
{% endif %}
{%- endmacro %}

{# Macro for generating replication configuration #}
{% macro replication_config(replication) -%}
{{ section_header('Replication Configuration', 3) }}
{{ bool_config('replication_enabled', replication.enabled, default=false) }}
{% if replication.enabled %}
{{ config_pair('replication_user', replication.user, default='replicator') }}
{{ config_pair('replication_password', replication.password, required=true) }}
{% if replication.master_host is defined %}
{{ config_pair('replication_master_host', replication.master_host) }}
{{ config_pair('replication_master_port', replication.master_port, default=5432) }}
{% endif %}
{% if replication.slave_hosts is defined %}
{{ list_config('replication_slave_hosts', replication.slave_hosts) }}
{% endif %}
{{ config_pair('replication_method', replication.method, default='streaming') }}
{{ config_pair('replication_slot_name', replication.slot_name, default='replication_slot') }}
{{ bool_config('replication_synchronous', replication.synchronous, default=false) }}
{% endif %}
{%- endmacro %}

{# Macro for generating backup configuration #}
{% macro database_backup_config(backup) -%}
{{ section_header('Database Backup Configuration', 3) }}
{{ bool_config('backup_enabled', backup.enabled, default=true) }}
{% if backup.enabled %}
{{ config_pair('backup_method', backup.method, default='pg_dump') }}
{{ config_pair('backup_schedule', backup.schedule, default='0 2 * * *') }}
{{ config_pair('backup_retention_days', backup.retention_days, default=30) }}
{{ config_pair('backup_directory', backup.directory, default='/backup/postgresql') }}
{{ bool_config('backup_compression', backup.compression, default=true) }}
{{ bool_config('backup_encryption', backup.encryption, default=false) }}
{% if backup.encryption %}
{{ config_pair('backup_encryption_key', backup.encryption_key, required=true) }}
{% endif %}
{{ config_pair('backup_format', backup.format, default='custom') }}
{% if backup.parallel_jobs is defined %}
{{ config_pair('backup_parallel_jobs', backup.parallel_jobs) }}
{% endif %}
{% endif %}
{%- endmacro %}
EOF
```

Create template using macros:

```jinja2
# Create templates/services/nginx-with-macros.j2
cat > templates/services/nginx-with-macros.j2 << 'EOF'
{# Nginx Configuration Template Using Macros #}
{# Import macro libraries #}
{% from 'macros/common.j2' import section_header, config_pair, bool_config, list_config, ssl_config_block, monitoring_config_block, logging_config_block, performance_config_block %}
{% from 'macros/web.j2' import virtual_host_config, upstream_config, rate_limiting_config, cache_config %}

# {{ ansible_managed }}
{{ section_header('Nginx Configuration', 1) }}

# Basic Information
{{ config_pair('service_name', 'nginx') }}
{{ config_pair('service_version', nginx_version, default='latest') }}
{{ config_pair('generated_date', ansible_date_time.iso8601) }}
{{ config_pair('hostname', ansible_hostname) }}
{{ config_pair('environment', environment | upper) }}

# Basic Nginx Configuration
{{ section_header('Basic Configuration', 2) }}
{{ config_pair('nginx_user', nginx_user, default='www-data') }}
{{ config_pair('nginx_worker_processes', nginx_worker_processes, default='auto') }}
{{ config_pair('nginx_worker_connections', nginx_worker_connections, default=1024) }}
{{ config_pair('nginx_keepalive_timeout', nginx_keepalive_timeout, default=65) }}
{{ config_pair('nginx_client_max_body_size', nginx_client_max_body_size, default='64m') }}

# Gzip Configuration
{{ section_header('Gzip Configuration', 2) }}
{{ bool_config('gzip_enabled', nginx_gzip_enabled, default=true) }}
{% if nginx_gzip_enabled | default(true) %}
{{ config_pair('gzip_level', nginx_gzip_level, default=6) }}
{{ list_config('gzip_types', nginx_gzip_types, default=['text/plain', 'text/css', 'application/json']) }}
{% endif %}

# SSL Configuration
{% if ssl is defined %}
{{ ssl_config_block(ssl.enabled, ssl.certificate, ssl.private_key, ssl.protocols, ssl.ciphers) }}
{% endif %}

# Performance Configuration
{{ performance_config_block(performance_profile | default('default'), performance_custom_settings) }}

# Monitoring Configuration
{% if monitoring is defined %}
{{ monitoring_config_block(monitoring.enabled, monitoring.port, monitoring.endpoint, monitoring.metrics, monitoring.alerts) }}
{% endif %}

# Logging Configuration
{% if logging is defined %}
{{ logging_config_block(logging.level, logging.format, logging.file, logging.rotation, logging.syslog) }}
{% endif %}

# Virtual Hosts Configuration
{% if nginx_vhosts is defined %}
{{ section_header('Virtual Hosts', 2) }}
{% for vhost in nginx_vhosts %}
{{ virtual_host_config(vhost, loop.index) }}
{% endfor %}
{% endif %}

# Upstream Servers Configuration
{% if nginx_upstreams is defined %}
{{ section_header('Upstream Servers', 2) }}
{% for upstream in nginx_upstreams %}
{{ upstream_config(upstream, loop.index) }}
{% endfor %}
{% endif %}

# Rate Limiting Configuration
{% if nginx_rate_limiting is defined %}
{{ rate_limiting_config(nginx_rate_limiting.zones) }}
{% endif %}

# Cache Configuration
{% if nginx_cache is defined %}
{{ cache_config(nginx_cache.zones) }}
{% endif %}

# Custom Configuration
{% if nginx_custom_config is defined %}
{{ section_header('Custom Configuration', 2) }}
{% for key, value in nginx_custom_config.items() %}
{{ config_pair(key, value) }}
{% endfor %}
{% endif %}

{{ section_header('End of Configuration', 1) }}
EOF
```

Create test for macros:

```yaml
# Create test-macros.yml
cat > test-macros.yml << 'EOF'
---
- name: Test reusable macros and functions
  hosts: web_servers
  gather_facts: yes
  vars:
    nginx_version: "1.20"
    nginx_user: "www-data"
    nginx_worker_processes: 4
    nginx_worker_connections: 2048
    nginx_keepalive_timeout: 30
    nginx_gzip_enabled: true
    nginx_gzip_level: 6
    nginx_gzip_types:
      - "text/plain"
      - "text/css"
      - "application/json"
      - "application/javascript"
    
    performance_profile: "high"
    performance_custom_settings:
      cache_size: "1GB"
      buffer_size: "64k"
    
    ssl:
      enabled: true
      certificate: "/etc/ssl/certs/app.crt"
      private_key: "/etc/ssl/private/app.key"
      protocols: ["TLSv1.2", "TLSv1.3"]
      ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256"
    
    monitoring:
      enabled: true
      port: 9090
      endpoint: "/metrics"
      metrics:
        response_time:
          enabled: true
          interval: 30
        error_rate:
          enabled: true
          interval: 60
      alerts:
        enabled: true
        channels: ["email", "slack"]
        thresholds:
          critical: 95
          warning: 80
    
    logging:
      level: "INFO"
      format: "json"
      file:
        path: "/var/log/nginx/access.log"
        max_size: "100MB"
        max_count: 10
      rotation:
        enabled: true
        schedule: "daily"
      syslog:
        enabled: false
    
    nginx_vhosts:
      - server_name: "app.example.com"
        document_root: "/var/www/app"
        ssl_enabled: true
        proxy_pass: "http://app_backend"
        aliases: ["www.app.example.com"]
      - server_name: "api.example.com"
        document_root: "/var/www/api"
        ssl_enabled: true
        custom_config:
          rate_limit_zone: "api"
          client_timeout: "30s"
    
    nginx_upstreams:
      - name: "app_backend"
        servers: ["192.168.1.10:8080", "192.168.1.11:8080"]
        method: "least_conn"
        max_fails: 3
        fail_timeout: "30s"
        health_check:
          enabled: true
          uri: "/health"
          interval: "10s"
    
    nginx_rate_limiting:
      zones:
        - name: "api"
          key: "$binary_remote_addr"
          size: "10m"
          rate: "10r/s"
          burst: 20
        - name: "login"
          key: "$binary_remote_addr"
          size: "10m"
          rate: "1r/s"
          burst: 5
    
    nginx_cache:
      zones:
        - name: "static"
          path: "/var/cache/nginx/static"
          levels: "1:2"
          keys_zone: "static:10m"
          max_size: "1g"
          inactive: "60m"
  
  tasks:
    - name: Generate configuration using macros
      template:
        src: templates/services/nginx-with-macros.j2
        dest: "/tmp/nginx-macros-{{ environment }}.conf"
        mode: '0644'
    
    - name: Display macro usage summary
      debug:
        msg: |
          Macro Usage Summary:
          - Common macros: section_header, config_pair, bool_config, list_config
          - SSL macro: ssl_config_block
          - Monitoring macro: monitoring_config_block
          - Logging macro: logging_config_block
          - Performance macro: performance_config_block
          - Web macros: virtual_host_config, upstream_config, rate_limiting_config, cache_config
          
          Generated configuration: /tmp/nginx-macros-{{ environment }}.conf
          Virtual hosts configured: {{ nginx_vhosts | length }}
          Upstream servers configured: {{ nginx_upstreams | length }}
          Rate limiting zones: {{ nginx_rate_limiting.zones | length }}
          Cache zones: {{ nginx_cache.zones | length }}
EOF

# Run macros test
ansible-playbook -i inventory.ini test-macros.yml
```

## Exercise 3: Advanced Template Composition (7 minutes)

### Task: Implement advanced template composition patterns

Create template composition framework:

```bash
# Create composition framework
mkdir -p templates/{composition,includes,partials}
```

Create composition base template:

```jinja2
# Create templates/composition/service-stack.j2
cat > templates/composition/service-stack.j2 << 'EOF'
{# Service Stack Composition Template #}
{# This template composes multiple service configurations into a unified stack #}

{# Import all macro libraries #}
{% from 'macros/common.j2' import section_header, config_pair, bool_config %}
{% from 'macros/web.j2' import virtual_host_config, upstream_config %}
{% from 'macros/database.j2' import connection_pool_config, database_config %}

# {{ ansible_managed }}
{{ section_header('Service Stack Configuration', 1, 100) }}

# Stack Information
{{ config_pair('stack_name', stack_name, required=true) }}
{{ config_pair('stack_version', stack_version, default='1.0.0') }}
{{ config_pair('stack_environment', environment | upper) }}
{{ config_pair('generated_timestamp', ansible_date_time.iso8601) }}
{{ config_pair('managed_by', 'Ansible ' + ansible_version.full) }}

# Service Composition
{% set services = stack_services | default([]) %}
{{ config_pair('total_services', services | length) }}
{{ config_pair('service_types', services | map(attribute='type') | unique | list | join(',')) }}

{% for service in services %}
{{ section_header('Service: ' + service.name + ' (' + service.type + ')', 2, 80) }}

# Service Basic Information
{{ config_pair('service_' + loop.index|string + '_name', service.name) }}
{{ config_pair('service_' + loop.index|string + '_type', service.type) }}
{{ config_pair('service_' + loop.index|string + '_version', service.version, default='latest') }}
{{ bool_config('service_' + loop.index|string + '_enabled', service.enabled, default=true) }}

# Include service-specific configuration
{% if service.type == 'web' %}
{% include 'partials/web-service-config.j2' %}
{% elif service.type == 'database' %}
{% include 'partials/database-service-config.j2' %}
{% elif service.type == 'cache' %}
{% include 'partials/cache-service-config.j2' %}
{% elif service.type == 'queue' %}
{% include 'partials/queue-service-config.j2' %}
{% else %}
# Unknown service type: {{ service.type }}
{% include 'partials/generic-service-config.j2' %}
{% endif %}

# Service Dependencies
{% if service.depends_on is defined %}
{{ config_pair('service_' + loop.index|string + '_depends_on', service.depends_on | join(',')) }}
{% endif %}

# Service Health Check
{% if service.health_check is defined %}
{{ bool_config('service_' + loop.index|string + '_health_check_enabled', service.health_check.enabled, default=true) }}
{% if service.health_check.enabled %}
{{ config_pair('service_' + loop.index|string + '_health_check_endpoint', service.health_check.endpoint, default='/health') }}
{{ config_pair('service_' + loop.index|string + '_health_check_interval', service.health_check.interval, default=30) }}
{{ config_pair('service_' + loop.index|string + '_health_check_timeout', service.health_check.timeout, default=5) }}
{% endif %}
{% endif %}

{% endfor %}

# Stack-level Configuration
{{ section_header('Stack Configuration', 2, 80) }}

# Load Balancer Configuration
{% if stack_load_balancer is defined %}
{{ bool_config('load_balancer_enabled', stack_load_balancer.enabled, default=false) }}
{% if stack_load_balancer.enabled %}
{{ config_pair('load_balancer_type', stack_load_balancer.type, default='nginx') }}
{{ config_pair('load_balancer_algorithm', stack_load_balancer.algorithm, default='round_robin') }}
{% if stack_load_balancer.backends is defined %}
{% for backend in stack_load_balancer.backends %}
{{ config_pair('load_balancer_backend_' + loop.index|string, backend.name + ':' + backend.port|string) }}
{% endfor %}
{% endif %}
{% endif %}
{% endif %}

# Service Discovery Configuration
{% if stack_service_discovery is defined %}
{{ bool_config('service_discovery_enabled', stack_service_discovery.enabled, default=false) }}
{% if stack_service_discovery.enabled %}
{{ config_pair('service_discovery_type', stack_service_discovery.type, default='consul') }}
{{ config_pair('service_discovery_endpoint', stack_service_discovery.endpoint) }}
{% endif %}
{% endif %}

# Monitoring Stack Configuration
{% if stack_monitoring is defined %}
{{ bool_config('stack_monitoring_enabled', stack_monitoring.enabled, default=true) }}
{% if stack_monitoring.enabled %}
{{ config_pair('monitoring_stack_type', stack_monitoring.type, default='prometheus') }}
{{ config_pair('monitoring_retention_days', stack_monitoring.retention_days, default=30) }}
{% if stack_monitoring.exporters is defined %}
{{ config_pair('monitoring_exporters', stack_monitoring.exporters | join(',')) }}
{% endif %}
{% endif %}
{% endif %}

# Logging Stack Configuration
{% if stack_logging is defined %}
{{ bool_config('stack_logging_enabled', stack_logging.enabled, default=true) }}
{% if stack_logging.enabled %}
{{ config_pair('logging_stack_type', stack_logging.type, default='elk') }}
{{ config_pair('log_retention_days', stack_logging.retention_days, default=30) }}
{{ config_pair('log_aggregation_endpoint', stack_logging.endpoint) }}
{% endif %}
{% endif %}

# Backup Stack Configuration
{% if stack_backup is defined %}
{{ bool_config('stack_backup_enabled', stack_backup.enabled, default=true) }}
{% if stack_backup.enabled %}
{{ config_pair('backup_schedule', stack_backup.schedule, default='0 2 * * *') }}
{{ config_pair('backup_retention_days', stack_backup.retention_days, default=30) }}
{{ config_pair('backup_storage_type', stack_backup.storage_type, default='local') }}
{% endif %}
{% endif %}

{{ section_header('End of Stack Configuration', 1, 100) }}
EOF
```

Create service-specific partials:

```jinja2
# Create templates/partials/web-service-config.j2
cat > templates/partials/web-service-config.j2 << 'EOF'
{# Web Service Configuration Partial #}

# Web Service Specific Configuration
{{ config_pair('service_' + loop.index|string + '_port', service.port, default=80) }}
{{ config_pair('service_' + loop.index|string + '_ssl_port', service.ssl_port, default=443) }}
{{ bool_config('service_' + loop.index|string + '_ssl_enabled', service.ssl_enabled, default=false) }}

# Web Server Settings
{% if service.web_server is defined %}
{{ config_pair('service_' + loop.index|string + '_web_server_type', service.web_server.type, default='nginx') }}
{{ config_pair('service_' + loop.index|string + '_worker_processes', service.web_server.worker_processes, default='auto') }}
{{ config_pair('service_' + loop.index|string + '_worker_connections', service.web_server.worker_connections, default=1024) }}
{% endif %}

# Virtual Hosts
{% if service.virtual_hosts is defined %}
{% for vhost in service.virtual_hosts %}
{{ config_pair('service_' + loop.index0|string + '_vhost_' + loop.index|string + '_name', vhost.server_name) }}
{{ config_pair('service_' + loop.index0|string + '_vhost_' + loop.index|string + '_root', vhost.document_root) }}
{% endfor %}
{% endif %}

# Upstream Configuration
{% if service.upstreams is defined %}
{% for upstream in service.upstreams %}
{{ config_pair('service_' + loop.index0|string + '_upstream_' + loop.index|string + '_name', upstream.name) }}
{{ config_pair('service_' + loop.index0|string + '_upstream_' + loop.index|string + '_servers', upstream.servers | join(',')) }}
{% endfor %}
{% endif %}
EOF

# Create templates/partials/database-service-config.j2
cat > templates/partials/database-service-config.j2 << 'EOF'
{# Database Service Configuration Partial #}

# Database Service Specific Configuration
{{ config_pair('service_' + loop.index|string + '_port', service.port, default=5432) }}
{{ config_pair('service_' + loop.index|string + '_engine', service.engine, default='postgresql') }}
{{ config_pair('service_' + loop.index|string + '_data_dir', service.data_dir, default='/var/lib/postgresql/data') }}

# Database Connection Settings
{% if service.connection is defined %}
{{ config_pair('service_' + loop.index|string + '_max_connections', service.connection.max_connections, default=100) }}
{{ config_pair('service_' + loop.index|string + '_pool_size', service.connection.pool_size, default=20) }}
{% endif %}

# Database Performance Settings
{% if service.performance is defined %}
{{ config_pair('service_' + loop.index|string + '_shared_buffers', service.performance.shared_buffers, default='256MB') }}
{{ config_pair('service_' + loop.index|string + '_effective_cache_size', service.performance.effective_cache_size, default='1GB') }}
{% endif %}

# Database Backup Settings
{% if service.backup is defined %}
{{ bool_config('service_' + loop.index|string + '_backup_enabled', service.backup.enabled, default=true) }}
{{ config_pair('service_' + loop.index|string + '_backup_schedule', service.backup.schedule, default='0 2 * * *') }}
{% endif %}

# Database Replication
{% if service.replication is defined %}
{{ bool_config('service_' + loop.index|string + '_replication_enabled', service.replication.enabled, default=false) }}
{% if service.replication.enabled %}
{{ config_pair('service_' + loop.index|string + '_replication_role', service.replication.role, default='master') }}
{% endif %}
{% endif %}
EOF

# Create templates/partials/cache-service-config.j2
cat > templates/partials/cache-service-config.j2 << 'EOF'
{# Cache Service Configuration Partial #}

# Cache Service Specific Configuration
{{ config_pair('service_' + loop.index|string + '_port', service.port, default=6379) }}
{{ config_pair('service_' + loop.index|string + '_engine', service.engine, default='redis') }}
{{ config_pair('service_' + loop.index|string + '_memory_limit', service.memory_limit, default='256MB') }}

# Cache Settings
{% if service.cache_settings is defined %}
{{ config_pair('service_' + loop.index|string + '_max_memory_policy', service.cache_settings.max_memory_policy, default='allkeys-lru') }}
{{ config_pair('service_' + loop.index|string + '_timeout', service.cache_settings.timeout, default=0) }}
{{ config_pair('service_' + loop.index|string + '_tcp_keepalive', service.cache_settings.tcp_keepalive, default=300) }}
{% endif %}

# Persistence Settings
{% if service.persistence is defined %}
{{ bool_config('service_' + loop.index|string + '_persistence_enabled', service.persistence.enabled, default=true) }}
{% if service.persistence.enabled %}
{{ config_pair('service_' + loop.index|string + '_save_schedule', service.persistence.save_schedule, default='900 1 300 10 60 10000') }}
{% endif %}
{% endif %}

# Clustering
{% if service.cluster is defined %}
{{ bool_config('service_' + loop.index|string + '_cluster_enabled', service.cluster.enabled, default=false) }}
{% if service.cluster.enabled %}
{{ config_pair('service_' + loop.index|string + '_cluster_nodes', service.cluster.nodes | join(',')) }}
{% endif %}
{% endif %}
EOF
```

Create comprehensive composition test:

```yaml
# Create test-composition.yml
cat > test-composition.yml << 'EOF'
---
- name: Test advanced template composition
  hosts: localhost
  gather_facts: yes
  vars:
    stack_name: "enterprise-web-stack"
    stack_version: "2.1.0"
    
    stack_services:
      - name: "web-frontend"
        type: "web"
        version: "1.8.0"
        enabled: true
        port: 80
        ssl_port: 443
        ssl_enabled: true
        web_server:
          type: "nginx"
          worker_processes: 4
          worker_connections: 2048
        virtual_hosts:
          - server_name: "app.example.com"
            document_root: "/var/www/app"
          - server_name: "api.example.com"
            document_root: "/var/www/api"
        upstreams:
          - name: "app_backend"
            servers: ["192.168.1.10:8080", "192.168.1.11:8080"]
        health_check:
          enabled: true
          endpoint: "/health"
          interval: 30
          timeout: 5
        depends_on: ["database-primary", "cache-redis"]
      
      - name: "database-primary"
        type: "database"
        version: "13.0"
        enabled: true
        port: 5432
        engine: "postgresql"
        data_dir: "/var/lib/postgresql/13/main"
        connection:
          max_connections: 200
          pool_size: 50
        performance:
          shared_buffers: "512MB"
          effective_cache_size: "2GB"
        backup:
          enabled: true
          schedule: "0 2 * * *"
        replication:
          enabled: true
          role: "master"
        health_check:
          enabled: true
          endpoint: "SELECT 1"
          interval: 60
          timeout: 10
      
      - name: "cache-redis"
        type: "cache"
        version: "6.2"
        enabled: true
        port: 6379
        engine: "redis"
        memory_limit: "1GB"
        cache_settings:
          max_memory_policy: "allkeys-lru"
          timeout: 300
          tcp_keepalive: 300
        persistence:
          enabled: true
          save_schedule: "900 1 300 10 60 10000"
        cluster:
          enabled: false
        health_check:
          enabled: true
          endpoint: "PING"
          interval: 30
          timeout: 3
    
    stack_load_balancer:
      enabled: true
      type: "nginx"
      algorithm: "least_conn"
      backends:
        - name: "web-frontend"
          port: 80
    
    stack_service_discovery:
      enabled: true
      type: "consul"
      endpoint: "http://consul.service.consul:8500"
    
    stack_monitoring:
      enabled: true
      type: "prometheus"
      retention_days: 30
      exporters: ["node_exporter", "postgres_exporter", "redis_exporter"]
    
    stack_logging:
      enabled: true
      type: "elk"
      retention_days: 30
      endpoint: "http://elasticsearch.service.consul:9200"
    
    stack_backup:
      enabled: true
      schedule: "0 1 * * *"
      retention_days: 30
      storage_type: "s3"
  
  tasks:
    - name: Generate service stack configuration using composition
      template:
        src: templates/composition/service-stack.j2
        dest: "/tmp/service-stack-{{ environment | default('production') }}.conf"
        mode: '0644'
    
    - name: Display composition summary
      debug:
        msg: |
          Service Stack Composition Summary:
          
          Stack Information:
          - Name: {{ stack_name }}
          - Version: {{ stack_version }}
          - Environment: {{ environment | default('production') | upper }}
          - Total Services: {{ stack_services | length }}
          
          Services Configured:
          {% for service in stack_services %}
          - {{ service.name }} ({{ service.type }}) - {{ service.version }}
            Port: {{ service.port }}
            Health Check: {{ 'Enabled' if service.health_check.enabled else 'Disabled' }}
            Dependencies: {{ service.depends_on | default([]) | join(', ') or 'None' }}
          {% endfor %}
          
          Stack Features:
          - Load Balancer: {{ 'Enabled' if stack_load_balancer.enabled else 'Disabled' }}
          - Service Discovery: {{ 'Enabled' if stack_service_discovery.enabled else 'Disabled' }}
          - Monitoring: {{ 'Enabled' if stack_monitoring.enabled else 'Disabled' }}
          - Logging: {{ 'Enabled' if stack_logging.enabled else 'Disabled' }}
          - Backup: {{ 'Enabled' if stack_backup.enabled else 'Disabled' }}
          
          Generated Configuration: /tmp/service-stack-{{ environment | default('production') }}.conf
EOF

# Run composition test
ansible-playbook -i inventory.ini test-composition.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Check generated configurations
echo "=== Template Inheritance Results ==="
ls -la /tmp/*config*.conf 2>/dev/null || echo "No inheritance configs generated"

echo "=== Macro-based Configuration ==="
ls -la /tmp/nginx-macros-*.conf 2>/dev/null || echo "No macro configs generated"

echo "=== Service Stack Composition ==="
ls -la /tmp/service-stack-*.conf 2>/dev/null || echo "No composition configs generated"

# Show sample content
echo "=== Sample Configuration Content ==="
head -30 /tmp/nginx-macros-production.conf 2>/dev/null || echo "No sample content available"
```

### 2. Discussion Points
- How do you manage template complexity while maintaining readability?
- What are the benefits of template inheritance versus composition?
- How do you organize macro libraries for team collaboration?
- What strategies do you use for template testing and validation?

### 3. Clean Up
```bash
# Keep files for next exercises
# rm -rf /tmp/*config* /tmp/*nginx*
```

## Key Takeaways
- Template inheritance provides hierarchical configuration management
- Reusable macros eliminate code duplication and improve consistency
- Template composition enables complex multi-service configurations
- Proper organization of templates and macros improves maintainability
- Advanced template patterns support enterprise-scale automation
- Modular design enables flexible configuration generation

## Next Steps
Proceed to Lab 6.4: Data Transformation and Processing

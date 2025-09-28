# Lab 6.1: Advanced Jinja2 Syntax and Expressions

## Objective
Master complex Jinja2 expressions, filters, control structures, and advanced templating techniques for enterprise automation.

## Duration
25 minutes

## Prerequisites
- Basic understanding of Jinja2 templating
- Knowledge of YAML and JSON data structures

## Lab Setup

```bash
cd ~/ansible-labs
mkdir -p module-06/lab-6.1
cd module-06/lab-6.1

# Create inventory for testing
cat > inventory.ini << 'EOF'
[test_servers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Complex Expressions and Filters (10 minutes)

### Task: Master advanced Jinja2 expressions and built-in filters

Create comprehensive test data:

```yaml
# Create test-data.yml
cat > test-data.yml << 'EOF'
---
# Complex test data for Jinja2 templating

# Server inventory data
servers:
  web_servers:
    - name: "web-01"
      ip: "192.168.1.10"
      cpu_cores: 4
      memory_gb: 16
      disk_gb: 500
      environment: "production"
      services: ["nginx", "php-fpm", "redis"]
      tags:
        role: "frontend"
        tier: "web"
        backup: "enabled"
      last_updated: "2024-01-15T10:30:00Z"
      metrics:
        cpu_usage: 65.5
        memory_usage: 78.2
        disk_usage: 45.8
    - name: "web-02"
      ip: "192.168.1.11"
      cpu_cores: 4
      memory_gb: 16
      disk_gb: 500
      environment: "production"
      services: ["nginx", "php-fpm"]
      tags:
        role: "frontend"
        tier: "web"
        backup: "enabled"
      last_updated: "2024-01-15T11:15:00Z"
      metrics:
        cpu_usage: 72.1
        memory_usage: 82.5
        disk_usage: 52.3
  
  database_servers:
    - name: "db-01"
      ip: "192.168.1.20"
      cpu_cores: 8
      memory_gb: 64
      disk_gb: 2000
      environment: "production"
      services: ["postgresql", "pgbouncer"]
      tags:
        role: "database"
        tier: "data"
        backup: "critical"
      last_updated: "2024-01-15T09:45:00Z"
      metrics:
        cpu_usage: 45.2
        memory_usage: 68.9
        disk_usage: 73.1
    - name: "db-02"
      ip: "192.168.1.21"
      cpu_cores: 8
      memory_gb: 64
      disk_gb: 2000
      environment: "production"
      services: ["postgresql"]
      tags:
        role: "database"
        tier: "data"
        backup: "critical"
      last_updated: "2024-01-15T10:00:00Z"
      metrics:
        cpu_usage: 38.7
        memory_usage: 71.4
        disk_usage: 69.8

# Application configuration
applications:
  web_app:
    name: "Enterprise Web Application"
    version: "2.1.0"
    port: 8080
    ssl_enabled: true
    database_connections:
      primary: "postgresql://app_user:password@192.168.1.20:5432/app_db"
      replica: "postgresql://app_user:password@192.168.1.21:5432/app_db"
    cache_config:
      type: "redis"
      host: "192.168.1.10"
      port: 6379
      ttl: 3600
    features:
      - "user_authentication"
      - "api_gateway"
      - "real_time_notifications"
      - "analytics_tracking"
    
# Network configuration
networks:
  production:
    cidr: "192.168.1.0/24"
    gateway: "192.168.1.1"
    dns_servers: ["8.8.8.8", "8.8.4.4"]
    vlans:
      web: 10
      database: 20
      management: 30
  
  staging:
    cidr: "192.168.2.0/24"
    gateway: "192.168.2.1"
    dns_servers: ["8.8.8.8", "1.1.1.1"]
    vlans:
      web: 110
      database: 120

# User and permissions data
users:
  - username: "admin"
    full_name: "System Administrator"
    email: "admin@company.com"
    roles: ["admin", "operator"]
    permissions: ["read", "write", "execute", "delete"]
    last_login: "2024-01-15T14:30:00Z"
    active: true
  - username: "developer"
    full_name: "Application Developer"
    email: "dev@company.com"
    roles: ["developer"]
    permissions: ["read", "write"]
    last_login: "2024-01-14T16:45:00Z"
    active: true
  - username: "readonly"
    full_name: "Read Only User"
    email: "readonly@company.com"
    roles: ["viewer"]
    permissions: ["read"]
    last_login: "2024-01-13T09:15:00Z"
    active: false

# Monitoring thresholds
monitoring:
  thresholds:
    cpu:
      warning: 70
      critical: 85
    memory:
      warning: 80
      critical: 90
    disk:
      warning: 75
      critical: 85
  
  alert_channels:
    - type: "email"
      recipients: ["ops@company.com", "admin@company.com"]
    - type: "slack"
      webhook: "https://hooks.slack.com/services/xxx/yyy/zzz"
    - type: "pagerduty"
      service_key: "abcd1234efgh5678"

# Time-based data
schedules:
  backup:
    daily: "02:00"
    weekly: "Sunday 01:00"
    monthly: "1st Sunday 00:00"
  maintenance:
    patching: "2nd Sunday 03:00"
    restart: "Every Sunday 04:00"
EOF
```

Create advanced expressions template:

```jinja2
# Create templates/advanced-expressions.j2
mkdir -p templates
cat > templates/advanced-expressions.j2 << 'EOF'
{# Advanced Jinja2 Expressions and Filters Template #}
{# {{ ansible_managed }} #}

{# ========================================= #}
{# BASIC EXPRESSIONS AND VARIABLE ACCESS    #}
{# ========================================= #}

Server Count: {{ servers.web_servers | length + servers.database_servers | length }}
Total CPU Cores: {{ (servers.web_servers | sum(attribute='cpu_cores')) + (servers.database_servers | sum(attribute='cpu_cores')) }}
Total Memory: {{ (servers.web_servers | sum(attribute='memory_gb')) + (servers.database_servers | sum(attribute='memory_gb')) }}GB

{# ========================================= #}
{# CONDITIONAL EXPRESSIONS                   #}
{# ========================================= #}

{# Ternary operator usage #}
Environment Status: {{ 'PRODUCTION' if servers.web_servers[0].environment == 'production' else 'NON-PRODUCTION' }}
SSL Configuration: {{ 'Enabled' if applications.web_app.ssl_enabled else 'Disabled' }}

{# Complex conditional with multiple conditions #}
System Health: {{ 'HEALTHY' if (servers.web_servers | selectattr('metrics.cpu_usage', 'lt', monitoring.thresholds.cpu.warning) | list | length) > 0 else 'WARNING' }}

{# ========================================= #}
{# STRING MANIPULATION FILTERS              #}
{# ========================================= #}

{# String formatting and manipulation #}
Application Title: {{ applications.web_app.name | title }}
Application Slug: {{ applications.web_app.name | lower | replace(' ', '-') }}
Version Info: {{ applications.web_app.name }} v{{ applications.web_app.version | regex_replace('^(\d+\.\d+).*', '\1') }}

{# String padding and alignment #}
{% for server in servers.web_servers %}
Server: {{ server.name | ljust(10) }} | IP: {{ server.ip | rjust(15) }} | CPU: {{ server.metrics.cpu_usage | round(1) | string | rjust(6) }}%
{% endfor %}

{# ========================================= #}
{# NUMERIC FILTERS AND CALCULATIONS         #}
{# ========================================= #}

{# Mathematical operations #}
Average CPU Usage (Web): {{ (servers.web_servers | sum(attribute='metrics.cpu_usage')) / (servers.web_servers | length) | round(2) }}%
Average Memory Usage (DB): {{ (servers.database_servers | sum(attribute='metrics.memory_usage')) / (servers.database_servers | length) | round(2) }}%

{# Min/Max calculations #}
Highest CPU Usage: {{ servers.web_servers | map(attribute='metrics.cpu_usage') | max }}%
Lowest Disk Usage: {{ servers.database_servers | map(attribute='metrics.disk_usage') | min }}%

{# Number formatting #}
Total Storage: {{ ((servers.web_servers | sum(attribute='disk_gb')) + (servers.database_servers | sum(attribute='disk_gb'))) | filesizeformat }}
Memory per Server: {{ servers.web_servers[0].memory_gb * 1024 * 1024 * 1024 | filesizeformat }}

{# ========================================= #}
{# LIST AND DICTIONARY FILTERS             #}
{# ========================================= #}

{# List operations #}
All Server Names: {{ (servers.web_servers + servers.database_servers) | map(attribute='name') | join(', ') }}
Unique Services: {{ (servers.web_servers + servers.database_servers) | map(attribute='services') | flatten | unique | sort | join(', ') }}

{# Dictionary operations #}
Network VLANs: {{ networks.production.vlans | dictsort | map('join', ': ') | join(', ') }}
User Roles: {{ users | groupby('roles') | map('first') | join(', ') }}

{# ========================================= #}
{# ADVANCED FILTERING AND SELECTION         #}
{# ========================================= #}

{# Complex filtering #}
High CPU Servers:
{% for server in (servers.web_servers + servers.database_servers) | selectattr('metrics.cpu_usage', 'gt', monitoring.thresholds.cpu.warning) %}
  - {{ server.name }}: {{ server.metrics.cpu_usage }}% CPU
{% endfor %}

Critical Backup Servers:
{% for server in (servers.web_servers + servers.database_servers) | selectattr('tags.backup', 'equalto', 'critical') %}
  - {{ server.name }} ({{ server.tags.role }})
{% endfor %}

Active Users with Write Permissions:
{% for user in users | selectattr('active') | selectattr('permissions', 'contains', 'write') %}
  - {{ user.full_name }} ({{ user.username }})
{% endfor %}

{# ========================================= #}
{# DATE AND TIME MANIPULATION               #}
{# ========================================= #}

{# Date formatting and manipulation #}
{% for server in servers.web_servers %}
{{ server.name }} last updated: {{ server.last_updated | to_datetime('%Y-%m-%dT%H:%M:%SZ') | strftime('%B %d, %Y at %I:%M %p') }}
{% endfor %}

{# Time-based calculations #}
{% set current_time = ansible_date_time.iso8601 %}
Report Generated: {{ current_time | to_datetime('%Y-%m-%dT%H:%M:%SZ') | strftime('%A, %B %d, %Y at %I:%M %p UTC') }}

{# ========================================= #}
{# CUSTOM LOGIC AND COMPLEX EXPRESSIONS     #}
{# ========================================= #}

{# Complex nested expressions #}
Server Resource Allocation:
{% for server_type, server_list in servers.items() %}
{{ server_type | replace('_', ' ') | title }}:
{% for server in server_list %}
  - {{ server.name }}:
    Resource Score: {{ ((server.cpu_cores * 25) + (server.memory_gb * 1.5) + (server.disk_gb * 0.01)) | int }}
    Status: {{ 'Overloaded' if server.metrics.cpu_usage > monitoring.thresholds.cpu.critical else 'Normal' }}
    Tier: {{ server.tags.tier | upper }}
{% endfor %}
{% endfor %}

{# Advanced conditional logic #}
System Recommendations:
{% set total_servers = servers.web_servers | length + servers.database_servers | length %}
{% set avg_cpu = ((servers.web_servers | sum(attribute='metrics.cpu_usage')) + (servers.database_servers | sum(attribute='metrics.cpu_usage'))) / total_servers %}
{% if avg_cpu > monitoring.thresholds.cpu.critical %}
  ⚠️  CRITICAL: Average CPU usage ({{ avg_cpu | round(1) }}%) exceeds critical threshold
  Recommendation: Scale out infrastructure immediately
{% elif avg_cpu > monitoring.thresholds.cpu.warning %}
  ⚠️  WARNING: Average CPU usage ({{ avg_cpu | round(1) }}%) exceeds warning threshold
  Recommendation: Plan for capacity expansion
{% else %}
  ✅ HEALTHY: Average CPU usage ({{ avg_cpu | round(1) }}%) is within normal range
  Recommendation: Continue monitoring
{% endif %}

{# ========================================= #}
{# ADVANCED LOOPING AND GROUPING            #}
{# ========================================= #}

{# Grouping and advanced iteration #}
Servers by Environment:
{% for env, server_list in (servers.web_servers + servers.database_servers) | groupby('environment') %}
{{ env | upper }}:
{% for server in server_list %}
  - {{ server.name }} ({{ server.tags.role }}) - {{ server.ip }}
{% endfor %}
{% endfor %}

Services Distribution:
{% for service, server_list in (servers.web_servers + servers.database_servers) | selectattr('services') | map(attribute='services') | flatten | groupby %}
{{ service }}:
  Count: {{ server_list | length }}
  Servers: {{ (servers.web_servers + servers.database_servers) | selectattr('services', 'contains', service) | map(attribute='name') | join(', ') }}
{% endfor %}

{# ========================================= #}
{# ERROR HANDLING AND DEFAULTS              #}
{# ========================================= #}

{# Safe navigation and defaults #}
Backup Configuration:
{% for server in (servers.web_servers + servers.database_servers) %}
  {{ server.name }}: {{ server.tags.backup | default('not_configured') | title }}
{% endfor %}

Optional Features:
{% for feature in applications.web_app.features | default([]) %}
  - {{ feature | replace('_', ' ') | title }}
{% endfor %}

{# Handling missing data #}
Network Configuration:
{% if networks.production is defined %}
Production Network: {{ networks.production.cidr }}
Gateway: {{ networks.production.gateway }}
DNS: {{ networks.production.dns_servers | join(', ') }}
{% else %}
Production Network: Not configured
{% endif %}

{# ========================================= #}
{# TEMPLATE COMMENTS AND DOCUMENTATION      #}
{# ========================================= #}

{#
  This template demonstrates advanced Jinja2 features:
  - Complex expressions and calculations
  - Advanced filtering and selection
  - String manipulation and formatting
  - Date/time operations
  - Conditional logic and error handling
  - Grouping and advanced iteration
  
  Generated on: {{ ansible_date_time.iso8601 }}
  Template version: 1.0
#}
EOF
```

Create test playbook:

```yaml
# Create test-advanced-expressions.yml
cat > test-advanced-expressions.yml << 'EOF'
---
- name: Test advanced Jinja2 expressions
  hosts: test_servers
  gather_facts: yes
  vars_files:
    - test-data.yml
  
  tasks:
    - name: Generate advanced expressions report
      template:
        src: templates/advanced-expressions.j2
        dest: /tmp/advanced-expressions-report.txt
        mode: '0644'
    
    - name: Display generated report
      debug:
        msg: "Advanced expressions report generated at /tmp/advanced-expressions-report.txt"
    
    - name: Show sample of advanced expressions
      debug:
        msg: |
          Sample Advanced Expressions:
          - Total Servers: {{ servers.web_servers | length + servers.database_servers | length }}
          - Average CPU: {{ ((servers.web_servers | sum(attribute='metrics.cpu_usage')) + (servers.database_servers | sum(attribute='metrics.cpu_usage'))) / (servers.web_servers | length + servers.database_servers | length) | round(2) }}%
          - SSL Status: {{ 'Enabled' if applications.web_app.ssl_enabled else 'Disabled' }}
          - Active Users: {{ users | selectattr('active') | list | length }}
          - Unique Services: {{ (servers.web_servers + servers.database_servers) | map(attribute='services') | flatten | unique | length }}
EOF

# Run the test
ansible-playbook -i inventory.ini test-advanced-expressions.yml
```

## Exercise 2: Control Structures and Logic (8 minutes)

### Task: Master advanced control structures, loops, and conditional logic

Create control structures template:

```jinja2
# Create templates/control-structures.j2
cat > templates/control-structures.j2 << 'EOF'
{# Advanced Control Structures Template #}
{# {{ ansible_managed }} #}

{# ========================================= #}
{# ADVANCED IF-ELSE LOGIC                   #}
{# ========================================= #}

{# Multi-condition if statements #}
{% if servers.web_servers | length > 2 and servers.database_servers | length >= 1 %}
Infrastructure Status: HIGH AVAILABILITY CONFIGURED
  - Web Servers: {{ servers.web_servers | length }} (Redundant)
  - Database Servers: {{ servers.database_servers | length }} ({{ 'Redundant' if servers.database_servers | length > 1 else 'Single Point of Failure' }})
{% elif servers.web_servers | length > 0 %}
Infrastructure Status: BASIC CONFIGURATION
  - Web Servers: {{ servers.web_servers | length }}
  - Database Servers: {{ servers.database_servers | length }}
{% else %}
Infrastructure Status: NO SERVERS CONFIGURED
{% endif %}

{# Nested conditional logic #}
Security Assessment:
{% for server in (servers.web_servers + servers.database_servers) %}
{{ server.name }}:
{% if server.tags.backup == 'critical' %}
  {% if server.metrics.disk_usage > monitoring.thresholds.disk.critical %}
  ❌ CRITICAL: High disk usage on critical backup server
  {% elif server.metrics.disk_usage > monitoring.thresholds.disk.warning %}
  ⚠️  WARNING: Elevated disk usage on critical backup server
  {% else %}
  ✅ OK: Critical backup server disk usage normal
  {% endif %}
{% elif server.tags.backup == 'enabled' %}
  {% if server.metrics.disk_usage > monitoring.thresholds.disk.warning %}
  ⚠️  WARNING: High disk usage on backup-enabled server
  {% else %}
  ✅ OK: Backup-enabled server disk usage normal
  {% endif %}
{% else %}
  ℹ️  INFO: No backup configured for {{ server.name }}
{% endif %}
{% endfor %}

{# ========================================= #}
{# ADVANCED LOOPING TECHNIQUES              #}
{# ========================================= #}

{# Loop with complex filtering #}
High Resource Servers (CPU > 70% OR Memory > 75%):
{% for server in (servers.web_servers + servers.database_servers) if server.metrics.cpu_usage > 70 or server.metrics.memory_usage > 75 %}
  {{ loop.index }}. {{ server.name }}:
     CPU: {{ server.metrics.cpu_usage }}% {{ '(HIGH)' if server.metrics.cpu_usage > monitoring.thresholds.cpu.warning else '' }}
     Memory: {{ server.metrics.memory_usage }}% {{ '(HIGH)' if server.metrics.memory_usage > monitoring.thresholds.memory.warning else '' }}
     Priority: {{ 'CRITICAL' if loop.first else 'HIGH' if loop.index <= 3 else 'MEDIUM' }}
{% else %}
  ✅ All servers operating within normal resource limits
{% endfor %}

{# Loop with grouping and sorting #}
Servers by Role and Performance:
{% for role, role_servers in (servers.web_servers + servers.database_servers) | groupby('tags.role') %}
{{ role | upper }} SERVERS:
{% for server in role_servers | sort(attribute='metrics.cpu_usage', reverse=true) %}
  {{ loop.index }}. {{ server.name }}:
     Performance Score: {{ (100 - server.metrics.cpu_usage) | int }}/100
     Status: {{ 'Needs Attention' if server.metrics.cpu_usage > 80 else 'Good' }}
     {% if loop.last %}
     └── End of {{ role }} servers
     {% endif %}
{% endfor %}
{% endfor %}

{# Nested loops with conditions #}
Service Distribution Matrix:
{% set all_services = (servers.web_servers + servers.database_servers) | map(attribute='services') | flatten | unique | sort %}
{% for service in all_services %}
{{ service | upper }}:
{% for server_type, server_list in servers.items() %}
  {{ server_type | replace('_', ' ') | title }}:
  {% for server in server_list if service in server.services %}
    - {{ server.name }} ({{ server.ip }})
  {% else %}
    - No {{ service }} instances
  {% endfor %}
{% endfor %}
{% endfor %}

{# ========================================= #}
{# LOOP VARIABLES AND METADATA              #}
{# ========================================= #}

Server Processing Status:
{% for server in (servers.web_servers + servers.database_servers) %}
[{{ loop.index0 | string | rjust(2, '0') }}/{{ loop.length | string | rjust(2, '0') }}] Processing {{ server.name }}...
  - Position: {{ 'First' if loop.first else 'Last' if loop.last else 'Middle' }}
  - Progress: {{ (loop.index / loop.length * 100) | round(1) }}%
  - Remaining: {{ loop.length - loop.index }} servers
  {% if loop.index % 3 == 0 %}
  - Checkpoint: Processed {{ loop.index }} of {{ loop.length }} servers
  {% endif %}
{% endfor %}

{# ========================================= #}
{# COMPLEX CONDITIONAL EXPRESSIONS          #}
{# ========================================= #}

{# Multi-level conditional assignments #}
{% set server_categories = {} %}
{% for server in (servers.web_servers + servers.database_servers) %}
  {% set category = 'critical' if server.metrics.cpu_usage > monitoring.thresholds.cpu.critical 
                    else 'warning' if server.metrics.cpu_usage > monitoring.thresholds.cpu.warning 
                    else 'normal' %}
  {% if server_categories.update({category: server_categories.get(category, []) + [server.name]}) %}{% endif %}
{% endfor %}

Server Categories:
{% for category, server_names in server_categories.items() %}
{{ category | upper }}: {{ server_names | join(', ') }}
{% endfor %}

{# Complex business logic #}
Maintenance Window Recommendations:
{% for server in (servers.web_servers + servers.database_servers) %}
{{ server.name }}:
{% set maintenance_priority = 'high' if server.metrics.cpu_usage > 80 or server.metrics.memory_usage > 85
                              else 'medium' if server.metrics.disk_usage > 70
                              else 'low' %}
{% set maintenance_window = 'immediate' if maintenance_priority == 'high'
                           else 'next_weekend' if maintenance_priority == 'medium'
                           else 'next_month' %}
  Priority: {{ maintenance_priority | upper }}
  Recommended Window: {{ maintenance_window | replace('_', ' ') | title }}
  Reason: {% if maintenance_priority == 'high' %}High resource utilization detected
          {% elif maintenance_priority == 'medium' %}Elevated disk usage
          {% else %}Routine maintenance
          {% endif %}
{% endfor %}

{# ========================================= #}
{# ERROR HANDLING IN CONTROL STRUCTURES     #}
{# ========================================= #}

{# Safe iteration with error handling #}
User Access Report:
{% for user in users | default([]) %}
{% if user.username is defined and user.active is defined %}
{{ user.username }}:
  Status: {{ 'Active' if user.active else 'Inactive' }}
  Roles: {{ user.roles | default(['none']) | join(', ') }}
  Last Login: {{ user.last_login | default('Never') }}
  {% if user.permissions is defined and user.permissions | length > 0 %}
  Permissions: {{ user.permissions | join(', ') }}
  {% else %}
  Permissions: No permissions assigned
  {% endif %}
{% else %}
  ⚠️  WARNING: Incomplete user data for entry {{ loop.index }}
{% endif %}
{% endfor %}

{# Conditional blocks with fallbacks #}
{% if monitoring is defined and monitoring.thresholds is defined %}
Monitoring Configuration:
  CPU Warning: {{ monitoring.thresholds.cpu.warning }}%
  CPU Critical: {{ monitoring.thresholds.cpu.critical }}%
  Memory Warning: {{ monitoring.thresholds.memory.warning }}%
  Memory Critical: {{ monitoring.thresholds.memory.critical }}%
  
Alert Channels:
{% for channel in monitoring.alert_channels | default([]) %}
  - {{ channel.type | title }}: {{ channel.recipients | default([channel.webhook, channel.service_key]) | select | first | default('Not configured') }}
{% endfor %}
{% else %}
Monitoring Configuration: Not available or incomplete
{% endif %}

{# ========================================= #}
{# ADVANCED PATTERN MATCHING                #}
{# ========================================= #}

{# Pattern-based server classification #}
Server Classification by Name Pattern:
{% for server in (servers.web_servers + servers.database_servers) %}
{{ server.name }}:
{% if server.name | regex_search('^web-') %}
  Type: Web Server (by naming convention)
  Expected Services: HTTP, HTTPS, Application Runtime
{% elif server.name | regex_search('^db-') %}
  Type: Database Server (by naming convention)
  Expected Services: Database Engine, Backup Tools
{% elif server.name | regex_search('^lb-') %}
  Type: Load Balancer (by naming convention)
  Expected Services: Load Balancing, Health Checks
{% else %}
  Type: Unknown (naming convention not followed)
  Recommendation: Review naming standards
{% endif %}
{% endfor %}

{# ========================================= #}
{# TEMPLATE DEBUGGING AND INFORMATION       #}
{# ========================================= #}

Template Processing Information:
- Template: control-structures.j2
- Generated: {{ ansible_date_time.iso8601 }}
- Total Servers Processed: {{ (servers.web_servers + servers.database_servers) | length }}
- Total Users Processed: {{ users | length }}
- Template Complexity: Advanced Control Structures
- Processing Time: {{ ansible_date_time.epoch }} (Unix timestamp)
EOF
```

Create control structures test:

```yaml
# Create test-control-structures.yml
cat > test-control-structures.yml << 'EOF'
---
- name: Test advanced control structures
  hosts: test_servers
  gather_facts: yes
  vars_files:
    - test-data.yml
  
  tasks:
    - name: Generate control structures report
      template:
        src: templates/control-structures.j2
        dest: /tmp/control-structures-report.txt
        mode: '0644'
    
    - name: Test conditional logic
      debug:
        msg: |
          Control Structure Tests:
          {% set total_servers = servers.web_servers | length + servers.database_servers | length %}
          - Infrastructure Type: {{ 'Enterprise' if total_servers > 5 else 'Standard' if total_servers > 2 else 'Basic' }}
          - High Availability: {{ 'Yes' if servers.database_servers | length > 1 else 'No' }}
          - Critical Servers: {{ (servers.web_servers + servers.database_servers) | selectattr('tags.backup', 'equalto', 'critical') | list | length }}
    
    - name: Test loop variables
      debug:
        msg: "Server {{ item.name }} is #{{ loop.index }} of {{ servers.web_servers | length }}"
      loop: "{{ servers.web_servers }}"
      loop_control:
        extended: yes
      when: servers.web_servers is defined
EOF

# Run the control structures test
ansible-playbook -i inventory.ini test-control-structures.yml
```

## Exercise 3: Custom Filters and Functions (7 minutes)

### Task: Create and use custom Jinja2 filters for specialized data processing

Create custom filter plugins:

```python
# Create filter plugins directory
mkdir -p filter_plugins

# Create custom filters
cat > filter_plugins/custom_filters.py << 'EOF'
#!/usr/bin/env python3
"""
Custom Jinja2 filters for advanced templating
"""

import re
import json
from datetime import datetime, timedelta
from ansible.errors import AnsibleFilterError

class FilterModule(object):
    """Custom filter module"""
    
    def filters(self):
        return {
            'calculate_uptime': self.calculate_uptime,
            'format_bytes': self.format_bytes,
            'network_info': self.network_info,
            'health_status': self.health_status,
            'generate_password': self.generate_password,
            'validate_config': self.validate_config,
            'extract_version': self.extract_version,
            'sort_by_metric': self.sort_by_metric,
            'group_by_threshold': self.group_by_threshold,
            'calculate_score': self.calculate_score
        }
    
    def calculate_uptime(self, last_updated_str):
        """Calculate uptime from last updated timestamp"""
        try:
            last_updated = datetime.fromisoformat(last_updated_str.replace('Z', '+00:00'))
            now = datetime.now(last_updated.tzinfo)
            uptime = now - last_updated
            
            days = uptime.days
            hours, remainder = divmod(uptime.seconds, 3600)
            minutes, _ = divmod(remainder, 60)
            
            if days > 0:
                return f"{days}d {hours}h {minutes}m"
            elif hours > 0:
                return f"{hours}h {minutes}m"
            else:
                return f"{minutes}m"
        except Exception as e:
            return f"Error: {str(e)}"
    
    def format_bytes(self, bytes_value, unit='auto'):
        """Format bytes into human readable format"""
        try:
            bytes_value = float(bytes_value)
            
            if unit == 'auto':
                for unit_name in ['B', 'KB', 'MB', 'GB', 'TB']:
                    if bytes_value < 1024.0:
                        return f"{bytes_value:.1f} {unit_name}"
                    bytes_value /= 1024.0
                return f"{bytes_value:.1f} PB"
            else:
                units = {'B': 1, 'KB': 1024, 'MB': 1024**2, 'GB': 1024**3, 'TB': 1024**4}
                if unit in units:
                    return f"{bytes_value / units[unit]:.2f} {unit}"
                else:
                    raise AnsibleFilterError(f"Invalid unit: {unit}")
        except Exception as e:
            raise AnsibleFilterError(f"Error formatting bytes: {str(e)}")
    
    def network_info(self, ip_address):
        """Extract network information from IP address"""
        try:
            octets = ip_address.split('.')
            if len(octets) != 4:
                raise ValueError("Invalid IP address format")
            
            network_class = 'A' if int(octets[0]) < 128 else 'B' if int(octets[0]) < 192 else 'C'
            is_private = (octets[0] == '10' or 
                         (octets[0] == '172' and 16 <= int(octets[1]) <= 31) or
                         (octets[0] == '192' and octets[1] == '168'))
            
            return {
                'class': network_class,
                'private': is_private,
                'octets': octets,
                'network': f"{octets[0]}.{octets[1]}.{octets[2]}.0"
            }
        except Exception as e:
            raise AnsibleFilterError(f"Error processing IP address: {str(e)}")
    
    def health_status(self, metrics, thresholds):
        """Calculate health status based on metrics and thresholds"""
        try:
            status = 'healthy'
            issues = []
            
            for metric, value in metrics.items():
                if metric in thresholds:
                    threshold = thresholds[metric]
                    if value > threshold.get('critical', 100):
                        status = 'critical'
                        issues.append(f"{metric}: {value}% > {threshold['critical']}% (critical)")
                    elif value > threshold.get('warning', 80):
                        if status != 'critical':
                            status = 'warning'
                        issues.append(f"{metric}: {value}% > {threshold['warning']}% (warning)")
            
            return {
                'status': status,
                'issues': issues,
                'score': max(0, 100 - len(issues) * 20)
            }
        except Exception as e:
            raise AnsibleFilterError(f"Error calculating health status: {str(e)}")
    
    def generate_password(self, length=12, complexity='medium'):
        """Generate password based on complexity requirements"""
        import random
        import string
        
        try:
            if complexity == 'low':
                chars = string.ascii_letters + string.digits
            elif complexity == 'medium':
                chars = string.ascii_letters + string.digits + '!@#$%'
            elif complexity == 'high':
                chars = string.ascii_letters + string.digits + '!@#$%^&*()_+-=[]{}|;:,.<>?'
            else:
                raise ValueError(f"Invalid complexity level: {complexity}")
            
            password = ''.join(random.choice(chars) for _ in range(length))
            return password
        except Exception as e:
            raise AnsibleFilterError(f"Error generating password: {str(e)}")
    
    def validate_config(self, config, schema):
        """Validate configuration against schema"""
        try:
            errors = []
            
            for field, requirements in schema.items():
                if requirements.get('required', False) and field not in config:
                    errors.append(f"Missing required field: {field}")
                elif field in config:
                    value = config[field]
                    field_type = requirements.get('type')
                    
                    if field_type and not isinstance(value, eval(field_type)):
                        errors.append(f"Field {field} should be {field_type}, got {type(value).__name__}")
                    
                    if 'min' in requirements and value < requirements['min']:
                        errors.append(f"Field {field} value {value} is below minimum {requirements['min']}")
                    
                    if 'max' in requirements and value > requirements['max']:
                        errors.append(f"Field {field} value {value} is above maximum {requirements['max']}")
            
            return {
                'valid': len(errors) == 0,
                'errors': errors
            }
        except Exception as e:
            raise AnsibleFilterError(f"Error validating configuration: {str(e)}")
    
    def extract_version(self, version_string, format='semantic'):
        """Extract version components from version string"""
        try:
            if format == 'semantic':
                match = re.match(r'^v?(\d+)\.(\d+)\.(\d+)(?:-(.+))?(?:\+(.+))?$', version_string)
                if match:
                    return {
                        'major': int(match.group(1)),
                        'minor': int(match.group(2)),
                        'patch': int(match.group(3)),
                        'prerelease': match.group(4),
                        'build': match.group(5),
                        'full': version_string
                    }
            
            # Fallback to simple numeric extraction
            numbers = re.findall(r'\d+', version_string)
            return {
                'numbers': [int(n) for n in numbers],
                'full': version_string
            }
        except Exception as e:
            raise AnsibleFilterError(f"Error extracting version: {str(e)}")
    
    def sort_by_metric(self, servers, metric_path, reverse=False):
        """Sort servers by nested metric value"""
        try:
            def get_nested_value(obj, path):
                keys = path.split('.')
                for key in keys:
                    obj = obj.get(key, 0)
                return obj
            
            return sorted(servers, 
                         key=lambda x: get_nested_value(x, metric_path), 
                         reverse=reverse)
        except Exception as e:
            raise AnsibleFilterError(f"Error sorting by metric: {str(e)}")
    
    def group_by_threshold(self, servers, metric_path, thresholds):
        """Group servers by threshold levels"""
        try:
            def get_nested_value(obj, path):
                keys = path.split('.')
                for key in keys:
                    obj = obj.get(key, 0)
                return obj
            
            groups = {'critical': [], 'warning': [], 'normal': []}
            
            for server in servers:
                value = get_nested_value(server, metric_path)
                if value > thresholds.get('critical', 90):
                    groups['critical'].append(server)
                elif value > thresholds.get('warning', 70):
                    groups['warning'].append(server)
                else:
                    groups['normal'].append(server)
            
            return groups
        except Exception as e:
            raise AnsibleFilterError(f"Error grouping by threshold: {str(e)}")
    
    def calculate_score(self, metrics, weights=None):
        """Calculate weighted score from metrics"""
        try:
            if weights is None:
                weights = {k: 1.0 for k in metrics.keys()}
            
            total_score = 0
            total_weight = 0
            
            for metric, value in metrics.items():
                weight = weights.get(metric, 1.0)
                # Invert percentage metrics (lower is better)
                score = max(0, 100 - value) if isinstance(value, (int, float)) else 0
                total_score += score * weight
                total_weight += weight
            
            return round(total_score / total_weight if total_weight > 0 else 0, 2)
        except Exception as e:
            raise AnsibleFilterError(f"Error calculating score: {str(e)}")
EOF
```

Create custom filters test template:

```jinja2
# Create templates/custom-filters.j2
cat > templates/custom-filters.j2 << 'EOF'
{# Custom Filters Demonstration Template #}
{# {{ ansible_managed }} #}

{# ========================================= #}
{# CUSTOM FILTER DEMONSTRATIONS             #}
{# ========================================= #}

Server Uptime Analysis:
{% for server in (servers.web_servers + servers.database_servers) %}
{{ server.name }}: Last seen {{ server.last_updated | calculate_uptime }} ago
{% endfor %}

Storage Information:
{% for server in (servers.web_servers + servers.database_servers) %}
{{ server.name }}: {{ (server.disk_gb * 1024 * 1024 * 1024) | format_bytes }}
{% endfor %}

Network Analysis:
{% for server in (servers.web_servers + servers.database_servers) %}
{% set net_info = server.ip | network_info %}
{{ server.name }} ({{ server.ip }}):
  - Network Class: {{ net_info.class }}
  - Private IP: {{ 'Yes' if net_info.private else 'No' }}
  - Network: {{ net_info.network }}
{% endfor %}

Health Status Assessment:
{% for server in (servers.web_servers + servers.database_servers) %}
{% set health = server.metrics | health_status(monitoring.thresholds) %}
{{ server.name }}:
  Status: {{ health.status | upper }}
  Score: {{ health.score }}/100
  {% if health.issues %}
  Issues:
  {% for issue in health.issues %}
    - {{ issue }}
  {% endfor %}
  {% endif %}
{% endfor %}

Version Information:
{% set version_info = applications.web_app.version | extract_version %}
Application Version Analysis:
  Full Version: {{ version_info.full }}
  Major: {{ version_info.major }}
  Minor: {{ version_info.minor }}
  Patch: {{ version_info.patch }}

Server Performance Ranking:
{% for server in (servers.web_servers + servers.database_servers) | sort_by_metric('metrics.cpu_usage', reverse=true) %}
{{ loop.index }}. {{ server.name }}: {{ server.metrics.cpu_usage }}% CPU
{% endfor %}

Servers Grouped by CPU Usage:
{% set cpu_groups = (servers.web_servers + servers.database_servers) | group_by_threshold('metrics.cpu_usage', monitoring.thresholds.cpu) %}
Critical (>{{ monitoring.thresholds.cpu.critical }}%): {{ cpu_groups.critical | map(attribute='name') | join(', ') or 'None' }}
Warning (>{{ monitoring.thresholds.cpu.warning }}%): {{ cpu_groups.warning | map(attribute='name') | join(', ') or 'None' }}
Normal: {{ cpu_groups.normal | map(attribute='name') | join(', ') or 'None' }}

Performance Scores:
{% for server in (servers.web_servers + servers.database_servers) %}
{{ server.name }}: {{ server.metrics | calculate_score({'cpu_usage': 0.4, 'memory_usage': 0.4, 'disk_usage': 0.2}) }}/100
{% endfor %}

{# Configuration Validation Example #}
{% set config_schema = {
    'port': {'required': true, 'type': 'int', 'min': 1, 'max': 65535},
    'ssl_enabled': {'required': true, 'type': 'bool'},
    'version': {'required': true, 'type': 'str'}
} %}

{% set validation_result = applications.web_app | validate_config(config_schema) %}
Configuration Validation:
  Valid: {{ 'Yes' if validation_result.valid else 'No' }}
  {% if validation_result.errors %}
  Errors:
  {% for error in validation_result.errors %}
    - {{ error }}
  {% endfor %}
  {% endif %}

{# Password Generation Example #}
Generated Passwords:
  Low Complexity: {{ '' | generate_password(8, 'low') }}
  Medium Complexity: {{ '' | generate_password(12, 'medium') }}
  High Complexity: {{ '' | generate_password(16, 'high') }}
EOF
```

Create custom filters test:

```yaml
# Create test-custom-filters.yml
cat > test-custom-filters.yml << 'EOF'
---
- name: Test custom Jinja2 filters
  hosts: test_servers
  gather_facts: yes
  vars_files:
    - test-data.yml
  
  tasks:
    - name: Generate custom filters report
      template:
        src: templates/custom-filters.j2
        dest: /tmp/custom-filters-report.txt
        mode: '0644'
    
    - name: Test individual custom filters
      debug:
        msg: |
          Custom Filter Tests:
          - Uptime: {{ servers.web_servers[0].last_updated | calculate_uptime }}
          - Storage: {{ (servers.web_servers[0].disk_gb * 1024 * 1024 * 1024) | format_bytes }}
          - Network: {{ servers.web_servers[0].ip | network_info }}
          - Health: {{ servers.web_servers[0].metrics | health_status(monitoring.thresholds) }}
          - Version: {{ applications.web_app.version | extract_version }}
EOF

# Run the custom filters test
ansible-playbook -i inventory.ini test-custom-filters.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Check generated reports
echo "=== Advanced Expressions Report ==="
head -50 /tmp/advanced-expressions-report.txt 2>/dev/null || echo "Report not generated"

echo "=== Control Structures Report ==="
head -30 /tmp/control-structures-report.txt 2>/dev/null || echo "Report not generated"

echo "=== Custom Filters Report ==="
head -30 /tmp/custom-filters-report.txt 2>/dev/null || echo "Report not generated"

# Test filter plugins
echo "=== Filter Plugins ==="
ls -la filter_plugins/
```

### 2. Discussion Points
- How do you handle complex data transformations in your current templates?
- What custom filters would be most useful in your environment?
- How do you manage template complexity while maintaining readability?
- What are the performance implications of complex Jinja2 expressions?

### 3. Clean Up
```bash
# Keep files for next exercises
# rm -rf templates/ filter_plugins/ /tmp/*-report.txt
```

## Key Takeaways
- Advanced Jinja2 expressions enable sophisticated data processing
- Control structures provide powerful logic capabilities in templates
- Custom filters extend Jinja2 functionality for specialized use cases
- Proper error handling prevents template failures
- Complex expressions should be balanced with template readability
- Performance considerations are important for large datasets

## Next Steps
Proceed to Lab 6.2: Dynamic Configuration Generation

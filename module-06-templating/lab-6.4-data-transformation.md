# Lab 6.4: Data Transformation and Processing

## Objective
Master advanced data manipulation and transformation techniques using Jinja2 for complex enterprise data processing scenarios.

## Duration
25 minutes

## Prerequisites
- Completed Labs 6.1-6.3
- Understanding of data structures and transformation concepts
- Knowledge of enterprise data integration patterns

## Lab Setup

```bash
cd ~/ansible-labs/module-06
mkdir -p lab-6.4
cd lab-6.4

# Create inventory for testing
cat > inventory.ini << 'EOF'
[data_processors]
processor1 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Complex Data Structure Transformation (10 minutes)

### Task: Transform complex nested data structures for different output formats

Create comprehensive test data:

```yaml
# Create complex-data.yml
cat > complex-data.yml << 'EOF'
---
# Complex Enterprise Data for Transformation

# Raw server inventory data
raw_server_data:
  - hostname: "web-prod-01"
    ip_address: "192.168.1.10"
    environment: "production"
    role: "web"
    specs:
      cpu: 8
      memory: 32
      disk: 500
    services:
      - name: "nginx"
        port: 80
        status: "running"
        config:
          worker_processes: 4
          worker_connections: 1024
      - name: "php-fpm"
        port: 9000
        status: "running"
        config:
          pm_max_children: 50
          pm_start_servers: 5
    monitoring:
      enabled: true
      metrics:
        cpu_usage: 65.5
        memory_usage: 78.2
        disk_usage: 45.8
        network_in: 1024000
        network_out: 2048000
    tags:
      - "production"
      - "web-tier"
      - "critical"
    last_updated: "2024-01-15T10:30:00Z"
    
  - hostname: "web-prod-02"
    ip_address: "192.168.1.11"
    environment: "production"
    role: "web"
    specs:
      cpu: 8
      memory: 32
      disk: 500
    services:
      - name: "nginx"
        port: 80
        status: "running"
        config:
          worker_processes: 4
          worker_connections: 1024
      - name: "php-fpm"
        port: 9000
        status: "running"
        config:
          pm_max_children: 50
          pm_start_servers: 5
    monitoring:
      enabled: true
      metrics:
        cpu_usage: 72.1
        memory_usage: 82.5
        disk_usage: 52.3
        network_in: 1536000
        network_out: 3072000
    tags:
      - "production"
      - "web-tier"
      - "critical"
    last_updated: "2024-01-15T11:15:00Z"

  - hostname: "db-prod-01"
    ip_address: "192.168.1.20"
    environment: "production"
    role: "database"
    specs:
      cpu: 16
      memory: 128
      disk: 2000
    services:
      - name: "postgresql"
        port: 5432
        status: "running"
        config:
          max_connections: 200
          shared_buffers: "4GB"
          effective_cache_size: "12GB"
      - name: "pgbouncer"
        port: 6432
        status: "running"
        config:
          pool_size: 25
          max_client_conn: 100
    monitoring:
      enabled: true
      metrics:
        cpu_usage: 45.2
        memory_usage: 68.9
        disk_usage: 73.1
        network_in: 512000
        network_out: 1024000
        db_connections: 85
        db_queries_per_sec: 1250
    tags:
      - "production"
      - "database-tier"
      - "critical"
      - "backup-enabled"
    last_updated: "2024-01-15T09:45:00Z"

# Application deployment data
deployment_data:
  applications:
    - name: "web-frontend"
      version: "2.1.0"
      build: "20240115-1430"
      environment: "production"
      instances:
        - server: "web-prod-01"
          port: 8080
          status: "healthy"
          response_time: 150
          error_rate: 0.02
        - server: "web-prod-02"
          port: 8080
          status: "healthy"
          response_time: 175
          error_rate: 0.01
      dependencies:
        - service: "database"
          type: "postgresql"
          connection: "primary"
        - service: "cache"
          type: "redis"
          connection: "cluster"
      configuration:
        database_pool_size: 20
        cache_ttl: 3600
        session_timeout: 1800
        max_request_size: "10MB"
      
    - name: "api-backend"
      version: "1.8.5"
      build: "20240115-1200"
      environment: "production"
      instances:
        - server: "web-prod-01"
          port: 8081
          status: "healthy"
          response_time: 95
          error_rate: 0.005
        - server: "web-prod-02"
          port: 8081
          status: "degraded"
          response_time: 250
          error_rate: 0.03
      dependencies:
        - service: "database"
          type: "postgresql"
          connection: "primary"
        - service: "message_queue"
          type: "rabbitmq"
          connection: "cluster"
      configuration:
        database_pool_size: 15
        queue_workers: 5
        batch_size: 100
        timeout: 30

# Network topology data
network_topology:
  subnets:
    - name: "web-tier"
      cidr: "192.168.1.0/24"
      gateway: "192.168.1.1"
      dns: ["8.8.8.8", "8.8.4.4"]
      servers: ["web-prod-01", "web-prod-02"]
      security_groups:
        - name: "web-sg"
          rules:
            - protocol: "tcp"
              port: 80
              source: "0.0.0.0/0"
            - protocol: "tcp"
              port: 443
              source: "0.0.0.0/0"
            - protocol: "tcp"
              port: 22
              source: "10.0.0.0/8"
    
    - name: "database-tier"
      cidr: "192.168.2.0/24"
      gateway: "192.168.2.1"
      dns: ["8.8.8.8", "8.8.4.4"]
      servers: ["db-prod-01"]
      security_groups:
        - name: "db-sg"
          rules:
            - protocol: "tcp"
              port: 5432
              source: "192.168.1.0/24"
            - protocol: "tcp"
              port: 22
              source: "10.0.0.0/8"

# Monitoring and metrics data
monitoring_data:
  time_series:
    - timestamp: "2024-01-15T10:00:00Z"
      metrics:
        web-prod-01:
          cpu: 60.5
          memory: 75.2
          disk: 44.1
          requests_per_sec: 125
        web-prod-02:
          cpu: 68.3
          memory: 80.1
          disk: 50.8
          requests_per_sec: 142
        db-prod-01:
          cpu: 42.1
          memory: 65.5
          disk: 71.2
          queries_per_sec: 1180
    
    - timestamp: "2024-01-15T10:15:00Z"
      metrics:
        web-prod-01:
          cpu: 65.5
          memory: 78.2
          disk: 45.8
          requests_per_sec: 135
        web-prod-02:
          cpu: 72.1
          memory: 82.5
          disk: 52.3
          requests_per_sec: 158
        db-prod-01:
          cpu: 45.2
          memory: 68.9
          disk: 73.1
          queries_per_sec: 1250

# Configuration templates data
config_templates:
  nginx:
    base_config:
      worker_processes: "auto"
      worker_connections: 1024
      keepalive_timeout: 65
    environments:
      development:
        worker_processes: 2
        worker_connections: 512
        error_log_level: "debug"
      production:
        worker_processes: 8
        worker_connections: 2048
        error_log_level: "error"
    
  postgresql:
    base_config:
      max_connections: 100
      shared_buffers: "256MB"
      effective_cache_size: "1GB"
    environments:
      development:
        max_connections: 50
        shared_buffers: "128MB"
        log_statement: "all"
      production:
        max_connections: 200
        shared_buffers: "4GB"
        log_statement: "none"
EOF
```

Create data transformation templates:

```jinja2
# Create templates/data-transformations.j2
mkdir -p templates
cat > templates/data-transformations.j2 << 'EOF'
{# Data Transformation Template #}
{# {{ ansible_managed }} #}

{# ========================================= #}
{# SERVER INVENTORY TRANSFORMATIONS         #}
{# ========================================= #}

# Server Inventory Summary
{% set servers_by_role = raw_server_data | groupby('role') %}
{% set total_servers = raw_server_data | length %}
{% set total_cpu = raw_server_data | sum(attribute='specs.cpu') %}
{% set total_memory = raw_server_data | sum(attribute='specs.memory') %}
{% set total_disk = raw_server_data | sum(attribute='specs.disk') %}

INVENTORY_SUMMARY:
  total_servers: {{ total_servers }}
  total_cpu_cores: {{ total_cpu }}
  total_memory_gb: {{ total_memory }}
  total_disk_gb: {{ total_disk }}
  
  servers_by_role:
{% for role, servers in servers_by_role %}
    {{ role }}:
      count: {{ servers | length }}
      cpu_cores: {{ servers | sum(attribute='specs.cpu') }}
      memory_gb: {{ servers | sum(attribute='specs.memory') }}
      servers: {{ servers | map(attribute='hostname') | list | to_json }}
{% endfor %}

# Server Health Matrix
SERVER_HEALTH_MATRIX:
{% for server in raw_server_data %}
  {{ server.hostname }}:
    role: {{ server.role }}
    environment: {{ server.environment }}
    health_score: {{ (100 - ((server.monitoring.metrics.cpu_usage + server.monitoring.metrics.memory_usage + server.monitoring.metrics.disk_usage) / 3)) | round(1) }}
    status: {{ 'critical' if server.monitoring.metrics.cpu_usage > 90 or server.monitoring.metrics.memory_usage > 90 
                else 'warning' if server.monitoring.metrics.cpu_usage > 75 or server.monitoring.metrics.memory_usage > 75 
                else 'healthy' }}
    services_count: {{ server.services | length }}
    tags: {{ server.tags | to_json }}
{% endfor %}

# Service Distribution Analysis
SERVICE_DISTRIBUTION:
{% set all_services = raw_server_data | map(attribute='services') | flatten %}
{% set services_by_name = all_services | groupby('name') %}
{% for service_name, service_instances in services_by_name %}
  {{ service_name }}:
    total_instances: {{ service_instances | length }}
    servers: {{ service_instances | map(attribute='port') | list | length }}
    average_port: {{ (service_instances | sum(attribute='port')) / (service_instances | length) }}
    configurations:
{% for instance in service_instances %}
      - server: {{ raw_server_data | selectattr('services', 'contains', instance) | map(attribute='hostname') | first }}
        port: {{ instance.port }}
        status: {{ instance.status }}
        config: {{ instance.config | to_json }}
{% endfor %}
{% endfor %}

{# ========================================= #}
{# APPLICATION DEPLOYMENT TRANSFORMATIONS   #}
{# ========================================= #}

# Application Deployment Matrix
DEPLOYMENT_MATRIX:
{% for app in deployment_data.applications %}
  {{ app.name }}:
    version: {{ app.version }}
    build: {{ app.build }}
    environment: {{ app.environment }}
    
    # Instance health analysis
    instances:
      total: {{ app.instances | length }}
      healthy: {{ app.instances | selectattr('status', 'equalto', 'healthy') | list | length }}
      degraded: {{ app.instances | selectattr('status', 'equalto', 'degraded') | list | length }}
      unhealthy: {{ app.instances | selectattr('status', 'equalto', 'unhealthy') | list | length }}
    
    # Performance metrics
    performance:
      avg_response_time: {{ (app.instances | sum(attribute='response_time')) / (app.instances | length) | round(1) }}ms
      max_response_time: {{ app.instances | map(attribute='response_time') | max }}ms
      min_response_time: {{ app.instances | map(attribute='response_time') | min }}ms
      avg_error_rate: {{ ((app.instances | sum(attribute='error_rate')) / (app.instances | length) * 100) | round(3) }}%
      total_error_rate: {{ (app.instances | sum(attribute='error_rate') * 100) | round(3) }}%
    
    # Dependency mapping
    dependencies:
{% for dep in app.dependencies %}
      - service: {{ dep.service }}
        type: {{ dep.type }}
        connection: {{ dep.connection }}
{% endfor %}
    
    # Configuration summary
    configuration: {{ app.configuration | to_json }}
    
    # Instance details
    instance_details:
{% for instance in app.instances %}
      - server: {{ instance.server }}
        port: {{ instance.port }}
        status: {{ instance.status }}
        health_score: {{ (100 - (instance.error_rate * 1000 + instance.response_time / 10)) | round(1) }}
        performance_tier: {{ 'excellent' if instance.response_time < 100 and instance.error_rate < 0.01
                             else 'good' if instance.response_time < 200 and instance.error_rate < 0.02
                             else 'acceptable' if instance.response_time < 300 and instance.error_rate < 0.05
                             else 'poor' }}
{% endfor %}
{% endfor %}

{# ========================================= #}
{# NETWORK TOPOLOGY TRANSFORMATIONS         #}
{# ========================================= #}

# Network Topology Analysis
NETWORK_TOPOLOGY:
  total_subnets: {{ network_topology.subnets | length }}
  total_servers: {{ network_topology.subnets | sum(attribute='servers') | length }}
  
  subnet_analysis:
{% for subnet in network_topology.subnets %}
    {{ subnet.name }}:
      cidr: {{ subnet.cidr }}
      gateway: {{ subnet.gateway }}
      dns_servers: {{ subnet.dns | to_json }}
      server_count: {{ subnet.servers | length }}
      servers: {{ subnet.servers | to_json }}
      
      # Calculate network utilization (assuming /24 subnets)
      {% set subnet_size = 254 %}  {# /24 subnet has 254 usable IPs #}
      utilization: {{ ((subnet.servers | length) / subnet_size * 100) | round(1) }}%
      available_ips: {{ subnet_size - (subnet.servers | length) }}
      
      security_groups:
{% for sg in subnet.security_groups %}
        {{ sg.name }}:
          rules_count: {{ sg.rules | length }}
          open_ports: {{ sg.rules | map(attribute='port') | list | to_json }}
          protocols: {{ sg.rules | map(attribute='protocol') | unique | list | to_json }}
{% endfor %}
{% endfor %}

# Security Analysis
SECURITY_ANALYSIS:
  total_security_groups: {{ network_topology.subnets | sum(attribute='security_groups') | length }}
  
  port_exposure_analysis:
{% set all_rules = network_topology.subnets | map(attribute='security_groups') | flatten | map(attribute='rules') | flatten %}
{% set rules_by_port = all_rules | groupby('port') %}
{% for port, rules in rules_by_port %}
    port_{{ port }}:
      protocol: {{ rules | map(attribute='protocol') | unique | first }}
      exposure_count: {{ rules | length }}
      sources: {{ rules | map(attribute='source') | unique | list | to_json }}
      risk_level: {{ 'high' if '0.0.0.0/0' in (rules | map(attribute='source') | list) and port in [22, 3389]
                     else 'medium' if '0.0.0.0/0' in (rules | map(attribute='source') | list)
                     else 'low' }}
{% endfor %}

{# ========================================= #}
{# TIME SERIES DATA TRANSFORMATIONS         #}
{# ========================================= #}

# Time Series Analysis
TIME_SERIES_ANALYSIS:
  data_points: {{ monitoring_data.time_series | length }}
  time_range:
    start: {{ monitoring_data.time_series | map(attribute='timestamp') | min }}
    end: {{ monitoring_data.time_series | map(attribute='timestamp') | max }}
  
  # Server performance trends
  server_trends:
{% set servers = monitoring_data.time_series[0].metrics.keys() %}
{% for server in servers %}
    {{ server }}:
      # Calculate trends between first and last data points
      {% set first_metrics = monitoring_data.time_series[0].metrics[server] %}
      {% set last_metrics = monitoring_data.time_series[-1].metrics[server] %}
      
      cpu_trend: {{ ((last_metrics.cpu - first_metrics.cpu) / first_metrics.cpu * 100) | round(2) }}%
      memory_trend: {{ ((last_metrics.memory - first_metrics.memory) / first_metrics.memory * 100) | round(2) }}%
      disk_trend: {{ ((last_metrics.disk - first_metrics.disk) / first_metrics.disk * 100) | round(2) }}%
      
      # Performance classification
      performance_class: {{ 'degrading' if (last_metrics.cpu - first_metrics.cpu) > 5 or (last_metrics.memory - first_metrics.memory) > 5
                           else 'improving' if (last_metrics.cpu - first_metrics.cpu) < -2 or (last_metrics.memory - first_metrics.memory) < -2
                           else 'stable' }}
      
      # Current status
      current_metrics:
        cpu: {{ last_metrics.cpu }}%
        memory: {{ last_metrics.memory }}%
        disk: {{ last_metrics.disk }}%
        {% if 'requests_per_sec' in last_metrics %}
        requests_per_sec: {{ last_metrics.requests_per_sec }}
        {% endif %}
        {% if 'queries_per_sec' in last_metrics %}
        queries_per_sec: {{ last_metrics.queries_per_sec }}
        {% endif %}
{% endfor %}

{# ========================================= #}
{# CONFIGURATION TEMPLATE TRANSFORMATIONS   #}
{# ========================================= #}

# Configuration Template Analysis
CONFIG_TEMPLATE_ANALYSIS:
{% for service, config in config_templates.items() %}
  {{ service }}:
    base_configuration: {{ config.base_config | to_json }}
    
    environment_variations:
{% for env, env_config in config.environments.items() %}
      {{ env }}:
        # Merge base config with environment-specific overrides
        {% set merged_config = config.base_config | combine(env_config) %}
        merged_configuration: {{ merged_config | to_json }}
        
        # Calculate configuration differences from base
        overrides:
{% for key, value in env_config.items() %}
          {{ key }}: {{ value }} (was: {{ config.base_config.get(key, 'not_set') }})
{% endfor %}
        
        # Configuration complexity score
        complexity_score: {{ (merged_config.keys() | length) * (env_config.keys() | length) }}
{% endfor %}
{% endfor %}

{# ========================================= #}
{# CROSS-REFERENCE ANALYSIS                 #}
{# ========================================= #}

# Cross-Reference Analysis
CROSS_REFERENCE_ANALYSIS:
  # Map applications to servers
  application_server_mapping:
{% for app in deployment_data.applications %}
    {{ app.name }}:
{% for instance in app.instances %}
      {% set server_data = raw_server_data | selectattr('hostname', 'equalto', instance.server) | first %}
      - server: {{ instance.server }}
        server_role: {{ server_data.role }}
        server_specs: {{ server_data.specs | to_json }}
        instance_port: {{ instance.port }}
        instance_status: {{ instance.status }}
        resource_utilization:
          cpu_available: {{ server_data.specs.cpu - (server_data.monitoring.metrics.cpu_usage / 100 * server_data.specs.cpu) | round(1) }}
          memory_available: {{ server_data.specs.memory - (server_data.monitoring.metrics.memory_usage / 100 * server_data.specs.memory) | round(1) }}GB
{% endfor %}
{% endfor %}
  
  # Resource allocation efficiency
  resource_efficiency:
    total_allocated_cpu: {{ raw_server_data | sum(attribute='specs.cpu') }}
    total_used_cpu: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / 100 * (raw_server_data | sum(attribute='specs.cpu'))) | round(1) }}
    cpu_efficiency: {{ ((raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum) / (raw_server_data | length)) | round(1) }}%
    
    total_allocated_memory: {{ raw_server_data | sum(attribute='specs.memory') }}GB
    total_used_memory: {{ ((raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / 100) * (raw_server_data | sum(attribute='specs.memory'))) | round(1) }}GB
    memory_efficiency: {{ ((raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum) / (raw_server_data | length)) | round(1) }}%

{# ========================================= #}
{# SUMMARY AND RECOMMENDATIONS              #}
{# ========================================= #}

# Summary and Recommendations
SUMMARY_AND_RECOMMENDATIONS:
  infrastructure_summary:
    total_servers: {{ raw_server_data | length }}
    total_applications: {{ deployment_data.applications | length }}
    total_subnets: {{ network_topology.subnets | length }}
    monitoring_data_points: {{ monitoring_data.time_series | length }}
  
  health_summary:
    healthy_servers: {{ raw_server_data | selectattr('monitoring.metrics.cpu_usage', 'lt', 75) | selectattr('monitoring.metrics.memory_usage', 'lt', 75) | list | length }}
    warning_servers: {{ raw_server_data | selectattr('monitoring.metrics.cpu_usage', 'ge', 75) | selectattr('monitoring.metrics.cpu_usage', 'lt', 90) | list | length + raw_server_data | selectattr('monitoring.metrics.memory_usage', 'ge', 75) | selectattr('monitoring.metrics.memory_usage', 'lt', 90) | list | length }}
    critical_servers: {{ raw_server_data | selectattr('monitoring.metrics.cpu_usage', 'ge', 90) | list | length + raw_server_data | selectattr('monitoring.metrics.memory_usage', 'ge', 90) | list | length }}
  
  recommendations:
{% if raw_server_data | selectattr('monitoring.metrics.cpu_usage', 'ge', 90) | list | length > 0 %}
    - "CRITICAL: {{ raw_server_data | selectattr('monitoring.metrics.cpu_usage', 'ge', 90) | list | length }} server(s) have CPU usage above 90%"
{% endif %}
{% if raw_server_data | selectattr('monitoring.metrics.memory_usage', 'ge', 90) | list | length > 0 %}
    - "CRITICAL: {{ raw_server_data | selectattr('monitoring.metrics.memory_usage', 'ge', 90) | list | length }} server(s) have memory usage above 90%"
{% endif %}
{% if deployment_data.applications | selectattr('instances', 'selectattr', 'status', 'equalto', 'degraded') | list | length > 0 %}
    - "WARNING: Some application instances are in degraded state"
{% endif %}
{% set avg_cpu = (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum) / (raw_server_data | length) %}
{% if avg_cpu < 50 %}
    - "INFO: Average CPU utilization is {{ avg_cpu | round(1) }}% - consider consolidating workloads"
{% elif avg_cpu > 80 %}
    - "WARNING: Average CPU utilization is {{ avg_cpu | round(1) }}% - consider scaling out"
{% endif %}
EOF
```

Create JSON transformation template:

```jinja2
# Create templates/json-transformation.j2
cat > templates/json-transformation.j2 << 'EOF'
{
  "_metadata": {
    "generated": "{{ ansible_date_time.iso8601 }}",
    "template": "json-transformation.j2",
    "ansible_version": "{{ ansible_version.full }}",
    "transformation_type": "enterprise_data_processing"
  },
  
  "infrastructure": {
    "servers": [
{% for server in raw_server_data %}
      {
        "hostname": "{{ server.hostname }}",
        "ip_address": "{{ server.ip_address }}",
        "environment": "{{ server.environment }}",
        "role": "{{ server.role }}",
        "specifications": {
          "cpu_cores": {{ server.specs.cpu }},
          "memory_gb": {{ server.specs.memory }},
          "disk_gb": {{ server.specs.disk }},
          "total_capacity_score": {{ (server.specs.cpu * 10 + server.specs.memory + server.specs.disk / 10) | int }}
        },
        "services": [
{% for service in server.services %}
          {
            "name": "{{ service.name }}",
            "port": {{ service.port }},
            "status": "{{ service.status }}",
            "configuration": {{ service.config | to_json }},
            "health_check": "{{ service.name }}:{{ service.port }}"
          }{% if not loop.last %},{% endif %}
{% endfor %}
        ],
        "monitoring": {
          "enabled": {{ server.monitoring.enabled | lower }},
          "metrics": {
            "cpu_usage_percent": {{ server.monitoring.metrics.cpu_usage }},
            "memory_usage_percent": {{ server.monitoring.metrics.memory_usage }},
            "disk_usage_percent": {{ server.monitoring.metrics.disk_usage }},
            {% if server.monitoring.metrics.network_in is defined %}
            "network_in_bytes": {{ server.monitoring.metrics.network_in }},
            "network_out_bytes": {{ server.monitoring.metrics.network_out }},
            {% endif %}
            {% if server.monitoring.metrics.db_connections is defined %}
            "database_connections": {{ server.monitoring.metrics.db_connections }},
            "database_queries_per_second": {{ server.monitoring.metrics.db_queries_per_sec }},
            {% endif %}
            "health_score": {{ (100 - ((server.monitoring.metrics.cpu_usage + server.monitoring.metrics.memory_usage + server.monitoring.metrics.disk_usage) / 3)) | round(1) }},
            "status": "{{ 'critical' if server.monitoring.metrics.cpu_usage > 90 or server.monitoring.metrics.memory_usage > 90 
                          else 'warning' if server.monitoring.metrics.cpu_usage > 75 or server.monitoring.metrics.memory_usage > 75 
                          else 'healthy' }}"
          }
        },
        "tags": {{ server.tags | to_json }},
        "last_updated": "{{ server.last_updated }}"
      }{% if not loop.last %},{% endif %}
{% endfor %}
    ],
    
    "summary": {
      "total_servers": {{ raw_server_data | length }},
      "servers_by_role": {
{% set servers_by_role = raw_server_data | groupby('role') %}
{% for role, servers in servers_by_role %}
        "{{ role }}": {
          "count": {{ servers | length }},
          "total_cpu_cores": {{ servers | sum(attribute='specs.cpu') }},
          "total_memory_gb": {{ servers | sum(attribute='specs.memory') }},
          "total_disk_gb": {{ servers | sum(attribute='specs.disk') }},
          "average_cpu_usage": {{ (servers | sum(attribute='monitoring.metrics.cpu_usage')) / (servers | length) | round(1) }},
          "average_memory_usage": {{ (servers | sum(attribute='monitoring.metrics.memory_usage')) / (servers | length) | round(1) }}
        }{% if not loop.last %},{% endif %}
{% endfor %}
      },
      "resource_totals": {
        "cpu_cores": {{ raw_server_data | sum(attribute='specs.cpu') }},
        "memory_gb": {{ raw_server_data | sum(attribute='specs.memory') }},
        "disk_gb": {{ raw_server_data | sum(attribute='specs.disk') }},
        "estimated_cost_per_month": {{ (raw_server_data | sum(attribute='specs.cpu') * 50 + raw_server_data | sum(attribute='specs.memory') * 10 + raw_server_data | sum(attribute='specs.disk') * 0.1) | int }}
      }
    }
  },
  
  "applications": {
    "deployments": [
{% for app in deployment_data.applications %}
      {
        "name": "{{ app.name }}",
        "version": "{{ app.version }}",
        "build": "{{ app.build }}",
        "environment": "{{ app.environment }}",
        "instances": [
{% for instance in app.instances %}
          {
            "server": "{{ instance.server }}",
            "port": {{ instance.port }},
            "status": "{{ instance.status }}",
            "performance": {
              "response_time_ms": {{ instance.response_time }},
              "error_rate_percent": {{ (instance.error_rate * 100) | round(3) }},
              "health_score": {{ (100 - (instance.error_rate * 1000 + instance.response_time / 10)) | round(1) }},
              "performance_tier": "{{ 'excellent' if instance.response_time < 100 and instance.error_rate < 0.01
                                     else 'good' if instance.response_time < 200 and instance.error_rate < 0.02
                                     else 'acceptable' if instance.response_time < 300 and instance.error_rate < 0.05
                                     else 'poor' }}"
            }
          }{% if not loop.last %},{% endif %}
{% endfor %}
        ],
        "dependencies": {{ app.dependencies | to_json }},
        "configuration": {{ app.configuration | to_json }},
        "metrics": {
          "total_instances": {{ app.instances | length }},
          "healthy_instances": {{ app.instances | selectattr('status', 'equalto', 'healthy') | list | length }},
          "degraded_instances": {{ app.instances | selectattr('status', 'equalto', 'degraded') | list | length }},
          "average_response_time": {{ (app.instances | sum(attribute='response_time')) / (app.instances | length) | round(1) }},
          "total_error_rate": {{ (app.instances | sum(attribute='error_rate') * 100) | round(3) }},
          "availability_percent": {{ ((app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / (app.instances | length) * 100) | round(1) }}
        }
      }{% if not loop.last %},{% endif %}
{% endfor %}
    ]
  },
  
  "network": {
    "topology": {
      "subnets": [
{% for subnet in network_topology.subnets %}
        {
          "name": "{{ subnet.name }}",
          "cidr": "{{ subnet.cidr }}",
          "gateway": "{{ subnet.gateway }}",
          "dns_servers": {{ subnet.dns | to_json }},
          "servers": {{ subnet.servers | to_json }},
          "utilization": {
            "total_ips": 254,
            "used_ips": {{ subnet.servers | length }},
            "utilization_percent": {{ ((subnet.servers | length) / 254 * 100) | round(1) }},
            "available_ips": {{ 254 - (subnet.servers | length) }}
          },
          "security_groups": [
{% for sg in subnet.security_groups %}
            {
              "name": "{{ sg.name }}",
              "rules": [
{% for rule in sg.rules %}
                {
                  "protocol": "{{ rule.protocol }}",
                  "port": {{ rule.port }},
                  "source": "{{ rule.source }}",
                  "risk_level": "{{ 'high' if rule.source == '0.0.0.0/0' and rule.port in [22, 3389]
                                   else 'medium' if rule.source == '0.0.0.0/0'
                                   else 'low' }}"
                }{% if not loop.last %},{% endif %}
{% endfor %}
              ]
            }{% if not loop.last %},{% endif %}
{% endfor %}
          ]
        }{% if not loop.last %},{% endif %}
{% endfor %}
      ]
    }
  },
  
  "monitoring": {
    "time_series": [
{% for datapoint in monitoring_data.time_series %}
      {
        "timestamp": "{{ datapoint.timestamp }}",
        "metrics": {
{% for server, metrics in datapoint.metrics.items() %}
          "{{ server }}": {{ metrics | to_json }}{% if not loop.last %},{% endif %}
{% endfor %}
        }
      }{% if not loop.last %},{% endif %}
{% endfor %}
    ],
    "analysis": {
{% set servers = monitoring_data.time_series[0].metrics.keys() %}
{% for server in servers %}
      "{{ server }}": {
        {% set first_metrics = monitoring_data.time_series[0].metrics[server] %}
        {% set last_metrics = monitoring_data.time_series[-1].metrics[server] %}
        "trend_analysis": {
          "cpu_trend_percent": {{ ((last_metrics.cpu - first_metrics.cpu) / first_metrics.cpu * 100) | round(2) }},
          "memory_trend_percent": {{ ((last_metrics.memory - first_metrics.memory) / first_metrics.memory * 100) | round(2) }},
          "disk_trend_percent": {{ ((last_metrics.disk - first_metrics.disk) / first_metrics.disk * 100) | round(2) }},
          "performance_classification": "{{ 'degrading' if (last_metrics.cpu - first_metrics.cpu) > 5 or (last_metrics.memory - first_metrics.memory) > 5
                                           else 'improving' if (last_metrics.cpu - first_metrics.cpu) < -2 or (last_metrics.memory - first_metrics.memory) < -2
                                           else 'stable' }}"
        },
        "current_status": {{ last_metrics | to_json }}
      }{% if not loop.last %},{% endif %}
{% endfor %}
    }
  }
}
EOF
```

Create test for data transformation:

```yaml
# Create test-data-transformation.yml
cat > test-data-transformation.yml << 'EOF'
---
- name: Test complex data structure transformation
  hosts: data_processors
  gather_facts: yes
  vars_files:
    - complex-data.yml
  
  tasks:
    - name: Generate comprehensive data transformation report
      template:
        src: templates/data-transformations.j2
        dest: /tmp/data-transformation-report.yml
        mode: '0644'
    
    - name: Generate JSON transformation output
      template:
        src: templates/json-transformation.j2
        dest: /tmp/data-transformation.json
        mode: '0644'
    
    - name: Validate JSON output
      shell: python3 -m json.tool /tmp/data-transformation.json
      register: json_validation
      changed_when: false
      failed_when: json_validation.rc != 0
    
    - name: Display transformation summary
      debug:
        msg: |
          Data Transformation Summary:
          
          Source Data:
          - Raw Servers: {{ raw_server_data | length }}
          - Applications: {{ deployment_data.applications | length }}
          - Network Subnets: {{ network_topology.subnets | length }}
          - Monitoring Data Points: {{ monitoring_data.time_series | length }}
          
          Transformations Applied:
          - Server inventory analysis and grouping
          - Application performance metrics calculation
          - Network topology and security analysis
          - Time series trend analysis
          - Cross-reference mapping
          - Resource efficiency calculations
          
          Generated Files:
          - YAML Report: /tmp/data-transformation-report.yml
          - JSON Output: /tmp/data-transformation.json
          - JSON Validation: {{ 'PASSED' if json_validation.rc == 0 else 'FAILED' }}
          
          Key Insights:
          - Total CPU Cores: {{ raw_server_data | sum(attribute='specs.cpu') }}
          - Total Memory: {{ raw_server_data | sum(attribute='specs.memory') }}GB
          - Average CPU Usage: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length) | round(1) }}%
          - Healthy Servers: {{ raw_server_data | selectattr('monitoring.metrics.cpu_usage', 'lt', 75) | selectattr('monitoring.metrics.memory_usage', 'lt', 75) | list | length }}
EOF

# Run data transformation test
ansible-playbook -i inventory.ini test-data-transformation.yml
```

## Exercise 2: Dynamic Data Processing and Aggregation (8 minutes)

### Task: Implement dynamic data processing with real-time aggregation

Create dynamic processing template:

```jinja2
# Create templates/dynamic-processing.j2
cat > templates/dynamic-processing.j2 << 'EOF'
{# Dynamic Data Processing Template #}
{# {{ ansible_managed }} #}

{# ========================================= #}
{# DYNAMIC AGGREGATION FUNCTIONS            #}
{# ========================================= #}

{# Macro for calculating weighted averages #}
{% macro weighted_average(items, value_attr, weight_attr) -%}
{% set total_weighted = items | sum(attribute=weight_attr) %}
{% if total_weighted > 0 %}
{{ ((items | map('extract', value_attr) | list | zip(items | map('extract', weight_attr) | list) | map('product') | sum) / total_weighted) | round(2) }}
{% else %}
0
{% endif %}
{%- endmacro %}

{# Macro for percentile calculations #}
{% macro percentile(values, p) -%}
{% set sorted_values = values | sort %}
{% set index = ((sorted_values | length - 1) * p / 100) | int %}
{{ sorted_values[index] }}
{%- endmacro %}

{# Macro for trend analysis #}
{% macro trend_analysis(current, previous, threshold=5) -%}
{% set change = ((current - previous) / previous * 100) if previous > 0 else 0 %}
{{ 'increasing' if change > threshold else 'decreasing' if change < -threshold else 'stable' }}
{%- endmacro %}

{# ========================================= #}
{# REAL-TIME METRICS PROCESSING             #}
{# ========================================= #}

# Real-Time Infrastructure Metrics
REAL_TIME_METRICS:
  timestamp: {{ ansible_date_time.iso8601 }}
  
  # Server performance aggregation
  server_performance:
    total_servers: {{ raw_server_data | length }}
    
    # CPU metrics
    cpu_metrics:
      average: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length) | round(1) }}%
      median: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list, 50) }}%
      p95: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list, 95) }}%
      p99: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list, 99) }}%
      min: {{ raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | min }}%
      max: {{ raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | max }}%
      
    # Memory metrics
    memory_metrics:
      average: {{ (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / raw_server_data | length) | round(1) }}%
      median: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list, 50) }}%
      p95: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list, 95) }}%
      p99: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list, 99) }}%
      min: {{ raw_server_data | map(attribute='monitoring.metrics.memory_usage') | min }}%
      max: {{ raw_server_data | map(attribute='monitoring.metrics.memory_usage') | max }}%
    
    # Disk metrics
    disk_metrics:
      average: {{ (raw_server_data | map(attribute='monitoring.metrics.disk_usage') | sum / raw_server_data | length) | round(1) }}%
      median: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.disk_usage') | list, 50) }}%
      p95: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.disk_usage') | list, 95) }}%
      p99: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.disk_usage') | list, 99) }}%
      min: {{ raw_server_data | map(attribute='monitoring.metrics.disk_usage') | min }}%
      max: {{ raw_server_data | map(attribute='monitoring.metrics.disk_usage') | max }}%

  # Application performance aggregation
  application_performance:
{% for app in deployment_data.applications %}
    {{ app.name }}:
      instance_count: {{ app.instances | length }}
      
      # Response time analysis
      response_time:
        average: {{ (app.instances | sum(attribute='response_time') / app.instances | length) | round(1) }}ms
        median: {{ percentile(app.instances | map(attribute='response_time') | list, 50) }}ms
        p95: {{ percentile(app.instances | map(attribute='response_time') | list, 95) }}ms
        p99: {{ percentile(app.instances | map(attribute='response_time') | list, 99) }}ms
        min: {{ app.instances | map(attribute='response_time') | min }}ms
        max: {{ app.instances | map(attribute='response_time') | max }}ms
        
      # Error rate analysis
      error_rate:
        average: {{ ((app.instances | sum(attribute='error_rate') / app.instances | length) * 100) | round(3) }}%
        total: {{ (app.instances | sum(attribute='error_rate') * 100) | round(3) }}%
        max: {{ (app.instances | map(attribute='error_rate') | max * 100) | round(3) }}%
        
      # Availability calculation
      availability:
        healthy_instances: {{ app.instances | selectattr('status', 'equalto', 'healthy') | list | length }}
        total_instances: {{ app.instances | length }}
        availability_percent: {{ ((app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / app.instances | length * 100) | round(1) }}%
        sla_compliance: {{ 'compliant' if ((app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / app.instances | length * 100) >= 99.5 
                          else 'at_risk' if ((app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / app.instances | length * 100) >= 99.0 
                          else 'non_compliant' }}
{% endfor %}

{# ========================================= #}
{# DYNAMIC THRESHOLD ANALYSIS               #}
{# ========================================= #}

# Dynamic Threshold Analysis
THRESHOLD_ANALYSIS:
  # Calculate dynamic thresholds based on historical data
  dynamic_thresholds:
    cpu:
      # Use 95th percentile as warning threshold
      warning: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list, 95) | round(1) }}%
      # Use 99th percentile as critical threshold
      critical: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list, 99) | round(1) }}%
      
    memory:
      warning: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list, 95) | round(1) }}%
      critical: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list, 99) | round(1) }}%
      
    disk:
      warning: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.disk_usage') | list, 95) | round(1) }}%
      critical: {{ percentile(raw_server_data | map(attribute='monitoring.metrics.disk_usage') | list, 99) | round(1) }}%

  # Alert analysis based on dynamic thresholds
  alert_analysis:
{% for server in raw_server_data %}
    {{ server.hostname }}:
      cpu_status: {{ 'critical' if server.monitoring.metrics.cpu_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list, 99)
                     else 'warning' if server.monitoring.metrics.cpu_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list, 95)
                     else 'normal' }}
      memory_status: {{ 'critical' if server.monitoring.metrics.memory_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list, 99)
                        else 'warning' if server.monitoring.metrics.memory_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list, 95)
                        else 'normal' }}
      disk_status: {{ 'critical' if server.monitoring.metrics.disk_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.disk_usage') | list, 99)
                      else 'warning' if server.monitoring.metrics.disk_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.disk_usage') | list, 95)
                      else 'normal' }}
      overall_status: {{ 'critical' if (server.monitoring.metrics.cpu_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list, 99) or 
                                       server.monitoring.metrics.memory_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list, 99) or
                                       server.monitoring.metrics.disk_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.disk_usage') | list, 99))
                         else 'warning' if (server.monitoring.metrics.cpu_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list, 95) or 
                                           server.monitoring.metrics.memory_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list, 95) or
                                           server.monitoring.metrics.disk_usage > percentile(raw_server_data | map(attribute='monitoring.metrics.disk_usage') | list, 95))
                         else 'normal' }}
{% endfor %}

{# ========================================= #}
{# CAPACITY PLANNING ANALYSIS               #}
{# ========================================= #}

# Capacity Planning Analysis
CAPACITY_PLANNING:
  current_utilization:
    cpu:
      total_cores: {{ raw_server_data | sum(attribute='specs.cpu') }}
      used_cores: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / 100 * raw_server_data | sum(attribute='specs.cpu') / raw_server_data | length) | round(1) }}
      utilization_percent: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length) | round(1) }}%
      
    memory:
      total_gb: {{ raw_server_data | sum(attribute='specs.memory') }}
      used_gb: {{ (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / 100 * raw_server_data | sum(attribute='specs.memory') / raw_server_data | length) | round(1) }}
      utilization_percent: {{ (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / raw_server_data | length) | round(1) }}%
      
    disk:
      total_gb: {{ raw_server_data | sum(attribute='specs.disk') }}
      used_gb: {{ (raw_server_data | map(attribute='monitoring.metrics.disk_usage') | sum / 100 * raw_server_data | sum(attribute='specs.disk') / raw_server_data | length) | round(1) }}
      utilization_percent: {{ (raw_server_data | map(attribute='monitoring.metrics.disk_usage') | sum / raw_server_data | length) | round(1) }}%

  # Growth projections (assuming linear growth)
  growth_projections:
{% if monitoring_data.time_series | length >= 2 %}
    {% set first_data = monitoring_data.time_series[0] %}
    {% set last_data = monitoring_data.time_series[-1] %}
    {% set time_diff_hours = ((last_data.timestamp | to_datetime('%Y-%m-%dT%H:%M:%SZ')) - (first_data.timestamp | to_datetime('%Y-%m-%dT%H:%M:%SZ'))).total_seconds() / 3600 %}
    
    # Calculate average growth rates
    {% set servers = first_data.metrics.keys() %}
    cpu_growth_rate_per_hour: {{ ((last_data.metrics.values() | sum(attribute='cpu') - first_data.metrics.values() | sum(attribute='cpu')) / (first_data.metrics.values() | sum(attribute='cpu')) / time_diff_hours * 100) | round(4) }}%
    memory_growth_rate_per_hour: {{ ((last_data.metrics.values() | sum(attribute='memory') - first_data.metrics.values() | sum(attribute='memory')) / (first_data.metrics.values() | sum(attribute='memory')) / time_diff_hours * 100) | round(4) }}%
    
    # Project capacity needs for next 30 days (720 hours)
    projected_30_days:
      cpu_utilization: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length + ((last_data.metrics.values() | sum(attribute='cpu') - first_data.metrics.values() | sum(attribute='cpu')) / (first_data.metrics.values() | sum(attribute='cpu')) / time_diff_hours * 720)) | round(1) }}%
      memory_utilization: {{ (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / raw_server_data | length + ((last_data.metrics.values() | sum(attribute='memory') - first_data.metrics.values() | sum(attribute='memory')) / (first_data.metrics.values() | sum(attribute='memory')) / time_diff_hours * 720)) | round(1) }}%
{% endif %}

  # Scaling recommendations
  scaling_recommendations:
{% set avg_cpu = (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length) %}
{% set avg_memory = (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / raw_server_data | length) %}
{% if avg_cpu > 80 or avg_memory > 80 %}
    - action: "scale_out"
      reason: "High resource utilization detected"
      priority: "high"
      recommended_additional_servers: {{ ((avg_cpu + avg_memory) / 100) | int }}
{% elif avg_cpu < 30 and avg_memory < 30 %}
    - action: "consolidate"
      reason: "Low resource utilization detected"
      priority: "medium"
      potential_server_reduction: {{ (raw_server_data | length * 0.3) | int }}
{% else %}
    - action: "monitor"
      reason: "Resource utilization within acceptable range"
      priority: "low"
{% endif %}

{# ========================================= #}
{# ANOMALY DETECTION                        #}
{# ========================================= #}

# Anomaly Detection
ANOMALY_DETECTION:
  statistical_analysis:
    # Calculate standard deviations for anomaly detection
    cpu_stats:
      mean: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length) | round(2) }}
      std_dev: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | map('float') | list | map('pow', 2) | sum / raw_server_data | length - ((raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length) ** 2)) ** 0.5 | round(2) }}
      
    memory_stats:
      mean: {{ (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / raw_server_data | length) | round(2) }}
      std_dev: {{ (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | map('float') | list | map('pow', 2) | sum / raw_server_data | length - ((raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / raw_server_data | length) ** 2)) ** 0.5 | round(2) }}

  # Identify anomalies (values more than 2 standard deviations from mean)
  anomalies:
{% set cpu_mean = (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length) %}
{% set cpu_std = (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | map('float') | list | map('pow', 2) | sum / raw_server_data | length - (cpu_mean ** 2)) ** 0.5 %}
{% set memory_mean = (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / raw_server_data | length) %}
{% set memory_std = (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | map('float') | list | map('pow', 2) | sum / raw_server_data | length - (memory_mean ** 2)) ** 0.5 %}

{% for server in raw_server_data %}
    {% if (server.monitoring.metrics.cpu_usage - cpu_mean) | abs > (cpu_std * 2) or (server.monitoring.metrics.memory_usage - memory_mean) | abs > (memory_std * 2) %}
    {{ server.hostname }}:
      cpu_anomaly: {{ 'yes' if (server.monitoring.metrics.cpu_usage - cpu_mean) | abs > (cpu_std * 2) else 'no' }}
      cpu_deviation: {{ ((server.monitoring.metrics.cpu_usage - cpu_mean) / cpu_std) | round(2) }} sigma
      memory_anomaly: {{ 'yes' if (server.monitoring.metrics.memory_usage - memory_mean) | abs > (memory_std * 2) else 'no' }}
      memory_deviation: {{ ((server.monitoring.metrics.memory_usage - memory_mean) / memory_std) | round(2) }} sigma
      investigation_required: true
    {% endif %}
{% endfor %}

{# ========================================= #}
{# BUSINESS IMPACT ANALYSIS                 #}
{# ========================================= #}

# Business Impact Analysis
BUSINESS_IMPACT:
  service_criticality:
{% for app in deployment_data.applications %}
    {{ app.name }}:
      availability: {{ ((app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / app.instances | length * 100) | round(1) }}%
      performance_score: {{ (100 - (app.instances | sum(attribute='response_time') / app.instances | length / 10 + app.instances | sum(attribute='error_rate') * 1000)) | round(1) }}
      business_impact: {{ 'high' if ((app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / app.instances | length * 100) < 99.0
                          else 'medium' if ((app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / app.instances | length * 100) < 99.9
                          else 'low' }}
      estimated_revenue_impact_per_hour: ${{ (1000 * (1 - (app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / app.instances | length)) | int }}
{% endfor %}

  total_estimated_impact:
    total_revenue_at_risk_per_hour: ${{ deployment_data.applications | sum(attribute='instances') | length * 100 }}
    sla_compliance_score: {{ ((deployment_data.applications | map(attribute='instances') | flatten | selectattr('status', 'equalto', 'healthy') | list | length) / (deployment_data.applications | map(attribute='instances') | flatten | length) * 100) | round(1) }}%
EOF
```

Create test for dynamic processing:

```yaml
# Create test-dynamic-processing.yml
cat > test-dynamic-processing.yml << 'EOF'
---
- name: Test dynamic data processing and aggregation
  hosts: data_processors
  gather_facts: yes
  vars_files:
    - complex-data.yml
  
  tasks:
    - name: Generate dynamic processing report
      template:
        src: templates/dynamic-processing.j2
        dest: /tmp/dynamic-processing-report.yml
        mode: '0644'
    
    - name: Display dynamic processing summary
      debug:
        msg: |
          Dynamic Data Processing Summary:
          
          Real-Time Metrics:
          - Total Servers: {{ raw_server_data | length }}
          - Average CPU Usage: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length) | round(1) }}%
          - Average Memory Usage: {{ (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / raw_server_data | length) | round(1) }}%
          
          Application Performance:
          {% for app in deployment_data.applications %}
          - {{ app.name }}: {{ ((app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / app.instances | length * 100) | round(1) }}% availability
          {% endfor %}
          
          Dynamic Thresholds:
          - CPU Warning: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | list | sort)[(raw_server_data | length * 0.95) | int] }}%
          - Memory Warning: {{ (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | list | sort)[(raw_server_data | length * 0.95) | int] }}%
          
          Capacity Planning:
          - Current CPU Utilization: {{ (raw_server_data | map(attribute='monitoring.metrics.cpu_usage') | sum / raw_server_data | length) | round(1) }}%
          - Current Memory Utilization: {{ (raw_server_data | map(attribute='monitoring.metrics.memory_usage') | sum / raw_server_data | length) | round(1) }}%
          
          Generated Report: /tmp/dynamic-processing-report.yml
EOF

# Run dynamic processing test
ansible-playbook -i inventory.ini test-dynamic-processing.yml
```

## Exercise 3: Enterprise Data Integration Patterns (7 minutes)

### Task: Implement enterprise data integration and ETL patterns

Create integration template:

```jinja2
# Create templates/data-integration.j2
cat > templates/data-integration.j2 << 'EOF'
{# Enterprise Data Integration Template #}
{# {{ ansible_managed }} #}

{# ========================================= #}
{# DATA EXTRACTION PATTERNS                 #}
{# ========================================= #}

# Data Extraction Summary
DATA_EXTRACTION:
  extraction_timestamp: {{ ansible_date_time.iso8601 }}
  extraction_source: "ansible_automation"
  data_sources:
    - name: "server_inventory"
      records: {{ raw_server_data | length }}
      last_updated: {{ raw_server_data | map(attribute='last_updated') | max }}
    - name: "application_deployments"
      records: {{ deployment_data.applications | length }}
      total_instances: {{ deployment_data.applications | map(attribute='instances') | flatten | length }}
    - name: "network_topology"
      records: {{ network_topology.subnets | length }}
      total_security_groups: {{ network_topology.subnets | map(attribute='security_groups') | flatten | length }}
    - name: "monitoring_metrics"
      records: {{ monitoring_data.time_series | length }}
      time_range: {{ monitoring_data.time_series | map(attribute='timestamp') | min }} to {{ monitoring_data.time_series | map(attribute='timestamp') | max }}

{# ========================================= #}
{# DATA TRANSFORMATION PATTERNS             #}
{# ========================================= #}

# Data Transformation Pipeline
DATA_TRANSFORMATION:
  transformation_rules:
    - rule: "server_normalization"
      description: "Normalize server data to standard format"
      applied_to: "raw_server_data"
      transformations:
        - "Convert specs to standardized units"
        - "Calculate health scores"
        - "Normalize service configurations"
        - "Extract and categorize tags"
    
    - rule: "application_aggregation"
      description: "Aggregate application performance metrics"
      applied_to: "deployment_data"
      transformations:
        - "Calculate average response times"
        - "Compute availability percentages"
        - "Analyze error rates"
        - "Map dependencies"
    
    - rule: "network_analysis"
      description: "Analyze network topology and security"
      applied_to: "network_topology"
      transformations:
        - "Calculate subnet utilization"
        - "Assess security risk levels"
        - "Map server-to-subnet relationships"
        - "Analyze port exposure"

  # Transformed data structures
  transformed_data:
    servers:
{% for server in raw_server_data %}
      - id: "{{ server.hostname }}"
        normalized_data:
          basic_info:
            hostname: "{{ server.hostname }}"
            ip_address: "{{ server.ip_address }}"
            environment: "{{ server.environment }}"
            role: "{{ server.role }}"
          
          specifications:
            cpu_cores: {{ server.specs.cpu }}
            memory_gb: {{ server.specs.memory }}
            disk_gb: {{ server.specs.disk }}
            capacity_score: {{ (server.specs.cpu * 10 + server.specs.memory + server.specs.disk / 10) | int }}
          
          performance_metrics:
            cpu_usage_percent: {{ server.monitoring.metrics.cpu_usage }}
            memory_usage_percent: {{ server.monitoring.metrics.memory_usage }}
            disk_usage_percent: {{ server.monitoring.metrics.disk_usage }}
            health_score: {{ (100 - ((server.monitoring.metrics.cpu_usage + server.monitoring.metrics.memory_usage + server.monitoring.metrics.disk_usage) / 3)) | round(1) }}
            status: "{{ 'critical' if server.monitoring.metrics.cpu_usage > 90 or server.monitoring.metrics.memory_usage > 90 
                        else 'warning' if server.monitoring.metrics.cpu_usage > 75 or server.monitoring.metrics.memory_usage > 75 
                        else 'healthy' }}"
          
          services:
{% for service in server.services %}
            - name: "{{ service.name }}"
              port: {{ service.port }}
              status: "{{ service.status }}"
              health_endpoint: "{{ server.ip_address }}:{{ service.port }}"
              configuration_hash: "{{ service.config | to_json | hash('md5') }}"
{% endfor %}
          
          categorization:
            tier: "{{ 'tier1' if 'critical' in server.tags else 'tier2' if 'production' in server.tags else 'tier3' }}"
            backup_priority: "{{ 'high' if 'backup-enabled' in server.tags else 'low' }}"
            monitoring_level: "{{ 'detailed' if server.role == 'database' else 'standard' }}"
{% endfor %}

{# ========================================= #}
{# DATA LOADING PATTERNS                    #}
{# ========================================= #}

# Data Loading Configuration
DATA_LOADING:
  target_systems:
    - name: "monitoring_dashboard"
      format: "json"
      endpoint: "http://dashboard.company.com/api/metrics"
      data_mapping:
        servers: "infrastructure.servers"
        applications: "applications.deployments"
        alerts: "monitoring.alerts"
      
    - name: "cmdb_system"
      format: "xml"
      endpoint: "http://cmdb.company.com/api/import"
      data_mapping:
        configuration_items: "servers"
        relationships: "dependencies"
        attributes: "specifications"
      
    - name: "data_warehouse"
      format: "csv"
      endpoint: "s3://data-warehouse/infrastructure/"
      data_mapping:
        fact_servers: "servers.performance_metrics"
        dim_applications: "applications"
        dim_time: "monitoring.time_series"

  # Generate data for different target systems
  export_formats:
    monitoring_dashboard:
      format: "json"
      structure:
        timestamp: "{{ ansible_date_time.iso8601 }}"
        infrastructure:
          servers:
{% for server in raw_server_data %}
            - hostname: "{{ server.hostname }}"
              metrics:
                cpu: {{ server.monitoring.metrics.cpu_usage }}
                memory: {{ server.monitoring.metrics.memory_usage }}
                disk: {{ server.monitoring.metrics.disk_usage }}
                health_score: {{ (100 - ((server.monitoring.metrics.cpu_usage + server.monitoring.metrics.memory_usage + server.monitoring.metrics.disk_usage) / 3)) | round(1) }}
{% endfor %}
        applications:
{% for app in deployment_data.applications %}
          - name: "{{ app.name }}"
            availability: {{ ((app.instances | selectattr('status', 'equalto', 'healthy') | list | length) / app.instances | length * 100) | round(1) }}
            response_time: {{ (app.instances | sum(attribute='response_time') / app.instances | length) | round(1) }}
            error_rate: {{ (app.instances | sum(attribute='error_rate') * 100) | round(3) }}
{% endfor %}

    cmdb_system:
      format: "csv"
      headers: ["hostname", "ip_address", "role", "environment", "cpu_cores", "memory_gb", "disk_gb", "status"]
      data:
{% for server in raw_server_data %}
        - ["{{ server.hostname }}", "{{ server.ip_address }}", "{{ server.role }}", "{{ server.environment }}", {{ server.specs.cpu }}, {{ server.specs.memory }}, {{ server.specs.disk }}, "{{ 'critical' if server.monitoring.metrics.cpu_usage > 90 or server.monitoring.metrics.memory_usage > 90 else 'warning' if server.monitoring.metrics.cpu_usage > 75 or server.monitoring.metrics.memory_usage > 75 else 'healthy' }}"]
{% endfor %}

{# ========================================= #}
{# DATA QUALITY AND VALIDATION              #}
{# ========================================= #}

# Data Quality Assessment
DATA_QUALITY:
  validation_rules:
    - rule: "completeness_check"
      description: "Ensure all required fields are present"
      results:
        servers_with_complete_data: {{ raw_server_data | selectattr('hostname') | selectattr('ip_address') | selectattr('specs') | selectattr('monitoring') | list | length }}
        total_servers: {{ raw_server_data | length }}
        completeness_percentage: {{ (raw_server_data | selectattr('hostname') | selectattr('ip_address') | selectattr('specs') | selectattr('monitoring') | list | length / raw_server_data | length * 100) | round(1) }}%
    
    - rule: "consistency_check"
      description: "Verify data consistency across sources"
      results:
        consistent_server_references: {{ deployment_data.applications | map(attribute='instances') | flatten | map(attribute='server') | unique | list | length }}
        total_server_references: {{ deployment_data.applications | map(attribute='instances') | flatten | map(attribute='server') | list | length }}
        consistency_percentage: {{ (deployment_data.applications | map(attribute='instances') | flatten | map(attribute='server') | unique | list | length / deployment_data.applications | map(attribute='instances') | flatten | map(attribute='server') | list | length * 100) | round(1) }}%
    
    - rule: "accuracy_check"
      description: "Validate metric ranges and logical constraints"
      results:
        valid_cpu_metrics: {{ raw_server_data | selectattr('monitoring.metrics.cpu_usage', 'ge', 0) | selectattr('monitoring.metrics.cpu_usage', 'le', 100) | list | length }}
        valid_memory_metrics: {{ raw_server_data | selectattr('monitoring.metrics.memory_usage', 'ge', 0) | selectattr('monitoring.metrics.memory_usage', 'le', 100) | list | length }}
        accuracy_percentage: {{ ((raw_server_data | selectattr('monitoring.metrics.cpu_usage', 'ge', 0) | selectattr('monitoring.metrics.cpu_usage', 'le', 100) | list | length + raw_server_data | selectattr('monitoring.metrics.memory_usage', 'ge', 0) | selectattr('monitoring.metrics.memory_usage', 'le', 100) | list | length) / (raw_server_data | length * 2) * 100) | round(1) }}%

  data_lineage:
    source_systems:
      - name: "infrastructure_monitoring"
        last_update: "{{ raw_server_data | map(attribute='last_updated') | max }}"
        record_count: {{ raw_server_data | length }}
      - name: "application_monitoring"
        last_update: "{{ ansible_date_time.iso8601 }}"
        record_count: {{ deployment_data.applications | length }}
      - name: "network_management"
        last_update: "{{ ansible_date_time.iso8601 }}"
        record_count: {{ network_topology.subnets | length }}
    
    transformation_steps:
      - step: 1
        operation: "data_extraction"
        timestamp: "{{ ansible_date_time.iso8601 }}"
        records_processed: {{ raw_server_data | length + deployment_data.applications | length + network_topology.subnets | length }}
      - step: 2
        operation: "data_normalization"
        timestamp: "{{ ansible_date_time.iso8601 }}"
        records_processed: {{ raw_server_data | length }}
      - step: 3
        operation: "data_aggregation"
        timestamp: "{{ ansible_date_time.iso8601 }}"
        records_processed: {{ deployment_data.applications | length }}
      - step: 4
        operation: "data_validation"
        timestamp: "{{ ansible_date_time.iso8601 }}"
        validation_rules_applied: 3

{# ========================================= #}
{# INTEGRATION MONITORING                   #}
{# ========================================= #}

# Integration Monitoring
INTEGRATION_MONITORING:
  pipeline_status:
    overall_status: "success"
    execution_time: "{{ ansible_date_time.iso8601 }}"
    
  performance_metrics:
    records_per_second: {{ ((raw_server_data | length + deployment_data.applications | length + network_topology.subnets | length) / 60) | round(2) }}
    data_volume_mb: {{ ((raw_server_data | to_json | length + deployment_data | to_json | length + network_topology | to_json | length) / 1024 / 1024) | round(2) }}
    transformation_efficiency: {{ ((raw_server_data | length * 100) / (raw_server_data | length + deployment_data.applications | length + network_topology.subnets | length)) | round(1) }}%
  
  error_handling:
    total_errors: 0
    warning_count: 0
    data_quality_issues: 0
    
  next_execution:
    scheduled_time: "{{ (ansible_date_time.epoch | int + 3600) | strftime('%Y-%m-%dT%H:%M:%SZ') }}"
    estimated_duration: "5 minutes"
    expected_record_count: {{ raw_server_data | length + 10 }}
EOF
```

Create test for data integration:

```yaml
# Create test-data-integration.yml
cat > test-data-integration.yml << 'EOF'
---
- name: Test enterprise data integration patterns
  hosts: data_processors
  gather_facts: yes
  vars_files:
    - complex-data.yml
  
  tasks:
    - name: Generate data integration report
      template:
        src: templates/data-integration.j2
        dest: /tmp/data-integration-report.yml
        mode: '0644'
    
    - name: Display integration summary
      debug:
        msg: |
          Enterprise Data Integration Summary:
          
          Data Sources:
          - Server Inventory: {{ raw_server_data | length }} records
          - Applications: {{ deployment_data.applications | length }} applications
          - Network Topology: {{ network_topology.subnets | length }} subnets
          - Monitoring Data: {{ monitoring_data.time_series | length }} time series points
          
          Data Quality:
          - Completeness: {{ (raw_server_data | selectattr('hostname') | selectattr('ip_address') | selectattr('specs') | selectattr('monitoring') | list | length / raw_server_data | length * 100) | round(1) }}%
          - Consistency: {{ (deployment_data.applications | map(attribute='instances') | flatten | map(attribute='server') | unique | list | length / deployment_data.applications | map(attribute='instances') | flatten | map(attribute='server') | list | length * 100) | round(1) }}%
          - Accuracy: {{ ((raw_server_data | selectattr('monitoring.metrics.cpu_usage', 'ge', 0) | selectattr('monitoring.metrics.cpu_usage', 'le', 100) | list | length + raw_server_data | selectattr('monitoring.metrics.memory_usage', 'ge', 0) | selectattr('monitoring.metrics.memory_usage', 'le', 100) | list | length) / (raw_server_data | length * 2) * 100) | round(1) }}%
          
          Integration Targets:
          - Monitoring Dashboard (JSON format)
          - CMDB System (CSV format)
          - Data Warehouse (Multiple formats)
          
          Performance:
          - Processing Rate: {{ ((raw_server_data | length + deployment_data.applications | length + network_topology.subnets | length) / 60) | round(2) }} records/second
          - Data Volume: {{ ((raw_server_data | to_json | length + deployment_data | to_json | length + network_topology | to_json | length) / 1024 / 1024) | round(2) }} MB
          
          Generated Report: /tmp/data-integration-report.yml
EOF

# Run data integration test
ansible-playbook -i inventory.ini test-data-integration.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Check generated reports
echo "=== Data Transformation Results ==="
ls -la /tmp/data-transformation* 2>/dev/null || echo "No transformation files generated"

echo "=== Dynamic Processing Results ==="
ls -la /tmp/dynamic-processing* 2>/dev/null || echo "No processing files generated"

echo "=== Data Integration Results ==="
ls -la /tmp/data-integration* 2>/dev/null || echo "No integration files generated"

# Validate JSON output
echo "=== JSON Validation ==="
python3 -m json.tool /tmp/data-transformation.json > /dev/null 2>&1 && echo "✅ JSON is valid" || echo "❌ JSON is invalid"
```

### 2. Discussion Points
- How do you handle complex data transformations in your current automation?
- What patterns do you use for enterprise data integration?
- How do you ensure data quality in automated processing pipelines?
- What are the performance considerations for large-scale data processing?

### 3. Clean Up
```bash
# Keep files for reference
# rm -rf /tmp/data-* /tmp/dynamic-*
```

## Key Takeaways
- Complex data transformations enable sophisticated enterprise automation
- Dynamic processing patterns provide real-time insights and analysis
- Enterprise integration patterns support multiple target systems
- Data quality validation ensures reliable automation outcomes
- Performance optimization is crucial for large-scale data processing
- Template-based ETL patterns provide maintainable data pipelines

## Next Steps
Proceed to Lab 6.5: Enterprise Template Management

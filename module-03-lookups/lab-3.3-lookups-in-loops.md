# Lab 3.3: Lookups in Loops and Complex Scenarios

## Objective
Use lookups effectively in loops, conditionals, and dynamic configurations for advanced enterprise automation patterns.

## Duration
25 minutes

## Prerequisites
- Completed Labs 3.1 and 3.2
- Understanding of Ansible loops and conditionals

## Lab Setup

```bash
cd ~/ansible-labs/module-03
mkdir -p lab-3.3
cd lab-3.3

cat > inventory.ini << EOF
[production]
prod-web01 ansible_host=localhost ansible_connection=local
prod-web02 ansible_host=localhost ansible_connection=local
prod-db01 ansible_host=localhost ansible_connection=local

[staging]
stage-web01 ansible_host=localhost ansible_connection=local
stage-db01 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF

# Create complex data structures
mkdir -p data configs templates

# Create environment-specific configuration files
cat > data/production-config.yml << EOF
---
environment: production
database:
  host: prod-db.company.com
  port: 5432
  ssl_enabled: true
  max_connections: 200
web:
  replicas: 3
  memory_limit: "2Gi"
  cpu_limit: "1000m"
cache:
  enabled: true
  ttl: 3600
  max_memory: "1Gi"
monitoring:
  enabled: true
  retention_days: 30
  alert_threshold: 80
EOF

cat > data/staging-config.yml << EOF
---
environment: staging
database:
  host: stage-db.company.com
  port: 5432
  ssl_enabled: false
  max_connections: 50
web:
  replicas: 1
  memory_limit: "1Gi"
  cpu_limit: "500m"
cache:
  enabled: false
  ttl: 1800
  max_memory: "512Mi"
monitoring:
  enabled: true
  retention_days: 7
  alert_threshold: 90
EOF

# Create service definitions
cat > data/services.csv << EOF
service_name,port,health_endpoint,config_file,dependencies
web-frontend,3000,/health,web-frontend.conf,api-backend
api-backend,8080,/api/health,api-backend.conf,database
user-service,8081,/users/health,user-service.conf,database
notification-service,8082,/notify/health,notification-service.conf,user-service
admin-panel,9090,/admin/health,admin-panel.conf,api-backend
EOF

# Create deployment matrix
cat > data/deployment-matrix.json << EOF
{
  "environments": {
    "production": {
      "services": ["web-frontend", "api-backend", "user-service"],
      "replicas": 3,
      "resources": {
        "cpu": "1000m",
        "memory": "2Gi"
      }
    },
    "staging": {
      "services": ["web-frontend", "api-backend"],
      "replicas": 1,
      "resources": {
        "cpu": "500m",
        "memory": "1Gi"
      }
    }
  },
  "feature_flags": {
    "production": {
      "new_ui": false,
      "beta_features": false,
      "debug_mode": false
    },
    "staging": {
      "new_ui": true,
      "beta_features": true,
      "debug_mode": true
    }
  }
}
EOF

# Create template for service configuration
cat > templates/service-config.j2 << EOF
# {{ service_name }} Configuration
# Environment: {{ environment }}

[service]
name={{ service_name }}
port={{ port }}
health_endpoint={{ health_endpoint }}
environment={{ environment }}

[resources]
cpu_limit={{ resources.cpu }}
memory_limit={{ resources.memory }}
replicas={{ replicas }}

[dependencies]
{% for dep in dependencies %}
requires={{ dep }}
{% endfor %}

[features]
{% for feature, enabled in feature_flags.items() %}
{{ feature }}={{ enabled | lower }}
{% endfor %}

# Generated at {{ ansible_date_time.iso8601 }}
EOF
```

## Exercise 1: Lookups in Complex Loops (10 minutes)

### Task: Use lookups within loops for dynamic service deployment

Create `lookups-in-loops.yml`:
```yaml
---
- name: Lookups in complex loops demonstration
  hosts: all
  vars:
    # Determine environment from inventory group
    current_environment: "{{ 'production' if inventory_hostname.startswith('prod') else 'staging' }}"
    
  tasks:
    # Load environment-specific configuration
    - name: Load environment configuration using file lookup
      set_fact:
        env_config: "{{ lookup('file', 'data/' + current_environment + '-config.yml') | from_yaml }}"
    
    - name: Display environment configuration
      debug:
        msg: |
          Environment Configuration for {{ current_environment }}:
          {{ env_config | to_nice_yaml }}
    
    # Load deployment matrix
    - name: Load deployment matrix
      set_fact:
        deployment_matrix: "{{ lookup('file', 'data/deployment-matrix.json') | from_json }}"
    
    # Get services for current environment
    - name: Get services for current environment
      set_fact:
        environment_services: "{{ deployment_matrix.environments[current_environment].services }}"
        environment_replicas: "{{ deployment_matrix.environments[current_environment].replicas }}"
        environment_resources: "{{ deployment_matrix.environments[current_environment].resources }}"
        environment_features: "{{ deployment_matrix.feature_flags[current_environment] }}"
    
    # Loop through services and use CSV lookup for each
    - name: Generate service configurations using lookups in loops
      set_fact:
        service_configs: |
          {% set configs = [] %}
          {% for service in environment_services %}
          {% set port = lookup('csvfile', service + ' file=data/services.csv delimiter=, col=1', errors='ignore') %}
          {% set health_endpoint = lookup('csvfile', service + ' file=data/services.csv delimiter=, col=2', errors='ignore') %}
          {% set config_file = lookup('csvfile', service + ' file=data/services.csv delimiter=, col=3', errors='ignore') %}
          {% set dependencies = lookup('csvfile', service + ' file=data/services.csv delimiter=, col=4', errors='ignore').split(',') if lookup('csvfile', service + ' file=data/services.csv delimiter=, col=4', errors='ignore') else [] %}
          {% if port %}
          {% set config = {
            'name': service,
            'port': port,
            'health_endpoint': health_endpoint,
            'config_file': config_file,
            'dependencies': dependencies,
            'replicas': environment_replicas,
            'resources': environment_resources,
            'features': environment_features
          } %}
          {% set _ = configs.append(config) %}
          {% endif %}
          {% endfor %}
          {{ configs }}
    
    - name: Display generated service configurations
      debug:
        msg: |
          Service Configuration for {{ item.name }}:
          - Port: {{ item.port }}
          - Health Endpoint: {{ item.health_endpoint }}
          - Dependencies: {{ item.dependencies | join(', ') if item.dependencies else 'None' }}
          - Replicas: {{ item.replicas }}
          - CPU: {{ item.resources.cpu }}
          - Memory: {{ item.resources.memory }}
      loop: "{{ service_configs }}"
    
    # Create service configuration files using template lookup in loop
    - name: Create service configuration files using template lookup
      copy:
        content: "{{ lookup('template', 'templates/service-config.j2') }}"
        dest: "/tmp/{{ inventory_hostname }}-{{ item.name }}.conf"
        mode: '0644'
      vars:
        service_name: "{{ item.name }}"
        port: "{{ item.port }}"
        health_endpoint: "{{ item.health_endpoint }}"
        environment: "{{ current_environment }}"
        replicas: "{{ item.replicas }}"
        resources: "{{ item.resources }}"
        dependencies: "{{ item.dependencies }}"
        feature_flags: "{{ item.features }}"
      loop: "{{ service_configs }}"
    
    # Advanced loop with conditional lookups
    - name: Conditional configuration based on environment and service
      copy:
        content: |
          # Advanced Configuration for {{ item.name }}
          # Environment: {{ current_environment }}
          # Host: {{ inventory_hostname }}
          
          [basic]
          service={{ item.name }}
          port={{ item.port }}
          environment={{ current_environment }}
          
          [advanced]
          {% if current_environment == 'production' %}
          # Production-specific settings from lookup
          ssl_enabled={{ env_config.database.ssl_enabled | default(false) | lower }}
          max_connections={{ env_config.database.max_connections | default(100) }}
          monitoring_enabled={{ env_config.monitoring.enabled | default(true) | lower }}
          alert_threshold={{ env_config.monitoring.alert_threshold | default(80) }}
          {% else %}
          # Staging-specific settings from lookup
          ssl_enabled={{ env_config.database.ssl_enabled | default(false) | lower }}
          max_connections={{ env_config.database.max_connections | default(50) }}
          debug_mode={{ item.features.debug_mode | default(false) | lower }}
          beta_features={{ item.features.beta_features | default(false) | lower }}
          {% endif %}
          
          [cache]
          {% if env_config.cache.enabled %}
          cache_enabled=true
          cache_ttl={{ env_config.cache.ttl }}
          cache_memory={{ env_config.cache.max_memory }}
          {% else %}
          cache_enabled=false
          {% endif %}
          
          [metadata]
          generated_at={{ ansible_date_time.iso8601 }}
          config_source=lookup_based
        dest: "/tmp/{{ inventory_hostname }}-{{ item.name }}-advanced.conf"
        mode: '0644'
      loop: "{{ service_configs }}"
```

### Run lookups in loops:
```bash
# Run lookups in complex loops
ansible-playbook -i inventory.ini lookups-in-loops.yml

# Check generated configurations
ls -la /tmp/prod-*-*.conf
ls -la /tmp/stage-*-*.conf
cat /tmp/prod-web01-web-frontend.conf
cat /tmp/stage-web01-web-frontend-advanced.conf
```

## Exercise 2: Dynamic Infrastructure Configuration (8 minutes)

### Task: Use lookups for dynamic infrastructure provisioning and configuration

Create `dynamic-infrastructure.yml`:
```yaml
---
- name: Dynamic infrastructure configuration using lookups
  hosts: localhost
  vars:
    # Infrastructure templates
    infrastructure_types:
      - "web-tier"
      - "database-tier"
      - "cache-tier"
      - "monitoring-tier"
      
    # Environment mappings
    environment_mappings:
      production: "prod"
      staging: "stage"
      development: "dev"
      
  tasks:
    # Create infrastructure templates
    - name: Create infrastructure templates
      copy:
        content: |
          # {{ item }} Infrastructure Template
          [{{ item }}]
          {% if item == 'web-tier' %}
          instance_type=t3.medium
          min_instances=2
          max_instances=10
          load_balancer=true
          ssl_certificate=required
          {% elif item == 'database-tier' %}
          instance_type=r5.large
          storage_type=gp3
          backup_retention=30
          multi_az=true
          encryption=true
          {% elif item == 'cache-tier' %}
          instance_type=r5.medium
          cluster_mode=enabled
          backup_enabled=true
          ttl_default=3600
          {% elif item == 'monitoring-tier' %}
          instance_type=t3.small
          storage_size=100
          retention_days=90
          alerting=enabled
          {% endif %}
          
          # Common settings
          vpc_id={{ '${vpc_id}' }}
          subnet_ids={{ '${subnet_ids}' }}
          security_groups={{ '${security_groups}' }}
        dest: "/tmp/{{ item }}-template.conf"
        mode: '0644'
      loop: "{{ infrastructure_types }}"
    
    # Dynamic infrastructure configuration using loops and lookups
    - name: Generate environment-specific infrastructure configurations
      set_fact:
        infrastructure_configs: |
          {% set configs = {} %}
          {% for env_name, env_prefix in environment_mappings.items() %}
          {% set env_configs = [] %}
          {% for infra_type in infrastructure_types %}
          {% set template_content = lookup('file', '/tmp/' + infra_type + '-template.conf', errors='ignore') %}
          {% if template_content %}
          {% set config = {
            'type': infra_type,
            'environment': env_name,
            'prefix': env_prefix,
            'template': template_content
          } %}
          {% set _ = env_configs.append(config) %}
          {% endif %}
          {% endfor %}
          {% set _ = configs.update({env_name: env_configs}) %}
          {% endfor %}
          {{ configs }}
    
    - name: Create environment-specific infrastructure files
      copy:
        content: |
          # Infrastructure Configuration
          # Environment: {{ outer_item.key }}
          # Type: {{ item.type }}
          
          {{ item.template }}
          
          [environment_specific]
          environment={{ item.environment }}
          name_prefix={{ item.prefix }}-{{ item.type }}
          
          {% if item.environment == 'production' %}
          backup_enabled=true
          monitoring_level=detailed
          high_availability=true
          {% elif item.environment == 'staging' %}
          backup_enabled=false
          monitoring_level=basic
          high_availability=false
          {% else %}
          backup_enabled=false
          monitoring_level=minimal
          high_availability=false
          {% endif %}
          
          [metadata]
          generated_at={{ ansible_date_time.iso8601 }}
          config_method=dynamic_lookup
        dest: "/tmp/{{ item.prefix }}-{{ item.type }}-infra.conf"
        mode: '0644'
      loop: "{{ outer_item.value }}"
      loop_control:
        loop_var: item
      with_dict: "{{ infrastructure_configs }}"
      loop_control:
        loop_var: outer_item
    
    # Network configuration using pipe lookups in loops
    - name: Generate network configurations using pipe lookups
      set_fact:
        network_configs: |
          {% set configs = [] %}
          {% for env_name in environment_mappings.keys() %}
          {% set network_info = {
            'environment': env_name,
            'vpc_cidr': '10.' + (loop.index * 10) | string + '.0.0/16',
            'public_subnets': [],
            'private_subnets': []
          } %}
          {% for i in range(1, 4) %}
          {% set public_cidr = '10.' + (loop.index0 * 10 + 10) | string + '.' + i | string + '.0/24' %}
          {% set private_cidr = '10.' + (loop.index0 * 10 + 10) | string + '.' + (i + 10) | string + '.0/24' %}
          {% set _ = network_info.public_subnets.append(public_cidr) %}
          {% set _ = network_info.private_subnets.append(private_cidr) %}
          {% endfor %}
          {% set _ = configs.append(network_info) %}
          {% endfor %}
          {{ configs }}
    
    - name: Create network configuration files
      copy:
        content: |
          # Network Configuration
          # Environment: {{ item.environment }}
          
          [vpc]
          cidr_block={{ item.vpc_cidr }}
          enable_dns_hostnames=true
          enable_dns_support=true
          
          [public_subnets]
          {% for subnet in item.public_subnets %}
          public_subnet_{{ loop.index }}={{ subnet }}
          {% endfor %}
          
          [private_subnets]
          {% for subnet in item.private_subnets %}
          private_subnet_{{ loop.index }}={{ subnet }}
          {% endfor %}
          
          [routing]
          internet_gateway=enabled
          nat_gateway=enabled
          
          [security]
          {% if item.environment == 'production' %}
          vpc_flow_logs=enabled
          network_acls=strict
          {% else %}
          vpc_flow_logs=disabled
          network_acls=permissive
          {% endif %}
          
          [metadata]
          generated_at={{ ansible_date_time.iso8601 }}
          total_subnets={{ (item.public_subnets | length) + (item.private_subnets | length) }}
        dest: "/tmp/{{ item.environment }}-network.conf"
        mode: '0644'
      loop: "{{ network_configs }}"
```

### Run dynamic infrastructure configuration:
```bash
# Run dynamic infrastructure configuration
ansible-playbook -i inventory.ini dynamic-infrastructure.yml

# Check generated infrastructure configurations
ls -la /tmp/*-infra.conf
ls -la /tmp/*-network.conf
cat /tmp/prod-web-tier-infra.conf
cat /tmp/production-network.conf
```

## Exercise 3: Advanced Lookup Patterns and Optimization (7 minutes)

### Task: Implement advanced lookup patterns with caching and optimization

Create `advanced-lookup-patterns.yml`:
```yaml
---
- name: Advanced lookup patterns and optimization
  hosts: localhost
  vars:
    # Configuration sources
    config_sources:
      - type: "file"
        source: "data/production-config.yml"
        format: "yaml"
      - type: "file"
        source: "data/deployment-matrix.json"
        format: "json"
      - type: "env"
        source: "HOME"
        format: "string"
      - type: "pipe"
        source: "hostname"
        format: "string"
        
    # Lookup cache for optimization
    lookup_cache: {}
    
  tasks:
    # Cached lookup pattern
    - name: Implement cached lookup pattern
      set_fact:
        cached_lookups: |
          {% set cache = {} %}
          {% for source in config_sources %}
          {% set cache_key = source.type + '_' + source.source | replace('/', '_') | replace('.', '_') %}
          {% if source.type == 'file' %}
          {% set content = lookup('file', source.source, errors='ignore') %}
          {% if content %}
          {% if source.format == 'yaml' %}
          {% set parsed_content = content | from_yaml %}
          {% elif source.format == 'json' %}
          {% set parsed_content = content | from_json %}
          {% else %}
          {% set parsed_content = content %}
          {% endif %}
          {% set _ = cache.update({cache_key: {'content': parsed_content, 'type': source.type, 'format': source.format}}) %}
          {% endif %}
          {% elif source.type == 'env' %}
          {% set content = lookup('env', source.source, errors='ignore') %}
          {% if content %}
          {% set _ = cache.update({cache_key: {'content': content, 'type': source.type, 'format': source.format}}) %}
          {% endif %}
          {% elif source.type == 'pipe' %}
          {% set content = lookup('pipe', source.source, errors='ignore') %}
          {% if content %}
          {% set _ = cache.update({cache_key: {'content': content, 'type': source.type, 'format': source.format}}) %}
          {% endif %}
          {% endif %}
          {% endfor %}
          {{ cache }}
    
    - name: Display cached lookup results
      debug:
        msg: |
          Cached Lookup: {{ item.key }}
          Type: {{ item.value.type }}
          Format: {{ item.value.format }}
          Content Preview: {{ item.value.content | string | truncate(100) }}
      loop: "{{ cached_lookups | dict2items }}"
    
    # Conditional lookup execution
    - name: Conditional lookup execution based on environment
      set_fact:
        conditional_configs: |
          {% set configs = {} %}
          {% set current_hour = ansible_date_time.hour | int %}
          {% if current_hour >= 9 and current_hour <= 17 %}
          {% set time_period = 'business_hours' %}
          {% else %}
          {% set time_period = 'off_hours' %}
          {% endif %}
          
          {% for env in ['production', 'staging'] %}
          {% set env_config = {} %}
          
          # Load base configuration
          {% if 'file_data_' + env + '-config_yml' in cached_lookups %}
          {% set base_config = cached_lookups['file_data_' + env + '-config_yml'].content %}
          {% set _ = env_config.update(base_config) %}
          {% endif %}
          
          # Add time-based configuration
          {% if time_period == 'business_hours' %}
          {% set _ = env_config.update({
            'scaling': {
              'enabled': true,
              'min_instances': 2,
              'max_instances': 10
            },
            'monitoring': {
              'level': 'detailed',
              'alerts': 'enabled'
            }
          }) %}
          {% else %}
          {% set _ = env_config.update({
            'scaling': {
              'enabled': false,
              'min_instances': 1,
              'max_instances': 3
            },
            'monitoring': {
              'level': 'basic',
              'alerts': 'reduced'
            }
          }) %}
          {% endif %}
          
          {% set _ = configs.update({env: env_config}) %}
          {% endfor %}
          {{ configs }}
    
    - name: Create time-based configurations
      copy:
        content: |
          # Time-Based Configuration
          # Environment: {{ item.key }}
          # Generated at: {{ ansible_date_time.iso8601 }}
          # Time period: {{ 'business_hours' if ansible_date_time.hour | int >= 9 and ansible_date_time.hour | int <= 17 else 'off_hours' }}
          
          [base_config]
          {% if item.value.database is defined %}
          database_host={{ item.value.database.host }}
          database_port={{ item.value.database.port }}
          database_ssl={{ item.value.database.ssl_enabled | default(false) | lower }}
          {% endif %}
          
          [scaling]
          scaling_enabled={{ item.value.scaling.enabled | lower }}
          min_instances={{ item.value.scaling.min_instances }}
          max_instances={{ item.value.scaling.max_instances }}
          
          [monitoring]
          monitoring_level={{ item.value.monitoring.level }}
          alerts={{ item.value.monitoring.alerts }}
          
          [web_config]
          {% if item.value.web is defined %}
          replicas={{ item.value.web.replicas }}
          memory_limit={{ item.value.web.memory_limit }}
          cpu_limit={{ item.value.web.cpu_limit }}
          {% endif %}
          
          [metadata]
          config_source=cached_lookups
          time_based=true
          cache_keys={{ cached_lookups.keys() | list | join(',') }}
        dest: "/tmp/{{ item.key }}-time-based.conf"
        mode: '0644'
      loop: "{{ conditional_configs | dict2items }}"
    
    # Lookup performance optimization
    - name: Optimize lookup performance with batching
      set_fact:
        batch_results: |
          {% set results = {} %}
          {% set file_lookups = [] %}
          {% set env_lookups = [] %}
          {% set pipe_lookups = [] %}
          
          # Batch similar lookup types
          {% for source in config_sources %}
          {% if source.type == 'file' %}
          {% set _ = file_lookups.append(source) %}
          {% elif source.type == 'env' %}
          {% set _ = env_lookups.append(source) %}
          {% elif source.type == 'pipe' %}
          {% set _ = pipe_lookups.append(source) %}
          {% endif %}
          {% endfor %}
          
          # Process batches
          {% set _ = results.update({'file_batch': file_lookups | length}) %}
          {% set _ = results.update({'env_batch': env_lookups | length}) %}
          {% set _ = results.update({'pipe_batch': pipe_lookups | length}) %}
          {% set _ = results.update({'total_lookups': config_sources | length}) %}
          {% set _ = results.update({'cache_hits': cached_lookups | length}) %}
          
          {{ results }}
    
    - name: Create performance report
      copy:
        content: |
          # Lookup Performance Report
          # Generated at: {{ ansible_date_time.iso8601 }}
          
          [batch_statistics]
          file_lookups={{ batch_results.file_batch }}
          env_lookups={{ batch_results.env_batch }}
          pipe_lookups={{ batch_results.pipe_batch }}
          total_lookups={{ batch_results.total_lookups }}
          cache_hits={{ batch_results.cache_hits }}
          
          [optimization_metrics]
          cache_hit_ratio={{ (batch_results.cache_hits / batch_results.total_lookups * 100) | round(2) }}%
          lookup_efficiency={{ 'HIGH' if batch_results.cache_hits > batch_results.total_lookups * 0.7 else 'MEDIUM' if batch_results.cache_hits > batch_results.total_lookups * 0.4 else 'LOW' }}
          
          [recommendations]
          {% if batch_results.cache_hits < batch_results.total_lookups * 0.5 %}
          cache_optimization=recommended
          {% endif %}
          {% if batch_results.file_batch > 5 %}
          file_consolidation=recommended
          {% endif %}
          {% if batch_results.pipe_batch > 3 %}
          pipe_optimization=recommended
          {% endif %}
          
          [cache_details]
          {% for key, value in cached_lookups.items() %}
          {{ key }}_type={{ value.type }}
          {% endfor %}
          
          [metadata]
          report_type=lookup_performance
          analysis_timestamp={{ ansible_date_time.iso8601 }}
        dest: "/tmp/lookup-performance-report.conf"
        mode: '0644'
    
    - name: Display optimization summary
      debug:
        msg: |
          Lookup Optimization Summary:
          - Total lookups: {{ batch_results.total_lookups }}
          - Cache hits: {{ batch_results.cache_hits }}
          - Cache hit ratio: {{ (batch_results.cache_hits / batch_results.total_lookups * 100) | round(2) }}%
          - Efficiency: {{ 'HIGH' if batch_results.cache_hits > batch_results.total_lookups * 0.7 else 'MEDIUM' if batch_results.cache_hits > batch_results.total_lookups * 0.4 else 'LOW' }}
```

### Run advanced lookup patterns:
```bash
# Run advanced lookup patterns
ansible-playbook -i inventory.ini advanced-lookup-patterns.yml

# Check generated configurations
ls -la /tmp/*-time-based.conf
cat /tmp/production-time-based.conf
cat /tmp/lookup-performance-report.conf
```

## Verification and Discussion

### 1. Check Results
```bash
# Check loops and lookups results
ls -la /tmp/prod-*-*.conf
ls -la /tmp/stage-*-*.conf
cat /tmp/prod-web01-web-frontend.conf

# Check dynamic infrastructure results
ls -la /tmp/*-infra.conf
ls -la /tmp/*-network.conf
cat /tmp/prod-web-tier-infra.conf

# Check advanced patterns results
cat /tmp/production-time-based.conf
cat /tmp/lookup-performance-report.conf
```

### 2. Discussion Points
- How do you handle complex data transformations in your current automation?
- What strategies do you use for optimizing lookup performance?
- How do you implement dynamic configuration based on time or environment?

### 3. Clean Up
```bash
# Remove demo files
rm -rf data/ configs/ templates/ /tmp/*-template.conf /tmp/*-*.conf /tmp/*-infra.conf /tmp/*-network.conf /tmp/*-time-based.conf /tmp/lookup-performance-report.conf
```

## Key Takeaways
- Lookups in loops enable dynamic configuration generation at scale
- Caching lookup results improves performance for repeated operations
- Conditional lookups allow environment and time-based configuration
- Batching similar lookup types optimizes execution efficiency
- Complex data transformations can be achieved through nested lookups
- Performance monitoring helps identify optimization opportunities

## Module 3 Summary

### What We Covered
1. **Basic Lookup Plugins** - File, environment, variable, and template lookups
2. **Advanced Data Lookups** - CSV, DNS, URL, and pipe lookups for external integration
3. **Lookups in Loops and Complex Scenarios** - Dynamic patterns and optimization strategies

### Key Skills Developed
- Integrating external data sources into Ansible automation
- Using lookups for dynamic configuration generation
- Implementing performance optimization patterns for lookups
- Creating complex data transformation workflows
- Building enterprise-grade configuration management systems

### Next Steps
You now have comprehensive knowledge of Ansible lookup plugins and their advanced usage patterns. These skills will be essential for integrating external systems and creating dynamic, data-driven automation in the subsequent modules.

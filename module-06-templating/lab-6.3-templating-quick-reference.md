# Lab 6.3: Advanced Templating (Quick Reference)

## Objective
Quick examples of advanced Jinja2 patterns for self-study

## Pattern 1: Template Inheritance

### Base Template Pattern
```jinja2
# Create base-config.j2
{# Base configuration template #}
# {{ app_name | default('Application') }} Configuration
# Generated: {{ ansible_date_time.iso8601 }}

[application]
name={{ app_name | default('MyApp') }}
version={{ app_version | default('1.0.0') }}
environment={{ environment | default('production') }}

{% block database_config %}
# Default database configuration
[database]
host=localhost
port=5432
{% endblock %}

{% block cache_config %}
# Default cache configuration  
[cache]
enabled=false
{% endblock %}

{% block custom_config %}
# Custom configuration block
{% endblock %}
```

### Extended Templates
```jinja2
# Create web-config.j2 (extends base)
{% extends "base-config.j2" %}

{% block database_config %}
[database]
host={{ db_host | default('web-db.example.com') }}
port={{ db_port | default(5432) }}
name={{ db_name | default('webapp') }}
pool_size={{ db_pool_size | default(10) }}
{% endblock %}

{% block cache_config %}
[cache]
enabled=true
servers={{ cache_servers | join(',') }}
ttl={{ cache_ttl | default(3600) }}
{% endblock %}

{% block custom_config %}
[web_server]
port={{ web_port | default(8080) }}
workers={{ web_workers | default(4) }}
{% endblock %}
```

### Usage Example
```yaml
# Using template inheritance
- name: Generate web server config
  template:
    src: web-config.j2
    dest: /etc/webapp/config.ini
  vars:
    app_name: "WebPortal"
    db_host: "prod-db.company.com"
    cache_servers: ["redis1:6379", "redis2:6379"]
    web_port: 80
```

## Pattern 2: Macros and Reusable Components

### Define Macros
```jinja2
# Create macros.j2
{% macro server_block(name, port, ssl=false) %}
server {
    listen {{ port }}{% if ssl %} ssl{% endif %};
    server_name {{ name }};
    
    {% if ssl %}
    ssl_certificate /etc/ssl/{{ name }}.crt;
    ssl_certificate_key /etc/ssl/{{ name }}.key;
    {% endif %}
    
    location / {
        proxy_pass http://backend-{{ name }};
    }
}
{% endmacro %}

{% macro database_connection(env, host, port=5432) %}
[{{ env }}_database]
host={{ host }}
port={{ port }}
database=app_{{ env }}
{% if env == 'production' %}
pool_size=20
timeout=30
{% else %}
pool_size=5
timeout=10
{% endif %}
{% endmacro %}
```

### Use Macros
```jinja2
# Create nginx-sites.j2
{% from 'macros.j2' import server_block %}

# Nginx Configuration with Macros
{% for site in sites %}
{{ server_block(site.name, site.port, site.ssl | default(false)) }}
{% endfor %}
```

## Pattern 3: Advanced Filters and Data Processing

### Custom Filter Usage
```yaml
# Advanced filtering examples
- name: Complex data processing
  copy:
    content: |
      # Server Statistics
      {% set prod_servers = servers | selectattr('env', 'equalto', 'production') | list %}
      {% set total_memory = prod_servers | sum(attribute='memory_gb') %}
      
      Production Servers: {{ prod_servers | length }}
      Total Memory: {{ total_memory }}GB
      Average Memory: {{ (total_memory / (prod_servers | length)) | round(1) }}GB
      
      # High Memory Servers (>16GB)
      {% for server in servers | selectattr('memory_gb', 'gt', 16) %}
      {{ server.name }}: {{ server.memory_gb }}GB
      {% endfor %}
      
      # Grouped by Environment
      {% for env, group in servers | groupby('env') %}
      {{ env | title }} Environment:
      {% for server in group %}
        - {{ server.name }} ({{ server.memory_gb }}GB)
      {% endfor %}
      {% endfor %}
    dest: /tmp/server-stats.txt
```

### JSON/YAML Processing
```yaml
# Working with complex data structures
- name: Generate service discovery config
  copy:
    content: |
      {
        "services": {
      {% for service_name, instances in services.items() %}
          "{{ service_name }}": [
      {% for instance in instances %}
            {
              "id": "{{ instance.id }}",
              "address": "{{ instance.host }}",
              "port": {{ instance.port }},
              "tags": {{ instance.tags | to_json }},
              "health": "{{ instance.health | default('passing') }}"
            }{{ ',' if not loop.last else '' }}
      {% endfor %}
          ]{{ ',' if not loop.last else '' }}
      {% endfor %}
        }
      }
    dest: /tmp/service-discovery.json
```

## Pattern 4: Environment-Specific Templates

### Multi-Environment Configuration
```jinja2
# Create multi-env-template.j2
{% set env_config = {
  'development': {
    'log_level': 'DEBUG',
    'db_pool': 2,
    'cache_ttl': 60,
    'monitoring': false
  },
  'staging': {
    'log_level': 'INFO', 
    'db_pool': 5,
    'cache_ttl': 300,
    'monitoring': true
  },
  'production': {
    'log_level': 'WARN',
    'db_pool': 20,
    'cache_ttl': 3600,
    'monitoring': true
  }
} %}

{% set current_env = env_config[environment] %}

# {{ environment | title }} Configuration
[logging]
level={{ current_env.log_level }}

[database]
pool_size={{ current_env.db_pool }}

[cache]
ttl={{ current_env.cache_ttl }}

{% if current_env.monitoring %}
[monitoring]
enabled=true
endpoint=/metrics
{% endif %}
```

## Pattern 5: Dynamic Configuration Generation

### Loop-Based Configuration
```yaml
# Generate multiple configuration files
- name: Generate per-service configurations
  template:
    src: service-template.j2
    dest: "/etc/services/{{ item.name }}.conf"
  loop:
    - name: "web"
      port: 8080
      replicas: 3
    - name: "api" 
      port: 3000
      replicas: 2
    - name: "worker"
      port: 0
      replicas: 5
```

### Conditional Service Configuration
```jinja2
# service-template.j2
[service]
name={{ item.name }}
{% if item.port > 0 %}
port={{ item.port }}
{% endif %}
replicas={{ item.replicas }}

{% if item.name == 'web' %}
# Web-specific configuration
static_files=/var/www/static
{% elif item.name == 'api' %}
# API-specific configuration
rate_limit=1000
{% elif item.name == 'worker' %}
# Worker-specific configuration
queue=background_jobs
{% endif %}
```

## Best Practices Summary

### Template Organization:
- Use template inheritance for common patterns
- Create reusable macros for repeated blocks
- Organize templates in logical directories
- Version control template changes

### Performance:
- Cache complex calculations in variables
- Use `set` for expensive operations
- Avoid nested loops when possible
- Test templates with realistic data sizes

### Maintainability:
- Comment complex template logic
- Use meaningful variable names
- Validate template output
- Document template dependencies

### Security:
- Escape user input appropriately
- Validate template variables
- Use Ansible Vault for sensitive data
- Restrict template file permissions

## Common Jinja2 Filters Reference

### String Filters:
```jinja2
{{ "hello world" | title }}           # Hello World
{{ "  spaces  " | trim }}             # spaces
{{ "text" | upper }}                  # TEXT
{{ "TEXT" | lower }}                  # text
{{ "hello" | replace("l", "x") }}     # hexxo
```

### List Filters:
```jinja2
{{ [1,2,3] | length }}                # 3
{{ [1,2,3] | first }}                 # 1
{{ [1,2,3] | last }}                  # 3
{{ ['a','b','c'] | join(',') }}       # a,b,c
{{ [3,1,2] | sort }}                  # [1,2,3]
```

### Dictionary Filters:
```jinja2
{{ dict.keys() | list }}              # List of keys
{{ dict.values() | list }}            # List of values  
{{ dict.items() | list }}             # List of key-value pairs
```

### Ansible-Specific Filters:
```jinja2
{{ data | to_json }}                  # Convert to JSON
{{ data | to_yaml }}                  # Convert to YAML
{{ data | b64encode }}                # Base64 encode
{{ ip_address | ipaddr('network') }}  # IP network operations
```

## Troubleshooting Templates

### Common Issues:
1. **Undefined variables** - Use `default()` filter
2. **Type errors** - Check data types with `type_debug`
3. **Loop issues** - Verify list/dict structure
4. **Syntax errors** - Check Jinja2 syntax carefully

### Debug Techniques:
```yaml
# Debug template variables
- debug:
    var: variable_name

# Test template syntax
- template:
    src: test-template.j2
    dest: /tmp/test-output
  check_mode: yes
```

---

**Note**: This quick reference covers advanced patterns for experienced users. Practice with simple examples first, then gradually incorporate these advanced techniques into your templates.

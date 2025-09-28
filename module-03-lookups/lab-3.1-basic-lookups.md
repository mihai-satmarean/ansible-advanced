# Lab 3.1: Basic Lookup Plugins

## Objective
Master fundamental lookup plugins including file, env, vars, and template lookups for dynamic data retrieval.

## Duration
25 minutes

## Prerequisites
- Completed Module 1 and 2 labs
- Understanding of Ansible variables and data structures

## Lab Setup

```bash
cd ~/ansible-labs
mkdir -p module-03/lab-3.1
cd module-03/lab-3.1

cat > inventory.ini << EOF
[servers]
web01 ansible_host=localhost ansible_connection=local
web02 ansible_host=localhost ansible_connection=local

[databases]
db01 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF

# Create test data files
mkdir -p data templates

# Create sample configuration files
cat > data/app-config.json << EOF
{
  "application": {
    "name": "enterprise-app",
    "version": "3.1.0",
    "port": 8080,
    "database": {
      "host": "db01",
      "port": 5432,
      "name": "app_db"
    }
  },
  "features": {
    "authentication": true,
    "caching": true,
    "monitoring": true
  }
}
EOF

cat > data/secrets.yml << EOF
---
database_password: "super_secret_password"
api_key: "abc123def456ghi789"
encryption_key: "my_encryption_key_2023"
jwt_secret: "jwt_signing_secret"
EOF

cat > data/environment.properties << EOF
# Environment Configuration
ENVIRONMENT=production
LOG_LEVEL=INFO
MAX_CONNECTIONS=100
TIMEOUT=30
CACHE_TTL=3600
DEBUG_MODE=false
EOF

# Create template file
cat > templates/nginx.conf.j2 << EOF
# Nginx Configuration Template
upstream {{ app_name }}_backend {
{% for server in backend_servers %}
    server {{ server.host }}:{{ server.port }} weight={{ server.weight | default(1) }};
{% endfor %}
}

server {
    listen {{ listen_port | default(80) }};
    server_name {{ server_name | default('localhost') }};
    
    location / {
        proxy_pass http://{{ app_name }}_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # Generated at {{ ansible_date_time.iso8601 }}
}
EOF
```

## Exercise 1: File and Environment Lookups (10 minutes)

### Task: Use file and environment lookups for configuration management

Create `file-env-lookups.yml`:
```yaml
---
- name: File and environment lookup demonstrations
  hosts: localhost
  vars:
    # Environment variables for testing
    test_env_vars:
      APP_NAME: "lookup-demo"
      APP_VERSION: "1.0.0"
      DATABASE_URL: "postgresql://localhost:5432/demo"
      
  tasks:
    # Set environment variables for testing
    - name: Set test environment variables
      shell: |
        export {{ item.key }}="{{ item.value }}"
        echo "{{ item.key }}={{ item.value }}" >> /tmp/test-env
      loop: "{{ test_env_vars | dict2items }}"
      environment: "{{ test_env_vars }}"
    
    # File lookup examples
    - name: Read JSON configuration using file lookup
      set_fact:
        app_config: "{{ lookup('file', 'data/app-config.json') | from_json }}"
    
    - name: Display parsed JSON configuration
      debug:
        msg: |
          Application Configuration:
          - Name: {{ app_config.application.name }}
          - Version: {{ app_config.application.version }}
          - Port: {{ app_config.application.port }}
          - Database: {{ app_config.application.database.host }}:{{ app_config.application.database.port }}
          - Features: {{ app_config.features.keys() | list | join(', ') }}
    
    - name: Read YAML secrets using file lookup
      set_fact:
        secrets: "{{ lookup('file', 'data/secrets.yml') | from_yaml }}"
    
    - name: Display secrets (masked for security)
      debug:
        msg: |
          Secrets loaded:
          - Database password: {{ secrets.database_password | regex_replace('.', '*') }}
          - API key: {{ secrets.api_key[:8] }}***
          - Encryption key: {{ secrets.encryption_key[:8] }}***
          - JWT secret: {{ secrets.jwt_secret[:8] }}***
    
    - name: Read properties file using file lookup
      set_fact:
        env_properties: "{{ lookup('file', 'data/environment.properties') }}"
    
    - name: Parse properties file content
      set_fact:
        parsed_properties: |
          {% set props = {} %}
          {% for line in env_properties.split('\n') %}
          {% if line and not line.startswith('#') and '=' in line %}
          {% set key_value = line.split('=', 1) %}
          {% set _ = props.update({key_value[0]: key_value[1]}) %}
          {% endif %}
          {% endfor %}
          {{ props }}
    
    - name: Display parsed properties
      debug:
        msg: |
          Environment Properties:
          {{ parsed_properties | to_nice_yaml }}
    
    # Environment lookup examples
    - name: Read environment variables using env lookup
      debug:
        msg: |
          Environment Variables:
          - User: {{ lookup('env', 'USER') | default('unknown') }}
          - Home: {{ lookup('env', 'HOME') | default('unknown') }}
          - Path: {{ lookup('env', 'PATH')[:50] }}...
          - Shell: {{ lookup('env', 'SHELL') | default('unknown') }}
    
    - name: Read custom environment variables
      debug:
        msg: |
          Custom Environment Variables:
          - App Name: {{ lookup('env', 'APP_NAME') | default('not_set') }}
          - App Version: {{ lookup('env', 'APP_VERSION') | default('not_set') }}
          - Database URL: {{ lookup('env', 'DATABASE_URL') | default('not_set') }}
      environment: "{{ test_env_vars }}"
    
    # Combine file and environment lookups
    - name: Create dynamic configuration using lookups
      copy:
        content: |
          # Dynamic Configuration
          # Generated from file and environment lookups
          
          [application]
          name={{ app_config.application.name }}
          version={{ app_config.application.version }}
          port={{ app_config.application.port }}
          environment={{ lookup('env', 'ENVIRONMENT') | default('development') }}
          
          [database]
          host={{ app_config.application.database.host }}
          port={{ app_config.application.database.port }}
          name={{ app_config.application.database.name }}
          password={{ secrets.database_password }}
          
          [features]
          {% for feature, enabled in app_config.features.items() %}
          {{ feature }}={{ enabled | lower }}
          {% endfor %}
          
          [runtime]
          user={{ lookup('env', 'USER') }}
          generated_at={{ ansible_date_time.iso8601 }}
        dest: "/tmp/dynamic-config.conf"
        mode: '0600'  # Secure permissions for secrets
```

### Run file and environment lookups:
```bash
# Set some environment variables for testing
export ENVIRONMENT=production
export LOG_LEVEL=DEBUG

# Run file and environment lookups
ansible-playbook -i inventory.ini file-env-lookups.yml

# Check generated configuration
cat /tmp/dynamic-config.conf
```

## Exercise 2: Variable and Template Lookups (8 minutes)

### Task: Use vars and template lookups for advanced configuration management

Create `vars-template-lookups.yml`:
```yaml
---
- name: Variable and template lookup demonstrations
  hosts: all
  vars:
    # Global application settings
    global_config:
      app_name: "nginx-proxy"
      environment: "production"
      version: "2.1.0"
      
    # Host-specific backend configurations
    backend_configs:
      web01:
        backend_servers:
          - host: "192.168.1.10"
            port: 8080
            weight: 3
          - host: "192.168.1.11"
            port: 8080
            weight: 2
        listen_port: 80
        server_name: "web01.example.com"
      web02:
        backend_servers:
          - host: "192.168.1.12"
            port: 8080
            weight: 2
          - host: "192.168.1.13"
            port: 8080
            weight: 3
        listen_port: 80
        server_name: "web02.example.com"
      db01:
        backend_servers:
          - host: "192.168.1.20"
            port: 5432
            weight: 1
        listen_port: 5432
        server_name: "db01.example.com"
        
    # Service-specific variables
    service_configs:
      nginx:
        worker_processes: 4
        worker_connections: 1024
        keepalive_timeout: 65
      postgresql:
        max_connections: 200
        shared_buffers: "256MB"
        effective_cache_size: "1GB"
        
  tasks:
    # Variable lookup examples
    - name: Use vars lookup to access nested configuration
      debug:
        msg: |
          Variable Lookups:
          - App name: {{ lookup('vars', 'global_config')['app_name'] }}
          - Environment: {{ lookup('vars', 'global_config')['environment'] }}
          - Version: {{ lookup('vars', 'global_config')['version'] }}
    
    - name: Access host-specific configuration using vars lookup
      debug:
        msg: |
          Host-specific Configuration for {{ inventory_hostname }}:
          - Server name: {{ lookup('vars', 'backend_configs')[inventory_hostname]['server_name'] }}
          - Listen port: {{ lookup('vars', 'backend_configs')[inventory_hostname]['listen_port'] }}
          - Backend servers: {{ lookup('vars', 'backend_configs')[inventory_hostname]['backend_servers'] | length }}
      when: inventory_hostname in backend_configs
    
    - name: Dynamic variable lookup based on conditions
      set_fact:
        service_type: "{{ 'database' if 'db' in inventory_hostname else 'web' }}"
        service_config_key: "{{ 'postgresql' if 'db' in inventory_hostname else 'nginx' }}"
    
    - name: Use dynamic variable lookup
      debug:
        msg: |
          Dynamic Service Configuration:
          - Service type: {{ service_type }}
          - Config key: {{ service_config_key }}
          - Configuration: {{ lookup('vars', 'service_configs')[service_config_key] }}
    
    # Template lookup examples
    - name: Generate configuration using template lookup
      set_fact:
        nginx_config: "{{ lookup('template', 'templates/nginx.conf.j2') }}"
      vars:
        app_name: "{{ global_config.app_name }}"
        backend_servers: "{{ backend_configs[inventory_hostname]['backend_servers'] }}"
        listen_port: "{{ backend_configs[inventory_hostname]['listen_port'] }}"
        server_name: "{{ backend_configs[inventory_hostname]['server_name'] }}"
      when: inventory_hostname in backend_configs
    
    - name: Save generated configuration
      copy:
        content: "{{ nginx_config }}"
        dest: "/tmp/{{ inventory_hostname }}-nginx.conf"
        mode: '0644'
      when: 
        - inventory_hostname in backend_configs
        - nginx_config is defined
    
    # Advanced template lookup with dynamic variables
    - name: Create dynamic template content
      copy:
        content: |
          # Dynamic Service Configuration Template
          # Service: {{ service_type }}
          
          {% if service_type == 'web' %}
          [nginx]
          worker_processes={{ worker_processes }}
          worker_connections={{ worker_connections }}
          keepalive_timeout={{ keepalive_timeout }}
          
          upstream backend {
          {% for server in backend_servers %}
              server {{ server.host }}:{{ server.port }} weight={{ server.weight }};
          {% endfor %}
          }
          {% elif service_type == 'database' %}
          [postgresql]
          max_connections={{ max_connections }}
          shared_buffers={{ shared_buffers }}
          effective_cache_size={{ effective_cache_size }}
          
          listen_addresses = '*'
          port = {{ listen_port }}
          {% endif %}
          
          # Generated at {{ ansible_date_time.iso8601 }}
        dest: "/tmp/dynamic-service-template.j2"
        mode: '0644'
    
    - name: Use dynamic template lookup
      set_fact:
        dynamic_service_config: "{{ lookup('template', '/tmp/dynamic-service-template.j2') }}"
      vars:
        # Merge service-specific and host-specific variables
        worker_processes: "{{ service_configs[service_config_key]['worker_processes'] | default(2) }}"
        worker_connections: "{{ service_configs[service_config_key]['worker_connections'] | default(1024) }}"
        keepalive_timeout: "{{ service_configs[service_config_key]['keepalive_timeout'] | default(60) }}"
        max_connections: "{{ service_configs[service_config_key]['max_connections'] | default(100) }}"
        shared_buffers: "{{ service_configs[service_config_key]['shared_buffers'] | default('128MB') }}"
        effective_cache_size: "{{ service_configs[service_config_key]['effective_cache_size'] | default('512MB') }}"
        backend_servers: "{{ backend_configs[inventory_hostname]['backend_servers'] | default([]) }}"
        listen_port: "{{ backend_configs[inventory_hostname]['listen_port'] | default(80) }}"
      when: inventory_hostname in backend_configs
    
    - name: Save dynamic service configuration
      copy:
        content: "{{ dynamic_service_config }}"
        dest: "/tmp/{{ inventory_hostname }}-service.conf"
        mode: '0644'
      when: dynamic_service_config is defined
    
    # Combine multiple lookups
    - name: Create comprehensive configuration using multiple lookups
      copy:
        content: |
          # Comprehensive Configuration
          # Generated using multiple lookup types
          
          [global]
          app_name={{ lookup('vars', 'global_config')['app_name'] }}
          version={{ lookup('vars', 'global_config')['version'] }}
          environment={{ lookup('vars', 'global_config')['environment'] }}
          
          [host]
          hostname={{ inventory_hostname }}
          ansible_user={{ lookup('env', 'USER') | default('ansible') }}
          
          [secrets]
          # Loaded from file lookup
          api_key={{ lookup('file', 'data/secrets.yml') | from_yaml | json_query('api_key') }}
          
          [runtime]
          generated_at={{ ansible_date_time.iso8601 }}
          generated_by={{ lookup('env', 'USER') | default('ansible') }}
        dest: "/tmp/{{ inventory_hostname }}-comprehensive.conf"
        mode: '0600'
```

### Run variable and template lookups:
```bash
# Run variable and template lookups
ansible-playbook -i inventory.ini vars-template-lookups.yml

# Check generated configurations
ls -la /tmp/*-nginx.conf
ls -la /tmp/*-service.conf
cat /tmp/web01-nginx.conf
cat /tmp/db01-service.conf
```

## Exercise 3: Lookup Error Handling and Defaults (7 minutes)

### Task: Implement robust lookup patterns with error handling

Create `lookup-error-handling.yml`:
```yaml
---
- name: Lookup error handling and defaults
  hosts: localhost
  vars:
    # Configuration files that may or may not exist
    optional_configs:
      - "data/optional-config.json"
      - "data/missing-config.yml"
      - "data/backup-config.properties"
      
    # Environment variables that may not be set
    optional_env_vars:
      - "OPTIONAL_SETTING"
      - "BACKUP_URL"
      - "FEATURE_FLAG"
      
  tasks:
    # Safe file lookups with error handling
    - name: Attempt to read optional configuration files
      set_fact:
        config_data: |
          {% set configs = {} %}
          {% for config_file in optional_configs %}
          {% set config_content = lookup('file', config_file, errors='ignore') %}
          {% if config_content %}
          {% set _ = configs.update({config_file: config_content}) %}
          {% endif %}
          {% endfor %}
          {{ configs }}
    
    - name: Display available configuration files
      debug:
        msg: |
          Available Configuration Files:
          {% for file, content in config_data.items() %}
          - {{ file }}: {{ content[:50] }}...
          {% endfor %}
          {% if config_data | length == 0 %}
          - No optional configuration files found
          {% endif %}
    
    # Environment variable lookups with defaults
    - name: Read environment variables with fallback defaults
      debug:
        msg: |
          Environment Variables with Defaults:
          - Optional Setting: {{ lookup('env', 'OPTIONAL_SETTING') | default('default_value') }}
          - Backup URL: {{ lookup('env', 'BACKUP_URL') | default('http://backup.example.com') }}
          - Feature Flag: {{ lookup('env', 'FEATURE_FLAG') | default('false') | bool }}
          - Database URL: {{ lookup('env', 'DATABASE_URL') | default('postgresql://localhost:5432/default') }}
    
    # Create test files for demonstration
    - name: Create optional configuration file
      copy:
        content: |
          {
            "optional_feature": true,
            "timeout": 30,
            "retries": 3
          }
        dest: "data/optional-config.json"
        mode: '0644'
    
    - name: Create backup configuration
      copy:
        content: |
          # Backup Configuration
          backup_enabled=true
          backup_interval=3600
          backup_retention=7
        dest: "data/backup-config.properties"
        mode: '0644'
    
    # Re-run lookups after creating files
    - name: Re-attempt configuration file lookups
      set_fact:
        updated_config_data: |
          {% set configs = {} %}
          {% for config_file in optional_configs %}
          {% set config_content = lookup('file', config_file, errors='ignore') %}
          {% if config_content %}
          {% set _ = configs.update({config_file: config_content}) %}
          {% endif %}
          {% endfor %}
          {{ configs }}
    
    - name: Display updated configuration files
      debug:
        msg: |
          Updated Configuration Files:
          {% for file, content in updated_config_data.items() %}
          - {{ file }}: Available ({{ content | length }} characters)
          {% endfor %}
    
    # Advanced error handling with multiple fallbacks
    - name: Configuration with multiple fallback strategies
      set_fact:
        final_config: |
          {
            "database": {
              "host": "{{ lookup('env', 'DB_HOST') | default(lookup('file', 'data/db-host.txt', errors='ignore')) | default('localhost') }}",
              "port": {{ lookup('env', 'DB_PORT') | default('5432') | int }},
              "name": "{{ lookup('env', 'DB_NAME') | default('app_db') }}"
            },
            "cache": {
              "enabled": {{ lookup('env', 'CACHE_ENABLED') | default('true') | bool | lower }},
              "ttl": {{ lookup('env', 'CACHE_TTL') | default('3600') | int }}
            },
            "features": {
              "optional_feature": {{ (lookup('file', 'data/optional-config.json', errors='ignore') | from_json).optional_feature | default(false) | lower }},
              "backup_enabled": {{ 'true' if 'backup_enabled=true' in lookup('file', 'data/backup-config.properties', errors='ignore') else 'false' }}
            }
          }
    
    - name: Display final configuration with fallbacks
      debug:
        msg: |
          Final Configuration (with fallbacks):
          {{ final_config | from_json | to_nice_yaml }}
    
    # Template lookup with error handling
    - name: Create template with error handling
      copy:
        content: |
          # Configuration Template with Error Handling
          [database]
          host={{ db_host | default('localhost') }}
          port={{ db_port | default(5432) }}
          
          [application]
          name={{ app_name | default('unknown_app') }}
          version={{ app_version | default('1.0.0') }}
          
          {% if optional_settings is defined %}
          [optional]
          {% for key, value in optional_settings.items() %}
          {{ key }}={{ value }}
          {% endfor %}
          {% endif %}
          
          # Generated at {{ ansible_date_time.iso8601 }}
        dest: "/tmp/error-handling-template.j2"
        mode: '0644'
    
    - name: Use template with missing variables (should handle gracefully)
      set_fact:
        template_result: "{{ lookup('template', '/tmp/error-handling-template.j2', errors='ignore') }}"
      vars:
        db_host: "{{ lookup('env', 'DB_HOST') | default('fallback-db') }}"
        db_port: "{{ lookup('env', 'DB_PORT') | default(5432) }}"
        app_name: "error-handling-demo"
        # optional_settings intentionally not defined to test error handling
    
    - name: Display template result
      debug:
        msg: |
          Template Result:
          {{ template_result }}
    
    - name: Save final configuration
      copy:
        content: "{{ final_config }}"
        dest: "/tmp/final-config.json"
        mode: '0644'
```

### Run lookup error handling:
```bash
# Run lookup error handling
ansible-playbook -i inventory.ini lookup-error-handling.yml

# Set some environment variables and run again
export DB_HOST=production-db.example.com
export DB_PORT=5433
export CACHE_ENABLED=true
export OPTIONAL_SETTING=custom_value

ansible-playbook -i inventory.ini lookup-error-handling.yml

# Check generated files
cat /tmp/final-config.json | python3 -m json.tool
ls -la data/
```

## Verification and Discussion

### 1. Check Results
```bash
# Check file and environment lookup results
cat /tmp/dynamic-config.conf

# Check template lookup results
ls -la /tmp/*-nginx.conf
cat /tmp/web01-nginx.conf

# Check error handling results
cat /tmp/final-config.json | python3 -m json.tool
ls -la data/
```

### 2. Discussion Points
- How do you currently handle external configuration data in your automation?
- What strategies do you use for managing secrets and sensitive data?
- How do you implement fallback mechanisms for missing configuration?

### 3. Clean Up
```bash
# Remove demo files
rm -rf data/ templates/ /tmp/dynamic-config.conf /tmp/*-nginx.conf /tmp/*-service.conf /tmp/*-comprehensive.conf /tmp/final-config.json /tmp/error-handling-template.j2 /tmp/dynamic-service-template.j2 /tmp/test-env
```

## Key Takeaways
- File lookups enable reading external configuration data into Ansible
- Environment lookups provide access to system and custom environment variables
- Variable lookups allow dynamic access to Ansible variables and facts
- Template lookups enable dynamic template processing with variable substitution
- Error handling with `errors='ignore'` and defaults prevents automation failures
- Multiple lookup types can be combined for comprehensive configuration management

## Next Steps
Proceed to Lab 3.2: Advanced Data Lookups

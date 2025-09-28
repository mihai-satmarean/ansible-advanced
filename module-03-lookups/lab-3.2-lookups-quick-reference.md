# Lab 3.2: Advanced Lookups (Quick Reference)

## Objective
Advanced lookup patterns and enterprise use cases for self-study

## Target Audience
**Victor & Vlad** - Intermediate participants for post-course exploration

## Duration
20-30 minutes (self-paced)

## Pattern 1: Network and URL Lookups

### DNS and Network Queries
```yaml
# DNS lookups for dynamic infrastructure
- name: Get server IPs from DNS
  debug:
    msg: "{{ lookup('dig', item + '.company.com') }}"
  loop:
    - web01
    - db01
    - cache01

# Reverse DNS lookups
- name: Get hostnames from IPs
  debug:
    msg: "{{ lookup('dig', item + '/PTR') }}"
  loop:
    - "192.168.1.10"
    - "192.168.1.20"

# MX record lookup for email configuration
- name: Get mail servers
  debug:
    msg: "Mail servers: {{ lookup('dig', 'company.com/MX') }}"
```

### URL and API Lookups
```yaml
# Fetch configuration from REST API
- name: Get application configuration from API
  set_fact:
    app_config: "{{ lookup('url', 'https://config-api.company.com/app/config') | from_json }}"

# Fetch with authentication
- name: Get secure configuration
  set_fact:
    secure_config: "{{ lookup('url', 'https://api.company.com/config', headers={'Authorization': 'Bearer ' + api_token}) | from_json }}"

# Download and use file content
- name: Get latest configuration template
  template:
    src: "{{ lookup('url', 'https://templates.company.com/nginx.j2') }}"
    dest: /etc/nginx/nginx.conf
```

## Pattern 2: Advanced File and Data Processing

### Multi-File Processing
```yaml
# Process multiple configuration files
- name: Merge configuration files
  set_fact:
    merged_config: |
      {% for file in config_files %}
      # Configuration from {{ file }}
      {{ lookup('file', file) }}
      
      {% endfor %}
  vars:
    config_files:
      - "configs/base.conf"
      - "configs/{{ environment }}.conf"
      - "configs/{{ ansible_hostname }}.conf"

# Read files with error handling
- name: Read optional configuration files
  set_fact:
    optional_config: "{{ lookup('file', item, errors='ignore') | default('') }}"
  loop:
    - "configs/optional.conf"
    - "configs/local.conf"
  when: lookup('file', item, errors='ignore') != ""
```

### CSV Data Processing
```yaml
# Advanced CSV processing with multiple columns
- name: Process server inventory from CSV
  set_fact:
    server_inventory: |
      {% for server in server_list %}
      {{ server }}:
        ip: {{ lookup('csvfile', server + ' file=data/servers.csv delimiter=, col=1') }}
        role: {{ lookup('csvfile', server + ' file=data/servers.csv delimiter=, col=2') }}
        environment: {{ lookup('csvfile', server + ' file=data/servers.csv delimiter=, col=3') }}
        cpu_cores: {{ lookup('csvfile', server + ' file=data/servers.csv delimiter=, col=4') }}
        memory_gb: {{ lookup('csvfile', server + ' file=data/servers.csv delimiter=, col=5') }}
      {% endfor %}
  vars:
    server_list: ["web01", "web02", "db01", "cache01"]

# Generate dynamic inventory from CSV
- name: Create dynamic groups from CSV data
  add_host:
    name: "{{ item }}"
    groups: "{{ lookup('csvfile', item + ' file=data/servers.csv delimiter=, col=2') }}"
    ansible_host: "{{ lookup('csvfile', item + ' file=data/servers.csv delimiter=, col=1') }}"
  loop: "{{ server_list }}"
```

## Pattern 3: Complex Command and Pipe Operations

### System Information Gathering
```yaml
# Gather complex system metrics
- name: Collect system performance data
  set_fact:
    system_metrics:
      cpu_usage: "{{ lookup('pipe', 'top -bn1 | grep \"Cpu(s)\" | awk \"{print $2}\" | cut -d\"%\" -f1') }}"
      memory_usage: "{{ lookup('pipe', 'free | grep Mem | awk \"{printf \"%.2f\", $3/$2 * 100.0}\"') }}"
      disk_usage: "{{ lookup('pipe', 'df -h / | tail -1 | awk \"{print $5}\" | cut -d\"%\" -f1') }}"
      load_average: "{{ lookup('pipe', 'uptime | awk -F\"load average:\" \"{print $2}\"') }}"

# Process log files
- name: Extract error patterns from logs
  set_fact:
    recent_errors: "{{ lookup('pipe', 'tail -1000 /var/log/application.log | grep ERROR | tail -10') }}"
    error_count: "{{ lookup('pipe', 'grep -c ERROR /var/log/application.log') }}"
```

### Database Queries
```yaml
# Query database for configuration
- name: Get database configuration
  set_fact:
    db_config: "{{ lookup('pipe', 'mysql -u admin -p' + db_password + ' -e \"SELECT config_key, config_value FROM app_config;\" --batch --raw') }}"

# Process database results
- name: Parse database configuration
  set_fact:
    parsed_config: |
      {% for line in db_config.split('\n')[1:] %}
      {% set key, value = line.split('\t') %}
      {{ key }}={{ value }}
      {% endfor %}
```

## Pattern 4: Lookup Optimization and Caching

### Caching Expensive Lookups
```yaml
# Cache API responses
- name: Cache API configuration (run once)
  set_fact:
    cached_api_config: "{{ lookup('url', api_endpoint) | from_json }}"
  run_once: true
  delegate_to: localhost

# Use cached data across hosts
- name: Use cached configuration
  template:
    src: app-config.j2
    dest: /etc/app/config.yml
  vars:
    api_config: "{{ hostvars['localhost']['cached_api_config'] }}"

# Conditional lookup execution
- name: Lookup only when needed
  set_fact:
    external_config: "{{ lookup('url', config_url) | from_json }}"
  when: 
    - external_config is not defined
    - use_external_config | default(false)
```

### Lookup Error Handling
```yaml
# Graceful error handling
- name: Safe file lookup with fallback
  set_fact:
    config_content: "{{ lookup('file', config_file, errors='ignore') | default(default_config) }}"
  vars:
    config_file: "configs/{{ environment }}.conf"
    default_config: |
      # Default configuration
      debug=false
      log_level=INFO

# Multiple fallback sources
- name: Configuration with multiple fallbacks
  set_fact:
    final_config: "{{ config_source }}"
  vars:
    config_source: |
      {% set sources = [
        lookup('file', 'configs/local.conf', errors='ignore'),
        lookup('env', 'APP_CONFIG', errors='ignore'),
        lookup('file', 'configs/default.conf', errors='ignore'),
        default_config
      ] %}
      {% for source in sources %}
      {% if source %}
      {{ source }}
      {% break %}
      {% endif %}
      {% endfor %}
```

## Pattern 5: Dynamic Inventory and Service Discovery

### Service Discovery Integration
```yaml
# Consul service discovery
- name: Discover services from Consul
  set_fact:
    consul_services: "{{ lookup('url', 'http://consul.company.com:8500/v1/catalog/services') | from_json }}"

# Generate service configuration
- name: Configure load balancer from service discovery
  template:
    src: haproxy.j2
    dest: /etc/haproxy/haproxy.cfg
  vars:
    backend_servers: |
      {% for service in consul_services %}
      {% set instances = lookup('url', 'http://consul.company.com:8500/v1/health/service/' + service) | from_json %}
      {% for instance in instances %}
      {% if instance.Checks | selectattr('Status', 'equalto', 'passing') | list %}
      server {{ instance.Service.ID }} {{ instance.Service.Address }}:{{ instance.Service.Port }} check
      {% endif %}
      {% endfor %}
      {% endfor %}
```

### Dynamic AWS Resource Discovery
```yaml
# AWS resource discovery (requires aws CLI)
- name: Get EC2 instances by tag
  set_fact:
    web_servers: "{{ lookup('pipe', 'aws ec2 describe-instances --filters \"Name=tag:Role,Values=webserver\" --query \"Reservations[].Instances[].PrivateIpAddress\" --output text').split() }}"

# Generate inventory from AWS
- name: Create dynamic inventory from AWS
  add_host:
    name: "{{ item }}"
    groups: "webservers"
    ansible_host: "{{ item }}"
  loop: "{{ web_servers }}"
```

## Pattern 6: Configuration Management Patterns

### Environment-Specific Configuration
```yaml
# Multi-environment configuration management
- name: Load environment-specific settings
  set_fact:
    env_config: |
      {% set config_files = [
        'configs/global.yml',
        'configs/' + environment + '.yml',
        'configs/' + environment + '-' + ansible_hostname + '.yml'
      ] %}
      {% for file in config_files %}
      {% set content = lookup('file', file, errors='ignore') %}
      {% if content %}
      {{ content }}
      ---
      {% endif %}
      {% endfor %}

# Merge YAML configurations
- name: Merge configuration files
  set_fact:
    merged_config: "{{ base_config | combine(env_config, recursive=True) }}"
  vars:
    base_config: "{{ lookup('file', 'configs/base.yml') | from_yaml }}"
    env_config: "{{ lookup('file', 'configs/' + environment + '.yml') | from_yaml }}"
```

### Secret Management Integration
```yaml
# HashiCorp Vault integration
- name: Get secrets from Vault
  set_fact:
    vault_secrets: "{{ lookup('url', vault_url + '/v1/secret/data/app', headers={'X-Vault-Token': vault_token}) | from_json }}"

# AWS Secrets Manager
- name: Get secrets from AWS Secrets Manager
  set_fact:
    aws_secrets: "{{ lookup('pipe', 'aws secretsmanager get-secret-value --secret-id app/database --query SecretString --output text') | from_json }}"

# Use secrets in configuration
- name: Generate secure configuration
  template:
    src: secure-config.j2
    dest: /etc/app/config.yml
    mode: '0600'
  vars:
    database_password: "{{ vault_secrets.data.data.db_password }}"
    api_key: "{{ aws_secrets.api_key }}"
```

## Performance and Best Practices

### Lookup Optimization Tips:
1. **Cache expensive lookups** - Use `set_fact` and `run_once`
2. **Error handling** - Always use `errors='ignore'` with fallbacks
3. **Minimize API calls** - Batch requests when possible
4. **Use appropriate lookup types** - Choose the right tool for the job
5. **Validate results** - Check lookup results before using

### Security Considerations:
- Never log sensitive lookup results
- Use Ansible Vault for API tokens and credentials
- Validate external data sources
- Implement proper error handling for security
- Use HTTPS for URL lookups

### Common Anti-Patterns:
```yaml
# DON'T: Lookup in loops without caching
- name: Bad - repeated API calls
  debug:
    msg: "{{ lookup('url', api_endpoint + '/' + item) }}"
  loop: "{{ large_list }}"

# DO: Cache once, use many times
- name: Good - cache API response
  set_fact:
    api_data: "{{ lookup('url', api_endpoint) | from_json }}"
  run_once: true

- name: Use cached data
  debug:
    msg: "{{ api_data[item] }}"
  loop: "{{ large_list }}"
```

## Troubleshooting Lookups

### Common Issues:
1. **File not found** - Use `errors='ignore'` and provide defaults
2. **Network timeouts** - Implement retry logic and timeouts
3. **Permission denied** - Check file permissions and user context
4. **Invalid data format** - Validate and sanitize lookup results

### Debug Techniques:
```yaml
# Debug lookup results
- name: Debug file lookup
  debug:
    var: lookup('file', 'config.txt', errors='ignore')

# Test lookup availability
- name: Test API availability
  uri:
    url: "{{ api_endpoint }}"
    method: HEAD
  register: api_check
  failed_when: false

- name: Use API only if available
  set_fact:
    config: "{{ lookup('url', api_endpoint) | from_json }}"
  when: api_check.status == 200
```

---

**Note**: This reference covers advanced lookup patterns for experienced users. Start with basic lookups and gradually incorporate these advanced techniques as your automation requirements grow in complexity.

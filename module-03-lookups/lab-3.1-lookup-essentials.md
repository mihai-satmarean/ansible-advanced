# Lab 3.1: Lookup Essentials (Interactive Demo)

## Objective
Master essential Ansible lookup plugins through interactive demonstration and practical examples suitable for all skill levels.

## Duration
20 minutes (interactive demo)

## Format
**Interactive demonstration** - Instructor-led with participant practice

## Prerequisites
- Completed Module 1 and 2 labs
- Understanding of Ansible variables

## Demo Setup

```bash
cd ~/ansible-labs
mkdir -p module-03
cd module-03

# Create simple inventory
cat > inventory.ini << 'EOF'
[demo_hosts]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF

# Create test data files
mkdir -p data
```

## Demo 1: File and Environment Lookups (6 minutes)

### What are Lookup Plugins?
**Instructor explains:**
- Lookups retrieve data from external sources
- Execute on the Ansible controller (not target hosts)
- Return data that can be used in variables, tasks, templates
- Essential for dynamic configurations

### File Lookup Demo
```bash
# Create sample configuration files
cat > data/database-config.txt << 'EOF'
host=prod-db.company.com
port=5432
database=webapp
max_connections=100
EOF

cat > data/api-key.txt << 'EOF'
sk-1234567890abcdef
EOF
```

```yaml
# Create file-lookup-demo.yml
cat > file-lookup-demo.yml << 'EOF'
---
- name: File Lookup Demo
  hosts: demo_hosts
  
  tasks:
    - name: Read database configuration
      debug:
        msg: "Database config: {{ lookup('file', 'data/database-config.txt') }}"
    
    - name: Read API key (secure)
      debug:
        msg: "API key loaded: {{ 'YES' if lookup('file', 'data/api-key.txt') else 'NO' }}"
    
    - name: Use file content in configuration
      copy:
        content: |
          # Application Configuration
          # Database settings from external file:
          {{ lookup('file', 'data/database-config.txt') }}
          
          # API Configuration
          api_key={{ lookup('file', 'data/api-key.txt') }}
        dest: /tmp/app-config.conf
        
    - name: Show generated config
      debug:
        msg: "Configuration created in /tmp/app-config.conf"
EOF

# Run file lookup demo
ansible-playbook -i inventory.ini file-lookup-demo.yml
cat /tmp/app-config.conf
```

### Environment Variable Lookup
```yaml
# Create env-lookup-demo.yml
cat > env-lookup-demo.yml << 'EOF'
---
- name: Environment Variable Lookup Demo
  hosts: demo_hosts
  
  vars:
    # Get environment variables with defaults
    app_environment: "{{ lookup('env', 'APP_ENV') | default('development') }}"
    database_url: "{{ lookup('env', 'DATABASE_URL') | default('localhost:5432') }}"
    debug_mode: "{{ lookup('env', 'DEBUG') | default('false') }}"
    
  tasks:
    - name: Show environment-based configuration
      debug:
        msg: |
          Environment: {{ app_environment }}
          Database: {{ database_url }}
          Debug: {{ debug_mode }}
    
    - name: Generate environment-specific config
      copy:
        content: |
          [application]
          environment={{ app_environment }}
          debug={{ debug_mode }}
          
          [database]
          url={{ database_url }}
          
          [runtime]
          user={{ lookup('env', 'USER') }}
          home={{ lookup('env', 'HOME') }}
        dest: /tmp/env-config.ini
EOF

# Set some environment variables and run
export APP_ENV=production
export DEBUG=true
ansible-playbook -i inventory.ini env-lookup-demo.yml
cat /tmp/env-config.ini
```

## Demo 2: Variable and Template Lookups (7 minutes)

### Variable Lookup Demo
```yaml
# Create vars-lookup-demo.yml
cat > vars-lookup-demo.yml << 'EOF'
---
- name: Variable Lookup Demo
  hosts: demo_hosts
  
  vars:
    app_settings:
      web:
        port: 8080
        workers: 4
      database:
        host: "db.company.com"
        port: 5432
      cache:
        enabled: true
        ttl: 3600
        
    # Dynamic variable names
    current_service: "web"
    config_key: "port"
    
  tasks:
    - name: Access nested variables dynamically
      debug:
        msg: |
          Service: {{ current_service }}
          Setting: {{ config_key }}
          Value: {{ lookup('vars', 'app_settings')[current_service][config_key] }}
    
    - name: Loop through service configurations
      debug:
        msg: "{{ service }}: {{ lookup('vars', 'app_settings')[service] }}"
      loop:
        - web
        - database
        - cache
      loop_control:
        loop_var: service
    
    - name: Generate service configurations
      copy:
        content: |
          # {{ service | title }} Service Configuration
          {% for key, value in lookup('vars', 'app_settings')[service].items() %}
          {{ key }}={{ value }}
          {% endfor %}
        dest: "/tmp/{{ service }}-config.conf"
      loop:
        - web
        - database
        - cache
      loop_control:
        loop_var: service
EOF

# Run variable lookup demo
ansible-playbook -i inventory.ini vars-lookup-demo.yml
ls -la /tmp/*-config.conf
cat /tmp/web-config.conf
```

### Template Lookup Demo
```bash
# Create a template file
cat > data/service-template.j2 << 'EOF'
# {{ service_name }} Service Configuration
# Generated: {{ ansible_date_time.iso8601 }}

[service]
name={{ service_name }}
port={{ service_port }}
workers={{ service_workers | default(2) }}

[logging]
level={{ log_level | default('INFO') }}
file=/var/log/{{ service_name }}.log

{% if service_name == 'web' %}
[web_specific]
static_files=/var/www/static
{% elif service_name == 'api' %}
[api_specific]
rate_limit=1000
{% endif %}
EOF
```

```yaml
# Create template-lookup-demo.yml
cat > template-lookup-demo.yml << 'EOF'
---
- name: Template Lookup Demo
  hosts: demo_hosts
  
  tasks:
    - name: Generate configurations using template lookup
      copy:
        content: "{{ lookup('template', 'data/service-template.j2') }}"
        dest: "/tmp/{{ item.name }}-service.conf"
      loop:
        - name: "web"
          port: 8080
          workers: 4
          log_level: "INFO"
        - name: "api"
          port: 3000
          workers: 2
          log_level: "DEBUG"
      vars:
        service_name: "{{ item.name }}"
        service_port: "{{ item.port }}"
        service_workers: "{{ item.workers }}"
        log_level: "{{ item.log_level }}"
        
    - name: Show generated configurations
      debug:
        msg: "Generated {{ item }}-service.conf"
      loop:
        - web
        - api
EOF

# Run template lookup demo
ansible-playbook -i inventory.ini template-lookup-demo.yml
cat /tmp/web-service.conf
echo "---"
cat /tmp/api-service.conf
```

## Demo 3: Practical Lookup Patterns (7 minutes)

### CSV Data Lookup
```bash
# Create CSV data file
cat > data/servers.csv << 'EOF'
hostname,ip,role,environment
web01,192.168.1.10,webserver,production
web02,192.168.1.11,webserver,production
db01,192.168.1.20,database,production
cache01,192.168.1.30,cache,production
EOF
```

```yaml
# Create csv-lookup-demo.yml
cat > csv-lookup-demo.yml << 'EOF'
---
- name: CSV Lookup Demo
  hosts: demo_hosts
  
  tasks:
    - name: Read server information from CSV
      debug:
        msg: |
          Server: {{ item }}
          IP: {{ lookup('csvfile', item + ' file=data/servers.csv delimiter=, col=1') }}
          Role: {{ lookup('csvfile', item + ' file=data/servers.csv delimiter=, col=2') }}
          Environment: {{ lookup('csvfile', item + ' file=data/servers.csv delimiter=, col=3') }}
      loop:
        - web01
        - db01
        - cache01
    
    - name: Generate hosts file from CSV
      copy:
        content: |
          # Hosts file generated from CSV data
          {% for server in ['web01', 'web02', 'db01', 'cache01'] %}
          {{ lookup('csvfile', server + ' file=data/servers.csv delimiter=, col=1') }} {{ server }}.company.com {{ server }}
          {% endfor %}
        dest: /tmp/hosts-from-csv
        
    - name: Show generated hosts file
      debug:
        msg: "Hosts file generated from CSV data"
EOF

# Run CSV lookup demo
ansible-playbook -i inventory.ini csv-lookup-demo.yml
cat /tmp/hosts-from-csv
```

### Pipe and Command Lookups
```yaml
# Create pipe-lookup-demo.yml
cat > pipe-lookup-demo.yml << 'EOF'
---
- name: Pipe and Command Lookup Demo
  hosts: demo_hosts
  
  tasks:
    - name: Get system information using pipe lookup
      debug:
        msg: |
          Current date: {{ lookup('pipe', 'date') }}
          Disk usage: {{ lookup('pipe', 'df -h / | tail -1') }}
          Memory info: {{ lookup('pipe', 'free -h | grep Mem') }}
    
    - name: Generate system report
      copy:
        content: |
          # System Report
          # Generated: {{ lookup('pipe', 'date') }}
          
          ## System Information
          Hostname: {{ lookup('pipe', 'hostname') }}
          Uptime: {{ lookup('pipe', 'uptime') }}
          
          ## Disk Usage
          {{ lookup('pipe', 'df -h') }}
          
          ## Memory Usage
          {{ lookup('pipe', 'free -h') }}
          
          ## Network Interfaces
          {{ lookup('pipe', 'ip addr show | grep inet') }}
        dest: /tmp/system-report.txt
        
    - name: Show report location
      debug:
        msg: "System report generated in /tmp/system-report.txt"
EOF

# Run pipe lookup demo
ansible-playbook -i inventory.ini pipe-lookup-demo.yml
head -15 /tmp/system-report.txt
```

## Interactive Practice Session

### Hands-On Exercise for Participants
```yaml
# Create practice exercise
cat > practice-lookups.yml << 'EOF'
---
- name: Participant Practice - Lookup Plugins
  hosts: demo_hosts
  
  vars:
    # TODO: Participants will work with these variables
    app_name: "MyWebApp"
    environments: ["dev", "staging", "prod"]
    
  tasks:
    # TODO: Participants create tasks using lookups:
    # 1. Read a configuration file using file lookup
    # 2. Use environment variables with defaults
    # 3. Generate configuration using template lookup
    # 4. Use pipe lookup to get system information
    
    - name: Example task - participants modify this
      debug:
        msg: "TODO: Use lookup plugins here"
        
    # Hint: Create data/app-config.txt file first
    # Hint: Use lookup('file', 'data/app-config.txt')
    # Hint: Use lookup('env', 'USER') | default('unknown')
EOF
```

**Practice Instructions:**
1. Create a data file with application configuration
2. Use file lookup to read the configuration
3. Use environment lookup with defaults
4. Create a template and use template lookup
5. Use pipe lookup to get current user or date

## Key Takeaways Summary

### Essential Lookup Plugins:
1. **file** - Read file contents
2. **env** - Get environment variables
3. **vars** - Access Ansible variables dynamically
4. **template** - Process Jinja2 templates
5. **csvfile** - Read CSV data
6. **pipe** - Execute shell commands

### Common Patterns:
- **Configuration files** - `lookup('file', 'config.txt')`
- **Environment settings** - `lookup('env', 'VAR') | default('fallback')`
- **Dynamic variables** - `lookup('vars', variable_name)`
- **Data processing** - `lookup('csvfile', 'key file=data.csv')`
- **System information** - `lookup('pipe', 'command')`

### Best Practices:
- Always provide defaults for optional data
- Use lookups for external data sources
- Keep lookup expressions simple and readable
- Cache lookup results in variables when used multiple times
- Validate lookup results before using them

### When to Use Lookups:
- **External configuration** - Files, databases, APIs
- **Environment-specific settings** - Different per environment
- **Dynamic data** - Generated at runtime
- **System integration** - OS commands, network queries
- **Data transformation** - Processing external formats

## Discussion Points

### For Alexandra & Gabriela (Beginners):
- "How would lookups help with your configuration management?"
- "What external data sources do you currently use manually?"
- "When would you use file lookup vs. environment lookup?"

### For Victor & Vlad (Intermediate):
- "How do you handle sensitive data in lookups?"
- "What's your strategy for lookup error handling?"
- "How do you integrate lookups with CI/CD pipelines?"

## Demo Cleanup
```bash
# Clean up demo files
rm -f /tmp/app-config.conf /tmp/env-config.ini /tmp/*-config.conf /tmp/*-service.conf /tmp/hosts-from-csv /tmp/system-report.txt
```

---

**Note for Instructor**: Focus on practical, immediately applicable lookup patterns. Emphasize the power of external data integration. Adjust complexity based on participant experience and questions.

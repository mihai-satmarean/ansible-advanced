# Lab 6.1: Jinja2 Essentials (Interactive Demo)

## Objective
Master essential Jinja2 templating concepts through interactive demonstration and practical examples suitable for all skill levels.


## Format
**Interactive demonstration** - Instructor-led with participant practice

## Prerequisites
- Basic understanding of YAML and variables
- Familiarity with Ansible playbooks

## Demo Setup

```bash
cd ~/ansible-labs
mkdir -p module-06
cd module-06

# Create simple inventory
cat > inventory.ini << 'EOF'
[demo_hosts]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Demo 1: Jinja2 Basics (8 minutes)

### Variables and Basic Syntax
```yaml
# Create demo-basics.yml
cat > demo-basics.yml << 'EOF'
---
- name: Jinja2 Basics Demo
  hosts: demo_hosts
  vars:
    app_name: "MyWebApp"
    app_version: "2.1.0"
    server_port: 8080
    debug_mode: true
    
  tasks:
    - name: Basic variable substitution
      debug:
        msg: "Application: {{ app_name }} version {{ app_version }}"
    
    - name: Basic template example
      copy:
        content: |
          # {{ app_name }} Configuration
          # Generated on {{ ansible_date_time.iso8601 }}
          
          [application]
          name={{ app_name }}
          version={{ app_version }}
          port={{ server_port }}
          debug={{ debug_mode | lower }}
          
          [server]
          hostname={{ ansible_hostname }}
          ip_address={{ ansible_default_ipv4.address | default('127.0.0.1') }}
        dest: /tmp/basic-config.ini
        
    - name: Show created file
      debug:
        msg: "Check /tmp/basic-config.ini for the generated configuration"
EOF

# Run basic demo
ansible-playbook -i inventory.ini demo-basics.yml
cat /tmp/basic-config.ini
```

### Key Concepts Explained:
- **Variable syntax**: `{{ variable_name }}`
- **Filters**: `{{ variable | filter_name }}`
- **Default values**: `{{ variable | default('fallback') }}`
- **Comments**: `{# This is a comment #}`

## Demo 2: Essential Filters (8 minutes)

### Common Filters Demo
```yaml
# Create demo-filters.yml
cat > demo-filters.yml << 'EOF'
---
- name: Essential Jinja2 Filters Demo
  hosts: demo_hosts
  vars:
    user_list: ["alice", "bob", "charlie"]
    server_data:
      cpu_cores: 4
      memory_gb: 16
      disk_space: "500GB"
    config_text: "  MyApp Configuration  "
    
  tasks:
    - name: String filters
      debug:
        msg: |
          Original: '{{ config_text }}'
          Upper: '{{ config_text | upper }}'
          Lower: '{{ config_text | lower }}'
          Strip: '{{ config_text | trim }}'
          Replace: '{{ config_text | replace("MyApp", "WebApp") }}'
    
    - name: List filters
      debug:
        msg: |
          Users: {{ user_list }}
          Count: {{ user_list | length }}
          First: {{ user_list | first }}
          Last: {{ user_list | last }}
          Joined: {{ user_list | join(', ') }}
    
    - name: Dictionary filters
      debug:
        msg: |
          Keys: {{ server_data.keys() | list }}
          Values: {{ server_data.values() | list }}
          CPU: {{ server_data.cpu_cores }}
    
    - name: Create user list file
      copy:
        content: |
          # User List - Generated {{ ansible_date_time.date }}
          {% for user in user_list %}
          User {{ loop.index }}: {{ user | title }}
          {% endfor %}
          
          Total users: {{ user_list | length }}
        dest: /tmp/user-list.txt
EOF

# Run filters demo
ansible-playbook -i inventory.ini demo-filters.yml
cat /tmp/user-list.txt
```

### Essential Filters Covered:
- **String**: `upper`, `lower`, `trim`, `replace`
- **List**: `length`, `first`, `last`, `join`
- **Dictionary**: `keys()`, `values()`
- **Default**: `default('fallback')`

## Demo 3: Control Structures (9 minutes)

### Loops and Conditionals
```yaml
# Create demo-control.yml
cat > demo-control.yml << 'EOF'
---
- name: Jinja2 Control Structures Demo
  hosts: demo_hosts
  vars:
    environments:
      - name: "development"
        port: 3000
        debug: true
      - name: "staging" 
        port: 4000
        debug: true
      - name: "production"
        port: 80
        debug: false
    
    features:
      ssl_enabled: true
      monitoring: true
      backup: false
      
  tasks:
    - name: Generate environment configurations
      copy:
        content: |
          # Multi-Environment Configuration
          # Generated: {{ ansible_date_time.iso8601 }}
          
          {% for env in environments %}
          [{{ env.name }}]
          port={{ env.port }}
          debug={{ env.debug | lower }}
          {% if env.debug %}
          log_level=DEBUG
          {% else %}
          log_level=INFO
          {% endif %}
          
          {% endfor %}
          
          # Feature Configuration
          {% for feature, enabled in features.items() %}
          {% if enabled %}
          {{ feature }}=enabled
          {% else %}
          {{ feature }}=disabled
          {% endif %}
          {% endfor %}
        dest: /tmp/multi-env-config.ini
        
    - name: Generate nginx-style config
      copy:
        content: |
          # Nginx Configuration Template
          {% for env in environments %}
          server {
              listen {{ env.port }};
              server_name {{ env.name }}.example.com;
              
              {% if features.ssl_enabled and env.name == 'production' %}
              ssl_certificate /etc/ssl/{{ env.name }}.crt;
              ssl_certificate_key /etc/ssl/{{ env.name }}.key;
              {% endif %}
              
              location / {
                  proxy_pass http://backend-{{ env.name }};
                  {% if env.debug %}
                  proxy_set_header X-Debug-Mode "on";
                  {% endif %}
              }
          }
          {% endfor %}
        dest: /tmp/nginx-config.conf
EOF

# Run control structures demo
ansible-playbook -i inventory.ini demo-control.yml
cat /tmp/multi-env-config.ini
echo "---"
cat /tmp/nginx-config.conf
```

### Control Structures Covered:
- **For loops**: `{% for item in list %}`
- **Conditionals**: `{% if condition %}`
- **Loop variables**: `loop.index`, `loop.first`, `loop.last`
- **Dictionary iteration**: `{% for key, value in dict.items() %}`

## Interactive Practice Session

### Hands-On Exercise for Participants:
```yaml
# Create practice-template.yml
cat > practice-template.yml << 'EOF'
---
- name: Participant Practice Session
  hosts: demo_hosts
  vars:
    # Participants will modify these variables
    my_app:
      name: "StudentApp"
      version: "1.0"
      port: 5000
    
    my_servers:
      - name: "web1"
        role: "frontend"
      - name: "db1" 
        role: "database"
        
  tasks:
    - name: Create your configuration
      copy:
        content: |
          # TODO: Participants create their template here
          # Use variables: my_app and my_servers
          # Include: app info, server list, current date
          
          Application: {{ my_app.name }}
          # Add more template content...
        dest: /tmp/my-config.txt
        
    - name: Show result
      debug:
        msg: "Check your configuration in /tmp/my-config.txt"
EOF
```

**Instructor Instructions:**
1. Show the template structure
2. Ask participants to add:
   - App version and port
   - Loop through servers
   - Add conditional for database servers
   - Include timestamp
3. Run and review results together

## Key Takeaways Summary

### Essential Jinja2 Concepts:
1. **Variables**: `{{ variable_name }}`
2. **Filters**: `{{ variable | filter }}`
3. **Loops**: `{% for item in list %}`
4. **Conditionals**: `{% if condition %}`
5. **Comments**: `{# comment #}`

### Most Useful Filters:
- `default('fallback')` - Provide fallback values
- `length` - Count items
- `join(', ')` - Combine list items
- `upper/lower` - Change case
- `trim` - Remove whitespace

### When to Use Templating:
- **Configuration files** - Dynamic configs based on environment
- **Service definitions** - Generate service files
- **Documentation** - Auto-generate docs from data
- **Scripts** - Create deployment scripts

### Best Practices:
- Keep templates simple and readable
- Use meaningful variable names
- Provide default values for optional variables
- Test templates with different data sets
- Comment complex logic


## Demo Cleanup
```bash
# Clean up demo files
rm -f /tmp/basic-config.ini /tmp/user-list.txt /tmp/multi-env-config.ini /tmp/nginx-config.conf /tmp/my-config.txt
```

---

**Note for Instructor**: This demo focuses on practical, immediately applicable Jinja2 concepts. Adjust depth based on participant questions and experience levels. Encourage hands-on practice during the interactive session.

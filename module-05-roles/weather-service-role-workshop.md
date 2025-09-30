# Weather Service Role Workshop - Ansible Best Practices

## Overview
This workshop demonstrates Ansible best practices through creating a reusable weather service role. We'll focus on gather_facts usage, role structure, and practical Ansible patterns that Gabriela requested.

## Duration: 90 minutes

## Learning Objectives
- Create structured Ansible roles
- Use gather_facts effectively for system information
- Implement environment-specific configurations
- Apply Ansible best practices and particularities
- Build reusable, maintainable automation

## Workshop Structure

### Part 1: Simple Template (15 minutes)
Start with a basic template to understand concepts

### Part 2: Role Creation (45 minutes) 
Build a proper Ansible role structure

### Part 3: Advanced Features (30 minutes)
Add gather_facts usage and best practices

---

## Part 1: Setup and Galaxy Role Creation

### Create Basic Project Structure
```bash
mkdir weather-service-workshop
cd weather-service-workshop
```

### Create Inventory for Role Testing
Create `hosts.yml`:
```yaml
all:
  hosts:
    host1:
      ansible_host: 13.51.178.222
      ansible_user: ubuntu
    host2:
      ansible_host: 16.16.202.25
      ansible_user: ubuntu
  vars:
    ansible_ssh_pass: "LetMe1n!"
    ansible_python_interpreter: /usr/bin/python3
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
  children:
    development:
      hosts:
        host1:
      vars:
        weather_environment: development
    
    production:
      hosts:
        host2:
      vars:
        weather_environment: production
```

## Part 2: Create Weather Service Role

### Step 1: Initialize Role Structure with Ansible Galaxy

First, let's create the role structure using Ansible Galaxy:

```bash
# Create role using ansible-galaxy command
ansible-galaxy init roles/weather-service

# This creates the complete role structure:
# roles/weather-service/
# ├── README.md
# ├── defaults/
# │   └── main.yml
# ├── files/
# ├── handlers/
# │   └── main.yml
# ├── meta/
# │   └── main.yml
# ├── tasks/
# │   └── main.yml
# ├── templates/
# ├── tests/
# │   ├── inventory
# │   └── test.yml
# └── vars/
#     └── main.yml
```

### Step 2: Verify Role Structure
```bash
# Check the created structure
tree roles/weather-service/

# Or use ls if tree is not available
find roles/weather-service/ -type f

# Check what Galaxy created for us
ls -la roles/weather-service/
ls -la roles/weather-service/*/
```

### Step 3: Understanding Ansible Galaxy

**Ansible Galaxy** is the official hub for sharing Ansible roles and collections:

- **Role Creation**: `ansible-galaxy init` creates standardized role structure
- **Role Installation**: Download and install community roles
- **Role Management**: Handle dependencies and versions
- **Best Practices**: Enforces consistent role organization

### Step 4: Create Requirements File (Optional)

Create `requirements.yml` for external role dependencies:

```yaml
---
# External roles from Ansible Galaxy
roles:
  - name: geerlingguy.nginx
    version: 3.1.4
  - name: geerlingguy.security
    version: 2.0.1

# Collections from Ansible Galaxy  
collections:
  - name: community.general
    version: ">=3.0.0"
  - name: ansible.posix
    version: ">=1.0.0"
```

### Step 5: Install External Dependencies
```bash
# Install roles and collections from requirements.yml
ansible-galaxy install -r requirements.yml

# Install specific role directly
ansible-galaxy install geerlingguy.nginx

# List installed roles
ansible-galaxy list

# Show role info
ansible-galaxy info geerlingguy.nginx
```

**Why Initialize First?**
- Galaxy creates the proper directory structure and skeleton files
- Ensures we follow Ansible best practices from the start
- Provides template files with proper YAML structure
- Sets up testing framework and documentation templates
- Makes our role compatible with Galaxy ecosystem

Now that we have the role structure created by Galaxy, let's populate it with our weather service code:

**Variable Organization Best Practice:**
- **`defaults/main.yml`**: Default values that can be overridden by users
- **`vars/main.yml`**: Internal role variables with higher precedence
- **Inventory**: Only environment-specific identifiers (not data)
- **Templates**: Use role variables for consistent data access

### Role Defaults
Edit `roles/weather-service/defaults/main.yml`:
```yaml
---
# Weather Service Role Defaults

# Service configuration
weather_service_port: 80
weather_service_user: www-data
weather_service_group: www-data

# Default weather data for development
development_weather:
  city_name: "Busteni"
  temperature: 18
  condition: "Mountain Fresh"
  humidity: 75
  wind_speed: 2.5
  description: "Cool mountain air with light breeze"

# Default weather data for production  
production_weather:
  city_name: "Bucharest"
  temperature: 25
  condition: "Sunny"
  humidity: 60
  wind_speed: 3.2
  description: "Clear skies with gentle breeze"

# API configuration (production)
weather_api_key: "c0ac323c4eed4a889f272444253009"
weather_api_url: "https://api.weatherapi.com/v1/current.json"

# Nginx configuration
nginx_worker_processes: auto
nginx_worker_connections: 1024

# Gather facts settings
weather_gather_system_info: true
weather_gather_network_info: true
```

### Role Variables
Edit `roles/weather-service/vars/main.yml`:
```yaml
---
# Weather Service Role Variables

# Environment-specific configurations
weather_environments:
  development:
    debug_mode: true
    log_level: debug
    cache_timeout: 60
    use_api: false
  production:
    debug_mode: false
    log_level: info
    cache_timeout: 3600
    use_api: true

# System requirements
required_packages:
  - nginx
  - curl

# Service paths
weather_service_paths:
  web_root: /var/www/html
  config_dir: /etc/nginx/sites-available
  log_dir: /var/log/weather-service

# Current environment weather data (set dynamically in tasks)
current_weather_data: {}
```

### Role Tasks
Edit `roles/weather-service/tasks/main.yml`:
```yaml
---
# Weather Service Role Tasks

- name: Gather system facts for weather service
  setup:
    gather_subset:
      - hardware
      - network
      - virtual
  when: weather_gather_system_info | bool
  tags: facts

- name: Display gathered system information
  debug:
    msg: |
      System Information Gathered:
      - Hostname: {{ ansible_hostname }}
      - Architecture: {{ ansible_architecture }}
      - Processor Count: {{ ansible_processor_count }}
      - Memory (MB): {{ ansible_memtotal_mb }}
      - Distribution: {{ ansible_distribution }} {{ ansible_distribution_version }}
      - Virtualization: {{ ansible_virtualization_type | default('physical') }}
  when: weather_gather_system_info | bool
  tags: facts

- name: Set weather data based on environment
  set_fact:
    current_weather_data: "{{ development_weather if weather_environment == 'development' else production_weather }}"
  tags: config

- name: Fetch live weather data for production
  block:
    - name: Call WeatherAPI.com for live data
      uri:
        url: "{{ weather_api_url }}?key={{ weather_api_key }}&q={{ current_weather_data.city_name }}&aqi=no"
        method: GET
        return_content: yes
      register: weather_api_response
      
    - name: Parse WeatherAPI.com response
      set_fact:
        current_weather_data:
          city_name: "{{ weather_api_response.json.location.name }}"
          temperature: "{{ weather_api_response.json.current.temp_c | round(1) }}"
          condition: "{{ weather_api_response.json.current.condition.text }}"
          humidity: "{{ weather_api_response.json.current.humidity }}"
          wind_speed: "{{ weather_api_response.json.current.wind_kph | float / 3.6 | round(1) }}"
          description: "{{ weather_api_response.json.current.condition.text }}"
          
  rescue:
    - name: API call failed - using default production data
      debug:
        msg: "WeatherAPI.com call failed. Using default production weather data."
        
    - name: Set fallback weather data for production
      set_fact:
        current_weather_data:
          city_name: "{{ production_weather.city_name }}"
          temperature: "{{ production_weather.temperature }}"
          condition: "{{ production_weather.condition }}"
          humidity: "{{ production_weather.humidity }}"
          wind_speed: "{{ production_weather.wind_speed }}"
          description: "{{ production_weather.description }} (API fallback)"
          
  when: 
    - weather_environment == 'production'
    - weather_environments[weather_environment].use_api | bool
  tags: config

- name: Create weather service directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ weather_service_user }}"
    group: "{{ weather_service_group }}"
    mode: '0755'
  loop:
    - "{{ weather_service_paths.web_root }}"
    - "{{ weather_service_paths.log_dir }}"
  tags: setup

- name: Install required packages
  package:
    name: "{{ required_packages }}"
    state: present
  tags: packages

- name: Configure nginx for weather service
  template:
    src: nginx.conf.j2
    dest: "{{ weather_service_paths.config_dir }}/weather-service"
    owner: root
    group: root
    mode: '0644'
  notify: restart nginx
  tags: config

- name: Deploy weather application
  template:
    src: weather-app.html.j2
    dest: "{{ weather_service_paths.web_root }}/index.html"
    owner: "{{ weather_service_user }}"
    group: "{{ weather_service_group }}"
    mode: '0644'
  notify: reload nginx
  tags: deploy

- name: Enable weather service site
  file:
    src: "{{ weather_service_paths.config_dir }}/weather-service"
    dest: /etc/nginx/sites-enabled/weather-service
    state: link
  notify: restart nginx
  tags: config

- name: Disable default nginx site
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: restart nginx
  tags: config

- name: Start and enable nginx
  service:
    name: nginx
    state: started
    enabled: yes
  tags: service

- name: Display deployment summary
  debug:
    msg: |
      ================================
      Weather Service Deployment
      ================================
      Environment: {{ weather_environment }}
      City: {{ current_weather_data.city_name }}
      Temperature: {{ current_weather_data.temperature }}°C
      Condition: {{ current_weather_data.condition }}
      Server: {{ ansible_hostname }} ({{ ansible_default_ipv4.address }})
      Architecture: {{ ansible_architecture }}
      Memory: {{ ansible_memtotal_mb }}MB
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      Service URL: http://{{ ansible_default_ipv4.address }}:{{ weather_service_port }}
      Status: Deployed Successfully
      Role: weather-service (Galaxy-initialized)
      Deployed At: {{ lookup('pipe', 'date "+%Y-%m-%d %H:%M:%S %Z"') }}
      ================================
  tags: summary
```

### Enhanced Weather Template
Create `roles/weather-service/templates/weather-app.html.j2`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Weather Service - {{ current_weather_data.city_name }}</title>
</head>
<body>
    <h1>Weather Service</h1>
    <h2>{{ current_weather_data.city_name }} - {{ weather_environment | upper }}</h2>
    
    <!-- Weather Information -->
    <h3>Current Weather</h3>
    <p><strong>Temperature:</strong> {{ current_weather_data.temperature }}°C</p>
    <p><strong>Condition:</strong> {{ current_weather_data.condition }}</p>
    <p><strong>Humidity:</strong> {{ current_weather_data.humidity }}%</p>
    <p><strong>Wind Speed:</strong> {{ current_weather_data.wind_speed }} m/s</p>
    <p><strong>Description:</strong> {{ current_weather_data.description }}</p>
    {% if weather_environment == 'development' %}
    <p><small>[MOCK DATA for development environment]</small></p>
    {% else %}
      {% if 'API fallback' in current_weather_data.description %}
    <p><small>[MOCK DATA - WeatherAPI.com failed, switched to fallback]</small></p>
      {% else %}
    <p><small>[LIVE DATA from WeatherAPI.com]</small></p>
      {% endif %}
    {% endif %}
    
    <!-- System Information (Gathered Facts) -->
    <h3>Server Information</h3>
    <table border="1" cellpadding="5">
        <tr><td><strong>Hostname</strong></td><td>{{ ansible_hostname }}</td></tr>
        <tr><td><strong>IP Address</strong></td><td>{{ ansible_default_ipv4.address }}</td></tr>
        <tr><td><strong>Architecture</strong></td><td>{{ ansible_architecture }}</td></tr>
        <tr><td><strong>CPU Cores</strong></td><td>{{ ansible_processor_count }}</td></tr>
        <tr><td><strong>Memory (MB)</strong></td><td>{{ ansible_memtotal_mb }}</td></tr>
        <tr><td><strong>OS Distribution</strong></td><td>{{ ansible_distribution }} {{ ansible_distribution_version }}</td></tr>
        <tr><td><strong>Kernel</strong></td><td>{{ ansible_kernel }}</td></tr>
        <tr><td><strong>Virtualization</strong></td><td>{{ ansible_virtualization_type | default('Physical') }}</td></tr>
    </table>
    
    <!-- Environment Configuration -->
    <h3>Environment Configuration</h3>
    <p><strong>Environment:</strong> {{ weather_environment }}</p>
    <p><strong>Debug Mode:</strong> {{ weather_environments[weather_environment].debug_mode }}</p>
    <p><strong>Log Level:</strong> {{ weather_environments[weather_environment].log_level }}</p>
    <p><strong>Cache Timeout:</strong> {{ weather_environments[weather_environment].cache_timeout }}s</p>
    <p><strong>API Usage:</strong> {{ weather_environments[weather_environment].use_api }}</p>
    
    <!-- Network Information -->
    {% if ansible_all_ipv4_addresses is defined %}
    <h3>Network Interfaces</h3>
    <ul>
    {% for ip in ansible_all_ipv4_addresses %}
        <li>{{ ip }}</li>
    {% endfor %}
    </ul>
    {% endif %}
    
    <hr>
    <p><em>Generated: {{ ansible_date_time.iso8601 }} by Ansible Galaxy Role</em></p>
    <p><em>Weather API: WeatherAPI.com</em></p>
    <p><em>Deployed on: {{ ansible_hostname }} ({{ ansible_architecture }})</em></p>
</body>
</html>
```

### Nginx Configuration Template
Create `roles/weather-service/templates/nginx.conf.j2`:
```nginx
server {
    listen {{ weather_service_port }};
    server_name {{ ansible_hostname }} {{ ansible_default_ipv4.address }};
    
    root {{ weather_service_paths.web_root }};
    index index.html;
    
    # Logging
    access_log {{ weather_service_paths.log_dir }}/access.log;
    error_log {{ weather_service_paths.log_dir }}/error.log;
    
    # Weather service location
    location / {
        try_files $uri $uri/ =404;
        
        # Add environment header
        add_header X-Environment {{ weather_environment }};
        add_header X-Server {{ ansible_hostname }};
        add_header X-City {{ current_weather_data.city_name }};
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "OK - {{ environment }} - {{ ansible_hostname }}";
        add_header Content-Type text/plain;
    }
}
```

### Role Handlers
Create `roles/weather-service/handlers/main.yml`:
```yaml
---
# Weather Service Role Handlers

- name: restart nginx
  service:
    name: nginx
    state: restarted
  
- name: reload nginx
  service:
    name: nginx
    state: reloaded
```

### Role Metadata for Galaxy
Create `roles/weather-service/meta/main.yml`:
```yaml
---
galaxy_info:
  # Role identification
  role_name: weather_service
  namespace: bittnet_training
  author: Bittnet Training Team
  description: Weather Service Role with API integration and error handling
  company: Bittnet Training
  
  # Licensing and version requirements
  license: MIT
  min_ansible_version: "2.9"
  
  # Supported platforms (Galaxy requirement)
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
        - bionic
    - name: Debian
      versions:
        - buster
        - bullseye
        - bookworm
    - name: EL  # Enterprise Linux (RHEL, CentOS, etc.)
      versions:
        - "7"
        - "8"
        - "9"

  # Galaxy search tags
  galaxy_tags:
    - web
    - nginx
    - weather
    - api
    - service
    - monitoring
    - development
    - production

  # GitHub repository (for Galaxy publishing)
  github_branch: main
  
# Role dependencies (other Galaxy roles)
dependencies:
  # Example: depend on nginx role
  # - role: geerlingguy.nginx
  #   version: ">=3.0.0"
```

### Publishing Role to Galaxy (Optional)

If you want to share your role on Ansible Galaxy:

```bash
# 1. Create GitHub repository for your role
git init
git add .
git commit -m "Initial weather service role"
git remote add origin https://github.com/YOUR_USERNAME/ansible-role-weather-service.git
git push -u origin main

# 2. Login to Ansible Galaxy
ansible-galaxy login

# 3. Import role from GitHub (via Galaxy web interface)
# Visit: https://galaxy.ansible.com/my-content/namespaces
# Connect your GitHub account and import the repository

# 4. Install your published role
ansible-galaxy install YOUR_USERNAME.weather_service
```

### Galaxy Role Best Practices

1. **Naming Convention**: Use `ansible-role-` prefix for GitHub repos
2. **Documentation**: Include comprehensive README.md
3. **Testing**: Add molecule tests in `molecule/` directory
4. **Versioning**: Use semantic versioning (1.0.0, 1.1.0, etc.)
5. **Dependencies**: Clearly define role dependencies
6. **Platform Support**: Test on multiple OS versions

### Role-based Playbook
Create `deploy-role.yml`:
```yaml
---
- name: Deploy Weather Service using Role
  hosts: all
  become: yes
  
  roles:
    - weather-service
  
  post_tasks:
    - name: Wait for nginx to be ready
      wait_for:
        port: 80
        host: "{{ ansible_default_ipv4.address }}"
        delay: 2
        timeout: 30
      
    - name: Test weather service endpoint
      uri:
        url: "http://{{ ansible_default_ipv4.address }}/health"
        method: GET
        status_code: [200, 404]  # Accept both until site is fully configured
      register: health_check
      retries: 3
      delay: 2
      
    - name: Display health check result
      debug:
        msg: |
          Health Check Results:
          - URL: http://{{ ansible_host }}/health
          - Status: {{ health_check.status }}
          - Response: {{ health_check.content | default('No response body') }}
          {% if health_check.status == 200 %}
          - Result: SUCCESS - Health endpoint is working
          {% else %}
          - Result: WARNING - Health endpoint returned {{ health_check.status }}
          {% endif %}
```

---

## Part 3: Advanced Usage and Best Practices

### Custom Facts for Weather Service
Create `roles/weather-service/tasks/custom_facts.yml`:
```yaml
---
# Custom facts for weather service

- name: Create custom facts directory
  file:
    path: /etc/ansible/facts.d
    state: directory
    mode: '0755'

- name: Deploy weather service custom facts
  template:
    src: weather_facts.fact.j2
    dest: /etc/ansible/facts.d/weather_service.fact
    mode: '0755'
  notify: refresh facts

- name: Refresh ansible facts
  setup:
  tags: facts
```

Create `roles/weather-service/templates/weather_facts.fact.j2`:
```bash
#!/bin/bash
# Custom facts for weather service

cat << EOF
{
  "environment": "{{ environment }}",
  "city": "{{ city_name }}",
  "deployment_time": "$(date -Iseconds)",
  "service_version": "1.0.0",
  "nginx_status": "$(systemctl is-active nginx)"
}
EOF
```

### Include Custom Facts in Main Tasks
Add to `roles/weather-service/tasks/main.yml`:
```yaml
- name: Setup custom facts
  include_tasks: custom_facts.yml
  tags: facts
```

### Testing and Validation
Create `test-role.yml`:
```yaml
---
- name: Test Weather Service Role
  hosts: all
  become: yes
  
  pre_tasks:
    - name: Ensure system is updated
      package:
        update_cache: yes
      when: ansible_os_family == "Debian"
  
  roles:
    - weather-service
  
  post_tasks:
    - name: Validate service is running
      service:
        name: nginx
        state: started
      check_mode: yes
      register: nginx_status
      
    - name: Test weather service response
      uri:
        url: "http://{{ ansible_default_ipv4.address }}"
        return_content: yes
      register: weather_response
      
    - name: Verify weather service content
      assert:
        that:
          - weather_response.status == 200
          - city_name in weather_response.content
          - environment in weather_response.content
        fail_msg: "Weather service validation failed"
        success_msg: "Weather service validation passed"
```

---

## Part 3: Simple Template Comparison (Optional)

For comparison, here's how the same functionality would look without using a role:

### Simple Template (Old Approach)
Create `templates/simple-weather.html.j2`:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Weather Service - {{ city_name }}</title>
</head>
<body>
    <h1>Weather Service</h1>
    <h2>{{ city_name }} ({{ environment }})</h2>
    
    <p><strong>Temperature:</strong> {{ weather_temp }}°C</p>
    <p><strong>Condition:</strong> {{ weather_condition }}</p>
    
    <hr>
    <h3>Server Information</h3>
    <p><strong>Hostname:</strong> {{ ansible_hostname }}</p>
    <p><strong>IP:</strong> {{ ansible_default_ipv4.address }}</p>
    <p><strong>OS:</strong> {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
    
    <hr>
    <p><em>Generated: {{ ansible_date_time.iso8601 }}</em></p>
</body>
</html>
```

### Simple Playbook (Old Approach)
Create `deploy-simple.yml`:
```yaml
---
- name: Deploy Simple Weather Service
  hosts: all
  become: yes
  vars:
    # Variables hardcoded in playbook - not reusable!
    city_name: "{{ 'Busteni' if weather_environment == 'development' else 'Bucharest' }}"
    weather_temp: "{{ 18 if weather_environment == 'development' else 25 }}"
    weather_condition: "{{ 'Mountain Fresh' if weather_environment == 'development' else 'Sunny' }}"
    environment: "{{ weather_environment }}"
  
  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present
        
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
        
    - name: Deploy weather page
      template:
        src: simple-weather.html.j2
        dest: /var/www/html/index.html
        mode: '0644'
      
    - name: Show deployment info
      debug:
        msg: |
          Deployed to: {{ inventory_hostname }}
          Environment: {{ environment }}
          City: {{ city_name }}
          URL: http://{{ ansible_host }}
```

### Problems with Simple Approach:
- **Not reusable**: Variables hardcoded in playbook
- **Not maintainable**: Logic scattered across files  
- **Not testable**: No standardized structure
- **Not shareable**: Cannot publish to Galaxy
- **Not scalable**: Difficult to extend

### Benefits of Role Approach:
- **Reusable**: Can be used in any project
- **Maintainable**: Clear structure and organization
- **Testable**: Standardized testing framework
- **Shareable**: Galaxy-compatible
- **Scalable**: Easy to extend and modify

---

## Workshop Exercises

### Exercise 1: Initialize Role with Galaxy (15 minutes)
```bash
# Create role structure with Galaxy
ansible-galaxy init roles/weather-service

# Explore the created structure
tree roles/weather-service/
cat roles/weather-service/defaults/main.yml
cat roles/weather-service/meta/main.yml

# Practice Galaxy commands
ansible-galaxy search nginx --max 5
ansible-galaxy info geerlingguy.nginx
```

### Exercise 2: Deploy Role Version (30 minutes)
```bash
# After populating role files, deploy using the role
ansible-playbook -i hosts.yml deploy-role.yml

# Test health endpoints
curl http://13.51.178.222/health
curl http://16.16.202.25/health

# Check detailed weather pages
curl http://13.51.178.222/
curl http://16.16.202.25/
```

### Exercise 3: Test and Validate (15 minutes)
```bash
# Run validation tests
ansible-playbook -i hosts.yml test-role.yml

# Check custom facts
ansible -i hosts.yml all -m setup -a "filter=ansible_local"
```

### Exercise 4: Explore Gathered Facts (15 minutes)
```bash
# Gather specific facts
ansible -i hosts.yml all -m setup -a "gather_subset=hardware,network"

# Check system information
ansible -i hosts.yml all -m setup -a "filter=ansible_processor*"
ansible -i hosts.yml all -m setup -a "filter=ansible_memory*"
```

### Exercise 5: Compare Approaches (Optional - 15 minutes)
```bash
# Initialize a new role with Galaxy
ansible-galaxy init roles/test-role

# Check role structure
tree roles/test-role/

# Search for roles on Galaxy
ansible-galaxy search nginx

# Get info about a specific role
ansible-galaxy info geerlingguy.nginx

# Install a role from Galaxy
ansible-galaxy install geerlingguy.nginx

# List installed roles
ansible-galaxy list

# Remove installed role
ansible-galaxy remove geerlingguy.nginx

# Create requirements file and install
cat > requirements.yml << EOF
---
roles:
  - name: geerlingguy.security
    version: 2.0.1
EOF

ansible-galaxy install -r requirements.yml
```

## WeatherAPI.com Integration

This role demonstrates integration with [WeatherAPI.com](https://www.weatherapi.com/), a modern weather API service:

### API Features Used:
- **Current Weather**: Real-time weather data for any location
- **JSON Response**: Lightweight, fast API responses (~200ms average)
- **Free Tier**: 1 million calls/month for development and testing
- **Global Coverage**: Weather data for millions of locations worldwide

### API Call Structure:
```bash
# WeatherAPI.com endpoint - parameters as query string
GET https://api.weatherapi.com/v1/current.json?key=YOUR_KEY&q=CITY&aqi=no

# Response structure (simplified)
{
  "location": {
    "name": "Bucharest",
    "country": "Romania"
  },
  "current": {
    "temp_c": 25.0,
    "condition": {
      "text": "Sunny"
    },
    "humidity": 60,
    "wind_kph": 11.5
  }
}
```

### Error Handling:
- **Block/Rescue**: Graceful fallback to mock data if API fails
- **Conditional Execution**: API calls only in production environment
- **Clear Indicators**: Template shows data source (Live API vs Mock fallback)

---

## Key Learning Points for Gabriela

1. **Ansible Galaxy Integration:**
   - Use `ansible-galaxy init` to create standardized role structure
   - Install community roles with `ansible-galaxy install`
   - Manage dependencies with `requirements.yml`
   - Search and explore roles on Galaxy hub

2. **Gather Facts Best Practices:**
   - Use `gather_subset` to collect only needed facts
   - Custom facts for application-specific data
   - Facts filtering for performance optimization
   - Hardware and network facts for system information

3. **Lookup Plugins Usage:**
   - `lookup('pipe', 'date "+%Y-%m-%d %H:%M:%S %Z"')` for current timestamp
   - Execute system commands from within Ansible templates/tasks
   - Integrate external utilities and scripts into playbooks
   - Useful for audit trails, deployment tracking, and dynamic data

4. **Professional Role Structure:**
   - Galaxy-compliant directory organization
   - Proper separation of defaults vs vars
   - Handler usage for service management
   - Comprehensive metadata for sharing

4. **Ansible Particularities:**
   - Variable precedence (defaults < vars < host_vars < extra_vars)
   - Template inheritance and reusability
   - Conditional task execution with `when`
   - Tag usage for selective execution
   - Block/rescue error handling patterns

5. **Production Best Practices:**
   - Role dependencies and version management
   - Cross-platform compatibility
   - Comprehensive testing strategies
   - Documentation and metadata standards

6. **Galaxy Ecosystem:**
   - Community role discovery and evaluation
   - Role publishing and sharing workflows
   - Version management and updates
   - Integration with CI/CD pipelines

This workshop provides a comprehensive foundation for building production-ready Ansible roles using Galaxy best practices, while demonstrating the gather_facts capabilities and advanced Ansible patterns that Gabriela requested for enterprise automation.

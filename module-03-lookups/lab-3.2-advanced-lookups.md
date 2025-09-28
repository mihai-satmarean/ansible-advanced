# Lab 3.2: Advanced Data Lookups

## Objective
Implement complex data retrieval using csvfile, dig, url, pipe, and other advanced lookup plugins for enterprise automation scenarios.

## Duration
30 minutes

## Prerequisites
- Completed Lab 3.1
- Understanding of external data formats and network concepts

## Lab Setup

```bash
cd ~/ansible-labs/module-03
mkdir -p lab-3.2
cd lab-3.2

cat > inventory.ini << EOF
[webservers]
web01 ansible_host=localhost ansible_connection=local
web02 ansible_host=localhost ansible_connection=local

[databases]
db01 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF

# Create test data files
mkdir -p data

# Create CSV data file
cat > data/servers.csv << EOF
hostname,ip_address,role,environment,cpu_cores,memory_gb,disk_gb
web01,192.168.1.10,webserver,production,4,8,100
web02,192.168.1.11,webserver,production,4,8,100
web03,192.168.1.12,webserver,staging,2,4,50
db01,192.168.1.20,database,production,8,32,500
db02,192.168.1.21,database,staging,4,16,250
cache01,192.168.1.30,cache,production,2,8,50
lb01,192.168.1.40,loadbalancer,production,2,4,50
EOF

# Create user data CSV
cat > data/users.csv << EOF
username,email,department,role,access_level
john.doe,john.doe@company.com,engineering,developer,read-write
jane.smith,jane.smith@company.com,engineering,senior-developer,admin
bob.wilson,bob.wilson@company.com,operations,sysadmin,admin
alice.brown,alice.brown@company.com,security,analyst,read-only
charlie.davis,charlie.davis@company.com,management,manager,read-only
EOF

# Create application configuration CSV
cat > data/applications.csv << EOF
app_name,version,port,health_endpoint,dependencies,environment
web-frontend,2.1.0,3000,/health,"api-backend,cache",production
api-backend,1.5.2,8080,/api/health,"database,cache",production
user-service,1.2.1,8081,/users/health,database,production
notification-service,1.0.5,8082,/notify/health,"user-service,queue",production
admin-panel,1.8.0,9090,/admin/health,"api-backend,database",production
EOF
```

## Exercise 1: CSV File Lookups (12 minutes)

### Task: Use csvfile lookups for dynamic server and user management

Create `csvfile-lookups.yml`:
```yaml
---
- name: CSV file lookup demonstrations
  hosts: all
  vars:
    csv_files:
      servers: "data/servers.csv"
      users: "data/users.csv"
      applications: "data/applications.csv"
      
  tasks:
    # Basic CSV lookups
    - name: Look up server information from CSV
      set_fact:
        server_info: "{{ lookup('csvfile', inventory_hostname + ' file=' + csv_files.servers + ' delimiter=, col=1') }}"
      failed_when: false
    
    - name: Display server information
      debug:
        msg: |
          Server Information for {{ inventory_hostname }}:
          - IP Address: {{ server_info | default('Not found') }}
      when: server_info is defined and server_info != ""
    
    # Advanced CSV lookups with multiple columns
    - name: Get comprehensive server details
      set_fact:
        server_details: |
          {
            "ip_address": "{{ lookup('csvfile', inventory_hostname + ' file=' + csv_files.servers + ' delimiter=, col=1', errors='ignore') }}",
            "role": "{{ lookup('csvfile', inventory_hostname + ' file=' + csv_files.servers + ' delimiter=, col=2', errors='ignore') }}",
            "environment": "{{ lookup('csvfile', inventory_hostname + ' file=' + csv_files.servers + ' delimiter=, col=3', errors='ignore') }}",
            "cpu_cores": "{{ lookup('csvfile', inventory_hostname + ' file=' + csv_files.servers + ' delimiter=, col=4', errors='ignore') }}",
            "memory_gb": "{{ lookup('csvfile', inventory_hostname + ' file=' + csv_files.servers + ' delimiter=, col=5', errors='ignore') }}",
            "disk_gb": "{{ lookup('csvfile', inventory_hostname + ' file=' + csv_files.servers + ' delimiter=, col=6', errors='ignore') }}"
          }
    
    - name: Parse server details
      set_fact:
        parsed_server_details: "{{ server_details | from_json }}"
      when: server_details is defined
    
    - name: Display comprehensive server details
      debug:
        msg: |
          Comprehensive Server Details for {{ inventory_hostname }}:
          {{ parsed_server_details | to_nice_yaml }}
      when: 
        - parsed_server_details is defined
        - parsed_server_details.ip_address != ""
    
    # Create server-specific configuration based on CSV data
    - name: Generate server configuration from CSV data
      copy:
        content: |
          # Server Configuration for {{ inventory_hostname }}
          # Generated from CSV data
          
          [server]
          hostname={{ inventory_hostname }}
          ip_address={{ parsed_server_details.ip_address }}
          role={{ parsed_server_details.role }}
          environment={{ parsed_server_details.environment }}
          
          [resources]
          cpu_cores={{ parsed_server_details.cpu_cores }}
          memory_gb={{ parsed_server_details.memory_gb }}
          disk_gb={{ parsed_server_details.disk_gb }}
          
          [metadata]
          generated_at={{ ansible_date_time.iso8601 }}
          source=csv_lookup
        dest: "/tmp/{{ inventory_hostname }}-server-config.conf"
        mode: '0644'
      when: 
        - parsed_server_details is defined
        - parsed_server_details.ip_address != ""
    
    # User management based on CSV data
    - name: Look up users by department
      set_fact:
        engineering_users: |
          {% set users = [] %}
          {% for user in ['john.doe', 'jane.smith', 'bob.wilson', 'alice.brown', 'charlie.davis'] %}
          {% set department = lookup('csvfile', user + ' file=' + csv_files.users + ' delimiter=, col=2', errors='ignore') %}
          {% set role = lookup('csvfile', user + ' file=' + csv_files.users + ' delimiter=, col=3', errors='ignore') %}
          {% set access_level = lookup('csvfile', user + ' file=' + csv_files.users + ' delimiter=, col=4', errors='ignore') %}
          {% if department == 'engineering' %}
          {% set _ = users.append({'username': user, 'role': role, 'access_level': access_level}) %}
          {% endif %}
          {% endfor %}
          {{ users }}
      run_once: true
    
    - name: Display engineering users
      debug:
        msg: |
          Engineering Department Users:
          {{ engineering_users | to_nice_yaml }}
      run_once: true
    
    # Application deployment based on CSV data
    - name: Look up applications for this server role
      set_fact:
        role_applications: |
          {% set apps = [] %}
          {% for app in ['web-frontend', 'api-backend', 'user-service', 'notification-service', 'admin-panel'] %}
          {% set app_port = lookup('csvfile', app + ' file=' + csv_files.applications + ' delimiter=, col=2', errors='ignore') %}
          {% set health_endpoint = lookup('csvfile', app + ' file=' + csv_files.applications + ' delimiter=, col=3', errors='ignore') %}
          {% set dependencies = lookup('csvfile', app + ' file=' + csv_files.applications + ' delimiter=, col=4', errors='ignore') %}
          {% if app_port != "" %}
          {% set _ = apps.append({'name': app, 'port': app_port, 'health_endpoint': health_endpoint, 'dependencies': dependencies}) %}
          {% endif %}
          {% endfor %}
          {{ apps }}
      when: 
        - parsed_server_details is defined
        - parsed_server_details.role in ['webserver', 'database']
    
    - name: Generate application deployment configuration
      copy:
        content: |
          # Application Deployment Configuration
          # Server: {{ inventory_hostname }} ({{ parsed_server_details.role }})
          
          {% for app in role_applications %}
          [{{ app.name }}]
          port={{ app.port }}
          health_endpoint={{ app.health_endpoint }}
          dependencies={{ app.dependencies }}
          
          {% endfor %}
          
          [deployment]
          server_role={{ parsed_server_details.role }}
          total_applications={{ role_applications | length }}
          generated_at={{ ansible_date_time.iso8601 }}
        dest: "/tmp/{{ inventory_hostname }}-applications.conf"
        mode: '0644'
      when: 
        - role_applications is defined
        - role_applications | length > 0
```

### Run CSV file lookups:
```bash
# Run CSV file lookups
ansible-playbook -i inventory.ini csvfile-lookups.yml

# Check generated configurations
ls -la /tmp/*-server-config.conf
ls -la /tmp/*-applications.conf
cat /tmp/web01-server-config.conf
cat /tmp/web01-applications.conf
```

## Exercise 2: Network and URL Lookups (10 minutes)

### Task: Use dig and url lookups for network information and API integration

Create `network-url-lookups.yml`:
```yaml
---
- name: Network and URL lookup demonstrations
  hosts: localhost
  vars:
    # Test domains and services
    test_domains:
      - "google.com"
      - "github.com"
      - "stackoverflow.com"
      
    # API endpoints for testing
    api_endpoints:
      - url: "https://httpbin.org/json"
        description: "JSON response test"
      - url: "https://api.github.com/users/octocat"
        description: "GitHub user API"
      - url: "https://httpbin.org/status/200"
        description: "HTTP status test"
        
  tasks:
    # DNS lookups using dig
    - name: Perform DNS lookups for test domains
      set_fact:
        dns_results: |
          {% set results = {} %}
          {% for domain in test_domains %}
          {% set ip_address = lookup('dig', domain, errors='ignore') %}
          {% if ip_address %}
          {% set _ = results.update({domain: ip_address}) %}
          {% endif %}
          {% endfor %}
          {{ results }}
    
    - name: Display DNS lookup results
      debug:
        msg: |
          DNS Lookup Results:
          {% for domain, ip in dns_results.items() %}
          - {{ domain }}: {{ ip }}
          {% endfor %}
    
    # Advanced DNS lookups
    - name: Perform different types of DNS lookups
      debug:
        msg: |
          Advanced DNS Lookups for {{ item }}:
          - A Record: {{ lookup('dig', item, 'qtype=A', errors='ignore') | default('Not found') }}
          - MX Record: {{ lookup('dig', item, 'qtype=MX', errors='ignore') | default('Not found') }}
          - TXT Record: {{ lookup('dig', item, 'qtype=TXT', errors='ignore') | default('Not found') }}
      loop: "{{ test_domains[:2] }}"  # Limit to first 2 domains to avoid too many requests
    
    # URL lookups for API integration
    - name: Fetch data from API endpoints
      set_fact:
        api_responses: |
          {% set responses = {} %}
          {% for endpoint in api_endpoints %}
          {% set response = lookup('url', endpoint.url, headers={'User-Agent': 'Ansible-Lookup'}, errors='ignore') %}
          {% if response %}
          {% set _ = responses.update({endpoint.description: response}) %}
          {% endif %}
          {% endfor %}
          {{ responses }}
    
    - name: Display API responses
      debug:
        msg: |
          API Response for {{ item.key }}:
          {{ item.value[:200] }}...
      loop: "{{ api_responses | dict2items }}"
      when: api_responses | length > 0
    
    # Parse JSON responses from APIs
    - name: Parse JSON API responses
      set_fact:
        parsed_api_data: |
          {% set parsed = {} %}
          {% for desc, response in api_responses.items() %}
          {% if 'json' in desc.lower() or 'github' in desc.lower() %}
          {% set json_data = response | from_json %}
          {% set _ = parsed.update({desc: json_data}) %}
          {% endif %}
          {% endfor %}
          {{ parsed }}
      when: api_responses | length > 0
    
    - name: Display parsed API data
      debug:
        msg: |
          Parsed API Data:
          {{ parsed_api_data | to_nice_yaml }}
      when: parsed_api_data | length > 0
    
    # Create network configuration based on lookups
    - name: Generate network configuration from lookups
      copy:
        content: |
          # Network Configuration
          # Generated from DNS and API lookups
          
          [dns_servers]
          {% for domain, ip in dns_results.items() %}
          {{ domain }}_ip={{ ip }}
          {% endfor %}
          
          [external_services]
          {% for desc, data in parsed_api_data.items() %}
          {% if 'github' in desc.lower() %}
          github_user_id={{ data.id | default('unknown') }}
          github_user_login={{ data.login | default('unknown') }}
          {% endif %}
          {% endfor %}
          
          [monitoring]
          {% for endpoint in api_endpoints %}
          {{ endpoint.description | replace(' ', '_') | lower }}_url={{ endpoint.url }}
          {% endfor %}
          
          [metadata]
          generated_at={{ ansible_date_time.iso8601 }}
          lookup_types=dns,url,api
        dest: "/tmp/network-config.conf"
        mode: '0644'
    
    # Health check configuration using URL lookups
    - name: Create health check configuration
      copy:
        content: |
          # Health Check Configuration
          # Based on URL lookup results
          
          [health_checks]
          {% for endpoint in api_endpoints %}
          {% set status_check = lookup('url', endpoint.url, headers={'User-Agent': 'Ansible-Health-Check'}, errors='ignore') %}
          {{ endpoint.description | replace(' ', '_') | lower }}_status={{ 'UP' if status_check else 'DOWN' }}
          {{ endpoint.description | replace(' ', '_') | lower }}_url={{ endpoint.url }}
          {% endfor %}
          
          [dns_health]
          {% for domain in test_domains %}
          {% set dns_check = lookup('dig', domain, errors='ignore') %}
          {{ domain | replace('.', '_') }}_dns={{ 'RESOLVED' if dns_check else 'FAILED' }}
          {% endfor %}
          
          [summary]
          total_api_checks={{ api_endpoints | length }}
          total_dns_checks={{ test_domains | length }}
          check_timestamp={{ ansible_date_time.iso8601 }}
        dest: "/tmp/health-check-config.conf"
        mode: '0644'
```

### Run network and URL lookups:
```bash
# Run network and URL lookups
ansible-playbook -i inventory.ini network-url-lookups.yml

# Check generated configurations
cat /tmp/network-config.conf
cat /tmp/health-check-config.conf
```

## Exercise 3: Pipe and Command Lookups (8 minutes)

### Task: Use pipe lookups for system integration and command execution

Create `pipe-command-lookups.yml`:
```yaml
---
- name: Pipe and command lookup demonstrations
  hosts: localhost
  vars:
    # System commands for information gathering
    system_commands:
      - command: "uname -a"
        description: "System information"
      - command: "df -h /"
        description: "Disk usage"
      - command: "free -h"
        description: "Memory usage"
      - command: "nproc"
        description: "CPU cores"
        
    # Network commands
    network_commands:
      - command: "ip route show default"
        description: "Default route"
      - command: "hostname -I"
        description: "IP addresses"
        
  tasks:
    # Basic pipe lookups
    - name: Execute system commands using pipe lookup
      set_fact:
        system_info: |
          {% set info = {} %}
          {% for cmd in system_commands %}
          {% set result = lookup('pipe', cmd.command, errors='ignore') %}
          {% if result %}
          {% set _ = info.update({cmd.description: result}) %}
          {% endif %}
          {% endfor %}
          {{ info }}
    
    - name: Display system information
      debug:
        msg: |
          System Information:
          {% for desc, result in system_info.items() %}
          - {{ desc }}: {{ result }}
          {% endfor %}
    
    # Network information using pipe lookups
    - name: Gather network information
      set_fact:
        network_info: |
          {% set info = {} %}
          {% for cmd in network_commands %}
          {% set result = lookup('pipe', cmd.command, errors='ignore') %}
          {% if result %}
          {% set _ = info.update({cmd.description: result}) %}
          {% endif %}
          {% endfor %}
          {{ info }}
    
    - name: Display network information
      debug:
        msg: |
          Network Information:
          {% for desc, result in network_info.items() %}
          - {{ desc }}: {{ result }}
          {% endfor %}
    
    # Advanced pipe lookups with data processing
    - name: Process system data using pipe lookups
      set_fact:
        processed_system_data: |
          {
            "hostname": "{{ lookup('pipe', 'hostname', errors='ignore') }}",
            "uptime_seconds": "{{ lookup('pipe', 'cat /proc/uptime | cut -d\" \" -f1', errors='ignore') }}",
            "load_average": "{{ lookup('pipe', 'cat /proc/loadavg | cut -d\" \" -f1-3', errors='ignore') }}",
            "total_processes": "{{ lookup('pipe', 'ps aux | wc -l', errors='ignore') }}",
            "listening_ports": "{{ lookup('pipe', 'ss -tuln | grep LISTEN | wc -l', errors='ignore') }}",
            "disk_usage_percent": "{{ lookup('pipe', 'df / | tail -1 | awk \"{print $5}\" | sed \"s/%//\"', errors='ignore') }}"
          }
    
    - name: Parse processed system data
      set_fact:
        parsed_system_data: "{{ processed_system_data | from_json }}"
    
    - name: Display processed system data
      debug:
        msg: |
          Processed System Data:
          {{ parsed_system_data | to_nice_yaml }}
    
    # Generate system report using pipe lookups
    - name: Create comprehensive system report
      copy:
        content: |
          # System Report
          # Generated using pipe lookups
          
          [basic_info]
          hostname={{ parsed_system_data.hostname }}
          uptime_seconds={{ parsed_system_data.uptime_seconds }}
          load_average={{ parsed_system_data.load_average }}
          
          [processes]
          total_processes={{ parsed_system_data.total_processes }}
          listening_ports={{ parsed_system_data.listening_ports }}
          
          [storage]
          disk_usage_percent={{ parsed_system_data.disk_usage_percent }}
          
          [detailed_info]
          {% for desc, result in system_info.items() %}
          # {{ desc }}
          {{ desc | replace(' ', '_') | lower }}={{ result | replace('\n', ' | ') }}
          
          {% endfor %}
          
          [network_details]
          {% for desc, result in network_info.items() %}
          # {{ desc }}
          {{ desc | replace(' ', '_') | lower }}={{ result | replace('\n', ' | ') }}
          
          {% endfor %}
          
          [metadata]
          generated_at={{ ansible_date_time.iso8601 }}
          generated_by=ansible_pipe_lookup
        dest: "/tmp/system-report.conf"
        mode: '0644'
    
    # Security-focused pipe lookups
    - name: Gather security-related information
      set_fact:
        security_info: |
          {
            "failed_login_attempts": "{{ lookup('pipe', 'grep \"Failed password\" /var/log/auth.log 2>/dev/null | wc -l || echo \"0\"', errors='ignore') }}",
            "active_ssh_sessions": "{{ lookup('pipe', 'who | grep pts | wc -l', errors='ignore') }}",
            "sudo_usage": "{{ lookup('pipe', 'grep sudo /var/log/auth.log 2>/dev/null | tail -5 | wc -l || echo \"0\"', errors='ignore') }}",
            "open_files": "{{ lookup('pipe', 'lsof | wc -l', errors='ignore') }}",
            "running_services": "{{ lookup('pipe', 'systemctl list-units --type=service --state=running | grep -c \"running\"', errors='ignore') }}"
          }
    
    - name: Parse security information
      set_fact:
        parsed_security_info: "{{ security_info | from_json }}"
    
    - name: Create security report
      copy:
        content: |
          # Security Report
          # Generated using pipe lookups
          
          [authentication]
          failed_login_attempts={{ parsed_security_info.failed_login_attempts }}
          active_ssh_sessions={{ parsed_security_info.active_ssh_sessions }}
          sudo_usage_recent={{ parsed_security_info.sudo_usage }}
          
          [system_activity]
          open_files={{ parsed_security_info.open_files }}
          running_services={{ parsed_security_info.running_services }}
          
          [alerts]
          {% if parsed_security_info.failed_login_attempts | int > 10 %}
          high_failed_logins=true
          {% endif %}
          {% if parsed_security_info.active_ssh_sessions | int > 5 %}
          many_ssh_sessions=true
          {% endif %}
          
          [metadata]
          scan_timestamp={{ ansible_date_time.iso8601 }}
          scan_type=security_baseline
        dest: "/tmp/security-report.conf"
        mode: '0600'  # Secure permissions for security data
    
    # Custom data processing with pipe lookups
    - name: Process custom application data
      set_fact:
        app_metrics: |
          {
            "python_processes": "{{ lookup('pipe', 'ps aux | grep python | grep -v grep | wc -l', errors='ignore') }}",
            "nginx_status": "{{ lookup('pipe', 'systemctl is-active nginx 2>/dev/null || echo \"inactive\"', errors='ignore') }}",
            "docker_containers": "{{ lookup('pipe', 'docker ps -q 2>/dev/null | wc -l || echo \"0\"', errors='ignore') }}",
            "available_updates": "{{ lookup('pipe', 'apt list --upgradable 2>/dev/null | wc -l || echo \"0\"', errors='ignore') }}"
          }
    
    - name: Display application metrics
      debug:
        msg: |
          Application Metrics:
          {{ app_metrics | from_json | to_nice_yaml }}
```

### Run pipe and command lookups:
```bash
# Run pipe and command lookups
ansible-playbook -i inventory.ini pipe-command-lookups.yml

# Check generated reports
cat /tmp/system-report.conf
cat /tmp/security-report.conf
```

## Verification and Discussion

### 1. Check Results
```bash
# Check CSV lookup results
ls -la /tmp/*-server-config.conf
cat /tmp/web01-server-config.conf

# Check network lookup results
cat /tmp/network-config.conf
cat /tmp/health-check-config.conf

# Check pipe lookup results
cat /tmp/system-report.conf
cat /tmp/security-report.conf
```

### 2. Discussion Points
- How do you currently integrate external data sources into your automation?
- What strategies do you use for network discovery and monitoring?
- How do you handle system information gathering in your current workflows?

### 3. Clean Up
```bash
# Remove demo files
rm -rf data/ /tmp/*-server-config.conf /tmp/*-applications.conf /tmp/network-config.conf /tmp/health-check-config.conf /tmp/system-report.conf /tmp/security-report.conf
```

## Key Takeaways
- CSV file lookups enable integration with spreadsheet and database exports
- DNS lookups provide network discovery and validation capabilities
- URL lookups enable API integration and external service monitoring
- Pipe lookups allow execution of system commands for information gathering
- Advanced lookups can be combined for comprehensive system analysis
- Error handling is crucial when dealing with external data sources and network operations

## Next Steps
Proceed to Lab 3.3: Lookups in Loops and Complex Scenarios

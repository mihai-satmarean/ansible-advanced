# Lab 8.1: Delegation Fundamentals

## Objective
Master the fundamentals of Ansible delegation by understanding delegation concepts, implementing basic task delegation, working with delegated facts and variables, and creating delegation patterns for common automation scenarios.

## Duration
20 minutes

## Prerequisites
- Understanding of Ansible inventories and host patterns
- Basic knowledge of task execution and facts gathering
- Familiarity with variables and host-specific data
- Understanding of service orchestration concepts

## Lab Setup

```bash
cd ~/ansible-labs
mkdir -p module-08
cd module-08

# Create inventory for delegation testing
cat > inventory.ini << 'EOF'
[web_servers]
web1 ansible_host=localhost ansible_connection=local web_port=8080 app_env=production
web2 ansible_host=localhost ansible_connection=local web_port=8081 app_env=production
web3 ansible_host=localhost ansible_connection=local web_port=8082 app_env=staging

[database_servers]
db1 ansible_host=localhost ansible_connection=local db_port=5432 db_role=primary
db2 ansible_host=localhost ansible_connection=local db_port=5433 db_role=replica

[load_balancers]
lb1 ansible_host=localhost ansible_connection=local lb_port=80 lb_algorithm=round_robin
lb2 ansible_host=localhost ansible_connection=local lb_port=443 lb_algorithm=least_conn

[monitoring_servers]
monitor1 ansible_host=localhost ansible_connection=local monitor_port=9090
monitor2 ansible_host=localhost ansible_connection=local monitor_port=9091

[management_servers]
mgmt1 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
datacenter=dc1
region=us-east-1
EOF
```

## Exercise 1: Basic Delegation Concepts (8 minutes)

### Task: Understand and implement basic task delegation patterns

Create basic delegation playbook:

```yaml
# Create basic-delegation.yml
cat > basic-delegation.yml << 'EOF'
---
- name: Basic Delegation Fundamentals
  hosts: web_servers
  gather_facts: yes
  
  vars:
    deployment_timestamp: "{{ ansible_date_time.iso8601 }}"
    
  tasks:
    - name: Display current target host
      debug:
        msg: |
          Current target host: {{ inventory_hostname }}
          Host IP: {{ ansible_default_ipv4.address | default('localhost') }}
          Web Port: {{ web_port }}
          Environment: {{ app_env }}
    
    - name: Create local application directory
      file:
        path: "/tmp/app-{{ inventory_hostname }}"
        state: directory
        mode: '0755'
    
    - name: Create application status file locally
      copy:
        content: |
          Application Status for {{ inventory_hostname }}
          =====================================
          Host: {{ inventory_hostname }}
          Port: {{ web_port }}
          Environment: {{ app_env }}
          Status: Running
          Last Updated: {{ deployment_timestamp }}
          Datacenter: {{ datacenter }}
          Region: {{ region }}
        dest: "/tmp/app-{{ inventory_hostname }}/status.txt"
        mode: '0644'
    
    # Basic delegation - execute task on different host
    - name: Register web server with load balancer (delegated)
      debug:
        msg: |
          Registering {{ inventory_hostname }}:{{ web_port }} with load balancer {{ item }}
          Load Balancer Algorithm: {{ hostvars[item]['lb_algorithm'] }}
          Registration Time: {{ deployment_timestamp }}
      delegate_to: "{{ item }}"
      loop: "{{ groups['load_balancers'] }}"
    
    # Delegation with file creation on target host
    - name: Create registration file on load balancer
      copy:
        content: |
          # Web Server Registration
          # Generated: {{ deployment_timestamp }}
          
          server {{ inventory_hostname }} {
            address = "{{ ansible_default_ipv4.address | default('127.0.0.1') }}"
            port = {{ web_port }}
            environment = "{{ app_env }}"
            health_check = "/health"
            weight = 100
            status = "active"
            registered_by = "{{ ansible_user_id }}"
            registration_time = "{{ deployment_timestamp }}"
          }
        dest: "/tmp/lb-{{ item }}/servers/{{ inventory_hostname }}.conf"
        mode: '0644'
      delegate_to: "{{ item }}"
      loop: "{{ groups['load_balancers'] }}"
    
    # Ensure directory exists on delegated host first
    - name: Create load balancer configuration directories
      file:
        path: "/tmp/lb-{{ item }}/servers"
        state: directory
        mode: '0755'
      delegate_to: "{{ item }}"
      loop: "{{ groups['load_balancers'] }}"
      run_once: true
    
    # Delegation to monitoring server
    - name: Register web server with monitoring (delegated)
      lineinfile:
        path: "/tmp/monitoring-{{ item }}/targets.yml"
        line: |2
            - targets: ['{{ ansible_default_ipv4.address | default('127.0.0.1') }}:{{ web_port }}']
              labels:
                job: 'web-server'
                instance: '{{ inventory_hostname }}'
                environment: '{{ app_env }}'
                datacenter: '{{ datacenter }}'
                region: '{{ region }}'
        create: yes
        mode: '0644'
      delegate_to: "{{ item }}"
      loop: "{{ groups['monitoring_servers'] }}"
    
    # Delegation with facts gathering
    - name: Gather facts from management server (delegated)
      setup:
      delegate_to: "{{ groups['management_servers'][0] }}"
      delegate_facts: true
    
    - name: Display management server information
      debug:
        msg: |
          Management Server Info (delegated facts):
          Hostname: {{ hostvars[groups['management_servers'][0]]['ansible_hostname'] }}
          OS: {{ hostvars[groups['management_servers'][0]]['ansible_distribution'] }} {{ hostvars[groups['management_servers'][0]]['ansible_distribution_version'] }}
          Memory: {{ hostvars[groups['management_servers'][0]]['ansible_memtotal_mb'] }}MB
          CPU Cores: {{ hostvars[groups['management_servers'][0]]['ansible_processor_vcpus'] }}
      run_once: true
    
    # Delegation with variable access
    - name: Create deployment summary on management server
      copy:
        content: |
          Deployment Summary
          ==================
          Timestamp: {{ deployment_timestamp }}
          Deployed by: {{ ansible_user_id }}
          
          Web Servers Deployed:
          {% for host in groups['web_servers'] %}
          - {{ host }}:
            Port: {{ hostvars[host]['web_port'] }}
            Environment: {{ hostvars[host]['app_env'] }}
            Status: Active
          {% endfor %}
          
          Load Balancers Updated:
          {% for host in groups['load_balancers'] %}
          - {{ host }}:
            Port: {{ hostvars[host]['lb_port'] }}
            Algorithm: {{ hostvars[host]['lb_algorithm'] }}
          {% endfor %}
          
          Monitoring Configured:
          {% for host in groups['monitoring_servers'] %}
          - {{ host }}:
            Port: {{ hostvars[host]['monitor_port'] }}
          {% endfor %}
          
          Total Servers: {{ groups['web_servers'] | length }}
          Environments: {{ groups['web_servers'] | map('extract', hostvars, 'app_env') | unique | list | join(', ') }}
        dest: "/tmp/mgmt-{{ groups['management_servers'][0] }}/deployment-summary.txt"
        mode: '0644'
      delegate_to: "{{ groups['management_servers'][0] }}"
      run_once: true
EOF

# Run basic delegation playbook
ansible-playbook -i inventory.ini basic-delegation.yml
```

Verify delegation results:

```bash
# Check delegation results
echo "=== Basic Delegation Results ==="

# Check local application files
echo "Local application directories:"
ls -la /tmp/app-* 2>/dev/null || echo "No application directories found"

# Check load balancer configurations
echo "Load balancer configurations:"
find /tmp/lb-* -name "*.conf" 2>/dev/null | head -5 | while read file; do
    echo "File: $file"
    head -5 "$file"
    echo "---"
done

# Check monitoring configurations
echo "Monitoring configurations:"
find /tmp/monitoring-* -name "targets.yml" 2>/dev/null | head -2 | while read file; do
    echo "File: $file"
    head -10 "$file"
    echo "---"
done

# Check management server deployment summary
echo "Deployment summary:"
find /tmp/mgmt-* -name "deployment-summary.txt" 2>/dev/null | while read file; do
    echo "File: $file"
    head -15 "$file"
done
```

## Exercise 2: Delegation with Facts and Variables (7 minutes)

### Task: Implement advanced delegation patterns with facts gathering and variable manipulation

Create advanced delegation playbook:

```yaml
# Create advanced-delegation.yml
cat > advanced-delegation.yml << 'EOF'
---
- name: Advanced Delegation with Facts and Variables
  hosts: database_servers
  gather_facts: yes
  
  vars:
    backup_timestamp: "{{ ansible_date_time.epoch }}"
    backup_location: "/tmp/backups"
    
  tasks:
    - name: Display database server information
      debug:
        msg: |
          Database Server: {{ inventory_hostname }}
          Port: {{ db_port }}
          Role: {{ db_role }}
          Backup Timestamp: {{ backup_timestamp }}
    
    # Delegation with conditional execution
    - name: Create backup directory on management server
      file:
        path: "{{ backup_location }}/{{ ansible_date_time.date }}"
        state: directory
        mode: '0755'
      delegate_to: "{{ groups['management_servers'][0] }}"
      run_once: true
    
    # Delegation with facts from multiple hosts
    - name: Gather facts from all web servers (delegated)
      setup:
      delegate_to: "{{ item }}"
      delegate_facts: true
      loop: "{{ groups['web_servers'] }}"
      run_once: true
    
    - name: Create database connection configuration based on web server facts
      copy:
        content: |
          # Database Connection Configuration
          # Generated: {{ ansible_date_time.iso8601 }}
          # For database: {{ inventory_hostname }}
          
          [database]
          host = "{{ ansible_default_ipv4.address | default('127.0.0.1') }}"
          port = {{ db_port }}
          role = "{{ db_role }}"
          
          [web_servers]
          {% for web_host in groups['web_servers'] %}
          [[web_servers.{{ web_host }}]]
          hostname = "{{ hostvars[web_host]['ansible_hostname'] | default(web_host) }}"
          ip_address = "{{ hostvars[web_host]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
          port = {{ hostvars[web_host]['web_port'] }}
          environment = "{{ hostvars[web_host]['app_env'] }}"
          memory_mb = {{ hostvars[web_host]['ansible_memtotal_mb'] | default(0) }}
          cpu_cores = {{ hostvars[web_host]['ansible_processor_vcpus'] | default(1) }}
          
          {% endfor %}
          
          [connection_pool]
          max_connections = {{ (groups['web_servers'] | length * 10) }}
          timeout = 30
          retry_attempts = 3
          
          [backup]
          enabled = true
          location = "{{ backup_location }}"
          timestamp = "{{ backup_timestamp }}"
        dest: "/tmp/db-{{ inventory_hostname }}/connection-config.toml"
        mode: '0644'
    
    # Create database directory first
    - name: Create database configuration directory
      file:
        path: "/tmp/db-{{ inventory_hostname }}"
        state: directory
        mode: '0755'
    
    # Delegation with variable calculation
    - name: Calculate cluster statistics on management server
      copy:
        content: |
          # Database Cluster Statistics
          # Generated: {{ ansible_date_time.iso8601 }}
          
          [cluster_info]
          total_databases = {{ groups['database_servers'] | length }}
          primary_databases = {{ groups['database_servers'] | map('extract', hostvars, 'db_role') | select('equalto', 'primary') | list | length }}
          replica_databases = {{ groups['database_servers'] | map('extract', hostvars, 'db_role') | select('equalto', 'replica') | list | length }}
          
          [databases]
          {% for db_host in groups['database_servers'] %}
          [[databases.{{ db_host }}]]
          hostname = "{{ db_host }}"
          port = {{ hostvars[db_host]['db_port'] }}
          role = "{{ hostvars[db_host]['db_role'] }}"
          ip_address = "{{ hostvars[db_host]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
          
          {% endfor %}
          
          [web_server_connections]
          total_web_servers = {{ groups['web_servers'] | length }}
          production_servers = {{ groups['web_servers'] | map('extract', hostvars, 'app_env') | select('equalto', 'production') | list | length }}
          staging_servers = {{ groups['web_servers'] | map('extract', hostvars, 'app_env') | select('equalto', 'staging') | list | length }}
          
          estimated_connections_per_db = {{ ((groups['web_servers'] | length * 5) / groups['database_servers'] | length) | round | int }}
          
          [load_balancers]
          total_load_balancers = {{ groups['load_balancers'] | length }}
          {% for lb_host in groups['load_balancers'] %}
          [[load_balancers.{{ lb_host }}]]
          hostname = "{{ lb_host }}"
          port = {{ hostvars[lb_host]['lb_port'] }}
          algorithm = "{{ hostvars[lb_host]['lb_algorithm'] }}"
          
          {% endfor %}
        dest: "/tmp/mgmt-{{ groups['management_servers'][0] }}/cluster-statistics.toml"
        mode: '0644'
      delegate_to: "{{ groups['management_servers'][0] }}"
      run_once: true
    
    # Delegation with conditional based on host role
    - name: Configure primary database backup (only for primary)
      copy:
        content: |
          #!/bin/bash
          # Primary Database Backup Script
          # Generated: {{ ansible_date_time.iso8601 }}
          
          DB_HOST="{{ ansible_default_ipv4.address | default('127.0.0.1') }}"
          DB_PORT="{{ db_port }}"
          BACKUP_DIR="{{ backup_location }}/{{ ansible_date_time.date }}"
          BACKUP_FILE="primary-db-{{ inventory_hostname }}-{{ backup_timestamp }}.sql"
          
          echo "Starting backup for primary database {{ inventory_hostname }}"
          echo "Timestamp: $(date)"
          echo "Backup location: $BACKUP_DIR/$BACKUP_FILE"
          
          # Simulate database backup
          mkdir -p "$BACKUP_DIR"
          echo "-- Database backup for {{ inventory_hostname }}" > "$BACKUP_DIR/$BACKUP_FILE"
          echo "-- Generated: {{ ansible_date_time.iso8601 }}" >> "$BACKUP_DIR/$BACKUP_FILE"
          echo "-- Database role: {{ db_role }}" >> "$BACKUP_DIR/$BACKUP_FILE"
          echo "-- Port: {{ db_port }}" >> "$BACKUP_DIR/$BACKUP_FILE"
          
          # Add some sample backup content
          for i in {1..10}; do
            echo "INSERT INTO backup_log (id, timestamp, status) VALUES ($i, '$(date)', 'completed');" >> "$BACKUP_DIR/$BACKUP_FILE"
          done
          
          echo "Backup completed successfully"
          echo "File size: $(wc -c < "$BACKUP_DIR/$BACKUP_FILE") bytes"
        dest: "/tmp/mgmt-{{ groups['management_servers'][0] }}/backup-primary-{{ inventory_hostname }}.sh"
        mode: '0755'
      delegate_to: "{{ groups['management_servers'][0] }}"
      when: db_role == 'primary'
    
    # Delegation with replica-specific configuration
    - name: Configure replica database sync (only for replicas)
      copy:
        content: |
          # Replica Sync Configuration
          # Generated: {{ ansible_date_time.iso8601 }}
          # Replica: {{ inventory_hostname }}
          
          [replica_config]
          replica_host = "{{ ansible_default_ipv4.address | default('127.0.0.1') }}"
          replica_port = {{ db_port }}
          replica_role = "{{ db_role }}"
          
          [primary_connection]
          {% set primary_db = groups['database_servers'] | map('extract', hostvars) | selectattr('db_role', 'equalto', 'primary') | first %}
          primary_host = "{{ primary_db['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
          primary_port = {{ primary_db['db_port'] }}
          
          [sync_settings]
          sync_interval = 30
          max_lag_seconds = 60
          auto_failover = false
          
          [monitoring]
          health_check_interval = 10
          log_sync_status = true
          alert_on_lag = true
          
          # Replica-specific settings
          read_only = true
          backup_enabled = false
          connection_limit = {{ (groups['web_servers'] | length * 3) }}
        dest: "/tmp/db-{{ inventory_hostname }}/replica-sync.conf"
        mode: '0644'
      when: db_role == 'replica'
    
    # Cross-host validation using delegation
    - name: Validate database connectivity from web servers
      shell: |
        echo "Testing connection from web server {{ item }} to database {{ inventory_hostname }}"
        echo "Web Server: {{ item }}:{{ hostvars[item]['web_port'] }}"
        echo "Database: {{ inventory_hostname }}:{{ db_port }}"
        echo "Connection test: SUCCESS"
        echo "Timestamp: {{ ansible_date_time.iso8601 }}"
      delegate_to: "{{ item }}"
      loop: "{{ groups['web_servers'] }}"
      register: connectivity_tests
      changed_when: false
    
    - name: Create connectivity report on management server
      copy:
        content: |
          # Database Connectivity Report
          # Generated: {{ ansible_date_time.iso8601 }}
          # Database: {{ inventory_hostname }}
          
          {% for test in connectivity_tests.results %}
          ## Test from {{ test.item }}
          {{ test.stdout }}
          
          {% endfor %}
          
          Summary:
          - Database: {{ inventory_hostname }}:{{ db_port }} ({{ db_role }})
          - Web servers tested: {{ groups['web_servers'] | length }}
          - All tests: PASSED
          - Test timestamp: {{ ansible_date_time.iso8601 }}
        dest: "/tmp/mgmt-{{ groups['management_servers'][0] }}/connectivity-{{ inventory_hostname }}.txt"
        mode: '0644'
      delegate_to: "{{ groups['management_servers'][0] }}"
EOF

# Run advanced delegation playbook
ansible-playbook -i inventory.ini advanced-delegation.yml
```

## Exercise 3: Delegation Patterns and Best Practices (5 minutes)

### Task: Implement common delegation patterns and understand best practices

Create delegation patterns playbook:

```yaml
# Create delegation-patterns.yml
cat > delegation-patterns.yml << 'EOF'
---
- name: Common Delegation Patterns and Best Practices
  hosts: all
  gather_facts: yes
  
  vars:
    orchestration_id: "{{ ansible_date_time.epoch }}"
    
  tasks:
    # Pattern 1: Service Registration Pattern
    - name: Service Registration Pattern
      block:
        - name: Register service with discovery system
          copy:
            content: |
              {
                "service_id": "{{ inventory_hostname }}",
                "service_name": "{{ group_names[0] if group_names else 'unknown' }}",
                "address": "{{ ansible_default_ipv4.address | default('127.0.0.1') }}",
                "port": {{ hostvars[inventory_hostname].get(group_names[0] + '_port', 8080) if group_names else 8080 }},
                "tags": {{ group_names | to_json }},
                "health_check": {
                  "http": "http://{{ ansible_default_ipv4.address | default('127.0.0.1') }}:{{ hostvars[inventory_hostname].get(group_names[0] + '_port', 8080) if group_names else 8080 }}/health",
                  "interval": "10s"
                },
                "meta": {
                  "datacenter": "{{ datacenter }}",
                  "region": "{{ region }}",
                  "registered_at": "{{ ansible_date_time.iso8601 }}",
                  "orchestration_id": "{{ orchestration_id }}"
                }
              }
            dest: "/tmp/service-registry/{{ inventory_hostname }}.json"
            mode: '0644'
          delegate_to: "{{ groups['management_servers'][0] }}"
        
        # Ensure directory exists
        - name: Create service registry directory
          file:
            path: "/tmp/service-registry"
            state: directory
            mode: '0755'
          delegate_to: "{{ groups['management_servers'][0] }}"
          run_once: true
      when: inventory_hostname in groups['web_servers'] or inventory_hostname in groups['database_servers']
    
    # Pattern 2: Configuration Distribution Pattern
    - name: Configuration Distribution Pattern
      block:
        - name: Generate configuration for each service type
          template:
            src: service-config.j2
            dest: "/tmp/configs/{{ group_names[0] }}/{{ inventory_hostname }}.conf"
            mode: '0644'
          delegate_to: "{{ groups['management_servers'][0] }}"
          when: group_names | length > 0
        
        # Create template inline for demo
        - name: Create configuration template
          copy:
            content: |
              # Configuration for {{ inventory_hostname }}
              # Service Type: {{ group_names[0] if group_names else 'unknown' }}
              # Generated: {{ ansible_date_time.iso8601 }}
              
              [service]
              name = "{{ inventory_hostname }}"
              type = "{{ group_names[0] if group_names else 'unknown' }}"
              datacenter = "{{ datacenter }}"
              region = "{{ region }}"
              
              [network]
              address = "{{ ansible_default_ipv4.address | default('127.0.0.1') }}"
              {% if inventory_hostname in groups['web_servers'] %}
              port = {{ web_port }}
              {% elif inventory_hostname in groups['database_servers'] %}
              port = {{ db_port }}
              {% elif inventory_hostname in groups['load_balancers'] %}
              port = {{ lb_port }}
              {% elif inventory_hostname in groups['monitoring_servers'] %}
              port = {{ monitor_port }}
              {% else %}
              port = 8080
              {% endif %}
              
              [metadata]
              orchestration_id = "{{ orchestration_id }}"
              {% if inventory_hostname in groups['web_servers'] %}
              environment = "{{ app_env }}"
              {% elif inventory_hostname in groups['database_servers'] %}
              role = "{{ db_role }}"
              {% elif inventory_hostname in groups['load_balancers'] %}
              algorithm = "{{ lb_algorithm }}"
              {% endif %}
            dest: "/tmp/configs/{{ group_names[0] if group_names else 'default' }}/{{ inventory_hostname }}.conf"
            mode: '0644'
          delegate_to: "{{ groups['management_servers'][0] }}"
        
        # Create config directories
        - name: Create configuration directories
          file:
            path: "/tmp/configs/{{ item }}"
            state: directory
            mode: '0755'
          delegate_to: "{{ groups['management_servers'][0] }}"
          loop:
            - web_servers
            - database_servers
            - load_balancers
            - monitoring_servers
            - default
          run_once: true
    
    # Pattern 3: Health Check Aggregation Pattern
    - name: Health Check Aggregation Pattern
      block:
        - name: Perform local health check
          shell: |
            echo "Health check for {{ inventory_hostname }}"
            echo "Service type: {{ group_names[0] if group_names else 'unknown' }}"
            echo "Status: HEALTHY"
            echo "Timestamp: {{ ansible_date_time.iso8601 }}"
            echo "Uptime: {{ ansible_uptime_seconds | default(0) }} seconds"
            echo "Memory usage: {{ ((ansible_memtotal_mb - ansible_memfree_mb) / ansible_memtotal_mb * 100) | round(1) }}%"
            echo "Load average: {{ ansible_loadavg['1m'] | default(0.0) }}"
          register: health_check
          changed_when: false
        
        - name: Aggregate health check results
          copy:
            content: |
              # Health Check Result for {{ inventory_hostname }}
              # Timestamp: {{ ansible_date_time.iso8601 }}
              
              {{ health_check.stdout }}
              
              # Additional Metrics
              CPU Cores: {{ ansible_processor_vcpus | default(1) }}
              Total Memory: {{ ansible_memtotal_mb | default(0) }}MB
              Free Memory: {{ ansible_memfree_mb | default(0) }}MB
              Disk Usage: {{ ansible_mounts[0]['size_available'] | default(0) if ansible_mounts else 0 }} bytes available
              
              # Service-specific health
              {% if inventory_hostname in groups['web_servers'] %}
              Web Port: {{ web_port }}
              Environment: {{ app_env }}
              {% elif inventory_hostname in groups['database_servers'] %}
              Database Port: {{ db_port }}
              Database Role: {{ db_role }}
              {% elif inventory_hostname in groups['load_balancers'] %}
              Load Balancer Port: {{ lb_port }}
              Algorithm: {{ lb_algorithm }}
              {% elif inventory_hostname in groups['monitoring_servers'] %}
              Monitor Port: {{ monitor_port }}
              {% endif %}
            dest: "/tmp/health-checks/{{ inventory_hostname }}.txt"
            mode: '0644'
          delegate_to: "{{ groups['management_servers'][0] }}"
        
        # Create health check directory
        - name: Create health check directory
          file:
            path: "/tmp/health-checks"
            state: directory
            mode: '0755'
          delegate_to: "{{ groups['management_servers'][0] }}"
          run_once: true
    
    # Pattern 4: Orchestration Summary Pattern
    - name: Create orchestration summary
      copy:
        content: |
          # Orchestration Summary
          # ID: {{ orchestration_id }}
          # Generated: {{ ansible_date_time.iso8601 }}
          # Executed by: {{ ansible_user_id }}
          
          ## Infrastructure Overview
          Total Hosts: {{ groups['all'] | length }}
          Web Servers: {{ groups['web_servers'] | length }}
          Database Servers: {{ groups['database_servers'] | length }}
          Load Balancers: {{ groups['load_balancers'] | length }}
          Monitoring Servers: {{ groups['monitoring_servers'] | length }}
          Management Servers: {{ groups['management_servers'] | length }}
          
          ## Service Registration
          Services Registered: {{ (groups['web_servers'] | length) + (groups['database_servers'] | length) }}
          
          ## Configuration Distribution
          Configurations Generated: {{ groups['all'] | length }}
          
          ## Health Checks
          Health Checks Performed: {{ groups['all'] | length }}
          
          ## Delegation Patterns Used
          1. Service Registration Pattern - ✓
          2. Configuration Distribution Pattern - ✓
          3. Health Check Aggregation Pattern - ✓
          4. Orchestration Summary Pattern - ✓
          
          ## Environment Distribution
          {% set env_counts = {} %}
          {% for host in groups['web_servers'] %}
          {%   set env = hostvars[host]['app_env'] %}
          {%   if env in env_counts %}
          {%     set _ = env_counts.update({env: env_counts[env] + 1}) %}
          {%   else %}
          {%     set _ = env_counts.update({env: 1}) %}
          {%   endif %}
          {% endfor %}
          {% for env, count in env_counts.items() %}
          {{ env }}: {{ count }} servers
          {% endfor %}
          
          ## Database Roles
          {% set role_counts = {} %}
          {% for host in groups['database_servers'] %}
          {%   set role = hostvars[host]['db_role'] %}
          {%   if role in role_counts %}
          {%     set _ = role_counts.update({role: role_counts[role] + 1}) %}
          {%   else %}
          {%     set _ = role_counts.update({role: 1}) %}
          {%   endif %}
          {% endfor %}
          {% for role, count in role_counts.items() %}
          {{ role }}: {{ count }} databases
          {% endfor %}
          
          ## Load Balancer Algorithms
          {% for host in groups['load_balancers'] %}
          {{ host }}: {{ hostvars[host]['lb_algorithm'] }}
          {% endfor %}
        dest: "/tmp/mgmt-{{ groups['management_servers'][0] }}/orchestration-summary-{{ orchestration_id }}.txt"
        mode: '0644'
      delegate_to: "{{ groups['management_servers'][0] }}"
      run_once: true
EOF

# Run delegation patterns playbook
ansible-playbook -i inventory.ini delegation-patterns.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Check delegation pattern results
echo "=== Delegation Patterns Results ==="

# Check service registry
echo "Service Registry:"
ls -la /tmp/service-registry/ 2>/dev/null | head -5

# Check configurations
echo "Generated Configurations:"
find /tmp/configs -name "*.conf" 2>/dev/null | head -5 | while read file; do
    echo "Config: $file"
    head -5 "$file"
    echo "---"
done

# Check health checks
echo "Health Check Results:"
find /tmp/health-checks -name "*.txt" 2>/dev/null | head -3 | while read file; do
    echo "Health Check: $file"
    head -10 "$file"
    echo "---"
done

# Check orchestration summary
echo "Orchestration Summary:"
find /tmp/mgmt-* -name "orchestration-summary-*.txt" 2>/dev/null | while read file; do
    echo "Summary: $file"
    head -20 "$file"
done

# Check database configurations
echo "Database Configurations:"
find /tmp/db-* -name "*.conf" -o -name "*.toml" 2>/dev/null | head -3 | while read file; do
    echo "DB Config: $file"
    head -10 "$file"
    echo "---"
done
```

### 2. Discussion Points
- When should you use delegation vs running tasks locally?
- How does delegation affect performance and execution time?
- What are the security considerations when using delegation?
- How do you handle delegation failures and error recovery?

### 3. Clean Up
```bash
# Clean up test files (optional)
# rm -rf /tmp/app-* /tmp/lb-* /tmp/monitoring-* /tmp/mgmt-* /tmp/db-* /tmp/service-registry /tmp/configs /tmp/health-checks /tmp/backups
```

## Key Takeaways
- Delegation allows tasks to run on different hosts than the target hosts
- `delegate_to` specifies the host where the task should execute
- `delegate_facts` controls whether facts are gathered for the delegated host
- `run_once` combined with delegation creates powerful orchestration patterns
- Delegation is essential for service registration, configuration distribution, and cross-host coordination
- Proper use of `hostvars` enables access to variables from any host in the inventory

This lab provides comprehensive hands-on experience with Ansible delegation fundamentals, preparing you for implementing complex orchestration workflows in enterprise environments.

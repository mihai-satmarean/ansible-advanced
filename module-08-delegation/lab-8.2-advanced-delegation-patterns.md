# Lab 8.2: Advanced Delegation Patterns

## Objective
Implement advanced delegation patterns for complex orchestration workflows, including load balancer management, database coordination, multi-environment deployments, and enterprise-scale delegation architectures.

## Duration
20 minutes

## Prerequisites
- Completed Lab 8.1 (Delegation Fundamentals)
- Understanding of load balancer concepts and database replication
- Knowledge of multi-environment deployment strategies
- Familiarity with service orchestration patterns

## Lab Setup

```bash
cd ~/ansible-labs/module-08
mkdir -p lab-8.2
cd lab-8.2

# Create comprehensive inventory for advanced delegation
cat > inventory.ini << 'EOF'
[web_servers_prod]
web-prod-01 ansible_host=localhost ansible_connection=local web_port=8080 app_env=production tier=frontend
web-prod-02 ansible_host=localhost ansible_connection=local web_port=8081 app_env=production tier=frontend
web-prod-03 ansible_host=localhost ansible_connection=local web_port=8082 app_env=production tier=frontend

[web_servers_staging]
web-stage-01 ansible_host=localhost ansible_connection=local web_port=9080 app_env=staging tier=frontend
web-stage-02 ansible_host=localhost ansible_connection=local web_port=9081 app_env=staging tier=frontend

[api_servers_prod]
api-prod-01 ansible_host=localhost ansible_connection=local api_port=3000 app_env=production tier=backend
api-prod-02 ansible_host=localhost ansible_connection=local api_port=3001 app_env=production tier=backend

[api_servers_staging]
api-stage-01 ansible_host=localhost ansible_connection=local api_port=4000 app_env=staging tier=backend

[database_primary]
db-primary-01 ansible_host=localhost ansible_connection=local db_port=5432 db_role=primary app_env=production

[database_replicas]
db-replica-01 ansible_host=localhost ansible_connection=local db_port=5433 db_role=replica app_env=production
db-replica-02 ansible_host=localhost ansible_connection=local db_port=5434 db_role=replica app_env=production

[database_staging]
db-stage-01 ansible_host=localhost ansible_connection=local db_port=5435 db_role=primary app_env=staging

[load_balancers_prod]
lb-prod-01 ansible_host=localhost ansible_connection=local lb_port=80 lb_type=external app_env=production
lb-prod-02 ansible_host=localhost ansible_connection=local lb_port=443 lb_type=external app_env=production

[load_balancers_internal]
lb-internal-01 ansible_host=localhost ansible_connection=local lb_port=8000 lb_type=internal app_env=production

[load_balancers_staging]
lb-stage-01 ansible_host=localhost ansible_connection=local lb_port=8080 lb_type=external app_env=staging

[cache_servers]
cache-01 ansible_host=localhost ansible_connection=local cache_port=6379 cache_type=redis app_env=production
cache-02 ansible_host=localhost ansible_connection=local cache_port=6380 cache_type=redis app_env=production

[monitoring_stack]
prometheus-01 ansible_host=localhost ansible_connection=local monitor_port=9090 monitor_type=metrics
grafana-01 ansible_host=localhost ansible_connection=local monitor_port=3000 monitor_type=dashboard
alertmanager-01 ansible_host=localhost ansible_connection=local monitor_port=9093 monitor_type=alerts

[orchestration_controllers]
controller-01 ansible_host=localhost ansible_connection=local

[bastion_hosts]
bastion-01 ansible_host=localhost ansible_connection=local

# Group aggregations
[web_servers:children]
web_servers_prod
web_servers_staging

[api_servers:children]
api_servers_prod
api_servers_staging

[database_servers:children]
database_primary
database_replicas
database_staging

[load_balancers:children]
load_balancers_prod
load_balancers_internal
load_balancers_staging

[production:children]
web_servers_prod
api_servers_prod
database_primary
database_replicas
load_balancers_prod
load_balancers_internal
cache_servers

[staging:children]
web_servers_staging
api_servers_staging
database_staging
load_balancers_staging

[all:vars]
ansible_python_interpreter=/usr/bin/python3
datacenter=dc1
region=us-east-1
deployment_id=deploy-{{ ansible_date_time.epoch }}
EOF
```

## Exercise 1: Load Balancer Orchestration Pattern (8 minutes)

### Task: Implement complex load balancer management with service discovery and health checks

Create load balancer orchestration playbook:

```yaml
# Create lb-orchestration.yml
cat > lb-orchestration.yml << 'EOF'
---
- name: Advanced Load Balancer Orchestration
  hosts: web_servers:api_servers
  gather_facts: yes
  serial: 1  # Deploy one at a time for zero-downtime
  
  vars:
    deployment_id: "{{ ansible_date_time.epoch }}"
    health_check_retries: 3
    health_check_delay: 5
    
  pre_tasks:
    - name: Create orchestration directories on controller
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      delegate_to: "{{ groups['orchestration_controllers'][0] }}"
      loop:
        - /tmp/lb-orchestration/configs
        - /tmp/lb-orchestration/health-checks
        - /tmp/lb-orchestration/deployments
        - /tmp/lb-orchestration/rollback
      run_once: true
    
    - name: Log deployment start
      lineinfile:
        path: /tmp/lb-orchestration/deployments/deployment-{{ deployment_id }}.log
        line: "{{ ansible_date_time.iso8601 }} - Starting deployment for {{ inventory_hostname }} ({{ app_env }})"
        create: yes
        mode: '0644'
      delegate_to: "{{ groups['orchestration_controllers'][0] }}"
  
  tasks:
    # Step 1: Remove from load balancer before deployment
    - name: Remove server from load balancer (drain connections)
      block:
        - name: Create drain configuration for external load balancers
          copy:
            content: |
              # Drain configuration for {{ inventory_hostname }}
              # Generated: {{ ansible_date_time.iso8601 }}
              # Deployment ID: {{ deployment_id }}
              
              server {{ inventory_hostname }} {
                address = "{{ ansible_default_ipv4.address | default('127.0.0.1') }}"
                {% if inventory_hostname in groups['web_servers'] %}
                port = {{ web_port }}
                {% elif inventory_hostname in groups['api_servers'] %}
                port = {{ api_port }}
                {% endif %}
                status = "draining"
                weight = 0
                health_check = "/health"
                drain_timeout = 30
                max_connections = 0
                environment = "{{ app_env }}"
                tier = "{{ tier }}"
              }
            dest: "/tmp/lb-{{ item }}/draining/{{ inventory_hostname }}.conf"
            mode: '0644'
          delegate_to: "{{ item }}"
          loop: "{{ groups['load_balancers_' + app_env] if app_env in ['prod', 'staging'] else groups['load_balancers'] }}"
        
        # Create draining directories
        - name: Create load balancer draining directories
          file:
            path: "/tmp/lb-{{ item }}/draining"
            state: directory
            mode: '0755'
          delegate_to: "{{ item }}"
          loop: "{{ groups['load_balancers_' + app_env] if app_env in ['prod', 'staging'] else groups['load_balancers'] }}"
        
        - name: Wait for connection draining
          pause:
            seconds: 10
            prompt: "Draining connections for {{ inventory_hostname }}"
        
        - name: Log draining completion
          lineinfile:
            path: /tmp/lb-orchestration/deployments/deployment-{{ deployment_id }}.log
            line: "{{ ansible_date_time.iso8601 }} - Connections drained for {{ inventory_hostname }}"
            mode: '0644'
          delegate_to: "{{ groups['orchestration_controllers'][0] }}"
    
    # Step 2: Deploy application (simulated)
    - name: Deploy application
      block:
        - name: Simulate application deployment
          copy:
            content: |
              # Application Deployment for {{ inventory_hostname }}
              # Deployment ID: {{ deployment_id }}
              # Generated: {{ ansible_date_time.iso8601 }}
              
              [application]
              name = "{{ 'web-app' if inventory_hostname in groups['web_servers'] else 'api-app' }}"
              version = "2.1.{{ deployment_id[-4:] }}"
              environment = "{{ app_env }}"
              tier = "{{ tier }}"
              
              [server]
              hostname = "{{ inventory_hostname }}"
              {% if inventory_hostname in groups['web_servers'] %}
              port = {{ web_port }}
              {% elif inventory_hostname in groups['api_servers'] %}
              port = {{ api_port }}
              {% endif %}
              
              [deployment]
              deployment_id = "{{ deployment_id }}"
              deployed_by = "{{ ansible_user_id }}"
              deployed_at = "{{ ansible_date_time.iso8601 }}"
              
              [health_check]
              endpoint = "/health"
              timeout = 5
              interval = 10
              
              [database]
              {% if app_env == 'production' %}
              primary_host = "{{ hostvars[groups['database_primary'][0]]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
              primary_port = {{ hostvars[groups['database_primary'][0]]['db_port'] }}
              replica_hosts = [
              {% for replica in groups['database_replicas'] %}
                "{{ hostvars[replica]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}:{{ hostvars[replica]['db_port'] }}"{{ ',' if not loop.last else '' }}
              {% endfor %}
              ]
              {% else %}
              primary_host = "{{ hostvars[groups['database_staging'][0]]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
              primary_port = {{ hostvars[groups['database_staging'][0]]['db_port'] }}
              {% endif %}
              
              [cache]
              {% if app_env == 'production' %}
              servers = [
              {% for cache in groups['cache_servers'] %}
                "{{ hostvars[cache]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}:{{ hostvars[cache]['cache_port'] }}"{{ ',' if not loop.last else '' }}
              {% endfor %}
              ]
              {% else %}
              servers = ["127.0.0.1:6379"]
              {% endif %}
            dest: "/tmp/app-{{ inventory_hostname }}/config.toml"
            mode: '0644'
        
        # Create app directory
        - name: Create application directory
          file:
            path: "/tmp/app-{{ inventory_hostname }}"
            state: directory
            mode: '0755'
        
        - name: Log deployment completion
          lineinfile:
            path: /tmp/lb-orchestration/deployments/deployment-{{ deployment_id }}.log
            line: "{{ ansible_date_time.iso8601 }} - Application deployed for {{ inventory_hostname }}"
            mode: '0644'
          delegate_to: "{{ groups['orchestration_controllers'][0] }}"
    
    # Step 3: Health check before adding back to load balancer
    - name: Perform health checks
      block:
        - name: Execute health check
          uri:
            url: "http://{{ ansible_default_ipv4.address | default('127.0.0.1') }}:{{ web_port if inventory_hostname in groups['web_servers'] else api_port }}/health"
            method: GET
            timeout: 5
          register: health_check
          failed_when: false
          changed_when: false
          retries: "{{ health_check_retries }}"
          delay: "{{ health_check_delay }}"
          # Simulate health check success for lab
          ignore_errors: yes
        
        - name: Create health check report
          copy:
            content: |
              # Health Check Report for {{ inventory_hostname }}
              # Deployment ID: {{ deployment_id }}
              # Timestamp: {{ ansible_date_time.iso8601 }}
              
              [server_info]
              hostname = "{{ inventory_hostname }}"
              {% if inventory_hostname in groups['web_servers'] %}
              port = {{ web_port }}
              service_type = "web"
              {% elif inventory_hostname in groups['api_servers'] %}
              port = {{ api_port }}
              service_type = "api"
              {% endif %}
              environment = "{{ app_env }}"
              
              [health_check]
              status = "HEALTHY"  # Simulated for lab
              response_time_ms = {{ range(50, 200) | random }}
              checks_performed = {{ health_check_retries }}
              last_check = "{{ ansible_date_time.iso8601 }}"
              
              [system_metrics]
              cpu_usage = {{ range(10, 80) | random }}%
              memory_usage = {{ ((ansible_memtotal_mb - ansible_memfree_mb) / ansible_memtotal_mb * 100) | round(1) }}%
              disk_usage = {{ range(20, 70) | random }}%
              load_average = {{ ansible_loadavg['1m'] | default(0.5) }}
              
              [connectivity]
              database_connection = "OK"
              cache_connection = "OK"
              external_services = "OK"
            dest: "/tmp/lb-orchestration/health-checks/{{ inventory_hostname }}-{{ deployment_id }}.txt"
            mode: '0644'
          delegate_to: "{{ groups['orchestration_controllers'][0] }}"
        
        - name: Log health check completion
          lineinfile:
            path: /tmp/lb-orchestration/deployments/deployment-{{ deployment_id }}.log
            line: "{{ ansible_date_time.iso8601 }} - Health check passed for {{ inventory_hostname }}"
            mode: '0644'
          delegate_to: "{{ groups['orchestration_controllers'][0] }}"
    
    # Step 4: Add back to load balancer
    - name: Add server back to load balancer
      block:
        - name: Create active configuration for load balancers
          copy:
            content: |
              # Active configuration for {{ inventory_hostname }}
              # Generated: {{ ansible_date_time.iso8601 }}
              # Deployment ID: {{ deployment_id }}
              
              server {{ inventory_hostname }} {
                address = "{{ ansible_default_ipv4.address | default('127.0.0.1') }}"
                {% if inventory_hostname in groups['web_servers'] %}
                port = {{ web_port }}
                {% elif inventory_hostname in groups['api_servers'] %}
                port = {{ api_port }}
                {% endif %}
                status = "active"
                weight = 100
                health_check = "/health"
                max_connections = 1000
                environment = "{{ app_env }}"
                tier = "{{ tier }}"
                deployment_id = "{{ deployment_id }}"
                last_deployed = "{{ ansible_date_time.iso8601 }}"
              }
            dest: "/tmp/lb-{{ item }}/active/{{ inventory_hostname }}.conf"
            mode: '0644'
          delegate_to: "{{ item }}"
          loop: "{{ groups['load_balancers_' + app_env] if app_env in ['prod', 'staging'] else groups['load_balancers'] }}"
        
        # Create active directories
        - name: Create load balancer active directories
          file:
            path: "/tmp/lb-{{ item }}/active"
            state: directory
            mode: '0755'
          delegate_to: "{{ item }}"
          loop: "{{ groups['load_balancers_' + app_env] if app_env in ['prod', 'staging'] else groups['load_balancers'] }}"
        
        - name: Remove draining configuration
          file:
            path: "/tmp/lb-{{ item }}/draining/{{ inventory_hostname }}.conf"
            state: absent
          delegate_to: "{{ item }}"
          loop: "{{ groups['load_balancers_' + app_env] if app_env in ['prod', 'staging'] else groups['load_balancers'] }}"
        
        - name: Log server activation
          lineinfile:
            path: /tmp/lb-orchestration/deployments/deployment-{{ deployment_id }}.log
            line: "{{ ansible_date_time.iso8601 }} - Server {{ inventory_hostname }} activated in load balancer"
            mode: '0644'
          delegate_to: "{{ groups['orchestration_controllers'][0] }}"
  
  post_tasks:
    - name: Create deployment summary
      copy:
        content: |
          # Deployment Summary
          # Deployment ID: {{ deployment_id }}
          # Completed: {{ ansible_date_time.iso8601 }}
          
          [deployment_info]
          deployment_id = "{{ deployment_id }}"
          started_at = "{{ ansible_date_time.iso8601 }}"
          deployed_by = "{{ ansible_user_id }}"
          strategy = "rolling_deployment"
          
          [servers_deployed]
          {% for host in ansible_play_hosts %}
          [[servers_deployed.{{ host }}]]
          hostname = "{{ host }}"
          environment = "{{ hostvars[host]['app_env'] }}"
          tier = "{{ hostvars[host]['tier'] }}"
          {% if host in groups['web_servers'] %}
          port = {{ hostvars[host]['web_port'] }}
          service_type = "web"
          {% elif host in groups['api_servers'] %}
          port = {{ hostvars[host]['api_port'] }}
          service_type = "api"
          {% endif %}
          status = "deployed"
          
          {% endfor %}
          
          [load_balancers_updated]
          {% set lb_groups = [] %}
          {% for host in ansible_play_hosts %}
          {%   set env = hostvars[host]['app_env'] %}
          {%   set lb_group = 'load_balancers_' + env %}
          {%   if lb_group in groups and lb_group not in lb_groups %}
          {%     set _ = lb_groups.append(lb_group) %}
          {%   endif %}
          {% endfor %}
          {% for lb_group in lb_groups %}
          {% for lb in groups[lb_group] %}
          [[load_balancers_updated.{{ lb }}]]
          hostname = "{{ lb }}"
          port = {{ hostvars[lb]['lb_port'] }}
          type = "{{ hostvars[lb]['lb_type'] }}"
          environment = "{{ hostvars[lb]['app_env'] }}"
          
          {% endfor %}
          {% endfor %}
          
          [statistics]
          total_servers = {{ ansible_play_hosts | length }}
          web_servers = {{ ansible_play_hosts | intersect(groups['web_servers']) | length }}
          api_servers = {{ ansible_play_hosts | intersect(groups['api_servers']) | length }}
          production_servers = {{ ansible_play_hosts | map('extract', hostvars, 'app_env') | select('equalto', 'production') | list | length }}
          staging_servers = {{ ansible_play_hosts | map('extract', hostvars, 'app_env') | select('equalto', 'staging') | list | length }}
        dest: "/tmp/lb-orchestration/deployments/summary-{{ deployment_id }}.toml"
        mode: '0644'
      delegate_to: "{{ groups['orchestration_controllers'][0] }}"
      run_once: true
EOF

# Run load balancer orchestration
ansible-playbook -i inventory.ini lb-orchestration.yml --limit "web_servers_prod[0:1],api_servers_prod[0]"
```

## Exercise 2: Database Coordination Pattern (7 minutes)

### Task: Implement database coordination with primary/replica management and cross-environment synchronization

Create database coordination playbook:

```yaml
# Create db-coordination.yml
cat > db-coordination.yml << 'EOF'
---
- name: Advanced Database Coordination
  hosts: database_servers
  gather_facts: yes
  
  vars:
    coordination_id: "{{ ansible_date_time.epoch }}"
    backup_retention_days: 7
    
  pre_tasks:
    - name: Create database coordination directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      delegate_to: "{{ groups['orchestration_controllers'][0] }}"
      loop:
        - /tmp/db-coordination/backups
        - /tmp/db-coordination/replication
        - /tmp/db-coordination/monitoring
        - /tmp/db-coordination/maintenance
      run_once: true
  
  tasks:
    # Primary database tasks
    - name: Primary Database Operations
      block:
        - name: Create primary database backup
          shell: |
            echo "Creating backup for primary database {{ inventory_hostname }}"
            echo "Backup ID: backup-{{ coordination_id }}"
            echo "Timestamp: {{ ansible_date_time.iso8601 }}"
            
            # Simulate database backup
            mkdir -p "/tmp/db-{{ inventory_hostname }}/backups"
            
            cat > "/tmp/db-{{ inventory_hostname }}/backups/backup-{{ coordination_id }}.sql" << 'BACKUP_EOF'
            -- Database Backup for {{ inventory_hostname }}
            -- Generated: {{ ansible_date_time.iso8601 }}
            -- Backup ID: backup-{{ coordination_id }}
            
            -- Database schema and data
            CREATE DATABASE app_production;
            USE app_production;
            
            CREATE TABLE users (
              id SERIAL PRIMARY KEY,
              username VARCHAR(50) UNIQUE NOT NULL,
              email VARCHAR(100) UNIQUE NOT NULL,
              created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );
            
            CREATE TABLE sessions (
              id SERIAL PRIMARY KEY,
              user_id INTEGER REFERENCES users(id),
              token VARCHAR(255) UNIQUE NOT NULL,
              expires_at TIMESTAMP NOT NULL
            );
            
            -- Sample data
            INSERT INTO users (username, email) VALUES 
            ('admin', 'admin@example.com'),
            ('user1', 'user1@example.com'),
            ('user2', 'user2@example.com');
            
            -- Backup metadata
            INSERT INTO backup_log (backup_id, created_at, size_bytes, status) 
            VALUES ('backup-{{ coordination_id }}', '{{ ansible_date_time.iso8601 }}', 2048, 'completed');
            BACKUP_EOF
            
            echo "Backup completed: $(wc -c < "/tmp/db-{{ inventory_hostname }}/backups/backup-{{ coordination_id }}.sql") bytes"
          register: backup_result
          changed_when: true
        
        - name: Create database directory
          file:
            path: "/tmp/db-{{ inventory_hostname }}/backups"
            state: directory
            mode: '0755'
        
        - name: Register backup with coordination controller
          copy:
            content: |
              # Primary Database Backup Registration
              # Coordination ID: {{ coordination_id }}
              # Generated: {{ ansible_date_time.iso8601 }}
              
              [backup_info]
              backup_id = "backup-{{ coordination_id }}"
              database = "{{ inventory_hostname }}"
              role = "{{ db_role }}"
              environment = "{{ app_env }}"
              created_at = "{{ ansible_date_time.iso8601 }}"
              
              [backup_details]
              file_path = "/tmp/db-{{ inventory_hostname }}/backups/backup-{{ coordination_id }}.sql"
              size_bytes = {{ backup_result.stdout_lines[-1].split()[-2] if backup_result.stdout_lines else 2048 }}
              compression = "none"
              encryption = "none"
              
              [replication]
              replicas_to_update = [
              {% for replica in groups['database_replicas'] %}
                "{{ replica }}"{{ ',' if not loop.last else '' }}
              {% endfor %}
              ]
              
              [retention]
              retention_days = {{ backup_retention_days }}
              auto_cleanup = true
            dest: "/tmp/db-coordination/backups/{{ inventory_hostname }}-{{ coordination_id }}.toml"
            mode: '0644'
          delegate_to: "{{ groups['orchestration_controllers'][0] }}"
      when: db_role == 'primary'
    
    # Replica database tasks
    - name: Replica Database Operations
      block:
        - name: Configure replication from primary
          copy:
            content: |
              # Replication Configuration for {{ inventory_hostname }}
              # Coordination ID: {{ coordination_id }}
              # Generated: {{ ansible_date_time.iso8601 }}
              
              [replica_config]
              replica_id = "{{ inventory_hostname }}"
              role = "{{ db_role }}"
              environment = "{{ app_env }}"
              
              [primary_connection]
              {% set primary_db = groups['database_primary'][0] %}
              primary_host = "{{ hostvars[primary_db]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
              primary_port = {{ hostvars[primary_db]['db_port'] }}
              primary_database = "app_production"
              
              [replication_settings]
              replication_mode = "streaming"
              sync_mode = "asynchronous"
              max_lag_seconds = 60
              auto_failover = false
              
              [monitoring]
              lag_check_interval = 30
              health_check_interval = 10
              log_replication_status = true
              
              [backup_settings]
              backup_from_replica = true
              backup_schedule = "0 2 * * *"  # Daily at 2 AM
              backup_retention = {{ backup_retention_days }}
              
              # Connection details
              connection_string = "postgresql://replicator@{{ hostvars[primary_db]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}:{{ hostvars[primary_db]['db_port'] }}/app_production"
            dest: "/tmp/db-{{ inventory_hostname }}/replication.conf"
            mode: '0600'
        
        - name: Create database directory for replica
          file:
            path: "/tmp/db-{{ inventory_hostname }}"
            state: directory
            mode: '0755'
        
        - name: Simulate replication sync
          shell: |
            echo "Syncing replica {{ inventory_hostname }} with primary"
            echo "Coordination ID: {{ coordination_id }}"
            echo "Sync started: {{ ansible_date_time.iso8601 }}"
            
            # Simulate replication lag check
            lag_ms=$(( RANDOM % 100 + 10 ))
            echo "Current replication lag: ${lag_ms}ms"
            
            # Simulate sync process
            sleep 2
            
            echo "Replication sync completed successfully"
            echo "Final lag: $(( lag_ms / 2 ))ms"
          register: replication_sync
          changed_when: true
        
        - name: Register replication status
          copy:
            content: |
              # Replication Status for {{ inventory_hostname }}
              # Coordination ID: {{ coordination_id }}
              # Updated: {{ ansible_date_time.iso8601 }}
              
              [replica_status]
              replica_id = "{{ inventory_hostname }}"
              role = "{{ db_role }}"
              environment = "{{ app_env }}"
              status = "healthy"
              
              [replication_metrics]
              last_sync = "{{ ansible_date_time.iso8601 }}"
              replication_lag_ms = {{ range(10, 100) | random }}
              sync_status = "active"
              
              [sync_log]
              {{ replication_sync.stdout | replace('\n', '\n              ') }}
              
              [health_indicators]
              connection_status = "connected"
              data_consistency = "verified"
              performance_impact = "minimal"
            dest: "/tmp/db-coordination/replication/{{ inventory_hostname }}-status.txt"
            mode: '0644'
          delegate_to: "{{ groups['orchestration_controllers'][0] }}"
      when: db_role == 'replica'
    
    # Cross-environment coordination
    - name: Cross-Environment Database Coordination
      block:
        - name: Create environment synchronization report
          copy:
            content: |
              # Cross-Environment Database Coordination Report
              # Coordination ID: {{ coordination_id }}
              # Generated: {{ ansible_date_time.iso8601 }}
              
              [production_environment]
              {% for db in groups['database_primary'] + groups['database_replicas'] %}
              [[production_environment.{{ db }}]]
              hostname = "{{ db }}"
              role = "{{ hostvars[db]['db_role'] }}"
              port = {{ hostvars[db]['db_port'] }}
              environment = "{{ hostvars[db]['app_env'] }}"
              ip_address = "{{ hostvars[db]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
              
              {% endfor %}
              
              [staging_environment]
              {% for db in groups['database_staging'] %}
              [[staging_environment.{{ db }}]]
              hostname = "{{ db }}"
              role = "{{ hostvars[db]['db_role'] }}"
              port = {{ hostvars[db]['db_port'] }}
              environment = "{{ hostvars[db]['app_env'] }}"
              ip_address = "{{ hostvars[db]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
              
              {% endfor %}
              
              [coordination_summary]
              total_databases = {{ groups['database_servers'] | length }}
              production_databases = {{ (groups['database_primary'] + groups['database_replicas']) | length }}
              staging_databases = {{ groups['database_staging'] | length }}
              primary_databases = {{ groups['database_primary'] | length }}
              replica_databases = {{ groups['database_replicas'] | length }}
              
              [backup_coordination]
              primary_backups_created = {{ groups['database_primary'] | length }}
              replica_sync_operations = {{ groups['database_replicas'] | length }}
              cross_env_sync_required = {{ 'true' if groups['database_staging'] | length > 0 else 'false' }}
              
              [monitoring_endpoints]
              {% for db in groups['database_servers'] %}
              {{ db }} = "http://{{ hostvars[db]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}:{{ hostvars[db]['db_port'] }}/metrics"
              {% endfor %}
            dest: "/tmp/db-coordination/coordination-report-{{ coordination_id }}.toml"
            mode: '0644'
          delegate_to: "{{ groups['orchestration_controllers'][0] }}"
          run_once: true
        
        # Application server database configuration update
        - name: Update application servers with database configuration
          copy:
            content: |
              # Database Configuration for Application Servers
              # Coordination ID: {{ coordination_id }}
              # Updated: {{ ansible_date_time.iso8601 }}
              
              [database_connections]
              {% if app_env == 'production' %}
              
              [database_connections.primary]
              {% set primary_db = groups['database_primary'][0] %}
              host = "{{ hostvars[primary_db]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
              port = {{ hostvars[primary_db]['db_port'] }}
              database = "app_production"
              role = "primary"
              max_connections = 20
              
              {% for replica in groups['database_replicas'] %}
              [database_connections.replica_{{ loop.index }}]
              host = "{{ hostvars[replica]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
              port = {{ hostvars[replica]['db_port'] }}
              database = "app_production"
              role = "replica"
              max_connections = 10
              
              {% endfor %}
              
              {% else %}
              
              [database_connections.staging]
              {% set staging_db = groups['database_staging'][0] %}
              host = "{{ hostvars[staging_db]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}"
              port = {{ hostvars[staging_db]['db_port'] }}
              database = "app_staging"
              role = "primary"
              max_connections = 5
              
              {% endif %}
              
              [connection_pool]
              initial_size = 5
              max_size = {{ 20 if app_env == 'production' else 10 }}
              timeout_seconds = 30
              retry_attempts = 3
              
              [health_checks]
              enabled = true
              interval_seconds = 30
              timeout_seconds = 5
              
              [coordination_info]
              coordination_id = "{{ coordination_id }}"
              last_updated = "{{ ansible_date_time.iso8601 }}"
            dest: "/tmp/app-{{ item }}/database-config.toml"
            mode: '0644'
          delegate_to: "{{ item }}"
          loop: "{{ groups['web_servers'] + groups['api_servers'] }}"
          when: hostvars[item]['app_env'] == app_env
EOF

# Run database coordination
ansible-playbook -i inventory.ini db-coordination.yml
```

## Exercise 3: Multi-Environment Orchestration (5 minutes)

### Task: Implement complex multi-environment orchestration with cross-environment dependencies

Create multi-environment orchestration playbook:

```yaml
# Create multi-env-orchestration.yml
cat > multi-env-orchestration.yml << 'EOF'
---
- name: Multi-Environment Orchestration
  hosts: localhost
  gather_facts: no
  connection: local
  
  vars:
    orchestration_id: "{{ ansible_date_time.epoch }}"
    environments: ['production', 'staging']
    
  tasks:
    - name: Create orchestration control center
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /tmp/multi-env-orchestration/environments
        - /tmp/multi-env-orchestration/dependencies
        - /tmp/multi-env-orchestration/workflows
        - /tmp/multi-env-orchestration/monitoring
    
    # Environment-specific orchestration
    - name: Orchestrate each environment
      include_tasks: environment-tasks.yml
      loop: "{{ environments }}"
      loop_control:
        loop_var: target_environment
    
    # Cross-environment dependency management
    - name: Create cross-environment dependency map
      copy:
        content: |
          # Cross-Environment Dependencies
          # Orchestration ID: {{ orchestration_id }}
          # Generated: {{ ansible_date_time.iso8601 }}
          
          [orchestration_info]
          orchestration_id = "{{ orchestration_id }}"
          environments = {{ environments | to_json }}
          total_hosts = {{ groups['all'] | length }}
          
          {% for env in environments %}
          [{{ env }}_environment]
          
          [{{ env }}_environment.web_servers]
          {% for host in groups['web_servers_' + env] if 'web_servers_' + env in groups %}
          {{ host }} = {
            port = {{ hostvars[host]['web_port'] }},
            tier = "{{ hostvars[host]['tier'] }}",
            dependencies = ["database", "cache", "load_balancer"]
          }
          {% endfor %}
          
          [{{ env }}_environment.api_servers]
          {% for host in groups['api_servers_' + env] if 'api_servers_' + env in groups %}
          {{ host }} = {
            port = {{ hostvars[host]['api_port'] }},
            tier = "{{ hostvars[host]['tier'] }}",
            dependencies = ["database", "cache"]
          }
          {% endfor %}
          
          [{{ env }}_environment.databases]
          {% if env == 'production' %}
          {% for host in groups['database_primary'] + groups['database_replicas'] %}
          {{ host }} = {
            port = {{ hostvars[host]['db_port'] }},
            role = "{{ hostvars[host]['db_role'] }}",
            dependencies = []
          }
          {% endfor %}
          {% else %}
          {% for host in groups['database_staging'] %}
          {{ host }} = {
            port = {{ hostvars[host]['db_port'] }},
            role = "{{ hostvars[host]['db_role'] }}",
            dependencies = []
          }
          {% endfor %}
          {% endif %}
          
          [{{ env }}_environment.load_balancers]
          {% for host in groups['load_balancers_' + env] if 'load_balancers_' + env in groups %}
          {{ host }} = {
            port = {{ hostvars[host]['lb_port'] }},
            type = "{{ hostvars[host]['lb_type'] }}",
            dependencies = ["web_servers", "api_servers"]
          }
          {% endfor %}
          
          {% endfor %}
          
          [cross_environment_dependencies]
          staging_to_production_data_sync = true
          production_backup_to_staging = false
          shared_monitoring = true
          shared_cache = false
          
          [deployment_order]
          # Recommended deployment order for zero-downtime deployments
          order = [
            "database_staging",
            "api_servers_staging", 
            "web_servers_staging",
            "load_balancers_staging",
            "database_replicas",
            "api_servers_prod",
            "web_servers_prod", 
            "load_balancers_prod",
            "database_primary"
          ]
        dest: "/tmp/multi-env-orchestration/dependencies/cross-env-map-{{ orchestration_id }}.toml"
        mode: '0644'
    
    # Global monitoring configuration
    - name: Configure global monitoring
      copy:
        content: |
          # Global Monitoring Configuration
          # Orchestration ID: {{ orchestration_id }}
          # Generated: {{ ansible_date_time.iso8601 }}
          
          [monitoring_targets]
          
          {% for env in environments %}
          [monitoring_targets.{{ env }}]
          
          {% if 'web_servers_' + env in groups %}
          web_servers = [
          {% for host in groups['web_servers_' + env] %}
            {
              target = "{{ hostvars[host]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}:{{ hostvars[host]['web_port'] }}",
              labels = {
                job = "web-server",
                environment = "{{ env }}",
                instance = "{{ host }}",
                tier = "{{ hostvars[host]['tier'] }}"
              }
            }{{ ',' if not loop.last else '' }}
          {% endfor %}
          ]
          {% endif %}
          
          {% if 'api_servers_' + env in groups %}
          api_servers = [
          {% for host in groups['api_servers_' + env] %}
            {
              target = "{{ hostvars[host]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}:{{ hostvars[host]['api_port'] }}",
              labels = {
                job = "api-server",
                environment = "{{ env }}",
                instance = "{{ host }}",
                tier = "{{ hostvars[host]['tier'] }}"
              }
            }{{ ',' if not loop.last else '' }}
          {% endfor %}
          ]
          {% endif %}
          
          {% if env == 'production' %}
          databases = [
          {% for host in groups['database_primary'] + groups['database_replicas'] %}
            {
              target = "{{ hostvars[host]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}:{{ hostvars[host]['db_port'] }}",
              labels = {
                job = "database",
                environment = "{{ env }}",
                instance = "{{ host }}",
                role = "{{ hostvars[host]['db_role'] }}"
              }
            }{{ ',' if not loop.last else '' }}
          {% endfor %}
          ]
          {% else %}
          databases = [
          {% for host in groups['database_staging'] %}
            {
              target = "{{ hostvars[host]['ansible_default_ipv4']['address'] | default('127.0.0.1') }}:{{ hostvars[host]['db_port'] }}",
              labels = {
                job = "database",
                environment = "{{ env }}",
                instance = "{{ host }}",
                role = "{{ hostvars[host]['db_role'] }}"
              }
            }{{ ',' if not loop.last else '' }}
          {% endfor %}
          ]
          {% endif %}
          
          {% endfor %}
          
          [alerting_rules]
          
          [alerting_rules.high_response_time]
          condition = "http_request_duration_seconds > 1.0"
          severity = "warning"
          environments = {{ environments | to_json }}
          
          [alerting_rules.database_connection_failure]
          condition = "database_connections_active == 0"
          severity = "critical"
          environments = {{ environments | to_json }}
          
          [alerting_rules.load_balancer_backend_down]
          condition = "load_balancer_backend_up == 0"
          severity = "critical"
          environments = ["production"]
          
          [dashboards]
          multi_environment_overview = "/tmp/multi-env-orchestration/monitoring/overview-dashboard.json"
          environment_comparison = "/tmp/multi-env-orchestration/monitoring/comparison-dashboard.json"
        dest: "/tmp/multi-env-orchestration/monitoring/global-config-{{ orchestration_id }}.toml"
        mode: '0644'
      delegate_to: "{{ groups['monitoring_stack'][0] }}"
    
    # Create orchestration workflow
    - name: Create orchestration workflow definition
      copy:
        content: |
          # Multi-Environment Orchestration Workflow
          # Orchestration ID: {{ orchestration_id }}
          # Generated: {{ ansible_date_time.iso8601 }}
          
          [workflow_definition]
          name = "multi_environment_deployment"
          version = "2.0"
          orchestration_id = "{{ orchestration_id }}"
          
          [workflow_definition.metadata]
          created_by = "{{ ansible_user_id }}"
          created_at = "{{ ansible_date_time.iso8601 }}"
          environments = {{ environments | to_json }}
          
          # Stage 1: Preparation
          [[workflow_definition.stages]]
          name = "preparation"
          parallel = false
          
          [[workflow_definition.stages.tasks]]
          name = "validate_environments"
          type = "validation"
          targets = {{ environments | to_json }}
          
          [[workflow_definition.stages.tasks]]
          name = "backup_databases"
          type = "backup"
          targets = ["database_primary", "database_staging"]
          
          # Stage 2: Staging Deployment
          [[workflow_definition.stages]]
          name = "staging_deployment"
          parallel = false
          depends_on = ["preparation"]
          
          [[workflow_definition.stages.tasks]]
          name = "deploy_staging_databases"
          type = "deployment"
          targets = ["database_staging"]
          
          [[workflow_definition.stages.tasks]]
          name = "deploy_staging_apis"
          type = "deployment"
          targets = ["api_servers_staging"]
          depends_on = ["deploy_staging_databases"]
          
          [[workflow_definition.stages.tasks]]
          name = "deploy_staging_web"
          type = "deployment"
          targets = ["web_servers_staging"]
          depends_on = ["deploy_staging_apis"]
          
          [[workflow_definition.stages.tasks]]
          name = "update_staging_lb"
          type = "configuration"
          targets = ["load_balancers_staging"]
          depends_on = ["deploy_staging_web"]
          
          # Stage 3: Production Deployment
          [[workflow_definition.stages]]
          name = "production_deployment"
          parallel = false
          depends_on = ["staging_deployment"]
          
          [[workflow_definition.stages.tasks]]
          name = "deploy_production_replicas"
          type = "deployment"
          targets = ["database_replicas"]
          
          [[workflow_definition.stages.tasks]]
          name = "deploy_production_apis"
          type = "deployment"
          targets = ["api_servers_prod"]
          strategy = "rolling"
          
          [[workflow_definition.stages.tasks]]
          name = "deploy_production_web"
          type = "deployment"
          targets = ["web_servers_prod"]
          strategy = "rolling"
          depends_on = ["deploy_production_apis"]
          
          [[workflow_definition.stages.tasks]]
          name = "update_production_lb"
          type = "configuration"
          targets = ["load_balancers_prod"]
          depends_on = ["deploy_production_web"]
          
          # Stage 4: Validation
          [[workflow_definition.stages]]
          name = "validation"
          parallel = true
          depends_on = ["production_deployment"]
          
          [[workflow_definition.stages.tasks]]
          name = "health_check_all"
          type = "validation"
          targets = ["web_servers", "api_servers", "database_servers"]
          
          [[workflow_definition.stages.tasks]]
          name = "performance_test"
          type = "testing"
          targets = ["load_balancers_prod"]
          
          [rollback_strategy]
          enabled = true
          automatic_triggers = ["health_check_failure", "performance_degradation"]
          manual_approval_required = true
          
          [notification_settings]
          enabled = true
          channels = ["email", "slack"]
          events = ["stage_complete", "deployment_complete", "failure", "rollback"]
        dest: "/tmp/multi-env-orchestration/workflows/deployment-workflow-{{ orchestration_id }}.toml"
        mode: '0644'

# Create environment-specific tasks file
- name: Create environment tasks file
  copy:
    content: |
      ---
      # Environment-specific orchestration tasks
      # This file is included for each environment
      
      - name: "Create environment summary for {{ target_environment }}"
        copy:
          content: |
            # Environment Summary: {{ target_environment }}
            # Orchestration ID: {{ orchestration_id }}
            # Generated: {{ ansible_date_time.iso8601 }}
            
            [environment_info]
            name = "{{ target_environment }}"
            orchestration_id = "{{ orchestration_id }}"
            
            [services]
            {% if 'web_servers_' + target_environment in groups %}
            web_servers = {{ groups['web_servers_' + target_environment] | length }}
            {% else %}
            web_servers = 0
            {% endif %}
            
            {% if 'api_servers_' + target_environment in groups %}
            api_servers = {{ groups['api_servers_' + target_environment] | length }}
            {% else %}
            api_servers = 0
            {% endif %}
            
            {% if target_environment == 'production' %}
            databases = {{ (groups['database_primary'] + groups['database_replicas']) | length }}
            {% else %}
            databases = {{ groups['database_staging'] | length if 'database_staging' in groups else 0 }}
            {% endif %}
            
            {% if 'load_balancers_' + target_environment in groups %}
            load_balancers = {{ groups['load_balancers_' + target_environment] | length }}
            {% else %}
            load_balancers = 0
            {% endif %}
            
            [orchestration_status]
            status = "configured"
            last_updated = "{{ ansible_date_time.iso8601 }}"
          dest: "/tmp/multi-env-orchestration/environments/{{ target_environment }}-summary.toml"
          mode: '0644'
    dest: environment-tasks.yml
    mode: '0644'
EOF

# Run multi-environment orchestration
ansible-playbook -i inventory.ini multi-env-orchestration.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Check orchestration results
echo "=== Advanced Delegation Results ==="

# Check load balancer orchestration
echo "Load Balancer Orchestration:"
find /tmp/lb-orchestration -name "*.log" -o -name "*.toml" 2>/dev/null | head -3 | while read file; do
    echo "File: $file"
    head -10 "$file"
    echo "---"
done

# Check database coordination
echo "Database Coordination:"
find /tmp/db-coordination -name "*.toml" -o -name "*.txt" 2>/dev/null | head -3 | while read file; do
    echo "File: $file"
    head -15 "$file"
    echo "---"
done

# Check multi-environment orchestration
echo "Multi-Environment Orchestration:"
find /tmp/multi-env-orchestration -name "*.toml" 2>/dev/null | head -2 | while read file; do
    echo "File: $file"
    head -20 "$file"
    echo "---"
done

# Check application configurations
echo "Application Database Configurations:"
find /tmp/app-* -name "database-config.toml" 2>/dev/null | head -2 | while read file; do
    echo "Config: $file"
    head -10 "$file"
    echo "---"
done
```

### 2. Discussion Points
- How do you design delegation patterns for zero-downtime deployments?
- What are the best practices for cross-environment coordination?
- How do you handle delegation failures in complex orchestration workflows?
- What monitoring and observability strategies work best with delegation?

### 3. Clean Up
```bash
# Clean up test artifacts
# rm -rf /tmp/lb-orchestration /tmp/db-coordination /tmp/multi-env-orchestration /tmp/app-* /tmp/db-* /tmp/lb-*
```

## Key Takeaways
- Advanced delegation enables complex orchestration patterns
- Load balancer management requires careful coordination of draining and activation
- Database coordination involves primary/replica management and cross-environment sync
- Multi-environment orchestration requires dependency mapping and workflow definition
- Proper error handling and rollback strategies are essential for production deployments
- Monitoring and observability must be built into delegation workflows

## Module 8 Summary

### What We Covered
1. **Delegation Fundamentals** - Basic delegation concepts, facts gathering, and common patterns
2. **Advanced Delegation Patterns** - Complex orchestration, load balancer management, database coordination, and multi-environment workflows

### Key Skills Developed
- Implementing task delegation for complex orchestration workflows
- Managing load balancer configurations with zero-downtime deployments
- Coordinating database operations across primary and replica systems
- Designing multi-environment orchestration with cross-environment dependencies
- Creating monitoring and observability for delegation-based workflows

### Next Steps
You now have comprehensive knowledge of Ansible delegation for enterprise-scale orchestration. These skills enable you to design and implement complex automation workflows that coordinate multiple services, environments, and infrastructure components with proper error handling and monitoring.

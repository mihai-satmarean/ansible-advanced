# Lab 2.2: Block, Rescue, and Always Patterns

## Objective
Implement complex error handling workflows using block, rescue, and always patterns for enterprise-grade automation.

## Duration
30 minutes

## Prerequisites
- Completed Lab 2.1
- Understanding of basic error handling concepts

## Lab Setup

```bash
cd ~/ansible-labs/module-02
mkdir -p lab-2.2
cd lab-2.2

cat > inventory.ini << EOF
[servers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Structured Error Handling Workflows (12 minutes)

### Task: Implement comprehensive error handling workflows

Create `structured-workflows.yml`:
```yaml
---
- name: Structured error handling workflows
  hosts: localhost
  become: yes
  vars:
    application:
      name: "enterprise-app"
      version: "2.1.0"
      environment: "production"
      critical: true
    
    deployment_steps:
      - name: "prerequisites"
        critical: true
      - name: "application"
        critical: true
      - name: "configuration"
        critical: false
      - name: "monitoring"
        critical: false
        
  tasks:
    # Multi-step deployment with structured error handling
    - name: "Step 1: Prerequisites Installation"
      block:
        - name: Update package cache
          apt:
            update_cache: yes
            cache_valid_time: 3600
          when: ansible_os_family == "Debian"
        
        - name: Install system dependencies
          package:
            name:
              - curl
              - wget
              - unzip
              - python3-pip
            state: present
        
        - name: Create application user
          user:
            name: "{{ application.name }}"
            system: yes
            shell: /bin/false
            home: "/opt/{{ application.name }}"
            create_home: yes
        
        - name: Create application directories
          file:
            path: "{{ item }}"
            state: directory
            owner: "{{ application.name }}"
            group: "{{ application.name }}"
            mode: '0755'
          loop:
            - "/opt/{{ application.name }}/bin"
            - "/opt/{{ application.name }}/config"
            - "/opt/{{ application.name }}/logs"
            - "/var/log/{{ application.name }}"
        
        - name: Set prerequisites completion flag
          set_fact:
            prerequisites_completed: true
      
      rescue:
        - name: Prerequisites installation failed
          debug:
            msg: "CRITICAL: Prerequisites installation failed - aborting deployment"
        
        - name: Log prerequisites failure
          lineinfile:
            path: "/var/log/{{ application.name }}-deployment.log"
            line: "{{ ansible_date_time.iso8601 }}: FAILED - Prerequisites installation"
            create: yes
            mode: '0644'
        
        - name: Clean up partial prerequisites
          user:
            name: "{{ application.name }}"
            state: absent
            remove: yes
          failed_when: false
        
        - name: Fail deployment due to critical error
          fail:
            msg: "Prerequisites installation failed - deployment cannot continue"
      
      always:
        - name: Log prerequisites attempt
          lineinfile:
            path: "/var/log/{{ application.name }}-deployment.log"
            line: "{{ ansible_date_time.iso8601 }}: Prerequisites step {{ 'COMPLETED' if prerequisites_completed | default(false) else 'FAILED' }}"
            create: yes
            mode: '0644'
      
      tags: prerequisites
    
    # Application deployment (depends on prerequisites)
    - name: "Step 2: Application Deployment"
      block:
        - name: Download application package (simulate)
          copy:
            content: |
              #!/bin/bash
              # {{ application.name }} v{{ application.version }}
              # Enterprise Application Simulator
              
              echo "Starting {{ application.name }} v{{ application.version }}"
              echo "Environment: {{ application.environment }}"
              echo "PID: $$"
              
              # Health check endpoint simulation
              while true; do
                echo "$(date): {{ application.name }} heartbeat"
                sleep 30
              done
            dest: "/opt/{{ application.name }}/bin/{{ application.name }}"
            owner: "{{ application.name }}"
            group: "{{ application.name }}"
            mode: '0755'
        
        - name: Create application configuration
          copy:
            content: |
              # {{ application.name }} Configuration
              [application]
              name={{ application.name }}
              version={{ application.version }}
              environment={{ application.environment }}
              
              [server]
              host=0.0.0.0
              port=8080
              
              [database]
              host=localhost
              port=5432
              name={{ application.name }}_db
              
              [logging]
              level=INFO
              file=/var/log/{{ application.name }}/app.log
              
              [deployment]
              deployed_by={{ ansible_user_id }}
              deployed_at={{ ansible_date_time.iso8601 }}
            dest: "/opt/{{ application.name }}/config/app.conf"
            owner: "{{ application.name }}"
            group: "{{ application.name }}"
            mode: '0644'
        
        - name: Create systemd service file
          copy:
            content: |
              [Unit]
              Description={{ application.name | title }} Service
              After=network.target
              
              [Service]
              Type=simple
              User={{ application.name }}
              Group={{ application.name }}
              WorkingDirectory=/opt/{{ application.name }}
              ExecStart=/opt/{{ application.name }}/bin/{{ application.name }}
              Restart=always
              RestartSec=10
              
              [Install]
              WantedBy=multi-user.target
            dest: "/etc/systemd/system/{{ application.name }}.service"
            mode: '0644'
          notify: reload systemd
        
        - name: Set application deployment flag
          set_fact:
            application_deployed: true
      
      rescue:
        - name: Application deployment failed
          debug:
            msg: "CRITICAL: Application deployment failed - attempting recovery"
        
        - name: Create minimal application fallback
          copy:
            content: |
              #!/bin/bash
              # {{ application.name }} Minimal Fallback
              echo "Minimal {{ application.name }} service running"
              echo "This is a fallback version due to deployment failure"
              sleep infinity
            dest: "/opt/{{ application.name }}/bin/{{ application.name }}-minimal"
            owner: "{{ application.name }}"
            group: "{{ application.name }}"
            mode: '0755'
        
        - name: Update service file for minimal version
          copy:
            content: |
              [Unit]
              Description={{ application.name | title }} Service (Minimal)
              After=network.target
              
              [Service]
              Type=simple
              User={{ application.name }}
              Group={{ application.name }}
              ExecStart=/opt/{{ application.name }}/bin/{{ application.name }}-minimal
              Restart=always
              
              [Install]
              WantedBy=multi-user.target
            dest: "/etc/systemd/system/{{ application.name }}.service"
            mode: '0644'
          notify: reload systemd
        
        - name: Set recovery flag
          set_fact:
            application_recovered: true
        
        - name: Log application recovery
          lineinfile:
            path: "/var/log/{{ application.name }}-deployment.log"
            line: "{{ ansible_date_time.iso8601 }}: Application deployment failed - minimal version deployed"
            create: yes
            mode: '0644'
      
      always:
        - name: Reload systemd daemon
          systemd:
            daemon_reload: yes
        
        - name: Log application deployment attempt
          lineinfile:
            path: "/var/log/{{ application.name }}-deployment.log"
            line: "{{ ansible_date_time.iso8601 }}: Application step {{ 'COMPLETED' if application_deployed | default(false) else 'RECOVERED' if application_recovered | default(false) else 'FAILED' }}"
            create: yes
            mode: '0644'
      
      when: prerequisites_completed | default(false)
      tags: application
    
    # Optional configuration (non-critical)
    - name: "Step 3: Advanced Configuration"
      block:
        - name: Configure monitoring endpoints
          copy:
            content: |
              # Monitoring Configuration
              [endpoints]
              health=/health
              metrics=/metrics
              status=/status
              
              [alerts]
              email=ops@company.com
              slack_webhook=https://hooks.slack.com/services/...
              
              [thresholds]
              cpu_warning=70
              cpu_critical=90
              memory_warning=80
              memory_critical=95
            dest: "/opt/{{ application.name }}/config/monitoring.conf"
            owner: "{{ application.name }}"
            group: "{{ application.name }}"
            mode: '0644'
        
        - name: Configure log rotation
          copy:
            content: |
              /var/log/{{ application.name }}/*.log {
                daily
                rotate 30
                compress
                delaycompress
                missingok
                notifempty
                create 0644 {{ application.name }} {{ application.name }}
              }
            dest: "/etc/logrotate.d/{{ application.name }}"
            mode: '0644'
        
        - name: Set advanced configuration flag
          set_fact:
            advanced_config_completed: true
      
      rescue:
        - name: Advanced configuration failed - using defaults
          debug:
            msg: "Advanced configuration failed, application will use default settings"
        
        - name: Create basic monitoring configuration
          copy:
            content: |
              # Basic Monitoring Configuration
              [endpoints]
              health=/health
              
              [alerts]
              # Advanced alerting disabled due to configuration failure
              enabled=false
            dest: "/opt/{{ application.name }}/config/monitoring.conf"
            owner: "{{ application.name }}"
            group: "{{ application.name }}"
            mode: '0644'
        
        - name: Set basic configuration flag
          set_fact:
            basic_config_applied: true
      
      always:
        - name: Log configuration attempt
          lineinfile:
            path: "/var/log/{{ application.name }}-deployment.log"
            line: "{{ ansible_date_time.iso8601 }}: Configuration step {{ 'ADVANCED' if advanced_config_completed | default(false) else 'BASIC' if basic_config_applied | default(false) else 'SKIPPED' }}"
            create: yes
            mode: '0644'
      
      when: 
        - prerequisites_completed | default(false)
        - application_deployed | default(false) or application_recovered | default(false)
      tags: configuration
  
  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes
```

### Run structured workflows:
```bash
# Run complete structured workflow
ansible-playbook -i inventory.ini structured-workflows.yml

# Run specific steps
ansible-playbook -i inventory.ini structured-workflows.yml --tags "prerequisites,application"

# Check deployment log
cat /var/log/enterprise-app-deployment.log
```

## Exercise 2: Transaction-like Operations (10 minutes)

### Task: Implement transaction-like operations with rollback capability

Create `transaction-operations.yml`:
```yaml
---
- name: Transaction-like operations with rollback
  hosts: localhost
  become: yes
  vars:
    database_config:
      name: "transaction-db"
      version: "1.0"
      backup_retention: 7
      
  tasks:
    # Database migration with rollback capability
    - name: Database Migration Transaction
      block:
        # Phase 1: Backup current state
        - name: Create backup directory
          file:
            path: "/var/backups/{{ database_config.name }}"
            state: directory
            mode: '0750'
        
        - name: Backup current database state
          shell: |
            # Simulate database backup
            echo "Database backup created at $(date)" > /var/backups/{{ database_config.name }}/backup-{{ ansible_date_time.epoch }}.sql
            echo "-- Backup of {{ database_config.name }}" >> /var/backups/{{ database_config.name }}/backup-{{ ansible_date_time.epoch }}.sql
            echo "-- Version: current" >> /var/backups/{{ database_config.name }}/backup-{{ ansible_date_time.epoch }}.sql
          register: backup_creation
        
        # Phase 2: Apply migration
        - name: Apply database migration
          copy:
            content: |
              -- Database Migration for {{ database_config.name }}
              -- Version: {{ database_config.version }}
              -- Applied: {{ ansible_date_time.iso8601 }}
              
              ALTER TABLE users ADD COLUMN email VARCHAR(255);
              ALTER TABLE users ADD COLUMN created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP;
              
              CREATE INDEX idx_users_email ON users(email);
              
              -- This might fail on some systems
              {% if ansible_hostname == 'nonexistent-host' %}
              INVALID SQL STATEMENT TO FORCE FAILURE;
              {% endif %}
            dest: "/tmp/migration-{{ database_config.version }}.sql"
            mode: '0644'
        
        - name: Execute migration (simulate)
          shell: |
            # Simulate migration execution
            echo "Executing migration from /tmp/migration-{{ database_config.version }}.sql"
            
            # Simulate potential failure
            if grep -q "INVALID SQL" /tmp/migration-{{ database_config.version }}.sql; then
              echo "Migration failed: Invalid SQL statement"
              exit 1
            fi
            
            echo "Migration completed successfully"
          register: migration_result
        
        - name: Update database version
          copy:
            content: |
              # Database Version Information
              name={{ database_config.name }}
              version={{ database_config.version }}
              migrated_at={{ ansible_date_time.iso8601 }}
              status=active
            dest: "/var/lib/{{ database_config.name }}/version.info"
            mode: '0644'
        
        - name: Set migration success flag
          set_fact:
            migration_successful: true
      
      rescue:
        - name: Migration failed - initiating rollback
          debug:
            msg: "Database migration failed, initiating rollback procedure"
        
        - name: Find latest backup
          find:
            paths: "/var/backups/{{ database_config.name }}"
            patterns: "backup-*.sql"
          register: backup_files
        
        - name: Restore from backup
          shell: |
            if [ {{ backup_files.files | length }} -gt 0 ]; then
              latest_backup=$(ls -t /var/backups/{{ database_config.name }}/backup-*.sql | head -1)
              echo "Restoring from backup: $latest_backup"
              echo "Database restored from backup at $(date)"
            else
              echo "No backup found for rollback"
              exit 1
            fi
          register: rollback_result
        
        - name: Reset database version to previous
          copy:
            content: |
              # Database Version Information (Rolled Back)
              name={{ database_config.name }}
              version=previous
              rolled_back_at={{ ansible_date_time.iso8601 }}
              status=rolled_back
              original_migration_version={{ database_config.version }}
            dest: "/var/lib/{{ database_config.name }}/version.info"
            mode: '0644'
        
        - name: Log rollback completion
          lineinfile:
            path: "/var/log/{{ database_config.name }}-migrations.log"
            line: "{{ ansible_date_time.iso8601 }}: ROLLBACK - Migration {{ database_config.version }} failed and was rolled back"
            create: yes
            mode: '0644'
        
        - name: Set rollback flag
          set_fact:
            migration_rolled_back: true
      
      always:
        - name: Clean up temporary migration files
          file:
            path: "/tmp/migration-{{ database_config.version }}.sql"
            state: absent
        
        - name: Log migration attempt
          lineinfile:
            path: "/var/log/{{ database_config.name }}-migrations.log"
            line: "{{ ansible_date_time.iso8601 }}: {{ 'SUCCESS' if migration_successful | default(false) else 'ROLLBACK' if migration_rolled_back | default(false) else 'FAILED' }} - Migration {{ database_config.version }}"
            create: yes
            mode: '0644'
        
        - name: Clean up old backups
          shell: |
            find /var/backups/{{ database_config.name }} -name "backup-*.sql" -mtime +{{ database_config.backup_retention }} -delete
          failed_when: false
        
        - name: Display final migration status
          debug:
            msg: |
              Migration Status: {{ 'SUCCESS' if migration_successful | default(false) else 'ROLLED BACK' if migration_rolled_back | default(false) else 'FAILED' }}
              Database: {{ database_config.name }}
              Target Version: {{ database_config.version }}
              Final State: {{ 'MIGRATED' if migration_successful | default(false) else 'PREVIOUS VERSION' }}
```

### Run transaction operations:
```bash
# Run transaction operations
ansible-playbook -i inventory.ini transaction-operations.yml

# Check results
cat /var/lib/transaction-db/version.info
cat /var/log/transaction-db-migrations.log
ls -la /var/backups/transaction-db/
```

## Exercise 3: Nested Error Handling (8 minutes)

### Task: Implement nested error handling for complex scenarios

Create `nested-error-handling.yml`:
```yaml
---
- name: Nested error handling patterns
  hosts: localhost
  become: yes
  vars:
    microservices:
      - name: "auth-service"
        port: 8081
        dependencies: ["database"]
        critical: true
      - name: "user-service"
        port: 8082
        dependencies: ["auth-service", "database"]
        critical: true
      - name: "notification-service"
        port: 8083
        dependencies: ["user-service"]
        critical: false
        
  tasks:
    # Nested error handling for microservices deployment
    - name: Deploy microservices with nested error handling
      block:
        # Outer block: Overall deployment
        - name: "Deploy {{ item.name }}"
          block:
            # Inner block: Service-specific deployment
            - name: "Create {{ item.name }} configuration"
              block:
                - name: Create service directory
                  file:
                    path: "/opt/{{ item.name }}"
                    state: directory
                    mode: '0755'
                
                - name: Generate service configuration
                  copy:
                    content: |
                      # {{ item.name }} Configuration
                      [service]
                      name={{ item.name }}
                      port={{ item.port }}
                      critical={{ item.critical | lower }}
                      
                      [dependencies]
                      {% for dep in item.dependencies %}
                      requires={{ dep }}
                      {% endfor %}
                      
                      [deployment]
                      deployed_at={{ ansible_date_time.iso8601 }}
                    dest: "/opt/{{ item.name }}/config.yml"
                    mode: '0644'
                
                - name: Create service startup script
                  copy:
                    content: |
                      #!/bin/bash
                      # {{ item.name }} startup script
                      
                      echo "Starting {{ item.name }} on port {{ item.port }}"
                      
                      # Check dependencies
                      {% for dep in item.dependencies %}
                      if ! systemctl is-active {{ dep }} >/dev/null 2>&1; then
                        echo "Dependency {{ dep }} is not running"
                        {% if item.critical %}
                        exit 1
                        {% else %}
                        echo "Warning: {{ dep }} not available, continuing anyway"
                        {% endif %}
                      fi
                      {% endfor %}
                      
                      # Simulate service
                      while true; do
                        echo "{{ item.name }} heartbeat $(date)"
                        sleep 60
                      done
                    dest: "/opt/{{ item.name }}/start.sh"
                    mode: '0755'
              
              rescue:
                - name: "{{ item.name }} configuration failed"
                  debug:
                    msg: "Configuration failed for {{ item.name }}, creating minimal config"
                
                - name: Create minimal configuration
                  copy:
                    content: |
                      # {{ item.name }} Minimal Configuration
                      [service]
                      name={{ item.name }}
                      port={{ item.port }}
                      mode=minimal
                      
                      [deployment]
                      deployed_at={{ ansible_date_time.iso8601 }}
                      status=minimal_config
                    dest: "/opt/{{ item.name }}/config.yml"
                    mode: '0644'
                
                - name: Set minimal config flag
                  set_fact:
                    "{{ item.name | replace('-', '_') }}_minimal": true
            
            # Service registration and startup
            - name: "Register {{ item.name }} service"
              block:
                - name: Create systemd service file
                  copy:
                    content: |
                      [Unit]
                      Description={{ item.name | title }}
                      After=network.target{% for dep in item.dependencies %} {{ dep }}.service{% endfor %}
                      
                      {% for dep in item.dependencies %}
                      {% if item.critical %}
                      Requires={{ dep }}.service
                      {% else %}
                      Wants={{ dep }}.service
                      {% endif %}
                      {% endfor %}
                      
                      [Service]
                      Type=simple
                      ExecStart=/opt/{{ item.name }}/start.sh
                      Restart=always
                      RestartSec=10
                      
                      [Install]
                      WantedBy=multi-user.target
                    dest: "/etc/systemd/system/{{ item.name }}.service"
                    mode: '0644'
                  notify: reload systemd
                
                - name: Enable and start service
                  systemd:
                    name: "{{ item.name }}"
                    enabled: yes
                    state: started
                    daemon_reload: yes
              
              rescue:
                - name: "{{ item.name }} service registration failed"
                  debug:
                    msg: "Service registration failed for {{ item.name }}"
                
                - name: Create service registration failure marker
                  copy:
                    content: |
                      Service: {{ item.name }}
                      Status: registration_failed
                      Time: {{ ansible_date_time.iso8601 }}
                      Critical: {{ item.critical }}
                    dest: "/tmp/{{ item.name }}-failure.marker"
                    mode: '0644'
                
                - name: Fail if critical service
                  fail:
                    msg: "Critical service {{ item.name }} failed to register"
                  when: item.critical
          
          rescue:
            - name: "{{ item.name }} deployment completely failed"
              debug:
                msg: "Complete deployment failure for {{ item.name }}"
            
            - name: Log deployment failure
              lineinfile:
                path: "/var/log/microservices-deployment.log"
                line: "{{ ansible_date_time.iso8601 }}: FAILED - {{ item.name }} deployment failed completely"
                create: yes
                mode: '0644'
            
            - name: Stop deployment if critical service failed
              fail:
                msg: "Critical service {{ item.name }} failed - stopping deployment"
              when: item.critical
          
          always:
            - name: Log deployment attempt for {{ item.name }}
              lineinfile:
                path: "/var/log/microservices-deployment.log"
                line: "{{ ansible_date_time.iso8601 }}: {{ item.name }} deployment attempt completed"
                create: yes
                mode: '0644'
        
        loop: "{{ microservices }}"
      
      rescue:
        - name: Overall deployment failed
          debug:
            msg: "Overall microservices deployment failed"
        
        - name: Generate failure report
          copy:
            content: |
              Microservices Deployment Failure Report
              =====================================
              
              Deployment Time: {{ ansible_date_time.iso8601 }}
              
              Services Status:
              {% for service in microservices %}
              - {{ service.name }}: {{ 'FAILED' }}
              {% endfor %}
              
              Check individual service logs for details.
            dest: "/tmp/deployment-failure-report.txt"
            mode: '0644'
      
      always:
        - name: Generate deployment summary
          copy:
            content: |
              Microservices Deployment Summary
              ==============================
              
              Deployment Time: {{ ansible_date_time.iso8601 }}
              
              Services:
              {% for service in microservices %}
              - {{ service.name }} (Port {{ service.port }}, Critical: {{ service.critical }})
              {% endfor %}
              
              Check /var/log/microservices-deployment.log for detailed logs.
            dest: "/tmp/deployment-summary.txt"
            mode: '0644'
  
  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes
```

### Run nested error handling:
```bash
# Run nested error handling
ansible-playbook -i inventory.ini nested-error-handling.yml

# Check results
cat /var/log/microservices-deployment.log
cat /tmp/deployment-summary.txt
ls -la /opt/*/config.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Check structured workflows results
ls -la /opt/enterprise-app/
cat /var/log/enterprise-app-deployment.log
systemctl status enterprise-app --no-pager || echo "Service not running"

# Check transaction operations results
cat /var/lib/transaction-db/version.info
ls -la /var/backups/transaction-db/

# Check nested error handling results
cat /tmp/deployment-summary.txt
ls -la /opt/*/
```

### 2. Discussion Points
- How do you implement transaction-like operations in your current automation?
- What strategies do you use for nested error handling in complex deployments?
- How do you balance error recovery with deployment speed?

### 3. Clean Up
```bash
# Stop and remove services
sudo systemctl stop enterprise-app auth-service user-service notification-service 2>/dev/null || true
sudo systemctl disable enterprise-app auth-service user-service notification-service 2>/dev/null || true

# Remove files
sudo rm -rf /opt/enterprise-app /opt/auth-service /opt/user-service /opt/notification-service
sudo rm -rf /var/lib/transaction-db /var/backups/transaction-db
sudo rm -f /etc/systemd/system/{enterprise-app,auth-service,user-service,notification-service}.service
sudo rm -f /var/log/*-deployment.log /var/log/*-migrations.log
sudo rm -f /tmp/deployment-*.txt /tmp/*-failure.marker
sudo systemctl daemon-reload
```

## Key Takeaways
- Block/rescue/always provides structured error handling for complex workflows
- Nested error handling enables granular control over failure scenarios
- Transaction-like operations require careful state management and rollback planning
- Always blocks ensure cleanup and logging regardless of success or failure
- Critical vs non-critical service handling enables flexible deployment strategies
- Proper logging and status tracking are essential for troubleshooting

## Next Steps
Proceed to Lab 2.3: Conditional Execution and Flow Control

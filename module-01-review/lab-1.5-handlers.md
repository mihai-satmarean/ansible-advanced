# Lab 1.5: Handlers and Notifications

## Objective
Master event-driven automation using handlers, notifications, and advanced handler patterns for efficient service management.

## Duration
25 minutes

## Prerequisites
- Completed previous labs
- Understanding of service management concepts

## Lab Setup

```bash
cd ~/ansible-labs/module-01
mkdir -p lab-1.5
cd lab-1.5

cat > inventory.ini << EOF
[servers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Basic Handler Implementation (10 minutes)

### Task: Implement basic handlers for service management

Create `basic-handlers.yml`:
```yaml
---
- name: Basic handler demonstration
  hosts: localhost
  become: yes
  vars:
    web_service_name: "demo-web"
    web_port: 8080
    
  tasks:
    # Configuration changes that trigger handlers
    - name: Create web service script
      copy:
        content: |
          #!/bin/bash
          # {{ web_service_name }} Service
          echo "Starting {{ web_service_name }} on port {{ web_port }}"
          
          # Simple HTTP server simulation
          while true; do
            echo -e "HTTP/1.1 200 OK\n\n<h1>{{ web_service_name }}</h1><p>Running on port {{ web_port }}</p>" | nc -l -p {{ web_port }} -q 1
          done
        dest: "/opt/{{ web_service_name }}.sh"
        mode: '0755'
      notify:
        - restart web service
        - validate web service
    
    - name: Create systemd service file
      copy:
        content: |
          [Unit]
          Description={{ web_service_name | title }} Service
          After=network.target
          
          [Service]
          Type=simple
          ExecStart=/opt/{{ web_service_name }}.sh
          Restart=always
          RestartSec=5
          
          [Install]
          WantedBy=multi-user.target
        dest: "/etc/systemd/system/{{ web_service_name }}.service"
        mode: '0644'
      notify:
        - reload systemd
        - restart web service
    
    - name: Create service configuration
      copy:
        content: |
          # {{ web_service_name }} Configuration
          [server]
          port={{ web_port }}
          name={{ web_service_name }}
          
          [logging]
          level=INFO
          file=/var/log/{{ web_service_name }}.log
          
          [health]
          check_interval=30
          timeout=5
        dest: "/etc/{{ web_service_name }}.conf"
        mode: '0644'
      notify:
        - restart web service
        - update service status
    
    # Task that doesn't trigger handlers (no changes)
    - name: Ensure service directory exists
      file:
        path: "/var/lib/{{ web_service_name }}"
        state: directory
        mode: '0755'
    
    # Force handler execution for demonstration
    - name: Force handler notification (for demo)
      debug:
        msg: "Forcing handler execution for demonstration"
      changed_when: true
      notify: display service info
  
  handlers:
    # Basic service management handlers
    - name: reload systemd
      systemd:
        daemon_reload: yes
    
    - name: restart web service
      systemd:
        name: "{{ web_service_name }}"
        state: restarted
        enabled: yes
      listen: "restart web service"
    
    - name: validate web service
      uri:
        url: "http://localhost:{{ web_port }}"
        method: GET
        timeout: 10
      register: service_validation
      failed_when: false
      retries: 3
      delay: 2
    
    - name: update service status
      copy:
        content: |
          service={{ web_service_name }}
          status=configured
          last_update={{ ansible_date_time.iso8601 }}
          port={{ web_port }}
        dest: "/var/lib/{{ web_service_name }}/status"
        mode: '0644'
    
    - name: display service info
      debug:
        msg: |
          Service Information:
          - Name: {{ web_service_name }}
          - Port: {{ web_port }}
          - Status: {{ service_validation.status | default('unknown') }}
          - Validation: {{ 'SUCCESS' if service_validation.status == 200 else 'FAILED' }}
```

### Run and observe handler execution:
```bash
# First run - handlers should execute
ansible-playbook -i inventory.ini basic-handlers.yml

# Second run - handlers should not execute (no changes)
ansible-playbook -i inventory.ini basic-handlers.yml

# Check service status
sudo systemctl status demo-web --no-pager || echo "Service not running"
```

## Exercise 2: Advanced Handler Patterns (10 minutes)

### Task: Implement advanced handler patterns with dependencies and conditions

Create `advanced-handlers.yml`:
```yaml
---
- name: Advanced handler patterns
  hosts: localhost
  become: yes
  vars:
    services:
      - name: "app-frontend"
        port: 3000
        dependencies: ["app-backend"]
        config_template: "frontend.conf.j2"
      - name: "app-backend"
        port: 8080
        dependencies: ["app-database"]
        config_template: "backend.conf.j2"
      - name: "app-database"
        port: 5432
        dependencies: []
        config_template: "database.conf.j2"
        
  tasks:
    # Create configurations that trigger handlers
    - name: Generate service configurations
      copy:
        content: |
          # {{ item.name }} Configuration
          [service]
          name={{ item.name }}
          port={{ item.port }}
          
          [dependencies]
          {% for dep in item.dependencies %}
          requires={{ dep }}
          {% endfor %}
          
          [metadata]
          template={{ item.config_template }}
          generated={{ ansible_date_time.iso8601 }}
        dest: "/etc/{{ item.name }}.conf"
        mode: '0644'
      loop: "{{ services }}"
      notify:
        - "restart {{ item.name }}"
        - "validate {{ item.name }}"
        - "update load balancer"
    
    - name: Create service startup scripts
      copy:
        content: |
          #!/bin/bash
          # {{ item.name }} startup script
          
          # Check dependencies
          {% for dep in item.dependencies %}
          if ! systemctl is-active {{ dep }} >/dev/null 2>&1; then
            echo "Dependency {{ dep }} is not running"
            exit 1
          fi
          {% endfor %}
          
          echo "Starting {{ item.name }} on port {{ item.port }}"
          
          # Simulate service
          while true; do
            echo "{{ item.name }} heartbeat $(date)"
            sleep 30
          done
        dest: "/opt/{{ item.name }}.sh"
        mode: '0755'
      loop: "{{ services }}"
      notify:
        - "restart {{ item.name }}"
    
    - name: Create systemd service files
      copy:
        content: |
          [Unit]
          Description={{ item.name | title }} Service
          After=network.target{% for dep in item.dependencies %} {{ dep }}.service{% endfor %}
          
          {% for dep in item.dependencies %}
          Requires={{ dep }}.service
          {% endfor %}
          
          [Service]
          Type=simple
          ExecStart=/opt/{{ item.name }}.sh
          Restart=always
          RestartSec=10
          
          [Install]
          WantedBy=multi-user.target
        dest: "/etc/systemd/system/{{ item.name }}.service"
        mode: '0644'
      loop: "{{ services }}"
      notify:
        - reload systemd
        - "restart services in order"
    
    # Configuration that affects multiple services
    - name: Update global configuration
      copy:
        content: |
          # Global application configuration
          [cluster]
          name=demo-cluster
          environment=development
          
          [services]
          {% for service in services %}
          {{ service.name }}=http://localhost:{{ service.port }}
          {% endfor %}
          
          [monitoring]
          enabled=true
          interval=60
        dest: "/etc/global-app.conf"
        mode: '0644'
      notify:
        - restart all services
        - update monitoring config
  
  handlers:
    # Systemd management
    - name: reload systemd
      systemd:
        daemon_reload: yes
    
    # Individual service handlers (dynamic)
    - name: "restart app-frontend"
      systemd:
        name: "app-frontend"
        state: restarted
        enabled: yes
      listen: "restart app-frontend"
    
    - name: "restart app-backend"
      systemd:
        name: "app-backend"
        state: restarted
        enabled: yes
      listen: "restart app-backend"
    
    - name: "restart app-database"
      systemd:
        name: "app-database"
        state: restarted
        enabled: yes
      listen: "restart app-database"
    
    # Validation handlers
    - name: "validate app-frontend"
      debug:
        msg: "Validating app-frontend on port 3000"
      listen: "validate app-frontend"
    
    - name: "validate app-backend"
      debug:
        msg: "Validating app-backend on port 8080"
      listen: "validate app-backend"
    
    - name: "validate app-database"
      debug:
        msg: "Validating app-database on port 5432"
      listen: "validate app-database"
    
    # Ordered restart handler
    - name: "restart services in order"
      systemd:
        name: "{{ item }}"
        state: restarted
        enabled: yes
      loop:
        - "app-database"
        - "app-backend"
        - "app-frontend"
    
    # Bulk operations
    - name: restart all services
      systemd:
        name: "{{ item.name }}"
        state: restarted
        enabled: yes
      loop: "{{ services }}"
    
    # External system integration
    - name: update load balancer
      copy:
        content: |
          # Load balancer configuration update
          # Generated: {{ ansible_date_time.iso8601 }}
          
          upstream backend {
          {% for service in services %}
          {% if 'backend' in service.name %}
            server localhost:{{ service.port }};
          {% endif %}
          {% endfor %}
          }
          
          upstream frontend {
          {% for service in services %}
          {% if 'frontend' in service.name %}
            server localhost:{{ service.port }};
          {% endif %}
          {% endfor %}
          }
        dest: "/etc/nginx-upstream.conf"
        mode: '0644'
    
    - name: update monitoring config
      copy:
        content: |
          # Monitoring configuration
          # Generated: {{ ansible_date_time.iso8601 }}
          
          [targets]
          {% for service in services %}
          {{ service.name }}=localhost:{{ service.port }}
          {% endfor %}
          
          [checks]
          interval=60
          timeout=10
          retries=3
        dest: "/etc/monitoring.conf"
        mode: '0644'
```

### Run advanced handlers:
```bash
# Run with advanced handlers
ansible-playbook -i inventory.ini advanced-handlers.yml

# Check created services
sudo systemctl list-unit-files | grep app-
ls -la /etc/app-*.conf
cat /etc/global-app.conf
```

## Exercise 3: Handler Error Handling and Recovery (5 minutes)

### Task: Implement robust handler error handling

Create `handler-error-handling.yml`:
```yaml
---
- name: Handler error handling and recovery
  hosts: localhost
  become: yes
  vars:
    critical_service: "critical-app"
    
  tasks:
    - name: Create critical service configuration
      copy:
        content: |
          # Critical service configuration
          [service]
          name={{ critical_service }}
          port=9999
          critical=true
          
          [recovery]
          max_retries=3
          backup_command=/opt/restore-backup.sh
        dest: "/etc/{{ critical_service }}.conf"
        mode: '0644'
      notify:
        - restart critical service safely
        - validate critical service
        - notify operations team
    
    - name: Create service script with potential failure
      copy:
        content: |
          #!/bin/bash
          # Critical service with potential failure points
          
          # Check configuration
          if [ ! -f "/etc/{{ critical_service }}.conf" ]; then
            echo "Configuration missing!"
            exit 1
          fi
          
          # Simulate service startup
          echo "Starting {{ critical_service }}..."
          
          # Simulate potential failure (uncomment to test)
          # exit 1
          
          echo "{{ critical_service }} started successfully"
          sleep infinity
        dest: "/opt/{{ critical_service }}.sh"
        mode: '0755'
      notify: restart critical service safely
    
    - name: Create backup script
      copy:
        content: |
          #!/bin/bash
          # Backup and recovery script
          echo "Creating backup of {{ critical_service }} configuration"
          cp /etc/{{ critical_service }}.conf /etc/{{ critical_service }}.conf.backup
          echo "Backup created at $(date)"
        dest: "/opt/restore-backup.sh"
        mode: '0755'
  
  handlers:
    # Safe restart with error handling
    - name: restart critical service safely
      block:
        - name: Stop critical service
          systemd:
            name: "{{ critical_service }}"
            state: stopped
          failed_when: false
        
        - name: Validate configuration before restart
          command: "/bin/bash -n /opt/{{ critical_service }}.sh"
          register: config_validation
        
        - name: Start critical service
          systemd:
            name: "{{ critical_service }}"
            state: started
            enabled: yes
          register: service_start
      
      rescue:
        - name: Service restart failed - attempting recovery
          debug:
            msg: "Critical service restart failed, attempting recovery"
        
        - name: Run backup restoration
          command: "/opt/restore-backup.sh"
          register: backup_restore
        
        - name: Attempt service start with backup config
          systemd:
            name: "{{ critical_service }}"
            state: started
          failed_when: false
          register: recovery_attempt
        
        - name: Log recovery attempt
          copy:
            content: |
              Recovery attempt for {{ critical_service }}
              Time: {{ ansible_date_time.iso8601 }}
              Backup restore: {{ backup_restore.rc == 0 }}
              Service recovery: {{ recovery_attempt.state | default('failed') }}
            dest: "/var/log/{{ critical_service }}-recovery.log"
            mode: '0644'
      
      always:
        - name: Record service restart attempt
          lineinfile:
            path: "/var/log/{{ critical_service }}-restarts.log"
            line: "{{ ansible_date_time.iso8601 }}: Restart attempt - {{ 'SUCCESS' if service_start is succeeded else 'FAILED' }}"
            create: yes
            mode: '0644'
    
    # Validation with retry logic
    - name: validate critical service
      uri:
        url: "http://localhost:9999/health"
        method: GET
        timeout: 5
      register: health_check
      retries: 5
      delay: 3
      failed_when: false
    
    # Notification handler
    - name: notify operations team
      copy:
        content: |
          ALERT: Critical Service Update
          
          Service: {{ critical_service }}
          Time: {{ ansible_date_time.iso8601 }}
          Host: {{ ansible_hostname }}
          
          Health Check: {{ 'PASSED' if health_check.status == 200 else 'FAILED' }}
          
          {% if health_check.status != 200 %}
          ATTENTION REQUIRED: Service may not be responding properly
          {% endif %}
        dest: "/tmp/ops-notification.txt"
        mode: '0644'
```

### Test error handling:
```bash
# Run handler error handling
ansible-playbook -i inventory.ini handler-error-handling.yml

# Check logs and notifications
cat /var/log/critical-app-restarts.log 2>/dev/null || echo "No restart log"
cat /tmp/ops-notification.txt 2>/dev/null || echo "No notification"
```

## Verification and Discussion

### 1. Check Results
```bash
# Check basic handlers results
sudo systemctl status demo-web --no-pager || echo "Demo web service not running"
ls -la /var/lib/demo-web/

# Check advanced handlers results
sudo systemctl list-unit-files | grep app-
cat /etc/nginx-upstream.conf
cat /etc/monitoring.conf

# Check error handling results
ls -la /var/log/critical-app* 2>/dev/null || echo "No critical app logs"
```

### 2. Discussion Points
- How do you currently handle service restarts in your automation?
- What strategies do you use for error handling in service management?
- How do you implement notification systems for critical services?

### 3. Clean Up
```bash
# Stop and remove services
sudo systemctl stop demo-web app-frontend app-backend app-database critical-app 2>/dev/null || true
sudo systemctl disable demo-web app-frontend app-backend app-database critical-app 2>/dev/null || true

# Remove files
sudo rm -f /etc/systemd/system/{demo-web,app-*,critical-app}.service
sudo rm -f /etc/{demo-web,app-*,critical-app,global-app,nginx-upstream,monitoring}.conf
sudo rm -f /opt/{demo-web,app-*,critical-app,restore-backup}.sh
sudo rm -rf /var/lib/demo-web
sudo rm -f /var/log/critical-app* /tmp/ops-notification.txt

sudo systemctl daemon-reload
```

## Key Takeaways
- Handlers provide event-driven automation for efficient service management
- Handler execution is deferred until all tasks complete
- Multiple tasks can notify the same handler
- Handler error handling prevents automation failures
- Advanced handler patterns enable complex service orchestration
- Proper handler design improves automation reliability and efficiency

## Next Steps
Proceed to Lab 1.6: Playbook Execution Methods

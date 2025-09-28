# Lab 1.3: Task Organization and Dependencies

## Objective
Learn to organize tasks effectively, understand execution flow, and implement task dependencies for complex automation scenarios.

## Duration
35 minutes

## Prerequisites
- Completed Labs 1.1 and 1.2
- Understanding of basic task execution

## Lab Setup

```bash
cd ~/ansible-labs/module-01
mkdir -p lab-1.3
cd lab-1.3

cat > inventory.ini << EOF
[webservers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Task Dependencies and Ordering (15 minutes)

### Task: Implement a web application deployment with proper task dependencies

Create `web-deployment.yml`:
```yaml
---
- name: Web application deployment with dependencies
  hosts: webservers
  become: yes
  vars:
    app_name: webapp
    app_version: "2.1.0"
    app_port: 8080
    
  tasks:
    # Phase 1: System preparation (must run first)
    - name: Update package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
      tags: preparation
    
    - name: Install required system packages
      package:
        name:
          - nginx
          - python3
          - python3-pip
          - curl
        state: present
      tags: preparation
    
    # Phase 2: User and directory setup (depends on system prep)
    - name: Create application user
      user:
        name: "{{ app_name }}"
        system: yes
        shell: /bin/false
        home: "/opt/{{ app_name }}"
        create_home: yes
      tags: setup
    
    - name: Create application directory structure
      file:
        path: "{{ item }}"
        state: directory
        owner: "{{ app_name }}"
        group: "{{ app_name }}"
        mode: '0755'
      loop:
        - "/opt/{{ app_name }}/app"
        - "/opt/{{ app_name }}/config"
        - "/opt/{{ app_name }}/logs"
        - "/var/log/{{ app_name }}"
      tags: setup
    
    # Phase 3: Application installation (depends on user/directories)
    - name: Create application script
      copy:
        content: |
          #!/usr/bin/env python3
          # {{ app_name }} v{{ app_version }}
          import http.server
          import socketserver
          import os
          
          PORT = {{ app_port }}
          
          class MyHandler(http.server.SimpleHTTPRequestHandler):
              def do_GET(self):
                  if self.path == '/health':
                      self.send_response(200)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(b'{"status": "healthy", "version": "{{ app_version }}"}')
                  else:
                      self.send_response(200)
                      self.send_header('Content-type', 'text/html')
                      self.end_headers()
                      self.wfile.write(b'<h1>{{ app_name }} v{{ app_version }}</h1>')
          
          with socketserver.TCPServer(("", PORT), MyHandler) as httpd:
              print(f"Server running on port {PORT}")
              httpd.serve_forever()
        dest: "/opt/{{ app_name }}/app/server.py"
        owner: "{{ app_name }}"
        group: "{{ app_name }}"
        mode: '0755'
      tags: application
    
    # Phase 4: Configuration (depends on application files)
    - name: Generate application configuration
      copy:
        content: |
          # {{ app_name }} Configuration
          [server]
          name={{ app_name }}
          version={{ app_version }}
          port={{ app_port }}
          
          [logging]
          level=INFO
          file=/var/log/{{ app_name }}/app.log
          
          [deployment]
          deployed_by={{ ansible_user_id }}
          deployed_at={{ ansible_date_time.iso8601 }}
          host={{ ansible_hostname }}
        dest: "/opt/{{ app_name }}/config/app.conf"
        owner: "{{ app_name }}"
        group: "{{ app_name }}"
        mode: '0644'
      tags: configuration
    
    # Phase 5: Service setup (depends on all previous phases)
    - name: Create systemd service file
      copy:
        content: |
          [Unit]
          Description={{ app_name }} Web Application
          After=network.target
          
          [Service]
          Type=simple
          User={{ app_name }}
          Group={{ app_name }}
          WorkingDirectory=/opt/{{ app_name }}/app
          ExecStart=/usr/bin/python3 /opt/{{ app_name }}/app/server.py
          Restart=always
          RestartSec=5
          
          [Install]
          WantedBy=multi-user.target
        dest: "/etc/systemd/system/{{ app_name }}.service"
        mode: '0644'
      tags: service
      notify: reload systemd
    
    # Phase 6: Verification (runs after everything else)
    - name: Verify application files exist
      stat:
        path: "{{ item }}"
      register: file_check
      failed_when: not file_check.stat.exists
      loop:
        - "/opt/{{ app_name }}/app/server.py"
        - "/opt/{{ app_name }}/config/app.conf"
        - "/etc/systemd/system/{{ app_name }}.service"
      tags: verification
    
    - name: Display deployment summary
      debug:
        msg: |
          Deployment Summary:
          - Application: {{ app_name }} v{{ app_version }}
          - Port: {{ app_port }}
          - User: {{ app_name }}
          - Files verified: {{ file_check.results | selectattr('stat.exists') | list | length }}
      tags: verification
  
  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes
```

### Run with different tag combinations:
```bash
# Run all phases
ansible-playbook -i inventory.ini web-deployment.yml

# Run only preparation and setup
ansible-playbook -i inventory.ini web-deployment.yml --tags "preparation,setup"

# Skip verification
ansible-playbook -i inventory.ini web-deployment.yml --skip-tags "verification"
```

## Exercise 2: Task List Patterns and Organization (10 minutes)

### Task: Implement different task organization patterns

Create `task-patterns.yml`:
```yaml
---
- name: Task organization patterns
  hosts: localhost
  vars:
    services:
      - name: web
        port: 80
        config_template: "web.conf.j2"
        dependencies: ["database"]
      - name: api
        port: 8080
        config_template: "api.conf.j2"
        dependencies: ["database", "cache"]
      - name: database
        port: 3306
        config_template: "db.conf.j2"
        dependencies: []
      - name: cache
        port: 6379
        config_template: "cache.conf.j2"
        dependencies: []
        
  tasks:
    # Pattern 1: Sequential execution with dependencies
    - name: Install services in dependency order
      debug:
        msg: "Installing {{ item.name }} service (port {{ item.port }})"
      loop: "{{ services | sort(attribute='dependencies', reverse=true) }}"
      tags: sequential
    
    # Pattern 2: Grouped execution by function
    - name: Create service directories
      file:
        path: "/tmp/services/{{ item.name }}"
        state: directory
        mode: '0755'
      loop: "{{ services }}"
      tags: directories
    
    - name: Generate service configurations
      copy:
        content: |
          # {{ item.name }} service configuration
          [service]
          name={{ item.name }}
          port={{ item.port }}
          
          [dependencies]
          {% for dep in item.dependencies %}
          requires={{ dep }}
          {% endfor %}
          
          [metadata]
          created={{ ansible_date_time.iso8601 }}
        dest: "/tmp/services/{{ item.name }}/config.conf"
        mode: '0644'
      loop: "{{ services }}"
      tags: configuration
    
    # Pattern 3: Conditional execution based on dependencies
    - name: Start services without dependencies first
      debug:
        msg: "Starting {{ item.name }} (no dependencies)"
      loop: "{{ services }}"
      when: item.dependencies | length == 0
      tags: start_independent
    
    - name: Start services with dependencies
      debug:
        msg: "Starting {{ item.name }} (depends on: {{ item.dependencies | join(', ') }})"
      loop: "{{ services }}"
      when: item.dependencies | length > 0
      tags: start_dependent
    
    # Pattern 4: Validation and rollback preparation
    - name: Validate service configurations
      stat:
        path: "/tmp/services/{{ item.name }}/config.conf"
      register: config_validation
      loop: "{{ services }}"
      tags: validation
    
    - name: Report validation results
      debug:
        msg: |
          Service: {{ item.item.name }}
          Config exists: {{ item.stat.exists }}
          Config size: {{ item.stat.size | default(0) }} bytes
      loop: "{{ config_validation.results }}"
      tags: validation
```

### Run different patterns:
```bash
# Run all patterns
ansible-playbook -i inventory.ini task-patterns.yml

# Run only directory and configuration setup
ansible-playbook -i inventory.ini task-patterns.yml --tags "directories,configuration"

# Run validation only
ansible-playbook -i inventory.ini task-patterns.yml --tags "validation"
```

## Exercise 3: Complex Task Dependencies with Error Handling (10 minutes)

### Task: Implement complex dependencies with proper error handling

Create `complex-dependencies.yml`:
```yaml
---
- name: Complex task dependencies with error handling
  hosts: localhost
  become: yes
  vars:
    deployment_phases:
      - name: "infrastructure"
        tasks: ["create_users", "setup_directories", "install_packages"]
        critical: true
      - name: "application"
        tasks: ["deploy_code", "configure_app", "setup_database"]
        critical: true
      - name: "services"
        tasks: ["start_services", "configure_monitoring"]
        critical: false
      - name: "verification"
        tasks: ["health_checks", "performance_tests"]
        critical: false
        
  tasks:
    # Infrastructure Phase
    - name: "Phase: Infrastructure Setup"
      block:
        - name: Create application users
          user:
            name: "{{ item }}"
            system: yes
            shell: /bin/false
            home: "/opt/{{ item }}"
            create_home: yes
          loop:
            - "webuser"
            - "dbuser"
            - "cacheuser"
          register: user_creation
        
        - name: Setup application directories
          file:
            path: "{{ item }}"
            state: directory
            owner: "webuser"
            group: "webuser"
            mode: '0755'
          loop:
            - "/opt/webapp"
            - "/opt/webapp/config"
            - "/opt/webapp/logs"
            - "/var/log/webapp"
        
        - name: Install required packages
          package:
            name:
              - curl
              - wget
              - python3
            state: present
      
      rescue:
        - name: Infrastructure setup failed
          debug:
            msg: "CRITICAL: Infrastructure setup failed - stopping deployment"
        
        - name: Cleanup partial infrastructure
          user:
            name: "{{ item }}"
            state: absent
            remove: yes
          loop:
            - "webuser"
            - "dbuser"
            - "cacheuser"
          failed_when: false
        
        - name: Fail deployment
          fail:
            msg: "Infrastructure phase failed - deployment aborted"
      
      tags: infrastructure
    
    # Application Phase (depends on infrastructure)
    - name: "Phase: Application Deployment"
      block:
        - name: Deploy application code
          copy:
            content: |
              #!/bin/bash
              # Application startup script
              echo "Starting webapp..."
              echo "Configuration loaded from /opt/webapp/config/"
              echo "Logging to /var/log/webapp/"
            dest: "/opt/webapp/start.sh"
            owner: "webuser"
            group: "webuser"
            mode: '0755'
        
        - name: Configure application
          copy:
            content: |
              # Application configuration
              [app]
              name=webapp
              port=8080
              
              [database]
              host=localhost
              port=3306
              
              [logging]
              level=INFO
              file=/var/log/webapp/app.log
            dest: "/opt/webapp/config/app.conf"
            owner: "webuser"
            group: "webuser"
            mode: '0644'
        
        - name: Setup database connection
          debug:
            msg: "Database connection configured"
      
      rescue:
        - name: Application deployment failed
          debug:
            msg: "WARNING: Application deployment failed - attempting recovery"
        
        - name: Deploy minimal application
          copy:
            content: |
              #!/bin/bash
              echo "Minimal webapp running..."
            dest: "/opt/webapp/minimal.sh"
            owner: "webuser"
            group: "webuser"
            mode: '0755'
      
      tags: application
    
    # Services Phase (depends on application)
    - name: "Phase: Services Configuration"
      block:
        - name: Start application services
          debug:
            msg: "Starting service: {{ item }}"
          loop:
            - "webapp"
            - "database"
            - "cache"
        
        - name: Configure monitoring
          copy:
            content: |
              # Monitoring configuration
              [monitoring]
              enabled=true
              interval=60
              
              [alerts]
              email=admin@company.com
              threshold=80
            dest: "/opt/webapp/config/monitoring.conf"
            owner: "webuser"
            group: "webuser"
            mode: '0644'
      
      rescue:
        - name: Services configuration failed
          debug:
            msg: "WARNING: Services configuration failed - continuing with basic setup"
      
      tags: services
    
    # Verification Phase
    - name: "Phase: Deployment Verification"
      block:
        - name: Verify application files
          stat:
            path: "{{ item }}"
          register: file_verification
          loop:
            - "/opt/webapp/start.sh"
            - "/opt/webapp/config/app.conf"
        
        - name: Run health checks
          uri:
            url: "http://localhost:8080/health"
            method: GET
          register: health_check
          failed_when: false
        
        - name: Display deployment status
          debug:
            msg: |
              Deployment Status:
              - Files verified: {{ file_verification.results | selectattr('stat.exists') | list | length }}/{{ file_verification.results | length }}
              - Health check: {{ 'PASSED' if health_check.status == 200 else 'FAILED' }}
              - Deployment: {{ 'SUCCESS' if (file_verification.results | selectattr('stat.exists') | list | length) == (file_verification.results | length) else 'PARTIAL' }}
      
      rescue:
        - name: Verification failed
          debug:
            msg: "WARNING: Some verification checks failed"
      
      tags: verification
```

### Run with error simulation:
```bash
# Run complete deployment
ansible-playbook -i inventory.ini complex-dependencies.yml

# Run specific phases
ansible-playbook -i inventory.ini complex-dependencies.yml --tags "infrastructure,application"

# Test error handling by running without sudo (should fail gracefully)
ansible-playbook -i inventory.ini complex-dependencies.yml --tags "infrastructure" --ask-become-pass
```

## Verification and Discussion

### 1. Check Results
```bash
# Verify web deployment
ls -la /opt/webapp/
cat /opt/webapp/config/app.conf
systemctl status webapp --no-pager || echo "Service not started"

# Check task patterns results
ls -la /tmp/services/
cat /tmp/services/web/config.conf

# Verify users created
id webuser dbuser cacheuser 2>/dev/null || echo "Some users not created"
```

### 2. Discussion Points
- How do you currently handle task dependencies in your playbooks?
- What strategies do you use for error handling in complex deployments?
- How do you organize tasks for maintainability in large playbooks?

### 3. Clean Up
```bash
# Remove demo files and users
sudo rm -rf /opt/webapp /tmp/services /var/log/webapp
sudo systemctl stop webapp 2>/dev/null || true
sudo systemctl disable webapp 2>/dev/null || true
sudo rm -f /etc/systemd/system/webapp.service
sudo systemctl daemon-reload
sudo userdel -r webuser dbuser cacheuser 2>/dev/null || true
```

## Key Takeaways
- Task dependencies ensure proper execution order
- Tags enable selective execution and testing
- Error handling with block/rescue prevents deployment failures
- Proper task organization improves maintainability
- Verification tasks ensure deployment success
- Phase-based deployment provides better control and debugging

## Next Steps
Proceed to Lab 1.4: State Tracking and Idempotency

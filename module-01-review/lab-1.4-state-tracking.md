# Lab 1.4: State Tracking and Idempotency

## Objective
Understand and implement idempotent operations, state tracking, and change detection in Ansible automation.

## Duration
20 minutes

## Prerequisites
- Completed previous labs
- Understanding of Ansible's desired state concept

## Lab Setup

```bash
cd ~/ansible-labs/module-01
mkdir -p lab-1.4
cd lab-1.4

cat > inventory.ini << EOF
[servers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Idempotency Demonstration (8 minutes)

### Task: Create idempotent operations and observe state changes

Create `idempotency-demo.yml`:
```yaml
---
- name: Idempotency demonstration
  hosts: localhost
  become: yes
  vars:
    app_name: "idempotent-app"
    config_content: |
      # Application Configuration
      app.name={{ app_name }}
      app.version=1.0.0
      app.environment=production
      app.debug=false
      
      # Database settings
      db.host=localhost
      db.port=5432
      db.name={{ app_name }}_db
      
      # Generated on {{ ansible_date_time.iso8601 }}
      
  tasks:
    # Idempotent file operations
    - name: Create application directory (idempotent)
      file:
        path: "/opt/{{ app_name }}"
        state: directory
        owner: root
        group: root
        mode: '0755'
      register: dir_result
    
    - name: Display directory creation result
      debug:
        msg: |
          Directory creation changed: {{ dir_result.changed }}
          Directory path: {{ dir_result.path }}
    
    # Idempotent content management
    - name: Create configuration file (idempotent)
      copy:
        content: "{{ config_content }}"
        dest: "/opt/{{ app_name }}/app.conf"
        owner: root
        group: root
        mode: '0644'
      register: config_result
    
    - name: Display configuration result
      debug:
        msg: |
          Configuration changed: {{ config_result.changed }}
          Checksum: {{ config_result.checksum | default('N/A') }}
    
    # Idempotent package installation
    - name: Install required packages (idempotent)
      package:
        name:
          - curl
          - wget
        state: present
      register: package_result
    
    - name: Display package installation result
      debug:
        msg: |
          Packages changed: {{ package_result.changed }}
          Packages: {{ package_result.results | default([]) | length }} processed
    
    # Idempotent service configuration
    - name: Create systemd service file (idempotent)
      copy:
        content: |
          [Unit]
          Description={{ app_name }} Service
          After=network.target
          
          [Service]
          Type=simple
          ExecStart=/bin/bash -c 'echo "{{ app_name }} running"; sleep infinity'
          Restart=always
          
          [Install]
          WantedBy=multi-user.target
        dest: "/etc/systemd/system/{{ app_name }}.service"
        mode: '0644'
      register: service_file_result
      notify: reload systemd
    
    - name: Display service file result
      debug:
        msg: |
          Service file changed: {{ service_file_result.changed }}
          Service file exists: {{ service_file_result.dest is defined }}
    
    # Demonstrate non-idempotent vs idempotent approaches
    - name: Non-idempotent timestamp (always changes)
      copy:
        content: "Last run: {{ ansible_date_time.iso8601 }}"
        dest: "/opt/{{ app_name }}/timestamp.txt"
        mode: '0644'
      register: timestamp_result
    
    - name: Idempotent status file (only changes when needed)
      copy:
        content: |
          status=active
          version=1.0.0
          environment=production
        dest: "/opt/{{ app_name }}/status.conf"
        mode: '0644'
      register: status_result
    
    - name: Compare idempotent vs non-idempotent results
      debug:
        msg: |
          Timestamp file (non-idempotent): {{ timestamp_result.changed }}
          Status file (idempotent): {{ status_result.changed }}
  
  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes
```

### Run multiple times to observe idempotency:
```bash
# First run - should show changes
echo "=== FIRST RUN ==="
ansible-playbook -i inventory.ini idempotency-demo.yml

# Second run - should show minimal changes
echo "=== SECOND RUN ==="
ansible-playbook -i inventory.ini idempotency-demo.yml

# Third run - should be fully idempotent
echo "=== THIRD RUN ==="
ansible-playbook -i inventory.ini idempotency-demo.yml
```

## Exercise 2: State Tracking and Change Detection (7 minutes)

### Task: Implement advanced state tracking mechanisms

Create `state-tracking.yml`:
```yaml
---
- name: Advanced state tracking
  hosts: localhost
  become: yes
  vars:
    services:
      - name: web-service
        port: 8080
        enabled: true
        config:
          workers: 4
          timeout: 30
      - name: api-service
        port: 8081
        enabled: true
        config:
          workers: 2
          timeout: 60
      - name: admin-service
        port: 9090
        enabled: false
        config:
          workers: 1
          timeout: 120
          
  tasks:
    # Track state before changes
    - name: Gather current state information
      block:
        - name: Check existing service files
          find:
            paths: /etc/systemd/system
            patterns: "*-service.service"
          register: existing_services
        
        - name: Check existing configuration files
          find:
            paths: /opt
            patterns: "*.conf"
            recurse: yes
          register: existing_configs
        
        - name: Display current state
          debug:
            msg: |
              Current State:
              - Existing services: {{ existing_services.files | length }}
              - Existing configs: {{ existing_configs.files | length }}
      tags: state_check
    
    # Make changes and track them
    - name: Configure services with state tracking
      block:
        - name: Create service configurations
          copy:
            content: |
              # {{ item.name }} Configuration
              [service]
              name={{ item.name }}
              port={{ item.port }}
              enabled={{ item.enabled | lower }}
              
              [workers]
              count={{ item.config.workers }}
              timeout={{ item.config.timeout }}
              
              [metadata]
              created={{ ansible_date_time.iso8601 }}
              managed_by=ansible
            dest: "/opt/{{ item.name }}.conf"
            mode: '0644'
          loop: "{{ services }}"
          register: config_changes
          when: item.enabled | default(false)
        
        - name: Create systemd service files
          copy:
            content: |
              [Unit]
              Description={{ item.name | title }}
              After=network.target
              
              [Service]
              Type=simple
              ExecStart=/bin/bash -c 'echo "{{ item.name }} on port {{ item.port }}"; sleep infinity'
              Restart=always
              
              [Install]
              WantedBy=multi-user.target
            dest: "/etc/systemd/system/{{ item.name }}.service"
            mode: '0644'
          loop: "{{ services }}"
          register: service_changes
          when: item.enabled | default(false)
          notify: reload systemd
        
        - name: Track configuration changes
          debug:
            msg: |
              Service: {{ item.item.name }}
              Config changed: {{ item.changed | default(false) }}
              {% if item.changed | default(false) %}
              Config file: {{ item.dest | default('N/A') }}
              {% endif %}
          loop: "{{ config_changes.results | default([]) }}"
          when: item.item is defined
        
        - name: Track service file changes
          debug:
            msg: |
              Service: {{ item.item.name }}
              Service file changed: {{ item.changed | default(false) }}
              {% if item.changed | default(false) %}
              Service file: {{ item.dest | default('N/A') }}
              {% endif %}
          loop: "{{ service_changes.results | default([]) }}"
          when: item.item is defined
      tags: configure_services
    
    # Verify final state
    - name: Verify final state
      block:
        - name: Check final service files
          find:
            paths: /etc/systemd/system
            patterns: "*-service.service"
          register: final_services
        
        - name: Check final configuration files
          find:
            paths: /opt
            patterns: "*-service.conf"
          register: final_configs
        
        - name: Calculate state changes
          set_fact:
            services_added: "{{ final_services.files | length - existing_services.files | length }}"
            configs_added: "{{ final_configs.files | length - existing_configs.files | length }}"
        
        - name: Display state change summary
          debug:
            msg: |
              State Change Summary:
              - Services added: {{ services_added }}
              - Configurations added: {{ configs_added }}
              - Total enabled services: {{ services | selectattr('enabled') | list | length }}
              - Total services configured: {{ final_services.files | length }}
      tags: verify_state
  
  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes
```

### Run with state tracking:
```bash
# Run with state tracking
ansible-playbook -i inventory.ini state-tracking.yml

# Run again to see idempotency
ansible-playbook -i inventory.ini state-tracking.yml

# Run only state verification
ansible-playbook -i inventory.ini state-tracking.yml --tags "verify_state"
```

## Exercise 3: Change Detection and Conditional Actions (5 minutes)

### Task: Implement conditional actions based on state changes

Create `conditional-actions.yml`:
```yaml
---
- name: Conditional actions based on state changes
  hosts: localhost
  become: yes
  vars:
    application_config:
      name: "state-app"
      version: "2.0.0"
      port: 8082
      
  tasks:
    # Baseline state capture
    - name: Capture baseline state
      stat:
        path: "/opt/{{ application_config.name }}/app.conf"
      register: baseline_config
    
    # Make configuration changes
    - name: Update application configuration
      copy:
        content: |
          # {{ application_config.name }} v{{ application_config.version }}
          [application]
          name={{ application_config.name }}
          version={{ application_config.version }}
          port={{ application_config.port }}
          
          [runtime]
          max_memory=512M
          max_connections=100
          
          [features]
          logging=enabled
          metrics=enabled
          debug=disabled
        dest: "/opt/{{ application_config.name }}/app.conf"
        mode: '0644'
        backup: yes
      register: config_update
    
    # Conditional actions based on changes
    - name: Create backup notification (only if config changed)
      copy:
        content: |
          Configuration backup created at {{ ansible_date_time.iso8601 }}
          Original file: {{ config_update.backup_file | default('N/A') }}
          New checksum: {{ config_update.checksum | default('N/A') }}
        dest: "/opt/{{ application_config.name }}/backup-notice.txt"
        mode: '0644'
      when: config_update.changed
    
    - name: Restart application (only if config changed)
      debug:
        msg: "Restarting {{ application_config.name }} due to configuration change"
      when: config_update.changed
      changed_when: config_update.changed
    
    - name: Update version tracking (only if config changed)
      lineinfile:
        path: "/opt/{{ application_config.name }}/version-history.log"
        line: "{{ ansible_date_time.iso8601 }}: Updated to version {{ application_config.version }}"
        create: yes
        mode: '0644'
      when: config_update.changed
    
    # Always run verification
    - name: Verify configuration integrity
      stat:
        path: "/opt/{{ application_config.name }}/app.conf"
        checksum_algorithm: sha256
      register: final_config
    
    - name: Display change detection results
      debug:
        msg: |
          Change Detection Results:
          - Configuration existed before: {{ baseline_config.stat.exists }}
          - Configuration changed: {{ config_update.changed }}
          - Backup created: {{ config_update.backup_file is defined }}
          - Final checksum: {{ final_config.stat.checksum }}
          - Conditional actions triggered: {{ config_update.changed }}
```

### Run change detection:
```bash
# First run - should trigger conditional actions
ansible-playbook -i inventory.ini conditional-actions.yml

# Second run - should not trigger conditional actions
ansible-playbook -i inventory.ini conditional-actions.yml

# Verify results
ls -la /opt/state-app/
cat /opt/state-app/version-history.log
```

## Verification and Discussion

### 1. Check Results
```bash
# Check idempotency results
ls -la /opt/idempotent-app/
cat /opt/idempotent-app/status.conf
cat /opt/idempotent-app/timestamp.txt

# Check state tracking results
ls -la /opt/*-service.conf
systemctl list-unit-files | grep service.service

# Check change detection results
ls -la /opt/state-app/
cat /opt/state-app/backup-notice.txt 2>/dev/null || echo "No backup notice (no changes)"
```

### 2. Discussion Points
- How do you ensure idempotency in your current automation?
- What strategies do you use for tracking configuration changes?
- How do you handle conditional actions based on state changes?

### 3. Clean Up
```bash
# Remove demo files and services
sudo rm -rf /opt/idempotent-app /opt/state-app /opt/*-service.conf
sudo rm -f /etc/systemd/system/idempotent-app.service /etc/systemd/system/*-service.service
sudo systemctl daemon-reload
```

## Key Takeaways
- Idempotency ensures consistent results across multiple runs
- State tracking helps understand what changes during automation
- Change detection enables conditional actions and optimizations
- Proper state management improves automation reliability
- Verification tasks confirm desired state achievement
- Understanding changed vs unchanged operations improves debugging

## Next Steps
Proceed to Lab 1.5: Handlers and Notifications

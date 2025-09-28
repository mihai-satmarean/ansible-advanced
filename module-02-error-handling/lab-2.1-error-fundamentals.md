# Lab 2.1: Error Handling Fundamentals

## Objective
Master basic error handling techniques including failed_when, ignore_errors, and rescue blocks for robust automation.

## Duration
25 minutes

## Prerequisites
- Completed Module 1 labs
- Understanding of task execution flow

## Lab Setup

```bash
cd ~/ansible-labs
mkdir -p module-02/lab-2.1
cd module-02/lab-2.1

cat > inventory.ini << EOF
[servers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Basic Error Detection and Handling (10 minutes)

### Task: Implement basic error detection patterns

Create `error-detection.yml`:
```yaml
---
- name: Error detection fundamentals
  hosts: localhost
  vars:
    test_scenarios:
      - name: "valid_command"
        command: "echo 'Success'"
        expected_rc: 0
      - name: "invalid_command"
        command: "nonexistent_command_test"
        expected_rc: 127
      - name: "custom_failure"
        command: "echo 'Custom failure scenario'"
        expected_rc: 0
        should_fail: true
        
  tasks:
    # Basic failed_when usage
    - name: Test command execution with custom failure conditions
      shell: "{{ item.command }}"
      register: command_results
      failed_when: 
        - command_results.rc != item.expected_rc
        - not (item.should_fail | default(false))
      ignore_errors: yes
      loop: "{{ test_scenarios }}"
    
    - name: Display command results
      debug:
        msg: |
          Scenario: {{ item.item.name }}
          Command: {{ item.item.command }}
          Return Code: {{ item.rc | default('N/A') }}
          Failed: {{ item.failed | default(false) }}
          Stdout: {{ item.stdout | default('N/A') }}
      loop: "{{ command_results.results }}"
    
    # String-based failure detection
    - name: Test service status with string-based failure detection
      shell: |
        if [ "{{ item }}" = "failing_service" ]; then
          echo "ERROR: Service is down"
          exit 0
        else
          echo "OK: Service is running"
          exit 0
        fi
      register: service_check
      failed_when: "'ERROR' in service_check.stdout"
      ignore_errors: yes
      loop:
        - "working_service"
        - "failing_service"
        - "another_service"
    
    - name: Display service check results
      debug:
        msg: |
          Service: {{ item.item }}
          Status: {{ item.stdout }}
          Failed: {{ item.failed | default(false) }}
      loop: "{{ service_check.results }}"
    
    # Complex failure conditions
    - name: Test disk space with complex failure conditions
      shell: |
        # Simulate disk usage check
        usage={{ 95 if item == '/var' else 45 }}
        echo "Disk usage for {{ item }}: ${usage}%"
        echo $usage
      register: disk_check
      failed_when: 
        - disk_check.stdout_lines[1] | int > 90
        - "'var' in item"
      ignore_errors: yes
      loop:
        - "/home"
        - "/var"
        - "/tmp"
    
    - name: Display disk check results
      debug:
        msg: |
          Mount: {{ item.item }}
          Usage: {{ item.stdout_lines[0] }}
          Failed: {{ item.failed | default(false) }}
          Critical: {{ 'YES' if item.failed | default(false) else 'NO' }}
      loop: "{{ disk_check.results }}"
```

### Run error detection tests:
```bash
# Run error detection scenarios
ansible-playbook -i inventory.ini error-detection.yml

# Run with verbose output to see error details
ansible-playbook -i inventory.ini error-detection.yml -v
```

## Exercise 2: Ignore Errors and Continue Execution (8 minutes)

### Task: Implement selective error ignoring strategies

Create `ignore-errors.yml`:
```yaml
---
- name: Selective error ignoring strategies
  hosts: localhost
  vars:
    optional_packages:
      - name: "curl"
        critical: false
      - name: "nonexistent-package-test"
        critical: false
      - name: "python3"
        critical: true
    
    services_to_check:
      - name: "ssh"
        optional: false
      - name: "nonexistent-service"
        optional: true
      - name: "systemd-resolved"
        optional: false
        
  tasks:
    # Ignore errors for optional operations
    - name: Install optional packages
      package:
        name: "{{ item.name }}"
        state: present
      register: package_install
      ignore_errors: "{{ not item.critical }}"
      loop: "{{ optional_packages }}"
    
    - name: Report package installation results
      debug:
        msg: |
          Package: {{ item.item.name }}
          Critical: {{ item.item.critical }}
          Success: {{ not (item.failed | default(false)) }}
          {% if item.failed | default(false) %}
          Error: {{ item.msg | default('Unknown error') }}
          {% endif %}
      loop: "{{ package_install.results }}"
    
    # Conditional error ignoring
    - name: Check service status with conditional error handling
      systemd:
        name: "{{ item.name }}"
      register: service_status
      ignore_errors: "{{ item.optional }}"
      failed_when: 
        - service_status.failed | default(false)
        - not item.optional
      loop: "{{ services_to_check }}"
    
    - name: Report service check results
      debug:
        msg: |
          Service: {{ item.item.name }}
          Optional: {{ item.item.optional }}
          Status: {{ 'RUNNING' if not (item.failed | default(false)) else 'FAILED' }}
          {% if item.failed | default(false) and not item.item.optional %}
          CRITICAL: Required service is down!
          {% endif %}
      loop: "{{ service_status.results }}"
    
    # Graceful degradation pattern
    - name: Attempt advanced configuration (may fail gracefully)
      block:
        - name: Try to configure advanced feature
          shell: |
            # Simulate advanced configuration that might not be available
            if [ "{{ ansible_distribution }}" = "Ubuntu" ]; then
              echo "Configuring advanced Ubuntu features"
              # This might fail on some systems
              exit 0
            else
              echo "Advanced features not available on this system"
              exit 1
            fi
          register: advanced_config
        
        - name: Apply advanced configuration
          copy:
            content: |
              # Advanced Configuration Applied
              System: {{ ansible_distribution }}
              Features: enabled
              Configured: {{ ansible_date_time.iso8601 }}
            dest: "/tmp/advanced-config.conf"
            mode: '0644'
          when: advanced_config.rc == 0
      
      rescue:
        - name: Fall back to basic configuration
          copy:
            content: |
              # Basic Configuration Applied
              System: {{ ansible_distribution }}
              Features: basic
              Fallback: true
              Configured: {{ ansible_date_time.iso8601 }}
            dest: "/tmp/basic-config.conf"
            mode: '0644'
        
        - name: Log configuration fallback
          debug:
            msg: "Advanced configuration failed, using basic configuration"
    
    # Summary of error handling results
    - name: Generate error handling summary
      copy:
        content: |
          Error Handling Summary
          =====================
          
          Package Installation:
          {% for result in package_install.results %}
          - {{ result.item.name }}: {{ 'SUCCESS' if not (result.failed | default(false)) else 'FAILED' }} (Critical: {{ result.item.critical }})
          {% endfor %}
          
          Service Checks:
          {% for result in service_status.results %}
          - {{ result.item.name }}: {{ 'RUNNING' if not (result.failed | default(false)) else 'FAILED' }} (Optional: {{ result.item.optional }})
          {% endfor %}
          
          Configuration:
          - Advanced config: {{ 'SUCCESS' if advanced_config.rc == 0 else 'FAILED (fallback applied)' }}
          
          Generated: {{ ansible_date_time.iso8601 }}
        dest: "/tmp/error-handling-summary.txt"
        mode: '0644'
```

### Run ignore errors scenarios:
```bash
# Run ignore errors demonstration
ansible-playbook -i inventory.ini ignore-errors.yml

# Check results
cat /tmp/error-handling-summary.txt
ls -la /tmp/*config.conf
```

## Exercise 3: Rescue Blocks and Error Recovery (7 minutes)

### Task: Implement comprehensive error recovery patterns

Create `rescue-blocks.yml`:
```yaml
---
- name: Rescue blocks and error recovery
  hosts: localhost
  become: yes
  vars:
    deployment_config:
      app_name: "rescue-demo"
      version: "1.0.0"
      backup_enabled: true
      
  tasks:
    # Complex deployment with error recovery
    - name: Deploy application with error recovery
      block:
        - name: Create application directory
          file:
            path: "/opt/{{ deployment_config.app_name }}"
            state: directory
            mode: '0755'
        
        - name: Download application (simulate potential failure)
          get_url:
            url: "https://nonexistent-domain-for-testing.com/app.tar.gz"
            dest: "/tmp/{{ deployment_config.app_name }}.tar.gz"
            timeout: 5
          register: download_result
        
        - name: Extract application
          unarchive:
            src: "/tmp/{{ deployment_config.app_name }}.tar.gz"
            dest: "/opt/{{ deployment_config.app_name }}"
            remote_src: yes
        
        - name: Configure application
          copy:
            content: |
              # {{ deployment_config.app_name }} Configuration
              version={{ deployment_config.version }}
              deployed={{ ansible_date_time.iso8601 }}
            dest: "/opt/{{ deployment_config.app_name }}/config.conf"
            mode: '0644'
      
      rescue:
        - name: Log deployment failure
          debug:
            msg: "Primary deployment failed, attempting recovery"
        
        - name: Use local backup/fallback
          copy:
            content: |
              #!/bin/bash
              # Fallback application for {{ deployment_config.app_name }}
              echo "Fallback version running"
              echo "Version: {{ deployment_config.version }}-fallback"
              echo "Started: $(date)"
            dest: "/opt/{{ deployment_config.app_name }}/fallback-app.sh"
            mode: '0755'
        
        - name: Create fallback configuration
          copy:
            content: |
              # {{ deployment_config.app_name }} Fallback Configuration
              version={{ deployment_config.version }}-fallback
              mode=recovery
              deployed={{ ansible_date_time.iso8601 }}
              original_failure=download_timeout
            dest: "/opt/{{ deployment_config.app_name }}/config.conf"
            mode: '0644'
        
        - name: Set recovery flag
          set_fact:
            deployment_recovered: true
      
      always:
        - name: Clean up temporary files
          file:
            path: "/tmp/{{ deployment_config.app_name }}.tar.gz"
            state: absent
        
        - name: Log deployment attempt
          lineinfile:
            path: "/var/log/deployment.log"
            line: "{{ ansible_date_time.iso8601 }}: {{ deployment_config.app_name }} deployment {{ 'RECOVERED' if deployment_recovered | default(false) else 'SUCCESS' }}"
            create: yes
            mode: '0644'
        
        - name: Display final status
          debug:
            msg: |
              Deployment Status: {{ 'RECOVERED' if deployment_recovered | default(false) else 'SUCCESS' }}
              Application: {{ deployment_config.app_name }}
              Version: {{ deployment_config.version }}{{ '-fallback' if deployment_recovered | default(false) else '' }}
    
    # Database setup with transaction-like behavior
    - name: Database setup with rollback capability
      block:
        - name: Create database directory
          file:
            path: "/var/lib/{{ deployment_config.app_name }}-db"
            state: directory
            mode: '0750'
        
        - name: Initialize database (simulate failure)
          shell: |
            # Simulate database initialization that might fail
            if [ "{{ deployment_config.app_name }}" = "rescue-demo" ]; then
              echo "Database initialization failed" >&2
              exit 1
            fi
            echo "Database initialized successfully"
          register: db_init
        
        - name: Create database schema
          copy:
            content: |
              -- Database schema for {{ deployment_config.app_name }}
              CREATE TABLE users (id INT, name VARCHAR(100));
              CREATE TABLE sessions (id INT, user_id INT);
            dest: "/var/lib/{{ deployment_config.app_name }}-db/schema.sql"
            mode: '0644'
      
      rescue:
        - name: Database setup failed - performing rollback
          debug:
            msg: "Database setup failed, performing rollback"
        
        - name: Remove partial database files
          file:
            path: "/var/lib/{{ deployment_config.app_name }}-db"
            state: absent
        
        - name: Create minimal database fallback
          file:
            path: "/var/lib/{{ deployment_config.app_name }}-db-minimal"
            state: directory
            mode: '0750'
        
        - name: Log database rollback
          lineinfile:
            path: "/var/log/deployment.log"
            line: "{{ ansible_date_time.iso8601 }}: Database rollback performed for {{ deployment_config.app_name }}"
            create: yes
            mode: '0644'
      
      always:
        - name: Verify final database state
          stat:
            path: "/var/lib/{{ deployment_config.app_name }}-db"
          register: db_check
        
        - name: Display database status
          debug:
            msg: |
              Database Status: {{ 'ACTIVE' if db_check.stat.exists else 'MINIMAL FALLBACK' }}
              Path: {{ '/var/lib/' + deployment_config.app_name + '-db' if db_check.stat.exists else '/var/lib/' + deployment_config.app_name + '-db-minimal' }}
```

### Run rescue block scenarios:
```bash
# Run rescue blocks demonstration
ansible-playbook -i inventory.ini rescue-blocks.yml

# Check deployment results
ls -la /opt/rescue-demo/
cat /opt/rescue-demo/config.conf
cat /var/log/deployment.log
ls -la /var/lib/rescue-demo*
```

## Verification and Discussion

### 1. Check Results
```bash
# Check error detection results
cat /tmp/error-handling-summary.txt

# Check rescue deployment results
ls -la /opt/rescue-demo/
cat /opt/rescue-demo/fallback-app.sh

# Check logs
cat /var/log/deployment.log
```

### 2. Discussion Points
- How do you currently handle errors in your automation?
- What strategies do you use for graceful degradation?
- How do you implement rollback mechanisms in your deployments?

### 3. Clean Up
```bash
# Remove demo files
sudo rm -rf /opt/rescue-demo /var/lib/rescue-demo* /tmp/*config.conf /tmp/error-handling-summary.txt
sudo rm -f /var/log/deployment.log
```

## Key Takeaways
- `failed_when` provides custom failure conditions beyond return codes
- `ignore_errors` enables graceful handling of non-critical failures
- Block/rescue/always patterns provide structured error handling
- Error recovery should include cleanup and fallback mechanisms
- Proper logging helps with troubleshooting and auditing
- Conditional error handling enables flexible automation strategies

## Next Steps
Proceed to Lab 2.2: Block, Rescue, and Always Patterns

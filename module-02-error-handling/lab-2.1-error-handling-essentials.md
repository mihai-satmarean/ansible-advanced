# Lab 2.1: Error Handling Essentials (Interactive Demo)

## Objective
Master essential error handling techniques through interactive demonstration and practical examples suitable for all skill levels.

## Duration
25 minutes (interactive demo)

## Format
**Interactive demonstration** - Instructor-led with participant practice

## Prerequisites
- Completed Module 1 labs
- Understanding of task execution flow

## Demo Setup

```bash
cd ~/ansible-labs
mkdir -p module-02
cd module-02

# Create simple inventory
cat > inventory.ini << 'EOF'
[demo_hosts]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Demo 1: Basic Error Detection (8 minutes)

### Why Error Handling Matters
**Instructor explains:**
- Ansible tasks can fail for various reasons
- Default behavior: stop execution on first failure
- Error handling allows graceful recovery and continuation
- Essential for production automation

### Failed When Conditions
```yaml
# Create failed-when-demo.yml
cat > failed-when-demo.yml << 'EOF'
---
- name: Error Detection Demo
  hosts: demo_hosts
  
  tasks:
    - name: Check disk space (custom failure condition)
      shell: df -h / | tail -1 | awk '{print $5}' | sed 's/%//'
      register: disk_usage
      failed_when: disk_usage.stdout | int > 90
      
    - name: Verify service is running (simulate check)
      shell: echo "service_status=running"
      register: service_check
      failed_when: "'stopped' in service_check.stdout"
      
    - name: Check application response (simulate)
      shell: echo "HTTP_STATUS=200"
      register: http_check
      failed_when: "'200' not in http_check.stdout"
      
    - name: Custom validation example
      shell: echo "config_valid=true"
      register: config_check
      failed_when: config_check.stdout != "config_valid=true"
      
    - name: Show results
      debug:
        msg: |
          Disk usage: {{ disk_usage.stdout }}%
          Service status: {{ service_check.stdout }}
          HTTP status: {{ http_check.stdout }}
          Config check: {{ config_check.stdout }}
EOF

# Run the demo
ansible-playbook -i inventory.ini failed-when-demo.yml
```

### Ignore Errors Pattern
```yaml
# Create ignore-errors-demo.yml
cat > ignore-errors-demo.yml << 'EOF'
---
- name: Ignore Errors Demo
  hosts: demo_hosts
  
  tasks:
    - name: Try to install optional package (might fail)
      package:
        name: nonexistent-package
        state: present
      ignore_errors: yes
      register: optional_install
      
    - name: Show optional install result
      debug:
        msg: |
          Optional package install: {{ 'SUCCESS' if optional_install is succeeded else 'FAILED (ignored)' }}
          
    - name: Try to stop optional service (might not exist)
      service:
        name: optional-service
        state: stopped
      ignore_errors: yes
      register: optional_service
      
    - name: Continue with main tasks regardless
      debug:
        msg: "Main deployment continues regardless of optional failures"
        
    - name: Cleanup temporary files (ignore if they don't exist)
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /tmp/nonexistent-file1
        - /tmp/nonexistent-file2
      ignore_errors: yes
EOF

# Run the demo
ansible-playbook -i inventory.ini ignore-errors-demo.yml
```

## Demo 2: Block, Rescue, Always (10 minutes)

### Basic Block Structure
```yaml
# Create block-rescue-demo.yml
cat > block-rescue-demo.yml << 'EOF'
---
- name: Block, Rescue, Always Demo
  hosts: demo_hosts
  
  tasks:
    - name: Web server deployment with error handling
      block:
        - name: Download application
          get_url:
            url: "https://httpbin.org/status/200"
            dest: /tmp/app-download
            timeout: 10
          register: download_result
          
        - name: Verify download
          stat:
            path: /tmp/app-download
          register: download_check
          failed_when: not download_check.stat.exists
          
        - name: Simulate application installation
          copy:
            content: |
              # Application installed successfully
              # Version: 2.1.0
              # Date: {{ ansible_date_time.iso8601 }}
            dest: /tmp/app-installed
            
        - name: Show success message
          debug:
            msg: "Application deployed successfully!"
            
      rescue:
        - name: Handle deployment failure
          debug:
            msg: "Deployment failed, initiating recovery..."
            
        - name: Cleanup failed installation
          file:
            path: "{{ item }}"
            state: absent
          loop:
            - /tmp/app-download
            - /tmp/app-installed
          ignore_errors: yes
          
        - name: Use fallback version
          copy:
            content: |
              # Fallback application version
              # Version: 2.0.0 (stable)
              # Date: {{ ansible_date_time.iso8601 }}
            dest: /tmp/app-fallback
            
        - name: Notify about fallback
          debug:
            msg: "Deployed fallback version due to installation failure"
            
      always:
        - name: Update deployment log
          lineinfile:
            path: /tmp/deployment.log
            line: "{{ ansible_date_time.iso8601 }} - Deployment attempt completed"
            create: yes
            
        - name: Send notification (simulate)
          debug:
            msg: "Deployment notification sent to operations team"
EOF

# Run the demo
ansible-playbook -i inventory.ini block-rescue-demo.yml
cat /tmp/deployment.log
```

### Nested Error Handling
```yaml
# Create nested-blocks-demo.yml
cat > nested-blocks-demo.yml << 'EOF'
---
- name: Nested Error Handling Demo
  hosts: demo_hosts
  
  tasks:
    - name: Database deployment with nested error handling
      block:
        - name: Primary database setup
          block:
            - name: Create database directory
              file:
                path: /tmp/database
                state: directory
                
            - name: Initialize database
              copy:
                content: |
                  # Database initialized
                  # Type: Primary
                  # Status: Active
                dest: /tmp/database/primary.db
                
            - name: Start database service (simulate)
              debug:
                msg: "Primary database started successfully"
                
          rescue:
            - name: Try secondary database setup
              block:
                - name: Setup secondary database
                  copy:
                    content: |
                      # Database initialized
                      # Type: Secondary
                      # Status: Active
                    dest: /tmp/database/secondary.db
                    
                - name: Start secondary database
                  debug:
                    msg: "Secondary database started as fallback"
                    
              rescue:
                - name: Use minimal database
                  copy:
                    content: |
                      # Minimal database
                      # Type: SQLite
                      # Status: Limited
                    dest: /tmp/database/minimal.db
                    
                - name: Warn about limited functionality
                  debug:
                    msg: "WARNING: Running with minimal database functionality"
                    
      rescue:
        - name: Complete deployment failure
          debug:
            msg: "CRITICAL: Database deployment failed completely"
            
      always:
        - name: Check what was deployed
          find:
            paths: /tmp/database
            patterns: "*.db"
          register: deployed_dbs
          
        - name: Show deployment summary
          debug:
            msg: |
              Deployed databases: {{ deployed_dbs.files | map(attribute='path') | list }}
              Total databases: {{ deployed_dbs.matched }}
EOF

# Run the demo
ansible-playbook -i inventory.ini nested-blocks-demo.yml
ls -la /tmp/database/
```

## Demo 3: Practical Error Handling Patterns (7 minutes)

### Service Management with Error Handling
```yaml
# Create service-management-demo.yml
cat > service-management-demo.yml << 'EOF'
---
- name: Service Management with Error Handling
  hosts: demo_hosts
  
  vars:
    services_to_manage:
      - name: "web-app"
        port: 8080
        critical: true
      - name: "background-worker"
        port: 0
        critical: false
      - name: "monitoring-agent"
        port: 9090
        critical: false
        
  tasks:
    - name: Manage services with appropriate error handling
      block:
        - name: Create service configuration
          copy:
            content: |
              # Service: {{ item.name }}
              # Port: {{ item.port if item.port > 0 else 'N/A' }}
              # Critical: {{ item.critical }}
              # Status: Configured
            dest: "/tmp/{{ item.name }}.conf"
            
        - name: Start service (simulate)
          debug:
            msg: "Starting {{ item.name }} service..."
            
        - name: Verify service is running
          shell: echo "{{ item.name }}_status=running"
          register: service_status
          failed_when: "'running' not in service_status.stdout"
          
      rescue:
        - name: Handle service failure
          debug:
            msg: "Service {{ item.name }} failed to start"
            
        - name: Fail deployment if critical service fails
          fail:
            msg: "Critical service {{ item.name }} failed - stopping deployment"
          when: item.critical
          
        - name: Continue if non-critical service fails
          debug:
            msg: "Non-critical service {{ item.name }} failure - continuing deployment"
          when: not item.critical
          
      loop: "{{ services_to_manage }}"
      loop_control:
        loop_var: item
EOF

# Run the demo
ansible-playbook -i inventory.ini service-management-demo.yml
ls -la /tmp/*.conf
```

### Configuration Validation Pattern
```yaml
# Create config-validation-demo.yml
cat > config-validation-demo.yml << 'EOF'
---
- name: Configuration Validation Demo
  hosts: demo_hosts
  
  vars:
    app_config:
      database_url: "postgresql://localhost:5432/myapp"
      redis_url: "redis://localhost:6379"
      api_key: "sk-1234567890"
      debug_mode: false
      
  tasks:
    - name: Validate and deploy configuration
      block:
        - name: Validate database URL format
          assert:
            that:
              - app_config.database_url is defined
              - app_config.database_url | length > 0
              - "'postgresql://' in app_config.database_url"
            fail_msg: "Invalid database URL format"
            
        - name: Validate API key format
          assert:
            that:
              - app_config.api_key is defined
              - app_config.api_key | length >= 10
              - app_config.api_key.startswith('sk-')
            fail_msg: "Invalid API key format"
            
        - name: Deploy validated configuration
          copy:
            content: |
              # Application Configuration
              # Generated: {{ ansible_date_time.iso8601 }}
              
              [database]
              url={{ app_config.database_url }}
              
              [cache]
              url={{ app_config.redis_url }}
              
              [api]
              key={{ app_config.api_key }}
              
              [application]
              debug={{ app_config.debug_mode | lower }}
            dest: /tmp/app-config.ini
            
        - name: Validate deployed configuration
          shell: grep -q "postgresql://" /tmp/app-config.ini
          changed_when: false
          
      rescue:
        - name: Use default configuration on validation failure
          copy:
            content: |
              # Default Configuration (Validation Failed)
              # Generated: {{ ansible_date_time.iso8601 }}
              
              [database]
              url=sqlite:///tmp/default.db
              
              [cache]
              url=memory://
              
              [api]
              key=default-key
              
              [application]
              debug=true
            dest: /tmp/app-config.ini
            
        - name: Warn about default configuration
          debug:
            msg: "WARNING: Using default configuration due to validation failure"
EOF

# Run the demo
ansible-playbook -i inventory.ini config-validation-demo.yml
cat /tmp/app-config.ini
```

## Interactive Practice Session

### Hands-On Exercise for Participants
```yaml
# Create practice-exercise.yml
cat > practice-exercise.yml << 'EOF'
---
- name: Participant Practice - Error Handling
  hosts: demo_hosts
  
  vars:
    # TODO: Participants will work with this scenario
    web_servers:
      - name: "web01"
        port: 80
        required: true
      - name: "web02"
        port: 8080
        required: false
        
  tasks:
    # TODO: Participants create error handling for web server deployment
    # Requirements:
    # 1. Use block/rescue/always structure
    # 2. Handle failures differently for required vs optional servers
    # 3. Use failed_when for custom failure conditions
    # 4. Implement cleanup in rescue blocks
    # 5. Always log deployment attempts
    
    - name: Deploy web servers with error handling
      debug:
        msg: "TODO: Implement error handling here"
        
    # Hints:
    # - Use block for main deployment tasks
    # - Use rescue for failure handling
    # - Use always for logging
    # - Use failed_when for custom conditions
    # - Use fail module to stop on critical failures
EOF
```

**Practice Instructions:**
1. Create a block structure for web server deployment
2. Add rescue block for handling failures
3. Use always block for logging
4. Implement different handling for required vs optional servers
5. Test with both success and failure scenarios

## Key Takeaways Summary

### Essential Error Handling Techniques:
1. **failed_when** - Custom failure conditions
2. **ignore_errors** - Continue on non-critical failures
3. **block/rescue/always** - Structured error handling
4. **assert** - Validate conditions and data
5. **fail** - Explicitly fail with custom messages

### Error Handling Patterns:
- **Graceful degradation** - Fallback to simpler alternatives
- **Retry logic** - Attempt operations multiple times
- **Validation** - Check inputs and outputs
- **Cleanup** - Remove partial changes on failure
- **Logging** - Record all attempts and outcomes

### Best Practices:
- Handle errors close to where they occur
- Provide meaningful error messages
- Clean up resources in rescue blocks
- Use always blocks for logging and notifications
- Distinguish between critical and non-critical failures
- Validate inputs before processing
- Test both success and failure paths

### When to Use Each Technique:
- **failed_when**: Custom business logic validation
- **ignore_errors**: Optional operations that may fail
- **block/rescue**: Complex operations needing cleanup
- **assert**: Input validation and prerequisites
- **fail**: Explicit failure with clear messaging

## Demo Cleanup
```bash
# Clean up demo files
rm -f /tmp/app-* /tmp/*.conf /tmp/*.db /tmp/deployment.log
rm -rf /tmp/database/
```

---

**Note for Instructor**: Focus on practical, immediately applicable error handling patterns. Emphasize the importance of graceful failure handling in production environments. Adjust complexity based on participant experience with automation and error scenarios.

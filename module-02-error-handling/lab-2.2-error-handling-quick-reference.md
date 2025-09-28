# Lab 2.2: Advanced Error Handling (Quick Reference)

## Objective
Advanced error handling patterns and enterprise resilience strategies for self-study

## Target Audience
**Victor & Vlad** - Intermediate participants for post-course exploration

## Duration
30-45 minutes (self-paced)

## Pattern 1: Advanced Execution Strategies

### Linear vs Free Strategy
```yaml
# Linear strategy (default) - tasks execute in order across all hosts
- name: Linear execution example
  hosts: webservers
  strategy: linear
  serial: 2  # Process 2 hosts at a time
  
  tasks:
    - name: Update package cache
      apt:
        update_cache: yes
      
    - name: Install application
      package:
        name: myapp
        state: present

# Free strategy - hosts execute independently
- name: Free execution example  
  hosts: webservers
  strategy: free
  
  tasks:
    - name: Independent task execution
      shell: |
        # Each host runs at its own pace
        sleep {{ ansible_host.split('.')[-1] }}
        echo "Host {{ inventory_hostname }} completed"
```

### Serial Execution Control
```yaml
# Batch processing with error handling
- name: Rolling deployment with error control
  hosts: webservers
  serial: "25%"  # Process 25% of hosts at a time
  max_fail_percentage: 10  # Fail if more than 10% of hosts fail
  
  tasks:
    - name: Deploy application
      block:
        - name: Stop service
          service:
            name: myapp
            state: stopped
            
        - name: Update application
          copy:
            src: /path/to/new/version
            dest: /opt/myapp/
            backup: yes
          register: update_result
          
        - name: Start service
          service:
            name: myapp
            state: started
            
        - name: Verify deployment
          uri:
            url: "http://{{ ansible_host }}:8080/health"
            method: GET
            status_code: 200
          retries: 5
          delay: 10
          
      rescue:
        - name: Rollback on failure
          copy:
            src: "{{ update_result.backup_file }}"
            dest: /opt/myapp/
            remote_src: yes
          when: update_result.backup_file is defined
          
        - name: Restart service after rollback
          service:
            name: myapp
            state: restarted
            
        - name: Mark host as failed
          fail:
            msg: "Deployment failed on {{ inventory_hostname }}"
```

### Async Tasks with Error Handling
```yaml
# Asynchronous execution with monitoring
- name: Async operations with error handling
  hosts: databases
  
  tasks:
    - name: Start long-running backup
      shell: |
        pg_dump -h localhost -U postgres myapp > /backup/myapp_$(date +%Y%m%d_%H%M%S).sql
      async: 3600  # 1 hour timeout
      poll: 0      # Fire and forget
      register: backup_job
      
    - name: Start maintenance tasks
      shell: |
        # Run maintenance while backup is running
        vacuum analyze;
        reindex database myapp;
      async: 1800
      poll: 0
      register: maintenance_job
      
    - name: Monitor backup completion
      async_status:
        jid: "{{ backup_job.ansible_job_id }}"
      register: backup_result
      until: backup_result.finished
      retries: 360  # Check every 10 seconds for 1 hour
      delay: 10
      failed_when: backup_result.failed or backup_result.rc != 0
      
    - name: Monitor maintenance completion
      async_status:
        jid: "{{ maintenance_job.ansible_job_id }}"
      register: maintenance_result
      until: maintenance_result.finished
      retries: 180
      delay: 10
      ignore_errors: yes  # Maintenance is optional
```

## Pattern 2: Circuit Breaker and Retry Logic

### Retry with Exponential Backoff
```yaml
# Advanced retry patterns
- name: Resilient API operations
  hosts: localhost
  
  tasks:
    - name: API call with exponential backoff
      uri:
        url: "https://api.example.com/deploy"
        method: POST
        body_format: json
        body:
          application: "myapp"
          version: "{{ app_version }}"
        headers:
          Authorization: "Bearer {{ api_token }}"
        status_code: [200, 202]
      register: api_result
      retries: 5
      delay: "{{ 2 ** (ansible_loop.index0) }}"  # 1, 2, 4, 8, 16 seconds
      until: api_result.status in [200, 202]
      failed_when: false  # Handle failure manually
      
    - name: Handle API failure with circuit breaker
      block:
        - name: Check if circuit breaker should open
          set_fact:
            circuit_breaker_open: true
          when: 
            - api_result.status is defined
            - api_result.status >= 500
            - consecutive_failures | default(0) >= 3
            
        - name: Increment failure counter
          set_fact:
            consecutive_failures: "{{ consecutive_failures | default(0) | int + 1 }}"
          when: api_result.status >= 400
          
        - name: Reset failure counter on success
          set_fact:
            consecutive_failures: 0
          when: api_result.status < 400
          
      when: api_result.status >= 400
      
    - name: Use fallback mechanism when circuit breaker is open
      debug:
        msg: "Circuit breaker open - using local deployment method"
      when: circuit_breaker_open | default(false)
```

### Health Check and Recovery
```yaml
# Health monitoring with automatic recovery
- name: Service health monitoring and recovery
  hosts: webservers
  
  vars:
    health_check_retries: 3
    recovery_attempts: 2
    
  tasks:
    - name: Comprehensive health check
      block:
        - name: Check service status
          service_facts:
          
        - name: Verify service is running
          assert:
            that:
              - ansible_facts.services['myapp.service'].state == 'running'
            fail_msg: "Service is not running"
            
        - name: Check application endpoint
          uri:
            url: "http://localhost:8080/health"
            method: GET
            timeout: 10
          register: health_response
          
        - name: Verify health response
          assert:
            that:
              - health_response.status == 200
              - health_response.json.status == 'healthy'
            fail_msg: "Application health check failed"
            
        - name: Check resource usage
          shell: |
            cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
            memory_usage=$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100.0}')
            echo "cpu:${cpu_usage},memory:${memory_usage}"
          register: resource_usage
          
        - name: Verify resource thresholds
          assert:
            that:
              - resource_usage.stdout.split(',')[0].split(':')[1] | float < 80
              - resource_usage.stdout.split(',')[1].split(':')[1] | float < 90
            fail_msg: "Resource usage too high"
            
      rescue:
        - name: Attempt service recovery
          include_tasks: service_recovery.yml
          vars:
            recovery_attempt: "{{ ansible_loop.index }}"
          loop: "{{ range(recovery_attempts) | list }}"
          loop_control:
            loop_var: ansible_loop
          when: not service_recovered | default(false)
```

### Service Recovery Tasks
```yaml
# service_recovery.yml - Reusable recovery tasks
---
- name: "Recovery attempt {{ recovery_attempt + 1 }}"
  block:
    - name: Stop problematic service
      service:
        name: myapp
        state: stopped
      ignore_errors: yes
      
    - name: Clear temporary files
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /tmp/myapp.pid
        - /tmp/myapp.lock
        - /var/log/myapp/error.log
      ignore_errors: yes
      
    - name: Check disk space
      shell: df -h /opt/myapp | tail -1 | awk '{print $5}' | sed 's/%//'
      register: disk_usage
      
    - name: Clean up logs if disk is full
      shell: |
        find /var/log/myapp -name "*.log" -mtime +7 -delete
        find /opt/myapp/temp -type f -mtime +1 -delete
      when: disk_usage.stdout | int > 85
      
    - name: Restart service
      service:
        name: myapp
        state: started
        
    - name: Wait for service to be ready
      wait_for:
        port: 8080
        host: localhost
        timeout: 60
        
    - name: Verify recovery
      uri:
        url: "http://localhost:8080/health"
        method: GET
        status_code: 200
      register: recovery_check
      
    - name: Mark recovery as successful
      set_fact:
        service_recovered: true
      when: recovery_check.status == 200
      
  rescue:
    - name: Log recovery failure
      debug:
        msg: "Recovery attempt {{ recovery_attempt + 1 }} failed"
        
    - name: Final failure after all attempts
      fail:
        msg: "Service recovery failed after {{ recovery_attempts }} attempts"
      when: recovery_attempt == (recovery_attempts - 1)
```

## Pattern 3: Transaction-like Operations

### Database Transaction Pattern
```yaml
# Database operations with rollback
- name: Database schema migration with rollback
  hosts: databases
  
  vars:
    migration_id: "{{ ansible_date_time.epoch }}"
    
  tasks:
    - name: Database migration with transaction safety
      block:
        - name: Create migration backup
          shell: |
            pg_dump -h localhost -U postgres myapp > /backup/pre_migration_{{ migration_id }}.sql
          register: backup_result
          
        - name: Verify backup was created
          stat:
            path: "/backup/pre_migration_{{ migration_id }}.sql"
          register: backup_check
          failed_when: not backup_check.stat.exists
          
        - name: Apply database migrations
          shell: |
            psql -h localhost -U postgres -d myapp -f /migrations/{{ item }}.sql
          loop:
            - "001_add_user_table"
            - "002_add_indexes"
            - "003_update_constraints"
          register: migration_results
          
        - name: Verify migration success
          shell: |
            psql -h localhost -U postgres -d myapp -c "SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 1;"
          register: current_version
          
        - name: Update application configuration
          template:
            src: app_config.j2
            dest: /opt/myapp/config.yml
            backup: yes
          register: config_update
          
        - name: Restart application
          service:
            name: myapp
            state: restarted
            
        - name: Verify application works with new schema
          uri:
            url: "http://localhost:8080/api/test"
            method: GET
            status_code: 200
          retries: 5
          delay: 10
          
      rescue:
        - name: Rollback database changes
          shell: |
            psql -h localhost -U postgres -d myapp < /backup/pre_migration_{{ migration_id }}.sql
          ignore_errors: yes
          
        - name: Restore application configuration
          copy:
            src: "{{ config_update.backup_file }}"
            dest: /opt/myapp/config.yml
            remote_src: yes
          when: config_update.backup_file is defined
          ignore_errors: yes
          
        - name: Restart application after rollback
          service:
            name: myapp
            state: restarted
          ignore_errors: yes
          
        - name: Clean up failed migration backup
          file:
            path: "/backup/pre_migration_{{ migration_id }}.sql"
            state: absent
          ignore_errors: yes
          
        - name: Fail with detailed error
          fail:
            msg: |
              Database migration failed and was rolled back.
              Migration ID: {{ migration_id }}
              Check logs for details.
              
      always:
        - name: Log migration attempt
          lineinfile:
            path: /var/log/migrations.log
            line: "{{ ansible_date_time.iso8601 }} - Migration {{ migration_id }}: {{ 'SUCCESS' if ansible_failed_result is not defined else 'FAILED' }}"
            create: yes
```

## Pattern 4: Multi-Environment Error Handling

### Environment-Specific Error Tolerance
```yaml
# Different error handling per environment
- name: Environment-aware error handling
  hosts: all
  
  vars:
    environment_config:
      production:
        max_failures: 0
        retry_count: 5
        rollback_enabled: true
        monitoring_required: true
      staging:
        max_failures: 1
        retry_count: 3
        rollback_enabled: true
        monitoring_required: false
      development:
        max_failures: 3
        retry_count: 1
        rollback_enabled: false
        monitoring_required: false
        
    current_env_config: "{{ environment_config[environment] }}"
    
  tasks:
    - name: Deploy with environment-specific error handling
      block:
        - name: Validate environment configuration
          assert:
            that:
              - environment in environment_config.keys()
              - current_env_config is defined
            fail_msg: "Invalid environment: {{ environment }}"
            
        - name: Deploy application
          include_tasks: deploy_application.yml
          vars:
            max_retries: "{{ current_env_config.retry_count }}"
            
        - name: Run post-deployment tests
          include_tasks: run_tests.yml
          vars:
            test_suite: "{{ 'full' if environment == 'production' else 'basic' }}"
            
      rescue:
        - name: Handle deployment failure
          include_tasks: handle_deployment_failure.yml
          vars:
            rollback_enabled: "{{ current_env_config.rollback_enabled }}"
            
        - name: Determine if failure is acceptable
          set_fact:
            deployment_acceptable: "{{ current_env_config.max_failures > 0 }}"
            
        - name: Fail if not acceptable in this environment
          fail:
            msg: "Deployment failure not acceptable in {{ environment }} environment"
          when: not deployment_acceptable
          
        - name: Continue with degraded functionality
          debug:
            msg: "Continuing with degraded functionality in {{ environment }} environment"
          when: deployment_acceptable
          
      always:
        - name: Send monitoring alerts
          uri:
            url: "{{ monitoring_webhook }}"
            method: POST
            body_format: json
            body:
              environment: "{{ environment }}"
              status: "{{ 'success' if ansible_failed_result is not defined else 'failure' }}"
              timestamp: "{{ ansible_date_time.iso8601 }}"
          when: current_env_config.monitoring_required
          ignore_errors: yes
```

## Pattern 5: Performance and Resource Management

### Resource Constraint Handling
```yaml
# Handle resource constraints gracefully
- name: Resource-aware deployment
  hosts: all
  
  tasks:
    - name: Check system resources before deployment
      block:
        - name: Check available memory
          shell: free -m | grep '^Mem:' | awk '{print $7}'
          register: available_memory
          
        - name: Check available disk space
          shell: df -BG /opt | tail -1 | awk '{print $4}' | sed 's/G//'
          register: available_disk
          
        - name: Check CPU load
          shell: uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//'
          register: cpu_load
          
        - name: Determine deployment strategy based on resources
          set_fact:
            deployment_strategy: |
              {% if available_memory.stdout | int < 1024 %}
              minimal
              {% elif available_disk.stdout | int < 5 %}
              compact
              {% elif cpu_load.stdout | float > 2.0 %}
              low_priority
              {% else %}
              full
              {% endif %}
              
        - name: Deploy based on available resources
          include_tasks: "deploy_{{ deployment_strategy }}.yml"
          
      rescue:
        - name: Fallback to minimal deployment
          include_tasks: deploy_minimal.yml
          
        - name: Schedule resource cleanup
          cron:
            name: "cleanup_resources"
            minute: "0"
            hour: "2"
            job: "/opt/scripts/cleanup.sh"
            
      always:
        - name: Log resource usage
          lineinfile:
            path: /var/log/resource_usage.log
            line: "{{ ansible_date_time.iso8601 }} - Memory: {{ available_memory.stdout }}MB, Disk: {{ available_disk.stdout }}GB, Load: {{ cpu_load.stdout }}"
            create: yes
```

## Best Practices Summary

### Error Handling Strategy:
1. **Fail Fast** - Detect errors early in the process
2. **Graceful Degradation** - Provide fallback functionality
3. **Circuit Breaker** - Prevent cascading failures
4. **Retry Logic** - Handle transient failures
5. **Resource Monitoring** - Prevent resource exhaustion

### Performance Considerations:
1. **Async Operations** - Use async for long-running tasks
2. **Batch Processing** - Process hosts in manageable groups
3. **Resource Awareness** - Adapt to available resources
4. **Timeout Management** - Set appropriate timeouts
5. **Connection Pooling** - Reuse connections when possible

### Monitoring and Observability:
1. **Comprehensive Logging** - Log all operations and outcomes
2. **Metrics Collection** - Track performance and error rates
3. **Alerting** - Notify on critical failures
4. **Health Checks** - Continuous monitoring of service health
5. **Audit Trails** - Track all changes and operations

### Security in Error Handling:
1. **Sensitive Data** - Don't log sensitive information in errors
2. **Error Messages** - Provide helpful but not revealing messages
3. **Cleanup** - Ensure cleanup doesn't leave sensitive data
4. **Access Control** - Restrict access to error logs and recovery tools
5. **Audit Logging** - Log security-relevant errors and recoveries

---

**Note**: This reference covers advanced error handling patterns for experienced users. Start with basic error handling and gradually incorporate these advanced techniques as your automation complexity and reliability requirements grow.

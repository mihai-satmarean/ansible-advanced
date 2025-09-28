# Lab 2.4: Execution Strategies and Performance

## Objective
Optimize playbook execution with different strategies, parallelism, and performance tuning techniques.

## Duration
25 minutes

## Prerequisites
- Completed previous Module 2 labs
- Understanding of Ansible execution concepts

## Lab Setup

```bash
cd ~/ansible-labs/module-02
mkdir -p lab-2.4
cd lab-2.4

cat > inventory.ini << EOF
[webservers]
web01 ansible_host=localhost ansible_connection=local
web02 ansible_host=localhost ansible_connection=local
web03 ansible_host=localhost ansible_connection=local

[databases]
db01 ansible_host=localhost ansible_connection=local
db02 ansible_host=localhost ansible_connection=local

[loadbalancers]
lb01 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Execution Strategies Comparison (10 minutes)

### Task: Compare different execution strategies and their impact

Create `execution-strategies.yml`:
```yaml
---
# Play 1: Linear strategy (default)
- name: Linear execution strategy
  hosts: webservers
  strategy: linear
  gather_facts: no
  vars:
    strategy_name: "linear"
    
  tasks:
    - name: "{{ strategy_name | title }} - Start timestamp"
      debug:
        msg: "{{ inventory_hostname }} starting at {{ ansible_date_time.time }}"
    
    - name: "{{ strategy_name | title }} - Simulate work (varying duration)"
      pause:
        seconds: "{{ range(2, 8) | random }}"
    
    - name: "{{ strategy_name | title }} - Task 1"
      debug:
        msg: "{{ inventory_hostname }} executing task 1 at {{ ansible_date_time.time }}"
    
    - name: "{{ strategy_name | title }} - Task 2"
      debug:
        msg: "{{ inventory_hostname }} executing task 2 at {{ ansible_date_time.time }}"
    
    - name: "{{ strategy_name | title }} - End timestamp"
      debug:
        msg: "{{ inventory_hostname }} completed at {{ ansible_date_time.time }}"

# Play 2: Free strategy
- name: Free execution strategy
  hosts: webservers
  strategy: free
  gather_facts: no
  vars:
    strategy_name: "free"
    
  tasks:
    - name: "{{ strategy_name | title }} - Start timestamp"
      debug:
        msg: "{{ inventory_hostname }} starting at {{ ansible_date_time.time }}"
    
    - name: "{{ strategy_name | title }} - Simulate work (varying duration)"
      pause:
        seconds: "{{ range(2, 8) | random }}"
    
    - name: "{{ strategy_name | title }} - Task 1"
      debug:
        msg: "{{ inventory_hostname }} executing task 1 at {{ ansible_date_time.time }}"
    
    - name: "{{ strategy_name | title }} - Task 2"
      debug:
        msg: "{{ inventory_hostname }} executing task 2 at {{ ansible_date_time.time }}"
    
    - name: "{{ strategy_name | title }} - End timestamp"
      debug:
        msg: "{{ inventory_hostname }} completed at {{ ansible_date_time.time }}"

# Play 3: Serial execution with batching
- name: Serial execution with batching
  hosts: all
  serial: 2  # Process 2 hosts at a time
  gather_facts: no
  vars:
    strategy_name: "serial"
    
  tasks:
    - name: "{{ strategy_name | title }} - Display batch information"
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Batch: {{ ansible_play_batch }}
          Batch size: {{ ansible_play_batch | length }}
          Total hosts: {{ ansible_play_hosts_all | length }}
    
    - name: "{{ strategy_name | title }} - Simulate deployment"
      pause:
        seconds: 3
        prompt: "Deploying to {{ inventory_hostname }} in batch {{ ansible_play_batch }}"
    
    - name: "{{ strategy_name | title }} - Verify deployment"
      debug:
        msg: "{{ inventory_hostname }} deployment verified in batch {{ ansible_play_batch }}"

# Play 4: Performance comparison with timing
- name: Performance measurement
  hosts: webservers
  gather_facts: no
  vars:
    tasks_to_execute:
      - name: "cpu_intensive"
        duration: 2
      - name: "io_intensive"
        duration: 1
      - name: "network_check"
        duration: 1
      - name: "validation"
        duration: 1
        
  tasks:
    - name: Record start time
      set_fact:
        start_time: "{{ ansible_date_time.epoch }}"
    
    - name: Execute performance test tasks
      shell: |
        echo "Executing {{ item.name }} on {{ inventory_hostname }}"
        sleep {{ item.duration }}
        echo "{{ item.name }} completed on {{ inventory_hostname }}"
      loop: "{{ tasks_to_execute }}"
      register: task_results
    
    - name: Record end time
      set_fact:
        end_time: "{{ ansible_date_time.epoch }}"
    
    - name: Calculate execution time
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Total execution time: {{ end_time | int - start_time | int }} seconds
          Tasks completed: {{ task_results.results | length }}
```

### Run execution strategies comparison:
```bash
# Run all strategies and observe timing differences
echo "=== Running execution strategies comparison ==="
time ansible-playbook -i inventory.ini execution-strategies.yml

# Run with different fork settings
echo "=== Running with increased forks ==="
time ansible-playbook -i inventory.ini execution-strategies.yml -f 10

# Run only specific plays
echo "=== Running only free strategy ==="
ansible-playbook -i inventory.ini execution-strategies.yml --tags "never" -e "ansible_play_name='Free execution strategy'"
```

## Exercise 2: Performance Optimization Techniques (8 minutes)

### Task: Implement performance optimization strategies

Create `performance-optimization.yml`:
```yaml
---
- name: Performance optimization techniques
  hosts: all
  gather_facts: yes
  vars:
    optimization_techniques:
      - name: "fact_caching"
        enabled: true
      - name: "pipelining"
        enabled: true
      - name: "connection_reuse"
        enabled: true
        
  tasks:
    # Optimize fact gathering
    - name: Gather only essential facts
      setup:
        gather_subset:
          - "!all"
          - "!any"
          - "network"
          - "hardware"
      when: optimization_techniques | selectattr('name', 'equalto', 'fact_caching') | selectattr('enabled') | list | length > 0
    
    # Batch operations for efficiency
    - name: Create multiple directories in batch
      file:
        path: "/tmp/perf-test/{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - "dir1"
        - "dir2"
        - "dir3"
        - "dir4"
        - "dir5"
    
    # Use efficient modules
    - name: Create multiple files efficiently
      copy:
        content: |
          # Performance test file {{ item }}
          Host: {{ inventory_hostname }}
          Created: {{ ansible_date_time.iso8601 }}
          Optimization: batch_operations
        dest: "/tmp/perf-test/dir{{ item }}/file{{ item }}.txt"
        mode: '0644'
      loop: "{{ range(1, 6) | list }}"
    
    # Parallel task execution simulation
    - name: Simulate parallel operations
      shell: |
        # Simulate different types of operations
        case {{ item.type }} in
          "cpu")
            echo "CPU intensive task on {{ inventory_hostname }}"
            # Simulate CPU work
            python3 -c "sum(i*i for i in range(10000))"
            ;;
          "io")
            echo "I/O intensive task on {{ inventory_hostname }}"
            # Simulate I/O work
            dd if=/dev/zero of=/tmp/test-{{ item.id }}.tmp bs=1M count=10 2>/dev/null
            rm -f /tmp/test-{{ item.id }}.tmp
            ;;
          "network")
            echo "Network task on {{ inventory_hostname }}"
            # Simulate network check
            ping -c 3 localhost >/dev/null 2>&1 || true
            ;;
        esac
        echo "Task {{ item.id }} ({{ item.type }}) completed on {{ inventory_hostname }}"
      loop:
        - { id: 1, type: "cpu" }
        - { id: 2, type: "io" }
        - { id: 3, type: "network" }
        - { id: 4, type: "cpu" }
        - { id: 5, type: "io" }
      register: parallel_results
      async: 30  # Run asynchronously
      poll: 0    # Don't wait for completion
    
    # Wait for async tasks to complete
    - name: Wait for parallel operations to complete
      async_status:
        jid: "{{ item.ansible_job_id }}"
      register: async_results
      until: async_results.finished
      retries: 10
      delay: 3
      loop: "{{ parallel_results.results }}"
    
    # Performance monitoring
    - name: Collect performance metrics
      shell: |
        echo "=== Performance Metrics for {{ inventory_hostname }} ==="
        echo "CPU cores: $(nproc)"
        echo "Memory: $(free -h | awk 'NR==2{print $2}')"
        echo "Load average: $(uptime | awk -F'load average:' '{print $2}')"
        echo "Disk usage: $(df -h / | awk 'NR==2{print $5}')"
        echo "Active connections: $(ss -tuln | wc -l)"
      register: performance_metrics
      changed_when: false
    
    - name: Display performance summary
      debug:
        msg: |
          {{ performance_metrics.stdout }}
          
          Async tasks completed: {{ async_results | selectattr('finished') | list | length }}/{{ parallel_results.results | length }}
          
          Optimization techniques applied:
          {% for technique in optimization_techniques %}
          - {{ technique.name }}: {{ 'ENABLED' if technique.enabled else 'DISABLED' }}
          {% endfor %}
```

### Run performance optimization:
```bash
# Run performance optimization
time ansible-playbook -i inventory.ini performance-optimization.yml

# Run with different async settings
time ansible-playbook -i inventory.ini performance-optimization.yml -e "async_timeout=60"

# Check created files
ls -la /tmp/perf-test/*/
```

## Exercise 3: Advanced Execution Control (7 minutes)

### Task: Implement advanced execution control patterns

Create `advanced-execution-control.yml`:
```yaml
---
- name: Advanced execution control patterns
  hosts: all
  max_fail_percentage: 25  # Allow 25% of hosts to fail
  vars:
    deployment_phases:
      - name: "preparation"
        max_fail_percentage: 0  # No failures allowed in preparation
        tasks: ["system_check", "backup", "prerequisites"]
      - name: "deployment"
        max_fail_percentage: 10  # Allow 10% failure in deployment
        tasks: ["deploy_app", "configure", "start_services"]
      - name: "verification"
        max_fail_percentage: 50  # Allow 50% failure in verification
        tasks: ["health_check", "smoke_test", "performance_test"]
        
  tasks:
    # Phase 1: Preparation (strict)
    - name: "Phase 1: Preparation"
      block:
        - name: System compatibility check
          shell: |
            # Check system requirements
            if [ $(free -m | awk 'NR==2{print $2}') -lt 1024 ]; then
              echo "Insufficient memory"
              exit 1
            fi
            
            if [ $(nproc) -lt 1 ]; then
              echo "Insufficient CPU cores"
              exit 1
            fi
            
            echo "System check passed on {{ inventory_hostname }}"
          register: system_check
        
        - name: Create backup
          shell: |
            mkdir -p /tmp/backup-{{ ansible_date_time.epoch }}
            echo "Backup created for {{ inventory_hostname }}" > /tmp/backup-{{ ansible_date_time.epoch }}/backup.info
            echo "Backup completed on {{ inventory_hostname }}"
          register: backup_creation
        
        - name: Install prerequisites
          shell: |
            # Simulate prerequisite installation
            echo "Installing prerequisites on {{ inventory_hostname }}"
            # Simulate potential failure on specific host
            if [ "{{ inventory_hostname }}" = "web03" ] && [ "{{ simulate_prep_failure | default(false) }}" = "true" ]; then
              echo "Prerequisite installation failed"
              exit 1
            fi
            echo "Prerequisites installed on {{ inventory_hostname }}"
          register: prerequisites
      
      rescue:
        - name: Preparation phase failed
          debug:
            msg: "CRITICAL: Preparation failed on {{ inventory_hostname }}"
        
        - name: Stop deployment on preparation failure
          meta: end_host
      
      tags: preparation
    
    # Phase 2: Deployment (moderate tolerance)
    - name: "Phase 2: Deployment"
      block:
        - name: Deploy application
          shell: |
            echo "Deploying application on {{ inventory_hostname }}"
            # Simulate deployment
            mkdir -p /tmp/app-deployment
            echo "App deployed on {{ inventory_hostname }} at $(date)" > /tmp/app-deployment/deploy.info
            
            # Simulate potential deployment failure
            if [ "{{ inventory_hostname }}" = "db02" ] && [ "{{ simulate_deploy_failure | default(false) }}" = "true" ]; then
              echo "Deployment failed"
              exit 1
            fi
            
            echo "Deployment successful on {{ inventory_hostname }}"
          register: app_deployment
        
        - name: Configure application
          copy:
            content: |
              # Application Configuration
              host={{ inventory_hostname }}
              deployed_at={{ ansible_date_time.iso8601 }}
              phase=deployment
              status=configured
            dest: "/tmp/app-deployment/config.conf"
            mode: '0644'
        
        - name: Start services
          shell: |
            echo "Starting services on {{ inventory_hostname }}"
            # Simulate service startup
            echo "Services started on {{ inventory_hostname }}" > /tmp/app-deployment/services.status
          register: service_startup
      
      rescue:
        - name: Deployment phase failed
          debug:
            msg: "WARNING: Deployment failed on {{ inventory_hostname }}, but continuing with other hosts"
        
        - name: Mark host as deployment failed
          set_fact:
            deployment_failed: true
      
      tags: deployment
    
    # Phase 3: Verification (high tolerance)
    - name: "Phase 3: Verification"
      block:
        - name: Health check
          shell: |
            echo "Running health check on {{ inventory_hostname }}"
            
            # Check if deployment was successful
            if [ -f "/tmp/app-deployment/deploy.info" ]; then
              echo "Health check passed on {{ inventory_hostname }}"
            else
              echo "Health check failed - no deployment found"
              exit 1
            fi
          register: health_check
          when: deployment_failed is not defined
        
        - name: Smoke test
          shell: |
            echo "Running smoke test on {{ inventory_hostname }}"
            # Simulate smoke test
            if [ -f "/tmp/app-deployment/config.conf" ]; then
              echo "Smoke test passed on {{ inventory_hostname }}"
            else
              echo "Smoke test failed"
              exit 1
            fi
          register: smoke_test
          when: deployment_failed is not defined
        
        - name: Performance test
          shell: |
            echo "Running performance test on {{ inventory_hostname }}"
            # Simulate performance test that might fail
            if [ "{{ inventory_hostname }}" = "lb01" ] && [ "{{ simulate_perf_failure | default(false) }}" = "true" ]; then
              echo "Performance test failed"
              exit 1
            fi
            echo "Performance test passed on {{ inventory_hostname }}"
          register: performance_test
          when: deployment_failed is not defined
      
      rescue:
        - name: Verification phase failed
          debug:
            msg: "INFO: Verification failed on {{ inventory_hostname }}, but this is acceptable"
        
        - name: Mark verification as failed but continue
          set_fact:
            verification_failed: true
      
      tags: verification
    
    # Summary
    - name: Deployment summary
      debug:
        msg: |
          Deployment Summary for {{ inventory_hostname }}:
          - Preparation: SUCCESS
          - Deployment: {{ 'FAILED' if deployment_failed is defined else 'SUCCESS' }}
          - Verification: {{ 'FAILED' if verification_failed is defined else 'SUCCESS' }}
          - Overall Status: {{ 'PARTIAL' if (deployment_failed is defined or verification_failed is defined) else 'SUCCESS' }}
      tags: summary
```

### Run advanced execution control:
```bash
# Run normal execution
ansible-playbook -i inventory.ini advanced-execution-control.yml

# Run with simulated failures
ansible-playbook -i inventory.ini advanced-execution-control.yml -e "simulate_prep_failure=true simulate_deploy_failure=true simulate_perf_failure=true"

# Run specific phases
ansible-playbook -i inventory.ini advanced-execution-control.yml --tags "preparation,deployment"

# Check deployment results
ls -la /tmp/app-deployment/ 2>/dev/null || echo "No deployment directory"
ls -la /tmp/backup-*/ 2>/dev/null || echo "No backup directories"
```

## Verification and Discussion

### 1. Check Results
```bash
# Check performance test results
ls -la /tmp/perf-test/*/
cat /tmp/perf-test/dir1/file1.txt

# Check deployment results
ls -la /tmp/app-deployment/
cat /tmp/app-deployment/config.conf 2>/dev/null || echo "No config file"

# Check backup directories
ls -la /tmp/backup-*/
```

### 2. Discussion Points
- How do you optimize playbook execution in your current environment?
- What strategies do you use for handling partial failures in large deployments?
- How do you balance execution speed with reliability?

### 3. Clean Up
```bash
# Remove demo files
rm -rf /tmp/perf-test /tmp/app-deployment /tmp/backup-*
```

## Key Takeaways
- Different execution strategies (linear, free, serial) have distinct performance characteristics
- Async tasks enable parallel execution for improved performance
- Serial execution with batching provides controlled rolling deployments
- max_fail_percentage allows flexible failure tolerance
- Performance optimization requires balancing speed, reliability, and resource usage
- Advanced execution control enables sophisticated deployment patterns

## Next Steps
Proceed to Lab 2.5: Advanced Error Recovery Patterns

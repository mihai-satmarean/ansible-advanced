# Lab 1.6: Playbook Execution Methods

## Objective
Master different playbook execution options, debugging techniques, and optimization strategies for various deployment scenarios.

## Duration
15 minutes

## Prerequisites
- Completed all previous labs
- Understanding of Ansible command-line options

## Lab Setup

```bash
cd ~/ansible-labs/module-01
mkdir -p lab-1.6
cd lab-1.6

cat > inventory.ini << EOF
[webservers]
web01 ansible_host=localhost ansible_connection=local
web02 ansible_host=localhost ansible_connection=local

[databases]
db01 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Execution Options and Debugging (8 minutes)

### Task: Practice different execution methods and debugging techniques

Create `execution-demo.yml`:
```yaml
---
- name: Execution methods demonstration
  hosts: all
  gather_facts: yes
  vars:
    app_name: "execution-demo"
    deployment_version: "1.0.0"
    
  tasks:
    - name: Display host information
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Groups: {{ group_names | join(', ') }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          IP: {{ ansible_default_ipv4.address | default('N/A') }}
      tags: info
    
    - name: Create application directory
      file:
        path: "/tmp/{{ app_name }}"
        state: directory
        mode: '0755'
      tags: 
        - setup
        - directories
    
    - name: Generate configuration file
      copy:
        content: |
          # {{ app_name }} Configuration
          [application]
          name={{ app_name }}
          version={{ deployment_version }}
          host={{ inventory_hostname }}
          
          [deployment]
          timestamp={{ ansible_date_time.iso8601 }}
          user={{ ansible_user_id }}
          
          [environment]
          {% for group in group_names %}
          group_{{ loop.index }}={{ group }}
          {% endfor %}
        dest: "/tmp/{{ app_name }}/config.conf"
        mode: '0644'
      tags:
        - setup
        - configuration
    
    - name: Simulate potential failure point
      fail:
        msg: "Simulated failure for debugging demonstration"
      when: 
        - simulate_failure | default(false)
        - inventory_hostname == "web01"
      tags: failure_test
    
    - name: Create deployment marker
      copy:
        content: |
          Deployment completed successfully
          Host: {{ inventory_hostname }}
          Version: {{ deployment_version }}
          Time: {{ ansible_date_time.iso8601 }}
        dest: "/tmp/{{ app_name }}/deployment.marker"
        mode: '0644'
      tags:
        - setup
        - finalization
    
    - name: Display completion message
      debug:
        msg: "Deployment completed on {{ inventory_hostname }}"
      tags: info
```

### Practice different execution methods:

```bash
# 1. Normal execution
echo "=== Normal Execution ==="
ansible-playbook -i inventory.ini execution-demo.yml

# 2. Check mode (dry run)
echo "=== Check Mode (Dry Run) ==="
ansible-playbook -i inventory.ini execution-demo.yml --check

# 3. Diff mode (show changes)
echo "=== Diff Mode ==="
ansible-playbook -i inventory.ini execution-demo.yml --diff

# 4. Check + Diff mode
echo "=== Check + Diff Mode ==="
ansible-playbook -i inventory.ini execution-demo.yml --check --diff

# 5. Verbose execution
echo "=== Verbose Execution ==="
ansible-playbook -i inventory.ini execution-demo.yml -v

# 6. Extra verbose (connection info)
echo "=== Extra Verbose ==="
ansible-playbook -i inventory.ini execution-demo.yml -vv

# 7. Tag-based execution
echo "=== Tag-based Execution (info only) ==="
ansible-playbook -i inventory.ini execution-demo.yml --tags "info"

# 8. Skip tags
echo "=== Skip Tags (no setup) ==="
ansible-playbook -i inventory.ini execution-demo.yml --skip-tags "setup"

# 9. Limit to specific hosts
echo "=== Limited to webservers ==="
ansible-playbook -i inventory.ini execution-demo.yml --limit "webservers"

# 10. Step-by-step execution (interactive)
echo "=== Step-by-step Execution ==="
echo "Note: This requires manual confirmation for each task"
# ansible-playbook -i inventory.ini execution-demo.yml --step

# 11. Start at specific task
echo "=== Start at specific task ==="
ansible-playbook -i inventory.ini execution-demo.yml --start-at-task "Generate configuration file"

# 12. Simulate failure for debugging
echo "=== Simulate Failure ==="
ansible-playbook -i inventory.ini execution-demo.yml -e "simulate_failure=true" --tags "failure_test" || echo "Expected failure occurred"
```

## Exercise 2: Advanced Execution Strategies (4 minutes)

### Task: Implement advanced execution patterns

Create `advanced-execution.yml`:
```yaml
---
- name: Advanced execution strategies
  hosts: all
  serial: 1  # Process hosts one at a time
  max_fail_percentage: 25  # Allow 25% failure rate
  vars:
    batch_size: 2
    
  tasks:
    - name: Pre-deployment validation
      block:
        - name: Check system resources
          debug:
            msg: |
              System Check for {{ inventory_hostname }}:
              - Memory: {{ ansible_memtotal_mb }}MB
              - CPU cores: {{ ansible_processor_vcpus }}
              - Disk space: Available
          
        - name: Validate prerequisites
          stat:
            path: "/tmp"
          register: tmp_check
          failed_when: not tmp_check.stat.exists
      
      rescue:
        - name: Handle validation failure
          debug:
            msg: "Validation failed for {{ inventory_hostname }}, skipping deployment"
          
        - name: Mark host as failed
          set_fact:
            deployment_failed: true
      
      tags: validation
    
    - name: Deploy application (only if validation passed)
      block:
        - name: Create deployment directory
          file:
            path: "/tmp/advanced-deployment"
            state: directory
            mode: '0755'
        
        - name: Deploy application files
          copy:
            content: |
              # Advanced Deployment
              Host: {{ inventory_hostname }}
              Batch: {{ ansible_play_batch }}
              Serial: {{ ansible_play_hosts_all.index(inventory_hostname) + 1 }}
              Total hosts: {{ ansible_play_hosts_all | length }}
            dest: "/tmp/advanced-deployment/info.txt"
            mode: '0644'
        
        - name: Simulate deployment time
          pause:
            seconds: 2
            prompt: "Deploying to {{ inventory_hostname }}..."
        
        - name: Verify deployment
          stat:
            path: "/tmp/advanced-deployment/info.txt"
          register: deployment_check
          failed_when: not deployment_check.stat.exists
      
      when: deployment_failed is not defined
      tags: deployment
    
    - name: Post-deployment tasks
      debug:
        msg: |
          Deployment Summary for {{ inventory_hostname }}:
          - Status: {{ 'FAILED' if deployment_failed is defined else 'SUCCESS' }}
          - Processed in batch: {{ ansible_play_batch }}
          - Total runtime: {{ ansible_date_time.epoch | int - hostvars[inventory_hostname].ansible_date_time.epoch | int | default(0) }}s
      tags: summary

# Second play with different strategy
- name: Parallel execution example
  hosts: webservers
  strategy: free  # Allow hosts to run independently
  gather_facts: no
  
  tasks:
    - name: Parallel task execution
      debug:
        msg: "{{ inventory_hostname }} executing independently at {{ ansible_date_time.time }}"
    
    - name: Simulate varying execution times
      pause:
        seconds: "{{ range(1, 6) | random }}"
    
    - name: Report completion
      debug:
        msg: "{{ inventory_hostname }} completed parallel execution"
```

### Run advanced execution strategies:
```bash
# Run with serial execution
ansible-playbook -i inventory.ini advanced-execution.yml

# Run with different serial batch size
ansible-playbook -i inventory.ini advanced-execution.yml -e "serial=2"

# Run only validation
ansible-playbook -i inventory.ini advanced-execution.yml --tags "validation"

# Run with custom strategy
ansible-playbook -i inventory.ini advanced-execution.yml --tags "summary"
```

## Exercise 3: Debugging and Troubleshooting (3 minutes)

### Task: Practice debugging techniques

Create `debugging-demo.yml`:
```yaml
---
- name: Debugging demonstration
  hosts: localhost
  vars:
    debug_mode: true
    test_data:
      - name: "service1"
        port: 8080
        enabled: true
      - name: "service2"
        port: 8081
        enabled: false
      - name: "service3"
        port: 8082
        enabled: true
        
  tasks:
    - name: Debug variable inspection
      debug:
        var: test_data
      when: debug_mode | default(false)
      tags: debug
    
    - name: Debug with custom message
      debug:
        msg: |
          Debug Information:
          - Total services: {{ test_data | length }}
          - Enabled services: {{ test_data | selectattr('enabled') | list | length }}
          - Service ports: {{ test_data | map(attribute='port') | list }}
      when: debug_mode | default(false)
      tags: debug
    
    - name: Process services with debugging
      debug:
        msg: |
          Processing: {{ item.name }}
          Port: {{ item.port }}
          Enabled: {{ item.enabled }}
          Index: {{ ansible_loop.index }}
          First: {{ ansible_loop.first }}
          Last: {{ ansible_loop.last }}
      loop: "{{ test_data }}"
      loop_control:
        extended: yes
      when: debug_mode | default(false)
      tags: debug
    
    - name: Conditional debugging
      debug:
        msg: "Service {{ item.name }} is enabled and will be processed"
      loop: "{{ test_data }}"
      when: 
        - debug_mode | default(false)
        - item.enabled
      tags: debug
    
    - name: Register and debug task results
      shell: echo "Processing {{ item.name }} on port {{ item.port }}"
      loop: "{{ test_data }}"
      register: processing_results
      when: item.enabled
      tags: processing
    
    - name: Debug registered results
      debug:
        msg: |
          Task Results:
          {% for result in processing_results.results %}
          {% if not result.skipped | default(false) %}
          - {{ result.item.name }}: {{ result.stdout }}
          {% endif %}
          {% endfor %}
      when: debug_mode | default(false)
      tags: debug
```

### Practice debugging techniques:
```bash
# Run with debugging enabled
ansible-playbook -i inventory.ini debugging-demo.yml

# Run without debugging
ansible-playbook -i inventory.ini debugging-demo.yml -e "debug_mode=false"

# Run only debug tasks
ansible-playbook -i inventory.ini debugging-demo.yml --tags "debug"

# Run with maximum verbosity for troubleshooting
ansible-playbook -i inventory.ini debugging-demo.yml -vvvv --tags "processing"
```

## Verification and Discussion

### 1. Check Results
```bash
# Verify execution results
ls -la /tmp/execution-demo/
cat /tmp/execution-demo/config.conf

ls -la /tmp/advanced-deployment/
cat /tmp/advanced-deployment/info.txt
```

### 2. Discussion Points
- Which execution methods are most useful for your deployment scenarios?
- How do you currently debug Ansible playbook issues?
- What strategies do you use for rolling deployments?

### 3. Clean Up
```bash
# Remove demo files
rm -rf /tmp/execution-demo /tmp/advanced-deployment
```

## Key Takeaways
- Check mode enables safe testing without making changes
- Diff mode shows what changes will be made
- Verbose modes provide detailed execution information
- Tags enable selective task execution
- Serial execution controls deployment order and risk
- Debugging techniques help troubleshoot complex playbooks
- Different execution strategies suit different deployment needs

## Module 1 Summary

### What We Covered
1. **Modules, Tasks, and Playbooks** - Foundation components and organization
2. **Host Targeting and User Management** - Flexible targeting and user automation
3. **Task Organization and Dependencies** - Structured automation with proper flow
4. **State Tracking and Idempotency** - Reliable and repeatable automation
5. **Handlers and Notifications** - Event-driven service management
6. **Execution Methods** - Debugging, testing, and deployment strategies

### Key Skills Developed
- Organizing playbooks for maintainability
- Implementing idempotent automation
- Managing users and services effectively
- Using handlers for efficient service management
- Debugging and troubleshooting playbooks
- Choosing appropriate execution strategies

### Next Steps
You are now ready to proceed to advanced Ansible topics in the subsequent modules. The foundation skills from Module 1 will be essential for understanding and implementing the advanced concepts covered in Modules 2-9.

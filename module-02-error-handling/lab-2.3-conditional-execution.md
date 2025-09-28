# Lab 2.3: Conditional Execution and Flow Control

## Objective
Master conditional execution patterns and flow control mechanisms for intelligent automation workflows.

## Duration
20 minutes

## Prerequisites
- Completed Labs 2.1 and 2.2
- Understanding of Ansible conditionals

## Lab Setup

```bash
cd ~/ansible-labs/module-02
mkdir -p lab-2.3
cd lab-2.3

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

## Exercise 1: Advanced Conditional Patterns (8 minutes)

### Task: Implement sophisticated conditional logic

Create `conditional-patterns.yml`:
```yaml
---
- name: Advanced conditional execution patterns
  hosts: all
  vars:
    deployment_config:
      environment: "production"
      features:
        - name: "authentication"
          enabled: true
          required_for: ["production", "staging"]
        - name: "caching"
          enabled: true
          required_for: ["production"]
        - name: "debugging"
          enabled: false
          required_for: ["development"]
        - name: "monitoring"
          enabled: true
          required_for: ["production", "staging"]
    
    host_roles:
      web01: ["webserver", "loadbalancer"]
      web02: ["webserver"]
      db01: ["database", "backup"]
      
  tasks:
    # Multi-condition evaluation
    - name: Deploy features based on environment and host role
      debug:
        msg: |
          Deploying {{ item.name }} feature on {{ inventory_hostname }}
          Environment: {{ deployment_config.environment }}
          Host roles: {{ host_roles[inventory_hostname] | default(['unknown']) | join(', ') }}
          Feature required: {{ deployment_config.environment in item.required_for }}
      loop: "{{ deployment_config.features }}"
      when:
        - item.enabled | default(false)
        - deployment_config.environment in item.required_for
        - inventory_hostname in host_roles
    
    # Complex conditional with multiple criteria
    - name: Configure load balancer (complex conditions)
      copy:
        content: |
          # Load Balancer Configuration
          # Generated for {{ inventory_hostname }}
          
          upstream backend {
          {% for host in groups['webservers'] %}
          {% if host != inventory_hostname %}
            server {{ host }}:8080;
          {% endif %}
          {% endfor %}
          }
          
          server {
            listen 80;
            location / {
              proxy_pass http://backend;
            }
          }
        dest: "/tmp/{{ inventory_hostname }}-lb.conf"
        mode: '0644'
      when:
        - "'loadbalancer' in host_roles[inventory_hostname] | default([])"
        - groups['webservers'] | length > 1
        - deployment_config.environment in ['production', 'staging']
    
    # Conditional execution based on previous task results
    - name: Check system resources
      shell: |
        echo "memory_mb:$(free -m | awk 'NR==2{print $2}')"
        echo "cpu_cores:$(nproc)"
        echo "disk_gb:$(df -BG / | awk 'NR==2{print $2}' | sed 's/G//')"
      register: system_resources
      changed_when: false
    
    - name: Parse system resources
      set_fact:
        memory_mb: "{{ system_resources.stdout_lines | select('match', 'memory_mb:.*') | first | regex_replace('memory_mb:', '') | int }}"
        cpu_cores: "{{ system_resources.stdout_lines | select('match', 'cpu_cores:.*') | first | regex_replace('cpu_cores:', '') | int }}"
        disk_gb: "{{ system_resources.stdout_lines | select('match', 'disk_gb:.*') | first | regex_replace('disk_gb:', '') | int }}"
    
    - name: Deploy resource-intensive features conditionally
      debug:
        msg: |
          Deploying {{ item.name }} on {{ inventory_hostname }}
          Resource requirements met: {{ item.min_memory <= memory_mb and item.min_cpu <= cpu_cores }}
      loop:
        - name: "analytics-engine"
          min_memory: 4096
          min_cpu: 4
        - name: "cache-cluster"
          min_memory: 2048
          min_cpu: 2
        - name: "log-aggregator"
          min_memory: 1024
          min_cpu: 1
      when:
        - item.min_memory <= memory_mb
        - item.min_cpu <= cpu_cores
        - "'webserver' in host_roles[inventory_hostname] | default([])"
    
    # Conditional execution with error handling
    - name: Attempt advanced configuration with fallback
      block:
        - name: Try advanced configuration
          copy:
            content: |
              # Advanced Configuration for {{ inventory_hostname }}
              [advanced]
              enabled=true
              features={{ deployment_config.features | selectattr('enabled') | map(attribute='name') | join(',') }}
              
              [resources]
              memory={{ memory_mb }}MB
              cpu={{ cpu_cores }} cores
              disk={{ disk_gb }}GB
            dest: "/tmp/{{ inventory_hostname }}-advanced.conf"
            mode: '0644'
          when:
            - memory_mb >= 2048
            - cpu_cores >= 2
            - deployment_config.environment == "production"
        
        - name: Verify advanced configuration
          stat:
            path: "/tmp/{{ inventory_hostname }}-advanced.conf"
          register: advanced_config_check
        
        - name: Fail if advanced config expected but not created
          fail:
            msg: "Advanced configuration was expected but not created"
          when:
            - deployment_config.environment == "production"
            - memory_mb >= 2048
            - cpu_cores >= 2
            - not advanced_config_check.stat.exists
      
      rescue:
        - name: Fall back to basic configuration
          copy:
            content: |
              # Basic Configuration for {{ inventory_hostname }}
              [basic]
              enabled=true
              mode=fallback
              
              [resources]
              memory={{ memory_mb }}MB
              cpu={{ cpu_cores }} cores
              
              [note]
              advanced_config_failed=true
              fallback_reason=insufficient_resources_or_environment
            dest: "/tmp/{{ inventory_hostname }}-basic.conf"
            mode: '0644'
```

### Run conditional patterns:
```bash
# Run conditional patterns
ansible-playbook -i inventory.ini conditional-patterns.yml

# Check generated configurations
ls -la /tmp/*-*.conf
cat /tmp/web01-lb.conf 2>/dev/null || echo "No load balancer config"
cat /tmp/web01-advanced.conf 2>/dev/null || cat /tmp/web01-basic.conf 2>/dev/null || echo "No config files"
```

## Exercise 2: Flow Control with Loops and Conditions (7 minutes)

### Task: Implement complex flow control patterns

Create `flow-control.yml`:
```yaml
---
- name: Advanced flow control patterns
  hosts: localhost
  vars:
    services:
      - name: "database"
        priority: 1
        dependencies: []
        health_check: "pg_isready -h localhost"
        critical: true
      - name: "cache"
        priority: 2
        dependencies: []
        health_check: "redis-cli ping"
        critical: false
      - name: "api"
        priority: 3
        dependencies: ["database"]
        health_check: "curl -f http://localhost:8080/health"
        critical: true
      - name: "web"
        priority: 4
        dependencies: ["api", "cache"]
        health_check: "curl -f http://localhost:3000/health"
        critical: true
      - name: "monitoring"
        priority: 5
        dependencies: ["api"]
        health_check: "curl -f http://localhost:9090/health"
        critical: false
        
  tasks:
    # Ordered execution with dependency checking
    - name: Deploy services in dependency order
      block:
        - name: "Deploy {{ item.name }} service"
          debug:
            msg: |
              Deploying {{ item.name }}:
              - Priority: {{ item.priority }}
              - Dependencies: {{ item.dependencies | join(', ') if item.dependencies else 'None' }}
              - Critical: {{ item.critical }}
          
          # Check dependencies before deployment
        - name: "Check dependencies for {{ item.name }}"
          debug:
            msg: "Dependency {{ dep }} is required for {{ item.name }}"
          loop: "{{ item.dependencies }}"
          loop_control:
            loop_var: dep
          when: item.dependencies | length > 0
        
        - name: "Create {{ item.name }} service configuration"
          copy:
            content: |
              # {{ item.name | title }} Service Configuration
              [service]
              name={{ item.name }}
              priority={{ item.priority }}
              critical={{ item.critical | lower }}
              
              [dependencies]
              {% for dep in item.dependencies %}
              requires={{ dep }}
              {% endfor %}
              
              [health]
              check_command={{ item.health_check }}
              
              [deployment]
              deployed_at={{ ansible_date_time.iso8601 }}
              status=configured
            dest: "/tmp/{{ item.name }}-service.conf"
            mode: '0644'
      
      loop: "{{ services | sort(attribute='priority') }}"
      when: item.critical or (not item.critical and deploy_optional | default(true))
    
    # Conditional loops with break-like behavior
    - name: Find first available port
      shell: |
        for port in 8080 8081 8082 8083 8084; do
          if ! netstat -tuln | grep -q ":$port "; then
            echo "available_port:$port"
            exit 0
          fi
        done
        echo "available_port:none"
        exit 1
      register: port_check
      failed_when: false
      changed_when: false
    
    - name: Parse available port
      set_fact:
        available_port: "{{ port_check.stdout | regex_replace('available_port:', '') }}"
    
    - name: Configure service with available port
      copy:
        content: |
          # Dynamic Port Configuration
          [server]
          port={{ available_port }}
          status={{ 'available' if available_port != 'none' else 'no_ports_available' }}
          
          [scan_result]
          checked_ports=[8080, 8081, 8082, 8083, 8084]
          selected_port={{ available_port }}
        dest: "/tmp/dynamic-port.conf"
        mode: '0644'
      when: available_port != "none"
    
    # Nested loops with conditions
    - name: Configure service endpoints
      copy:
        content: |
          # {{ outer_item.name }} Endpoints Configuration
          [endpoints]
          {% for endpoint in outer_item.endpoints %}
          {% if endpoint.enabled | default(true) %}
          {{ endpoint.path }}={{ endpoint.method | default('GET') }}:{{ endpoint.port | default(outer_item.default_port) }}
          {% endif %}
          {% endfor %}
          
          [security]
          {% for endpoint in outer_item.endpoints %}
          {% if endpoint.auth_required | default(false) %}
          {{ endpoint.path }}_auth=required
          {% endif %}
          {% endfor %}
        dest: "/tmp/{{ outer_item.name }}-endpoints.conf"
        mode: '0644'
      loop:
        - name: "api-service"
          default_port: 8080
          endpoints:
            - path: "/health"
              method: "GET"
              enabled: true
              auth_required: false
            - path: "/users"
              method: "GET"
              enabled: true
              auth_required: true
            - path: "/admin"
              method: "POST"
              enabled: false
              auth_required: true
        - name: "web-service"
          default_port: 3000
          endpoints:
            - path: "/"
              method: "GET"
              enabled: true
              auth_required: false
            - path: "/dashboard"
              method: "GET"
              enabled: true
              auth_required: true
      loop_control:
        loop_var: outer_item
      when: outer_item.endpoints | selectattr('enabled', 'equalto', true) | list | length > 0
```

### Run flow control patterns:
```bash
# Run flow control patterns
ansible-playbook -i inventory.ini flow-control.yml

# Run with optional services disabled
ansible-playbook -i inventory.ini flow-control.yml -e "deploy_optional=false"

# Check generated configurations
ls -la /tmp/*-service.conf
cat /tmp/dynamic-port.conf
cat /tmp/api-service-endpoints.conf
```

## Exercise 3: State-Based Execution Control (5 minutes)

### Task: Implement state-based execution control

Create `state-based-control.yml`:
```yaml
---
- name: State-based execution control
  hosts: localhost
  vars:
    application_state_file: "/tmp/app-state.json"
    
  tasks:
    # Initialize or read current state
    - name: Read current application state
      slurp:
        src: "{{ application_state_file }}"
      register: current_state_raw
      failed_when: false
    
    - name: Parse current state
      set_fact:
        current_state: "{{ current_state_raw.content | b64decode | from_json }}"
      when: 
        - current_state_raw is succeeded
        - current_state_raw.content is defined
    
    - name: Initialize state if not exists
      set_fact:
        current_state:
          version: "0.0.0"
          services: {}
          last_deployment: "never"
          status: "new"
      when: current_state is not defined
    
    # State-based conditional execution
    - name: Deploy based on current state
      block:
        - name: Deploy new version (first time deployment)
          debug:
            msg: "Performing initial deployment"
          when: current_state.status == "new"
        
        - name: Upgrade existing deployment
          debug:
            msg: |
              Upgrading from version {{ current_state.version }}
              Last deployment: {{ current_state.last_deployment }}
          when: 
            - current_state.status != "new"
            - current_state.version is version('1.0.0', '<')
        
        - name: Skip deployment (already current)
          debug:
            msg: "Application is already at the latest version {{ current_state.version }}"
          when:
            - current_state.status != "new"
            - current_state.version is version('1.0.0', '>=')
        
        # Update state based on actions taken
        - name: Update application state
          copy:
            content: |
              {
                "version": "1.0.0",
                "services": {
                  "web": {
                    "status": "deployed",
                    "port": 3000,
                    "health": "unknown"
                  },
                  "api": {
                    "status": "deployed", 
                    "port": 8080,
                    "health": "unknown"
                  }
                },
                "last_deployment": "{{ ansible_date_time.iso8601 }}",
                "status": "deployed",
                "deployment_count": {{ (current_state.deployment_count | default(0)) + 1 }}
              }
            dest: "{{ application_state_file }}"
            mode: '0644'
          when: 
            - current_state.status == "new" or current_state.version is version('1.0.0', '<')
    
    # Conditional service management based on state
    - name: Manage services based on state
      debug:
        msg: |
          Service: {{ item.key }}
          Current Status: {{ item.value.status }}
          Action: {{ 'restart' if item.value.status == 'deployed' else 'start' }}
      loop: "{{ current_state.services | dict2items }}"
      when: 
        - current_state.services is defined
        - item.value.status in ['deployed', 'running']
    
    # State validation and cleanup
    - name: Validate final state
      slurp:
        src: "{{ application_state_file }}"
      register: final_state_raw
    
    - name: Display final state
      debug:
        msg: |
          Final Application State:
          {{ final_state_raw.content | b64decode | from_json | to_nice_json }}
```

### Run state-based control:
```bash
# First run - should perform initial deployment
ansible-playbook -i inventory.ini state-based-control.yml

# Second run - should skip deployment (already current)
ansible-playbook -i inventory.ini state-based-control.yml

# Check state file
cat /tmp/app-state.json | python3 -m json.tool
```

## Verification and Discussion

### 1. Check Results
```bash
# Check conditional patterns results
ls -la /tmp/*-*.conf
cat /tmp/web01-lb.conf 2>/dev/null || echo "No load balancer config generated"

# Check flow control results
ls -la /tmp/*-service.conf
cat /tmp/dynamic-port.conf

# Check state-based control results
cat /tmp/app-state.json | python3 -m json.tool 2>/dev/null || cat /tmp/app-state.json
```

### 2. Discussion Points
- How do you implement complex conditional logic in your current automation?
- What strategies do you use for state management in long-running deployments?
- How do you handle dependency ordering in your service deployments?

### 3. Clean Up
```bash
# Remove demo files
rm -f /tmp/*-*.conf /tmp/app-state.json
```

## Key Takeaways
- Complex conditionals enable intelligent automation decisions
- Flow control with loops and conditions provides flexible execution patterns
- State-based execution prevents unnecessary work and enables incremental deployments
- Dependency ordering ensures proper service startup sequences
- Resource-based conditionals enable adaptive deployments
- State persistence enables resumable and incremental automation

## Next Steps
Proceed to Lab 2.4: Execution Strategies and Performance

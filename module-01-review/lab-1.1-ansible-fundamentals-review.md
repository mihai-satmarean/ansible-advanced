# Lab 1.1: Ansible Fundamentals Review (Interactive Demo)

## Objective
Quick review of essential Ansible concepts through interactive demonstration and practical examples suitable for mixed skill levels.

## Duration
30 minutes (interactive demo)

## Format
**Interactive demonstration** - Instructor-led with participant practice

## Prerequisites
- Ansible installed on control machine
- Basic understanding of YAML and command line

## Demo Setup

```bash
cd ~/ansible-labs
mkdir -p module-01
cd module-01

# Create simple inventory for review
cat > inventory.ini << 'EOF'
[demo_hosts]
localhost ansible_connection=local

[webservers]
localhost ansible_connection=local

[databases]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Demo 1: Modules, Tasks, and Playbooks (10 minutes)

### Quick Ansible Concepts Review
**Instructor explains:**
- **Modules**: Units of work (file, copy, service, package, etc.)
- **Tasks**: Calls to modules with parameters
- **Playbooks**: YAML files containing plays and tasks
- **Plays**: Map tasks to hosts

### Basic Module Usage
```yaml
# Create basic-review.yml
cat > basic-review.yml << 'EOF'
---
- name: Ansible Fundamentals Review
  hosts: demo_hosts
  gather_facts: yes
  
  tasks:
    - name: Show system information
      debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_os_family }}
          Python: {{ ansible_python_version }}
          Task var: {{ var_from_this_task }}
      vars:
        var_from_this_task: "Ansble advanced 2025"
         
    
    - name: Create directory structure
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /tmp/ansible-demo
        - /tmp/ansible-demo/config
        - /tmp/ansible-demo/logs
    
    - name: Create configuration file
      copy:
        content: |
          # Demo Configuration
          # Generated: {{ ansible_date_time.iso8601 }}
          
          [application]
          name=demo-app
          version=1.0.0
          environment=development
          
          [logging]
          level=INFO
          file=/tmp/ansible-demo/logs/app.log
        dest: /tmp/ansible-demo/config/app.conf
        
    - name: Verify file was created
      stat:
        path: /tmp/ansible-demo/config/app.conf
      register: config_file
      
    - name: Show file information
      debug:
        msg: |
          File exists: {{ config_file.stat.exists }}
          File size: {{ config_file.stat.size }} bytes
          Modified: {{ config_file.stat.mtime }}
EOF

# Run the review playbook
ansible-playbook -i inventory.ini basic-review.yml
```

### Variables and Facts
```yaml
# Create variables-demo.yml
cat > variables-demo.yml << 'EOF'
---
- name: Variables and Facts Demo
  hosts: demo_hosts
  
  vars:
    app_name: "review-app"
    app_version: "2.1.0"
    environments:
      - development
      - staging
      - production
      
  tasks:
    - name: Show variable usage
      debug:
        msg: |
          Application: {{ app_name }}
          Version: {{ app_version }}
          Environments: {{ environments | join(', ') }}
    
    - name: Show useful facts
      debug:
        msg: |
          Architecture: {{ ansible_architecture }}
          Processor cores: {{ ansible_processor_cores }}
          Memory (MB): {{ ansible_memtotal_mb }}
          Disk space: {{ ansible_mounts[0].size_total // (1024*1024*1024) }}GB
          
    - name: Create environment-specific configs
      copy:
        content: |
          # {{ env | title }} Environment Configuration
          [app]
          name={{ app_name }}
          version={{ app_version }}
          environment={{ env }}
          debug={{ 'true' if env == 'development' else 'false' }}
        dest: "/tmp/ansible-demo/config/{{ env }}.conf"
      loop: "{{ environments }}"
      loop_control:
        loop_var: env
EOF

# Run variables demo
ansible-playbook -i inventory.ini variables-demo.yml
ls -la /tmp/ansible-demo/config/
```

## Demo 2: Host Targeting and Inventory (8 minutes)

### Host Patterns Review
```yaml
# Create host-patterns-demo.yml
cat > host-patterns-demo.yml << 'EOF'
---
# Play 1: Target specific group
- name: Target webservers only
  hosts: webservers
  gather_facts: no
  
  tasks:
    - name: Web server specific task
      debug:
        msg: "This runs only on webservers: {{ inventory_hostname }}"

# Play 2: Target multiple groups
- name: Target webservers and databases
  hosts: webservers:databases
  gather_facts: no
  
  tasks:
    - name: Multi-group task
      debug:
        msg: "This runs on web and db servers: {{ inventory_hostname }}"

# Play 3: Target all hosts
- name: Target all hosts
  hosts: all
  gather_facts: no
  
  tasks:
    - name: Universal task
      debug:
        msg: "This runs everywhere: {{ inventory_hostname }} in group {{ group_names }}"
EOF

# Run host patterns demo
ansible-playbook -i inventory.ini host-patterns-demo.yml
```

### Inventory Variables
```bash
# Create group_vars directory
mkdir -p group_vars host_vars

# Create group variables
cat > group_vars/webservers.yml << 'EOF'
# Web server specific variables
http_port: 80
https_port: 443
max_connections: 1000
service_name: nginx
EOF

cat > group_vars/databases.yml << 'EOF'
# Database specific variables
db_port: 5432
max_connections: 200
service_name: postgresql
backup_enabled: true
EOF

# Create host-specific variables
cat > host_vars/localhost.yml << 'EOF'
# Localhost specific variables
local_user: "{{ ansible_user_id }}"
local_home: "{{ ansible_env.HOME }}"
EOF
```

```yaml
# Create inventory-vars-demo.yml
cat > inventory-vars-demo.yml << 'EOF'
---
- name: Inventory Variables Demo
  hosts: all
  gather_facts: yes
  
  tasks:
    - name: Show group-specific variables
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Groups: {{ group_names }}
          Service: {{ service_name | default('none') }}
          Port: {{ http_port | default(db_port) | default('N/A') }}
          Max Connections: {{ max_connections | default('N/A') }}
          
    - name: Show host-specific variables
      debug:
        msg: |
          Local user: {{ local_user | default('N/A') }}
          Local home: {{ local_home | default('N/A') }}
      when: inventory_hostname == 'localhost'
EOF

# Run inventory variables demo
ansible-playbook -i inventory.ini inventory-vars-demo.yml
```

## Demo 3: Handlers and Task Organization (12 minutes)

### Basic Handlers
```yaml
# Create handlers-demo.yml
cat > handlers-demo.yml << 'EOF'
---
- name: Handlers Demo
  hosts: demo_hosts
  
  vars:
    service_name: "demo-service"
    
  tasks:
    - name: Create service script
      copy:
        content: |
          #!/bin/bash
          # {{ service_name }} service script
          echo "Service {{ service_name }} is running"
          echo "PID: $$"
          sleep 300  # Simulate long-running service
        dest: "/tmp/{{ service_name }}.sh"
        mode: '0755'
      notify: restart service
      
    - name: Create service configuration
      copy:
        content: |
          # {{ service_name }} Configuration
          # Generated: {{ ansible_date_time.iso8601 }}
          
          [service]
          name={{ service_name }}
          user={{ ansible_user_id }}
          working_dir=/tmp
          
          [logging]
          enabled=true
          level=INFO
        dest: "/tmp/{{ service_name }}.conf"
      notify:
        - validate config
        - restart service
        
    - name: Show service files
      debug:
        msg: "Service files created for {{ service_name }}"
        
  handlers:
    - name: validate config
      shell: |
        echo "Validating configuration for {{ service_name }}"
        # Simulate config validation
        if grep -q "name={{ service_name }}" /tmp/{{ service_name }}.conf; then
          echo "Configuration valid"
        else
          echo "Configuration invalid"
          exit 1
        fi
      listen: validate config
      
    - name: restart service
      debug:
        msg: "Restarting {{ service_name }} service (simulated)"
      listen: restart service
EOF

# Run handlers demo
ansible-playbook -i inventory.ini handlers-demo.yml
```

### Task Organization with Blocks
```yaml
# Create task-organization-demo.yml
cat > task-organization-demo.yml << 'EOF'
---
- name: Task Organization Demo
  hosts: demo_hosts
  
  vars:
    app_name: "organized-app"
    
  tasks:
    - name: Application deployment block
      block:
        - name: Create application directory
          file:
            path: "/tmp/{{ app_name }}"
            state: directory
            
        - name: Download application (simulate)
          copy:
            content: |
              #!/bin/bash
              # {{ app_name }} Application
              echo "Starting {{ app_name }}..."
              echo "Application running on PID: $$"
            dest: "/tmp/{{ app_name }}/app.sh"
            mode: '0755'
            
        - name: Create application config
          copy:
            content: |
              [app]
              name={{ app_name }}
              version=1.0.0
              
              [runtime]
              user={{ ansible_user_id }}
              working_dir=/tmp/{{ app_name }}
            dest: "/tmp/{{ app_name }}/config.ini"
            
        - name: Verify deployment
          stat:
            path: "/tmp/{{ app_name }}/app.sh"
          register: app_file
          failed_when: not app_file.stat.exists
          
      rescue:
        - name: Handle deployment failure
          debug:
            msg: "Deployment failed, cleaning up..."
            
        - name: Remove failed deployment
          file:
            path: "/tmp/{{ app_name }}"
            state: absent
            
      always:
        - name: Log deployment attempt
          lineinfile:
            path: /tmp/deployment.log
            line: "{{ ansible_date_time.iso8601 }} - {{ app_name }} deployment: {{ 'SUCCESS' if app_file.stat.exists else 'FAILED' }}"
            create: yes
            
    - name: Show deployment log
      debug:
        msg: "Check deployment log at /tmp/deployment.log"
EOF

# Run task organization demo
ansible-playbook -i inventory.ini task-organization-demo.yml
cat /tmp/deployment.log
```

## Interactive Practice Session

### Hands-On Exercise for Participants
```yaml
# Create practice-exercise.yml
cat > practice-exercise.yml << 'EOF'
---
- name: Participant Practice - Ansible Review
  hosts: demo_hosts
  
  vars:
    # TODO: Participants will work with these variables
    my_app:
      name: "student-app"
      version: "1.0"
      port: 3000
    
  tasks:
    # TODO: Participants create tasks using fundamental concepts:
    # 1. Create directory structure
    # 2. Generate configuration files using variables
    # 3. Use loops to create multiple files
    # 4. Add a handler for configuration changes
    # 5. Use debug to show results
    
    - name: Example task - participants modify this
      debug:
        msg: "TODO: Create your application deployment here"
        
    # Hints:
    # - Use file module to create directories
    # - Use copy or template for configuration files
    # - Use loop to create multiple environments
    # - Add notify to trigger handlers
    # - Use variables from my_app dictionary
EOF
```

**Practice Instructions:**
1. Create directory structure for the application
2. Generate configuration files using the my_app variables
3. Create multiple environment configs using loops
4. Add a handler that responds to configuration changes
5. Use debug tasks to show deployment progress

## Key Takeaways Summary

### Essential Ansible Concepts:
1. **Modules** - Building blocks of automation (file, copy, service, etc.)
2. **Tasks** - Individual actions using modules
3. **Playbooks** - YAML files organizing tasks into plays
4. **Inventory** - Defines hosts and groups
5. **Variables** - Make playbooks flexible and reusable
6. **Handlers** - Event-driven tasks for service management

### Best Practices Reviewed:
- Use meaningful task names
- Organize tasks logically with blocks
- Leverage variables for flexibility
- Use handlers for service management
- Implement proper error handling
- Keep playbooks idempotent

### Common Patterns:
- **Directory creation** before file operations
- **Configuration management** with templates/copy
- **Service management** with handlers
- **Error handling** with blocks and rescue
- **Logging** for audit trails

### When to Use Each Concept:
- **Variables**: Configuration that changes between environments
- **Loops**: Repetitive tasks with different parameters
- **Handlers**: Service restarts and configuration reloads
- **Blocks**: Grouping related tasks with error handling
- **Facts**: System information for conditional logic

## Demo Cleanup
```bash
# Clean up demo files
rm -rf /tmp/ansible-demo /tmp/demo-service* /tmp/organized-app /tmp/deployment.log
rm -f *.yml group_vars/* host_vars/*
```

---

**Note for Instructor**: This review focuses on reinforcing essential concepts while assessing participant skill levels. Adjust depth and pace based on participant responses and questions. Use this as an opportunity to identify knowledge gaps before proceeding to advanced modules.

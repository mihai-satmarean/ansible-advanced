# Lab 1.1: Modules, Tasks, and Playbooks Review

## Objective
Review and practice basic Ansible components: modules, tasks, and playbooks with practical scenarios.

## Duration
30 minutes

## Prerequisites
- Ansible installed on control machine
- Access to target hosts (local or remote)
- Basic text editor

## Lab Setup

### 1. Create Lab Directory
```bash
mkdir -p ~/ansible-labs/module-01/lab-1.1
cd ~/ansible-labs/module-01/lab-1.1
```

### 2. Create Basic Inventory
```bash
cat > inventory.ini << EOF
[webservers]
localhost ansible_connection=local

[databases]  
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Basic Module Usage (10 minutes)

### Task: Create a simple playbook using common modules

Create `basic-modules.yml`:
```yaml
---
- name: Basic module demonstration
  hosts: localhost
  gather_facts: yes
  become: yes
  
  tasks:
    - name: Ensure directory exists
      file:
        path: /tmp/ansible-demo
        state: directory
        mode: '0755'
    
    - name: Create a simple file
      copy:
        content: |
          # Ansible Demo File
          Created by: {{ ansible_user_id }}
          Hostname: {{ ansible_hostname }}
          Date: {{ ansible_date_time.iso8601 }}
        dest: /tmp/ansible-demo/info.txt
        mode: '0644'
    
    - name: Install package (example)
      package:
        name: curl
        state: present
    
    - name: Check service status
      service_facts:
    
    - name: Display system information
      debug:
        msg: |
          System: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Memory: {{ ansible_memtotal_mb }}MB
          CPU cores: {{ ansible_processor_vcpus }}
```

### Run the playbook:
```bash
ansible-playbook -i inventory.ini basic-modules.yml
```

## Exercise 2: Task Organization (10 minutes)

### Task: Organize tasks logically with variables and conditions

Create `organized-tasks.yml`:
```yaml
---
- name: Organized task demonstration
  hosts: localhost
  vars:
    app_name: demo-app
    app_version: "1.0.0"
    app_port: 8080
    
  tasks:
    # Preparation phase
    - name: Create application user
      user:
        name: "{{ app_name }}"
        system: yes
        shell: /bin/false
        home: "/opt/{{ app_name }}"
        create_home: yes
      become: yes
    
    - name: Create application directories
      file:
        path: "{{ item }}"
        state: directory
        owner: "{{ app_name }}"
        group: "{{ app_name }}"
        mode: '0755'
      loop:
        - "/opt/{{ app_name }}/bin"
        - "/opt/{{ app_name }}/config"
        - "/opt/{{ app_name }}/logs"
        - "/var/log/{{ app_name }}"
      become: yes
    
    # Configuration phase
    - name: Generate application configuration
      copy:
        content: |
          # {{ app_name }} Configuration
          app.name={{ app_name }}
          app.version={{ app_version }}
          app.port={{ app_port }}
          app.environment={{ ansible_env.USER | default('unknown') }}
        dest: "/opt/{{ app_name }}/config/app.conf"
        owner: "{{ app_name }}"
        group: "{{ app_name }}"
        mode: '0640'
      become: yes
    
    # Verification phase
    - name: Verify installation
      stat:
        path: "/opt/{{ app_name }}/config/app.conf"
      register: config_file
    
    - name: Display verification results
      debug:
        msg: |
          Configuration file exists: {{ config_file.stat.exists }}
          File size: {{ config_file.stat.size | default(0) }} bytes
          Owner: {{ config_file.stat.pw_name | default('unknown') }}
```

### Run the playbook:
```bash
ansible-playbook -i inventory.ini organized-tasks.yml
```

## Exercise 3: Playbook Structure Best Practices (10 minutes)

### Task: Create a well-structured playbook with multiple plays

Create `structured-playbook.yml`:
```yaml
---
# Play 1: System preparation
- name: System preparation
  hosts: localhost
  become: yes
  vars:
    required_packages:
      - curl
      - wget
      - unzip
      
  tasks:
    - name: Update package cache (Ubuntu/Debian)
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
    
    - name: Install required packages
      package:
        name: "{{ required_packages }}"
        state: present
    
    - name: Create system directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /opt/shared
        - /var/log/shared

# Play 2: Application setup
- name: Application configuration
  hosts: localhost
  vars:
    services:
      - name: web-service
        port: 80
        enabled: true
      - name: api-service
        port: 8080
        enabled: true
      - name: admin-service
        port: 9090
        enabled: false
        
  tasks:
    - name: Generate service configurations
      copy:
        content: |
          [{{ item.name }}]
          port={{ item.port }}
          enabled={{ item.enabled | lower }}
          created={{ ansible_date_time.iso8601 }}
        dest: "/opt/shared/{{ item.name }}.conf"
        mode: '0644'
      loop: "{{ services }}"
      when: item.enabled | default(false)
    
    - name: List created configurations
      find:
        paths: /opt/shared
        patterns: "*.conf"
      register: config_files
    
    - name: Display created files
      debug:
        msg: "Created {{ config_files.files | length }} configuration files"

# Play 3: Verification and cleanup
- name: Verification
  hosts: localhost
  tasks:
    - name: Gather information about created resources
      shell: |
        echo "=== Directories ==="
        ls -la /opt/shared/ /var/log/shared/ 2>/dev/null || echo "Directories not found"
        echo "=== Processes ==="
        ps aux | grep -E "(curl|wget)" | grep -v grep || echo "No matching processes"
      register: system_info
      changed_when: false
    
    - name: Display system information
      debug:
        var: system_info.stdout_lines
```

### Run the playbook:
```bash
ansible-playbook -i inventory.ini structured-playbook.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Verify created files and directories
ls -la /tmp/ansible-demo/
ls -la /opt/demo-app/
ls -la /opt/shared/

# Check created user
id demo-app 2>/dev/null || echo "User not created"
```

### 2. Discussion Points
- Which modules were most useful for your daily tasks?
- How do you organize tasks in your current playbooks?
- What challenges do you face with task organization?

### 3. Clean Up
```bash
# Remove demo files (optional)
sudo rm -rf /tmp/ansible-demo /opt/demo-app /opt/shared /var/log/shared
sudo userdel demo-app 2>/dev/null || true
```

## Key Takeaways
- Modules are the building blocks of Ansible automation
- Task organization improves playbook readability and maintenance
- Variables and loops make playbooks more flexible and reusable
- Multiple plays allow logical separation of concerns
- Always verify your automation results

## Next Steps
Proceed to Lab 1.2: Host Targeting and User Management

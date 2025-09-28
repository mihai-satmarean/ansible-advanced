# Lab 1.2: Host Targeting and User Management

## Objective
Practice host patterns, targeting strategies, and user management scenarios relevant to enterprise environments.

## Duration
25 minutes

## Prerequisites
- Completed Lab 1.1
- Understanding of basic Ansible inventory concepts

## Lab Setup

### 1. Create Enhanced Inventory
```bash
cd ~/ansible-labs/module-01
mkdir -p lab-1.2
cd lab-1.2

cat > inventory.ini << EOF
[webservers]
web01 ansible_host=localhost ansible_connection=local
web02 ansible_host=localhost ansible_connection=local

[databases]
db01 ansible_host=localhost ansible_connection=local
db02 ansible_host=localhost ansible_connection=local

[loadbalancers]
lb01 ansible_host=localhost ansible_connection=local

[production:children]
webservers
databases
loadbalancers

[staging]
staging-web ansible_host=localhost ansible_connection=local
staging-db ansible_host=localhost ansible_connection=local

[development]
dev-all ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Host Pattern Targeting (10 minutes)

### Task: Practice different host targeting patterns

Create `host-patterns.yml`:
```yaml
---
- name: Target all webservers
  hosts: webservers
  gather_facts: no
  tasks:
    - name: Display webserver information
      debug:
        msg: "Processing webserver: {{ inventory_hostname }}"

- name: Target specific host pattern
  hosts: "web*"
  gather_facts: no
  tasks:
    - name: Display web-related hosts
      debug:
        msg: "Web host: {{ inventory_hostname }}"

- name: Target production environment
  hosts: production
  gather_facts: no
  tasks:
    - name: Display production hosts
      debug:
        msg: "Production host: {{ inventory_hostname }} in group {{ group_names }}"

- name: Target with exclusions
  hosts: "all:!staging:!development"
  gather_facts: no
  tasks:
    - name: Display non-staging/dev hosts
      debug:
        msg: "Host {{ inventory_hostname }} is not in staging or development"

- name: Target intersection of groups
  hosts: "webservers:&production"
  gather_facts: no
  tasks:
    - name: Display production webservers
      debug:
        msg: "Production webserver: {{ inventory_hostname }}"
```

### Run with different limits:
```bash
# Run all patterns
ansible-playbook -i inventory.ini host-patterns.yml

# Run with limit to specific hosts
ansible-playbook -i inventory.ini host-patterns.yml --limit "web01,db01"

# Run with limit to group
ansible-playbook -i inventory.ini host-patterns.yml --limit "webservers"
```

## Exercise 2: User Management Scenarios (15 minutes)

### Task: Implement common user management tasks

Create `user-management.yml`:
```yaml
---
- name: User management demonstration
  hosts: localhost
  become: yes
  vars:
    # Application users
    app_users:
      - name: webapp
        comment: "Web Application User"
        system: yes
        shell: /bin/false
        home: /opt/webapp
        
      - name: dbadmin
        comment: "Database Administrator"
        system: no
        shell: /bin/bash
        home: /home/dbadmin
        groups: ["sudo"]
        
      - name: monitor
        comment: "Monitoring Service User"
        system: yes
        shell: /bin/false
        home: /var/lib/monitor
    
    # SSH keys for users (example keys - do not use in production)
    ssh_keys:
      dbadmin: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC... dbadmin@company.com"
      
  tasks:
    # Create application users
    - name: Create application users
      user:
        name: "{{ item.name }}"
        comment: "{{ item.comment }}"
        system: "{{ item.system | default(false) }}"
        shell: "{{ item.shell | default('/bin/bash') }}"
        home: "{{ item.home | default('/home/' + item.name) }}"
        create_home: yes
        groups: "{{ item.groups | default([]) }}"
        state: present
      loop: "{{ app_users }}"
    
    # Set up SSH keys for non-system users
    - name: Setup SSH keys for users
      authorized_key:
        user: "{{ item.key }}"
        key: "{{ item.value }}"
        state: present
      loop: "{{ ssh_keys | dict2items }}"
      when: ssh_keys is defined
    
    # Create application directories with proper ownership
    - name: Create application directories
      file:
        path: "{{ item.path }}"
        state: directory
        owner: "{{ item.owner }}"
        group: "{{ item.group | default(item.owner) }}"
        mode: "{{ item.mode | default('0755') }}"
      loop:
        - { path: "/opt/webapp/bin", owner: "webapp" }
        - { path: "/opt/webapp/config", owner: "webapp", mode: "0750" }
        - { path: "/opt/webapp/logs", owner: "webapp", mode: "0755" }
        - { path: "/var/lib/monitor", owner: "monitor" }
        - { path: "/var/log/monitor", owner: "monitor" }
    
    # Configure sudo access for admin users
    - name: Configure sudo access
      copy:
        content: |
          # Database admin sudo rules
          dbadmin ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart mysql
          dbadmin ALL=(ALL) NOPASSWD: /usr/bin/systemctl status mysql
          dbadmin ALL=(ALL) NOPASSWD: /usr/bin/mysql
        dest: /etc/sudoers.d/dbadmin
        mode: '0440'
        validate: 'visudo -cf %s'
      when: "'dbadmin' in (app_users | map(attribute='name') | list)"
    
    # Set up user environment
    - name: Create user-specific configurations
      copy:
        content: |
          # {{ item.name }} environment configuration
          export PATH=$PATH:/opt/{{ item.name }}/bin
          export {{ item.name | upper }}_HOME={{ item.home }}
          
          # Application-specific aliases
          alias logs='tail -f {{ item.home }}/logs/*.log'
          alias status='systemctl status {{ item.name }}'
        dest: "{{ item.home }}/.profile"
        owner: "{{ item.name }}"
        group: "{{ item.name }}"
        mode: '0644'
      loop: "{{ app_users }}"
      when: not item.system
    
    # Verify user creation
    - name: Verify users were created
      getent:
        database: passwd
        key: "{{ item.name }}"
      loop: "{{ app_users }}"
      register: user_check
    
    - name: Display user verification results
      debug:
        msg: |
          User: {{ item.item.name }}
          UID: {{ item.ansible_facts.getent_passwd[item.item.name][1] }}
          Home: {{ item.ansible_facts.getent_passwd[item.item.name][4] }}
          Shell: {{ item.ansible_facts.getent_passwd[item.item.name][5] }}
      loop: "{{ user_check.results }}"
```

### Run the user management playbook:
```bash
ansible-playbook -i inventory.ini user-management.yml
```

### Verify user creation:
```bash
# Check created users
sudo cat /etc/passwd | grep -E "(webapp|dbadmin|monitor)"

# Check user directories
sudo ls -la /opt/webapp/
sudo ls -la /var/lib/monitor/

# Check sudo configuration
sudo cat /etc/sudoers.d/dbadmin
```

## Exercise 3: Host-Specific Variables and Targeting

### Task: Use host-specific variables and conditional targeting

Create `host-specific.yml`:
```yaml
---
- name: Host-specific configuration
  hosts: all
  vars:
    # Host-specific configurations
    host_configs:
      web01:
        role: "primary-web"
        port: 80
        ssl_enabled: true
      web02:
        role: "secondary-web"  
        port: 80
        ssl_enabled: false
      db01:
        role: "master-db"
        port: 3306
        backup_enabled: true
      db02:
        role: "slave-db"
        port: 3306
        backup_enabled: false
      lb01:
        role: "loadbalancer"
        port: 443
        ssl_enabled: true
        
  tasks:
    - name: Display host-specific information
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Role: {{ host_configs[inventory_hostname].role | default('undefined') }}
          Port: {{ host_configs[inventory_hostname].port | default('N/A') }}
          SSL: {{ host_configs[inventory_hostname].ssl_enabled | default(false) }}
      when: inventory_hostname in host_configs
    
    - name: Configure based on host role
      copy:
        content: |
          # Configuration for {{ inventory_hostname }}
          ROLE={{ host_configs[inventory_hostname].role }}
          PORT={{ host_configs[inventory_hostname].port }}
          SSL_ENABLED={{ host_configs[inventory_hostname].ssl_enabled | default(false) | lower }}
          ENVIRONMENT={{ group_names | join(',') }}
          GENERATED={{ ansible_date_time.iso8601 }}
        dest: "/tmp/{{ inventory_hostname }}-config.env"
        mode: '0644'
      when: 
        - inventory_hostname in host_configs
        - host_configs[inventory_hostname].role is defined
    
    - name: Create role-specific directories
      file:
        path: "/tmp/{{ host_configs[inventory_hostname].role }}"
        state: directory
        mode: '0755'
      when: 
        - inventory_hostname in host_configs
        - host_configs[inventory_hostname].role is defined
```

### Run with different targeting:
```bash
# Run on all hosts
ansible-playbook -i inventory.ini host-specific.yml

# Run only on webservers
ansible-playbook -i inventory.ini host-specific.yml --limit webservers

# Run on specific host
ansible-playbook -i inventory.ini host-specific.yml --limit web01
```

## Verification and Discussion

### 1. Check Results
```bash
# Verify created users
id webapp dbadmin monitor 2>/dev/null || echo "Some users not created"

# Check configuration files
ls -la /tmp/*-config.env
cat /tmp/web01-config.env

# Check role directories
ls -la /tmp/ | grep -E "(primary-web|master-db|loadbalancer)"
```

### 2. Discussion Points
- How do you currently manage users across different environments?
- What host targeting patterns would be useful in your infrastructure?
- How do you handle environment-specific configurations?

### 3. Clean Up
```bash
# Remove demo files
sudo rm -f /tmp/*-config.env
sudo rm -rf /tmp/primary-web /tmp/master-db /tmp/loadbalancer

# Remove created users (optional)
sudo userdel -r webapp 2>/dev/null || true
sudo userdel -r dbadmin 2>/dev/null || true  
sudo userdel -r monitor 2>/dev/null || true
sudo rm -f /etc/sudoers.d/dbadmin
```

## Key Takeaways
- Host patterns provide flexible targeting options
- User management requires careful consideration of security and permissions
- Host-specific variables enable customized configurations
- Conditional execution based on host properties improves flexibility
- Verification tasks ensure automation success

## Next Steps
Proceed to Lab 1.3: Task Organization and Dependencies

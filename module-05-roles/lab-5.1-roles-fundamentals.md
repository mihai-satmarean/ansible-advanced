# Lab 5.1: Ansible Roles Fundamentals (Interactive Demo)

## Objective
Master essential Ansible roles concepts through interactive demonstration and practical examples suitable for all skill levels.

## Duration
25 minutes (interactive demo)

## Format
**Interactive demonstration** - Instructor-led with participant practice

## Prerequisites
- Basic understanding of Ansible playbooks
- Familiarity with YAML structure

## Demo Setup

```bash
cd ~/ansible-labs
mkdir -p module-05
cd module-05

# Create simple inventory
cat > inventory.ini << 'EOF'
[demo_hosts]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Demo 1: Role Structure and Creation (8 minutes)

### What is an Ansible Role?
**Instructor explains:**
- Roles organize playbook content into reusable components
- Standard directory structure for consistency
- Encapsulation of tasks, variables, files, templates

### Create Your First Role
```bash
# Create a simple web server role
ansible-galaxy init roles/webserver

# Show the created structure
tree roles/webserver
```

**Structure Explanation:**
```
roles/webserver/
├── defaults/          # Default variables (lowest priority)
├── files/            # Static files to copy
├── handlers/         # Handler tasks
├── meta/            # Role metadata and dependencies
├── tasks/           # Main task files
├── templates/       # Jinja2 templates
├── tests/           # Test playbooks
└── vars/            # Role variables (higher priority)
```

### Basic Role Implementation
```yaml
# Edit roles/webserver/tasks/main.yml
cat > roles/webserver/tasks/main.yml << 'EOF'
---
# Main tasks for webserver role
- name: Install web server package
  package:
    name: "{{ webserver_package }}"
    state: present
  become: yes

- name: Create web content directory
  file:
    path: "{{ webserver_document_root }}"
    state: directory
    owner: "{{ webserver_user }}"
    group: "{{ webserver_group }}"
    mode: '0755'
  become: yes

- name: Deploy index page
  template:
    src: index.html.j2
    dest: "{{ webserver_document_root }}/index.html"
    owner: "{{ webserver_user }}"
    group: "{{ webserver_group }}"
    mode: '0644'
  become: yes
  notify: restart webserver

- name: Start and enable web server
  service:
    name: "{{ webserver_service }}"
    state: started
    enabled: yes
  become: yes
EOF

# Create default variables
cat > roles/webserver/defaults/main.yml << 'EOF'
---
# Default variables for webserver role
webserver_package: nginx
webserver_service: nginx
webserver_user: www-data
webserver_group: www-data
webserver_document_root: /var/www/html
webserver_port: 80
EOF

# Create a simple template
cat > roles/webserver/templates/index.html.j2 << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>{{ ansible_hostname }} Web Server</title>
</head>
<body>
    <h1>Welcome to {{ ansible_hostname }}</h1>
    <p>Server: {{ inventory_hostname }}</p>
    <p>Generated: {{ ansible_date_time.iso8601 }}</p>
    <p>Web Server: {{ webserver_package }}</p>
</body>
</html>
EOF

# Create handler
cat > roles/webserver/handlers/main.yml << 'EOF'
---
# Handlers for webserver role
- name: restart webserver
  service:
    name: "{{ webserver_service }}"
    state: restarted
  become: yes
EOF
```

## Demo 2: Using Roles in Playbooks (8 minutes)

### Simple Role Usage
```yaml
# Create a playbook that uses the role
cat > webserver-playbook.yml << 'EOF'
---
- name: Deploy Web Server using Role
  hosts: demo_hosts
  become: yes
  
  roles:
    - webserver
EOF

# Run the playbook (dry run for demo)
ansible-playbook -i inventory.ini webserver-playbook.yml --check
```

### Role with Custom Variables
```yaml
# Create playbook with custom variables
cat > custom-webserver.yml << 'EOF'
---
- name: Deploy Custom Web Server
  hosts: demo_hosts
  become: yes
  
  vars:
    webserver_package: apache2
    webserver_service: apache2
    webserver_user: www-data
    webserver_group: www-data
    webserver_document_root: /var/www/myapp
  
  roles:
    - webserver
EOF

# Show how variables override defaults
ansible-playbook -i inventory.ini custom-webserver.yml --check
```

### Multiple Roles in One Playbook
```yaml
# Create a multi-role playbook
cat > full-stack.yml << 'EOF'
---
- name: Full Stack Deployment
  hosts: demo_hosts
  become: yes
  
  roles:
    - role: webserver
      vars:
        webserver_port: 8080
        webserver_document_root: /var/www/frontend
        
    # Could add more roles here:
    # - database
    # - monitoring
    # - security
EOF
```

## Demo 3: Role Variables and Precedence (9 minutes)

### Variable Precedence Demo
```yaml
# Create role with multiple variable sources
mkdir -p roles/webserver/vars

# High priority variables
cat > roles/webserver/vars/main.yml << 'EOF'
---
# High priority role variables
webserver_max_connections: 1000
webserver_timeout: 30
EOF

# Create environment-specific variables
cat > roles/webserver/vars/production.yml << 'EOF'
---
# Production-specific variables
webserver_worker_processes: 4
webserver_keepalive_timeout: 65
EOF

# Update tasks to show variable usage
cat >> roles/webserver/tasks/main.yml << 'EOF'

- name: Display variable precedence
  debug:
    msg: |
      Package: {{ webserver_package }}
      Service: {{ webserver_service }}
      Max Connections: {{ webserver_max_connections }}
      Timeout: {{ webserver_timeout }}
EOF
```

### Variable Precedence Explanation
**Instructor explains precedence (highest to lowest):**
1. Extra vars (`-e` command line)
2. Task vars
3. Block vars  
4. Role vars (`vars/main.yml`)
5. Play vars
6. Host vars
7. Group vars
8. Role defaults (`defaults/main.yml`)

### Practical Variable Usage
```yaml
# Create playbook showing variable precedence
cat > variable-demo.yml << 'EOF'
---
- name: Variable Precedence Demo
  hosts: demo_hosts
  
  vars:
    webserver_package: "httpd"  # Play vars override role defaults
  
  tasks:
    - name: Use role with task-specific vars
      include_role:
        name: webserver
      vars:
        webserver_service: "httpd"  # Task vars override play vars
        
    - name: Show final variable values
      debug:
        msg: |
          Final package: {{ webserver_package }}
          Final service: {{ webserver_service }}
EOF

# Run to show precedence
ansible-playbook -i inventory.ini variable-demo.yml --check
```

## Interactive Practice Session

### Hands-On Exercise for Participants
```yaml
# Create practice role template
ansible-galaxy init roles/myapp

# Participants will modify this role
cat > practice-exercise.yml << 'EOF'
---
- name: Participant Practice - Create Database Role
  hosts: demo_hosts
  
  # TODO: Participants create a simple database role
  # Requirements:
  # 1. Install database package (mysql-server or postgresql)
  # 2. Create database user
  # 3. Create database
  # 4. Use variables for database name and user
  # 5. Add a handler to restart database service
  
  roles:
    - myapp  # Participants will modify this role
EOF
```

**Practice Instructions:**
1. Modify `roles/myapp/tasks/main.yml` to install database
2. Add variables in `roles/myapp/defaults/main.yml`
3. Create handler in `roles/myapp/handlers/main.yml`
4. Test with the practice playbook

## Key Takeaways Summary

### Essential Role Concepts:
1. **Structure** - Standard directory layout for organization
2. **Reusability** - Write once, use many times
3. **Variables** - Flexible configuration through defaults and vars
4. **Handlers** - Event-driven actions (restart services, etc.)
5. **Templates** - Dynamic file generation

### Role Benefits:
- **Organization** - Clean, structured code
- **Reusability** - Share across projects and teams
- **Maintainability** - Centralized logic and configuration
- **Testing** - Isolated components for easier testing
- **Collaboration** - Standard structure for team development

### When to Use Roles:
- **Repeated tasks** - Installing and configuring services
- **Complex configurations** - Multi-step setups
- **Team collaboration** - Shared components
- **Environment consistency** - Same setup across dev/staging/prod

### Best Practices:
- Use meaningful role names
- Provide sensible defaults
- Document role variables and usage
- Keep roles focused on single responsibility
- Use handlers for service management
- Test roles independently

## Discussion Points

### For Alexandra & Gabriela (Beginners):
- "How would roles help organize your playbooks?"
- "What's the difference between tasks and roles?"
- "When would you create a new role vs. adding tasks to a playbook?"

### For Victor & Vlad (Intermediate):
- "How do you structure roles in your CI/CD pipelines?"
- "What's your strategy for role versioning and distribution?"
- "How do you handle role dependencies in complex deployments?"

## Demo Cleanup
```bash
# Clean up demo files
rm -rf roles/ *.yml
```

---

**Note for Instructor**: Focus on practical role creation and usage. Emphasize the organizational benefits and reusability. Adjust complexity based on participant experience and questions.

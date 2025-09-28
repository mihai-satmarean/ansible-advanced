# Lab 5.2: Ansible Galaxy Essentials (Interactive Demo)

## Objective
Learn to find, install, and use roles from Ansible Galaxy through practical demonstration suitable for all skill levels.

## Duration
20 minutes (interactive demo)

## Format
**Interactive demonstration** - Instructor-led with participant practice

## Prerequisites
- Completed Lab 5.1 (Roles Fundamentals)
- Internet connection for Galaxy access

## Demo Setup

```bash
cd ~/ansible-labs/module-05
mkdir -p galaxy-demo
cd galaxy-demo

# Create inventory for testing
cat > inventory.ini << 'EOF'
[demo_hosts]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Demo 1: Exploring Ansible Galaxy (5 minutes)

### What is Ansible Galaxy?
**Instructor explains:**
- Community hub for sharing Ansible roles
- Pre-built roles for common tasks
- Quality ratings and documentation
- Public and private Galaxy servers

### Browse Galaxy Web Interface
```bash
# Show Galaxy website (instructor demonstrates)
# https://galaxy.ansible.com/

# Search for popular roles:
# - nginx roles
# - docker roles  
# - security roles
```

### Galaxy Command Line Basics
```bash
# Search for roles from command line
ansible-galaxy search nginx

# Get role information
ansible-galaxy info geerlingguy.nginx

# List installed roles
ansible-galaxy list
```

## Demo 2: Installing and Using Galaxy Roles (8 minutes)

### Install Popular Roles
```bash
# Install a well-maintained nginx role
ansible-galaxy install geerlingguy.nginx

# Install multiple roles
ansible-galaxy install geerlingguy.docker geerlingguy.pip

# Show where roles are installed
ansible-galaxy list
ls ~/.ansible/roles/
```

### Create Requirements File
```yaml
# Create requirements.yml for project dependencies
cat > requirements.yml << 'EOF'
---
# Project role dependencies
roles:
  - name: geerlingguy.nginx
    version: "3.1.4"
    
  - name: geerlingguy.docker
    version: "6.1.0"
    
  - name: geerlingguy.pip
    version: "2.2.0"

# Can also install from Git repositories
  - src: https://github.com/example/custom-role.git
    name: custom-role
    version: main
EOF

# Install all requirements
ansible-galaxy install -r requirements.yml
```

### Use Galaxy Role in Playbook
```yaml
# Create playbook using Galaxy roles
cat > galaxy-playbook.yml << 'EOF'
---
- name: Deploy Web Server with Galaxy Roles
  hosts: demo_hosts
  become: yes
  
  vars:
    # Configure nginx role variables
    nginx_remove_default_vhost: true
    nginx_vhosts:
      - listen: "80"
        server_name: "demo.local"
        root: "/var/www/demo"
        index: "index.html"
        
  pre_tasks:
    - name: Update package cache (Debian/Ubuntu)
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"
      
  roles:
    - geerlingguy.nginx
    
  post_tasks:
    - name: Create demo content
      copy:
        content: |
          <h1>Demo Site</h1>
          <p>Deployed with Ansible Galaxy role!</p>
          <p>Server: {{ ansible_hostname }}</p>
        dest: /var/www/demo/index.html
      become: yes
EOF

# Run playbook (dry run for demo)
ansible-playbook -i inventory.ini galaxy-playbook.yml --check
```

## Demo 3: Role Management Best Practices (7 minutes)

### Project-Specific Role Installation
```bash
# Create project-specific roles directory
mkdir -p roles

# Install roles to project directory
ansible-galaxy install -r requirements.yml -p roles/

# Show project structure
tree -L 2
```

### Version Management
```yaml
# Update requirements.yml with specific versions
cat > requirements.yml << 'EOF'
---
roles:
  # Always specify versions for production
  - name: geerlingguy.nginx
    version: "3.1.4"  # Specific version
    
  # Can use version ranges
  - name: geerlingguy.docker
    version: ">=6.0.0,<7.0.0"
    
  # Development can use latest
  - name: geerlingguy.pip
    version: "latest"  # Only for development!
EOF
```

### Role Configuration and Customization
```yaml
# Create playbook showing role customization
cat > customized-roles.yml << 'EOF'
---
- name: Customized Galaxy Roles Demo
  hosts: demo_hosts
  become: yes
  
  vars:
    # Nginx customization
    nginx_user: "www-data"
    nginx_worker_processes: "auto"
    nginx_worker_connections: "1024"
    nginx_keepalive_timeout: "65"
    
    nginx_extra_conf_options: |
      worker_rlimit_nofile 8192;
      
    nginx_vhosts:
      - listen: "80"
        server_name: "app.local"
        root: "/var/www/app"
        index: "index.php index.html"
        extra_parameters: |
          location ~ \.php$ {
              fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
              fastcgi_index index.php;
              include fastcgi_params;
          }
  
  roles:
    - role: geerlingguy.nginx
      tags: ['nginx', 'webserver']
      
  post_tasks:
    - name: Verify nginx configuration
      command: nginx -t
      changed_when: false
      tags: ['nginx', 'verify']
EOF
```

### Managing Role Updates
```bash
# Check for role updates
ansible-galaxy info geerlingguy.nginx

# Update specific role
ansible-galaxy install geerlingguy.nginx --force

# Update all roles from requirements
ansible-galaxy install -r requirements.yml --force

# Remove unused roles
ansible-galaxy remove geerlingguy.pip
```

## Interactive Practice Session

### Hands-On Exercise for Participants
```yaml
# Create practice exercise
cat > practice-galaxy.yml << 'EOF'
---
- name: Participant Practice - Galaxy Role Usage
  hosts: demo_hosts
  become: yes
  
  # TODO: Participants will:
  # 1. Find a suitable role for their use case
  # 2. Install it using ansible-galaxy
  # 3. Configure it with appropriate variables
  # 4. Create a playbook that uses the role
  
  vars:
    # Example: Configure a database role
    # mysql_root_password: "secure_password"
    # mysql_databases:
    #   - name: "myapp"
    # mysql_users:
    #   - name: "appuser"
    #     password: "userpass"
    #     priv: "myapp.*:ALL"
    
  roles:
    # - role_name_here
    
  tasks:
    - name: Verify role installation
      debug:
        msg: "Role successfully applied!"
EOF
```

**Practice Instructions:**
1. Search Galaxy for a role (database, monitoring, security)
2. Read the role documentation on Galaxy
3. Install the role using `ansible-galaxy install`
4. Configure variables based on role documentation
5. Test the playbook

## Key Takeaways Summary

### Ansible Galaxy Benefits:
1. **Time Saving** - Pre-built, tested roles
2. **Community Knowledge** - Best practices from experts
3. **Quality Assurance** - Community ratings and reviews
4. **Documentation** - Well-documented role usage
5. **Maintenance** - Regular updates and bug fixes

### Galaxy Best Practices:
- **Version Pinning** - Always specify versions in production
- **Requirements File** - Use `requirements.yml` for dependencies
- **Local Installation** - Install to project directories
- **Role Evaluation** - Check ratings, downloads, and documentation
- **Regular Updates** - Keep roles updated for security

### Popular Galaxy Roles:
- **geerlingguy.nginx** - Web server configuration
- **geerlingguy.docker** - Docker installation and setup
- **geerlingguy.mysql** - MySQL database server
- **geerlingguy.security** - Basic security hardening
- **community.general** - Collection of general modules

### When to Use Galaxy vs. Custom Roles:
**Use Galaxy when:**
- Standard, well-known software (nginx, docker, mysql)
- Quick prototyping and development
- Learning best practices
- Common security configurations

**Create Custom when:**
- Company-specific applications
- Unique business requirements
- Proprietary software
- Highly customized configurations

## Discussion Points

### For Alexandra & Gabriela (Beginners):
- "What types of roles would be useful for your projects?"
- "How do you evaluate if a Galaxy role is trustworthy?"
- "What's the difference between installing globally vs. locally?"

### For Victor & Vlad (Intermediate):
- "How do you manage role versions in your CI/CD pipelines?"
- "Do you use private Galaxy servers in your organization?"
- "What's your strategy for role security and compliance?"

## Common Galaxy Commands Reference

```bash
# Search and discovery
ansible-galaxy search <keyword>
ansible-galaxy info <role_name>

# Installation
ansible-galaxy install <role_name>
ansible-galaxy install -r requirements.yml
ansible-galaxy install -p ./roles <role_name>

# Management
ansible-galaxy list
ansible-galaxy remove <role_name>
ansible-galaxy install <role_name> --force

# Requirements file
ansible-galaxy install -r requirements.yml --force
```

## Demo Cleanup
```bash
# Clean up demo files
rm -f *.yml
ansible-galaxy remove geerlingguy.nginx geerlingguy.docker geerlingguy.pip
```

---

**Note for Instructor**: Emphasize the practical benefits of Galaxy for rapid development. Show real examples of popular roles and their documentation. Encourage participants to explore Galaxy for their specific use cases.

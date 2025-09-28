# Lab 5.1: Role Architecture and Structure

## Objective
Master role directory structure, organization, and best practices for enterprise-grade role development.

## Duration
25 minutes

## Prerequisites
- Understanding of basic Ansible roles
- Knowledge of YAML and Jinja2 templating

## Lab Setup

```bash
cd ~/ansible-labs
mkdir -p module-05/lab-5.1
cd module-05/lab-5.1

# Create inventory for testing
cat > inventory.ini << 'EOF'
[web_servers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Complete Role Structure Creation (10 minutes)

### Task: Create a comprehensive role with all standard directories and files

Create a complete web server role:

```bash
# Create role structure using ansible-galaxy
ansible-galaxy init roles/enterprise_webserver

# Examine the created structure
tree roles/enterprise_webserver
```

Enhance the role structure with enterprise-specific directories:

```bash
# Add enterprise-specific directories
mkdir -p roles/enterprise_webserver/{library,filter_plugins,lookup_plugins,module_utils}
mkdir -p roles/enterprise_webserver/{molecule,docs,examples}
mkdir -p roles/enterprise_webserver/vars/{environments,applications}
mkdir -p roles/enterprise_webserver/tasks/{install,configure,security,monitoring}
```

Create comprehensive role metadata:

```yaml
# Update roles/enterprise_webserver/meta/main.yml
cat > roles/enterprise_webserver/meta/main.yml << 'EOF'
---
galaxy_info:
  role_name: enterprise_webserver
  namespace: company
  author: Enterprise DevOps Team
  description: Enterprise-grade web server role with security and monitoring
  company: Enterprise Corp
  license: MIT
  min_ansible_version: 2.9
  
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: EL
      versions:
        - 8
        - 9
  
  galaxy_tags:
    - web
    - nginx
    - security
    - monitoring
    - enterprise

dependencies:
  - role: company.security_baseline
    when: security_hardening_enabled | default(true)
  - role: company.monitoring_agent
    when: monitoring_enabled | default(true)

collections:
  - community.general
  - ansible.posix
EOF
```

Create comprehensive role documentation:

```markdown
# Create roles/enterprise_webserver/README.md
cat > roles/enterprise_webserver/README.md << 'EOF'
# Enterprise Web Server Role

A comprehensive Ansible role for deploying and managing enterprise-grade web servers with security hardening and monitoring integration.

## Features

- **Multi-platform support**: Ubuntu and RHEL/CentOS
- **Security hardening**: Integrated security baseline
- **Monitoring integration**: Built-in monitoring agent deployment
- **SSL/TLS management**: Automated certificate management
- **Performance optimization**: Tuned configurations for enterprise workloads
- **Backup integration**: Automated configuration backup
- **Health checks**: Comprehensive service validation

## Requirements

- Ansible >= 2.9
- Target systems: Ubuntu 20.04+, RHEL/CentOS 8+
- Sudo privileges on target hosts

## Role Variables

### Required Variables

```yaml
webserver_domain: "example.com"
webserver_admin_email: "admin@example.com"
```

### Optional Variables

```yaml
# Web server configuration
webserver_software: "nginx"  # nginx or apache
webserver_version: "latest"
webserver_port: 80
webserver_ssl_port: 443

# SSL Configuration
webserver_ssl_enabled: true
webserver_ssl_certificate_path: "/etc/ssl/certs/{{ webserver_domain }}.crt"
webserver_ssl_private_key_path: "/etc/ssl/private/{{ webserver_domain }}.key"

# Security settings
security_hardening_enabled: true
firewall_enabled: true
fail2ban_enabled: true

# Monitoring
monitoring_enabled: true
monitoring_agent: "datadog"

# Performance tuning
webserver_worker_processes: "auto"
webserver_worker_connections: 1024
webserver_keepalive_timeout: 65

# Backup configuration
backup_enabled: true
backup_schedule: "0 2 * * *"
backup_retention_days: 30
```

## Dependencies

- `company.security_baseline`: Provides security hardening
- `company.monitoring_agent`: Installs and configures monitoring

## Example Playbook

```yaml
---
- hosts: web_servers
  become: yes
  roles:
    - role: company.enterprise_webserver
      vars:
        webserver_domain: "myapp.company.com"
        webserver_admin_email: "devops@company.com"
        webserver_ssl_enabled: true
        monitoring_enabled: true
```

## Directory Structure

```
enterprise_webserver/
├── defaults/main.yml          # Default variables
├── vars/main.yml              # Role variables
├── vars/environments/         # Environment-specific variables
├── tasks/main.yml             # Main task file
├── tasks/install/             # Installation tasks
├── tasks/configure/           # Configuration tasks
├── tasks/security/            # Security hardening tasks
├── tasks/monitoring/          # Monitoring setup tasks
├── handlers/main.yml          # Handlers
├── templates/                 # Jinja2 templates
├── files/                     # Static files
├── library/                   # Custom modules
├── filter_plugins/            # Custom filters
├── lookup_plugins/            # Custom lookups
├── tests/                     # Test playbooks
├── molecule/                  # Molecule tests
├── docs/                      # Additional documentation
└── examples/                  # Usage examples
```

## Testing

```bash
# Run molecule tests
molecule test

# Manual testing
ansible-playbook tests/test.yml -i tests/inventory
```

## License

MIT

## Author Information

Enterprise DevOps Team
devops@company.com
EOF
```

Create comprehensive default variables:

```yaml
# Create roles/enterprise_webserver/defaults/main.yml
cat > roles/enterprise_webserver/defaults/main.yml << 'EOF'
---
# Enterprise Web Server Role Default Variables

# Basic web server configuration
webserver_software: "nginx"
webserver_version: "latest"
webserver_port: 80
webserver_ssl_port: 443
webserver_user: "www-data"
webserver_group: "www-data"

# Domain and contact information
webserver_domain: "{{ ansible_fqdn | default(ansible_hostname) }}"
webserver_admin_email: "admin@{{ webserver_domain }}"

# SSL/TLS Configuration
webserver_ssl_enabled: true
webserver_ssl_certificate_path: "/etc/ssl/certs/{{ webserver_domain }}.crt"
webserver_ssl_private_key_path: "/etc/ssl/private/{{ webserver_domain }}.key"
webserver_ssl_protocols: ["TLSv1.2", "TLSv1.3"]
webserver_ssl_ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384"

# Performance tuning
webserver_worker_processes: "auto"
webserver_worker_connections: 1024
webserver_keepalive_timeout: 65
webserver_client_max_body_size: "64M"
webserver_gzip_enabled: true
webserver_gzip_types:
  - "text/plain"
  - "text/css"
  - "application/json"
  - "application/javascript"
  - "text/xml"
  - "application/xml"
  - "application/xml+rss"
  - "text/javascript"

# Security settings
security_hardening_enabled: true
firewall_enabled: true
fail2ban_enabled: true
webserver_hide_version: true
webserver_server_tokens: "off"

# Monitoring and logging
monitoring_enabled: true
monitoring_agent: "datadog"
log_format: "combined"
log_rotation_enabled: true
log_retention_days: 30

# Backup configuration
backup_enabled: true
backup_schedule: "0 2 * * *"
backup_retention_days: 30
backup_destination: "/backup/webserver"

# Health check configuration
health_check_enabled: true
health_check_path: "/health"
health_check_interval: 30

# Custom error pages
custom_error_pages_enabled: true
error_pages:
  404: "/usr/share/nginx/html/404.html"
  500: "/usr/share/nginx/html/50x.html"
  502: "/usr/share/nginx/html/50x.html"
  503: "/usr/share/nginx/html/50x.html"
  504: "/usr/share/nginx/html/50x.html"

# Virtual hosts configuration
virtual_hosts: []
# Example:
# virtual_hosts:
#   - name: "app1.example.com"
#     document_root: "/var/www/app1"
#     ssl_enabled: true
#   - name: "app2.example.com"
#     document_root: "/var/www/app2"
#     ssl_enabled: false

# Upstream servers for load balancing
upstream_servers: []
# Example:
# upstream_servers:
#   - name: "backend"
#     servers:
#       - "192.168.1.10:8080"
#       - "192.168.1.11:8080"
#     method: "round_robin"

# Environment-specific overrides
webserver_environment: "{{ environment | default('production') }}"
EOF
```

Create organized task structure:

```yaml
# Create roles/enterprise_webserver/tasks/main.yml
cat > roles/enterprise_webserver/tasks/main.yml << 'EOF'
---
# Enterprise Web Server Role - Main Tasks

- name: Include environment-specific variables
  include_vars: "environments/{{ webserver_environment }}.yml"
  when: webserver_environment is defined
  tags: always

- name: Validate required variables
  assert:
    that:
      - webserver_domain is defined
      - webserver_admin_email is defined
      - webserver_software in ['nginx', 'apache']
    fail_msg: "Required variables are not properly defined"
    success_msg: "Variable validation passed"
  tags: always

- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"
  tags: always

- name: Display role information
  debug:
    msg: |
      Deploying {{ webserver_software }} web server
      Domain: {{ webserver_domain }}
      Environment: {{ webserver_environment }}
      SSL Enabled: {{ webserver_ssl_enabled }}
      Monitoring Enabled: {{ monitoring_enabled }}
  tags: always

# Installation tasks
- name: Include installation tasks
  include_tasks: install/main.yml
  tags: [install, webserver]

# Configuration tasks
- name: Include configuration tasks
  include_tasks: configure/main.yml
  tags: [configure, webserver]

# Security hardening tasks
- name: Include security tasks
  include_tasks: security/main.yml
  when: security_hardening_enabled | default(true)
  tags: [security, webserver]

# Monitoring setup tasks
- name: Include monitoring tasks
  include_tasks: monitoring/main.yml
  when: monitoring_enabled | default(true)
  tags: [monitoring, webserver]

# Service management
- name: Ensure web server is started and enabled
  service:
    name: "{{ webserver_service_name }}"
    state: started
    enabled: yes
  tags: [service, webserver]

# Validation tasks
- name: Include validation tasks
  include_tasks: validate.yml
  tags: [validate, webserver]
EOF

# Create installation tasks
mkdir -p roles/enterprise_webserver/tasks/install
cat > roles/enterprise_webserver/tasks/install/main.yml << 'EOF'
---
# Web Server Installation Tasks

- name: Update package cache (Debian/Ubuntu)
  apt:
    update_cache: yes
    cache_valid_time: 3600
  when: ansible_os_family == "Debian"
  tags: install

- name: Install web server package
  package:
    name: "{{ webserver_package_name }}"
    state: "{{ 'latest' if webserver_version == 'latest' else 'present' }}"
  notify: restart webserver
  tags: install

- name: Install additional packages
  package:
    name: "{{ webserver_additional_packages }}"
    state: present
  when: webserver_additional_packages is defined
  tags: install

- name: Create web server user
  user:
    name: "{{ webserver_user }}"
    group: "{{ webserver_group }}"
    system: yes
    shell: /bin/false
    home: /var/www
    create_home: no
  tags: install

- name: Create necessary directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ webserver_user }}"
    group: "{{ webserver_group }}"
    mode: '0755'
  loop:
    - /var/www
    - /var/log/{{ webserver_software }}
    - /etc/{{ webserver_software }}/sites-available
    - /etc/{{ webserver_software }}/sites-enabled
  tags: install
EOF

# Create configuration tasks
mkdir -p roles/enterprise_webserver/tasks/configure
cat > roles/enterprise_webserver/tasks/configure/main.yml << 'EOF'
---
# Web Server Configuration Tasks

- name: Generate main configuration file
  template:
    src: "{{ webserver_software }}/{{ webserver_software }}.conf.j2"
    dest: "/etc/{{ webserver_software }}/{{ webserver_software }}.conf"
    owner: root
    group: root
    mode: '0644'
    backup: yes
  notify: restart webserver
  tags: configure

- name: Generate virtual host configurations
  template:
    src: "{{ webserver_software }}/vhost.conf.j2"
    dest: "/etc/{{ webserver_software }}/sites-available/{{ item.name }}.conf"
    owner: root
    group: root
    mode: '0644'
  loop: "{{ virtual_hosts }}"
  when: virtual_hosts is defined and virtual_hosts | length > 0
  notify: restart webserver
  tags: configure

- name: Enable virtual hosts
  file:
    src: "/etc/{{ webserver_software }}/sites-available/{{ item.name }}.conf"
    dest: "/etc/{{ webserver_software }}/sites-enabled/{{ item.name }}.conf"
    state: link
  loop: "{{ virtual_hosts }}"
  when: virtual_hosts is defined and virtual_hosts | length > 0
  notify: restart webserver
  tags: configure

- name: Create document root directories
  file:
    path: "{{ item.document_root }}"
    state: directory
    owner: "{{ webserver_user }}"
    group: "{{ webserver_group }}"
    mode: '0755'
  loop: "{{ virtual_hosts }}"
  when: virtual_hosts is defined and virtual_hosts | length > 0
  tags: configure

- name: Generate SSL certificates (self-signed for testing)
  command: >
    openssl req -x509 -nodes -days 365 -newkey rsa:2048
    -keyout {{ webserver_ssl_private_key_path }}
    -out {{ webserver_ssl_certificate_path }}
    -subj "/C=US/ST=State/L=City/O=Organization/CN={{ webserver_domain }}"
  args:
    creates: "{{ webserver_ssl_certificate_path }}"
  when: webserver_ssl_enabled and not ansible_check_mode
  tags: configure
EOF
```

Create handlers:

```yaml
# Create roles/enterprise_webserver/handlers/main.yml
cat > roles/enterprise_webserver/handlers/main.yml << 'EOF'
---
# Enterprise Web Server Role Handlers

- name: restart webserver
  service:
    name: "{{ webserver_service_name }}"
    state: restarted
  listen: restart webserver

- name: reload webserver
  service:
    name: "{{ webserver_service_name }}"
    state: reloaded
  listen: reload webserver

- name: restart firewall
  service:
    name: "{{ firewall_service_name | default('ufw') }}"
    state: restarted
  listen: restart firewall

- name: restart fail2ban
  service:
    name: fail2ban
    state: restarted
  when: fail2ban_enabled | default(false)
  listen: restart fail2ban
EOF
```

Create OS-specific variables:

```yaml
# Create roles/enterprise_webserver/vars/Debian.yml
cat > roles/enterprise_webserver/vars/Debian.yml << 'EOF'
---
# Debian/Ubuntu specific variables
webserver_package_name: "{{ 'nginx' if webserver_software == 'nginx' else 'apache2' }}"
webserver_service_name: "{{ 'nginx' if webserver_software == 'nginx' else 'apache2' }}"
webserver_config_path: "/etc/{{ webserver_software }}"
webserver_user: "www-data"
webserver_group: "www-data"

webserver_additional_packages:
  - openssl
  - curl
  - logrotate

firewall_service_name: "ufw"
EOF

# Create roles/enterprise_webserver/vars/RedHat.yml
cat > roles/enterprise_webserver/vars/RedHat.yml << 'EOF'
---
# RedHat/CentOS specific variables
webserver_package_name: "{{ 'nginx' if webserver_software == 'nginx' else 'httpd' }}"
webserver_service_name: "{{ 'nginx' if webserver_software == 'nginx' else 'httpd' }}"
webserver_config_path: "/etc/{{ webserver_software }}"
webserver_user: "{{ 'nginx' if webserver_software == 'nginx' else 'apache' }}"
webserver_group: "{{ 'nginx' if webserver_software == 'nginx' else 'apache' }}"

webserver_additional_packages:
  - openssl
  - curl
  - logrotate

firewall_service_name: "firewalld"
EOF
```

Test the role structure:

```bash
# Test role syntax
ansible-playbook --syntax-check -i inventory.ini test-role.yml

# Create a simple test playbook
cat > test-role.yml << 'EOF'
---
- name: Test enterprise webserver role
  hosts: web_servers
  become: yes
  roles:
    - role: enterprise_webserver
      vars:
        webserver_domain: "test.local"
        webserver_admin_email: "admin@test.local"
        webserver_ssl_enabled: false
        monitoring_enabled: false
        security_hardening_enabled: false
EOF

# Validate role structure
find roles/enterprise_webserver -type f -name "*.yml" -exec ansible-lint {} \; 2>/dev/null || echo "Ansible-lint not available, skipping"
```

## Exercise 2: Role Templates and Files Organization (8 minutes)

### Task: Create comprehensive templates and file organization

Create Nginx configuration template:

```jinja2
# Create roles/enterprise_webserver/templates/nginx/nginx.conf.j2
mkdir -p roles/enterprise_webserver/templates/nginx
cat > roles/enterprise_webserver/templates/nginx/nginx.conf.j2 << 'EOF'
# {{ ansible_managed }}
# Enterprise Nginx Configuration
# Generated for {{ webserver_domain }} on {{ ansible_date_time.iso8601 }}

user {{ webserver_user }};
worker_processes {{ webserver_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ webserver_worker_connections }};
    use epoll;
    multi_accept on;
}

http {
    # Basic Settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout {{ webserver_keepalive_timeout }};
    types_hash_max_size 2048;
    client_max_body_size {{ webserver_client_max_body_size }};
    
    {% if webserver_hide_version %}
    server_tokens {{ webserver_server_tokens }};
    {% endif %}

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # SSL Settings
    {% if webserver_ssl_enabled %}
    ssl_protocols {{ webserver_ssl_protocols | join(' ') }};
    ssl_prefer_server_ciphers on;
    ssl_ciphers {{ webserver_ssl_ciphers }};
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    {% endif %}

    # Gzip Settings
    {% if webserver_gzip_enabled %}
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types {{ webserver_gzip_types | join(' ') }};
    {% endif %}

    # Logging Settings
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log;

    # Security Headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Upstream Configuration
    {% if upstream_servers is defined and upstream_servers | length > 0 %}
    {% for upstream in upstream_servers %}
    upstream {{ upstream.name }} {
        {% for server in upstream.servers %}
        server {{ server }};
        {% endfor %}
    }
    {% endfor %}
    {% endif %}

    # Default server configuration
    server {
        listen {{ webserver_port }} default_server;
        {% if webserver_ssl_enabled %}
        listen {{ webserver_ssl_port }} ssl default_server;
        ssl_certificate {{ webserver_ssl_certificate_path }};
        ssl_certificate_key {{ webserver_ssl_private_key_path }};
        {% endif %}

        server_name {{ webserver_domain }};
        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        # Health check endpoint
        {% if health_check_enabled %}
        location {{ health_check_path }} {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        {% endif %}

        # Custom error pages
        {% if custom_error_pages_enabled %}
        {% for code, page in error_pages.items() %}
        error_page {{ code }} {{ page }};
        {% endfor %}
        {% endif %}

        location / {
            try_files $uri $uri/ =404;
        }
    }

    # Include additional configurations
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
EOF
```

Create virtual host template:

```jinja2
# Create roles/enterprise_webserver/templates/nginx/vhost.conf.j2
cat > roles/enterprise_webserver/templates/nginx/vhost.conf.j2 << 'EOF'
# {{ ansible_managed }}
# Virtual Host Configuration for {{ item.name }}

server {
    listen {{ webserver_port }};
    {% if item.ssl_enabled | default(webserver_ssl_enabled) %}
    listen {{ webserver_ssl_port }} ssl;
    ssl_certificate {{ webserver_ssl_certificate_path }};
    ssl_certificate_key {{ webserver_ssl_private_key_path }};
    {% endif %}

    server_name {{ item.name }};
    root {{ item.document_root }};
    index index.html index.htm index.php;

    # Logging
    access_log /var/log/nginx/{{ item.name }}_access.log main;
    error_log /var/log/nginx/{{ item.name }}_error.log;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    {% if item.ssl_enabled | default(webserver_ssl_enabled) %}
    # SSL redirect
    if ($scheme != "https") {
        return 301 https://$host$request_uri;
    }
    {% endif %}

    location / {
        try_files $uri $uri/ =404;
        {% if item.upstream is defined %}
        proxy_pass http://{{ item.upstream }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        {% endif %}
    }

    {% if item.php_enabled | default(false) %}
    # PHP configuration
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
    }
    {% endif %}

    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
}
EOF
```

Create static files:

```bash
# Create static files
mkdir -p roles/enterprise_webserver/files/{html,ssl,scripts}

# Create default index page
cat > roles/enterprise_webserver/files/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Enterprise Web Server</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f4f4f4; }
        .container { background: white; padding: 30px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        h1 { color: #333; }
        .status { color: #28a745; font-weight: bold; }
        .info { background: #e9ecef; padding: 15px; border-radius: 4px; margin: 20px 0; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Enterprise Web Server</h1>
        <p class="status">✓ Server is running successfully</p>
        <div class="info">
            <h3>Server Information</h3>
            <p><strong>Deployed by:</strong> Ansible Enterprise Web Server Role</p>
            <p><strong>Version:</strong> 1.0.0</p>
            <p><strong>Environment:</strong> Production Ready</p>
        </div>
        <p>This server has been configured with enterprise-grade security and monitoring.</p>
    </div>
</body>
</html>
EOF

# Create custom error pages
cat > roles/enterprise_webserver/files/html/404.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Page Not Found</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 100px; }
        h1 { color: #e74c3c; }
    </style>
</head>
<body>
    <h1>404 - Page Not Found</h1>
    <p>The requested page could not be found.</p>
</body>
</html>
EOF

# Create health check script
cat > roles/enterprise_webserver/files/scripts/health_check.sh << 'EOF'
#!/bin/bash
# Enterprise Web Server Health Check Script

WEBSERVER_SERVICE="nginx"
WEBSERVER_PORT="80"
WEBSERVER_SSL_PORT="443"

echo "=== Enterprise Web Server Health Check ==="
echo "Timestamp: $(date)"
echo

# Check service status
echo "1. Service Status:"
if systemctl is-active --quiet $WEBSERVER_SERVICE; then
    echo "   ✓ $WEBSERVER_SERVICE is running"
else
    echo "   ✗ $WEBSERVER_SERVICE is not running"
    exit 1
fi

# Check port connectivity
echo "2. Port Connectivity:"
if netstat -tuln | grep -q ":$WEBSERVER_PORT "; then
    echo "   ✓ Port $WEBSERVER_PORT is listening"
else
    echo "   ✗ Port $WEBSERVER_PORT is not listening"
fi

if netstat -tuln | grep -q ":$WEBSERVER_SSL_PORT "; then
    echo "   ✓ Port $WEBSERVER_SSL_PORT is listening"
else
    echo "   ⚠ Port $WEBSERVER_SSL_PORT is not listening (SSL may be disabled)"
fi

# Check HTTP response
echo "3. HTTP Response:"
if curl -s -o /dev/null -w "%{http_code}" http://localhost:$WEBSERVER_PORT | grep -q "200"; then
    echo "   ✓ HTTP response is healthy"
else
    echo "   ✗ HTTP response is not healthy"
fi

echo
echo "Health check completed successfully!"
EOF

chmod +x roles/enterprise_webserver/files/scripts/health_check.sh
```

## Exercise 3: Role Testing and Validation (7 minutes)

### Task: Create comprehensive role testing structure

Create test playbooks:

```yaml
# Create roles/enterprise_webserver/tests/test.yml
mkdir -p roles/enterprise_webserver/tests
cat > roles/enterprise_webserver/tests/test.yml << 'EOF'
---
# Test playbook for enterprise_webserver role
- hosts: localhost
  remote_user: root
  become: yes
  vars:
    webserver_domain: "test.example.com"
    webserver_admin_email: "test@example.com"
    webserver_ssl_enabled: false
    monitoring_enabled: false
    security_hardening_enabled: false
  roles:
    - enterprise_webserver
EOF

# Create test inventory
cat > roles/enterprise_webserver/tests/inventory << 'EOF'
localhost ansible_connection=local
EOF
```

Create validation tasks:

```yaml
# Create roles/enterprise_webserver/tasks/validate.yml
cat > roles/enterprise_webserver/tasks/validate.yml << 'EOF'
---
# Validation tasks for enterprise webserver role

- name: Validate web server service is running
  service_facts:
  
- name: Check if web server service is active
  assert:
    that:
      - ansible_facts.services[webserver_service_name + '.service'].state == 'running'
    fail_msg: "Web server service {{ webserver_service_name }} is not running"
    success_msg: "Web server service {{ webserver_service_name }} is running"

- name: Validate web server is listening on configured ports
  wait_for:
    port: "{{ item }}"
    host: "{{ ansible_default_ipv4.address | default('127.0.0.1') }}"
    timeout: 10
  loop:
    - "{{ webserver_port }}"
    - "{{ webserver_ssl_port if webserver_ssl_enabled else [] }}"
  when: item != []

- name: Test HTTP response
  uri:
    url: "http://{{ ansible_default_ipv4.address | default('127.0.0.1') }}:{{ webserver_port }}"
    method: GET
    status_code: 200
  register: http_response
  
- name: Validate HTTP response
  assert:
    that:
      - http_response.status == 200
    fail_msg: "HTTP response test failed"
    success_msg: "HTTP response test passed"

- name: Test health check endpoint
  uri:
    url: "http://{{ ansible_default_ipv4.address | default('127.0.0.1') }}:{{ webserver_port }}{{ health_check_path }}"
    method: GET
    status_code: 200
  when: health_check_enabled
  register: health_response

- name: Validate configuration files exist
  stat:
    path: "{{ item }}"
  register: config_files
  loop:
    - "/etc/{{ webserver_software }}/{{ webserver_software }}.conf"
    - "{{ webserver_ssl_certificate_path if webserver_ssl_enabled else '/dev/null' }}"
  when: item != '/dev/null'

- name: Check configuration file syntax
  command: "{{ webserver_software }} -t"
  register: config_test
  changed_when: false
  failed_when: config_test.rc != 0

- name: Display validation summary
  debug:
    msg: |
      Validation Summary:
      - Service Status: {{ ansible_facts.services[webserver_service_name + '.service'].state }}
      - HTTP Response: {{ http_response.status }}
      - Configuration Test: {{ 'PASSED' if config_test.rc == 0 else 'FAILED' }}
      - Health Check: {{ 'ENABLED' if health_check_enabled else 'DISABLED' }}
      - SSL Enabled: {{ webserver_ssl_enabled }}
EOF
```

Create example usage:

```yaml
# Create roles/enterprise_webserver/examples/basic-usage.yml
mkdir -p roles/enterprise_webserver/examples
cat > roles/enterprise_webserver/examples/basic-usage.yml << 'EOF'
---
# Basic usage example for enterprise_webserver role
- name: Deploy basic web server
  hosts: web_servers
  become: yes
  roles:
    - role: enterprise_webserver
      vars:
        webserver_domain: "www.example.com"
        webserver_admin_email: "admin@example.com"
        webserver_ssl_enabled: true
        monitoring_enabled: true
EOF

# Create advanced usage example
cat > roles/enterprise_webserver/examples/advanced-usage.yml << 'EOF'
---
# Advanced usage example with multiple virtual hosts
- name: Deploy advanced web server configuration
  hosts: web_servers
  become: yes
  roles:
    - role: enterprise_webserver
      vars:
        webserver_domain: "main.example.com"
        webserver_admin_email: "devops@example.com"
        webserver_ssl_enabled: true
        monitoring_enabled: true
        security_hardening_enabled: true
        
        virtual_hosts:
          - name: "app1.example.com"
            document_root: "/var/www/app1"
            ssl_enabled: true
          - name: "app2.example.com"
            document_root: "/var/www/app2"
            ssl_enabled: true
            upstream: "backend"
        
        upstream_servers:
          - name: "backend"
            servers:
              - "192.168.1.10:8080"
              - "192.168.1.11:8080"
            method: "round_robin"
        
        webserver_worker_processes: 8
        webserver_worker_connections: 2048
EOF
```

Test the complete role:

```bash
# Run syntax check
ansible-playbook --syntax-check roles/enterprise_webserver/tests/test.yml -i roles/enterprise_webserver/tests/inventory

# Run the test (dry run)
ansible-playbook --check roles/enterprise_webserver/tests/test.yml -i roles/enterprise_webserver/tests/inventory

# Display role structure
echo "=== Final Role Structure ==="
tree roles/enterprise_webserver

# Validate role with ansible-galaxy
ansible-galaxy role info roles/enterprise_webserver 2>/dev/null || echo "Role validation completed"
```

## Verification and Discussion

### 1. Check Results
```bash
# Examine the complete role structure
find roles/enterprise_webserver -type f | sort

# Check role metadata
cat roles/enterprise_webserver/meta/main.yml

# Review documentation
head -50 roles/enterprise_webserver/README.md

# Test role syntax
ansible-playbook --syntax-check roles/enterprise_webserver/tests/test.yml -i roles/enterprise_webserver/tests/inventory
```

### 2. Discussion Points
- How do you organize complex roles in your current environment?
- What standards do you follow for role documentation and testing?
- How do you handle role versioning and distribution?
- What are the benefits of this structured approach versus simpler roles?

### 3. Clean Up
```bash
# Keep the role for next exercises
# rm -rf roles/
```

## Key Takeaways
- Proper role structure improves maintainability and reusability
- Comprehensive documentation is essential for enterprise roles
- OS-specific variables enable cross-platform compatibility
- Template organization and file structure impact role clarity
- Testing and validation ensure role reliability
- Examples and documentation facilitate role adoption

## Next Steps
Proceed to Lab 5.2: Role Dependencies and Relationships

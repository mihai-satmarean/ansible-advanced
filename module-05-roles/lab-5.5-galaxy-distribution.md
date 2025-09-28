# Lab 5.5: Role Distribution and Galaxy Integration

## Objective
Package, distribute, and manage roles using Ansible Galaxy, including private Galaxy servers and enterprise distribution strategies.

## Duration
20 minutes

## Prerequisites
- Completed Labs 5.1-5.4
- Understanding of package management and distribution
- Access to Ansible Galaxy (public or private)

## Lab Setup

```bash
cd ~/ansible-labs/module-05
mkdir -p lab-5.5
cd lab-5.5

# Create inventory for testing
cat > inventory.ini << 'EOF'
[test_servers]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Preparing Roles for Galaxy Distribution (8 minutes)

### Task: Prepare roles for publication to Ansible Galaxy

Create a distribution-ready role:

```bash
# Create a new role for distribution
ansible-galaxy init roles/company_nginx

# Update role metadata for Galaxy
cat > roles/company_nginx/meta/main.yml << 'EOF'
---
galaxy_info:
  # Role identification
  role_name: nginx
  namespace: company
  author: DevOps Team
  description: Enterprise-grade Nginx web server role with security and monitoring
  company: Enterprise Corp
  
  # Licensing and support
  license: MIT
  min_ansible_version: 2.9
  
  # Platform support
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: EL
      versions:
        - 8
        - 9
    - name: Debian
      versions:
        - bullseye
        - bookworm
  
  # Galaxy metadata
  galaxy_tags:
    - web
    - nginx
    - webserver
    - proxy
    - loadbalancer
    - ssl
    - security
    - monitoring
    - enterprise
  
  # GitHub integration
  github_branch: main
  
  # Quality and maintenance
  standalone: false
  
dependencies:
  - role: company.security_baseline
    when: nginx_security_enabled | default(true)

collections:
  - community.general
  - ansible.posix
EOF
```

Create comprehensive role documentation:

```markdown
# Create roles/company_nginx/README.md
cat > roles/company_nginx/README.md << 'EOF'
# Company Nginx Role

[![Ansible Galaxy](https://img.shields.io/badge/ansible--galaxy-company.nginx-blue.svg)](https://galaxy.ansible.com/company/nginx)
[![Build Status](https://github.com/company/ansible-role-nginx/workflows/CI/badge.svg)](https://github.com/company/ansible-role-nginx/actions)
[![License](https://img.shields.io/badge/license-MIT-brightgreen.svg)](LICENSE)

An enterprise-grade Ansible role for deploying and managing Nginx web servers with advanced security, monitoring, and performance optimization features.

## Features

- 🚀 **High Performance**: Optimized configurations for enterprise workloads
- 🔒 **Security Hardened**: SSL/TLS, security headers, and access controls
- 📊 **Monitoring Ready**: Integration with popular monitoring solutions
- 🔄 **Load Balancing**: Advanced load balancing and proxy configurations
- 📱 **Multi-Platform**: Support for Ubuntu, CentOS, and Debian
- 🧪 **Well Tested**: Comprehensive test suite with Molecule
- 📚 **Documented**: Extensive documentation and examples

## Requirements

- **Ansible**: >= 2.9
- **Operating Systems**: 
  - Ubuntu 20.04+ (Focal, Jammy)
  - CentOS 8+
  - Debian 11+ (Bullseye, Bookworm)
- **Privileges**: sudo/root access required
- **Python**: >= 3.6

## Quick Start

```yaml
- hosts: webservers
  become: yes
  roles:
    - role: company.nginx
      vars:
        nginx_server_name: "example.com"
        nginx_ssl_enabled: true
```

## Role Variables

### Basic Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_user` | `www-data` | Nginx process user |
| `nginx_worker_processes` | `auto` | Number of worker processes |
| `nginx_worker_connections` | `1024` | Worker connections limit |
| `nginx_server_name` | `{{ ansible_fqdn }}` | Default server name |

### SSL/TLS Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_ssl_enabled` | `false` | Enable SSL/TLS |
| `nginx_ssl_certificate` | `/etc/ssl/certs/nginx.crt` | SSL certificate path |
| `nginx_ssl_private_key` | `/etc/ssl/private/nginx.key` | SSL private key path |
| `nginx_ssl_protocols` | `['TLSv1.2', 'TLSv1.3']` | Supported SSL protocols |

### Performance Tuning

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_keepalive_timeout` | `65` | Keep-alive timeout |
| `nginx_client_max_body_size` | `64m` | Maximum request body size |
| `nginx_gzip_enabled` | `true` | Enable gzip compression |
| `nginx_cache_enabled` | `false` | Enable caching |

### Security Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_security_enabled` | `true` | Enable security features |
| `nginx_hide_version` | `true` | Hide Nginx version |
| `nginx_rate_limiting` | `false` | Enable rate limiting |
| `nginx_access_log_enabled` | `true` | Enable access logging |

### Virtual Hosts

```yaml
nginx_vhosts:
  - server_name: "app1.example.com"
    document_root: "/var/www/app1"
    ssl_enabled: true
    proxy_pass: "http://backend1"
  - server_name: "app2.example.com"
    document_root: "/var/www/app2"
    ssl_enabled: false
```

### Load Balancing

```yaml
nginx_upstreams:
  - name: "backend1"
    servers:
      - "192.168.1.10:8080"
      - "192.168.1.11:8080"
    method: "round_robin"
    health_check: true
```

## Dependencies

This role depends on:

- **company.security_baseline**: Provides security hardening (optional)

## Example Playbooks

### Basic Web Server

```yaml
---
- name: Deploy basic web server
  hosts: webservers
  become: yes
  roles:
    - role: company.nginx
      vars:
        nginx_server_name: "www.example.com"
        nginx_ssl_enabled: true
        nginx_worker_processes: 4
```

### Load Balancer with SSL

```yaml
---
- name: Deploy load balancer
  hosts: loadbalancers
  become: yes
  roles:
    - role: company.nginx
      vars:
        nginx_server_name: "lb.example.com"
        nginx_ssl_enabled: true
        nginx_upstreams:
          - name: "web_backend"
            servers:
              - "10.0.1.10:80"
              - "10.0.1.11:80"
            method: "least_conn"
        nginx_vhosts:
          - server_name: "app.example.com"
            proxy_pass: "http://web_backend"
            ssl_enabled: true
```

### High-Performance Configuration

```yaml
---
- name: Deploy high-performance web server
  hosts: webservers
  become: yes
  roles:
    - role: company.nginx
      vars:
        nginx_worker_processes: 8
        nginx_worker_connections: 2048
        nginx_keepalive_timeout: 30
        nginx_gzip_enabled: true
        nginx_cache_enabled: true
        nginx_rate_limiting: true
```

## Testing

This role includes comprehensive tests using Molecule:

```bash
# Install test dependencies
pip install molecule[docker] ansible-lint yamllint

# Run all tests
molecule test

# Run tests for specific scenarios
molecule test -s default
molecule test -s ubuntu
molecule test -s centos
```

### Test Scenarios

- **default**: Basic functionality test
- **ubuntu**: Ubuntu-specific testing  
- **centos**: CentOS-specific testing
- **ssl**: SSL/TLS configuration testing
- **loadbalancer**: Load balancing functionality

## Platform Support

| Platform | Version | Status | Notes |
|----------|---------|--------|-------|
| Ubuntu | 20.04 (Focal) | ✅ Fully Supported | Primary development platform |
| Ubuntu | 22.04 (Jammy) | ✅ Fully Supported | Latest LTS |
| CentOS | 8 | ✅ Fully Supported | Enterprise standard |
| CentOS | 9 | 🧪 Beta Support | Testing in progress |
| Debian | 11 (Bullseye) | ✅ Fully Supported | Stable release |
| Debian | 12 (Bookworm) | 🧪 Beta Support | Latest release |

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for detailed version history.

## Contributing

We welcome contributions! Please see our [Contributing Guide](CONTRIBUTING.md) for details.

### Development Setup

1. Fork the repository
2. Create a feature branch
3. Install development dependencies:
   ```bash
   pip install -r requirements-dev.txt
   ```
4. Make your changes
5. Run tests: `molecule test`
6. Submit a pull request

## Security

For security issues, please see our [Security Policy](SECURITY.md).

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

- 📖 **Documentation**: [Internal Wiki](https://wiki.company.com/ansible/nginx)
- 🐛 **Bug Reports**: [GitHub Issues](https://github.com/company/ansible-role-nginx/issues)
- 💬 **Discussion**: [GitHub Discussions](https://github.com/company/ansible-role-nginx/discussions)
- 💼 **Enterprise Support**: Contact DevOps Team

## Author Information

- **Author**: DevOps Team
- **Email**: devops@company.com
- **Organization**: Enterprise Corp
- **Maintained Since**: 2024

---

**Role Version**: 1.0.0  
**Last Updated**: 2024-01-15  
**Ansible Compatibility**: 2.9+  
**Galaxy Namespace**: company.nginx
EOF
```

Create role defaults and tasks:

```yaml
# Create roles/company_nginx/defaults/main.yml
cat > roles/company_nginx/defaults/main.yml << 'EOF'
---
# Company Nginx Role - Default Variables

# Basic configuration
nginx_user: "{{ 'www-data' if ansible_os_family == 'Debian' else 'nginx' }}"
nginx_group: "{{ nginx_user }}"
nginx_worker_processes: "auto"
nginx_worker_connections: 1024
nginx_server_name: "{{ ansible_fqdn | default(ansible_hostname) }}"

# Service configuration
nginx_service_enabled: true
nginx_service_state: "started"

# Performance settings
nginx_keepalive_timeout: 65
nginx_client_max_body_size: "64m"
nginx_sendfile: true
nginx_tcp_nopush: true
nginx_tcp_nodelay: true

# Compression
nginx_gzip_enabled: true
nginx_gzip_types:
  - "text/plain"
  - "text/css"
  - "application/json"
  - "application/javascript"
  - "text/xml"
  - "application/xml"
  - "application/xml+rss"
  - "text/javascript"

# Security settings
nginx_security_enabled: true
nginx_hide_version: true
nginx_server_tokens: "off"
nginx_rate_limiting: false
nginx_rate_limit: "10r/s"

# SSL/TLS configuration
nginx_ssl_enabled: false
nginx_ssl_certificate: "/etc/ssl/certs/{{ nginx_server_name }}.crt"
nginx_ssl_private_key: "/etc/ssl/private/{{ nginx_server_name }}.key"
nginx_ssl_protocols: ["TLSv1.2", "TLSv1.3"]
nginx_ssl_ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384"

# Logging
nginx_access_log_enabled: true
nginx_access_log_path: "/var/log/nginx/access.log"
nginx_error_log_path: "/var/log/nginx/error.log"
nginx_log_format: "main"

# Virtual hosts
nginx_vhosts: []
# Example:
# nginx_vhosts:
#   - server_name: "example.com"
#     document_root: "/var/www/example"
#     ssl_enabled: true

# Upstream servers
nginx_upstreams: []
# Example:
# nginx_upstreams:
#   - name: "backend"
#     servers:
#       - "192.168.1.10:8080"
#       - "192.168.1.11:8080"
#     method: "round_robin"

# Caching
nginx_cache_enabled: false
nginx_cache_path: "/var/cache/nginx"
nginx_cache_levels: "1:2"
nginx_cache_keys_zone: "main:10m"
nginx_cache_max_size: "1g"

# Monitoring
nginx_status_enabled: false
nginx_status_path: "/nginx_status"
nginx_status_allowed_ips: ["127.0.0.1"]
EOF

# Create roles/company_nginx/tasks/main.yml
cat > roles/company_nginx/tasks/main.yml << 'EOF'
---
# Company Nginx Role - Main Tasks

- name: Display Nginx deployment information
  debug:
    msg: |
      Deploying Nginx Web Server
      Server Name: {{ nginx_server_name }}
      SSL Enabled: {{ nginx_ssl_enabled }}
      Security Enabled: {{ nginx_security_enabled }}
      Virtual Hosts: {{ nginx_vhosts | length }}
      Upstreams: {{ nginx_upstreams | length }}

- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: Install Nginx
  package:
    name: "{{ nginx_package_name }}"
    state: present
  notify: restart nginx

- name: Create Nginx directories
  file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - /etc/nginx/sites-available
    - /etc/nginx/sites-enabled
    - /etc/nginx/conf.d
    - "{{ nginx_cache_path if nginx_cache_enabled else '/tmp' }}"
  when: item != '/tmp'

- name: Generate main Nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes
  notify: restart nginx

- name: Generate virtual host configurations
  template:
    src: vhost.conf.j2
    dest: "/etc/nginx/sites-available/{{ item.server_name }}.conf"
    owner: root
    group: root
    mode: '0644'
  loop: "{{ nginx_vhosts }}"
  notify: restart nginx

- name: Enable virtual hosts
  file:
    src: "/etc/nginx/sites-available/{{ item.server_name }}.conf"
    dest: "/etc/nginx/sites-enabled/{{ item.server_name }}.conf"
    state: link
  loop: "{{ nginx_vhosts }}"
  notify: restart nginx

- name: Generate SSL certificates (self-signed for testing)
  command: >
    openssl req -x509 -nodes -days 365 -newkey rsa:2048
    -keyout {{ nginx_ssl_private_key }}
    -out {{ nginx_ssl_certificate }}
    -subj "/C=US/ST=State/L=City/O=Organization/CN={{ nginx_server_name }}"
  args:
    creates: "{{ nginx_ssl_certificate }}"
  when: nginx_ssl_enabled and not ansible_check_mode

- name: Start and enable Nginx service
  service:
    name: nginx
    state: "{{ nginx_service_state }}"
    enabled: "{{ nginx_service_enabled }}"

- name: Verify Nginx is running
  uri:
    url: "http://localhost"
    method: GET
    status_code: 200
  register: nginx_check
  retries: 3
  delay: 5
EOF
```

Create handlers and templates:

```yaml
# Create roles/company_nginx/handlers/main.yml
cat > roles/company_nginx/handlers/main.yml << 'EOF'
---
# Company Nginx Role - Handlers

- name: restart nginx
  service:
    name: nginx
    state: restarted

- name: reload nginx
  service:
    name: nginx
    state: reloaded

- name: validate nginx config
  command: nginx -t
  changed_when: false
EOF

# Create OS-specific variables
mkdir -p roles/company_nginx/vars
cat > roles/company_nginx/vars/Debian.yml << 'EOF'
---
nginx_package_name: nginx
nginx_service_name: nginx
nginx_config_path: /etc/nginx
nginx_user: www-data
nginx_group: www-data
EOF

cat > roles/company_nginx/vars/RedHat.yml << 'EOF'
---
nginx_package_name: nginx
nginx_service_name: nginx
nginx_config_path: /etc/nginx
nginx_user: nginx
nginx_group: nginx
EOF
```

Create Galaxy requirements and build files:

```yaml
# Create galaxy.yml for collection (if needed)
cat > galaxy.yml << 'EOF'
---
namespace: company
name: nginx_collection
version: 1.0.0
readme: README.md
authors:
  - DevOps Team <devops@company.com>
description: Enterprise Nginx collection with roles and modules
license_file: LICENSE
tags:
  - nginx
  - webserver
  - enterprise
dependencies: {}
repository: https://github.com/company/ansible-nginx-collection
documentation: https://docs.company.com/ansible/nginx
homepage: https://github.com/company/ansible-nginx-collection
issues: https://github.com/company/ansible-nginx-collection/issues
build_ignore:
  - "*.tar.gz"
  - ".git"
  - ".github"
  - "tests"
  - "molecule"
EOF

# Create requirements file for role dependencies
cat > requirements.yml << 'EOF'
---
# Ansible Galaxy Requirements

roles:
  # External roles from Galaxy
  - name: geerlingguy.security
    version: ">=1.0.0"
  
  # Internal company roles
  - name: company.security_baseline
    src: https://github.com/company/ansible-role-security-baseline
    version: main
  
  - name: company.monitoring_agent
    src: https://github.com/company/ansible-role-monitoring-agent
    version: main

collections:
  - name: community.general
    version: ">=3.0.0"
  - name: ansible.posix
    version: ">=1.0.0"
EOF
```

## Exercise 2: Publishing to Ansible Galaxy (6 minutes)

### Task: Publish roles to public and private Galaxy instances

Create Galaxy publishing configuration:

```bash
# Create Galaxy configuration
mkdir -p ~/.ansible
cat > ~/.ansible/galaxy_token << 'EOF'
# Ansible Galaxy API Token
# Replace with your actual token from https://galaxy.ansible.com/me/preferences
token: your_galaxy_api_token_here
EOF

# Create Galaxy server configuration
cat > ~/.ansible/ansible.cfg << 'EOF'
[defaults]
roles_path = ./roles
collections_path = ./collections

[galaxy]
server_list = public_galaxy, private_galaxy

[galaxy_server.public_galaxy]
url = https://galaxy.ansible.com/
token = your_public_galaxy_token

[galaxy_server.private_galaxy]
url = https://galaxy.company.com/
token = your_private_galaxy_token
username = your_username
password = your_password
EOF
```

Create publishing scripts:

```bash
# Create publishing script
cat > scripts/publish-role.sh << 'EOF'
#!/bin/bash
# Role publishing script

set -e

ROLE_NAME=$1
GALAXY_SERVER=${2:-"public_galaxy"}
DRY_RUN=${3:-"false"}

if [ -z "$ROLE_NAME" ]; then
    echo "Usage: $0 <role_name> [galaxy_server] [dry_run]"
    echo "Example: $0 company_nginx public_galaxy false"
    exit 1
fi

ROLE_DIR="roles/$ROLE_NAME"

if [ ! -d "$ROLE_DIR" ]; then
    echo "Error: Role directory not found: $ROLE_DIR"
    exit 1
fi

echo "Publishing role: $ROLE_NAME to $GALAXY_SERVER"

# Validate role structure
echo "Validating role structure..."
if [ ! -f "$ROLE_DIR/meta/main.yml" ]; then
    echo "Error: meta/main.yml not found"
    exit 1
fi

if [ ! -f "$ROLE_DIR/README.md" ]; then
    echo "Error: README.md not found"
    exit 1
fi

# Run quality checks
echo "Running quality checks..."
ansible-lint "$ROLE_DIR" || {
    echo "Warning: ansible-lint found issues"
    read -p "Continue anyway? (y/N): " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        exit 1
    fi
}

# Test role syntax
echo "Testing role syntax..."
ansible-playbook --syntax-check "$ROLE_DIR/tests/test.yml" -i "$ROLE_DIR/tests/inventory" || {
    echo "Error: Syntax check failed"
    exit 1
}

# Build and publish
if [ "$DRY_RUN" = "true" ]; then
    echo "DRY RUN: Would publish $ROLE_NAME to $GALAXY_SERVER"
    ansible-galaxy role info "$ROLE_NAME" --server "$GALAXY_SERVER" || echo "Role not found on server"
else
    echo "Publishing role to Galaxy..."
    ansible-galaxy role import --server "$GALAXY_SERVER" \
        $(git config user.name || echo "company") \
        $(basename $(git rev-parse --show-toplevel) || echo "ansible-role-$ROLE_NAME")
fi

echo "Role publishing completed successfully!"
EOF

chmod +x scripts/publish-role.sh

# Create role validation script
cat > scripts/validate-role.sh << 'EOF'
#!/bin/bash
# Role validation script

set -e

ROLE_NAME=$1

if [ -z "$ROLE_NAME" ]; then
    echo "Usage: $0 <role_name>"
    exit 1
fi

ROLE_DIR="roles/$ROLE_NAME"

echo "Validating role: $ROLE_NAME"

# Check required files
REQUIRED_FILES=(
    "meta/main.yml"
    "README.md"
    "defaults/main.yml"
    "tasks/main.yml"
    "handlers/main.yml"
)

for file in "${REQUIRED_FILES[@]}"; do
    if [ ! -f "$ROLE_DIR/$file" ]; then
        echo "❌ Missing required file: $file"
        exit 1
    else
        echo "✅ Found: $file"
    fi
done

# Validate meta/main.yml
echo "Validating meta/main.yml..."
python3 -c "
import yaml
import sys

try:
    with open('$ROLE_DIR/meta/main.yml', 'r') as f:
        meta = yaml.safe_load(f)
    
    required_fields = ['galaxy_info', 'dependencies']
    galaxy_required = ['role_name', 'author', 'description', 'license', 'min_ansible_version']
    
    for field in required_fields:
        if field not in meta:
            print(f'❌ Missing field in meta: {field}')
            sys.exit(1)
    
    for field in galaxy_required:
        if field not in meta['galaxy_info']:
            print(f'❌ Missing galaxy_info field: {field}')
            sys.exit(1)
    
    print('✅ meta/main.yml is valid')
    
except Exception as e:
    print(f'❌ Error validating meta/main.yml: {e}')
    sys.exit(1)
"

# Run ansible-lint
echo "Running ansible-lint..."
if command -v ansible-lint >/dev/null 2>&1; then
    ansible-lint "$ROLE_DIR" && echo "✅ ansible-lint passed" || echo "⚠️ ansible-lint found issues"
else
    echo "⚠️ ansible-lint not available"
fi

# Run yamllint
echo "Running yamllint..."
if command -v yamllint >/dev/null 2>&1; then
    yamllint "$ROLE_DIR" && echo "✅ yamllint passed" || echo "⚠️ yamllint found issues"
else
    echo "⚠️ yamllint not available"
fi

echo "Role validation completed!"
EOF

chmod +x scripts/validate-role.sh
```

Test role validation and publishing:

```bash
# Validate the role
./scripts/validate-role.sh company_nginx

# Test publishing (dry run)
./scripts/publish-role.sh company_nginx public_galaxy true

# Install role from Galaxy (test)
ansible-galaxy role install company.nginx --force
ansible-galaxy role list | grep nginx
```

## Exercise 3: Private Galaxy and Enterprise Distribution (6 minutes)

### Task: Set up private Galaxy server and enterprise distribution workflows

Create private Galaxy server configuration:

```yaml
# Create docker-compose.yml for private Galaxy
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  galaxy:
    image: ansible/galaxy:latest
    ports:
      - "8080:8000"
    environment:
      - GALAXY_SECRET_KEY=your-secret-key-here
      - GALAXY_DB_URL=postgresql://galaxy:galaxy@db:5432/galaxy
      - GALAXY_REDIS_URL=redis://redis:6379/0
    depends_on:
      - db
      - redis
    volumes:
      - galaxy_data:/var/lib/galaxy
      - ./galaxy_settings.py:/etc/galaxy/settings.py
    command: >
      sh -c "
        python manage.py migrate &&
        python manage.py collectstatic --noinput &&
        gunicorn galaxy.wsgi:application --bind 0.0.0.0:8000
      "

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=galaxy
      - POSTGRES_USER=galaxy
      - POSTGRES_PASSWORD=galaxy
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:6-alpine
    volumes:
      - redis_data:/data

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - galaxy

volumes:
  galaxy_data:
  postgres_data:
  redis_data:
EOF

# Create Galaxy settings
cat > galaxy_settings.py << 'EOF'
# Private Galaxy Settings

import os

# Basic settings
SECRET_KEY = os.environ.get('GALAXY_SECRET_KEY', 'your-secret-key-here')
DEBUG = False
ALLOWED_HOSTS = ['*']

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'galaxy',
        'USER': 'galaxy',
        'PASSWORD': 'galaxy',
        'HOST': 'db',
        'PORT': '5432',
    }
}

# Redis
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://redis:6379/0',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# Authentication
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'social_core.backends.github.GithubOAuth2',
    'social_core.backends.google.GoogleOAuth2',
]

# GitHub OAuth (optional)
SOCIAL_AUTH_GITHUB_KEY = os.environ.get('GITHUB_CLIENT_ID', '')
SOCIAL_AUTH_GITHUB_SECRET = os.environ.get('GITHUB_CLIENT_SECRET', '')

# Static files
STATIC_URL = '/static/'
STATIC_ROOT = '/var/lib/galaxy/static'

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = '/var/lib/galaxy/media'

# Logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'filename': '/var/log/galaxy/galaxy.log',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['file'],
            'level': 'INFO',
            'propagate': True,
        },
    },
}

# Security
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'
EOF
```

Create enterprise distribution workflows:

```bash
# Create enterprise distribution script
cat > scripts/enterprise-distribute.sh << 'EOF'
#!/bin/bash
# Enterprise role distribution script

set -e

ROLE_NAME=$1
DISTRIBUTION_TYPE=${2:-"all"}  # internal, external, all
VERSION=${3:-"auto"}

if [ -z "$ROLE_NAME" ]; then
    echo "Usage: $0 <role_name> [distribution_type] [version]"
    echo "Distribution types: internal, external, all"
    exit 1
fi

ROLE_DIR="roles/$ROLE_NAME"

echo "Distributing role: $ROLE_NAME"
echo "Distribution type: $DISTRIBUTION_TYPE"
echo "Version: $VERSION"

# Validate role
echo "Validating role..."
./scripts/validate-role.sh "$ROLE_NAME"

# Determine version
if [ "$VERSION" = "auto" ]; then
    VERSION=$(grep -E "^\s*version:" "$ROLE_DIR/meta/main.yml" | sed 's/.*version:\s*//' | tr -d '"' | tr -d "'")
    if [ -z "$VERSION" ]; then
        VERSION="1.0.0"
    fi
fi

echo "Using version: $VERSION"

# Create distribution package
echo "Creating distribution package..."
PACKAGE_NAME="${ROLE_NAME}-${VERSION}.tar.gz"
tar -czf "dist/$PACKAGE_NAME" -C roles "$ROLE_NAME"

# Internal distribution
if [ "$DISTRIBUTION_TYPE" = "internal" ] || [ "$DISTRIBUTION_TYPE" = "all" ]; then
    echo "Distributing to internal Galaxy..."
    ansible-galaxy role import --server private_galaxy \
        company "$ROLE_NAME" || echo "Internal distribution failed"
    
    # Upload to internal artifact repository
    echo "Uploading to internal artifact repository..."
    curl -X POST \
        -H "Authorization: Bearer $INTERNAL_REPO_TOKEN" \
        -F "file=@dist/$PACKAGE_NAME" \
        "$INTERNAL_REPO_URL/roles/$ROLE_NAME/$VERSION" || echo "Artifact upload failed"
fi

# External distribution
if [ "$DISTRIBUTION_TYPE" = "external" ] || [ "$DISTRIBUTION_TYPE" = "all" ]; then
    echo "Distributing to public Galaxy..."
    ansible-galaxy role import --server public_galaxy \
        company "$ROLE_NAME" || echo "External distribution failed"
fi

# Update role registry
echo "Updating role registry..."
python3 scripts/update-role-registry.py "$ROLE_NAME" "$VERSION" "$DISTRIBUTION_TYPE"

echo "Distribution completed successfully!"
EOF

chmod +x scripts/enterprise-distribute.sh

# Create role registry updater
cat > scripts/update-role-registry.py << 'EOF'
#!/usr/bin/env python3
"""
Role registry updater
Updates internal role registry with new role versions
"""

import json
import sys
import os
from datetime import datetime

def update_registry(role_name, version, distribution_type):
    """Update role registry with new version"""
    
    registry_file = "role-registry.json"
    
    # Load existing registry
    if os.path.exists(registry_file):
        with open(registry_file, 'r') as f:
            registry = json.load(f)
    else:
        registry = {"roles": {}, "last_updated": None}
    
    # Update role information
    if role_name not in registry["roles"]:
        registry["roles"][role_name] = {
            "name": role_name,
            "namespace": "company",
            "versions": [],
            "latest_version": version,
            "distribution_types": [],
            "created_at": datetime.now().isoformat(),
            "updated_at": datetime.now().isoformat()
        }
    
    role_info = registry["roles"][role_name]
    
    # Add version if not exists
    version_info = {
        "version": version,
        "distribution_type": distribution_type,
        "published_at": datetime.now().isoformat(),
        "status": "active"
    }
    
    # Check if version already exists
    existing_version = next((v for v in role_info["versions"] if v["version"] == version), None)
    if existing_version:
        existing_version.update(version_info)
    else:
        role_info["versions"].append(version_info)
    
    # Update latest version
    role_info["latest_version"] = version
    role_info["updated_at"] = datetime.now().isoformat()
    
    # Add distribution type if not present
    if distribution_type not in role_info["distribution_types"]:
        role_info["distribution_types"].append(distribution_type)
    
    # Update registry metadata
    registry["last_updated"] = datetime.now().isoformat()
    
    # Save registry
    with open(registry_file, 'w') as f:
        json.dump(registry, f, indent=2)
    
    print(f"Updated registry for {role_name} version {version}")

if __name__ == "__main__":
    if len(sys.argv) != 4:
        print("Usage: update-role-registry.py <role_name> <version> <distribution_type>")
        sys.exit(1)
    
    role_name = sys.argv[1]
    version = sys.argv[2]
    distribution_type = sys.argv[3]
    
    update_registry(role_name, version, distribution_type)
EOF

chmod +x scripts/update-role-registry.py

# Create role discovery script
cat > scripts/discover-roles.sh << 'EOF'
#!/bin/bash
# Role discovery script

echo "=== Enterprise Role Discovery ==="
echo

# Check local roles
echo "Local Roles:"
find roles/ -name "meta" -type d | while read meta_dir; do
    role_dir=$(dirname "$meta_dir")
    role_name=$(basename "$role_dir")
    
    if [ -f "$meta_dir/main.yml" ]; then
        version=$(grep -E "^\s*version:" "$meta_dir/main.yml" | sed 's/.*version:\s*//' | tr -d '"' | tr -d "'")
        description=$(grep -E "^\s*description:" "$meta_dir/main.yml" | sed 's/.*description:\s*//' | tr -d '"' | tr -d "'")
        
        echo "  - $role_name (v$version): $description"
    fi
done

echo

# Check installed roles
echo "Installed Roles:"
ansible-galaxy role list | grep -v "^#" | while read line; do
    if [ -n "$line" ]; then
        echo "  - $line"
    fi
done

echo

# Check role registry
if [ -f "role-registry.json" ]; then
    echo "Registry Roles:"
    python3 -c "
import json
with open('role-registry.json', 'r') as f:
    registry = json.load(f)

for role_name, role_info in registry['roles'].items():
    print(f'  - {role_name} (v{role_info[\"latest_version\"]}): {role_info.get(\"description\", \"No description\")}')
    print(f'    Distribution: {role_info[\"distribution_types\"]}')
    print(f'    Versions: {len(role_info[\"versions\"])}')
    print()
"
fi
EOF

chmod +x scripts/discover-roles.sh
```

Create role installation and management tools:

```bash
# Create role installer script
cat > scripts/install-roles.sh << 'EOF'
#!/bin/bash
# Enterprise role installer

set -e

REQUIREMENTS_FILE=${1:-"requirements.yml"}
GALAXY_SERVER=${2:-"private_galaxy"}
FORCE=${3:-"false"}

echo "Installing roles from: $REQUIREMENTS_FILE"
echo "Galaxy server: $GALAXY_SERVER"

if [ ! -f "$REQUIREMENTS_FILE" ]; then
    echo "Error: Requirements file not found: $REQUIREMENTS_FILE"
    exit 1
fi

# Install roles
FORCE_FLAG=""
if [ "$FORCE" = "true" ]; then
    FORCE_FLAG="--force"
fi

ansible-galaxy install -r "$REQUIREMENTS_FILE" \
    --server "$GALAXY_SERVER" \
    $FORCE_FLAG \
    --roles-path ./roles

echo "Role installation completed!"

# List installed roles
echo
echo "Installed roles:"
ansible-galaxy role list
EOF

chmod +x scripts/install-roles.sh

# Test the distribution workflow
mkdir -p dist
./scripts/enterprise-distribute.sh company_nginx internal auto
```

Test role discovery and management:

```bash
# Test role discovery
./scripts/discover-roles.sh

# Test role installation
./scripts/install-roles.sh requirements.yml private_galaxy true

# Check role registry
cat role-registry.json 2>/dev/null || echo "No registry file created yet"
```

## Verification and Discussion

### 1. Check Results
```bash
# Validate role structure
echo "=== Role Structure Validation ==="
find roles/company_nginx -type f | sort

# Check distribution artifacts
echo "=== Distribution Artifacts ==="
ls -la dist/ 2>/dev/null || echo "No distribution artifacts"

# Test role functionality
echo "=== Role Functionality Test ==="
ansible-playbook --syntax-check -i inventory.ini test-galaxy-role.yml 2>/dev/null || echo "No test playbook"

# Check Galaxy configuration
echo "=== Galaxy Configuration ==="
ls -la ~/.ansible/ 2>/dev/null || echo "No Galaxy configuration"
```

### 2. Discussion Points
- How do you manage role distribution in your current environment?
- What are the benefits of private Galaxy servers for enterprise use?
- How do you handle role versioning and dependency management?
- What approval processes do you have for role publication?

### 3. Clean Up
```bash
# Clean up test files
rm -rf dist/ role-registry.json
# Keep roles and scripts for reference
```

## Key Takeaways
- Proper role packaging is essential for Galaxy distribution
- Private Galaxy servers provide enterprise control and security
- Automated distribution workflows improve consistency and reliability
- Role registries enable discovery and governance
- Version management and dependency tracking are crucial for enterprise adoption
- Quality gates ensure only tested roles are distributed

## Module 5 Summary

### What We Covered
1. **Role Architecture and Structure** - Comprehensive role organization and best practices
2. **Role Dependencies and Relationships** - Complex dependency patterns and inter-role communication
3. **Role Reusability and Parameterization** - Advanced parameterization for maximum flexibility
4. **Enterprise Role Development Patterns** - Testing, CI/CD, and governance frameworks
5. **Role Distribution and Galaxy Integration** - Publishing and managing roles at enterprise scale

### Key Skills Developed
- Designing and implementing enterprise-grade role architectures
- Managing complex role dependencies and relationships
- Creating highly reusable and parameterized roles
- Implementing comprehensive testing and CI/CD pipelines
- Establishing role governance and approval processes
- Publishing and distributing roles through Galaxy

### Next Steps
You now have comprehensive knowledge of advanced Ansible roles for enterprise environments. These patterns and practices will enable you to build scalable, maintainable automation solutions that meet enterprise requirements for quality, security, and governance.

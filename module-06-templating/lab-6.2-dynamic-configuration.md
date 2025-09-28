# Lab 6.2: Dynamic Configuration Generation

## Objective
Generate complex configuration files from dynamic data sources, implementing enterprise patterns for configuration management and multi-environment deployments.

## Duration
30 minutes

## Prerequisites
- Completed Lab 6.1
- Understanding of various configuration file formats
- Knowledge of enterprise configuration patterns

## Lab Setup

```bash
cd ~/ansible-labs/module-06
mkdir -p lab-6.2
cd lab-6.2

# Create inventory for testing
cat > inventory.ini << 'EOF'
[web_servers]
web1 ansible_host=localhost ansible_connection=local environment=production
web2 ansible_host=localhost ansible_connection=local environment=staging

[database_servers]
db1 ansible_host=localhost ansible_connection=local environment=production

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Multi-Format Configuration Generation (12 minutes)

### Task: Generate configurations in various formats (YAML, JSON, INI, XML, TOML)

Create comprehensive configuration data:

```yaml
# Create config-data.yml
cat > config-data.yml << 'EOF'
---
# Enterprise Configuration Data

# Global application settings
application:
  name: "Enterprise Web Platform"
  version: "3.2.1"
  build: "20240115-1430"
  environment: "{{ environment | default('production') }}"
  debug_mode: "{{ 'true' if environment == 'development' else 'false' }}"
  
# Database configuration
database:
  primary:
    host: "{{ 'localhost' if environment == 'development' else 'prod-db-cluster.internal' }}"
    port: 5432
    name: "{{ application.name | lower | replace(' ', '_') }}_{{ environment }}"
    username: "app_user"
    password: "{{ vault_db_password | default('changeme') }}"
    pool_size: "{{ 5 if environment == 'development' else 20 }}"
    timeout: 30
    ssl_mode: "{{ 'disable' if environment == 'development' else 'require' }}"
  
  replica:
    enabled: "{{ environment in ['staging', 'production'] }}"
    host: "{{ 'prod-db-replica.internal' if environment == 'production' else 'stage-db-replica.internal' }}"
    port: 5432
    lag_threshold: 5

# Cache configuration
cache:
  redis:
    enabled: true
    host: "{{ 'localhost' if environment == 'development' else 'redis-cluster.internal' }}"
    port: 6379
    database: "{{ 0 if environment == 'development' else 1 }}"
    password: "{{ vault_redis_password | default('') }}"
    ttl: "{{ 300 if environment == 'development' else 3600 }}"
    cluster_mode: "{{ environment == 'production' }}"
    
# Web server configuration
web_server:
  nginx:
    worker_processes: "{{ 2 if environment == 'development' else 'auto' }}"
    worker_connections: "{{ 512 if environment == 'development' else 2048 }}"
    keepalive_timeout: 65
    client_max_body_size: "{{ '10m' if environment == 'development' else '100m' }}"
    
    # SSL configuration
    ssl:
      enabled: "{{ environment != 'development' }}"
      certificate: "/etc/ssl/certs/{{ application.name | lower | replace(' ', '-') }}.crt"
      private_key: "/etc/ssl/private/{{ application.name | lower | replace(' ', '-') }}.key"
      protocols: ["TLSv1.2", "TLSv1.3"]
      ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
    
    # Compression
    gzip:
      enabled: true
      level: 6
      types: ["text/plain", "text/css", "application/json", "application/javascript"]

# Logging configuration
logging:
  level: "{{ 'DEBUG' if environment == 'development' else 'INFO' if environment == 'staging' else 'WARN' }}"
  format: "json"
  outputs:
    file:
      enabled: true
      path: "/var/log/{{ application.name | lower | replace(' ', '-') }}/app.log"
      max_size: "{{ '10MB' if environment == 'development' else '100MB' }}"
      max_files: "{{ 3 if environment == 'development' else 10 }}"
    syslog:
      enabled: "{{ environment in ['staging', 'production'] }}"
      facility: "local0"
      tag: "{{ application.name | lower | replace(' ', '-') }}"
    elasticsearch:
      enabled: "{{ environment == 'production' }}"
      host: "elasticsearch.internal"
      port: 9200
      index: "{{ application.name | lower | replace(' ', '-') }}-logs"

# Monitoring configuration
monitoring:
  metrics:
    enabled: true
    port: 9090
    path: "/metrics"
    interval: "{{ 5 if environment == 'development' else 30 }}"
    
  health_checks:
    enabled: true
    endpoint: "/health"
    timeout: 5
    interval: "{{ 10 if environment == 'development' else 30 }}"
    
  alerts:
    enabled: "{{ environment in ['staging', 'production'] }}"
    channels:
      email: ["ops@company.com"]
      slack: "{{ '#alerts-prod' if environment == 'production' else '#alerts-stage' }}"
      pagerduty: "{{ 'prod-service-key' if environment == 'production' else '' }}"

# Security configuration
security:
  authentication:
    method: "{{ 'basic' if environment == 'development' else 'oauth2' }}"
    session_timeout: "{{ 3600 if environment == 'development' else 1800 }}"
    max_login_attempts: 5
    lockout_duration: 300
    
  authorization:
    rbac_enabled: true
    default_role: "user"
    admin_roles: ["admin", "operator"]
    
  encryption:
    at_rest: "{{ environment in ['staging', 'production'] }}"
    in_transit: "{{ environment != 'development' }}"
    algorithm: "AES-256-GCM"

# Feature flags
features:
  new_ui: "{{ environment in ['development', 'staging'] }}"
  advanced_analytics: "{{ environment == 'production' }}"
  beta_features: "{{ environment == 'development' }}"
  maintenance_mode: false
  rate_limiting: "{{ environment in ['staging', 'production'] }}"

# Environment-specific overrides
environment_overrides:
  development:
    database:
      pool_size: 5
    cache:
      ttl: 60
    logging:
      level: "DEBUG"
  staging:
    database:
      pool_size: 10
    monitoring:
      alerts:
        enabled: false
  production:
    database:
      pool_size: 50
    security:
      encryption:
        at_rest: true
        in_transit: true
EOF
```

Create YAML configuration template:

```jinja2
# Create templates/app-config.yml.j2
mkdir -p templates
cat > templates/app-config.yml.j2 << 'EOF'
# {{ ansible_managed }}
# Application Configuration - YAML Format
# Generated for {{ environment | default('unknown') }} environment

application:
  name: {{ application.name }}
  version: {{ application.version }}
  build: {{ application.build }}
  environment: {{ application.environment }}
  debug_mode: {{ application.debug_mode | bool }}

database:
  primary:
    host: {{ database.primary.host }}
    port: {{ database.primary.port }}
    name: {{ database.primary.name }}
    username: {{ database.primary.username }}
    password: {{ database.primary.password }}
    pool_size: {{ database.primary.pool_size }}
    timeout: {{ database.primary.timeout }}
    ssl_mode: {{ database.primary.ssl_mode }}
  
  {% if database.replica.enabled | bool %}
  replica:
    enabled: {{ database.replica.enabled | bool }}
    host: {{ database.replica.host }}
    port: {{ database.replica.port }}
    lag_threshold: {{ database.replica.lag_threshold }}
  {% endif %}

cache:
  redis:
    enabled: {{ cache.redis.enabled | bool }}
    host: {{ cache.redis.host }}
    port: {{ cache.redis.port }}
    database: {{ cache.redis.database }}
    {% if cache.redis.password %}
    password: {{ cache.redis.password }}
    {% endif %}
    ttl: {{ cache.redis.ttl }}
    cluster_mode: {{ cache.redis.cluster_mode | bool }}

logging:
  level: {{ logging.level }}
  format: {{ logging.format }}
  outputs:
    file:
      enabled: {{ logging.outputs.file.enabled | bool }}
      path: {{ logging.outputs.file.path }}
      max_size: {{ logging.outputs.file.max_size }}
      max_files: {{ logging.outputs.file.max_files }}
    {% if logging.outputs.syslog.enabled | bool %}
    syslog:
      enabled: {{ logging.outputs.syslog.enabled | bool }}
      facility: {{ logging.outputs.syslog.facility }}
      tag: {{ logging.outputs.syslog.tag }}
    {% endif %}
    {% if logging.outputs.elasticsearch.enabled | bool %}
    elasticsearch:
      enabled: {{ logging.outputs.elasticsearch.enabled | bool }}
      host: {{ logging.outputs.elasticsearch.host }}
      port: {{ logging.outputs.elasticsearch.port }}
      index: {{ logging.outputs.elasticsearch.index }}
    {% endif %}

monitoring:
  metrics:
    enabled: {{ monitoring.metrics.enabled | bool }}
    port: {{ monitoring.metrics.port }}
    path: {{ monitoring.metrics.path }}
    interval: {{ monitoring.metrics.interval }}
  
  health_checks:
    enabled: {{ monitoring.health_checks.enabled | bool }}
    endpoint: {{ monitoring.health_checks.endpoint }}
    timeout: {{ monitoring.health_checks.timeout }}
    interval: {{ monitoring.health_checks.interval }}
  
  {% if monitoring.alerts.enabled | bool %}
  alerts:
    enabled: {{ monitoring.alerts.enabled | bool }}
    channels:
      email: {{ monitoring.alerts.channels.email | to_json }}
      slack: {{ monitoring.alerts.channels.slack }}
      {% if monitoring.alerts.channels.pagerduty %}
      pagerduty: {{ monitoring.alerts.channels.pagerduty }}
      {% endif %}
  {% endif %}

security:
  authentication:
    method: {{ security.authentication.method }}
    session_timeout: {{ security.authentication.session_timeout }}
    max_login_attempts: {{ security.authentication.max_login_attempts }}
    lockout_duration: {{ security.authentication.lockout_duration }}
  
  authorization:
    rbac_enabled: {{ security.authorization.rbac_enabled | bool }}
    default_role: {{ security.authorization.default_role }}
    admin_roles: {{ security.authorization.admin_roles | to_json }}
  
  encryption:
    at_rest: {{ security.encryption.at_rest | bool }}
    in_transit: {{ security.encryption.in_transit | bool }}
    algorithm: {{ security.encryption.algorithm }}

features:
  new_ui: {{ features.new_ui | bool }}
  advanced_analytics: {{ features.advanced_analytics | bool }}
  beta_features: {{ features.beta_features | bool }}
  maintenance_mode: {{ features.maintenance_mode | bool }}
  rate_limiting: {{ features.rate_limiting | bool }}

# Environment-specific configuration
{% if environment == 'development' %}
development:
  hot_reload: true
  debug_toolbar: true
  mock_external_services: true
{% elif environment == 'staging' %}
staging:
  performance_testing: true
  load_testing_enabled: true
  synthetic_monitoring: true
{% elif environment == 'production' %}
production:
  high_availability: true
  disaster_recovery: true
  compliance_logging: true
  audit_trail: true
{% endif %}
EOF
```

Create JSON configuration template:

```jinja2
# Create templates/app-config.json.j2
cat > templates/app-config.json.j2 << 'EOF'
{
  "_comment": "{{ ansible_managed }}",
  "_generated": "{{ ansible_date_time.iso8601 }}",
  "_environment": "{{ environment | default('unknown') }}",
  
  "application": {
    "name": "{{ application.name }}",
    "version": "{{ application.version }}",
    "build": "{{ application.build }}",
    "environment": "{{ application.environment }}",
    "debug_mode": {{ application.debug_mode | bool | to_json }}
  },
  
  "database": {
    "primary": {
      "host": "{{ database.primary.host }}",
      "port": {{ database.primary.port }},
      "name": "{{ database.primary.name }}",
      "username": "{{ database.primary.username }}",
      "password": "{{ database.primary.password }}",
      "pool_size": {{ database.primary.pool_size }},
      "timeout": {{ database.primary.timeout }},
      "ssl_mode": "{{ database.primary.ssl_mode }}"
    }{% if database.replica.enabled | bool %},
    "replica": {
      "enabled": {{ database.replica.enabled | bool | to_json }},
      "host": "{{ database.replica.host }}",
      "port": {{ database.replica.port }},
      "lag_threshold": {{ database.replica.lag_threshold }}
    }{% endif %}
  },
  
  "cache": {
    "redis": {
      "enabled": {{ cache.redis.enabled | bool | to_json }},
      "host": "{{ cache.redis.host }}",
      "port": {{ cache.redis.port }},
      "database": {{ cache.redis.database }},
      {% if cache.redis.password %}"password": "{{ cache.redis.password }}",{% endif %}
      "ttl": {{ cache.redis.ttl }},
      "cluster_mode": {{ cache.redis.cluster_mode | bool | to_json }}
    }
  },
  
  "logging": {
    "level": "{{ logging.level }}",
    "format": "{{ logging.format }}",
    "outputs": {
      "file": {
        "enabled": {{ logging.outputs.file.enabled | bool | to_json }},
        "path": "{{ logging.outputs.file.path }}",
        "max_size": "{{ logging.outputs.file.max_size }}",
        "max_files": {{ logging.outputs.file.max_files }}
      }{% if logging.outputs.syslog.enabled | bool %},
      "syslog": {
        "enabled": {{ logging.outputs.syslog.enabled | bool | to_json }},
        "facility": "{{ logging.outputs.syslog.facility }}",
        "tag": "{{ logging.outputs.syslog.tag }}"
      }{% endif %}{% if logging.outputs.elasticsearch.enabled | bool %},
      "elasticsearch": {
        "enabled": {{ logging.outputs.elasticsearch.enabled | bool | to_json }},
        "host": "{{ logging.outputs.elasticsearch.host }}",
        "port": {{ logging.outputs.elasticsearch.port }},
        "index": "{{ logging.outputs.elasticsearch.index }}"
      }{% endif %}
    }
  },
  
  "monitoring": {
    "metrics": {
      "enabled": {{ monitoring.metrics.enabled | bool | to_json }},
      "port": {{ monitoring.metrics.port }},
      "path": "{{ monitoring.metrics.path }}",
      "interval": {{ monitoring.metrics.interval }}
    },
    "health_checks": {
      "enabled": {{ monitoring.health_checks.enabled | bool | to_json }},
      "endpoint": "{{ monitoring.health_checks.endpoint }}",
      "timeout": {{ monitoring.health_checks.timeout }},
      "interval": {{ monitoring.health_checks.interval }}
    }{% if monitoring.alerts.enabled | bool %},
    "alerts": {
      "enabled": {{ monitoring.alerts.enabled | bool | to_json }},
      "channels": {
        "email": {{ monitoring.alerts.channels.email | to_json }},
        "slack": "{{ monitoring.alerts.channels.slack }}"{% if monitoring.alerts.channels.pagerduty %},
        "pagerduty": "{{ monitoring.alerts.channels.pagerduty }}"{% endif %}
      }
    }{% endif %}
  },
  
  "security": {
    "authentication": {
      "method": "{{ security.authentication.method }}",
      "session_timeout": {{ security.authentication.session_timeout }},
      "max_login_attempts": {{ security.authentication.max_login_attempts }},
      "lockout_duration": {{ security.authentication.lockout_duration }}
    },
    "authorization": {
      "rbac_enabled": {{ security.authorization.rbac_enabled | bool | to_json }},
      "default_role": "{{ security.authorization.default_role }}",
      "admin_roles": {{ security.authorization.admin_roles | to_json }}
    },
    "encryption": {
      "at_rest": {{ security.encryption.at_rest | bool | to_json }},
      "in_transit": {{ security.encryption.in_transit | bool | to_json }},
      "algorithm": "{{ security.encryption.algorithm }}"
    }
  },
  
  "features": {
    "new_ui": {{ features.new_ui | bool | to_json }},
    "advanced_analytics": {{ features.advanced_analytics | bool | to_json }},
    "beta_features": {{ features.beta_features | bool | to_json }},
    "maintenance_mode": {{ features.maintenance_mode | bool | to_json }},
    "rate_limiting": {{ features.rate_limiting | bool | to_json }}
  }
}
EOF
```

Create INI configuration template:

```jinja2
# Create templates/app-config.ini.j2
cat > templates/app-config.ini.j2 << 'EOF'
# {{ ansible_managed }}
# Application Configuration - INI Format
# Generated for {{ environment | default('unknown') }} environment on {{ ansible_date_time.iso8601 }}

[application]
name = {{ application.name }}
version = {{ application.version }}
build = {{ application.build }}
environment = {{ application.environment }}
debug_mode = {{ application.debug_mode }}

[database]
primary_host = {{ database.primary.host }}
primary_port = {{ database.primary.port }}
primary_name = {{ database.primary.name }}
primary_username = {{ database.primary.username }}
primary_password = {{ database.primary.password }}
primary_pool_size = {{ database.primary.pool_size }}
primary_timeout = {{ database.primary.timeout }}
primary_ssl_mode = {{ database.primary.ssl_mode }}

{% if database.replica.enabled | bool %}
replica_enabled = {{ database.replica.enabled }}
replica_host = {{ database.replica.host }}
replica_port = {{ database.replica.port }}
replica_lag_threshold = {{ database.replica.lag_threshold }}
{% endif %}

[cache]
redis_enabled = {{ cache.redis.enabled }}
redis_host = {{ cache.redis.host }}
redis_port = {{ cache.redis.port }}
redis_database = {{ cache.redis.database }}
{% if cache.redis.password %}
redis_password = {{ cache.redis.password }}
{% endif %}
redis_ttl = {{ cache.redis.ttl }}
redis_cluster_mode = {{ cache.redis.cluster_mode }}

[logging]
level = {{ logging.level }}
format = {{ logging.format }}

[logging.file]
enabled = {{ logging.outputs.file.enabled }}
path = {{ logging.outputs.file.path }}
max_size = {{ logging.outputs.file.max_size }}
max_files = {{ logging.outputs.file.max_files }}

{% if logging.outputs.syslog.enabled | bool %}
[logging.syslog]
enabled = {{ logging.outputs.syslog.enabled }}
facility = {{ logging.outputs.syslog.facility }}
tag = {{ logging.outputs.syslog.tag }}
{% endif %}

{% if logging.outputs.elasticsearch.enabled | bool %}
[logging.elasticsearch]
enabled = {{ logging.outputs.elasticsearch.enabled }}
host = {{ logging.outputs.elasticsearch.host }}
port = {{ logging.outputs.elasticsearch.port }}
index = {{ logging.outputs.elasticsearch.index }}
{% endif %}

[monitoring]
metrics_enabled = {{ monitoring.metrics.enabled }}
metrics_port = {{ monitoring.metrics.port }}
metrics_path = {{ monitoring.metrics.path }}
metrics_interval = {{ monitoring.metrics.interval }}

health_checks_enabled = {{ monitoring.health_checks.enabled }}
health_checks_endpoint = {{ monitoring.health_checks.endpoint }}
health_checks_timeout = {{ monitoring.health_checks.timeout }}
health_checks_interval = {{ monitoring.health_checks.interval }}

{% if monitoring.alerts.enabled | bool %}
alerts_enabled = {{ monitoring.alerts.enabled }}
alerts_email = {{ monitoring.alerts.channels.email | join(',') }}
alerts_slack = {{ monitoring.alerts.channels.slack }}
{% if monitoring.alerts.channels.pagerduty %}
alerts_pagerduty = {{ monitoring.alerts.channels.pagerduty }}
{% endif %}
{% endif %}

[security]
auth_method = {{ security.authentication.method }}
auth_session_timeout = {{ security.authentication.session_timeout }}
auth_max_login_attempts = {{ security.authentication.max_login_attempts }}
auth_lockout_duration = {{ security.authentication.lockout_duration }}

authz_rbac_enabled = {{ security.authorization.rbac_enabled }}
authz_default_role = {{ security.authorization.default_role }}
authz_admin_roles = {{ security.authorization.admin_roles | join(',') }}

encryption_at_rest = {{ security.encryption.at_rest }}
encryption_in_transit = {{ security.encryption.in_transit }}
encryption_algorithm = {{ security.encryption.algorithm }}

[features]
new_ui = {{ features.new_ui }}
advanced_analytics = {{ features.advanced_analytics }}
beta_features = {{ features.beta_features }}
maintenance_mode = {{ features.maintenance_mode }}
rate_limiting = {{ features.rate_limiting }}

{% if environment == 'development' %}
[development]
hot_reload = true
debug_toolbar = true
mock_external_services = true
{% elif environment == 'staging' %}
[staging]
performance_testing = true
load_testing_enabled = true
synthetic_monitoring = true
{% elif environment == 'production' %}
[production]
high_availability = true
disaster_recovery = true
compliance_logging = true
audit_trail = true
{% endif %}
EOF
```

Create Nginx configuration template:

```jinja2
# Create templates/nginx.conf.j2
cat > templates/nginx.conf.j2 << 'EOF'
# {{ ansible_managed }}
# Nginx Configuration
# Generated for {{ environment | default('unknown') }} environment

user www-data;
worker_processes {{ web_server.nginx.worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ web_server.nginx.worker_connections }};
    use epoll;
    multi_accept on;
}

http {
    # Basic Settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout {{ web_server.nginx.keepalive_timeout }};
    types_hash_max_size 2048;
    client_max_body_size {{ web_server.nginx.client_max_body_size }};
    
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    {% if web_server.nginx.ssl.enabled | bool %}
    # SSL Settings
    ssl_protocols {{ web_server.nginx.ssl.protocols | join(' ') }};
    ssl_prefer_server_ciphers on;
    ssl_ciphers {{ web_server.nginx.ssl.ciphers }};
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    {% endif %}

    {% if web_server.nginx.gzip.enabled | bool %}
    # Gzip Settings
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level {{ web_server.nginx.gzip.level }};
    gzip_types {{ web_server.nginx.gzip.types | join(' ') }};
    {% endif %}

    # Logging Settings
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log {{ logging.outputs.file.path | replace('app.log', 'nginx_access.log') }} main;
    error_log {{ logging.outputs.file.path | replace('app.log', 'nginx_error.log') }};

    # Security Headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    {% if web_server.nginx.ssl.enabled | bool %}
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    {% endif %}

    # Rate Limiting (if enabled)
    {% if features.rate_limiting | bool %}
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
    {% endif %}

    # Upstream for application servers
    upstream app_backend {
        {% if environment == 'production' %}
        server 127.0.0.1:{{ application.port | default(8080) }} max_fails=3 fail_timeout=30s;
        server 127.0.0.1:{{ (application.port | default(8080)) + 1 }} max_fails=3 fail_timeout=30s backup;
        {% else %}
        server 127.0.0.1:{{ application.port | default(8080) }};
        {% endif %}
        keepalive 32;
    }

    # Main server block
    server {
        listen 80{% if environment == 'production' %} default_server{% endif %};
        {% if web_server.nginx.ssl.enabled | bool %}
        listen 443 ssl{% if environment == 'production' %} default_server{% endif %};
        ssl_certificate {{ web_server.nginx.ssl.certificate }};
        ssl_certificate_key {{ web_server.nginx.ssl.private_key }};
        {% endif %}

        server_name {{ application.name | lower | replace(' ', '-') }}.{{ 'company.com' if environment == 'production' else environment + '.company.com' }};
        root /var/www/{{ application.name | lower | replace(' ', '-') }};
        index index.html index.htm;

        {% if web_server.nginx.ssl.enabled | bool %}
        # Redirect HTTP to HTTPS
        if ($scheme != "https") {
            return 301 https://$host$request_uri;
        }
        {% endif %}

        # Health check endpoint
        location {{ monitoring.health_checks.endpoint }} {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # Metrics endpoint
        location {{ monitoring.metrics.path }} {
            {% if environment != 'development' %}
            allow 127.0.0.1;
            allow 10.0.0.0/8;
            allow 172.16.0.0/12;
            allow 192.168.0.0/16;
            deny all;
            {% endif %}
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # API endpoints
        location /api/ {
            {% if features.rate_limiting | bool %}
            limit_req zone=api burst=20 nodelay;
            {% endif %}
            
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 30s;
        }

        # Login endpoint with stricter rate limiting
        location /api/auth/login {
            {% if features.rate_limiting | bool %}
            limit_req zone=login burst=5 nodelay;
            {% endif %}
            
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Static files
        location /static/ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            try_files $uri $uri/ =404;
        }

        # Main application
        location / {
            try_files $uri $uri/ @app;
        }

        location @app {
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        {% if environment == 'development' %}
        # Development-specific locations
        location /debug/ {
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
        }
        {% endif %}

        # Error pages
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }

    {% if environment == 'production' %}
    # Additional server block for www redirect
    server {
        listen 80;
        listen 443 ssl;
        server_name www.{{ application.name | lower | replace(' ', '-') }}.company.com;
        
        {% if web_server.nginx.ssl.enabled | bool %}
        ssl_certificate {{ web_server.nginx.ssl.certificate }};
        ssl_certificate_key {{ web_server.nginx.ssl.private_key }};
        {% endif %}
        
        return 301 $scheme://{{ application.name | lower | replace(' ', '-') }}.company.com$request_uri;
    }
    {% endif %}
}
EOF
```

Create test playbook for multi-format generation:

```yaml
# Create test-multi-format.yml
cat > test-multi-format.yml << 'EOF'
---
- name: Test multi-format configuration generation
  hosts: all
  gather_facts: yes
  vars_files:
    - config-data.yml
  
  tasks:
    - name: Generate YAML configuration
      template:
        src: templates/app-config.yml.j2
        dest: "/tmp/app-config-{{ environment }}.yml"
        mode: '0644'
    
    - name: Generate JSON configuration
      template:
        src: templates/app-config.json.j2
        dest: "/tmp/app-config-{{ environment }}.json"
        mode: '0644'
    
    - name: Generate INI configuration
      template:
        src: templates/app-config-{{ environment }}.ini"
        dest: "/tmp/app-config-{{ environment }}.ini"
        mode: '0644'
    
    - name: Generate Nginx configuration
      template:
        src: templates/nginx.conf.j2
        dest: "/tmp/nginx-{{ environment }}.conf"
        mode: '0644'
    
    - name: Display generated configurations
      debug:
        msg: |
          Generated configurations for {{ environment }} environment:
          - YAML: /tmp/app-config-{{ environment }}.yml
          - JSON: /tmp/app-config-{{ environment }}.json
          - INI: /tmp/app-config-{{ environment }}.ini
          - Nginx: /tmp/nginx-{{ environment }}.conf
    
    - name: Validate JSON configuration
      shell: python3 -m json.tool "/tmp/app-config-{{ environment }}.json"
      register: json_validation
      changed_when: false
      failed_when: json_validation.rc != 0
    
    - name: Test Nginx configuration syntax
      shell: nginx -t -c "/tmp/nginx-{{ environment }}.conf"
      register: nginx_validation
      changed_when: false
      failed_when: false  # Don't fail if nginx is not installed
    
    - name: Show validation results
      debug:
        msg: |
          Configuration Validation Results:
          - JSON: {{ 'Valid' if json_validation.rc == 0 else 'Invalid' }}
          - Nginx: {{ 'Valid' if nginx_validation.rc == 0 else 'Not tested (nginx not available)' }}
EOF

# Run the multi-format test
ansible-playbook -i inventory.ini test-multi-format.yml
```

## Exercise 2: Environment-Specific Configuration Management (10 minutes)

### Task: Implement sophisticated environment-specific configuration patterns

Create environment-specific templates:

```bash
# Create environment-specific template directory
mkdir -p templates/environments

# Development environment template
cat > templates/environments/development.yml.j2 << 'EOF'
# {{ ansible_managed }}
# Development Environment Configuration

# Development-specific overrides
development:
  # Database settings
  database:
    show_sql: true
    connection_pool:
      min_size: 1
      max_size: 5
      idle_timeout: 300
    migrations:
      auto_apply: true
      
  # Debugging and development tools
  debug:
    enabled: true
    profiler: true
    query_analyzer: true
    memory_profiler: true
    
  # Hot reload and development features
  hot_reload:
    enabled: true
    watch_directories:
      - "/app/src"
      - "/app/templates"
      - "/app/static"
    excluded_patterns:
      - "*.pyc"
      - "__pycache__"
      - ".git"
      
  # Mock services for development
  mock_services:
    payment_gateway: true
    email_service: true
    sms_service: true
    external_api: true
    
  # Development-specific logging
  logging:
    console_output: true
    sql_queries: true
    request_response: true
    
  # Testing configuration
  testing:
    auto_test_runner: true
    coverage_reporting: true
    test_database: "{{ database.primary.name }}_test"
    
  # Development server settings
  server:
    auto_restart: true
    debug_mode: true
    template_debug: true
    static_file_serving: true
    
# Development tools configuration
tools:
  code_formatter: "black"
  linter: "flake8"
  type_checker: "mypy"
  documentation_generator: "sphinx"
  
# Development dependencies
dependencies:
  development_packages:
    - "pytest"
    - "pytest-cov"
    - "black"
    - "flake8"
    - "mypy"
    - "ipdb"
    - "django-debug-toolbar"
EOF

# Staging environment template
cat > templates/environments/staging.yml.j2 << 'EOF'
# {{ ansible_managed }}
# Staging Environment Configuration

# Staging-specific overrides
staging:
  # Performance testing
  performance:
    load_testing: true
    stress_testing: true
    benchmark_recording: true
    
  # Monitoring and observability
  monitoring:
    detailed_metrics: true
    performance_profiling: true
    error_tracking: true
    user_behavior_analytics: true
    
  # Testing features
  testing:
    integration_tests: true
    end_to_end_tests: true
    api_testing: true
    security_scanning: true
    
  # Staging-specific database settings
  database:
    query_logging: true
    slow_query_threshold: 1000  # ms
    connection_pool:
      min_size: 5
      max_size: 20
      
  # Feature flags for staging
  feature_flags:
    new_features: true
    experimental_ui: true
    beta_apis: true
    
  # Staging deployment settings
  deployment:
    blue_green: true
    canary_releases: true
    rollback_enabled: true
    
  # Data management
  data:
    anonymization: true
    synthetic_data: true
    data_refresh_schedule: "daily"
    
# Staging-specific integrations
integrations:
  ci_cd:
    automated_deployment: true
    test_automation: true
    quality_gates: true
    
  external_services:
    staging_payment_gateway: true
    test_email_service: true
    staging_apis: true
    
# Staging security settings
security:
  penetration_testing: true
  vulnerability_scanning: true
  security_headers_testing: true
EOF

# Production environment template
cat > templates/environments/production.yml.j2 << 'EOF'
# {{ ansible_managed }}
# Production Environment Configuration

# Production-specific overrides
production:
  # High availability settings
  high_availability:
    enabled: true
    failover_timeout: 30
    health_check_interval: 10
    circuit_breaker: true
    
  # Performance optimization
  performance:
    connection_pooling: true
    query_optimization: true
    caching_strategy: "aggressive"
    cdn_enabled: true
    
  # Production database settings
  database:
    read_replicas: 2
    backup_frequency: "hourly"
    point_in_time_recovery: true
    encryption_at_rest: true
    connection_pool:
      min_size: 10
      max_size: 100
      
  # Security hardening
  security:
    ssl_only: true
    hsts_enabled: true
    csrf_protection: true
    rate_limiting: true
    ip_whitelisting: false
    
  # Monitoring and alerting
  monitoring:
    real_time_alerts: true
    sla_monitoring: true
    business_metrics: true
    compliance_logging: true
    
  # Disaster recovery
  disaster_recovery:
    enabled: true
    backup_regions: ["us-west-2", "eu-west-1"]
    rto: 300  # seconds
    rpo: 60   # seconds
    
  # Compliance and auditing
  compliance:
    audit_logging: true
    data_retention: "7_years"
    gdpr_compliance: true
    sox_compliance: true
    
# Production integrations
integrations:
  payment_gateway:
    primary: "stripe"
    fallback: "paypal"
    
  email_service:
    provider: "sendgrid"
    backup_provider: "ses"
    
  monitoring_services:
    - "datadog"
    - "newrelic"
    - "pagerduty"
    
  cdn:
    provider: "cloudflare"
    cache_ttl: 3600
    
# Production scaling settings
scaling:
  auto_scaling: true
  min_instances: 3
  max_instances: 50
  scale_up_threshold: 70
  scale_down_threshold: 30
  
# Production maintenance
maintenance:
  automated_patching: false
  maintenance_window: "Sunday 02:00-04:00 UTC"
  rolling_updates: true
  zero_downtime_deployment: true
EOF
```

Create environment-aware configuration generator:

```jinja2
# Create templates/environment-aware-config.j2
cat > templates/environment-aware-config.j2 << 'EOF'
# {{ ansible_managed }}
# Environment-Aware Configuration
# Target Environment: {{ environment | upper }}
# Generated: {{ ansible_date_time.iso8601 }}

# Base configuration
{% include 'app-config.yml.j2' %}

---
# Environment-specific configuration
{% if environment == 'development' %}
{% include 'environments/development.yml.j2' %}
{% elif environment == 'staging' %}
{% include 'environments/staging.yml.j2' %}
{% elif environment == 'production' %}
{% include 'environments/production.yml.j2' %}
{% else %}
# Unknown environment: {{ environment }}
# Using minimal configuration
minimal_config:
  environment: {{ environment }}
  timestamp: {{ ansible_date_time.iso8601 }}
  warning: "Unknown environment - using default settings"
{% endif %}

---
# Runtime configuration
runtime:
  hostname: {{ ansible_hostname }}
  fqdn: {{ ansible_fqdn | default(ansible_hostname) }}
  ip_address: {{ ansible_default_ipv4.address | default('unknown') }}
  os_family: {{ ansible_os_family }}
  os_distribution: {{ ansible_distribution }}
  os_version: {{ ansible_distribution_version }}
  architecture: {{ ansible_architecture }}
  
  # System resources
  cpu_cores: {{ ansible_processor_vcpus }}
  memory_mb: {{ ansible_memtotal_mb }}
  
  # Environment detection
  detected_environment: {{ environment }}
  is_container: {{ ansible_virtualization_type == 'docker' }}
  is_cloud: {{ ansible_system_vendor in ['Amazon EC2', 'Microsoft Corporation', 'Google'] }}
  
  # Configuration metadata
  config_version: "{{ application.version }}"
  config_build: "{{ application.build }}"
  config_generated: "{{ ansible_date_time.iso8601 }}"
  config_by: "{{ ansible_user_id }}@{{ ansible_hostname }}"

---
# Validation and health checks
validation:
  required_services:
    {% if environment == 'production' %}
    - database_primary
    - database_replica
    - cache_cluster
    - load_balancer
    - monitoring
    {% elif environment == 'staging' %}
    - database_primary
    - cache_single
    - monitoring
    {% else %}
    - database_local
    {% endif %}
  
  health_checks:
    database: "SELECT 1"
    cache: "PING"
    {% if environment in ['staging', 'production'] %}
    external_api: "GET /health"
    {% endif %}
  
  performance_thresholds:
    response_time_ms: {{ 100 if environment == 'development' else 500 if environment == 'staging' else 200 }}
    cpu_usage_percent: {{ 90 if environment == 'development' else 80 if environment == 'staging' else 70 }}
    memory_usage_percent: {{ 90 if environment == 'development' else 85 if environment == 'staging' else 80 }}
    
  alerts:
    {% if environment == 'production' %}
    critical: ["pagerduty", "email", "slack"]
    warning: ["email", "slack"]
    info: ["slack"]
    {% elif environment == 'staging' %}
    critical: ["email", "slack"]
    warning: ["slack"]
    {% else %}
    critical: ["console"]
    {% endif %}
EOF
```

Create configuration validation template:

```jinja2
# Create templates/config-validation.j2
cat > templates/config-validation.j2 << 'EOF'
#!/bin/bash
# {{ ansible_managed }}
# Configuration Validation Script
# Environment: {{ environment | upper }}

set -e

echo "=== Configuration Validation for {{ environment | upper }} ==="
echo "Generated: {{ ansible_date_time.iso8601 }}"
echo

# Validation functions
validate_database() {
    echo "Validating database configuration..."
    
    # Check database connectivity
    {% if database.primary.host != 'localhost' %}
    if ! nc -z {{ database.primary.host }} {{ database.primary.port }}; then
        echo "❌ Cannot connect to database: {{ database.primary.host }}:{{ database.primary.port }}"
        return 1
    fi
    {% endif %}
    
    echo "✅ Database configuration valid"
}

validate_cache() {
    echo "Validating cache configuration..."
    
    {% if cache.redis.enabled | bool %}
    # Check Redis connectivity
    {% if cache.redis.host != 'localhost' %}
    if ! nc -z {{ cache.redis.host }} {{ cache.redis.port }}; then
        echo "❌ Cannot connect to Redis: {{ cache.redis.host }}:{{ cache.redis.port }}"
        return 1
    fi
    {% endif %}
    {% endif %}
    
    echo "✅ Cache configuration valid"
}

validate_ssl() {
    echo "Validating SSL configuration..."
    
    {% if web_server.nginx.ssl.enabled | bool %}
    # Check SSL certificate
    if [ ! -f "{{ web_server.nginx.ssl.certificate }}" ]; then
        echo "❌ SSL certificate not found: {{ web_server.nginx.ssl.certificate }}"
        return 1
    fi
    
    if [ ! -f "{{ web_server.nginx.ssl.private_key }}" ]; then
        echo "❌ SSL private key not found: {{ web_server.nginx.ssl.private_key }}"
        return 1
    fi
    
    # Check certificate validity
    if ! openssl x509 -in "{{ web_server.nginx.ssl.certificate }}" -noout -checkend 86400; then
        echo "⚠️  SSL certificate expires within 24 hours"
    fi
    {% endif %}
    
    echo "✅ SSL configuration valid"
}

validate_permissions() {
    echo "Validating file permissions..."
    
    # Check log directory permissions
    LOG_DIR=$(dirname "{{ logging.outputs.file.path }}")
    if [ ! -w "$LOG_DIR" ]; then
        echo "❌ Log directory not writable: $LOG_DIR"
        return 1
    fi
    
    echo "✅ File permissions valid"
}

validate_environment_specific() {
    echo "Validating environment-specific configuration..."
    
    {% if environment == 'production' %}
    # Production-specific validations
    if [ "{{ application.debug_mode }}" = "true" ]; then
        echo "❌ Debug mode should be disabled in production"
        return 1
    fi
    
    if [ "{{ security.encryption.at_rest }}" != "True" ]; then
        echo "❌ Encryption at rest must be enabled in production"
        return 1
    fi
    
    {% elif environment == 'staging' %}
    # Staging-specific validations
    if [ "{{ monitoring.alerts.enabled }}" = "True" ]; then
        echo "⚠️  Alerts are enabled in staging - this may cause noise"
    fi
    
    {% elif environment == 'development' %}
    # Development-specific validations
    if [ "{{ cache.redis.cluster_mode }}" = "True" ]; then
        echo "⚠️  Redis cluster mode enabled in development - may be unnecessary"
    fi
    {% endif %}
    
    echo "✅ Environment-specific configuration valid"
}

# Run validations
echo "Starting configuration validation..."
echo

validate_database
validate_cache
validate_ssl
validate_permissions
validate_environment_specific

echo
echo "=== Validation Summary ==="
echo "Environment: {{ environment | upper }}"
echo "Application: {{ application.name }} v{{ application.version }}"
echo "Configuration: Valid ✅"
echo "Timestamp: $(date -Iseconds)"
echo

# Configuration summary
cat << 'EOF_SUMMARY'
Configuration Summary:
- Database: {{ database.primary.host }}:{{ database.primary.port }}
- Cache: {{ cache.redis.host }}:{{ cache.redis.port }} ({{ 'enabled' if cache.redis.enabled else 'disabled' }})
- SSL: {{ 'enabled' if web_server.nginx.ssl.enabled else 'disabled' }}
- Monitoring: {{ 'enabled' if monitoring.metrics.enabled else 'disabled' }}
- Alerts: {{ 'enabled' if monitoring.alerts.enabled else 'disabled' }}
- Debug Mode: {{ application.debug_mode }}
- Environment: {{ environment | upper }}
EOF_SUMMARY

echo "Configuration validation completed successfully!"
EOF
```

Create environment-specific test:

```yaml
# Create test-environment-config.yml
cat > test-environment-config.yml << 'EOF'
---
- name: Test environment-specific configuration generation
  hosts: all
  gather_facts: yes
  vars_files:
    - config-data.yml
  
  tasks:
    - name: Generate environment-aware configuration
      template:
        src: templates/environment-aware-config.j2
        dest: "/tmp/environment-aware-{{ environment }}.yml"
        mode: '0644'
    
    - name: Generate configuration validation script
      template:
        src: templates/config-validation.j2
        dest: "/tmp/validate-config-{{ environment }}.sh"
        mode: '0755'
    
    - name: Show environment-specific settings
      debug:
        msg: |
          Environment-Specific Configuration for {{ environment | upper }}:
          
          Database Settings:
          - Host: {{ database.primary.host }}
          - Pool Size: {{ database.primary.pool_size }}
          - SSL Mode: {{ database.primary.ssl_mode }}
          
          Security Settings:
          - SSL Enabled: {{ web_server.nginx.ssl.enabled }}
          - Encryption at Rest: {{ security.encryption.at_rest }}
          - Authentication Method: {{ security.authentication.method }}
          
          Feature Flags:
          - Debug Mode: {{ application.debug_mode }}
          - New UI: {{ features.new_ui }}
          - Rate Limiting: {{ features.rate_limiting }}
          
          Monitoring:
          - Metrics Enabled: {{ monitoring.metrics.enabled }}
          - Alerts Enabled: {{ monitoring.alerts.enabled }}
          - Health Checks: {{ monitoring.health_checks.enabled }}
    
    - name: Validate generated configuration
      shell: "/tmp/validate-config-{{ environment }}.sh"
      register: validation_result
      changed_when: false
      failed_when: false
    
    - name: Show validation results
      debug:
        msg: |
          Configuration Validation Results:
          {{ validation_result.stdout }}
          
          {% if validation_result.stderr %}
          Errors/Warnings:
          {{ validation_result.stderr }}
          {% endif %}
EOF

# Run environment-specific tests
ansible-playbook -i inventory.ini test-environment-config.yml
```

## Exercise 3: Configuration Templating Best Practices (8 minutes)

### Task: Implement enterprise configuration templating patterns and best practices

Create configuration management framework:

```bash
# Create configuration management directory structure
mkdir -p templates/{base,fragments,validators}
mkdir -p configs/{schemas,defaults,overrides}
```

Create base configuration template:

```jinja2
# Create templates/base/base-config.j2
cat > templates/base/base-config.j2 << 'EOF'
{# Base Configuration Template #}
{# This template provides the foundation for all environment-specific configurations #}

{# Configuration header with metadata #}
# {{ ansible_managed }}
# Configuration: {{ config_name | default('application') }}
# Environment: {{ environment | upper }}
# Version: {{ config_version | default('1.0.0') }}
# Generated: {{ ansible_date_time.iso8601 }}
# Host: {{ ansible_hostname }}
# User: {{ ansible_user_id }}

{# Include configuration fragments based on enabled features #}
{% if include_application_config | default(true) %}
{% include 'fragments/application.j2' %}
{% endif %}

{% if include_database_config | default(true) %}
{% include 'fragments/database.j2' %}
{% endif %}

{% if include_cache_config | default(true) %}
{% include 'fragments/cache.j2' %}
{% endif %}

{% if include_logging_config | default(true) %}
{% include 'fragments/logging.j2' %}
{% endif %}

{% if include_monitoring_config | default(true) %}
{% include 'fragments/monitoring.j2' %}
{% endif %}

{% if include_security_config | default(true) %}
{% include 'fragments/security.j2' %}
{% endif %}

{# Environment-specific includes #}
{% if environment == 'development' %}
{% include 'fragments/development.j2' %}
{% elif environment == 'staging' %}
{% include 'fragments/staging.j2' %}
{% elif environment == 'production' %}
{% include 'fragments/production.j2' %}
{% endif %}

{# Custom configuration fragments #}
{% for fragment in custom_fragments | default([]) %}
{% include 'fragments/' + fragment + '.j2' %}
{% endfor %}

{# Configuration footer with validation info #}
# Configuration validation
validation:
  schema_version: "{{ config_schema_version | default('1.0') }}"
  required_sections: {{ required_config_sections | default(['application', 'database']) | to_json }}
  optional_sections: {{ optional_config_sections | default(['cache', 'monitoring']) | to_json }}
  generated_by: "Ansible {{ ansible_version.full }}"
  template_checksum: "{{ template_checksum | default('unknown') }}"
EOF
```

Create configuration fragments:

```jinja2
# Create templates/fragments/application.j2
cat > templates/fragments/application.j2 << 'EOF'
{# Application Configuration Fragment #}

# Application Configuration
application:
  # Basic application information
  name: {{ application.name | to_json }}
  version: {{ application.version | to_json }}
  build: {{ application.build | to_json }}
  environment: {{ application.environment | to_json }}
  
  # Runtime configuration
  debug_mode: {{ application.debug_mode | bool }}
  {% if application.port is defined %}
  port: {{ application.port }}
  {% endif %}
  {% if application.bind_address is defined %}
  bind_address: {{ application.bind_address | to_json }}
  {% endif %}
  
  # Application-specific settings
  {% if application.timezone is defined %}
  timezone: {{ application.timezone | to_json }}
  {% endif %}
  {% if application.locale is defined %}
  locale: {{ application.locale | to_json }}
  {% endif %}
  
  # Feature flags
  features:
    {% for feature, enabled in features.items() %}
    {{ feature }}: {{ enabled | bool }}
    {% endfor %}
EOF

# Create templates/fragments/database.j2
cat > templates/fragments/database.j2 << 'EOF'
{# Database Configuration Fragment #}

# Database Configuration
database:
  # Primary database
  primary:
    host: {{ database.primary.host | to_json }}
    port: {{ database.primary.port }}
    name: {{ database.primary.name | to_json }}
    username: {{ database.primary.username | to_json }}
    password: {{ database.primary.password | to_json }}
    
    # Connection pool settings
    pool:
      size: {{ database.primary.pool_size }}
      timeout: {{ database.primary.timeout }}
      max_overflow: {{ database.primary.max_overflow | default(10) }}
      
    # SSL configuration
    ssl:
      mode: {{ database.primary.ssl_mode | to_json }}
      {% if database.primary.ssl_cert is defined %}
      certificate: {{ database.primary.ssl_cert | to_json }}
      {% endif %}
      {% if database.primary.ssl_key is defined %}
      key: {{ database.primary.ssl_key | to_json }}
      {% endif %}
  
  {% if database.replica.enabled | default(false) %}
  # Replica database
  replica:
    enabled: {{ database.replica.enabled | bool }}
    host: {{ database.replica.host | to_json }}
    port: {{ database.replica.port }}
    lag_threshold: {{ database.replica.lag_threshold }}
    read_only: true
  {% endif %}
  
  # Database maintenance
  maintenance:
    backup_enabled: {{ database.backup_enabled | default(true) | bool }}
    {% if database.backup_schedule is defined %}
    backup_schedule: {{ database.backup_schedule | to_json }}
    {% endif %}
    vacuum_enabled: {{ database.vacuum_enabled | default(true) | bool }}
    analyze_enabled: {{ database.analyze_enabled | default(true) | bool }}
EOF

# Create templates/fragments/security.j2
cat > templates/fragments/security.j2 << 'EOF'
{# Security Configuration Fragment #}

# Security Configuration
security:
  # Authentication settings
  authentication:
    method: {{ security.authentication.method | to_json }}
    session_timeout: {{ security.authentication.session_timeout }}
    max_login_attempts: {{ security.authentication.max_login_attempts }}
    lockout_duration: {{ security.authentication.lockout_duration }}
    
    {% if security.authentication.method == 'oauth2' %}
    oauth2:
      client_id: {{ security.oauth2.client_id | to_json }}
      client_secret: {{ security.oauth2.client_secret | to_json }}
      authorization_url: {{ security.oauth2.authorization_url | to_json }}
      token_url: {{ security.oauth2.token_url | to_json }}
      scope: {{ security.oauth2.scope | default(['openid', 'profile']) | to_json }}
    {% endif %}
  
  # Authorization settings
  authorization:
    rbac_enabled: {{ security.authorization.rbac_enabled | bool }}
    default_role: {{ security.authorization.default_role | to_json }}
    admin_roles: {{ security.authorization.admin_roles | to_json }}
    
    # Permission matrix
    permissions:
      {% for role, perms in security.permissions | default({}).items() %}
      {{ role }}: {{ perms | to_json }}
      {% endfor %}
  
  # Encryption settings
  encryption:
    at_rest: {{ security.encryption.at_rest | bool }}
    in_transit: {{ security.encryption.in_transit | bool }}
    algorithm: {{ security.encryption.algorithm | to_json }}
    {% if security.encryption.key_rotation_days is defined %}
    key_rotation_days: {{ security.encryption.key_rotation_days }}
    {% endif %}
  
  # Security headers and policies
  headers:
    {% if security.headers is defined %}
    {% for header, value in security.headers.items() %}
    {{ header }}: {{ value | to_json }}
    {% endfor %}
    {% else %}
    # Default security headers
    x_frame_options: "DENY"
    x_content_type_options: "nosniff"
    x_xss_protection: "1; mode=block"
    {% if web_server.nginx.ssl.enabled | default(false) %}
    strict_transport_security: "max-age=31536000; includeSubDomains"
    {% endif %}
    {% endif %}
EOF
```

Create configuration validation schema:

```yaml
# Create configs/schemas/config-schema.yml
cat > configs/schemas/config-schema.yml << 'EOF'
---
# Configuration Schema Definition
schema_version: "1.0"
description: "Enterprise application configuration schema"

# Required sections
required_sections:
  - application
  - database
  - logging

# Optional sections
optional_sections:
  - cache
  - monitoring
  - security

# Field definitions
fields:
  application:
    name:
      type: string
      required: true
      description: "Application name"
    version:
      type: string
      required: true
      pattern: '^\d+\.\d+\.\d+$'
      description: "Semantic version number"
    environment:
      type: string
      required: true
      enum: ["development", "staging", "production"]
      description: "Deployment environment"
    debug_mode:
      type: boolean
      required: false
      default: false
      description: "Enable debug mode"
    port:
      type: integer
      required: false
      minimum: 1
      maximum: 65535
      description: "Application port number"
  
  database:
    primary:
      type: object
      required: true
      properties:
        host:
          type: string
          required: true
          description: "Database host"
        port:
          type: integer
          required: true
          minimum: 1
          maximum: 65535
          description: "Database port"
        name:
          type: string
          required: true
          description: "Database name"
        username:
          type: string
          required: true
          description: "Database username"
        password:
          type: string
          required: true
          description: "Database password"
        pool_size:
          type: integer
          required: false
          minimum: 1
          maximum: 1000
          default: 10
          description: "Connection pool size"
  
  logging:
    level:
      type: string
      required: true
      enum: ["DEBUG", "INFO", "WARN", "ERROR"]
      description: "Logging level"
    format:
      type: string
      required: false
      enum: ["json", "text"]
      default: "json"
      description: "Log format"

# Environment-specific constraints
environment_constraints:
  production:
    application.debug_mode: false
    security.encryption.at_rest: true
    security.encryption.in_transit: true
    logging.level: ["WARN", "ERROR"]
  
  staging:
    monitoring.alerts.enabled: false
    
  development:
    database.primary.ssl_mode: ["disable", "prefer"]
    security.encryption.at_rest: [true, false]

# Validation rules
validation_rules:
  - name: "ssl_in_production"
    condition: "environment == 'production'"
    requirement: "web_server.nginx.ssl.enabled == true"
    message: "SSL must be enabled in production"
  
  - name: "no_debug_in_production"
    condition: "environment == 'production'"
    requirement: "application.debug_mode == false"
    message: "Debug mode must be disabled in production"
  
  - name: "encryption_in_production"
    condition: "environment == 'production'"
    requirement: "security.encryption.at_rest == true and security.encryption.in_transit == true"
    message: "Encryption must be enabled in production"
EOF
```

Create configuration validator:

```python
# Create templates/validators/config_validator.py
cat > templates/validators/config_validator.py << 'EOF'
#!/usr/bin/env python3
"""
Configuration Validator
Validates generated configurations against schema
"""

import yaml
import json
import sys
import re
from pathlib import Path

class ConfigValidator:
    def __init__(self, schema_file):
        with open(schema_file, 'r') as f:
            self.schema = yaml.safe_load(f)
    
    def validate_config(self, config_file):
        """Validate configuration file against schema"""
        try:
            with open(config_file, 'r') as f:
                if config_file.endswith('.json'):
                    config = json.load(f)
                else:
                    config = yaml.safe_load(f)
            
            errors = []
            warnings = []
            
            # Validate required sections
            for section in self.schema.get('required_sections', []):
                if section not in config:
                    errors.append(f"Missing required section: {section}")
            
            # Validate field types and constraints
            errors.extend(self._validate_fields(config))
            
            # Validate environment-specific constraints
            environment = config.get('application', {}).get('environment')
            if environment:
                errors.extend(self._validate_environment_constraints(config, environment))
            
            # Validate custom rules
            errors.extend(self._validate_custom_rules(config))
            
            return {
                'valid': len(errors) == 0,
                'errors': errors,
                'warnings': warnings
            }
            
        except Exception as e:
            return {
                'valid': False,
                'errors': [f"Failed to parse configuration: {str(e)}"],
                'warnings': []
            }
    
    def _validate_fields(self, config):
        """Validate individual fields"""
        errors = []
        
        for section_name, section_schema in self.schema.get('fields', {}).items():
            if section_name in config:
                section_data = config[section_name]
                errors.extend(self._validate_section(section_data, section_schema, section_name))
        
        return errors
    
    def _validate_section(self, data, schema, path):
        """Validate a configuration section"""
        errors = []
        
        for field_name, field_schema in schema.items():
            field_path = f"{path}.{field_name}"
            
            if field_schema.get('required', False) and field_name not in data:
                errors.append(f"Missing required field: {field_path}")
                continue
            
            if field_name in data:
                field_value = data[field_name]
                errors.extend(self._validate_field(field_value, field_schema, field_path))
        
        return errors
    
    def _validate_field(self, value, schema, path):
        """Validate individual field"""
        errors = []
        
        # Type validation
        expected_type = schema.get('type')
        if expected_type:
            if expected_type == 'string' and not isinstance(value, str):
                errors.append(f"Field {path} should be string, got {type(value).__name__}")
            elif expected_type == 'integer' and not isinstance(value, int):
                errors.append(f"Field {path} should be integer, got {type(value).__name__}")
            elif expected_type == 'boolean' and not isinstance(value, bool):
                errors.append(f"Field {path} should be boolean, got {type(value).__name__}")
        
        # Range validation
        if isinstance(value, (int, float)):
            if 'minimum' in schema and value < schema['minimum']:
                errors.append(f"Field {path} value {value} is below minimum {schema['minimum']}")
            if 'maximum' in schema and value > schema['maximum']:
                errors.append(f"Field {path} value {value} is above maximum {schema['maximum']}")
        
        # Enum validation
        if 'enum' in schema and value not in schema['enum']:
            errors.append(f"Field {path} value '{value}' not in allowed values: {schema['enum']}")
        
        # Pattern validation
        if 'pattern' in schema and isinstance(value, str):
            if not re.match(schema['pattern'], value):
                errors.append(f"Field {path} value '{value}' does not match pattern: {schema['pattern']}")
        
        return errors
    
    def _validate_environment_constraints(self, config, environment):
        """Validate environment-specific constraints"""
        errors = []
        
        constraints = self.schema.get('environment_constraints', {}).get(environment, {})
        
        for field_path, allowed_values in constraints.items():
            current_value = self._get_nested_value(config, field_path)
            
            if current_value is not None:
                if isinstance(allowed_values, list):
                    if current_value not in allowed_values:
                        errors.append(f"Field {field_path} value '{current_value}' not allowed in {environment} environment. Allowed: {allowed_values}")
                else:
                    if current_value != allowed_values:
                        errors.append(f"Field {field_path} must be '{allowed_values}' in {environment} environment, got '{current_value}'")
        
        return errors
    
    def _validate_custom_rules(self, config):
        """Validate custom business rules"""
        errors = []
        
        for rule in self.schema.get('validation_rules', []):
            condition = rule.get('condition')
            requirement = rule.get('requirement')
            message = rule.get('message', f"Validation rule '{rule.get('name')}' failed")
            
            if self._evaluate_condition(config, condition):
                if not self._evaluate_condition(config, requirement):
                    errors.append(message)
        
        return errors
    
    def _get_nested_value(self, data, path):
        """Get nested value from configuration"""
        keys = path.split('.')
        current = data
        
        for key in keys:
            if isinstance(current, dict) and key in current:
                current = current[key]
            else:
                return None
        
        return current
    
    def _evaluate_condition(self, config, condition):
        """Evaluate a condition string"""
        # Simple condition evaluation - in production, use a proper expression evaluator
        try:
            # Replace field paths with actual values
            import re
            
            def replace_field(match):
                field_path = match.group(1)
                value = self._get_nested_value(config, field_path)
                if isinstance(value, str):
                    return f"'{value}'"
                return str(value)
            
            # Replace field references
            condition = re.sub(r'([a-zA-Z_][a-zA-Z0-9_.]*)', replace_field, condition)
            
            # Evaluate the condition
            return eval(condition)
        except:
            return False

def main():
    if len(sys.argv) != 3:
        print("Usage: config_validator.py <schema_file> <config_file>")
        sys.exit(1)
    
    schema_file = sys.argv[1]
    config_file = sys.argv[2]
    
    validator = ConfigValidator(schema_file)
    result = validator.validate_config(config_file)
    
    print(f"Configuration Validation Results for {config_file}:")
    print(f"Valid: {'Yes' if result['valid'] else 'No'}")
    
    if result['errors']:
        print("\nErrors:")
        for error in result['errors']:
            print(f"  ❌ {error}")
    
    if result['warnings']:
        print("\nWarnings:")
        for warning in result['warnings']:
            print(f"  ⚠️  {warning}")
    
    if result['valid']:
        print("\n✅ Configuration is valid!")
    else:
        print(f"\n❌ Configuration has {len(result['errors'])} error(s)")
        sys.exit(1)

if __name__ == '__main__':
    main()
EOF

chmod +x templates/validators/config_validator.py
```

Create comprehensive test for best practices:

```yaml
# Create test-best-practices.yml
cat > test-best-practices.yml << 'EOF'
---
- name: Test configuration templating best practices
  hosts: all
  gather_facts: yes
  vars_files:
    - config-data.yml
  vars:
    config_name: "enterprise-app"
    config_version: "2.1.0"
    config_schema_version: "1.0"
    include_application_config: true
    include_database_config: true
    include_cache_config: true
    include_logging_config: true
    include_monitoring_config: true
    include_security_config: true
    required_config_sections: ["application", "database", "logging"]
    optional_config_sections: ["cache", "monitoring", "security"]
    custom_fragments: []
  
  tasks:
    - name: Generate configuration using base template
      template:
        src: templates/base/base-config.j2
        dest: "/tmp/enterprise-config-{{ environment }}.yml"
        mode: '0644'
    
    - name: Validate generated configuration
      script: templates/validators/config_validator.py configs/schemas/config-schema.yml "/tmp/enterprise-config-{{ environment }}.yml"
      register: validation_result
      changed_when: false
      failed_when: false
    
    - name: Display validation results
      debug:
        msg: |
          Configuration Validation Results:
          {{ validation_result.stdout }}
          
          {% if validation_result.stderr %}
          Validation Errors:
          {{ validation_result.stderr }}
          {% endif %}
    
    - name: Generate configuration summary
      debug:
        msg: |
          Configuration Generation Summary:
          - Template: base-config.j2
          - Environment: {{ environment | upper }}
          - Config Name: {{ config_name }}
          - Config Version: {{ config_version }}
          - Schema Version: {{ config_schema_version }}
          - Fragments Included: {{ [include_application_config, include_database_config, include_cache_config, include_logging_config, include_monitoring_config, include_security_config] | select | list | length }}
          - Validation Status: {{ 'PASSED' if validation_result.rc == 0 else 'FAILED' }}
          - Generated File: /tmp/enterprise-config-{{ environment }}.yml
EOF

# Run best practices test
ansible-playbook -i inventory.ini test-best-practices.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Check generated configurations
echo "=== Generated Configuration Files ==="
ls -la /tmp/*config* /tmp/*nginx* 2>/dev/null || echo "No configuration files generated"

# Validate JSON configurations
echo "=== JSON Validation ==="
for file in /tmp/*.json; do
    if [ -f "$file" ]; then
        echo "Validating $file..."
        python3 -m json.tool "$file" > /dev/null && echo "✅ Valid" || echo "❌ Invalid"
    fi
done

# Check validation scripts
echo "=== Validation Scripts ==="
ls -la /tmp/validate-config-*.sh 2>/dev/null || echo "No validation scripts generated"
```

### 2. Discussion Points
- How do you manage configuration complexity across multiple environments?
- What strategies do you use for configuration validation and testing?
- How do you handle sensitive data in configuration templates?
- What are the benefits of modular configuration templates?

### 3. Clean Up
```bash
# Keep files for next exercises
# rm -rf /tmp/*config* /tmp/*nginx* /tmp/validate-*
```

## Key Takeaways
- Multi-format configuration generation supports diverse application requirements
- Environment-specific templates enable proper configuration management
- Modular template design improves maintainability and reusability
- Configuration validation prevents deployment issues
- Template fragments enable flexible configuration composition
- Best practices include schema validation, error handling, and documentation

## Next Steps
Proceed to Lab 6.3: Template Inheritance and Macros

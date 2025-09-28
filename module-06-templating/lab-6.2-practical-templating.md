# Lab 6.2: Practical Templating (Interactive Demo)

## Objective
Apply Jinja2 templating to real-world scenarios through practical examples and hands-on exercises suitable for all skill levels.

## Format
**Interactive demonstration** - Instructor-led with participant practice

## Prerequisites
- Completed Lab 6.1 (Jinja2 Essentials)
- Understanding of basic configuration file formats

## Demo Setup

```bash
cd ~/ansible-labs/module-06
mkdir -p lab-6.2
cd lab-6.2

# Create inventory with different environments
cat > inventory.ini << 'EOF'
[web_servers]
web-prod ansible_host=localhost ansible_connection=local env=production
web-stage ansible_host=localhost ansible_connection=local env=staging

[database_servers] 
db-prod ansible_host=localhost ansible_connection=local env=production

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Demo 1: Web Server Configuration (8 minutes)

### Apache/Nginx Configuration Template
```yaml
# Create web-config-demo.yml
cat > web-config-demo.yml << 'EOF'
---
- name: Web Server Configuration Demo
  hosts: web_servers
  vars:
    app_config:
      name: "WebApp"
      version: "2.1.0"
      
    # Environment-specific settings
    env_settings:
      production:
        port: 80
        ssl_enabled: true
        debug: false
        worker_processes: 4
        max_connections: 1000
      staging:
        port: 8080
        ssl_enabled: false
        debug: true
        worker_processes: 2
        max_connections: 100
        
  tasks:
    - name: Generate web server configuration
      copy:
        content: |
          # {{ app_config.name }} Configuration
          # Environment: {{ env }}
          # Generated: {{ ansible_date_time.iso8601 }}
          
          server {
              listen {{ env_settings[env].port }};
              server_name {{ inventory_hostname }}.example.com;
              
              {% if env_settings[env].ssl_enabled %}
              # SSL Configuration
              ssl_certificate /etc/ssl/{{ inventory_hostname }}.crt;
              ssl_certificate_key /etc/ssl/{{ inventory_hostname }}.key;
              ssl_protocols TLSv1.2 TLSv1.3;
              {% endif %}
              
              # Performance Settings
              worker_processes {{ env_settings[env].worker_processes }};
              worker_connections {{ env_settings[env].max_connections }};
              
              {% if env_settings[env].debug %}
              # Debug Mode Enabled
              error_log /var/log/nginx/{{ inventory_hostname }}_debug.log debug;
              access_log /var/log/nginx/{{ inventory_hostname }}_access.log combined;
              {% else %}
              # Production Logging
              error_log /var/log/nginx/{{ inventory_hostname }}_error.log warn;
              access_log /var/log/nginx/{{ inventory_hostname }}_access.log main;
              {% endif %}
              
              location / {
                  proxy_pass http://backend-{{ env }};
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  {% if env_settings[env].debug %}
                  proxy_set_header X-Debug-Mode "enabled";
                  {% endif %}
              }
          }
        dest: /tmp/{{ inventory_hostname }}-nginx.conf
        
    - name: Show generated configuration
      debug:
        msg: "Configuration created: /tmp/{{ inventory_hostname }}-nginx.conf"
EOF

# Run web config demo
ansible-playbook -i inventory.ini web-config-demo.yml

# Show results
echo "=== Production Web Server Config ==="
cat /tmp/web-prod-nginx.conf
echo
echo "=== Staging Web Server Config ==="
cat /tmp/web-stage-nginx.conf
```

### Key Patterns Demonstrated:
- **Environment-specific variables**
- **Conditional SSL configuration**
- **Dynamic hostnames and paths**
- **Debug vs production settings**

## Demo 2: Application Configuration (8 minutes)

### Multi-Format Configuration Generation
```yaml
# Create app-config-demo.yml
cat > app-config-demo.yml << 'EOF'
---
- name: Application Configuration Demo
  hosts: all
  vars:
    database_config:
      production:
        host: "prod-db.example.com"
        port: 5432
        name: "webapp_prod"
        pool_size: 20
      staging:
        host: "stage-db.example.com" 
        port: 5432
        name: "webapp_stage"
        pool_size: 5
        
    cache_servers:
      - host: "redis1.example.com"
        port: 6379
      - host: "redis2.example.com"
        port: 6379
        
    features:
      user_registration: true
      email_notifications: true
      analytics: "{{ env == 'production' }}"
      
  tasks:
    - name: Generate application properties (Java style)
      copy:
        content: |
          # {{ ansible_hostname }} Application Properties
          # Environment: {{ env }}
          # Generated: {{ ansible_date_time.iso8601 }}
          
          # Database Configuration
          db.host={{ database_config[env].host }}
          db.port={{ database_config[env].port }}
          db.name={{ database_config[env].name }}
          db.pool.size={{ database_config[env].pool_size }}
          
          # Cache Configuration
          {% for cache in cache_servers %}
          cache.server.{{ loop.index }}={{ cache.host }}:{{ cache.port }}
          {% endfor %}
          
          # Feature Flags
          {% for feature, enabled in features.items() %}
          feature.{{ feature }}={{ enabled | lower }}
          {% endfor %}
          
          # Environment Specific
          {% if env == 'production' %}
          logging.level=INFO
          monitoring.enabled=true
          {% else %}
          logging.level=DEBUG
          monitoring.enabled=false
          {% endif %}
        dest: /tmp/{{ inventory_hostname }}-app.properties
        
    - name: Generate YAML configuration
      copy:
        content: |
          # {{ ansible_hostname }} YAML Configuration
          application:
            name: "WebApp"
            environment: "{{ env }}"
            hostname: "{{ ansible_hostname }}"
            
          database:
            host: "{{ database_config[env].host }}"
            port: {{ database_config[env].port }}
            database: "{{ database_config[env].name }}"
            pool_size: {{ database_config[env].pool_size }}
            
          cache:
            servers:
          {% for cache in cache_servers %}
              - host: "{{ cache.host }}"
                port: {{ cache.port }}
          {% endfor %}
          
          features:
          {% for feature, enabled in features.items() %}
            {{ feature }}: {{ enabled | lower }}
          {% endfor %}
          
          logging:
            level: "{{ 'INFO' if env == 'production' else 'DEBUG' }}"
            
          monitoring:
            enabled: {{ (env == 'production') | lower }}
        dest: /tmp/{{ inventory_hostname }}-config.yml
EOF

# Run app config demo
ansible-playbook -i inventory.ini app-config-demo.yml

# Show results
echo "=== Application Properties (Production) ==="
cat /tmp/web-prod-app.properties
echo
echo "=== YAML Configuration (Production) ==="
cat /tmp/web-prod-config.yml
```

## Demo 3: Service and Script Generation (9 minutes)

### Systemd Service Template
```yaml
# Create service-demo.yml
cat > service-demo.yml << 'EOF'
---
- name: Service Generation Demo
  hosts: web_servers
  vars:
    service_config:
      user: "webapp"
      group: "webapp"
      working_dir: "/opt/webapp"
      java_opts: "{{ '-Xmx2g -Xms1g' if env == 'production' else '-Xmx512m -Xms256m' }}"
      
  tasks:
    - name: Generate systemd service file
      copy:
        content: |
          # {{ inventory_hostname }} Service Definition
          # Generated: {{ ansible_date_time.iso8601 }}
          
          [Unit]
          Description=WebApp Service ({{ env }})
          After=network.target
          Requires=network.target
          
          [Service]
          Type=forking
          User={{ service_config.user }}
          Group={{ service_config.group }}
          WorkingDirectory={{ service_config.working_dir }}
          
          # Environment specific settings
          {% if env == 'production' %}
          Environment="JAVA_OPTS={{ service_config.java_opts }}"
          Environment="SPRING_PROFILES_ACTIVE=production"
          {% else %}
          Environment="JAVA_OPTS={{ service_config.java_opts }}"
          Environment="SPRING_PROFILES_ACTIVE=staging,debug"
          {% endif %}
          
          ExecStart={{ service_config.working_dir }}/bin/webapp.sh start
          ExecStop={{ service_config.working_dir }}/bin/webapp.sh stop
          ExecReload={{ service_config.working_dir }}/bin/webapp.sh restart
          
          # Restart policy
          Restart=always
          RestartSec=10
          
          # Security settings
          {% if env == 'production' %}
          NoNewPrivileges=true
          PrivateTmp=true
          {% endif %}
          
          [Install]
          WantedBy=multi-user.target
        dest: /tmp/{{ inventory_hostname }}-webapp.service
        
    - name: Generate startup script
      copy:
        content: |
          #!/bin/bash
          # {{ inventory_hostname }} Startup Script
          # Environment: {{ env }}
          # Generated: {{ ansible_date_time.iso8601 }}
          
          APP_HOME="{{ service_config.working_dir }}"
          APP_USER="{{ service_config.user }}"
          PID_FILE="$APP_HOME/webapp.pid"
          
          {% if env == 'production' %}
          # Production settings
          JAVA_OPTS="{{ service_config.java_opts }} -server"
          LOG_LEVEL="INFO"
          {% else %}
          # Development/Staging settings
          JAVA_OPTS="{{ service_config.java_opts }} -client"
          LOG_LEVEL="DEBUG"
          {% endif %}
          
          start() {
              echo "Starting WebApp ({{ env }})..."
              {% if env == 'production' %}
              # Production startup with monitoring
              nohup java $JAVA_OPTS -jar $APP_HOME/webapp.jar \
                  --spring.profiles.active=production \
                  --logging.level.root=$LOG_LEVEL \
                  > $APP_HOME/logs/webapp.log 2>&1 &
              {% else %}
              # Development startup with console output
              java $JAVA_OPTS -jar $APP_HOME/webapp.jar \
                  --spring.profiles.active=staging,debug \
                  --logging.level.root=$LOG_LEVEL &
              {% endif %}
              echo $! > $PID_FILE
              echo "WebApp started with PID $(cat $PID_FILE)"
          }
          
          stop() {
              if [ -f $PID_FILE ]; then
                  PID=$(cat $PID_FILE)
                  echo "Stopping WebApp (PID: $PID)..."
                  kill $PID
                  rm -f $PID_FILE
              else
                  echo "WebApp is not running"
              fi
          }
          
          case "$1" in
              start)   start ;;
              stop)    stop ;;
              restart) stop; sleep 2; start ;;
              *)       echo "Usage: $0 {start|stop|restart}" ;;
          esac
        dest: /tmp/{{ inventory_hostname }}-webapp.sh
        mode: '0755'
EOF

# Run service demo
ansible-playbook -i inventory.ini service-demo.yml

# Show results
echo "=== Systemd Service File ==="
head -20 /tmp/web-prod-webapp.service
echo
echo "=== Startup Script ==="
head -15 /tmp/web-prod-webapp.sh
```

## Interactive Practice Session

### Hands-On Exercise for Participants:
```yaml
# Create practice-exercise.yml
cat > practice-exercise.yml << 'EOF'
---
- name: Participant Practice - Database Configuration
  hosts: database_servers
  vars:
    # Participants will work with this data
    db_settings:
      max_connections: 200
      shared_buffers: "256MB"
      work_mem: "4MB"
      
    backup_schedule:
      - time: "02:00"
        type: "full"
      - time: "14:00" 
        type: "incremental"
        
  tasks:
    - name: Generate PostgreSQL configuration
      copy:
        content: |
          # TODO: Participants create PostgreSQL config template
          # Include: basic settings, backup cron jobs, environment-specific tuning
          
          # Basic Configuration
          max_connections = {{ db_settings.max_connections }}
          
          # TODO: Add more configuration...
          # Hint: Use loops for backup schedule
          # Hint: Add conditional settings for production vs staging
        dest: /tmp/{{ inventory_hostname }}-postgresql.conf
        
    - name: Show your result
      debug:
        msg: "Check your PostgreSQL config in /tmp/{{ inventory_hostname }}-postgresql.conf"
EOF
```

**Practice Instructions:**
1. Complete the PostgreSQL configuration template
2. Add backup cron job entries using loops
3. Include environment-specific memory settings
4. Add comments with generation timestamp

## Key Takeaways Summary

### Practical Templating Patterns:
1. **Environment-specific configurations** - Different settings per environment
2. **Service definitions** - Generate systemd services and scripts
3. **Multi-format output** - Properties, YAML, INI, scripts
4. **Conditional features** - Enable/disable based on environment
5. **Dynamic lists** - Generate server lists, backup schedules

### Best Practices Demonstrated:
- Use descriptive variable names
- Group related settings in dictionaries
- Provide environment-specific defaults
- Include generation timestamps and metadata
- Test templates with different environments

### Common Use Cases:
- **Web server configs** - Nginx, Apache configurations
- **Application properties** - Java, Python, Node.js apps
- **Service definitions** - Systemd, init scripts
- **Database configurations** - PostgreSQL, MySQL, MongoDB
- **Monitoring configs** - Prometheus, Grafana dashboards

## Discussion Points

### For All Participants:
- "What configuration files do you manage manually that could be templated?"
- "How would templating help with your deployment process?"
- "What challenges do you face with environment-specific configurations?"

### For Advanced Users:
- "How do you handle sensitive data in templates?"
- "What's your strategy for template testing and validation?"
- "How do you manage template complexity as projects grow?"

## Demo Cleanup
```bash
# Clean up demo files
rm -f /tmp/*-nginx.conf /tmp/*-app.properties /tmp/*-config.yml /tmp/*-webapp.service /tmp/*-webapp.sh /tmp/*-postgresql.conf
```

---

**Note for Instructor**: Focus on practical, immediately applicable examples. Encourage participants to think about their own configuration management challenges. Adjust complexity based on audience response.

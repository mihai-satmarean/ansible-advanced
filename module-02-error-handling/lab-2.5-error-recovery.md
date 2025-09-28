# Lab 2.5: Advanced Error Recovery Patterns

## Objective
Implement sophisticated error recovery and rollback mechanisms for enterprise-grade automation resilience.

## Duration
20 minutes

## Prerequisites
- Completed all previous Module 2 labs
- Understanding of error handling and execution strategies

## Lab Setup

```bash
cd ~/ansible-labs/module-02
mkdir -p lab-2.5
cd lab-2.5

cat > inventory.ini << EOF
[production]
prod01 ansible_host=localhost ansible_connection=local
prod02 ansible_host=localhost ansible_connection=local

[staging]
stage01 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Automated Rollback Mechanisms (8 minutes)

### Task: Implement comprehensive rollback automation

Create `automated-rollback.yml`:
```yaml
---
- name: Automated rollback mechanisms
  hosts: all
  become: yes
  vars:
    application:
      name: "critical-app"
      current_version: "2.1.0"
      rollback_version: "2.0.5"
      health_check_url: "http://localhost:8080/health"
      rollback_timeout: 300
      
    deployment_state_file: "/var/lib/{{ application.name }}/deployment-state.json"
    
  tasks:
    # Pre-deployment state capture
    - name: Capture pre-deployment state
      block:
        - name: Create state directory
          file:
            path: "/var/lib/{{ application.name }}"
            state: directory
            mode: '0755'
        
        - name: Capture current application state
          shell: |
            # Capture current state
            if systemctl is-active {{ application.name }} >/dev/null 2>&1; then
              echo "service_status:active"
            else
              echo "service_status:inactive"
            fi
            
            if [ -f "/opt/{{ application.name }}/version.txt" ]; then
              echo "current_version:$(cat /opt/{{ application.name }}/version.txt)"
            else
              echo "current_version:unknown"
            fi
            
            echo "backup_timestamp:$(date +%s)"
            echo "host:{{ inventory_hostname }}"
          register: pre_deployment_state
        
        - name: Save pre-deployment state
          copy:
            content: |
              {
                "pre_deployment": {
                  {% for line in pre_deployment_state.stdout_lines %}
                  {% set key_value = line.split(':') %}
                  "{{ key_value[0] }}": "{{ key_value[1] }}",
                  {% endfor %}
                  "captured_at": "{{ ansible_date_time.iso8601 }}"
                },
                "rollback_available": true,
                "rollback_version": "{{ application.rollback_version }}"
              }
            dest: "{{ deployment_state_file }}"
            mode: '0644'
        
        - name: Create application backup
          shell: |
            backup_dir="/var/backups/{{ application.name }}/$(date +%Y%m%d-%H%M%S)"
            mkdir -p "$backup_dir"
            
            # Backup current application if exists
            if [ -d "/opt/{{ application.name }}" ]; then
              cp -r /opt/{{ application.name }}/* "$backup_dir/" 2>/dev/null || true
              echo "backup_path:$backup_dir"
            else
              echo "backup_path:none"
            fi
          register: backup_creation
      
      tags: pre_deployment
    
    # Main deployment with rollback capability
    - name: Deploy with rollback capability
      block:
        - name: Deploy new version
          block:
            - name: Create application directory
              file:
                path: "/opt/{{ application.name }}"
                state: directory
                mode: '0755'
            
            - name: Deploy application files
              copy:
                content: |
                  #!/bin/bash
                  # {{ application.name }} v{{ application.current_version }}
                  
                  echo "Starting {{ application.name }} v{{ application.current_version }}"
                  echo "Host: {{ inventory_hostname }}"
                  echo "Environment: {{ group_names[0] | default('unknown') }}"
                  
                  # Simulate application startup
                  # Introduce failure for demonstration
                  {% if inventory_hostname == 'prod02' and simulate_deployment_failure | default(false) %}
                  echo "CRITICAL ERROR: Deployment failed on {{ inventory_hostname }}"
                  exit 1
                  {% endif %}
                  
                  # Health check endpoint simulation
                  while true; do
                    echo "$(date): {{ application.name }} v{{ application.current_version }} running"
                    sleep 30
                  done
                dest: "/opt/{{ application.name }}/{{ application.name }}"
                mode: '0755'
            
            - name: Create version file
              copy:
                content: "{{ application.current_version }}"
                dest: "/opt/{{ application.name }}/version.txt"
                mode: '0644'
            
            - name: Create systemd service
              copy:
                content: |
                  [Unit]
                  Description={{ application.name | title }} v{{ application.current_version }}
                  After=network.target
                  
                  [Service]
                  Type=simple
                  ExecStart=/opt/{{ application.name }}/{{ application.name }}
                  Restart=always
                  RestartSec=10
                  
                  [Install]
                  WantedBy=multi-user.target
                dest: "/etc/systemd/system/{{ application.name }}.service"
                mode: '0644'
              notify: reload systemd
            
            - name: Start application service
              systemd:
                name: "{{ application.name }}"
                state: started
                enabled: yes
                daemon_reload: yes
            
            - name: Wait for application to be ready
              pause:
                seconds: 5
            
            - name: Verify deployment health
              shell: |
                # Simulate health check
                if systemctl is-active {{ application.name }} >/dev/null 2>&1; then
                  echo "Health check passed"
                  exit 0
                else
                  echo "Health check failed"
                  exit 1
                fi
              register: health_check
              retries: 3
              delay: 10
            
            - name: Update deployment state (success)
              copy:
                content: |
                  {
                    "deployment": {
                      "version": "{{ application.current_version }}",
                      "status": "success",
                      "deployed_at": "{{ ansible_date_time.iso8601 }}",
                      "host": "{{ inventory_hostname }}"
                    },
                    "rollback_available": false
                  }
                dest: "{{ deployment_state_file }}"
                mode: '0644'
        
        # Rollback block
        rescue:
          - name: Deployment failed - initiating automatic rollback
            debug:
              msg: "ALERT: Deployment failed on {{ inventory_hostname }}, initiating automatic rollback"
          
          - name: Stop failed service
            systemd:
              name: "{{ application.name }}"
              state: stopped
            failed_when: false
          
          - name: Read rollback information
            slurp:
              src: "{{ deployment_state_file }}"
            register: rollback_state_raw
            failed_when: false
          
          - name: Parse rollback state
            set_fact:
              rollback_state: "{{ rollback_state_raw.content | b64decode | from_json }}"
            when: rollback_state_raw is succeeded
          
          - name: Restore from backup
            shell: |
              if [ -d "/var/backups/{{ application.name }}" ]; then
                latest_backup=$(ls -t /var/backups/{{ application.name }}/ | head -1)
                if [ -n "$latest_backup" ]; then
                  echo "Restoring from backup: $latest_backup"
                  rm -rf /opt/{{ application.name }}/*
                  cp -r /var/backups/{{ application.name }}/$latest_backup/* /opt/{{ application.name }}/
                  echo "Backup restored successfully"
                else
                  echo "No backup found"
                  exit 1
                fi
              else
                echo "No backup directory found"
                exit 1
              fi
            register: backup_restore
          
          - name: Create rollback service file
            copy:
              content: |
                [Unit]
                Description={{ application.name | title }} v{{ application.rollback_version }} (Rolled Back)
                After=network.target
                
                [Service]
                Type=simple
                ExecStart=/opt/{{ application.name }}/{{ application.name }}
                Restart=always
                RestartSec=10
                
                [Install]
                WantedBy=multi-user.target
              dest: "/etc/systemd/system/{{ application.name }}.service"
              mode: '0644'
            notify: reload systemd
          
          - name: Start rolled back service
            systemd:
              name: "{{ application.name }}"
              state: started
              enabled: yes
              daemon_reload: yes
          
          - name: Verify rollback health
            shell: |
              if systemctl is-active {{ application.name }} >/dev/null 2>&1; then
                echo "Rollback health check passed"
                exit 0
              else
                echo "Rollback health check failed"
                exit 1
              fi
            register: rollback_health_check
            retries: 3
            delay: 5
          
          - name: Update deployment state (rollback)
            copy:
              content: |
                {
                  "deployment": {
                    "version": "{{ application.rollback_version }}",
                    "status": "rolled_back",
                    "rolled_back_at": "{{ ansible_date_time.iso8601 }}",
                    "host": "{{ inventory_hostname }}",
                    "original_failure": "deployment_failed"
                  },
                  "rollback_successful": true
                }
              dest: "{{ deployment_state_file }}"
              mode: '0644'
          
          - name: Send rollback notification
            copy:
              content: |
                ROLLBACK ALERT
                ==============
                
                Application: {{ application.name }}
                Host: {{ inventory_hostname }}
                Environment: {{ group_names[0] | default('unknown') }}
                
                Original Version: {{ application.current_version }}
                Rolled Back To: {{ application.rollback_version }}
                
                Rollback Time: {{ ansible_date_time.iso8601 }}
                Rollback Status: {{ 'SUCCESS' if rollback_health_check is succeeded else 'FAILED' }}
                
                Please investigate the deployment failure.
              dest: "/tmp/rollback-alert-{{ inventory_hostname }}.txt"
              mode: '0644'
          
          - name: Log rollback completion
            debug:
              msg: |
                Rollback completed on {{ inventory_hostname }}:
                - Original version: {{ application.current_version }}
                - Rolled back to: {{ application.rollback_version }}
                - Rollback health: {{ 'PASSED' if rollback_health_check is succeeded else 'FAILED' }}
      
      tags: deployment
  
  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes
```

### Run automated rollback:
```bash
# Run normal deployment
ansible-playbook -i inventory.ini automated-rollback.yml

# Run with simulated failure to trigger rollback
ansible-playbook -i inventory.ini automated-rollback.yml -e "simulate_deployment_failure=true"

# Check rollback results
cat /var/lib/critical-app/deployment-state.json | python3 -m json.tool 2>/dev/null || cat /var/lib/critical-app/deployment-state.json
cat /tmp/rollback-alert-*.txt 2>/dev/null || echo "No rollback alerts"
```

## Exercise 2: Circuit Breaker Pattern (6 minutes)

### Task: Implement circuit breaker pattern for resilient automation

Create `circuit-breaker.yml`:
```yaml
---
- name: Circuit breaker pattern implementation
  hosts: all
  vars:
    circuit_breaker:
      failure_threshold: 3
      recovery_timeout: 60
      half_open_max_calls: 2
      
    services_to_monitor:
      - name: "web-service"
        port: 8080
        critical: true
      - name: "api-service"
        port: 8081
        critical: true
      - name: "cache-service"
        port: 6379
        critical: false
        
  tasks:
    # Initialize circuit breaker state
    - name: Initialize circuit breaker state
      copy:
        content: |
          {
            "services": {
              {% for service in services_to_monitor %}
              "{{ service.name }}": {
                "state": "closed",
                "failure_count": 0,
                "last_failure": null,
                "last_success": null,
                "critical": {{ service.critical | lower }}
              }{{ ',' if not loop.last else '' }}
              {% endfor %}
            },
            "circuit_breaker_config": {
              "failure_threshold": {{ circuit_breaker.failure_threshold }},
              "recovery_timeout": {{ circuit_breaker.recovery_timeout }},
              "half_open_max_calls": {{ circuit_breaker.half_open_max_calls }}
            }
          }
        dest: "/tmp/circuit-breaker-state.json"
        mode: '0644'
    
    # Monitor services with circuit breaker logic
    - name: Monitor services with circuit breaker
      block:
        - name: Read current circuit breaker state
          slurp:
            src: "/tmp/circuit-breaker-state.json"
          register: cb_state_raw
        
        - name: Parse circuit breaker state
          set_fact:
            cb_state: "{{ cb_state_raw.content | b64decode | from_json }}"
        
        - name: Check service health with circuit breaker logic
          shell: |
            service_name="{{ item.name }}"
            service_state="{{ cb_state.services[item.name].state }}"
            failure_count="{{ cb_state.services[item.name].failure_count }}"
            
            echo "Checking $service_name (current state: $service_state)"
            
            # Simulate service check
            # Introduce failures for demonstration
            if [ "$service_name" = "api-service" ] && [ "{{ simulate_api_failure | default(false) }}" = "true" ]; then
              echo "Service check failed for $service_name"
              exit 1
            fi
            
            if [ "$service_name" = "cache-service" ] && [ "{{ simulate_cache_failure | default(false) }}" = "true" ]; then
              echo "Service check failed for $service_name"
              exit 1
            fi
            
            echo "Service check passed for $service_name"
            exit 0
          register: service_checks
          loop: "{{ services_to_monitor }}"
          failed_when: false
        
        - name: Update circuit breaker state based on results
          shell: |
            # Read current state
            cb_state='{{ cb_state | to_json }}'
            
            # Process each service result
            {% for result in service_checks.results %}
            {% set service = result.item %}
            service_name="{{ service.name }}"
            check_result={{ result.rc }}
            current_time=$(date +%s)
            
            # Get current service state
            current_state=$(echo "$cb_state" | python3 -c "
            import json, sys
            data = json.load(sys.stdin)
            print(data['services']['$service_name']['state'])
            ")
            
            failure_count=$(echo "$cb_state" | python3 -c "
            import json, sys
            data = json.load(sys.stdin)
            print(data['services']['$service_name']['failure_count'])
            ")
            
            echo "Processing $service_name: result=$check_result, state=$current_state, failures=$failure_count"
            
            # Circuit breaker logic
            if [ $check_result -eq 0 ]; then
              # Success - reset failure count and close circuit
              echo "$service_name: SUCCESS - resetting circuit breaker"
              new_state="closed"
              new_failure_count=0
              last_success=$current_time
              last_failure="null"
            else
              # Failure - increment failure count
              new_failure_count=$((failure_count + 1))
              last_failure=$current_time
              last_success="null"
              
              if [ $new_failure_count -ge {{ circuit_breaker.failure_threshold }} ]; then
                echo "$service_name: FAILURE THRESHOLD REACHED - opening circuit"
                new_state="open"
              else
                echo "$service_name: FAILURE $new_failure_count/{{ circuit_breaker.failure_threshold }}"
                new_state="closed"
              fi
            fi
            
            # Update state file
            python3 -c "
            import json
            with open('/tmp/circuit-breaker-state.json', 'r') as f:
                data = json.load(f)
            
            data['services']['$service_name']['state'] = '$new_state'
            data['services']['$service_name']['failure_count'] = $new_failure_count
            data['services']['$service_name']['last_failure'] = $last_failure
            data['services']['$service_name']['last_success'] = $last_success
            
            with open('/tmp/circuit-breaker-state.json', 'w') as f:
                json.dump(data, f, indent=2)
            "
            {% endfor %}
          register: cb_update
        
        - name: Read updated circuit breaker state
          slurp:
            src: "/tmp/circuit-breaker-state.json"
          register: updated_cb_state_raw
        
        - name: Parse updated state
          set_fact:
            updated_cb_state: "{{ updated_cb_state_raw.content | b64decode | from_json }}"
        
        - name: Take action based on circuit breaker state
          debug:
            msg: |
              Service: {{ item.key }}
              State: {{ item.value.state }}
              Failure Count: {{ item.value.failure_count }}
              Critical: {{ item.value.critical }}
              Action: {{ 'BLOCK_REQUESTS' if item.value.state == 'open' else 'ALLOW_REQUESTS' }}
              {% if item.value.state == 'open' and item.value.critical %}
              ALERT: Critical service {{ item.key }} circuit is OPEN!
              {% endif %}
          loop: "{{ updated_cb_state.services | dict2items }}"
        
        - name: Generate circuit breaker alerts
          copy:
            content: |
              Circuit Breaker Alert
              ====================
              
              Host: {{ inventory_hostname }}
              Time: {{ ansible_date_time.iso8601 }}
              
              Open Circuits (Failing Services):
              {% for service_name, service_data in updated_cb_state.services.items() %}
              {% if service_data.state == 'open' %}
              - {{ service_name }} ({{ 'CRITICAL' if service_data.critical else 'NON-CRITICAL' }})
                Failure Count: {{ service_data.failure_count }}
                Last Failure: {{ service_data.last_failure }}
              {% endif %}
              {% endfor %}
              
              {% if updated_cb_state.services.values() | selectattr('state', 'equalto', 'open') | selectattr('critical') | list | length > 0 %}
              IMMEDIATE ACTION REQUIRED: Critical services are failing!
              {% endif %}
            dest: "/tmp/circuit-breaker-alert-{{ inventory_hostname }}.txt"
            mode: '0644'
          when: updated_cb_state.services.values() | selectattr('state', 'equalto', 'open') | list | length > 0
      
      tags: circuit_breaker
```

### Run circuit breaker pattern:
```bash
# Run normal circuit breaker monitoring
ansible-playbook -i inventory.ini circuit-breaker.yml

# Run with simulated failures to trigger circuit breaker
ansible-playbook -i inventory.ini circuit-breaker.yml -e "simulate_api_failure=true simulate_cache_failure=true"

# Check circuit breaker state
cat /tmp/circuit-breaker-state.json | python3 -m json.tool
cat /tmp/circuit-breaker-alert-*.txt 2>/dev/null || echo "No circuit breaker alerts"
```

## Exercise 3: Self-Healing Automation (6 minutes)

### Task: Implement self-healing automation patterns

Create `self-healing.yml`:
```yaml
---
- name: Self-healing automation patterns
  hosts: all
  vars:
    healing_config:
      max_healing_attempts: 3
      healing_interval: 30
      escalation_threshold: 2
      
    monitored_services:
      - name: "web-app"
        healing_actions: ["restart_service", "clear_cache", "restart_dependencies"]
        escalation_actions: ["notify_ops", "failover"]
      - name: "database"
        healing_actions: ["restart_service", "check_disk_space", "repair_tables"]
        escalation_actions: ["notify_dba", "switch_to_replica"]
        
  tasks:
    # Initialize self-healing state
    - name: Initialize self-healing state
      copy:
        content: |
          {
            "services": {
              {% for service in monitored_services %}
              "{{ service.name }}": {
                "status": "unknown",
                "healing_attempts": 0,
                "last_healing": null,
                "escalated": false,
                "healing_history": []
              }{{ ',' if not loop.last else '' }}
              {% endfor %}
            },
            "healing_config": {{ healing_config | to_json }}
          }
        dest: "/tmp/self-healing-state.json"
        mode: '0644'
    
    # Self-healing monitoring and action loop
    - name: Self-healing monitoring and recovery
      block:
        - name: Check service health
          shell: |
            service_name="{{ item.name }}"
            
            # Simulate service health check
            echo "Checking health of $service_name"
            
            # Simulate service failures for demonstration
            if [ "$service_name" = "web-app" ] && [ "{{ simulate_web_failure | default(false) }}" = "true" ]; then
              echo "Service $service_name is unhealthy"
              exit 1
            fi
            
            if [ "$service_name" = "database" ] && [ "{{ simulate_db_failure | default(false) }}" = "true" ]; then
              echo "Service $service_name is unhealthy"
              exit 1
            fi
            
            echo "Service $service_name is healthy"
            exit 0
          register: health_checks
          loop: "{{ monitored_services }}"
          failed_when: false
        
        - name: Perform self-healing for unhealthy services
          block:
            - name: Read current healing state
              slurp:
                src: "/tmp/self-healing-state.json"
              register: healing_state_raw
            
            - name: Parse healing state
              set_fact:
                healing_state: "{{ healing_state_raw.content | b64decode | from_json }}"
            
            - name: Execute healing actions for failed services
              shell: |
                service_name="{{ item.item.name }}"
                healing_attempts="{{ healing_state.services[item.item.name].healing_attempts }}"
                max_attempts="{{ healing_config.max_healing_attempts }}"
                
                echo "Service $service_name failed health check"
                echo "Current healing attempts: $healing_attempts/$max_attempts"
                
                if [ $healing_attempts -lt $max_attempts ]; then
                  echo "Attempting self-healing for $service_name"
                  
                  # Execute healing actions
                  {% for action in item.item.healing_actions %}
                  echo "Executing healing action: {{ action }}"
                  case "{{ action }}" in
                    "restart_service")
                      echo "Restarting $service_name service"
                      # systemctl restart $service_name || true
                      ;;
                    "clear_cache")
                      echo "Clearing cache for $service_name"
                      # rm -rf /tmp/cache/$service_name/* || true
                      ;;
                    "restart_dependencies")
                      echo "Restarting dependencies for $service_name"
                      ;;
                    "check_disk_space")
                      echo "Checking disk space for $service_name"
                      df -h
                      ;;
                    "repair_tables")
                      echo "Repairing database tables for $service_name"
                      ;;
                  esac
                  {% endfor %}
                  
                  echo "Self-healing attempt completed for $service_name"
                  exit 0
                else
                  echo "Max healing attempts reached for $service_name - escalating"
                  exit 2
                fi
              register: healing_actions
              loop: "{{ health_checks.results }}"
              when: item.rc != 0
              failed_when: false
            
            - name: Update healing state
              shell: |
                python3 -c "
                import json
                from datetime import datetime
                
                with open('/tmp/self-healing-state.json', 'r') as f:
                    data = json.load(f)
                
                # Update healing state for each service
                {% for result in healing_actions.results | default([]) %}
                {% if result.item is defined %}
                service_name = '{{ result.item.item.name }}'
                healing_result = {{ result.rc | default(0) }}
                current_time = datetime.now().isoformat()
                
                if service_name in data['services']:
                    if healing_result == 0:
                        # Healing successful
                        data['services'][service_name]['healing_attempts'] += 1
                        data['services'][service_name]['last_healing'] = current_time
                        data['services'][service_name]['status'] = 'healing_attempted'
                        data['services'][service_name]['healing_history'].append({
                            'timestamp': current_time,
                            'action': 'healing_attempted',
                            'result': 'success'
                        })
                    elif healing_result == 2:
                        # Escalation needed
                        data['services'][service_name]['escalated'] = True
                        data['services'][service_name]['status'] = 'escalated'
                        data['services'][service_name]['healing_history'].append({
                            'timestamp': current_time,
                            'action': 'escalation_triggered',
                            'result': 'max_attempts_reached'
                        })
                {% endif %}
                {% endfor %}
                
                with open('/tmp/self-healing-state.json', 'w') as f:
                    json.dump(data, f, indent=2)
                "
            
            - name: Execute escalation actions for services requiring escalation
              shell: |
                service_name="{{ item.item.name }}"
                
                echo "Executing escalation actions for $service_name"
                
                {% for action in item.item.escalation_actions %}
                echo "Executing escalation action: {{ action }}"
                case "{{ action }}" in
                  "notify_ops")
                    echo "Sending notification to operations team for $service_name"
                    echo "ESCALATION: $service_name requires manual intervention" > /tmp/ops-notification-$service_name.txt
                    ;;
                  "notify_dba")
                    echo "Sending notification to DBA team for $service_name"
                    echo "ESCALATION: $service_name database requires DBA attention" > /tmp/dba-notification-$service_name.txt
                    ;;
                  "failover")
                    echo "Initiating failover for $service_name"
                    ;;
                  "switch_to_replica")
                    echo "Switching to replica for $service_name"
                    ;;
                esac
                {% endfor %}
                
                echo "Escalation actions completed for $service_name"
              loop: "{{ healing_actions.results | default([]) }}"
              when: 
                - item.item is defined
                - item.rc == 2
        
        - name: Generate self-healing report
          copy:
            content: |
              Self-Healing Report
              ==================
              
              Host: {{ inventory_hostname }}
              Report Time: {{ ansible_date_time.iso8601 }}
              
              Service Health Status:
              {% for check in health_checks.results %}
              - {{ check.item.name }}: {{ 'HEALTHY' if check.rc == 0 else 'UNHEALTHY' }}
              {% endfor %}
              
              Healing Actions Taken:
              {% for action in healing_actions.results | default([]) %}
              {% if action.item is defined %}
              - {{ action.item.item.name }}: {{ 'HEALING_ATTEMPTED' if action.rc == 0 else 'ESCALATED' if action.rc == 2 else 'NO_ACTION' }}
              {% endif %}
              {% endfor %}
              
              Escalations:
              {% for action in healing_actions.results | default([]) %}
              {% if action.item is defined and action.rc == 2 %}
              - {{ action.item.item.name }}: ESCALATED - Manual intervention required
              {% endif %}
              {% endfor %}
            dest: "/tmp/self-healing-report-{{ inventory_hostname }}.txt"
            mode: '0644'
      
      tags: self_healing
```

### Run self-healing automation:
```bash
# Run normal self-healing monitoring
ansible-playbook -i inventory.ini self-healing.yml

# Run with simulated failures to trigger self-healing
ansible-playbook -i inventory.ini self-healing.yml -e "simulate_web_failure=true simulate_db_failure=true"

# Check self-healing results
cat /tmp/self-healing-state.json | python3 -m json.tool
cat /tmp/self-healing-report-*.txt
ls -la /tmp/*-notification-*.txt 2>/dev/null || echo "No escalation notifications"
```

## Verification and Discussion

### 1. Check Results
```bash
# Check automated rollback results
sudo systemctl status critical-app --no-pager || echo "Service not running"
cat /var/lib/critical-app/deployment-state.json | python3 -m json.tool 2>/dev/null || echo "No deployment state"

# Check circuit breaker results
cat /tmp/circuit-breaker-state.json | python3 -m json.tool

# Check self-healing results
cat /tmp/self-healing-report-*.txt
cat /tmp/ops-notification-*.txt 2>/dev/null || echo "No ops notifications"
```

### 2. Discussion Points
- How do you implement rollback mechanisms in your current deployments?
- What patterns do you use for handling cascading failures?
- How do you balance automated recovery with manual intervention?

### 3. Clean Up
```bash
# Stop and remove services
sudo systemctl stop critical-app 2>/dev/null || true
sudo systemctl disable critical-app 2>/dev/null || true

# Remove files
sudo rm -rf /var/lib/critical-app /var/backups/critical-app
sudo rm -f /etc/systemd/system/critical-app.service
sudo rm -f /tmp/circuit-breaker-* /tmp/self-healing-* /tmp/rollback-alert-* /tmp/*-notification-*
sudo systemctl daemon-reload
```

## Key Takeaways
- Automated rollback mechanisms provide safety nets for failed deployments
- Circuit breaker patterns prevent cascading failures in distributed systems
- Self-healing automation reduces manual intervention and improves system resilience
- State management is crucial for tracking recovery attempts and escalations
- Proper monitoring and alerting enable proactive issue resolution
- Balancing automation with human oversight is essential for critical systems

## Module 2 Summary

### What We Covered
1. **Error Handling Fundamentals** - Basic error detection and handling patterns
2. **Block, Rescue, and Always Patterns** - Structured error handling workflows
3. **Conditional Execution and Flow Control** - Intelligent automation decisions
4. **Execution Strategies and Performance** - Optimization and parallelism
5. **Advanced Error Recovery Patterns** - Sophisticated resilience mechanisms

### Key Skills Developed
- Implementing comprehensive error handling strategies
- Building resilient automation with rollback capabilities
- Optimizing playbook execution for performance and reliability
- Creating self-healing and circuit breaker patterns
- Managing complex deployment scenarios with proper error recovery

### Next Steps
You are now equipped with advanced error handling and execution strategies. These skills will be essential for building robust, enterprise-grade automation in the subsequent modules.

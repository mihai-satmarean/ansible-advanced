# Lab 8.2: Advanced Delegation Patterns (Quick Reference)

## Pattern 1: Load Balancer Management

### Zero-Downtime Deployment
```yaml
---
- name: Zero-Downtime Deployment
  hosts: web_servers
  serial: 1  # One server at a time
  
  tasks:
    # Remove from load balancer
    - name: Drain server
      lineinfile:
        path: /tmp/lb-config/servers.conf
        regexp: "^{{ inventory_hostname }}"
        state: absent
      delegate_to: load_balancer
      
    - name: Wait for draining
      pause: seconds=10
        
    # Deploy application
    - name: Deploy new version
      copy:
        content: "App v2.0 on {{ inventory_hostname }}"
        dest: /tmp/app-{{ inventory_hostname }}/version.txt
        
    # Add back to load balancer
    - name: Re-register server
      lineinfile:
        path: /tmp/lb-config/servers.conf
        line: "{{ inventory_hostname }} active"
      delegate_to: load_balancer
```

### Test Setup:
```bash
# Create inventory
cat > inventory.ini << 'EOF'
[web_servers]
web1 ansible_host=localhost ansible_connection=local
web2 ansible_host=localhost ansible_connection=local

[load_balancer]
lb1 ansible_host=localhost ansible_connection=local
EOF

# Create directories
mkdir -p /tmp/lb-config /tmp/app-{web1,web2}

# Run playbook
ansible-playbook -i inventory.ini playbook.yml
```

## Pattern 2: Database Coordination

### Primary/Replica Management
```yaml
---
- name: Database Coordination
  hosts: database_servers
  
  tasks:
    # Primary operations
    - name: Create backup
      shell: echo "Backup {{ ansible_date_time.epoch }}" > /tmp/backup.sql
      when: db_role == 'primary'
      
    - name: Log backup
      lineinfile:
        path: /tmp/backup.log
        line: "{{ ansible_date_time.iso8601 }} - Backup created"
        create: yes
      delegate_to: management_server
      when: db_role == 'primary'
      
    # Replica operations
    - name: Update replica config
      copy:
        content: |
          primary={{ groups['database_primary'][0] }}
          last_sync={{ ansible_date_time.iso8601 }}
        dest: /tmp/replica-{{ inventory_hostname }}.conf
      when: db_role == 'replica'
```

### Test Setup:
```bash
cat > db-inventory.ini << 'EOF'
[database_primary]
db1 ansible_host=localhost ansible_connection=local db_role=primary

[database_replicas]
db2 ansible_host=localhost ansible_connection=local db_role=replica

[database_servers:children]
database_primary
database_replicas

[management_server]
mgmt1 ansible_host=localhost ansible_connection=local
EOF
```

## Pattern 3: Multi-Environment Workflow

### Staging → Production Pipeline
```yaml
---
- name: Multi-Environment Deployment
  hosts: localhost
  connection: local
  
  tasks:
    # Deploy to staging
    - name: Deploy staging
      debug: msg="Deploying to {{ item }}"
      delegate_to: "{{ item }}"
      loop: "{{ groups['staging'] }}"
      
    # Validate staging
    - name: Validate staging
      copy:
        content: "Staging OK - {{ ansible_date_time.iso8601 }}"
        dest: /tmp/staging-validation.txt
      delegate_to: management_server
      
    # Deploy to production (conditional)
    - name: Deploy production
      debug: msg="Deploying to {{ item }}"
      delegate_to: "{{ item }}"
      loop: "{{ groups['production'] }}"
      when: staging_validated | default(true)
```

## Key Concepts Summary

### Basic Delegation Syntax:
```yaml
- name: Task description
  module_name:
    # parameters
  delegate_to: target_host
```

### Common Patterns:
- **Cross-host coordination**: `delegate_to` different host
- **Centralized operations**: `delegate_to` management server
- **One-time operations**: `delegate_to` + `run_once: true`
- **Serial deployment**: `serial: 1` + delegation

### Use Cases:
1. **Service Registration** - Register with load balancers/monitoring
2. **Centralized Logging** - Collect status from multiple hosts
3. **Cross-Host Dependencies** - Coordinate between services
4. **Orchestration** - Complex deployment workflows

## Best Practices

### Performance:
- Use `run_once` to avoid redundant operations
- Consider network overhead of delegation
- Cache facts when possible

### Patterns:
- `serial: 1` for zero-downtime deployments
- Conditional delegation based on host roles
- Combine with loops for batch operations

## Verification

```bash
# Check results
ls -la /tmp/lb-config/ /tmp/app-*/
cat /tmp/backup.log
ls -la /tmp/replica-*.conf /tmp/staging-validation.txt

# Cleanup
rm -rf /tmp/lb-config /tmp/app-* /tmp/backup.* /tmp/replica-* /tmp/staging-*
```

## Next Steps

### Real-World Applications:
- CI/CD pipeline integration
- Cloud service orchestration
- Monitoring system integration
- Security automation workflows

### Additional Learning:
- Practice with real infrastructure
- Integrate with existing automation workflows

---


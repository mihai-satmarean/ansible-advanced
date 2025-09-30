# Lab 8.1: Delegation Demo (Interactive)

## Objective
Understand Ansible delegation concepts through interactive demonstration and simple practical examples suitable for all skill levels.

## Duration
15 minutes (interactive demo)

## Format
**Interactive demonstration** - Instructor-led with participant Q&A

## Prerequisites
- Basic understanding of Ansible playbooks and inventory
- Familiarity with task execution concepts

## Demo Setup

```bash
# Simple inventory for demonstration
cat > demo-inventory.ini << 'EOF'
[web_servers]
web1 ansible_host=localhost ansible_connection=local
web2 ansible_host=localhost ansible_connection=local

[database_servers]
db1 ansible_host=localhost ansible_connection=local

[management_servers]
mgmt1 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Demo 1: What is Delegation? (3 minutes)

### Concept Explanation
**Without Delegation (Normal Ansible):**
```yaml
- name: Install package on web servers
  hosts: web_servers
  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present
```
*Task runs ON the web servers*

**With Delegation:**
```yaml
- name: Register web servers with load balancer
  hosts: web_servers
  tasks:
    - name: Add server to load balancer config
      lineinfile:
        path: /etc/loadbalancer/servers.conf
        line: "server {{ inventory_hostname }} {{ ansible_default_ipv4.address }}"
      delegate_to: loadbalancer1
```
*Task runs ON the load balancer, but uses data FROM the web server*

### Key Points
- **Normal**: Task runs on target host
- **Delegation**: Task runs on different host but uses target host's data
- **Use cases**: Service registration, centralized logging, cross-host coordination

## Demo 2: Basic Delegation Syntax (4 minutes)

### Simple Example - Live Demo
```yaml
# demo-basic-delegation.yml
---
- name: Basic Delegation Demo
  hosts: web_servers
  gather_facts: yes
  
  tasks:
    - name: Show normal task (runs on web server)
      debug:
        msg: "This runs on {{ inventory_hostname }}"
    
    - name: Show delegated task (runs on management server)
      debug:
        msg: "This runs on mgmt1 but processes data from {{ inventory_hostname }}"
      delegate_to: mgmt1
    
    - name: Create file locally (normal)
      copy:
        content: "Created on {{ inventory_hostname }}"
        dest: "/tmp/local-{{ inventory_hostname }}.txt"
    
    - name: Create file on management server (delegated)
      copy:
        content: "Info about {{ inventory_hostname }} created on mgmt1"
        dest: "/tmp/remote-{{ inventory_hostname }}.txt"
      delegate_to: mgmt1
```

**Run and show results:**
```bash
ansible-playbook -i demo-inventory.ini demo-basic-delegation.yml
ls /tmp/local-* /tmp/remote-*
```

### Key Syntax
- `delegate_to: hostname` - Run task on specified host
- `delegate_facts: true` - Gather facts for delegated host
- `run_once: true` - Run only once when combined with delegation

## Demo 3: Practical Use Cases (5 minutes)

### Use Case 1: Service Registration
```yaml
- name: Register web server with monitoring
  hosts: web_servers
  tasks:
    - name: Add monitoring target
      lineinfile:
        path: /tmp/monitoring-targets.yml
        line: "  - {{ inventory_hostname }}:{{ web_port | default(80) }}"
        create: yes
      delegate_to: mgmt1
```

### Use Case 2: Centralized Logging
```yaml
- name: Log deployment status
  hosts: web_servers
  tasks:
    - name: Record deployment
      lineinfile:
        path: /tmp/deployment.log
        line: "{{ ansible_date_time.iso8601 }} - Deployed {{ inventory_hostname }}"
        create: yes
      delegate_to: mgmt1
```

### Use Case 3: Cross-Host Coordination
```yaml
- name: Backup database before web deployment
  hosts: web_servers
  tasks:
    - name: Create database backup
      shell: echo "Backup created for web deployment"
      delegate_to: db1
      run_once: true  # Only run once, not for each web server
```

## Demo 4: Common Patterns (3 minutes)

### Pattern 1: Collect Information Centrally
```yaml
- name: Collect server information
  hosts: web_servers
  tasks:
    - name: Gather server info centrally
      copy:
        content: |
          Server: {{ inventory_hostname }}
          IP: {{ ansible_default_ipv4.address | default('unknown') }}
          Memory: {{ ansible_memtotal_mb | default(0) }}MB
        dest: "/tmp/server-info-{{ inventory_hostname }}.txt"
      delegate_to: mgmt1
```

### Pattern 2: Load Balancer Management
```yaml
- name: Update load balancer
  hosts: web_servers
  serial: 1  # One at a time
  tasks:
    - name: Remove from load balancer
      debug:
        msg: "Removing {{ inventory_hostname }} from load balancer"
      delegate_to: mgmt1
    
    - name: Deploy application
      debug:
        msg: "Deploying on {{ inventory_hostname }}"
    
    - name: Add back to load balancer
      debug:
        msg: "Adding {{ inventory_hostname }} back to load balancer"
      delegate_to: mgmt1
```

## Interactive Q&A and Discussion

### Questions for Participants:
1. **When would you use delegation instead of running tasks normally?**
   - Service registration
   - Centralized logging
   - Cross-host coordination
   - Load balancer management

2. **What's the difference between `delegate_to` and `local_action`?**
   - `delegate_to`: Run on any specified host
   - `local_action`: Always run on Ansible control node

3. **How does delegation help with orchestration?**
   - Coordinate between different services
   - Centralize configuration management
   - Enable zero-downtime deployments


## Key Takeaways (Summary)

### What is Delegation?
- Run tasks on different hosts than the target hosts
- Access target host data from the delegated host
- Essential for orchestration and service coordination

### Basic Syntax:
```yaml
- name: Task description
  module_name:
    # module parameters
  delegate_to: target_host
```

### Common Use Cases:
1. **Service Registration** - Register services with load balancers/monitoring
2. **Centralized Logging** - Collect logs/status from multiple hosts
3. **Cross-Host Coordination** - Coordinate actions between different services
4. **Orchestration** - Manage complex deployment workflows

### When to Use:
- Need to coordinate between different hosts/services
- Centralized configuration management
- Service discovery and registration
- Complex deployment orchestration

### Next Steps for Advanced Users:
- Explore Lab 8.2 for complex orchestration patterns
- Practice with real load balancer configurations
- Implement in CI/CD pipelines

## Demo Files Cleanup
```bash
# Clean up demo files
rm -f /tmp/local-* /tmp/remote-* /tmp/server-info-* /tmp/monitoring-targets.yml /tmp/deployment.log
rm -f demo-inventory.ini demo-basic-delegation.yml
```

---


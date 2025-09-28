# Lab 4.1: Static vs Dynamic Inventory Fundamentals

## Objective
Understand inventory concepts and transition from static to dynamic approaches for scalable automation.

## Duration
20 minutes

## Prerequisites
- Completed previous modules
- Understanding of basic inventory concepts

## Lab Setup

```bash
cd ~/ansible-labs
mkdir -p module-04/lab-4.1
cd module-04/lab-4.1

# Create directory structure
mkdir -p static-inventory dynamic-inventory scripts data
```

## Exercise 1: Static Inventory Analysis (8 minutes)

### Task: Create and analyze traditional static inventory structures

Create static inventory files:

```bash
# Create basic static inventory
cat > static-inventory/hosts.ini << EOF
# Basic Static Inventory
[webservers]
web01 ansible_host=192.168.1.10 ansible_user=ubuntu
web02 ansible_host=192.168.1.11 ansible_user=ubuntu
web03 ansible_host=192.168.1.12 ansible_user=ubuntu

[databases]
db01 ansible_host=192.168.1.20 ansible_user=postgres
db02 ansible_host=192.168.1.21 ansible_user=postgres

[loadbalancers]
lb01 ansible_host=192.168.1.30 ansible_user=nginx

[production:children]
webservers
databases
loadbalancers

[webservers:vars]
http_port=80
https_port=443
app_env=production

[databases:vars]
db_port=5432
backup_enabled=true

[production:vars]
environment=production
monitoring_enabled=true
backup_retention=30
EOF

# Create group variables
mkdir -p static-inventory/group_vars
cat > static-inventory/group_vars/webservers.yml << EOF
---
# Web servers configuration
nginx_worker_processes: 4
nginx_worker_connections: 1024
ssl_certificate: "/etc/ssl/certs/company.crt"
ssl_private_key: "/etc/ssl/private/company.key"

# Application settings
app_name: "enterprise-web"
app_version: "2.1.0"
app_port: 8080

# Monitoring
monitoring_agent: "datadog"
log_level: "INFO"
EOF

cat > static-inventory/group_vars/databases.yml << EOF
---
# Database configuration
postgresql_version: "13"
postgresql_max_connections: 200
postgresql_shared_buffers: "256MB"
postgresql_effective_cache_size: "1GB"

# Backup configuration
backup_schedule: "0 2 * * *"
backup_retention_days: 30
backup_compression: true

# Security
ssl_enabled: true
password_encryption: "scram-sha-256"
EOF

# Create host variables
mkdir -p static-inventory/host_vars
cat > static-inventory/host_vars/web01.yml << EOF
---
# Web01 specific configuration
server_role: "primary"
nginx_worker_processes: 8  # Override group default
special_config: true

# Hardware specific
cpu_cores: 8
memory_gb: 16
disk_type: "ssd"
EOF

cat > static-inventory/host_vars/db01.yml << EOF
---
# DB01 specific configuration
server_role: "master"
postgresql_max_connections: 500  # Override group default
replication_enabled: true

# Hardware specific
cpu_cores: 16
memory_gb: 64
storage_type: "nvme"
raid_level: "10"
EOF
```

Create analysis playbook:

```yaml
# Create static-inventory-analysis.yml
cat > static-inventory-analysis.yml << 'EOF'
---
- name: Static inventory analysis
  hosts: all
  gather_facts: no
  tasks:
    - name: Display inventory information
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Groups: {{ group_names | join(', ') }}
          Ansible Host: {{ ansible_host | default('localhost') }}
          Environment: {{ environment | default('undefined') }}
          Server Role: {{ server_role | default('standard') }}
          
          {% if inventory_hostname in groups['webservers'] %}
          Web Server Config:
          - Worker Processes: {{ nginx_worker_processes }}
          - App Name: {{ app_name }}
          - App Version: {{ app_version }}
          {% endif %}
          
          {% if inventory_hostname in groups['databases'] %}
          Database Config:
          - PostgreSQL Version: {{ postgresql_version }}
          - Max Connections: {{ postgresql_max_connections }}
          - Backup Enabled: {{ backup_enabled }}
          {% endif %}
    
    - name: Analyze inventory structure
      debug:
        msg: |
          Inventory Analysis:
          - Total hosts: {{ groups['all'] | length }}
          - Web servers: {{ groups['webservers'] | length }}
          - Databases: {{ groups['databases'] | length }}
          - Load balancers: {{ groups['loadbalancers'] | length }}
          - Production hosts: {{ groups['production'] | length }}
      run_once: true
    
    - name: Identify static inventory limitations
      debug:
        msg: |
          Static Inventory Limitations Observed:
          - Manual host management required
          - No automatic discovery of new instances
          - Difficult to sync with cloud infrastructure
          - Manual updates needed for IP changes
          - No integration with external CMDBs
          - Limited scalability for large environments
      run_once: true
EOF
```

### Run static inventory analysis:
```bash
# Run static inventory analysis
ansible-playbook -i static-inventory/hosts.ini static-inventory-analysis.yml

# Test inventory commands
ansible-inventory -i static-inventory/hosts.ini --list
ansible-inventory -i static-inventory/hosts.ini --graph
```

## Exercise 2: Simple Dynamic Inventory Introduction (7 minutes)

### Task: Create a basic dynamic inventory script

Create simple dynamic inventory script:

```python
# Create scripts/simple-dynamic-inventory.py
cat > scripts/simple-dynamic-inventory.py << 'EOF'
#!/usr/bin/env python3
"""
Simple Dynamic Inventory Script
Demonstrates basic dynamic inventory concepts
"""

import json
import sys
import argparse
from datetime import datetime

def get_inventory():
    """Generate dynamic inventory"""
    
    # Simulate dynamic host discovery
    inventory = {
        "_meta": {
            "hostvars": {
                "dynamic-web01": {
                    "ansible_host": "192.168.1.50",
                    "ansible_user": "ubuntu",
                    "server_role": "web",
                    "cpu_cores": 4,
                    "memory_gb": 8,
                    "discovered_at": datetime.now().isoformat(),
                    "cloud_provider": "azure",
                    "instance_type": "Standard_B2s"
                },
                "dynamic-web02": {
                    "ansible_host": "192.168.1.51",
                    "ansible_user": "ubuntu", 
                    "server_role": "web",
                    "cpu_cores": 4,
                    "memory_gb": 8,
                    "discovered_at": datetime.now().isoformat(),
                    "cloud_provider": "azure",
                    "instance_type": "Standard_B2s"
                },
                "dynamic-db01": {
                    "ansible_host": "192.168.1.60",
                    "ansible_user": "postgres",
                    "server_role": "database",
                    "cpu_cores": 8,
                    "memory_gb": 32,
                    "discovered_at": datetime.now().isoformat(),
                    "cloud_provider": "azure",
                    "instance_type": "Standard_D4s_v3"
                }
            }
        },
        "webservers": {
            "hosts": ["dynamic-web01", "dynamic-web02"],
            "vars": {
                "http_port": 80,
                "https_port": 443,
                "app_env": "production",
                "discovery_method": "dynamic"
            }
        },
        "databases": {
            "hosts": ["dynamic-db01"],
            "vars": {
                "db_port": 5432,
                "backup_enabled": True,
                "discovery_method": "dynamic"
            }
        },
        "azure_instances": {
            "hosts": ["dynamic-web01", "dynamic-web02", "dynamic-db01"],
            "vars": {
                "cloud_provider": "azure",
                "managed_by": "ansible_dynamic_inventory"
            }
        },
        "production": {
            "children": ["webservers", "databases"],
            "vars": {
                "environment": "production",
                "monitoring_enabled": True,
                "inventory_type": "dynamic"
            }
        }
    }
    
    return inventory

def get_host_vars(hostname):
    """Get variables for specific host"""
    inventory = get_inventory()
    return inventory.get("_meta", {}).get("hostvars", {}).get(hostname, {})

def main():
    parser = argparse.ArgumentParser(description='Simple Dynamic Inventory')
    parser.add_argument('--list', action='store_true', help='List all hosts')
    parser.add_argument('--host', help='Get variables for specific host')
    
    args = parser.parse_args()
    
    if args.list:
        print(json.dumps(get_inventory(), indent=2))
    elif args.host:
        print(json.dumps(get_host_vars(args.host), indent=2))
    else:
        parser.print_help()
        sys.exit(1)

if __name__ == '__main__':
    main()
EOF

chmod +x scripts/simple-dynamic-inventory.py
```

Create dynamic inventory test playbook:

```yaml
# Create dynamic-inventory-test.yml
cat > dynamic-inventory-test.yml << 'EOF'
---
- name: Dynamic inventory demonstration
  hosts: all
  gather_facts: no
  tasks:
    - name: Display dynamic inventory information
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Groups: {{ group_names | join(', ') }}
          Ansible Host: {{ ansible_host }}
          Server Role: {{ server_role }}
          Cloud Provider: {{ cloud_provider }}
          Instance Type: {{ instance_type }}
          Discovered At: {{ discovered_at }}
          Discovery Method: {{ discovery_method | default('unknown') }}
    
    - name: Compare with static inventory
      debug:
        msg: |
          Dynamic Inventory Advantages:
          - Automatic host discovery
          - Real-time infrastructure sync
          - Cloud provider integration
          - Scalable for large environments
          - External system integration
          - Reduced manual maintenance
      run_once: true
    
    - name: Show inventory metadata
      debug:
        msg: |
          Inventory Metadata:
          - Inventory Type: {{ inventory_type | default('static') }}
          - Total Hosts: {{ groups['all'] | length }}
          - Azure Instances: {{ groups['azure_instances'] | length }}
          - Managed By: {{ managed_by | default('manual') }}
      run_once: true
EOF
```

### Run dynamic inventory test:
```bash
# Test dynamic inventory script
./scripts/simple-dynamic-inventory.py --list
./scripts/simple-dynamic-inventory.py --host dynamic-web01

# Run playbook with dynamic inventory
ansible-playbook -i scripts/simple-dynamic-inventory.py dynamic-inventory-test.yml

# Compare inventory outputs
echo "=== Static Inventory ==="
ansible-inventory -i static-inventory/hosts.ini --list --yaml

echo "=== Dynamic Inventory ==="
ansible-inventory -i scripts/simple-dynamic-inventory.py --list --yaml
```

## Exercise 3: Inventory Comparison and Migration Strategy (5 minutes)

### Task: Analyze differences and create migration strategy

Create comparison analysis:

```yaml
# Create inventory-comparison.yml
cat > inventory-comparison.yml << 'EOF'
---
- name: Inventory comparison and migration analysis
  hosts: localhost
  gather_facts: no
  vars:
    static_inventory_path: "static-inventory/hosts.ini"
    dynamic_inventory_script: "scripts/simple-dynamic-inventory.py"
    
  tasks:
    - name: Analyze static inventory characteristics
      debug:
        msg: |
          Static Inventory Characteristics:
          ✓ Simple to understand and maintain
          ✓ Version controlled with code
          ✓ Predictable and stable
          ✓ No external dependencies
          
          ✗ Manual host management
          ✗ No automatic discovery
          ✗ Difficult to scale
          ✗ No cloud integration
          ✗ Prone to configuration drift
    
    - name: Analyze dynamic inventory characteristics  
      debug:
        msg: |
          Dynamic Inventory Characteristics:
          ✓ Automatic host discovery
          ✓ Real-time infrastructure sync
          ✓ Cloud provider integration
          ✓ Scalable for large environments
          ✓ External system integration
          ✓ Reduced manual maintenance
          
          ✗ More complex to debug
          ✗ External dependencies
          ✗ Potential performance impact
          ✗ Requires error handling
    
    - name: Migration strategy recommendations
      debug:
        msg: |
          Migration Strategy:
          
          Phase 1: Assessment
          - Audit current static inventory
          - Identify dynamic requirements
          - Choose appropriate inventory sources
          
          Phase 2: Hybrid Approach
          - Keep static inventory for stable hosts
          - Add dynamic inventory for cloud resources
          - Use multiple inventory sources
          
          Phase 3: Full Dynamic
          - Migrate all hosts to dynamic sources
          - Implement comprehensive error handling
          - Add monitoring and alerting
          
          Best Practices:
          - Start with non-critical environments
          - Implement proper caching
          - Add fallback mechanisms
          - Monitor inventory performance
          - Document inventory sources
    
    - name: Create migration checklist
      copy:
        content: |
          # Dynamic Inventory Migration Checklist
          
          ## Pre-Migration Assessment
          - [ ] Document current static inventory structure
          - [ ] Identify all host groups and variables
          - [ ] Map hosts to their data sources (cloud, CMDB, etc.)
          - [ ] Assess network connectivity requirements
          - [ ] Review security and authentication needs
          
          ## Development Phase
          - [ ] Choose appropriate inventory plugins/scripts
          - [ ] Implement basic dynamic inventory
          - [ ] Add error handling and fallbacks
          - [ ] Implement caching mechanisms
          - [ ] Create comprehensive testing strategy
          
          ## Testing Phase
          - [ ] Test with development environment
          - [ ] Validate all host variables are preserved
          - [ ] Test performance under load
          - [ ] Verify error handling scenarios
          - [ ] Test inventory refresh mechanisms
          
          ## Production Migration
          - [ ] Plan rollback strategy
          - [ ] Implement monitoring and alerting
          - [ ] Execute phased rollout
          - [ ] Monitor performance and reliability
          - [ ] Document operational procedures
          
          ## Post-Migration
          - [ ] Remove obsolete static inventory files
          - [ ] Update documentation and runbooks
          - [ ] Train team on new inventory management
          - [ ] Establish maintenance procedures
          - [ ] Review and optimize performance
          
          Generated: {{ ansible_date_time.iso8601 }}
        dest: "/tmp/dynamic-inventory-migration-checklist.md"
        mode: '0644'
    
    - name: Generate inventory comparison report
      copy:
        content: |
          # Inventory Comparison Report
          Generated: {{ ansible_date_time.iso8601 }}
          
          ## Static Inventory Analysis
          - Configuration files: INI format
          - Host management: Manual
          - Variable management: group_vars/host_vars
          - Scalability: Limited
          - Maintenance: High effort
          
          ## Dynamic Inventory Analysis  
          - Configuration: Script/plugin based
          - Host management: Automatic
          - Variable management: Programmatic
          - Scalability: High
          - Maintenance: Low effort (after setup)
          
          ## Recommendations
          
          ### Immediate Actions
          1. Evaluate current inventory size and complexity
          2. Identify cloud/external resources
          3. Assess team technical capabilities
          4. Plan pilot implementation
          
          ### Long-term Strategy
          1. Implement hybrid approach initially
          2. Gradually migrate to full dynamic inventory
          3. Establish monitoring and maintenance procedures
          4. Train team on dynamic inventory concepts
          
          ### Success Metrics
          - Reduced manual inventory maintenance time
          - Improved infrastructure synchronization
          - Faster new host onboarding
          - Reduced configuration errors
          - Better scalability for growth
        dest: "/tmp/inventory-comparison-report.md"
        mode: '0644'
EOF
```

### Run comparison analysis:
```bash
# Run inventory comparison
ansible-playbook inventory-comparison.yml

# Check generated reports
cat /tmp/dynamic-inventory-migration-checklist.md
cat /tmp/inventory-comparison-report.md
```

## Verification and Discussion

### 1. Check Results
```bash
# Compare inventory outputs
echo "=== Static Inventory Structure ==="
ansible-inventory -i static-inventory/hosts.ini --graph

echo "=== Dynamic Inventory Structure ==="
ansible-inventory -i scripts/simple-dynamic-inventory.py --graph

# Check generated documentation
ls -la /tmp/*inventory*.md
```

### 2. Discussion Points
- What are the main challenges with static inventory in your current environment?
- How would dynamic inventory benefit your Azure/cloud infrastructure management?
- What external systems could serve as inventory sources in your organization?
- What concerns do you have about migrating to dynamic inventory?

### 3. Clean Up
```bash
# Remove demo files (keep for next labs)
# rm -rf static-inventory dynamic-inventory scripts data
# rm -f *.yml /tmp/*inventory*.md
```

## Key Takeaways
- Static inventory is simple but doesn't scale well for dynamic environments
- Dynamic inventory provides automatic discovery and synchronization
- Migration should be gradual with proper testing and fallback mechanisms
- Hybrid approaches can provide the best of both worlds during transition
- Proper error handling and caching are essential for dynamic inventory
- Documentation and team training are crucial for successful adoption

## Next Steps
Proceed to Lab 4.2: Custom Inventory Scripts Development

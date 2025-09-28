# Lab 4.4: Multiple Inventory Sources and Merging

## Objective
Combine multiple inventory sources and handle complex enterprise topologies with proper merging strategies.

## Duration
25 minutes

## Prerequisites
- Completed Labs 4.1, 4.2, and 4.3
- Understanding of inventory merging concepts

## Lab Setup

```bash
cd ~/ansible-labs/module-04
mkdir -p lab-4.4
cd lab-4.4

# Create directory structure
mkdir -p inventories/{static,dynamic,cloud,cmdb} configs scripts
```

## Exercise 1: Multiple Static Inventory Sources (8 minutes)

### Task: Create and merge multiple static inventory sources

Create different static inventory sources:

```bash
# Create production inventory
cat > inventories/static/production.ini << 'EOF'
# Production Environment Inventory
[web_prod]
web-prod-01 ansible_host=10.1.1.10 ansible_user=ubuntu
web-prod-02 ansible_host=10.1.1.11 ansible_user=ubuntu
web-prod-03 ansible_host=10.1.1.12 ansible_user=ubuntu

[db_prod]
db-prod-01 ansible_host=10.1.2.10 ansible_user=postgres
db-prod-02 ansible_host=10.1.2.11 ansible_user=postgres

[lb_prod]
lb-prod-01 ansible_host=10.1.3.10 ansible_user=nginx

[production:children]
web_prod
db_prod
lb_prod

[production:vars]
environment=production
backup_enabled=true
monitoring_level=detailed
ssl_required=true
EOF

# Create staging inventory
cat > inventories/static/staging.ini << 'EOF'
# Staging Environment Inventory
[web_staging]
web-stage-01 ansible_host=10.2.1.10 ansible_user=ubuntu
web-stage-02 ansible_host=10.2.1.11 ansible_user=ubuntu

[db_staging]
db-stage-01 ansible_host=10.2.2.10 ansible_user=postgres

[staging:children]
web_staging
db_staging

[staging:vars]
environment=staging
backup_enabled=false
monitoring_level=basic
ssl_required=false
EOF

# Create development inventory
cat > inventories/static/development.ini << 'EOF'
# Development Environment Inventory
[web_dev]
web-dev-01 ansible_host=10.3.1.10 ansible_user=ubuntu

[db_dev]
db-dev-01 ansible_host=10.3.2.10 ansible_user=postgres

[development:children]
web_dev
db_dev

[development:vars]
environment=development
backup_enabled=false
monitoring_level=minimal
ssl_required=false
debug_mode=true
EOF

# Create network infrastructure inventory
cat > inventories/static/network.ini << 'EOF'
# Network Infrastructure Inventory
[firewalls]
fw-01 ansible_host=10.0.1.1 ansible_user=admin ansible_network_os=fortios
fw-02 ansible_host=10.0.1.2 ansible_user=admin ansible_network_os=fortios

[switches]
sw-core-01 ansible_host=10.0.2.1 ansible_user=admin ansible_network_os=eos
sw-core-02 ansible_host=10.0.2.2 ansible_user=admin ansible_network_os=eos
sw-access-01 ansible_host=10.0.3.1 ansible_user=admin ansible_network_os=eos

[routers]
rtr-01 ansible_host=10.0.4.1 ansible_user=admin ansible_network_os=ios

[network:children]
firewalls
switches
routers

[network:vars]
ansible_connection=network_cli
backup_config=true
config_backup_dir=/backup/network
EOF
```

Create inventory directory configuration:

```bash
# Create ansible.cfg for multiple sources
cat > ansible.cfg << 'EOF'
[defaults]
inventory = inventories/
host_key_checking = False
timeout = 30

[inventory]
enable_plugins = auto, yaml, ini, script, advanced_host_list
cache = True
cache_plugin = memory
cache_timeout = 300
EOF
```

Test multiple static inventories:

```bash
# Test combined static inventories
ansible-inventory --list --yaml
ansible-inventory --graph

# Test specific inventory sources
ansible-inventory -i inventories/static/production.ini --graph
ansible-inventory -i inventories/static/staging.ini --graph
ansible-inventory -i inventories/static/network.ini --graph

# Create test playbook
cat > test-multiple-static.yml << 'EOF'
---
- name: Test multiple static inventory sources
  hosts: all
  gather_facts: no
  tasks:
    - name: Display host information from multiple sources
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Environment: {{ environment | default('unknown') }}
          Groups: {{ group_names | join(', ') }}
          Ansible Host: {{ ansible_host | default('localhost') }}
          Backup Enabled: {{ backup_enabled | default('unknown') }}
          Monitoring Level: {{ monitoring_level | default('unknown') }}
          {% if ansible_network_os is defined %}
          Network OS: {{ ansible_network_os }}
          {% endif %}
    
    - name: Environment summary
      debug:
        msg: |
          Inventory Summary:
          - Total hosts: {{ groups['all'] | length }}
          - Production: {{ groups['production'] | default([]) | length }}
          - Staging: {{ groups['staging'] | default([]) | length }}
          - Development: {{ groups['development'] | default([]) | length }}
          - Network devices: {{ groups['network'] | default([]) | length }}
      run_once: true
EOF

ansible-playbook test-multiple-static.yml
```

## Exercise 2: Hybrid Static and Dynamic Sources (10 minutes)

### Task: Combine static inventories with dynamic sources

Create cloud inventory simulation:

```python
# Create inventories/dynamic/cloud-inventory.py
cat > inventories/dynamic/cloud-inventory.py << 'EOF'
#!/usr/bin/env python3
"""
Cloud Inventory Script for Multiple Providers
Simulates Azure, AWS, and GCP resources
"""

import json
import sys
import argparse
from datetime import datetime

def get_cloud_inventory():
    """Generate multi-cloud inventory"""
    
    inventory = {
        "_meta": {
            "hostvars": {}
        },
        "cloud_instances": {
            "hosts": [],
            "vars": {
                "managed_by": "cloud_automation",
                "auto_scaling": True
            }
        },
        "azure_vms": {
            "hosts": [],
            "vars": {
                "cloud_provider": "azure",
                "region": "West Europe"
            }
        },
        "aws_instances": {
            "hosts": [],
            "vars": {
                "cloud_provider": "aws",
                "region": "eu-west-1"
            }
        },
        "gcp_instances": {
            "hosts": [],
            "vars": {
                "cloud_provider": "gcp",
                "region": "europe-west1"
            }
        }
    }
    
    # Azure VMs
    azure_vms = [
        {
            "name": "azure-web-prod-01",
            "ip": "20.50.1.10",
            "private_ip": "10.10.1.10",
            "size": "Standard_B2s",
            "environment": "production"
        },
        {
            "name": "azure-web-prod-02", 
            "ip": "20.50.1.11",
            "private_ip": "10.10.1.11",
            "size": "Standard_B2s",
            "environment": "production"
        },
        {
            "name": "azure-db-prod-01",
            "ip": None,
            "private_ip": "10.10.2.10",
            "size": "Standard_D4s_v3",
            "environment": "production"
        }
    ]
    
    # AWS Instances
    aws_instances = [
        {
            "name": "aws-web-prod-01",
            "ip": "54.123.45.67",
            "private_ip": "172.31.1.10",
            "type": "t3.medium",
            "environment": "production"
        },
        {
            "name": "aws-cache-prod-01",
            "ip": None,
            "private_ip": "172.31.2.10",
            "type": "r5.large",
            "environment": "production"
        }
    ]
    
    # GCP Instances
    gcp_instances = [
        {
            "name": "gcp-analytics-prod-01",
            "ip": "34.76.123.45",
            "private_ip": "10.20.1.10",
            "type": "n1-standard-4",
            "environment": "production"
        }
    ]
    
    # Process Azure VMs
    for vm in azure_vms:
        hostname = vm["name"]
        inventory["cloud_instances"]["hosts"].append(hostname)
        inventory["azure_vms"]["hosts"].append(hostname)
        
        inventory["_meta"]["hostvars"][hostname] = {
            "ansible_host": vm["ip"] or vm["private_ip"],
            "ansible_user": "azureuser",
            "cloud_provider": "azure",
            "azure_vm_size": vm["size"],
            "azure_public_ip": vm["ip"],
            "azure_private_ip": vm["private_ip"],
            "environment": vm["environment"],
            "discovered_at": datetime.now().isoformat(),
            "inventory_source": "cloud_dynamic"
        }
    
    # Process AWS Instances
    for instance in aws_instances:
        hostname = instance["name"]
        inventory["cloud_instances"]["hosts"].append(hostname)
        inventory["aws_instances"]["hosts"].append(hostname)
        
        inventory["_meta"]["hostvars"][hostname] = {
            "ansible_host": instance["ip"] or instance["private_ip"],
            "ansible_user": "ec2-user",
            "cloud_provider": "aws",
            "aws_instance_type": instance["type"],
            "aws_public_ip": instance["ip"],
            "aws_private_ip": instance["private_ip"],
            "environment": instance["environment"],
            "discovered_at": datetime.now().isoformat(),
            "inventory_source": "cloud_dynamic"
        }
    
    # Process GCP Instances
    for instance in gcp_instances:
        hostname = instance["name"]
        inventory["cloud_instances"]["hosts"].append(hostname)
        inventory["gcp_instances"]["hosts"].append(hostname)
        
        inventory["_meta"]["hostvars"][hostname] = {
            "ansible_host": instance["ip"] or instance["private_ip"],
            "ansible_user": "gcp-user",
            "cloud_provider": "gcp",
            "gcp_machine_type": instance["type"],
            "gcp_external_ip": instance["ip"],
            "gcp_internal_ip": instance["private_ip"],
            "environment": instance["environment"],
            "discovered_at": datetime.now().isoformat(),
            "inventory_source": "cloud_dynamic"
        }
    
    return inventory

def get_host_vars(hostname):
    """Get variables for specific host"""
    inventory = get_cloud_inventory()
    return inventory.get("_meta", {}).get("hostvars", {}).get(hostname, {})

def main():
    parser = argparse.ArgumentParser(description='Multi-Cloud Dynamic Inventory')
    parser.add_argument('--list', action='store_true', help='List all hosts')
    parser.add_argument('--host', help='Get variables for specific host')
    
    args = parser.parse_args()
    
    if args.list:
        print(json.dumps(get_cloud_inventory(), indent=2))
    elif args.host:
        print(json.dumps(get_host_vars(args.host), indent=2))
    else:
        parser.print_help()
        sys.exit(1)

if __name__ == '__main__':
    main()
EOF

chmod +x inventories/dynamic/cloud-inventory.py
```

Create CMDB inventory script:

```python
# Create inventories/cmdb/cmdb-inventory.py
cat > inventories/cmdb/cmdb-inventory.py << 'EOF'
#!/usr/bin/env python3
"""
CMDB Inventory Script
Provides enterprise asset information
"""

import json
import sys
import argparse
from datetime import datetime

def get_cmdb_inventory():
    """Generate CMDB-based inventory"""
    
    inventory = {
        "_meta": {
            "hostvars": {}
        },
        "cmdb_managed": {
            "hosts": [],
            "vars": {
                "inventory_source": "cmdb",
                "asset_tracking": True
            }
        },
        "physical_servers": {
            "hosts": [],
            "vars": {
                "server_type": "physical",
                "warranty_tracking": True
            }
        },
        "virtual_servers": {
            "hosts": [],
            "vars": {
                "server_type": "virtual",
                "hypervisor_managed": True
            }
        }
    }
    
    # CMDB servers
    cmdb_servers = [
        {
            "hostname": "cmdb-web-prod-01",
            "ip": "192.168.1.100",
            "type": "physical",
            "location": "datacenter-1",
            "rack": "R01-U10",
            "asset_tag": "AT-WEB-100",
            "environment": "production",
            "application": "web-frontend",
            "owner": "web-team"
        },
        {
            "hostname": "cmdb-db-prod-01",
            "ip": "192.168.1.200",
            "type": "physical",
            "location": "datacenter-1",
            "rack": "R02-U15",
            "asset_tag": "AT-DB-200",
            "environment": "production",
            "application": "database",
            "owner": "dba-team"
        },
        {
            "hostname": "cmdb-app-stage-01",
            "ip": "192.168.2.100",
            "type": "virtual",
            "location": "datacenter-2",
            "hypervisor": "vmware-host-01",
            "asset_tag": "AT-VM-100",
            "environment": "staging",
            "application": "application-server",
            "owner": "dev-team"
        },
        {
            "hostname": "cmdb-test-dev-01",
            "ip": "192.168.3.100",
            "type": "virtual",
            "location": "datacenter-2",
            "hypervisor": "vmware-host-02",
            "asset_tag": "AT-VM-101",
            "environment": "development",
            "application": "test-environment",
            "owner": "qa-team"
        }
    ]
    
    # Process CMDB servers
    for server in cmdb_servers:
        hostname = server["hostname"]
        inventory["cmdb_managed"]["hosts"].append(hostname)
        
        if server["type"] == "physical":
            inventory["physical_servers"]["hosts"].append(hostname)
        else:
            inventory["virtual_servers"]["hosts"].append(hostname)
        
        host_vars = {
            "ansible_host": server["ip"],
            "ansible_user": "ubuntu",
            "cmdb_hostname": server["hostname"],
            "cmdb_server_type": server["type"],
            "cmdb_location": server["location"],
            "cmdb_asset_tag": server["asset_tag"],
            "cmdb_environment": server["environment"],
            "cmdb_application": server["application"],
            "cmdb_owner": server["owner"],
            "discovered_at": datetime.now().isoformat(),
            "inventory_source": "cmdb"
        }
        
        if server["type"] == "physical":
            host_vars["cmdb_rack"] = server["rack"]
        else:
            host_vars["cmdb_hypervisor"] = server["hypervisor"]
        
        inventory["_meta"]["hostvars"][hostname] = host_vars
    
    return inventory

def get_host_vars(hostname):
    """Get variables for specific host"""
    inventory = get_cmdb_inventory()
    return inventory.get("_meta", {}).get("hostvars", {}).get(hostname, {})

def main():
    parser = argparse.ArgumentParser(description='CMDB Dynamic Inventory')
    parser.add_argument('--list', action='store_true', help='List all hosts')
    parser.add_argument('--host', help='Get variables for specific host')
    
    args = parser.parse_args()
    
    if args.list:
        print(json.dumps(get_cmdb_inventory(), indent=2))
    elif args.host:
        print(json.dumps(get_host_vars(args.host), indent=2))
    else:
        parser.print_help()
        sys.exit(1)

if __name__ == '__main__':
    main()
EOF

chmod +x inventories/cmdb/cmdb-inventory.py
```

Test hybrid inventory sources:

```bash
# Test individual sources
echo "=== Static Production ==="
ansible-inventory -i inventories/static/production.ini --graph

echo "=== Cloud Dynamic ==="
ansible-inventory -i inventories/dynamic/cloud-inventory.py --graph

echo "=== CMDB Dynamic ==="
ansible-inventory -i inventories/cmdb/cmdb-inventory.py --graph

echo "=== Combined Hybrid ==="
ansible-inventory --graph

# Create hybrid test playbook
cat > test-hybrid-inventory.yml << 'EOF'
---
- name: Test hybrid inventory sources
  hosts: all
  gather_facts: no
  tasks:
    - name: Display host information from hybrid sources
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Inventory Source: {{ inventory_source | default('static') }}
          Environment: {{ environment | default(cmdb_environment | default('unknown')) }}
          Groups: {{ group_names | join(', ') }}
          
          {% if inventory_source == 'cloud_dynamic' %}
          Cloud Provider: {{ cloud_provider | default('unknown') }}
          {% if cloud_provider == 'azure' %}
          Azure VM Size: {{ azure_vm_size | default('unknown') }}
          {% elif cloud_provider == 'aws' %}
          AWS Instance Type: {{ aws_instance_type | default('unknown') }}
          {% elif cloud_provider == 'gcp' %}
          GCP Machine Type: {{ gcp_machine_type | default('unknown') }}
          {% endif %}
          {% endif %}
          
          {% if inventory_source == 'cmdb' %}
          CMDB Details:
          - Server Type: {{ cmdb_server_type | default('unknown') }}
          - Location: {{ cmdb_location | default('unknown') }}
          - Asset Tag: {{ cmdb_asset_tag | default('unknown') }}
          - Owner: {{ cmdb_owner | default('unknown') }}
          {% endif %}
    
    - name: Hybrid inventory summary
      debug:
        msg: |
          Hybrid Inventory Summary:
          - Total hosts: {{ groups['all'] | length }}
          - Static hosts: {{ groups['all'] | select('match', '^(web|db|lb|fw|sw|rtr)-.*') | list | length }}
          - Cloud instances: {{ groups['cloud_instances'] | default([]) | length }}
          - CMDB managed: {{ groups['cmdb_managed'] | default([]) | length }}
          - Production environment: {{ groups['production'] | default([]) | length }}
      run_once: true
EOF

ansible-playbook test-hybrid-inventory.yml
```

## Exercise 3: Advanced Inventory Merging and Conflict Resolution (7 minutes)

### Task: Handle inventory conflicts and implement merging strategies

Create inventory merging configuration:

```yaml
# Create configs/inventory-merging.yml
cat > configs/inventory-merging.yml << 'EOF'
# Inventory Merging Configuration
plugin: constructed
strict: False

# Variable precedence and merging
compose:
    # Merge environment information with precedence
    final_environment: |
        environment if environment is defined else
        cmdb_environment if cmdb_environment is defined else
        'unknown'
    
    # Merge host connection information
    final_ansible_host: |
        ansible_host if ansible_host is defined else
        azure_public_ip if azure_public_ip is defined else
        aws_public_ip if aws_public_ip is defined else
        gcp_external_ip if gcp_external_ip is defined else
        'localhost'
    
    # Determine primary data source
    primary_source: |
        'cmdb' if inventory_source == 'cmdb' else
        'cloud' if inventory_source == 'cloud_dynamic' else
        'static'
    
    # Merge backup configuration
    backup_required: |
        backup_enabled if backup_enabled is defined else
        (final_environment == 'production')
    
    # Determine monitoring level
    monitoring_required: |
        monitoring_level if monitoring_level is defined else
        ('detailed' if final_environment == 'production' else 'basic')
    
    # Calculate host priority for conflict resolution
    host_priority: |
        10 if primary_source == 'cmdb' else
        8 if primary_source == 'cloud' else
        5
    
    # Merge owner information
    responsible_team: |
        cmdb_owner if cmdb_owner is defined else
        owner if owner is defined else
        'ops-team'

# Create merged groups
groups:
    # Environment-based groups (merged)
    all_production: final_environment == 'production'
    all_staging: final_environment == 'staging'
    all_development: final_environment == 'development'
    
    # Source-based groups
    cmdb_authoritative: primary_source == 'cmdb'
    cloud_managed: primary_source == 'cloud'
    static_defined: primary_source == 'static'
    
    # Service-based groups (merged)
    web_services: |
        'web' in group_names or
        'web' in (cmdb_application | default('')) or
        inventory_hostname | regex_search('web')
    
    database_services: |
        'db' in group_names or
        'database' in (cmdb_application | default('')) or
        inventory_hostname | regex_search('db')
    
    # Priority-based groups
    high_priority_hosts: host_priority | int >= 8
    medium_priority_hosts: host_priority | int >= 5 and host_priority | int < 8
    low_priority_hosts: host_priority | int < 5
    
    # Backup requirement groups
    backup_required_hosts: backup_required | bool
    backup_optional_hosts: not (backup_required | bool)

keyed_groups:
    # Create groups by merged attributes
    - key: final_environment + '_' + primary_source
      separator: ""
    - key: responsible_team | replace('-', '_')
      prefix: team
      separator: "_"
    - key: monitoring_required
      prefix: monitoring
      separator: "_"
EOF
```

Create conflict resolution test:

```bash
# Create conflicting inventory data
cat > inventories/static/conflicts.ini << 'EOF'
# Conflicting Static Inventory
[conflict_test]
# This host exists in multiple sources with different data
azure-web-prod-01 ansible_host=192.168.100.10 environment=staging owner=static-team

[conflict_test:vars]
backup_enabled=false
monitoring_level=none
source_priority=low
EOF
```

Create conflict analysis playbook:

```yaml
# Create conflict-analysis.yml
cat > conflict-analysis.yml << 'EOF'
---
- name: Inventory conflict analysis
  hosts: all
  gather_facts: no
  tasks:
    - name: Identify potential conflicts
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          
          Conflict Analysis:
          - Primary Source: {{ primary_source | default('unknown') }}
          - Host Priority: {{ host_priority | default(0) }}
          
          Environment Resolution:
          - Static Environment: {{ environment | default('not_set') }}
          - CMDB Environment: {{ cmdb_environment | default('not_set') }}
          - Final Environment: {{ final_environment | default('unknown') }}
          
          Connection Resolution:
          - Static ansible_host: {{ ansible_host | default('not_set') }}
          - Azure Public IP: {{ azure_public_ip | default('not_set') }}
          - Final ansible_host: {{ final_ansible_host | default('unknown') }}
          
          Ownership Resolution:
          - Static Owner: {{ owner | default('not_set') }}
          - CMDB Owner: {{ cmdb_owner | default('not_set') }}
          - Responsible Team: {{ responsible_team | default('unknown') }}
          
          Configuration Resolution:
          - Backup Required: {{ backup_required | default('unknown') }}
          - Monitoring Required: {{ monitoring_required | default('unknown') }}
      when: inventory_hostname in groups['all']
    
    - name: Conflict resolution summary
      debug:
        msg: |
          Conflict Resolution Summary:
          
          Source Priority Distribution:
          - CMDB Authoritative: {{ groups['cmdb_authoritative'] | default([]) | length }}
          - Cloud Managed: {{ groups['cloud_managed'] | default([]) | length }}
          - Static Defined: {{ groups['static_defined'] | default([]) | length }}
          
          Priority Distribution:
          - High Priority: {{ groups['high_priority_hosts'] | default([]) | length }}
          - Medium Priority: {{ groups['medium_priority_hosts'] | default([]) | length }}
          - Low Priority: {{ groups['low_priority_hosts'] | default([]) | length }}
          
          Merged Environment Distribution:
          - Production: {{ groups['all_production'] | default([]) | length }}
          - Staging: {{ groups['all_staging'] | default([]) | length }}
          - Development: {{ groups['all_development'] | default([]) | length }}
      run_once: true
    
    - name: Identify hosts with conflicts
      debug:
        msg: |
          Potential Conflicts Detected:
          {% set conflicts = [] %}
          {% for host in groups['all'] %}
          {% set h = hostvars[host] %}
          {% if h.environment is defined and h.cmdb_environment is defined and h.environment != h.cmdb_environment %}
          {% set _ = conflicts.append(host + ': environment mismatch') %}
          {% endif %}
          {% if h.ansible_host is defined and h.azure_public_ip is defined and h.ansible_host != h.azure_public_ip %}
          {% set _ = conflicts.append(host + ': IP address mismatch') %}
          {% endif %}
          {% endfor %}
          
          {% if conflicts %}
          {% for conflict in conflicts %}
          - {{ conflict }}
          {% endfor %}
          {% else %}
          No conflicts detected
          {% endif %}
      run_once: true
EOF
```

Test inventory merging and conflict resolution:

```bash
# Test with merging configuration
ansible-inventory -i inventories/ -i configs/inventory-merging.yml --list --yaml
ansible-inventory -i inventories/ -i configs/inventory-merging.yml --graph

# Run conflict analysis
ansible-playbook -i inventories/ -i configs/inventory-merging.yml conflict-analysis.yml

# Test specific conflict scenarios
echo "=== Testing Conflict Resolution ==="
ansible-inventory -i inventories/ -i configs/inventory-merging.yml --host azure-web-prod-01
```

Create inventory validation script:

```bash
# Create scripts/validate-inventory.py
cat > scripts/validate-inventory.py << 'EOF'
#!/usr/bin/env python3
"""
Inventory Validation Script
Validates merged inventory for conflicts and issues
"""

import json
import subprocess
import sys
from collections import defaultdict

def get_inventory():
    """Get merged inventory from ansible-inventory"""
    try:
        result = subprocess.run([
            'ansible-inventory', 
            '-i', 'inventories/',
            '-i', 'configs/inventory-merging.yml',
            '--list'
        ], capture_output=True, text=True, check=True)
        return json.loads(result.stdout)
    except subprocess.CalledProcessError as e:
        print(f"Error getting inventory: {e}", file=sys.stderr)
        return {}
    except json.JSONDecodeError as e:
        print(f"Error parsing inventory JSON: {e}", file=sys.stderr)
        return {}

def validate_inventory(inventory):
    """Validate inventory for common issues"""
    issues = []
    warnings = []
    
    hostvars = inventory.get('_meta', {}).get('hostvars', {})
    
    # Check for duplicate IP addresses
    ip_hosts = defaultdict(list)
    for host, vars in hostvars.items():
        ip = vars.get('final_ansible_host') or vars.get('ansible_host')
        if ip and ip != 'localhost':
            ip_hosts[ip].append(host)
    
    for ip, hosts in ip_hosts.items():
        if len(hosts) > 1:
            issues.append(f"Duplicate IP address {ip}: {', '.join(hosts)}")
    
    # Check for missing required variables
    for host, vars in hostvars.items():
        if not vars.get('final_ansible_host') and not vars.get('ansible_host'):
            issues.append(f"Host {host} missing connection information")
        
        if not vars.get('final_environment'):
            warnings.append(f"Host {host} missing environment classification")
    
    # Check for conflicting data
    for host, vars in hostvars.items():
        if (vars.get('environment') and vars.get('cmdb_environment') and 
            vars['environment'] != vars['cmdb_environment']):
            warnings.append(f"Host {host} has conflicting environment data")
    
    return issues, warnings

def main():
    print("Validating merged inventory...")
    
    inventory = get_inventory()
    if not inventory:
        print("Failed to get inventory")
        sys.exit(1)
    
    issues, warnings = validate_inventory(inventory)
    
    print(f"\nValidation Results:")
    print(f"- Total hosts: {len(inventory.get('_meta', {}).get('hostvars', {}))}")
    print(f"- Issues found: {len(issues)}")
    print(f"- Warnings: {len(warnings)}")
    
    if issues:
        print("\nISSUES (must be resolved):")
        for issue in issues:
            print(f"  ❌ {issue}")
    
    if warnings:
        print("\nWARNINGS (should be reviewed):")
        for warning in warnings:
            print(f"  ⚠️  {warning}")
    
    if not issues and not warnings:
        print("\n✅ Inventory validation passed!")
    
    return len(issues)

if __name__ == '__main__':
    sys.exit(main())
EOF

chmod +x scripts/validate-inventory.py
./scripts/validate-inventory.py
```

## Verification and Discussion

### 1. Check Results
```bash
# Compare different inventory combinations
echo "=== Static Only ==="
ansible-inventory -i inventories/static/ --graph

echo "=== Dynamic Only ==="
ansible-inventory -i inventories/dynamic/ -i inventories/cmdb/ --graph

echo "=== All Sources Combined ==="
ansible-inventory -i inventories/ --graph

echo "=== With Merging Configuration ==="
ansible-inventory -i inventories/ -i configs/inventory-merging.yml --graph

# Run validation
./scripts/validate-inventory.py
```

### 2. Discussion Points
- How do you handle inventory conflicts in your current environment?
- What strategies work best for maintaining inventory accuracy across multiple sources?
- How do you implement inventory validation in your CI/CD pipelines?
- What are the performance implications of multiple inventory sources?

### 3. Clean Up
```bash
# Keep files for next lab
# rm -rf inventories configs scripts
# rm -f *.yml *.py ansible.cfg
```

## Key Takeaways
- Multiple inventory sources provide comprehensive infrastructure coverage
- Proper merging strategies prevent conflicts and data inconsistencies
- Variable precedence rules ensure predictable conflict resolution
- Validation is crucial for maintaining inventory integrity
- Performance considerations are important with multiple sources
- Documentation of source priorities helps with troubleshooting

## Next Steps
Proceed to Lab 4.5: Advanced Variables and Group Management

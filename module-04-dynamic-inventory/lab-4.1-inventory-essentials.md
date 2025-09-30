# Lab 4.1: Dynamic Inventory Essentials (Interactive Demo)

## Objective
Master dynamic inventory concepts through interactive demonstration and practical examples suitable for all skill levels.

## Duration
25 minutes (interactive demo)

## Format
**Interactive demonstration** - Instructor-led with participant practice

## Prerequisites
- Completed previous modules
- Understanding of basic inventory concepts

## Demo Setup

```bash
cd ~/ansible-labs
mkdir -p module-04
cd module-04

# Create simple inventory for demo
cat > inventory.ini << 'EOF'
[demo_hosts]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Demo 1: Static vs Dynamic Inventory (8 minutes)

### What is Dynamic Inventory?
**Instructor explains:**
- Static inventory: Fixed files with predefined hosts
- Dynamic inventory: Generated at runtime from external sources
- Benefits: Scalability, accuracy, automation integration
- Use cases: Cloud environments, containers, service discovery

### Static Inventory Example
```bash
# Create traditional static inventory
cat > static-hosts.ini << 'EOF'
# Traditional Static Inventory
[webservers]
web01 ansible_host=192.168.1.10
web02 ansible_host=192.168.1.11
web03 ansible_host=192.168.1.12

[databases]
db01 ansible_host=192.168.1.20
db02 ansible_host=192.168.1.21

[production:children]
webservers
databases

[production:vars]
environment=production
backup_enabled=true
EOF

# Test static inventory
ansible-inventory -i static-hosts.ini --list
```

### Simple Dynamic Inventory Script
```bash
# Create basic dynamic inventory script
cat > dynamic-inventory.py << 'EOF'
#!/usr/bin/env python3
import json
import sys

def get_inventory():
    """Generate dynamic inventory"""
    inventory = {
        'webservers': {
            'hosts': ['web01', 'web02', 'web03'],
            'vars': {
                'http_port': 80,
                'max_connections': 1000
            }
        },
        'databases': {
            'hosts': ['db01', 'db02'],
            'vars': {
                'db_port': 5432,
                'max_connections': 200
            }
        },
        'production': {
            'children': ['webservers', 'databases'],
            'vars': {
                'environment': 'production',
                'backup_enabled': True
            }
        },
        '_meta': {
            'hostvars': {
                'web01': {'ansible_host': '192.168.1.10', 'role': 'frontend'},
                'web02': {'ansible_host': '192.168.1.11', 'role': 'frontend'},
                'web03': {'ansible_host': '192.168.1.12', 'role': 'frontend'},
                'db01': {'ansible_host': '192.168.1.20', 'role': 'primary'},
                'db02': {'ansible_host': '192.168.1.21', 'role': 'replica'}
            }
        }
    }
    return inventory

if __name__ == '__main__':
    if len(sys.argv) == 2 and sys.argv[1] == '--list':
        print(json.dumps(get_inventory(), indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == '--host':
        # Return empty dict for host-specific vars (using _meta instead)
        print(json.dumps({}))
    else:
        print("Usage: %s --list or %s --host <hostname>" % (sys.argv[0], sys.argv[0]))
        sys.exit(1)
EOF

# Make executable and test
chmod +x dynamic-inventory.py
./dynamic-inventory.py --list
ansible-inventory -i dynamic-inventory.py --list
```

## Demo 2: Real-World Dynamic Sources (8 minutes)

### CSV-Based Dynamic Inventory
```bash
# Create CSV data source
cat > servers.csv << 'EOF'
hostname,ip_address,role,environment,location
web01,192.168.1.10,webserver,production,datacenter1
web02,192.168.1.11,webserver,production,datacenter1
web03,192.168.1.12,webserver,staging,datacenter2
db01,192.168.1.20,database,production,datacenter1
db02,192.168.1.21,database,staging,datacenter2
cache01,192.168.1.30,cache,production,datacenter1
EOF

# Create CSV-based dynamic inventory
cat > csv-inventory.py << 'EOF'
#!/usr/bin/env python3
import json
import csv
import sys

def load_csv_inventory(csv_file):
    """Load inventory from CSV file"""
    inventory = {'_meta': {'hostvars': {}}}
    
    with open(csv_file, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            hostname = row['hostname']
            role = row['role']
            environment = row['environment']
            location = row['location']
            
            # Add to role group
            if role not in inventory:
                inventory[role] = {'hosts': [], 'vars': {}}
            inventory[role]['hosts'].append(hostname)
            
            # Add to environment group
            if environment not in inventory:
                inventory[environment] = {'hosts': [], 'vars': {}}
            inventory[environment]['hosts'].append(hostname)
            
            # Add to location group
            location_group = f"location_{location}"
            if location_group not in inventory:
                inventory[location_group] = {'hosts': [], 'vars': {}}
            inventory[location_group]['hosts'].append(hostname)
            
            # Add host variables
            inventory['_meta']['hostvars'][hostname] = {
                'ansible_host': row['ip_address'],
                'server_role': role,
                'environment': environment,
                'location': location
            }
    
    return inventory

if __name__ == '__main__':
    if len(sys.argv) == 2 and sys.argv[1] == '--list':
        inventory = load_csv_inventory('servers.csv')
        print(json.dumps(inventory, indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == '--host':
        print(json.dumps({}))
    else:
        print("Usage: %s --list" % sys.argv[0])
        sys.exit(1)
EOF

# Test CSV inventory
chmod +x csv-inventory.py
./csv-inventory.py --list
ansible-inventory -i csv-inventory.py --graph
```

### API-Based Dynamic Inventory (Simulation)
```bash
# Create mock API data
cat > api-servers.json << 'EOF'
{
  "servers": [
    {
      "name": "web-api-01",
      "ip": "10.0.1.10",
      "tags": ["webserver", "production", "frontend"],
      "metadata": {
        "cpu_cores": 4,
        "memory_gb": 8,
        "disk_gb": 100
      }
    },
    {
      "name": "web-api-02", 
      "ip": "10.0.1.11",
      "tags": ["webserver", "production", "frontend"],
      "metadata": {
        "cpu_cores": 4,
        "memory_gb": 8,
        "disk_gb": 100
      }
    },
    {
      "name": "db-api-01",
      "ip": "10.0.2.10", 
      "tags": ["database", "production", "backend"],
      "metadata": {
        "cpu_cores": 8,
        "memory_gb": 32,
        "disk_gb": 500
      }
    }
  ]
}
EOF

# Create API-based inventory script
cat > api-inventory.py << 'EOF'
#!/usr/bin/env python3
import json
import sys

def load_api_inventory():
    """Simulate loading from API"""
    # In real scenario, this would be: requests.get('https://api.company.com/servers')
    with open('api-servers.json', 'r') as f:
        api_data = json.load(f)
    
    inventory = {'_meta': {'hostvars': {}}}
    
    for server in api_data['servers']:
        hostname = server['name']
        
        # Create groups from tags
        for tag in server['tags']:
            if tag not in inventory:
                inventory[tag] = {'hosts': [], 'vars': {}}
            inventory[tag]['hosts'].append(hostname)
        
        # Add host variables
        inventory['_meta']['hostvars'][hostname] = {
            'ansible_host': server['ip'],
            'server_tags': server['tags'],
            'cpu_cores': server['metadata']['cpu_cores'],
            'memory_gb': server['metadata']['memory_gb'],
            'disk_gb': server['metadata']['disk_gb']
        }
    
    return inventory

if __name__ == '__main__':
    if len(sys.argv) == 2 and sys.argv[1] == '--list':
        inventory = load_api_inventory()
        print(json.dumps(inventory, indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == '--host':
        print(json.dumps({}))
    else:
        print("Usage: %s --list" % sys.argv[0])
        sys.exit(1)
EOF

# Test API inventory
chmod +x api-inventory.py
./api-inventory.py --list
ansible-inventory -i api-inventory.py --graph
```

## Demo 3: Using Dynamic Inventory in Playbooks (9 minutes)

### Playbook with Dynamic Groups
```yaml
# Create playbook using dynamic inventory
cat > dynamic-playbook.yml << 'EOF'
---
- name: Deploy to Web Servers (Dynamic)
  hosts: webserver
  gather_facts: no
  
  tasks:
    - name: Show server information
      debug:
        msg: |
          Server: {{ inventory_hostname }}
          IP: {{ ansible_host }}
          Role: {{ server_role | default('unknown') }}
          Environment: {{ environment | default('unknown') }}
          Location: {{ location | default('unknown') }}
    
    - name: Configure web servers by environment
      debug:
        msg: "Configuring {{ inventory_hostname }} for {{ environment }} environment"
      when: environment is defined

- name: Deploy to Production Only
  hosts: production
  gather_facts: no
  
  tasks:
    - name: Production-specific tasks
      debug:
        msg: "Running production deployment on {{ inventory_hostname }}"
      when: environment == "production"

- name: Location-based tasks
  hosts: location_datacenter1
  gather_facts: no
  
  tasks:
    - name: Datacenter1-specific configuration
      debug:
        msg: "Configuring {{ inventory_hostname }} for datacenter1"
EOF

# Run with different inventories
echo "=== Running with CSV inventory ==="
ansible-playbook -i csv-inventory.py dynamic-playbook.yml

echo "=== Running with API inventory ==="
ansible-playbook -i api-inventory.py dynamic-playbook.yml
```

### Multiple Inventory Sources
```bash
# Create inventory directory with multiple sources
mkdir -p inventory-sources

# Copy different inventory sources
cp static-hosts.ini inventory-sources/
cp csv-inventory.py inventory-sources/
cp servers.csv inventory-sources/
cp api-inventory.py inventory-sources/
cp api-servers.json inventory-sources/

# Test combined inventory
ansible-inventory -i inventory-sources/ --list
ansible-inventory -i inventory-sources/ --graph
```

## Interactive Practice Session

### Hands-On Exercise for Participants
```yaml
# Create practice exercise
cat > practice-dynamic.yml << 'EOF'
---
- name: Participant Practice - Dynamic Inventory
  hosts: localhost
  gather_facts: no
  
  tasks:
    # TODO: Participants will create their own dynamic inventory
    # Requirements:
    # 1. Create a JSON file with server data
    # 2. Write a Python script to read the JSON
    # 3. Generate proper Ansible inventory format
    # 4. Test with ansible-inventory command
    
    - name: Example task
      debug:
        msg: "Create your dynamic inventory script here"
        
    # Hints:
    # - Use JSON format for data storage
    # - Include groups: web, database, cache
    # - Add host variables: ansible_host, role, environment
    # - Test with: ansible-inventory -i your-script.py --list
EOF
```

**Practice Instructions:**
1. Create a JSON file with 5-6 servers
2. Include different roles (web, database, cache)
3. Add environment tags (production, staging)
4. Write Python script to generate inventory
5. Test with ansible-inventory command

## Key Takeaways Summary

### Dynamic Inventory Benefits:
1. **Scalability** - Handles large, changing infrastructures
2. **Accuracy** - Always up-to-date server information
3. **Automation** - Integrates with CI/CD and orchestration
4. **Flexibility** - Multiple data sources and formats
5. **Consistency** - Single source of truth

### Common Data Sources:
- **Cloud APIs** - AWS, Azure, GCP
- **CMDB Systems** - ServiceNow, Remedy
- **Container Orchestrators** - Kubernetes, Docker Swarm
- **Service Discovery** - Consul, etcd
- **Databases** - MySQL, PostgreSQL
- **Files** - CSV, JSON, YAML

### Dynamic Inventory Script Requirements:
- Accept `--list` parameter (return all inventory)
- Accept `--host <hostname>` parameter (return host vars)
- Output valid JSON format
- Include `_meta` section for performance
- Be executable and handle errors gracefully

### Best Practices:
- Cache expensive API calls
- Handle network timeouts and errors
- Use `_meta` section for better performance
- Validate data before generating inventory
- Log errors appropriately
- Make scripts configurable


## Common Use Cases

### Cloud Environments:
```bash
# AWS EC2 inventory (example)
ansible-inventory -i aws_ec2.yml --list

# Azure inventory (example) 
ansible-inventory -i azure_rm.yml --list
```

### Container Environments:
```bash
# Docker inventory (example)
ansible-inventory -i docker.yml --list

# Kubernetes inventory (example)
ansible-inventory -i k8s.yml --list
```

### Hybrid Environments:
```bash
# Multiple sources
ansible-inventory -i inventory/ --list
```

## Demo Cleanup
```bash
# Clean up demo files
rm -f *.ini *.py *.csv *.json *.yml
rm -rf inventory-sources/
```

---

**Note for Instructor**: Focus on practical benefits and real-world scenarios. Emphasize how dynamic inventory solves scalability and accuracy problems. Adjust complexity based on participant experience with cloud and automation tools.

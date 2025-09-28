# Lab 4.2: Custom Inventory Scripts Development

## Objective
Create custom inventory scripts for specific enterprise requirements including Azure integration and CMDB connectivity.

## Duration
30 minutes

## Prerequisites
- Completed Lab 4.1
- Basic Python knowledge
- Understanding of REST APIs and JSON

## Lab Setup

```bash
cd ~/ansible-labs/module-04
mkdir -p lab-4.2
cd lab-4.2

# Create directory structure
mkdir -p scripts data templates config
```

## Exercise 1: Azure-Inspired Inventory Script (12 minutes)

### Task: Create an inventory script that simulates Azure resource discovery

Create Azure simulation data:

```bash
# Create simulated Azure data
cat > data/azure-resources.json << 'EOF'
{
  "subscriptions": [
    {
      "id": "12345678-1234-1234-1234-123456789012",
      "name": "Production Subscription",
      "resource_groups": [
        {
          "name": "rg-web-prod",
          "location": "West Europe",
          "virtual_machines": [
            {
              "name": "vm-web-prod-01",
              "resource_id": "/subscriptions/12345678-1234-1234-1234-123456789012/resourceGroups/rg-web-prod/providers/Microsoft.Compute/virtualMachines/vm-web-prod-01",
              "location": "West Europe",
              "size": "Standard_B2s",
              "os_type": "Linux",
              "private_ip": "10.0.1.10",
              "public_ip": "20.50.1.10",
              "tags": {
                "Environment": "Production",
                "Application": "WebFrontend",
                "Owner": "DevOps Team",
                "CostCenter": "IT-001"
              },
              "status": "running"
            },
            {
              "name": "vm-web-prod-02",
              "resource_id": "/subscriptions/12345678-1234-1234-1234-123456789012/resourceGroups/rg-web-prod/providers/Microsoft.Compute/virtualMachines/vm-web-prod-02",
              "location": "West Europe",
              "size": "Standard_B2s",
              "os_type": "Linux",
              "private_ip": "10.0.1.11",
              "public_ip": "20.50.1.11",
              "tags": {
                "Environment": "Production",
                "Application": "WebFrontend",
                "Owner": "DevOps Team",
                "CostCenter": "IT-001"
              },
              "status": "running"
            }
          ]
        },
        {
          "name": "rg-db-prod",
          "location": "West Europe",
          "virtual_machines": [
            {
              "name": "vm-db-prod-01",
              "resource_id": "/subscriptions/12345678-1234-1234-1234-123456789012/resourceGroups/rg-db-prod/providers/Microsoft.Compute/virtualMachines/vm-db-prod-01",
              "location": "West Europe",
              "size": "Standard_D4s_v3",
              "os_type": "Linux",
              "private_ip": "10.0.2.10",
              "public_ip": null,
              "tags": {
                "Environment": "Production",
                "Application": "Database",
                "Owner": "DBA Team",
                "CostCenter": "IT-002"
              },
              "status": "running"
            }
          ]
        }
      ]
    },
    {
      "id": "87654321-4321-4321-4321-210987654321",
      "name": "Staging Subscription",
      "resource_groups": [
        {
          "name": "rg-web-stage",
          "location": "North Europe",
          "virtual_machines": [
            {
              "name": "vm-web-stage-01",
              "resource_id": "/subscriptions/87654321-4321-4321-4321-210987654321/resourceGroups/rg-web-stage/providers/Microsoft.Compute/virtualMachines/vm-web-stage-01",
              "location": "North Europe",
              "size": "Standard_B1s",
              "os_type": "Linux",
              "private_ip": "10.1.1.10",
              "public_ip": "20.51.1.10",
              "tags": {
                "Environment": "Staging",
                "Application": "WebFrontend",
                "Owner": "DevOps Team",
                "CostCenter": "IT-001"
              },
              "status": "running"
            }
          ]
        }
      ]
    }
  ]
}
EOF
```

Create Azure inventory script:

```python
# Create scripts/azure-inventory.py
cat > scripts/azure-inventory.py << 'EOF'
#!/usr/bin/env python3
"""
Azure Dynamic Inventory Script
Simulates Azure resource discovery for Ansible
"""

import json
import sys
import argparse
import os
from datetime import datetime

class AzureInventory:
    def __init__(self):
        self.inventory = {
            "_meta": {
                "hostvars": {}
            }
        }
        self.data_file = "data/azure-resources.json"
        
    def load_azure_data(self):
        """Load simulated Azure data"""
        try:
            with open(self.data_file, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            print(f"Error: {self.data_file} not found", file=sys.stderr)
            return {"subscriptions": []}
        except json.JSONDecodeError:
            print(f"Error: Invalid JSON in {self.data_file}", file=sys.stderr)
            return {"subscriptions": []}
    
    def generate_inventory(self):
        """Generate Ansible inventory from Azure data"""
        azure_data = self.load_azure_data()
        
        # Initialize group structures
        groups = {
            "azure_vms": {"hosts": [], "vars": {"cloud_provider": "azure"}},
            "production": {"hosts": [], "vars": {"environment": "production"}},
            "staging": {"hosts": [], "vars": {"environment": "staging"}},
            "web_servers": {"hosts": [], "vars": {"server_type": "web"}},
            "database_servers": {"hosts": [], "vars": {"server_type": "database"}},
            "west_europe": {"hosts": [], "vars": {"azure_region": "West Europe"}},
            "north_europe": {"hosts": [], "vars": {"azure_region": "North Europe"}},
            "standard_b_series": {"hosts": [], "vars": {"vm_series": "B"}},
            "standard_d_series": {"hosts": [], "vars": {"vm_series": "D"}},
        }
        
        # Process each subscription
        for subscription in azure_data.get("subscriptions", []):
            subscription_id = subscription["id"]
            subscription_name = subscription["name"]
            
            # Create subscription group
            sub_group_name = f"subscription_{subscription_name.lower().replace(' ', '_')}"
            groups[sub_group_name] = {
                "hosts": [],
                "vars": {
                    "azure_subscription_id": subscription_id,
                    "azure_subscription_name": subscription_name
                }
            }
            
            # Process resource groups
            for rg in subscription.get("resource_groups", []):
                rg_name = rg["name"]
                rg_location = rg["location"]
                
                # Create resource group
                rg_group_name = f"rg_{rg_name.replace('-', '_')}"
                groups[rg_group_name] = {
                    "hosts": [],
                    "vars": {
                        "azure_resource_group": rg_name,
                        "azure_location": rg_location
                    }
                }
                
                # Process virtual machines
                for vm in rg.get("virtual_machines", []):
                    vm_name = vm["name"]
                    
                    # Add to various groups based on properties
                    groups["azure_vms"]["hosts"].append(vm_name)
                    groups[sub_group_name]["hosts"].append(vm_name)
                    groups[rg_group_name]["hosts"].append(vm_name)
                    
                    # Environment-based grouping
                    env = vm["tags"].get("Environment", "").lower()
                    if env in groups:
                        groups[env]["hosts"].append(vm_name)
                    
                    # Application-based grouping
                    app = vm["tags"].get("Application", "").lower()
                    if "web" in app:
                        groups["web_servers"]["hosts"].append(vm_name)
                    elif "database" in app:
                        groups["database_servers"]["hosts"].append(vm_name)
                    
                    # Location-based grouping
                    location = vm["location"]
                    if location in groups:
                        groups[location.lower().replace(" ", "_")]["hosts"].append(vm_name)
                    
                    # VM size-based grouping
                    vm_size = vm["size"]
                    if "Standard_B" in vm_size:
                        groups["standard_b_series"]["hosts"].append(vm_name)
                    elif "Standard_D" in vm_size:
                        groups["standard_d_series"]["hosts"].append(vm_name)
                    
                    # Create host variables
                    self.inventory["_meta"]["hostvars"][vm_name] = {
                        "ansible_host": vm.get("public_ip") or vm.get("private_ip"),
                        "ansible_user": "azureuser",
                        "azure_resource_id": vm["resource_id"],
                        "azure_vm_name": vm_name,
                        "azure_resource_group": rg_name,
                        "azure_location": vm["location"],
                        "azure_vm_size": vm["size"],
                        "azure_os_type": vm["os_type"],
                        "azure_private_ip": vm["private_ip"],
                        "azure_public_ip": vm.get("public_ip"),
                        "azure_status": vm["status"],
                        "azure_subscription_id": subscription_id,
                        "azure_subscription_name": subscription_name,
                        "environment": vm["tags"].get("Environment", "unknown"),
                        "application": vm["tags"].get("Application", "unknown"),
                        "owner": vm["tags"].get("Owner", "unknown"),
                        "cost_center": vm["tags"].get("CostCenter", "unknown"),
                        "discovered_at": datetime.now().isoformat(),
                        "inventory_source": "azure_simulation"
                    }
        
        # Add groups to inventory
        for group_name, group_data in groups.items():
            if group_data["hosts"]:  # Only add groups with hosts
                self.inventory[group_name] = group_data
        
        return self.inventory
    
    def get_host_vars(self, hostname):
        """Get variables for specific host"""
        inventory = self.generate_inventory()
        return inventory.get("_meta", {}).get("hostvars", {}).get(hostname, {})

def main():
    parser = argparse.ArgumentParser(description='Azure Dynamic Inventory')
    parser.add_argument('--list', action='store_true', help='List all hosts')
    parser.add_argument('--host', help='Get variables for specific host')
    
    args = parser.parse_args()
    
    azure_inventory = AzureInventory()
    
    if args.list:
        print(json.dumps(azure_inventory.generate_inventory(), indent=2))
    elif args.host:
        print(json.dumps(azure_inventory.get_host_vars(args.host), indent=2))
    else:
        parser.print_help()
        sys.exit(1)

if __name__ == '__main__':
    main()
EOF

chmod +x scripts/azure-inventory.py
```

Test Azure inventory script:

```bash
# Test the Azure inventory script
./scripts/azure-inventory.py --list | python3 -m json.tool
./scripts/azure-inventory.py --host vm-web-prod-01

# Test with ansible-inventory
ansible-inventory -i scripts/azure-inventory.py --list --yaml
ansible-inventory -i scripts/azure-inventory.py --graph
```

## Exercise 2: CMDB Integration Script (10 minutes)

### Task: Create an inventory script that integrates with a Configuration Management Database

Create CMDB simulation data:

```bash
# Create simulated CMDB data
cat > data/cmdb-data.json << 'EOF'
{
  "servers": [
    {
      "hostname": "srv-web-001",
      "fqdn": "srv-web-001.company.local",
      "ip_address": "192.168.10.10",
      "os": "Ubuntu 20.04",
      "environment": "production",
      "application": "web-frontend",
      "owner": "web-team@company.com",
      "location": "datacenter-1",
      "rack": "R01-U15",
      "serial_number": "SN123456789",
      "asset_tag": "AT-WEB-001",
      "cpu_cores": 4,
      "memory_gb": 16,
      "disk_gb": 500,
      "network_interfaces": [
        {"name": "eth0", "ip": "192.168.10.10", "mac": "00:50:56:12:34:56"}
      ],
      "services": ["nginx", "nodejs"],
      "backup_enabled": true,
      "monitoring_enabled": true,
      "maintenance_window": "Sunday 02:00-04:00",
      "last_updated": "2024-01-15T10:30:00Z"
    },
    {
      "hostname": "srv-web-002",
      "fqdn": "srv-web-002.company.local",
      "ip_address": "192.168.10.11",
      "os": "Ubuntu 20.04",
      "environment": "production",
      "application": "web-frontend",
      "owner": "web-team@company.com",
      "location": "datacenter-1",
      "rack": "R01-U16",
      "serial_number": "SN123456790",
      "asset_tag": "AT-WEB-002",
      "cpu_cores": 4,
      "memory_gb": 16,
      "disk_gb": 500,
      "network_interfaces": [
        {"name": "eth0", "ip": "192.168.10.11", "mac": "00:50:56:12:34:57"}
      ],
      "services": ["nginx", "nodejs"],
      "backup_enabled": true,
      "monitoring_enabled": true,
      "maintenance_window": "Sunday 02:00-04:00",
      "last_updated": "2024-01-15T10:30:00Z"
    },
    {
      "hostname": "srv-db-001",
      "fqdn": "srv-db-001.company.local",
      "ip_address": "192.168.20.10",
      "os": "Ubuntu 20.04",
      "environment": "production",
      "application": "database",
      "owner": "dba-team@company.com",
      "location": "datacenter-2",
      "rack": "R02-U10",
      "serial_number": "SN123456791",
      "asset_tag": "AT-DB-001",
      "cpu_cores": 8,
      "memory_gb": 64,
      "disk_gb": 2000,
      "network_interfaces": [
        {"name": "eth0", "ip": "192.168.20.10", "mac": "00:50:56:12:34:58"}
      ],
      "services": ["postgresql"],
      "backup_enabled": true,
      "monitoring_enabled": true,
      "maintenance_window": "Sunday 01:00-03:00",
      "last_updated": "2024-01-15T10:30:00Z"
    },
    {
      "hostname": "srv-test-001",
      "fqdn": "srv-test-001.company.local",
      "ip_address": "192.168.30.10",
      "os": "Ubuntu 20.04",
      "environment": "testing",
      "application": "web-frontend",
      "owner": "qa-team@company.com",
      "location": "datacenter-1",
      "rack": "R03-U05",
      "serial_number": "SN123456792",
      "asset_tag": "AT-TEST-001",
      "cpu_cores": 2,
      "memory_gb": 8,
      "disk_gb": 200,
      "network_interfaces": [
        {"name": "eth0", "ip": "192.168.30.10", "mac": "00:50:56:12:34:59"}
      ],
      "services": ["nginx", "nodejs"],
      "backup_enabled": false,
      "monitoring_enabled": true,
      "maintenance_window": "Any time",
      "last_updated": "2024-01-15T10:30:00Z"
    }
  ]
}
EOF
```

Create CMDB inventory script:

```python
# Create scripts/cmdb-inventory.py
cat > scripts/cmdb-inventory.py << 'EOF'
#!/usr/bin/env python3
"""
CMDB Dynamic Inventory Script
Integrates with Configuration Management Database
"""

import json
import sys
import argparse
import os
from datetime import datetime

class CMDBInventory:
    def __init__(self):
        self.inventory = {
            "_meta": {
                "hostvars": {}
            }
        }
        self.data_file = "data/cmdb-data.json"
        
    def load_cmdb_data(self):
        """Load CMDB data"""
        try:
            with open(self.data_file, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            print(f"Error: {self.data_file} not found", file=sys.stderr)
            return {"servers": []}
        except json.JSONDecodeError:
            print(f"Error: Invalid JSON in {self.data_file}", file=sys.stderr)
            return {"servers": []}
    
    def generate_inventory(self):
        """Generate Ansible inventory from CMDB data"""
        cmdb_data = self.load_cmdb_data()
        
        # Initialize group structures
        groups = {
            "all_servers": {"hosts": [], "vars": {"inventory_source": "cmdb"}},
            "production": {"hosts": [], "vars": {"environment": "production"}},
            "testing": {"hosts": [], "vars": {"environment": "testing"}},
            "web_servers": {"hosts": [], "vars": {"server_type": "web"}},
            "database_servers": {"hosts": [], "vars": {"server_type": "database"}},
            "datacenter_1": {"hosts": [], "vars": {"location": "datacenter-1"}},
            "datacenter_2": {"hosts": [], "vars": {"location": "datacenter-2"}},
            "ubuntu_servers": {"hosts": [], "vars": {"os_family": "ubuntu"}},
            "backup_enabled": {"hosts": [], "vars": {"backup_policy": "enabled"}},
            "high_memory": {"hosts": [], "vars": {"memory_category": "high"}},
        }
        
        # Process each server
        for server in cmdb_data.get("servers", []):
            hostname = server["hostname"]
            
            # Add to basic groups
            groups["all_servers"]["hosts"].append(hostname)
            
            # Environment-based grouping
            env = server.get("environment", "unknown")
            if env in groups:
                groups[env]["hosts"].append(hostname)
            
            # Application-based grouping
            app = server.get("application", "unknown")
            if "web" in app:
                groups["web_servers"]["hosts"].append(hostname)
            elif "database" in app:
                groups["database_servers"]["hosts"].append(hostname)
            
            # Location-based grouping
            location = server.get("location", "unknown")
            location_group = location.replace("-", "_")
            if location_group in groups:
                groups[location_group]["hosts"].append(hostname)
            
            # OS-based grouping
            os_info = server.get("os", "").lower()
            if "ubuntu" in os_info:
                groups["ubuntu_servers"]["hosts"].append(hostname)
            
            # Feature-based grouping
            if server.get("backup_enabled", False):
                groups["backup_enabled"]["hosts"].append(hostname)
            
            if server.get("memory_gb", 0) >= 32:
                groups["high_memory"]["hosts"].append(hostname)
            
            # Owner-based grouping
            owner = server.get("owner", "").replace("@company.com", "").replace("-", "_")
            owner_group = f"team_{owner}"
            if owner_group not in groups:
                groups[owner_group] = {"hosts": [], "vars": {"team": owner}}
            groups[owner_group]["hosts"].append(hostname)
            
            # Create comprehensive host variables
            self.inventory["_meta"]["hostvars"][hostname] = {
                "ansible_host": server["ip_address"],
                "ansible_user": "ubuntu",
                "cmdb_hostname": server["hostname"],
                "cmdb_fqdn": server["fqdn"],
                "cmdb_ip_address": server["ip_address"],
                "cmdb_os": server["os"],
                "cmdb_environment": server["environment"],
                "cmdb_application": server["application"],
                "cmdb_owner": server["owner"],
                "cmdb_location": server["location"],
                "cmdb_rack": server["rack"],
                "cmdb_serial_number": server["serial_number"],
                "cmdb_asset_tag": server["asset_tag"],
                "cmdb_cpu_cores": server["cpu_cores"],
                "cmdb_memory_gb": server["memory_gb"],
                "cmdb_disk_gb": server["disk_gb"],
                "cmdb_services": server["services"],
                "cmdb_backup_enabled": server["backup_enabled"],
                "cmdb_monitoring_enabled": server["monitoring_enabled"],
                "cmdb_maintenance_window": server["maintenance_window"],
                "cmdb_last_updated": server["last_updated"],
                "cmdb_network_interfaces": server["network_interfaces"],
                "discovered_at": datetime.now().isoformat(),
                "inventory_source": "cmdb"
            }
        
        # Add groups to inventory (only non-empty groups)
        for group_name, group_data in groups.items():
            if group_data["hosts"]:
                self.inventory[group_name] = group_data
        
        return self.inventory
    
    def get_host_vars(self, hostname):
        """Get variables for specific host"""
        inventory = self.generate_inventory()
        return inventory.get("_meta", {}).get("hostvars", {}).get(hostname, {})

def main():
    parser = argparse.ArgumentParser(description='CMDB Dynamic Inventory')
    parser.add_argument('--list', action='store_true', help='List all hosts')
    parser.add_argument('--host', help='Get variables for specific host')
    
    args = parser.parse_args()
    
    cmdb_inventory = CMDBInventory()
    
    if args.list:
        print(json.dumps(cmdb_inventory.generate_inventory(), indent=2))
    elif args.host:
        print(json.dumps(cmdb_inventory.get_host_vars(args.host), indent=2))
    else:
        parser.print_help()
        sys.exit(1)

if __name__ == '__main__':
    main()
EOF

chmod +x scripts/cmdb-inventory.py
```

Test CMDB inventory script:

```bash
# Test the CMDB inventory script
./scripts/cmdb-inventory.py --list | python3 -m json.tool
./scripts/cmdb-inventory.py --host srv-web-001

# Test with ansible-inventory
ansible-inventory -i scripts/cmdb-inventory.py --list --yaml
ansible-inventory -i scripts/cmdb-inventory.py --graph
```

## Exercise 3: Advanced Inventory Script with Caching (8 minutes)

### Task: Create an advanced inventory script with caching and error handling

Create advanced inventory script:

```python
# Create scripts/advanced-inventory.py
cat > scripts/advanced-inventory.py << 'EOF'
#!/usr/bin/env python3
"""
Advanced Dynamic Inventory Script
Features: Caching, Error Handling, Multiple Sources
"""

import json
import sys
import argparse
import os
import time
import hashlib
from datetime import datetime, timedelta

class AdvancedInventory:
    def __init__(self):
        self.inventory = {
            "_meta": {
                "hostvars": {}
            }
        }
        self.cache_dir = "/tmp/ansible-inventory-cache"
        self.cache_ttl = 300  # 5 minutes
        self.sources = [
            {"type": "azure", "file": "data/azure-resources.json"},
            {"type": "cmdb", "file": "data/cmdb-data.json"}
        ]
        
        # Ensure cache directory exists
        os.makedirs(self.cache_dir, exist_ok=True)
    
    def get_cache_file(self, source_type):
        """Get cache file path for source type"""
        return os.path.join(self.cache_dir, f"{source_type}-inventory.json")
    
    def is_cache_valid(self, cache_file):
        """Check if cache file is valid and not expired"""
        if not os.path.exists(cache_file):
            return False
        
        cache_age = time.time() - os.path.getmtime(cache_file)
        return cache_age < self.cache_ttl
    
    def load_from_cache(self, source_type):
        """Load inventory from cache"""
        cache_file = self.get_cache_file(source_type)
        
        if self.is_cache_valid(cache_file):
            try:
                with open(cache_file, 'r') as f:
                    data = json.load(f)
                    print(f"Loaded {source_type} from cache", file=sys.stderr)
                    return data
            except (FileNotFoundError, json.JSONDecodeError):
                pass
        
        return None
    
    def save_to_cache(self, source_type, data):
        """Save inventory to cache"""
        cache_file = self.get_cache_file(source_type)
        
        try:
            with open(cache_file, 'w') as f:
                json.dump(data, f, indent=2)
            print(f"Saved {source_type} to cache", file=sys.stderr)
        except Exception as e:
            print(f"Failed to save {source_type} to cache: {e}", file=sys.stderr)
    
    def load_source_data(self, source):
        """Load data from source with caching"""
        source_type = source["type"]
        source_file = source["file"]
        
        # Try cache first
        cached_data = self.load_from_cache(source_type)
        if cached_data:
            return cached_data
        
        # Load from source file
        try:
            with open(source_file, 'r') as f:
                data = json.load(f)
                self.save_to_cache(source_type, data)
                print(f"Loaded {source_type} from source file", file=sys.stderr)
                return data
        except FileNotFoundError:
            print(f"Warning: {source_file} not found", file=sys.stderr)
            return {}
        except json.JSONDecodeError:
            print(f"Error: Invalid JSON in {source_file}", file=sys.stderr)
            return {}
    
    def process_azure_data(self, azure_data):
        """Process Azure data into inventory format"""
        hosts = {}
        groups = {
            "azure": {"hosts": [], "vars": {"cloud_provider": "azure"}}
        }
        
        for subscription in azure_data.get("subscriptions", []):
            for rg in subscription.get("resource_groups", []):
                for vm in rg.get("virtual_machines", []):
                    hostname = vm["name"]
                    hosts[hostname] = {
                        "ansible_host": vm.get("public_ip") or vm.get("private_ip"),
                        "ansible_user": "azureuser",
                        "source": "azure",
                        "azure_vm_size": vm["size"],
                        "azure_location": vm["location"],
                        "environment": vm["tags"].get("Environment", "unknown").lower(),
                        "application": vm["tags"].get("Application", "unknown").lower()
                    }
                    groups["azure"]["hosts"].append(hostname)
        
        return hosts, groups
    
    def process_cmdb_data(self, cmdb_data):
        """Process CMDB data into inventory format"""
        hosts = {}
        groups = {
            "cmdb": {"hosts": [], "vars": {"inventory_source": "cmdb"}}
        }
        
        for server in cmdb_data.get("servers", []):
            hostname = server["hostname"]
            hosts[hostname] = {
                "ansible_host": server["ip_address"],
                "ansible_user": "ubuntu",
                "source": "cmdb",
                "cmdb_location": server["location"],
                "cmdb_owner": server["owner"],
                "environment": server["environment"],
                "application": server["application"],
                "cmdb_cpu_cores": server["cpu_cores"],
                "cmdb_memory_gb": server["memory_gb"]
            }
            groups["cmdb"]["hosts"].append(hostname)
        
        return hosts, groups
    
    def merge_inventories(self):
        """Merge inventories from multiple sources"""
        all_hosts = {}
        all_groups = {
            "production": {"hosts": [], "vars": {"environment": "production"}},
            "staging": {"hosts": [], "vars": {"environment": "staging"}},
            "testing": {"hosts": [], "vars": {"environment": "testing"}},
            "web_applications": {"hosts": [], "vars": {"app_type": "web"}},
            "database_applications": {"hosts": [], "vars": {"app_type": "database"}},
        }
        
        # Process each source
        for source in self.sources:
            source_type = source["type"]
            source_data = self.load_source_data(source)
            
            if source_type == "azure":
                hosts, groups = self.process_azure_data(source_data)
            elif source_type == "cmdb":
                hosts, groups = self.process_cmdb_data(source_data)
            else:
                continue
            
            # Merge hosts
            for hostname, host_vars in hosts.items():
                if hostname in all_hosts:
                    # Merge variables from multiple sources
                    all_hosts[hostname].update(host_vars)
                    all_hosts[hostname]["sources"] = all_hosts[hostname].get("sources", []) + [source_type]
                else:
                    host_vars["sources"] = [source_type]
                    all_hosts[hostname] = host_vars
            
            # Merge groups
            for group_name, group_data in groups.items():
                if group_name not in all_groups:
                    all_groups[group_name] = group_data
                else:
                    all_groups[group_name]["hosts"].extend(group_data["hosts"])
        
        # Organize hosts into logical groups
        for hostname, host_vars in all_hosts.items():
            env = host_vars.get("environment", "unknown")
            if env in all_groups:
                all_groups[env]["hosts"].append(hostname)
            
            app = host_vars.get("application", "unknown")
            if "web" in app:
                all_groups["web_applications"]["hosts"].append(hostname)
            elif "database" in app:
                all_groups["database_applications"]["hosts"].append(hostname)
        
        # Build final inventory
        self.inventory["_meta"]["hostvars"] = all_hosts
        
        # Add non-empty groups
        for group_name, group_data in all_groups.items():
            if group_data["hosts"]:
                # Remove duplicates
                group_data["hosts"] = list(set(group_data["hosts"]))
                self.inventory[group_name] = group_data
        
        # Add metadata
        self.inventory["_meta"]["generated_at"] = datetime.now().isoformat()
        self.inventory["_meta"]["cache_ttl"] = self.cache_ttl
        self.inventory["_meta"]["sources"] = [s["type"] for s in self.sources]
        
        return self.inventory
    
    def get_host_vars(self, hostname):
        """Get variables for specific host"""
        inventory = self.merge_inventories()
        return inventory.get("_meta", {}).get("hostvars", {}).get(hostname, {})
    
    def cleanup_cache(self):
        """Clean up expired cache files"""
        try:
            for filename in os.listdir(self.cache_dir):
                if filename.endswith("-inventory.json"):
                    filepath = os.path.join(self.cache_dir, filename)
                    if not self.is_cache_valid(filepath):
                        os.remove(filepath)
                        print(f"Removed expired cache file: {filename}", file=sys.stderr)
        except Exception as e:
            print(f"Cache cleanup failed: {e}", file=sys.stderr)

def main():
    parser = argparse.ArgumentParser(description='Advanced Dynamic Inventory')
    parser.add_argument('--list', action='store_true', help='List all hosts')
    parser.add_argument('--host', help='Get variables for specific host')
    parser.add_argument('--refresh-cache', action='store_true', help='Refresh cache')
    
    args = parser.parse_args()
    
    inventory = AdvancedInventory()
    
    if args.refresh_cache:
        inventory.cleanup_cache()
        print("Cache refreshed", file=sys.stderr)
        return
    
    if args.list:
        print(json.dumps(inventory.merge_inventories(), indent=2))
    elif args.host:
        print(json.dumps(inventory.get_host_vars(args.host), indent=2))
    else:
        parser.print_help()
        sys.exit(1)

if __name__ == '__main__':
    main()
EOF

chmod +x scripts/advanced-inventory.py
```

Test advanced inventory script:

```bash
# Test the advanced inventory script
./scripts/advanced-inventory.py --list | python3 -m json.tool

# Test caching
echo "First run (should load from files):"
time ./scripts/advanced-inventory.py --list > /dev/null

echo "Second run (should load from cache):"
time ./scripts/advanced-inventory.py --list > /dev/null

# Test cache refresh
./scripts/advanced-inventory.py --refresh-cache

# Test with ansible-inventory
ansible-inventory -i scripts/advanced-inventory.py --graph
```

Create test playbook for custom inventory:

```yaml
# Create custom-inventory-test.yml
cat > custom-inventory-test.yml << 'EOF'
---
- name: Test custom inventory scripts
  hosts: all
  gather_facts: no
  tasks:
    - name: Display host information from custom inventory
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Groups: {{ group_names | join(', ') }}
          Ansible Host: {{ ansible_host }}
          Source: {{ source | default('unknown') }}
          Environment: {{ environment | default('unknown') }}
          Application: {{ application | default('unknown') }}
          
          {% if source == 'azure' %}
          Azure Details:
          - VM Size: {{ azure_vm_size | default('unknown') }}
          - Location: {{ azure_location | default('unknown') }}
          {% endif %}
          
          {% if source == 'cmdb' %}
          CMDB Details:
          - CPU Cores: {{ cmdb_cpu_cores | default('unknown') }}
          - Memory GB: {{ cmdb_memory_gb | default('unknown') }}
          - Owner: {{ cmdb_owner | default('unknown') }}
          {% endif %}
          
          {% if sources is defined %}
          Multiple Sources: {{ sources | join(', ') }}
          {% endif %}
    
    - name: Group analysis
      debug:
        msg: |
          Inventory Analysis:
          - Total hosts: {{ groups['all'] | length }}
          - Production hosts: {{ groups['production'] | default([]) | length }}
          - Azure hosts: {{ groups['azure'] | default([]) | length }}
          - CMDB hosts: {{ groups['cmdb'] | default([]) | length }}
      run_once: true
EOF
```

### Run custom inventory tests:
```bash
# Test each inventory script
echo "=== Testing Azure Inventory ==="
ansible-playbook -i scripts/azure-inventory.py custom-inventory-test.yml

echo "=== Testing CMDB Inventory ==="
ansible-playbook -i scripts/cmdb-inventory.py custom-inventory-test.yml

echo "=== Testing Advanced Inventory ==="
ansible-playbook -i scripts/advanced-inventory.py custom-inventory-test.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Compare different inventory sources
echo "=== Azure Inventory Structure ==="
ansible-inventory -i scripts/azure-inventory.py --graph

echo "=== CMDB Inventory Structure ==="
ansible-inventory -i scripts/cmdb-inventory.py --graph

echo "=== Advanced Inventory Structure ==="
ansible-inventory -i scripts/advanced-inventory.py --graph

# Check cache files
ls -la /tmp/ansible-inventory-cache/
```

### 2. Discussion Points
- How would you integrate these patterns with your existing Azure infrastructure?
- What other data sources could be valuable for your inventory scripts?
- How do you handle authentication and security for external data sources?
- What caching strategies would work best in your environment?

### 3. Clean Up
```bash
# Keep files for next lab
# rm -rf scripts data config /tmp/ansible-inventory-cache
# rm -f *.yml
```

## Key Takeaways
- Custom inventory scripts provide flexibility for specific enterprise requirements
- Caching improves performance and reduces load on external systems
- Error handling is crucial for reliable inventory operations
- Multiple data sources can be merged for comprehensive inventory
- Proper grouping and variable organization improves playbook efficiency
- Security and authentication must be considered for production use

## Next Steps
Proceed to Lab 4.3: Inventory Plugins and Cloud Integration

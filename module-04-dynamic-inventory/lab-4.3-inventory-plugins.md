# Lab 4.3: Inventory Plugins and Cloud Integration

## Objective
Implement modern inventory plugins for Azure and multi-cloud scenarios using Ansible's plugin architecture.

## Duration
25 minutes

## Prerequisites
- Completed Labs 4.1 and 4.2
- Understanding of Ansible plugin architecture
- Basic knowledge of YAML configuration

## Lab Setup

```bash
cd ~/ansible-labs/module-04
mkdir -p lab-4.3
cd lab-4.3

# Create directory structure
mkdir -p plugins/inventory configs data
```

## Exercise 1: Custom Inventory Plugin Development (10 minutes)

### Task: Create a custom inventory plugin using Ansible's plugin architecture

Create custom inventory plugin:

```python
# Create plugins/inventory/enterprise_cmdb.py
cat > plugins/inventory/enterprise_cmdb.py << 'EOF'
# Copyright: (c) 2024, Enterprise IT Team
# GNU General Public License v3.0+ (see COPYING or https://www.gnu.org/licenses/gpl-3.0.txt)

from __future__ import (absolute_import, division, print_function)
__metaclass__ = type

DOCUMENTATION = '''
    name: enterprise_cmdb
    plugin_type: inventory
    short_description: Enterprise CMDB inventory plugin
    description:
        - Fetches inventory from Enterprise Configuration Management Database
        - Supports grouping by environment, application, location, and owner
        - Provides comprehensive host variables from CMDB data
    options:
        plugin:
            description: Name of the plugin
            required: true
            choices: ['enterprise_cmdb']
        cmdb_file:
            description: Path to CMDB data file
            required: true
            type: str
        group_by:
            description: List of attributes to group hosts by
            type: list
            default: ['environment', 'application', 'location']
        include_vars:
            description: List of CMDB attributes to include as host variables
            type: list
            default: ['all']
        exclude_environments:
            description: List of environments to exclude
            type: list
            default: []
        cache:
            description: Enable caching
            type: bool
            default: true
        cache_timeout:
            description: Cache timeout in seconds
            type: int
            default: 300
'''

EXAMPLES = '''
# Basic usage
plugin: enterprise_cmdb
cmdb_file: /path/to/cmdb-data.json

# Advanced configuration
plugin: enterprise_cmdb
cmdb_file: /path/to/cmdb-data.json
group_by:
  - environment
  - application
  - owner
include_vars:
  - hostname
  - ip_address
  - cpu_cores
  - memory_gb
exclude_environments:
  - decommissioned
  - maintenance
cache: true
cache_timeout: 600
'''

import json
import os
from ansible.plugins.inventory import BaseInventoryPlugin, Constructable, Cacheable
from ansible.errors import AnsibleError, AnsibleParserError

class InventoryModule(BaseInventoryPlugin, Constructable, Cacheable):
    NAME = 'enterprise_cmdb'

    def verify_file(self, path):
        """Verify that the source file can be processed by this plugin"""
        return (
            super(InventoryModule, self).verify_file(path) and
            path.endswith(('cmdb.yml', 'cmdb.yaml'))
        )

    def parse(self, inventory, loader, path, cache=True):
        """Parse the inventory file and populate the inventory"""
        super(InventoryModule, self).parse(inventory, loader, path, cache)

        # Read configuration
        self._read_config_data(path)
        
        # Get configuration options
        cmdb_file = self.get_option('cmdb_file')
        group_by = self.get_option('group_by')
        include_vars = self.get_option('include_vars')
        exclude_environments = self.get_option('exclude_environments')
        
        # Load CMDB data
        try:
            cmdb_data = self._load_cmdb_data(cmdb_file)
        except Exception as e:
            raise AnsibleParserError(f"Failed to load CMDB data: {e}")
        
        # Process servers
        for server in cmdb_data.get('servers', []):
            hostname = server.get('hostname')
            if not hostname:
                continue
                
            # Skip excluded environments
            if server.get('environment') in exclude_environments:
                continue
            
            # Add host to inventory
            self.inventory.add_host(hostname)
            
            # Set host variables
            self._set_host_variables(hostname, server, include_vars)
            
            # Create groups
            self._create_groups(hostname, server, group_by)
    
    def _load_cmdb_data(self, cmdb_file):
        """Load CMDB data from file"""
        if not os.path.exists(cmdb_file):
            raise AnsibleError(f"CMDB file not found: {cmdb_file}")
        
        try:
            with open(cmdb_file, 'r') as f:
                return json.load(f)
        except json.JSONDecodeError as e:
            raise AnsibleError(f"Invalid JSON in CMDB file: {e}")
    
    def _set_host_variables(self, hostname, server, include_vars):
        """Set host variables from CMDB data"""
        if include_vars == ['all']:
            # Include all variables
            for key, value in server.items():
                self.inventory.set_variable(hostname, f'cmdb_{key}', value)
        else:
            # Include only specified variables
            for var in include_vars:
                if var in server:
                    self.inventory.set_variable(hostname, f'cmdb_{var}', server[var])
        
        # Always include basic Ansible variables
        self.inventory.set_variable(hostname, 'ansible_host', server.get('ip_address'))
        self.inventory.set_variable(hostname, 'ansible_user', 'ubuntu')
    
    def _create_groups(self, hostname, server, group_by):
        """Create groups based on server attributes"""
        for attribute in group_by:
            if attribute in server:
                value = server[attribute]
                if isinstance(value, str):
                    # Create group name from attribute value
                    group_name = f"{attribute}_{value}".replace('-', '_').replace(' ', '_').lower()
                    self.inventory.add_group(group_name)
                    self.inventory.add_child(group_name, hostname)
                    
                    # Set group variables
                    self.inventory.set_variable(group_name, attribute, value)
        
        # Create composite groups
        env = server.get('environment', 'unknown')
        app = server.get('application', 'unknown')
        composite_group = f"{env}_{app}".replace('-', '_').replace(' ', '_').lower()
        self.inventory.add_group(composite_group)
        self.inventory.add_child(composite_group, hostname)
EOF
```

Create inventory plugin configuration:

```yaml
# Create configs/cmdb-inventory.yml
cat > configs/cmdb-inventory.yml << 'EOF'
# Enterprise CMDB Inventory Configuration
plugin: enterprise_cmdb
cmdb_file: ../lab-4.2/data/cmdb-data.json
group_by:
  - environment
  - application
  - location
  - owner
include_vars:
  - hostname
  - fqdn
  - ip_address
  - os
  - cpu_cores
  - memory_gb
  - disk_gb
  - services
  - backup_enabled
  - monitoring_enabled
exclude_environments:
  - decommissioned
cache: true
cache_timeout: 300
EOF
```

Create ansible.cfg for plugin path:

```ini
# Create ansible.cfg
cat > ansible.cfg << 'EOF'
[defaults]
inventory_plugins = plugins/inventory
host_key_checking = False

[inventory]
enable_plugins = enterprise_cmdb, auto, yaml, ini
cache = True
cache_plugin = memory
cache_timeout = 300
EOF
```

Test custom inventory plugin:

```bash
# Test the custom inventory plugin
ansible-inventory -i configs/cmdb-inventory.yml --list --yaml
ansible-inventory -i configs/cmdb-inventory.yml --graph

# Test with playbook
cat > test-plugin.yml << 'EOF'
---
- name: Test custom inventory plugin
  hosts: all
  gather_facts: no
  tasks:
    - name: Display host information from plugin
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Groups: {{ group_names | join(', ') }}
          CMDB Hostname: {{ cmdb_hostname | default('N/A') }}
          CMDB Environment: {{ cmdb_environment | default('N/A') }}
          CMDB Application: {{ cmdb_application | default('N/A') }}
          CMDB CPU Cores: {{ cmdb_cpu_cores | default('N/A') }}
          CMDB Memory: {{ cmdb_memory_gb | default('N/A') }}GB
EOF

ansible-playbook -i configs/cmdb-inventory.yml test-plugin.yml
```

## Exercise 2: Cloud Provider Integration (8 minutes)

### Task: Configure inventory plugins for cloud providers (Azure simulation)

Create Azure inventory plugin configuration:

```yaml
# Create configs/azure-inventory.yml
cat > configs/azure-inventory.yml << 'EOF'
# Azure Inventory Plugin Configuration
# Note: This is a simulation using constructed plugin
plugin: constructed
strict: False
compose:
    # Compose new variables from existing ones
    ansible_host: azure_public_ip | default(azure_private_ip)
    ansible_user: "'azureuser'"
    cloud_provider: "'azure'"
    server_role: azure_tags.Application | default('unknown') | lower
    cost_center: azure_tags.CostCenter | default('unknown')
    
groups:
    # Create groups based on Azure properties
    azure_production: azure_tags.Environment == "Production"
    azure_staging: azure_tags.Environment == "Staging"
    azure_web_servers: "'web' in (azure_tags.Application | lower)"
    azure_database_servers: "'database' in (azure_tags.Application | lower)"
    azure_west_europe: azure_location == "West Europe"
    azure_north_europe: azure_location == "North Europe"
    azure_b_series: "'Standard_B' in azure_vm_size"
    azure_d_series: "'Standard_D' in azure_vm_size"
    
keyed_groups:
    # Create groups with keys
    - key: azure_tags.Environment | default('unknown')
      prefix: env
      separator: "_"
    - key: azure_tags.Application | default('unknown')
      prefix: app
      separator: "_"
    - key: azure_location | default('unknown') | replace(' ', '_')
      prefix: location
      separator: "_"
    - key: azure_vm_size | default('unknown')
      prefix: size
      separator: "_"
EOF
```

Create multi-cloud inventory configuration:

```yaml
# Create configs/multi-cloud-inventory.yml
cat > configs/multi-cloud-inventory.yml << 'EOF'
# Multi-Cloud Inventory Configuration
plugin: constructed
strict: False

# Compose variables for all cloud providers
compose:
    # Standardize connection variables
    ansible_host: |
        public_ip | default(
        azure_public_ip | default(
        aws_public_ip | default(
        private_ip | default(
        azure_private_ip | default(
        aws_private_ip | default('localhost'))))))
    
    ansible_user: |
        'azureuser' if cloud_provider == 'azure' else
        'ec2-user' if cloud_provider == 'aws' else
        'ubuntu'
    
    # Standardize environment classification
    environment_type: |
        'production' if (
            azure_tags.Environment == 'Production' or
            aws_tags.Environment == 'production' or
            cmdb_environment == 'production'
        ) else 'non_production'
    
    # Standardize application classification
    application_type: |
        'web' if (
            'web' in (azure_tags.Application | default('') | lower) or
            'web' in (aws_tags.Application | default('') | lower) or
            'web' in (cmdb_application | default('') | lower)
        ) else 'other'

groups:
    # Multi-cloud groups
    all_production: environment_type == 'production'
    all_non_production: environment_type == 'non_production'
    all_web_servers: application_type == 'web'
    all_cloud_instances: cloud_provider in ['azure', 'aws']
    all_on_premises: cloud_provider == 'on_premises' or cmdb_location is defined
    
    # Cloud-specific groups
    azure_instances: cloud_provider == 'azure'
    aws_instances: cloud_provider == 'aws'
    on_premises_servers: cloud_provider == 'on_premises' or cmdb_location is defined

keyed_groups:
    # Create groups by cloud provider and environment
    - key: cloud_provider + '_' + environment_type
      separator: ""
    # Create groups by application and cloud
    - key: application_type + '_' + cloud_provider
      separator: ""
EOF
```

Create test data for multi-cloud scenario:

```bash
# Create combined inventory data
cat > data/multi-cloud-hosts.yml << 'EOF'
# Multi-cloud host definitions for testing
all:
  children:
    azure_hosts:
      hosts:
        azure-web-01:
          cloud_provider: azure
          azure_public_ip: 20.50.1.10
          azure_private_ip: 10.0.1.10
          azure_vm_size: Standard_B2s
          azure_location: West Europe
          azure_tags:
            Environment: Production
            Application: WebFrontend
            Owner: DevOps Team
        azure-db-01:
          cloud_provider: azure
          azure_public_ip: null
          azure_private_ip: 10.0.2.10
          azure_vm_size: Standard_D4s_v3
          azure_location: West Europe
          azure_tags:
            Environment: Production
            Application: Database
            Owner: DBA Team
    
    aws_hosts:
      hosts:
        aws-web-01:
          cloud_provider: aws
          aws_public_ip: 54.123.45.67
          aws_private_ip: 172.31.1.10
          aws_instance_type: t3.medium
          aws_region: eu-west-1
          aws_tags:
            Environment: production
            Application: web-frontend
            Owner: devops-team
        aws-cache-01:
          cloud_provider: aws
          aws_public_ip: null
          aws_private_ip: 172.31.2.10
          aws_instance_type: r5.large
          aws_region: eu-west-1
          aws_tags:
            Environment: production
            Application: cache-cluster
            Owner: devops-team
    
    on_premises:
      hosts:
        onprem-db-01:
          cloud_provider: on_premises
          private_ip: 192.168.1.100
          cmdb_location: datacenter-1
          cmdb_environment: production
          cmdb_application: database
          cmdb_owner: dba-team
EOF
```

Test multi-cloud inventory:

```bash
# Test multi-cloud inventory
ansible-inventory -i data/multi-cloud-hosts.yml -i configs/multi-cloud-inventory.yml --list --yaml
ansible-inventory -i data/multi-cloud-hosts.yml -i configs/multi-cloud-inventory.yml --graph

# Create test playbook
cat > test-multi-cloud.yml << 'EOF'
---
- name: Test multi-cloud inventory
  hosts: all
  gather_facts: no
  tasks:
    - name: Display multi-cloud host information
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Cloud Provider: {{ cloud_provider }}
          Environment Type: {{ environment_type }}
          Application Type: {{ application_type }}
          Ansible Host: {{ ansible_host }}
          Ansible User: {{ ansible_user }}
          Groups: {{ group_names | join(', ') }}
    
    - name: Multi-cloud summary
      debug:
        msg: |
          Multi-Cloud Inventory Summary:
          - Total hosts: {{ groups['all'] | length }}
          - Production hosts: {{ groups['all_production'] | default([]) | length }}
          - Web servers: {{ groups['all_web_servers'] | default([]) | length }}
          - Azure instances: {{ groups['azure_instances'] | default([]) | length }}
          - AWS instances: {{ groups['aws_instances'] | default([]) | length }}
          - On-premises servers: {{ groups['on_premises_servers'] | default([]) | length }}
      run_once: true
EOF

ansible-playbook -i data/multi-cloud-hosts.yml -i configs/multi-cloud-inventory.yml test-multi-cloud.yml
```

## Exercise 3: Advanced Plugin Configuration (7 minutes)

### Task: Create advanced plugin configurations with filtering and transformation

Create advanced filtering configuration:

```yaml
# Create configs/advanced-filtering.yml
cat > configs/advanced-filtering.yml << 'EOF'
# Advanced Filtering and Transformation Configuration
plugin: constructed
strict: False

# Advanced variable composition
compose:
    # Calculate resource tier based on specs
    resource_tier: |
        'high' if (
            (azure_vm_size | default('') | regex_search('Standard_D[4-9]')) or
            (aws_instance_type | default('') | regex_search('[rm][4-9]')) or
            (cmdb_cpu_cores | default(0) | int >= 8)
        ) else (
            'medium' if (
                (azure_vm_size | default('') | regex_search('Standard_[BD][2-3]')) or
                (aws_instance_type | default('') | regex_search('t[2-3]\\.(medium|large)')) or
                (cmdb_cpu_cores | default(0) | int >= 4)
            ) else 'low'
        )
    
    # Determine backup strategy
    backup_strategy: |
        'critical' if (
            environment_type == 'production' and
            application_type in ['database', 'storage']
        ) else (
            'standard' if environment_type == 'production'
            else 'minimal'
        )
    
    # Calculate monthly cost estimate (simplified)
    estimated_monthly_cost: |
        (150 if resource_tier == 'high' else
         75 if resource_tier == 'medium' else
         25) * (1.5 if environment_type == 'production' else 1.0)
    
    # Determine patching window
    patching_window: |
        'weekend_night' if environment_type == 'production' else
        'weekday_evening'
    
    # Security classification
    security_level: |
        'high' if (
            'database' in application_type or
            'production' in environment_type
        ) else 'standard'

# Advanced grouping
groups:
    # Resource-based groups
    high_resource_hosts: resource_tier == 'high'
    medium_resource_hosts: resource_tier == 'medium'
    low_resource_hosts: resource_tier == 'low'
    
    # Cost-based groups
    high_cost_hosts: estimated_monthly_cost | int > 100
    medium_cost_hosts: estimated_monthly_cost | int > 50 and estimated_monthly_cost | int <= 100
    low_cost_hosts: estimated_monthly_cost | int <= 50
    
    # Backup strategy groups
    critical_backup: backup_strategy == 'critical'
    standard_backup: backup_strategy == 'standard'
    minimal_backup: backup_strategy == 'minimal'
    
    # Security groups
    high_security: security_level == 'high'
    standard_security: security_level == 'standard'
    
    # Patching groups
    weekend_patching: patching_window == 'weekend_night'
    weekday_patching: patching_window == 'weekday_evening'

keyed_groups:
    # Create groups by resource tier and environment
    - key: resource_tier + '_' + environment_type
      separator: ""
    # Create groups by backup strategy
    - key: backup_strategy + '_backup'
      separator: ""
    # Create groups by security level
    - key: security_level + '_security'
      separator: ""

# Conditional groups (only create if conditions are met)
conditional_groups:
    expensive_production: >-
        environment_type == 'production' and 
        estimated_monthly_cost | int > 100
    database_production: >-
        environment_type == 'production' and 
        'database' in application_type
EOF
```

Create comprehensive inventory test:

```yaml
# Create comprehensive-inventory-test.yml
cat > comprehensive-inventory-test.yml << 'EOF'
---
- name: Comprehensive inventory plugin testing
  hosts: all
  gather_facts: no
  tasks:
    - name: Display comprehensive host analysis
      debug:
        msg: |
          === Host Analysis: {{ inventory_hostname }} ===
          Basic Info:
          - Cloud Provider: {{ cloud_provider | default('unknown') }}
          - Environment: {{ environment_type | default('unknown') }}
          - Application: {{ application_type | default('unknown') }}
          
          Resource Analysis:
          - Resource Tier: {{ resource_tier | default('unknown') }}
          - Estimated Monthly Cost: ${{ estimated_monthly_cost | default(0) }}
          - Backup Strategy: {{ backup_strategy | default('unknown') }}
          
          Operational Info:
          - Patching Window: {{ patching_window | default('unknown') }}
          - Security Level: {{ security_level | default('unknown') }}
          
          Groups: {{ group_names | join(', ') }}
    
    - name: Resource tier analysis
      debug:
        msg: |
          Resource Tier Distribution:
          - High Resource: {{ groups['high_resource_hosts'] | default([]) | length }} hosts
          - Medium Resource: {{ groups['medium_resource_hosts'] | default([]) | length }} hosts
          - Low Resource: {{ groups['low_resource_hosts'] | default([]) | length }} hosts
      run_once: true
    
    - name: Cost analysis
      debug:
        msg: |
          Cost Distribution:
          - High Cost (>$100): {{ groups['high_cost_hosts'] | default([]) | length }} hosts
          - Medium Cost ($50-$100): {{ groups['medium_cost_hosts'] | default([]) | length }} hosts
          - Low Cost (<$50): {{ groups['low_cost_hosts'] | default([]) | length }} hosts
          
          Total Estimated Monthly Cost: ${{ groups['all'] | map('extract', hostvars, 'estimated_monthly_cost') | map('int') | sum }}
      run_once: true
    
    - name: Security and backup analysis
      debug:
        msg: |
          Security Classification:
          - High Security: {{ groups['high_security'] | default([]) | length }} hosts
          - Standard Security: {{ groups['standard_security'] | default([]) | length }} hosts
          
          Backup Strategy:
          - Critical Backup: {{ groups['critical_backup'] | default([]) | length }} hosts
          - Standard Backup: {{ groups['standard_backup'] | default([]) | length }} hosts
          - Minimal Backup: {{ groups['minimal_backup'] | default([]) | length }} hosts
      run_once: true
EOF
```

Test advanced filtering:

```bash
# Test advanced filtering configuration
ansible-inventory -i data/multi-cloud-hosts.yml -i configs/advanced-filtering.yml --list --yaml
ansible-inventory -i data/multi-cloud-hosts.yml -i configs/advanced-filtering.yml --graph

# Run comprehensive test
ansible-playbook -i data/multi-cloud-hosts.yml -i configs/advanced-filtering.yml comprehensive-inventory-test.yml
```

Create plugin performance comparison:

```bash
# Create performance test script
cat > test-performance.sh << 'EOF'
#!/bin/bash

echo "=== Inventory Plugin Performance Comparison ==="
echo

echo "1. Custom CMDB Plugin:"
time ansible-inventory -i configs/cmdb-inventory.yml --list > /dev/null 2>&1

echo
echo "2. Multi-cloud Constructed Plugin:"
time ansible-inventory -i data/multi-cloud-hosts.yml -i configs/multi-cloud-inventory.yml --list > /dev/null 2>&1

echo
echo "3. Advanced Filtering Plugin:"
time ansible-inventory -i data/multi-cloud-hosts.yml -i configs/advanced-filtering.yml --list > /dev/null 2>&1

echo
echo "4. Script-based Inventory (from Lab 4.2):"
time ansible-inventory -i ../lab-4.2/scripts/advanced-inventory.py --list > /dev/null 2>&1

echo
echo "Performance test completed."
EOF

chmod +x test-performance.sh
./test-performance.sh
```

## Verification and Discussion

### 1. Check Results
```bash
# Compare different plugin approaches
echo "=== Custom Plugin Structure ==="
ansible-inventory -i configs/cmdb-inventory.yml --graph

echo "=== Multi-cloud Plugin Structure ==="
ansible-inventory -i data/multi-cloud-hosts.yml -i configs/multi-cloud-inventory.yml --graph

echo "=== Advanced Filtering Structure ==="
ansible-inventory -i data/multi-cloud-hosts.yml -i configs/advanced-filtering.yml --graph

# Check plugin performance
./test-performance.sh
```

### 2. Discussion Points
- How do inventory plugins compare to custom scripts in terms of maintainability?
- What are the advantages of using the constructed plugin for complex transformations?
- How would you implement authentication for real cloud provider APIs?
- What caching strategies work best for large-scale inventory sources?

### 3. Clean Up
```bash
# Keep files for next lab
# rm -rf plugins configs data
# rm -f *.yml *.sh ansible.cfg
```

## Key Takeaways
- Inventory plugins provide a more structured approach than custom scripts
- The constructed plugin is powerful for data transformation and grouping
- Plugin configuration files are more maintainable than embedded logic
- Advanced filtering and grouping enable sophisticated inventory organization
- Performance considerations are important for large-scale inventories
- Plugin architecture supports better error handling and caching

## Next Steps
Proceed to Lab 4.4: Multiple Inventory Sources and Merging

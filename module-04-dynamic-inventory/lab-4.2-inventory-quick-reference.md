# Lab 4.2: Advanced Dynamic Inventory (Quick Reference)

## Objective
Advanced dynamic inventory patterns and enterprise integration for self-study

## Target Audience
**Victor & Vlad** - Intermediate participants for post-course exploration

## Duration
30-45 minutes (self-paced)

## Pattern 1: Cloud Provider Integration

### AWS EC2 Dynamic Inventory
```yaml
# aws_ec2.yml - Modern AWS inventory plugin
plugin: aws.ec2.aws_ec2
regions:
  - us-east-1
  - eu-west-1
  
# Group by instance state
keyed_groups:
  - key: instance_state_name
    prefix: state
  - key: instance_type
    prefix: type
  - key: placement.availability_zone
    prefix: az
  - key: tags.Environment
    prefix: env
    
# Include/exclude instances
filters:
  instance-state-name: running
  tag:Managed: ansible
  
# Host variables to include
hostnames:
  - tag:Name
  - dns-name
  - private-ip-address
  
compose:
  ansible_host: private_ip_address
  ec2_instance_type: instance_type
  ec2_placement_az: placement.availability_zone
  ec2_security_groups: security_groups | map(attribute='group_name') | list
```

### Azure Resource Manager Inventory
```yaml
# azure_rm.yml - Azure inventory plugin
plugin: azure.azcollection.azure_rm
auth_source: auto

# Include specific resource groups
include_vm_resource_groups:
  - rg-production
  - rg-staging
  
# Group by various attributes
keyed_groups:
  - key: location
    prefix: location
  - key: resource_group
    prefix: rg
  - key: tags.Environment
    prefix: env
  - key: tags.Application
    prefix: app
    
# Conditional groups
conditional_groups:
  webservers: tags.Role == "webserver"
  databases: tags.Role == "database"
  production: tags.Environment == "production"
  
# Host variables
compose:
  ansible_host: private_ipv4_addresses[0]
  azure_location: location
  azure_resource_group: resource_group
  azure_vm_size: vm_size
  azure_tags: tags
```

### Google Cloud Platform Inventory
```yaml
# gcp_compute.yml - GCP inventory plugin
plugin: google.cloud.gcp_compute
projects:
  - my-project-id
  
auth_kind: serviceaccount
service_account_file: /path/to/service-account.json

# Group by zones and labels
keyed_groups:
  - key: zone
    prefix: zone
  - key: labels.environment
    prefix: env
  - key: labels.application
    prefix: app
  - key: machineType.split('/')[-1]
    prefix: type
    
# Filter running instances
filters:
  - status = "RUNNING"
  - labels.managed = "ansible"
  
compose:
  ansible_host: networkInterfaces[0].accessConfigs[0].natIP | default(networkInterfaces[0].networkIP)
  gcp_machine_type: machineType.split('/')[-1]
  gcp_zone: zone.split('/')[-1]
  gcp_labels: labels
```

## Pattern 2: Custom Enterprise Inventory Scripts

### CMDB Integration Script
```python
#!/usr/bin/env python3
"""
Enterprise CMDB Dynamic Inventory Script
Integrates with ServiceNow or similar CMDB systems
"""
import json
import sys
import requests
from requests.auth import HTTPBasicAuth
import os

class CMDBInventory:
    def __init__(self):
        self.cmdb_url = os.environ.get('CMDB_URL', 'https://company.service-now.com')
        self.cmdb_user = os.environ.get('CMDB_USER')
        self.cmdb_pass = os.environ.get('CMDB_PASS')
        self.inventory = {'_meta': {'hostvars': {}}}
        
    def get_servers_from_cmdb(self):
        """Fetch server data from CMDB"""
        url = f"{self.cmdb_url}/api/now/table/cmdb_ci_server"
        params = {
            'sysparm_query': 'operational_status=1^install_status=1',  # Active servers
            'sysparm_fields': 'name,ip_address,os,environment,application,location,owner'
        }
        
        try:
            response = requests.get(
                url, 
                params=params,
                auth=HTTPBasicAuth(self.cmdb_user, self.cmdb_pass),
                headers={'Accept': 'application/json'},
                timeout=30
            )
            response.raise_for_status()
            return response.json().get('result', [])
        except Exception as e:
            print(f"Error fetching CMDB data: {e}", file=sys.stderr)
            return []
    
    def build_inventory(self):
        """Build Ansible inventory from CMDB data"""
        servers = self.get_servers_from_cmdb()
        
        for server in servers:
            hostname = server.get('name', '').lower()
            if not hostname:
                continue
                
            # Extract server attributes
            ip_address = server.get('ip_address', '')
            os_type = server.get('os', '').lower()
            environment = server.get('environment', '').lower()
            application = server.get('application', '').lower()
            location = server.get('location', '').lower()
            owner = server.get('owner', '').lower()
            
            # Create groups
            groups = []
            
            # OS-based groups
            if 'linux' in os_type:
                groups.append('linux')
            elif 'windows' in os_type:
                groups.append('windows')
                
            # Environment groups
            if environment:
                groups.append(f"env_{environment}")
                groups.append(environment)
                
            # Application groups
            if application:
                groups.append(f"app_{application}")
                
            # Location groups
            if location:
                groups.append(f"location_{location}")
                
            # Owner groups
            if owner:
                groups.append(f"owner_{owner}")
            
            # Add host to groups
            for group in groups:
                if group not in self.inventory:
                    self.inventory[group] = {'hosts': [], 'vars': {}}
                self.inventory[group]['hosts'].append(hostname)
            
            # Add host variables
            self.inventory['_meta']['hostvars'][hostname] = {
                'ansible_host': ip_address,
                'cmdb_os': os_type,
                'cmdb_environment': environment,
                'cmdb_application': application,
                'cmdb_location': location,
                'cmdb_owner': owner,
                'ansible_user': 'ansible' if 'linux' in os_type else 'Administrator'
            }
        
        return self.inventory

def main():
    if len(sys.argv) == 2 and sys.argv[1] == '--list':
        inventory = CMDBInventory()
        result = inventory.build_inventory()
        print(json.dumps(result, indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == '--host':
        # Return empty dict (using _meta for performance)
        print(json.dumps({}))
    else:
        print("Usage: %s --list or %s --host <hostname>" % (sys.argv[0], sys.argv[0]))
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### Kubernetes Dynamic Inventory
```python
#!/usr/bin/env python3
"""
Kubernetes Dynamic Inventory Script
Discovers pods, services, and nodes from Kubernetes clusters
"""
import json
import sys
import yaml
from kubernetes import client, config
import os

class K8sInventory:
    def __init__(self):
        self.inventory = {'_meta': {'hostvars': {}}}
        self.load_k8s_config()
        
    def load_k8s_config(self):
        """Load Kubernetes configuration"""
        try:
            # Try in-cluster config first
            config.load_incluster_config()
        except:
            # Fall back to kubeconfig
            config.load_kube_config()
        
        self.v1 = client.CoreV1Api()
        self.apps_v1 = client.AppsV1Api()
    
    def get_pods(self):
        """Get all pods from all namespaces"""
        try:
            pods = self.v1.list_pod_for_all_namespaces()
            return pods.items
        except Exception as e:
            print(f"Error fetching pods: {e}", file=sys.stderr)
            return []
    
    def get_services(self):
        """Get all services from all namespaces"""
        try:
            services = self.v1.list_service_for_all_namespaces()
            return services.items
        except Exception as e:
            print(f"Error fetching services: {e}", file=sys.stderr)
            return []
    
    def build_inventory(self):
        """Build inventory from Kubernetes resources"""
        pods = self.get_pods()
        services = self.get_services()
        
        # Process pods
        for pod in pods:
            if pod.status.phase != 'Running':
                continue
                
            pod_name = pod.metadata.name
            namespace = pod.metadata.namespace
            labels = pod.metadata.labels or {}
            
            # Create groups from labels
            groups = [f"namespace_{namespace}"]
            
            for key, value in labels.items():
                groups.append(f"label_{key}_{value}")
                if key == 'app':
                    groups.append(f"app_{value}")
                elif key == 'environment':
                    groups.append(f"env_{value}")
            
            # Add to groups
            for group in groups:
                if group not in self.inventory:
                    self.inventory[group] = {'hosts': [], 'vars': {}}
                self.inventory[group]['hosts'].append(pod_name)
            
            # Add host variables
            self.inventory['_meta']['hostvars'][pod_name] = {
                'ansible_host': pod.status.pod_ip,
                'k8s_namespace': namespace,
                'k8s_labels': labels,
                'k8s_node': pod.spec.node_name,
                'k8s_pod_ip': pod.status.pod_ip,
                'ansible_connection': 'kubectl',
                'ansible_kubectl_namespace': namespace,
                'ansible_kubectl_pod': pod_name
            }
        
        # Process services
        for service in services:
            service_name = f"svc_{service.metadata.name}"
            namespace = service.metadata.namespace
            
            if service.spec.cluster_ip and service.spec.cluster_ip != 'None':
                group_name = f"services_{namespace}"
                if group_name not in self.inventory:
                    self.inventory[group_name] = {'hosts': [], 'vars': {}}
                self.inventory[group_name]['hosts'].append(service_name)
                
                self.inventory['_meta']['hostvars'][service_name] = {
                    'ansible_host': service.spec.cluster_ip,
                    'k8s_namespace': namespace,
                    'k8s_service_type': service.spec.type,
                    'k8s_ports': [{'port': p.port, 'protocol': p.protocol} for p in service.spec.ports or []]
                }
        
        return self.inventory

def main():
    if len(sys.argv) == 2 and sys.argv[1] == '--list':
        inventory = K8sInventory()
        result = inventory.build_inventory()
        print(json.dumps(result, indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == '--host':
        print(json.dumps({}))
    else:
        print("Usage: %s --list or %s --host <hostname>" % (sys.argv[0], sys.argv[0]))
        sys.exit(1)

if __name__ == '__main__':
    main()
```

## Pattern 3: Inventory Plugin Development

### Custom Inventory Plugin Template
```python
# plugins/inventory/custom_source.py
"""
Custom Inventory Plugin Template
"""
from __future__ import (absolute_import, division, print_function)
__metaclass__ = type

DOCUMENTATION = '''
    name: custom_source
    plugin_type: inventory
    short_description: Custom inventory source
    description:
        - Fetches inventory from custom data source
        - Supports grouping and host variables
    options:
        plugin:
            description: Name of the plugin
            required: true
            choices: ['custom_source']
        data_source:
            description: URL or path to data source
            required: true
            type: str
        cache:
            description: Enable caching
            type: bool
            default: false
        cache_timeout:
            description: Cache timeout in seconds
            type: int
            default: 3600
'''

from ansible.plugins.inventory import BaseInventoryPlugin, Constructable, Cacheable
from ansible.errors import AnsibleError
import requests
import json

class InventoryModule(BaseInventoryPlugin, Constructable, Cacheable):
    NAME = 'custom_source'
    
    def verify_file(self, path):
        """Verify this is a valid inventory source"""
        return path.endswith(('custom_source.yml', 'custom_source.yaml'))
    
    def parse(self, inventory, loader, path, cache=True):
        """Parse the inventory source"""
        super(InventoryModule, self).parse(inventory, loader, path, cache)
        
        # Read configuration
        self._read_config_data(path)
        
        # Get configuration options
        data_source = self.get_option('data_source')
        cache_enabled = self.get_option('cache')
        cache_timeout = self.get_option('cache_timeout')
        
        # Set up caching
        cache_key = self.get_cache_key(path)
        
        # Try to get from cache
        if cache_enabled:
            try:
                data = self._cache[cache_key]
            except KeyError:
                data = self._fetch_data(data_source)
                self._cache[cache_key] = data
        else:
            data = self._fetch_data(data_source)
        
        # Process data and populate inventory
        self._populate_inventory(data)
    
    def _fetch_data(self, source):
        """Fetch data from source"""
        try:
            if source.startswith('http'):
                response = requests.get(source, timeout=30)
                response.raise_for_status()
                return response.json()
            else:
                with open(source, 'r') as f:
                    return json.load(f)
        except Exception as e:
            raise AnsibleError(f"Failed to fetch data from {source}: {e}")
    
    def _populate_inventory(self, data):
        """Populate inventory from data"""
        for item in data.get('hosts', []):
            hostname = item['name']
            
            # Add host
            self.inventory.add_host(hostname)
            
            # Set host variables
            for key, value in item.get('vars', {}).items():
                self.inventory.set_variable(hostname, key, value)
            
            # Add to groups
            for group in item.get('groups', []):
                self.inventory.add_group(group)
                self.inventory.add_child(group, hostname)
            
            # Use constructable features for dynamic grouping
            self._set_composite_vars(
                self.get_option('compose'), 
                item.get('vars', {}), 
                hostname
            )
            
            self._add_host_to_composed_groups(
                self.get_option('groups'), 
                item.get('vars', {}), 
                hostname
            )
            
            self._add_host_to_keyed_groups(
                self.get_option('keyed_groups'), 
                item.get('vars', {}), 
                hostname
            )
```

## Pattern 4: Performance Optimization

### Caching Strategies
```python
# Inventory caching example
import pickle
import os
import time

class CachedInventory:
    def __init__(self, cache_file='/tmp/ansible_inventory_cache.pkl', cache_timeout=3600):
        self.cache_file = cache_file
        self.cache_timeout = cache_timeout
    
    def is_cache_valid(self):
        """Check if cache is still valid"""
        if not os.path.exists(self.cache_file):
            return False
        
        cache_age = time.time() - os.path.getmtime(self.cache_file)
        return cache_age < self.cache_timeout
    
    def load_cache(self):
        """Load inventory from cache"""
        with open(self.cache_file, 'rb') as f:
            return pickle.load(f)
    
    def save_cache(self, inventory):
        """Save inventory to cache"""
        with open(self.cache_file, 'wb') as f:
            pickle.dump(inventory, f)
    
    def get_inventory(self):
        """Get inventory with caching"""
        if self.is_cache_valid():
            return self.load_cache()
        
        # Fetch fresh data
        inventory = self.fetch_fresh_inventory()
        self.save_cache(inventory)
        return inventory
```

### Parallel Data Fetching
```python
import concurrent.futures
import requests

class ParallelInventory:
    def __init__(self):
        self.endpoints = [
            'https://api1.company.com/servers',
            'https://api2.company.com/servers',
            'https://api3.company.com/servers'
        ]
    
    def fetch_endpoint(self, url):
        """Fetch data from single endpoint"""
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            return response.json()
        except Exception as e:
            print(f"Error fetching {url}: {e}")
            return []
    
    def fetch_all_parallel(self):
        """Fetch from all endpoints in parallel"""
        all_data = []
        
        with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
            future_to_url = {
                executor.submit(self.fetch_endpoint, url): url 
                for url in self.endpoints
            }
            
            for future in concurrent.futures.as_completed(future_to_url):
                url = future_to_url[future]
                try:
                    data = future.result()
                    all_data.extend(data)
                except Exception as e:
                    print(f"Error processing {url}: {e}")
        
        return all_data
```

## Pattern 5: Error Handling and Resilience

### Robust Error Handling
```python
import logging
import sys
from functools import wraps

def retry_on_failure(max_retries=3, delay=1):
    """Decorator for retrying failed operations"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    logging.warning(f"Attempt {attempt + 1} failed: {e}")
                    time.sleep(delay * (2 ** attempt))  # Exponential backoff
            return None
        return wrapper
    return decorator

class ResilientInventory:
    def __init__(self):
        self.logger = logging.getLogger(__name__)
        self.fallback_data = {'_meta': {'hostvars': {}}}
    
    @retry_on_failure(max_retries=3)
    def fetch_primary_source(self):
        """Fetch from primary data source with retry"""
        # Primary data source logic
        pass
    
    def fetch_fallback_source(self):
        """Fetch from fallback source"""
        # Fallback logic (local cache, static file, etc.)
        return self.fallback_data
    
    def get_inventory(self):
        """Get inventory with fallback"""
        try:
            return self.fetch_primary_source()
        except Exception as e:
            self.logger.error(f"Primary source failed: {e}")
            self.logger.info("Using fallback source")
            return self.fetch_fallback_source()
```

## Best Practices Summary

### Performance Optimization:
1. **Use _meta section** - Include all host vars in _meta for better performance
2. **Implement caching** - Cache expensive API calls and database queries
3. **Parallel processing** - Fetch from multiple sources concurrently
4. **Pagination** - Handle large datasets with pagination
5. **Connection pooling** - Reuse HTTP connections for multiple requests

### Error Handling:
1. **Graceful degradation** - Provide fallback data sources
2. **Retry logic** - Implement exponential backoff for transient failures
3. **Timeout handling** - Set appropriate timeouts for external calls
4. **Logging** - Log errors and performance metrics
5. **Validation** - Validate data before generating inventory

### Security Considerations:
1. **Credential management** - Use environment variables or secure vaults
2. **TLS verification** - Always verify SSL certificates
3. **Input validation** - Sanitize external data
4. **Access control** - Implement proper authentication and authorization
5. **Audit logging** - Log inventory access and changes

### Maintenance:
1. **Version control** - Track inventory script changes
2. **Testing** - Unit test inventory scripts
3. **Documentation** - Document data sources and group logic
4. **Monitoring** - Monitor inventory performance and errors
5. **Regular updates** - Keep dependencies and APIs updated

---

**Note**: This reference covers advanced dynamic inventory patterns for experienced users. Start with simple scripts and gradually incorporate these advanced techniques as your infrastructure complexity grows.

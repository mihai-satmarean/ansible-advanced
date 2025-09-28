# Lab 7.2: Advanced Pull Strategies

## Objective
Implement advanced ansible-pull strategies including secure configurations, distributed architectures, comprehensive monitoring, and failure recovery mechanisms for enterprise-scale pull-based automation.

## Duration
20 minutes

## Prerequisites
- Completed Lab 7.1 (Pull Fundamentals)
- Understanding of SSH key management and Git authentication
- Knowledge of systemd services and monitoring concepts
- Basic understanding of distributed systems concepts

## Lab Setup

```bash
cd ~/ansible-labs/module-07
mkdir -p lab-7.2
cd lab-7.2

# Create inventory for advanced pull testing
cat > inventory.ini << 'EOF'
[pull_masters]
master1 ansible_host=localhost ansible_connection=local

[pull_nodes]
node1 ansible_host=localhost ansible_connection=local
node2 ansible_host=localhost ansible_connection=local

[monitoring_nodes]
monitor1 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Secure Pull Configuration (8 minutes)

### Task: Implement secure ansible-pull with SSH keys, encrypted variables, and access controls

Set up SSH key-based authentication:

```bash
# Create SSH key for pull operations
mkdir -p ~/.ssh/ansible-pull
ssh-keygen -t ed25519 -f ~/.ssh/ansible-pull/id_ed25519 -N "" -C "ansible-pull-lab"

# Create SSH config for pull operations
cat > ~/.ssh/ansible-pull/config << 'EOF'
# SSH Configuration for Ansible Pull
Host pull-repo
    HostName localhost
    User git
    IdentityFile ~/.ssh/ansible-pull/id_ed25519
    IdentitiesOnly yes
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel ERROR

Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/ansible-pull/id_ed25519
    IdentitiesOnly yes

Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/ansible-pull/id_ed25519
    IdentitiesOnly yes
EOF

chmod 600 ~/.ssh/ansible-pull/config ~/.ssh/ansible-pull/id_ed25519
chmod 644 ~/.ssh/ansible-pull/id_ed25519.pub

echo "SSH keys generated for secure pull operations"
```

Create secure pull repository with encrypted variables:

```bash
# Create secure pull repository
mkdir -p secure-pull-repo
cd secure-pull-repo

git init
git config user.name "Secure Pull Lab"
git config user.email "secure@example.com"

# Create directory structure
mkdir -p {playbooks,inventory,group_vars,host_vars,vault,roles,templates}
```

Create Ansible Vault encrypted variables:

```bash
# Create vault password file (for lab purposes only)
echo "lab-vault-password-123" > .vault_pass
chmod 600 .vault_pass

# Create encrypted group variables
ansible-vault create group_vars/all.yml --vault-password-file .vault_pass << 'EOF'
---
# Encrypted variables for secure pull operations
database_password: "super_secure_db_password_123"
api_key: "sk-1234567890abcdef"
ssl_private_key_content: |
  -----BEGIN PRIVATE KEY-----
  MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC7...
  -----END PRIVATE KEY-----

encryption_key: "aes256-encryption-key-for-sensitive-data"
jwt_secret: "jwt-signing-secret-key-very-long-and-secure"

# Service credentials
service_accounts:
  monitoring:
    username: "monitor_user"
    password: "monitor_secure_pass_456"
  backup:
    username: "backup_user"
    password: "backup_secure_pass_789"

# External service configurations
external_services:
  payment_gateway:
    api_url: "https://api.payment.example.com"
    merchant_id: "MERCH123456"
    secret_key: "payment_secret_key_xyz"
  
  notification_service:
    api_url: "https://notifications.example.com"
    api_token: "notif_token_abcdef123456"
    webhook_secret: "webhook_secret_789xyz"

# Certificate and key paths
ssl_certificates:
  web_server:
    cert_path: "/etc/ssl/certs/webserver.crt"
    key_path: "/etc/ssl/private/webserver.key"
  api_server:
    cert_path: "/etc/ssl/certs/apiserver.crt"
    key_path: "/etc/ssl/private/apiserver.key"
EOF

# Create encrypted host-specific variables
ansible-vault create host_vars/node1.yml --vault-password-file .vault_pass << 'EOF'
---
# Node1 specific encrypted variables
node_id: "node-001-production"
node_secret: "node1-specific-secret-key"
database_connection_string: "postgresql://user:secure_pass@db1.internal:5432/app_db"

local_certificates:
  client_cert: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAKoK/heBjcOuMA0GCSqGSIb3DQEBCwUA...
    -----END CERTIFICATE-----
  client_key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDd...
    -----END PRIVATE KEY-----
EOF

ansible-vault create host_vars/node2.yml --vault-password-file .vault_pass << 'EOF'
---
# Node2 specific encrypted variables
node_id: "node-002-production"
node_secret: "node2-specific-secret-key"
database_connection_string: "postgresql://user:secure_pass@db2.internal:5432/app_db"

local_certificates:
  client_cert: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAKoK/heBjcOuMA0GCSqGSIb3DQEBCwUA...
    -----END CERTIFICATE-----
  client_key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDd...
    -----END PRIVATE KEY-----
EOF
```

Create secure pull playbook:

```yaml
# Create playbooks/secure-site.yml
cat > playbooks/secure-site.yml << 'EOF'
---
- name: Secure Ansible Pull Configuration
  hosts: all
  become: yes
  gather_facts: yes
  vars_files:
    - ../group_vars/all.yml
  
  vars:
    pull_timestamp: "{{ ansible_date_time.iso8601 }}"
    pull_user: "{{ ansible_user_id }}"
    config_version: "2.0.0-secure"
    
  tasks:
    - name: Ensure secure directories exist
      file:
        path: "{{ item }}"
        state: directory
        mode: '0750'
        owner: root
        group: root
      loop:
        - /etc/ansible-pull
        - /var/log/ansible-pull/secure
        - /opt/secure-app/config
        - /opt/secure-app/certs
    
    - name: Create secure configuration file
      template:
        src: secure-config.j2
        dest: /opt/secure-app/config/app.conf
        mode: '0600'
        owner: root
        group: root
      notify: restart secure application
    
    - name: Deploy SSL certificates
      copy:
        content: "{{ item.content }}"
        dest: "{{ item.dest }}"
        mode: "{{ item.mode }}"
        owner: root
        group: root
      loop:
        - content: "{{ local_certificates.client_cert }}"
          dest: "/opt/secure-app/certs/client.crt"
          mode: "0644"
        - content: "{{ local_certificates.client_key }}"
          dest: "/opt/secure-app/certs/client.key"
          mode: "0600"
      when: local_certificates is defined
      notify: restart secure application
    
    - name: Create service account files
      copy:
        content: |
          # Service Account Configuration
          # Generated: {{ pull_timestamp }}
          
          [monitoring]
          username={{ service_accounts.monitoring.username }}
          password={{ service_accounts.monitoring.password }}
          
          [backup]
          username={{ service_accounts.backup.username }}
          password={{ service_accounts.backup.password }}
        dest: /etc/ansible-pull/service-accounts.conf
        mode: '0600'
        owner: root
        group: root
    
    - name: Configure external service connections
      template:
        src: external-services.j2
        dest: /opt/secure-app/config/external-services.yml
        mode: '0600'
        owner: root
        group: root
      notify: restart secure application
    
    - name: Create secure pull status
      copy:
        content: |
          {
            "timestamp": "{{ pull_timestamp }}",
            "node_id": "{{ node_id | default('unknown') }}",
            "config_version": "{{ config_version }}",
            "security_level": "encrypted",
            "vault_used": true,
            "ssl_configured": {{ (local_certificates is defined) | lower }},
            "services_configured": {{ (service_accounts is defined) | lower }},
            "external_services": {{ (external_services is defined) | lower }},
            "pull_user": "{{ pull_user }}",
            "hostname": "{{ ansible_hostname }}",
            "status": "success"
          }
        dest: /var/log/ansible-pull/secure/status.json
        mode: '0644'
    
    - name: Log secure pull execution
      lineinfile:
        path: /var/log/ansible-pull/secure/execution.log
        line: "{{ pull_timestamp }} - Secure pull executed by {{ pull_user }} on {{ node_id | default(ansible_hostname) }} - Config {{ config_version }}"
        create: yes
        mode: '0644'
    
    - name: Validate security configuration
      block:
        - name: Check certificate files
          stat:
            path: "{{ item }}"
          register: cert_files
          loop:
            - /opt/secure-app/certs/client.crt
            - /opt/secure-app/certs/client.key
          when: local_certificates is defined
        
        - name: Verify certificate permissions
          assert:
            that:
              - item.stat.exists
              - item.stat.mode == "0644" or item.stat.mode == "0600"
            fail_msg: "Certificate file {{ item.item }} has incorrect permissions"
          loop: "{{ cert_files.results }}"
          when: local_certificates is defined and cert_files is defined
        
        - name: Check service account configuration
          stat:
            path: /etc/ansible-pull/service-accounts.conf
          register: service_accounts_file
        
        - name: Verify service account file permissions
          assert:
            that:
              - service_accounts_file.stat.exists
              - service_accounts_file.stat.mode == "0600"
            fail_msg: "Service accounts file has incorrect permissions"
  
  handlers:
    - name: restart secure application
      systemd:
        name: secure-app
        state: restarted
      failed_when: false
EOF
```

Create secure configuration templates:

```yaml
# Create templates/secure-config.j2
cat > templates/secure-config.j2 << 'EOF'
# Secure Application Configuration
# Generated by Ansible Pull on {{ ansible_date_time.iso8601 }}
# Node ID: {{ node_id | default('unknown') }}

[application]
name = "SecureApp"
version = "{{ config_version }}"
node_id = "{{ node_id | default(ansible_hostname) }}"
environment = "production"

[database]
connection_string = "{{ database_connection_string | default('sqlite:///tmp/app.db') }}"
password = "{{ database_password }}"
encryption_key = "{{ encryption_key }}"

[api]
api_key = "{{ api_key }}"
jwt_secret = "{{ jwt_secret }}"

[ssl]
{% if local_certificates is defined %}
cert_file = "/opt/secure-app/certs/client.crt"
key_file = "/opt/secure-app/certs/client.key"
enabled = true
{% else %}
enabled = false
{% endif %}

[monitoring]
enabled = true
username = "{{ service_accounts.monitoring.username }}"
password = "{{ service_accounts.monitoring.password }}"

[backup]
enabled = true
username = "{{ service_accounts.backup.username }}"
password = "{{ service_accounts.backup.password }}"

[security]
encryption_enabled = true
vault_integration = true
last_updated = "{{ ansible_date_time.iso8601 }}"
EOF

# Create templates/external-services.j2
cat > templates/external-services.j2 << 'EOF'
# External Services Configuration
# Generated: {{ ansible_date_time.iso8601 }}

external_services:
  payment_gateway:
    api_url: "{{ external_services.payment_gateway.api_url }}"
    merchant_id: "{{ external_services.payment_gateway.merchant_id }}"
    secret_key: "{{ external_services.payment_gateway.secret_key }}"
    timeout: 30
    retry_attempts: 3
    
  notification_service:
    api_url: "{{ external_services.notification_service.api_url }}"
    api_token: "{{ external_services.notification_service.api_token }}"
    webhook_secret: "{{ external_services.notification_service.webhook_secret }}"
    timeout: 15
    retry_attempts: 2

security:
  encryption_in_transit: true
  certificate_validation: true
  token_refresh_interval: 3600
  
logging:
  external_service_calls: true
  sensitive_data_masking: true
  log_level: "INFO"
EOF
```

Create secure pull script:

```bash
# Create secure-pull.sh
cat > secure-pull.sh << 'EOF'
#!/bin/bash
# Secure Ansible Pull Script
# Implements security best practices for pull operations

set -euo pipefail

# Configuration
REPO_URL="../secure-pull-repo.git"
BRANCH="main"
PLAYBOOK="playbooks/secure-site.yml"
INVENTORY="inventory/hosts"
VAULT_PASSWORD_FILE="../secure-pull-repo/.vault_pass"
SSH_CONFIG_FILE="$HOME/.ssh/ansible-pull/config"
PULL_DIR="/tmp/secure-pull-$(date +%Y%m%d-%H%M%S)"
LOG_FILE="/var/log/ansible-pull/secure/secure-pull-$(date +%Y%m%d).log"
LOCK_FILE="/var/run/secure-ansible-pull.lock"

# Ensure log directory exists
mkdir -p "$(dirname "${LOG_FILE}")"

# Logging function
log() {
    local level=$1
    shift
    local message="$*"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    echo "${timestamp} [${level}] ${message}" | tee -a "${LOG_FILE}"
}

# Security validation
validate_security() {
    log "INFO" "Performing security validation"
    
    # Check vault password file
    if [[ ! -f "${VAULT_PASSWORD_FILE}" ]]; then
        log "ERROR" "Vault password file not found: ${VAULT_PASSWORD_FILE}"
        return 1
    fi
    
    # Check vault password file permissions
    local vault_perms=$(stat -c "%a" "${VAULT_PASSWORD_FILE}")
    if [[ "${vault_perms}" != "600" ]]; then
        log "ERROR" "Vault password file has incorrect permissions: ${vault_perms}"
        return 1
    fi
    
    # Check SSH configuration
    if [[ ! -f "${SSH_CONFIG_FILE}" ]]; then
        log "WARN" "SSH config file not found: ${SSH_CONFIG_FILE}"
    fi
    
    # Validate repository URL
    if [[ ! "${REPO_URL}" =~ ^(git@|https://|file://) ]]; then
        log "ERROR" "Invalid repository URL format: ${REPO_URL}"
        return 1
    fi
    
    log "INFO" "Security validation passed"
    return 0
}

# Cleanup function
cleanup() {
    if [[ -f "${LOCK_FILE}" ]]; then
        rm -f "${LOCK_FILE}"
    fi
    
    if [[ -d "${PULL_DIR}" ]]; then
        # Secure cleanup - overwrite sensitive files
        find "${PULL_DIR}" -type f -name "*.yml" -exec shred -vfz -n 3 {} \; 2>/dev/null || true
        rm -rf "${PULL_DIR}"
    fi
}

# Error handling
error_exit() {
    log "ERROR" "$1"
    cleanup
    exit 1
}

# Signal handlers
trap cleanup EXIT
trap 'error_exit "Script interrupted"' INT TERM

# Main execution
main() {
    log "INFO" "Starting secure ansible-pull execution"
    
    # Check for lock file
    if [[ -f "${LOCK_FILE}" ]]; then
        error_exit "Another secure ansible-pull process is already running"
    fi
    
    # Create lock file
    echo $$ > "${LOCK_FILE}"
    
    # Security validation
    if ! validate_security; then
        error_exit "Security validation failed"
    fi
    
    # Set SSH configuration
    if [[ -f "${SSH_CONFIG_FILE}" ]]; then
        export GIT_SSH_COMMAND="ssh -F ${SSH_CONFIG_FILE}"
        log "INFO" "Using SSH config: ${SSH_CONFIG_FILE}"
    fi
    
    # Execute secure ansible-pull
    log "INFO" "Executing secure ansible-pull"
    log "INFO" "Repository: ${REPO_URL}"
    log "INFO" "Branch: ${BRANCH}"
    log "INFO" "Playbook: ${PLAYBOOK}"
    log "INFO" "Vault: Enabled"
    
    if ansible-pull \
        --url "${REPO_URL}" \
        --checkout "${BRANCH}" \
        --directory "${PULL_DIR}" \
        --inventory "${INVENTORY}" \
        --vault-password-file "${VAULT_PASSWORD_FILE}" \
        --limit localhost \
        --verbose \
        "${PLAYBOOK}" 2>&1 | tee -a "${LOG_FILE}"; then
        
        log "INFO" "Secure ansible-pull execution completed successfully"
        
        # Validate results
        if [[ -f "/var/log/ansible-pull/secure/status.json" ]]; then
            log "INFO" "Secure pull status file created"
        fi
        
    else
        error_exit "Secure ansible-pull execution failed"
    fi
    
    log "INFO" "Secure ansible-pull execution completed"
}

# Execute main function
main "$@"
EOF

chmod +x secure-pull.sh

# Commit secure repository
cd ../secure-pull-repo
git add .
git commit -m "Initial secure ansible-pull configuration with encrypted variables"

# Create bare repository for testing
cd ..
git clone --bare secure-pull-repo secure-pull-repo.git
cd secure-pull-repo
git remote add origin ../secure-pull-repo.git
git push origin main

cd ../lab-7.2

echo "Secure pull configuration completed!"
```

Test secure pull:

```bash
# Test secure pull execution
echo "Testing secure ansible-pull..."
./secure-pull.sh

# Verify secure configuration
echo "=== Secure Pull Results ==="
if [[ -f "/var/log/ansible-pull/secure/status.json" ]]; then
    echo "Secure status:"
    cat /var/log/ansible-pull/secure/status.json | python3 -m json.tool
fi

if [[ -f "/opt/secure-app/config/app.conf" ]]; then
    echo "Secure configuration created (first 10 lines):"
    head -10 /opt/secure-app/config/app.conf
fi
```

## Exercise 2: Distributed Pull Architecture and Monitoring (12 minutes)

### Task: Implement distributed pull architecture with centralized monitoring and failure recovery

Create distributed pull coordinator:

```bash
# Create distributed pull coordinator
cat > distributed-pull-coordinator.py << 'EOF'
#!/usr/bin/env python3
"""
Distributed Ansible Pull Coordinator
Manages multiple pull nodes and provides centralized monitoring
"""

import json
import subprocess
import threading
import time
import logging
import os
from datetime import datetime, timedelta
from pathlib import Path
import yaml

class PullNodeManager:
    def __init__(self, config_file="pull-coordinator.yml"):
        self.config_file = config_file
        self.config = self.load_config()
        self.setup_logging()
        self.nodes = {}
        self.monitoring_active = False
        
    def load_config(self):
        """Load coordinator configuration"""
        try:
            with open(self.config_file, 'r') as f:
                return yaml.safe_load(f)
        except FileNotFoundError:
            return self.get_default_config()
    
    def get_default_config(self):
        """Get default configuration"""
        return {
            'coordinator': {
                'name': 'ansible-pull-coordinator',
                'log_level': 'INFO',
                'log_file': '/var/log/ansible-pull/coordinator.log',
                'status_file': '/var/log/ansible-pull/coordinator-status.json',
                'monitoring_interval': 60,
                'failure_threshold': 3,
                'recovery_enabled': True
            },
            'repositories': {
                'primary': {
                    'url': '../secure-pull-repo.git',
                    'branch': 'main',
                    'playbook': 'playbooks/secure-site.yml',
                    'inventory': 'inventory/hosts'
                }
            },
            'nodes': {
                'node1': {
                    'hostname': 'localhost',
                    'connection': 'local',
                    'pull_interval': 900,  # 15 minutes
                    'max_failures': 3,
                    'recovery_action': 'restart',
                    'priority': 'high'
                },
                'node2': {
                    'hostname': 'localhost',
                    'connection': 'local',
                    'pull_interval': 1800,  # 30 minutes
                    'max_failures': 5,
                    'recovery_action': 'alert',
                    'priority': 'medium'
                }
            },
            'notifications': {
                'enabled': True,
                'email': {
                    'enabled': False,
                    'smtp_server': 'localhost',
                    'from_address': 'ansible-pull@example.com',
                    'to_addresses': ['admin@example.com']
                },
                'webhook': {
                    'enabled': False,
                    'url': 'https://hooks.example.com/ansible-pull',
                    'timeout': 30
                }
            }
        }
    
    def setup_logging(self):
        """Setup logging configuration"""
        log_level = getattr(logging, self.config['coordinator']['log_level'].upper())
        log_file = self.config['coordinator']['log_file']
        
        # Ensure log directory exists
        os.makedirs(os.path.dirname(log_file), exist_ok=True)
        
        logging.basicConfig(
            level=log_level,
            format='%(asctime)s [%(levelname)s] %(message)s',
            handlers=[
                logging.FileHandler(log_file),
                logging.StreamHandler()
            ]
        )
        
        self.logger = logging.getLogger(__name__)
    
    def initialize_nodes(self):
        """Initialize node tracking"""
        for node_id, node_config in self.config['nodes'].items():
            self.nodes[node_id] = {
                'config': node_config,
                'status': 'unknown',
                'last_pull': None,
                'last_success': None,
                'last_failure': None,
                'failure_count': 0,
                'total_pulls': 0,
                'success_rate': 0.0,
                'next_pull': None,
                'thread': None,
                'lock': threading.Lock()
            }
        
        self.logger.info(f"Initialized {len(self.nodes)} nodes for monitoring")
    
    def execute_pull(self, node_id):
        """Execute ansible-pull for a specific node"""
        node = self.nodes[node_id]
        node_config = node['config']
        repo_config = self.config['repositories']['primary']
        
        with node['lock']:
            node['status'] = 'pulling'
            pull_start = datetime.now()
            
            self.logger.info(f"Starting pull for node {node_id}")
            
            try:
                # Prepare ansible-pull command
                cmd = [
                    'ansible-pull',
                    '--url', repo_config['url'],
                    '--checkout', repo_config['branch'],
                    '--directory', f"/tmp/pull-{node_id}-{int(time.time())}",
                    '--inventory', repo_config['inventory'],
                    '--limit', node_id,
                    '--vault-password-file', '../secure-pull-repo/.vault_pass',
                    repo_config['playbook']
                ]
                
                # Execute pull
                result = subprocess.run(
                    cmd,
                    capture_output=True,
                    text=True,
                    timeout=300  # 5 minute timeout
                )
                
                pull_end = datetime.now()
                duration = (pull_end - pull_start).total_seconds()
                
                if result.returncode == 0:
                    # Success
                    node['status'] = 'success'
                    node['last_success'] = pull_end
                    node['failure_count'] = 0
                    
                    self.logger.info(f"Pull successful for node {node_id} (duration: {duration:.1f}s)")
                    
                else:
                    # Failure
                    node['status'] = 'failed'
                    node['last_failure'] = pull_end
                    node['failure_count'] += 1
                    
                    self.logger.error(f"Pull failed for node {node_id}: {result.stderr}")
                    
                    # Check if recovery action is needed
                    if node['failure_count'] >= node_config['max_failures']:
                        self.handle_node_failure(node_id)
                
                # Update statistics
                node['last_pull'] = pull_end
                node['total_pulls'] += 1
                
                # Calculate success rate
                if node['total_pulls'] > 0:
                    success_count = node['total_pulls'] - node['failure_count']
                    node['success_rate'] = (success_count / node['total_pulls']) * 100
                
                # Schedule next pull
                next_pull = pull_end + timedelta(seconds=node_config['pull_interval'])
                node['next_pull'] = next_pull
                
            except subprocess.TimeoutExpired:
                node['status'] = 'timeout'
                node['last_failure'] = datetime.now()
                node['failure_count'] += 1
                
                self.logger.error(f"Pull timeout for node {node_id}")
                
            except Exception as e:
                node['status'] = 'error'
                node['last_failure'] = datetime.now()
                node['failure_count'] += 1
                
                self.logger.error(f"Pull error for node {node_id}: {str(e)}")
    
    def handle_node_failure(self, node_id):
        """Handle node failure based on recovery configuration"""
        node = self.nodes[node_id]
        recovery_action = node['config']['recovery_action']
        
        self.logger.warning(f"Node {node_id} has exceeded failure threshold, executing recovery action: {recovery_action}")
        
        if recovery_action == 'restart':
            # Reset failure count and try again
            node['failure_count'] = 0
            self.logger.info(f"Reset failure count for node {node_id}")
            
        elif recovery_action == 'alert':
            # Send notification
            self.send_notification(
                f"Node {node_id} Failure Alert",
                f"Node {node_id} has failed {node['failure_count']} times. Manual intervention may be required."
            )
            
        elif recovery_action == 'disable':
            # Disable node
            node['status'] = 'disabled'
            self.logger.warning(f"Node {node_id} has been disabled due to repeated failures")
    
    def send_notification(self, subject, message):
        """Send notification about node status"""
        if not self.config['notifications']['enabled']:
            return
        
        self.logger.info(f"Notification: {subject} - {message}")
        
        # In a real implementation, this would send email, webhook, etc.
        notification_data = {
            'timestamp': datetime.now().isoformat(),
            'subject': subject,
            'message': message,
            'coordinator': self.config['coordinator']['name']
        }
        
        # Write notification to file for lab purposes
        notification_file = '/var/log/ansible-pull/notifications.log'
        os.makedirs(os.path.dirname(notification_file), exist_ok=True)
        
        with open(notification_file, 'a') as f:
            f.write(json.dumps(notification_data) + '\n')
    
    def node_scheduler(self, node_id):
        """Scheduler thread for individual node"""
        node = self.nodes[node_id]
        
        while self.monitoring_active:
            try:
                current_time = datetime.now()
                
                # Check if it's time for next pull
                if node['next_pull'] is None or current_time >= node['next_pull']:
                    if node['status'] != 'disabled':
                        self.execute_pull(node_id)
                
                # Sleep for a short interval
                time.sleep(10)
                
            except Exception as e:
                self.logger.error(f"Scheduler error for node {node_id}: {str(e)}")
                time.sleep(60)  # Wait longer on error
    
    def start_monitoring(self):
        """Start monitoring all nodes"""
        self.monitoring_active = True
        self.initialize_nodes()
        
        # Start scheduler threads for each node
        for node_id in self.nodes:
            thread = threading.Thread(
                target=self.node_scheduler,
                args=(node_id,),
                daemon=True,
                name=f"scheduler-{node_id}"
            )
            thread.start()
            self.nodes[node_id]['thread'] = thread
        
        self.logger.info("Started monitoring for all nodes")
        
        # Start status reporting thread
        status_thread = threading.Thread(
            target=self.status_reporter,
            daemon=True,
            name="status-reporter"
        )
        status_thread.start()
    
    def stop_monitoring(self):
        """Stop monitoring all nodes"""
        self.monitoring_active = False
        self.logger.info("Stopped monitoring")
    
    def status_reporter(self):
        """Generate periodic status reports"""
        while self.monitoring_active:
            try:
                self.generate_status_report()
                time.sleep(self.config['coordinator']['monitoring_interval'])
            except Exception as e:
                self.logger.error(f"Status reporter error: {str(e)}")
                time.sleep(60)
    
    def generate_status_report(self):
        """Generate comprehensive status report"""
        report = {
            'timestamp': datetime.now().isoformat(),
            'coordinator': self.config['coordinator']['name'],
            'total_nodes': len(self.nodes),
            'active_nodes': len([n for n in self.nodes.values() if n['status'] != 'disabled']),
            'nodes': {}
        }
        
        for node_id, node in self.nodes.items():
            report['nodes'][node_id] = {
                'status': node['status'],
                'last_pull': node['last_pull'].isoformat() if node['last_pull'] else None,
                'last_success': node['last_success'].isoformat() if node['last_success'] else None,
                'last_failure': node['last_failure'].isoformat() if node['last_failure'] else None,
                'failure_count': node['failure_count'],
                'total_pulls': node['total_pulls'],
                'success_rate': round(node['success_rate'], 1),
                'next_pull': node['next_pull'].isoformat() if node['next_pull'] else None,
                'priority': node['config']['priority']
            }
        
        # Write status report
        status_file = self.config['coordinator']['status_file']
        os.makedirs(os.path.dirname(status_file), exist_ok=True)
        
        with open(status_file, 'w') as f:
            json.dump(report, f, indent=2)
        
        # Log summary
        active_count = report['active_nodes']
        total_count = report['total_nodes']
        self.logger.info(f"Status report generated: {active_count}/{total_count} nodes active")
    
    def get_status(self):
        """Get current status of all nodes"""
        status = {}
        for node_id, node in self.nodes.items():
            with node['lock']:
                status[node_id] = {
                    'status': node['status'],
                    'last_pull': node['last_pull'],
                    'failure_count': node['failure_count'],
                    'success_rate': node['success_rate']
                }
        return status

def main():
    import sys
    import signal
    
    # Create coordinator configuration
    config = {
        'coordinator': {
            'name': 'ansible-pull-coordinator',
            'log_level': 'INFO',
            'log_file': '/var/log/ansible-pull/coordinator.log',
            'status_file': '/var/log/ansible-pull/coordinator-status.json',
            'monitoring_interval': 30,
            'failure_threshold': 3,
            'recovery_enabled': True
        },
        'repositories': {
            'primary': {
                'url': '../secure-pull-repo.git',
                'branch': 'main',
                'playbook': 'playbooks/secure-site.yml',
                'inventory': 'inventory/hosts'
            }
        },
        'nodes': {
            'node1': {
                'hostname': 'localhost',
                'connection': 'local',
                'pull_interval': 120,  # 2 minutes for demo
                'max_failures': 3,
                'recovery_action': 'restart',
                'priority': 'high'
            },
            'node2': {
                'hostname': 'localhost',
                'connection': 'local',
                'pull_interval': 180,  # 3 minutes for demo
                'max_failures': 2,
                'recovery_action': 'alert',
                'priority': 'medium'
            }
        },
        'notifications': {
            'enabled': True,
            'email': {'enabled': False},
            'webhook': {'enabled': False}
        }
    }
    
    # Write configuration file
    with open('pull-coordinator.yml', 'w') as f:
        yaml.dump(config, f, default_flow_style=False)
    
    # Create and start coordinator
    coordinator = PullNodeManager('pull-coordinator.yml')
    
    def signal_handler(signum, frame):
        print("\nShutting down coordinator...")
        coordinator.stop_monitoring()
        sys.exit(0)
    
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)
    
    print("Starting distributed pull coordinator...")
    print("Press Ctrl+C to stop")
    
    coordinator.start_monitoring()
    
    # Keep main thread alive
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        coordinator.stop_monitoring()

if __name__ == "__main__":
    main()
EOF

chmod +x distributed-pull-coordinator.py
```

Create monitoring dashboard:

```bash
# Create monitoring dashboard script
cat > pull-monitoring-dashboard.py << 'EOF'
#!/usr/bin/env python3
"""
Ansible Pull Monitoring Dashboard
Provides real-time monitoring and reporting for distributed pull operations
"""

import json
import time
import os
import sys
from datetime import datetime, timedelta
import subprocess

class PullMonitoringDashboard:
    def __init__(self):
        self.status_file = '/var/log/ansible-pull/coordinator-status.json'
        self.notifications_file = '/var/log/ansible-pull/notifications.log'
        
    def clear_screen(self):
        """Clear terminal screen"""
        os.system('clear' if os.name == 'posix' else 'cls')
    
    def load_status(self):
        """Load current status from coordinator"""
        try:
            if os.path.exists(self.status_file):
                with open(self.status_file, 'r') as f:
                    return json.load(f)
            return None
        except Exception as e:
            return {'error': str(e)}
    
    def load_recent_notifications(self, limit=5):
        """Load recent notifications"""
        notifications = []
        try:
            if os.path.exists(self.notifications_file):
                with open(self.notifications_file, 'r') as f:
                    lines = f.readlines()
                    for line in lines[-limit:]:
                        try:
                            notifications.append(json.loads(line.strip()))
                        except:
                            continue
            return notifications
        except Exception:
            return []
    
    def format_timestamp(self, timestamp_str):
        """Format timestamp for display"""
        if not timestamp_str:
            return "Never"
        
        try:
            dt = datetime.fromisoformat(timestamp_str.replace('Z', '+00:00'))
            now = datetime.now(dt.tzinfo)
            diff = now - dt
            
            if diff.total_seconds() < 60:
                return f"{int(diff.total_seconds())}s ago"
            elif diff.total_seconds() < 3600:
                return f"{int(diff.total_seconds() / 60)}m ago"
            elif diff.total_seconds() < 86400:
                return f"{int(diff.total_seconds() / 3600)}h ago"
            else:
                return f"{int(diff.total_seconds() / 86400)}d ago"
        except:
            return timestamp_str
    
    def get_status_color(self, status):
        """Get color code for status"""
        colors = {
            'success': '\033[92m',    # Green
            'failed': '\033[91m',     # Red
            'pulling': '\033[93m',    # Yellow
            'timeout': '\033[95m',    # Magenta
            'error': '\033[91m',      # Red
            'disabled': '\033[90m',   # Gray
            'unknown': '\033[37m'     # White
        }
        return colors.get(status, '\033[37m')
    
    def display_dashboard(self):
        """Display the monitoring dashboard"""
        self.clear_screen()
        
        print("=" * 80)
        print("🔄 ANSIBLE PULL MONITORING DASHBOARD")
        print("=" * 80)
        print(f"Last Updated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print()
        
        # Load status
        status = self.load_status()
        
        if not status:
            print("❌ No status data available")
            return
        
        if 'error' in status:
            print(f"❌ Error loading status: {status['error']}")
            return
        
        # Coordinator status
        print("📊 COORDINATOR STATUS")
        print("-" * 40)
        print(f"Name: {status.get('coordinator', 'Unknown')}")
        print(f"Total Nodes: {status.get('total_nodes', 0)}")
        print(f"Active Nodes: {status.get('active_nodes', 0)}")
        print(f"Report Time: {self.format_timestamp(status.get('timestamp'))}")
        print()
        
        # Node status
        print("🖥️  NODE STATUS")
        print("-" * 80)
        print(f"{'Node':<10} {'Status':<12} {'Last Pull':<15} {'Success Rate':<12} {'Failures':<10} {'Priority':<8}")
        print("-" * 80)
        
        nodes = status.get('nodes', {})
        for node_id, node_info in nodes.items():
            status_text = node_info.get('status', 'unknown')
            color = self.get_status_color(status_text)
            reset_color = '\033[0m'
            
            last_pull = self.format_timestamp(node_info.get('last_pull'))
            success_rate = f"{node_info.get('success_rate', 0):.1f}%"
            failure_count = node_info.get('failure_count', 0)
            priority = node_info.get('priority', 'unknown')
            
            print(f"{node_id:<10} {color}{status_text:<12}{reset_color} {last_pull:<15} {success_rate:<12} {failure_count:<10} {priority:<8}")
        
        print()
        
        # Recent activity
        print("📋 RECENT ACTIVITY")
        print("-" * 40)
        
        # Show next scheduled pulls
        next_pulls = []
        for node_id, node_info in nodes.items():
            next_pull = node_info.get('next_pull')
            if next_pull:
                next_pulls.append((node_id, next_pull))
        
        if next_pulls:
            next_pulls.sort(key=lambda x: x[1])
            print("Next scheduled pulls:")
            for node_id, next_pull in next_pulls[:3]:
                print(f"  • {node_id}: {self.format_timestamp(next_pull)}")
        
        print()
        
        # Recent notifications
        notifications = self.load_recent_notifications(3)
        if notifications:
            print("🔔 RECENT NOTIFICATIONS")
            print("-" * 40)
            for notif in notifications[-3:]:
                timestamp = self.format_timestamp(notif.get('timestamp'))
                subject = notif.get('subject', 'Unknown')
                print(f"  • {timestamp}: {subject}")
        
        print()
        
        # System health indicators
        print("💚 SYSTEM HEALTH")
        print("-" * 40)
        
        # Calculate overall health
        total_nodes = status.get('total_nodes', 0)
        active_nodes = status.get('active_nodes', 0)
        
        if total_nodes > 0:
            health_percentage = (active_nodes / total_nodes) * 100
            
            if health_percentage >= 90:
                health_status = "🟢 Excellent"
            elif health_percentage >= 75:
                health_status = "🟡 Good"
            elif health_percentage >= 50:
                health_status = "🟠 Warning"
            else:
                health_status = "🔴 Critical"
            
            print(f"Overall Health: {health_status} ({health_percentage:.1f}%)")
        
        # Check for recent failures
        recent_failures = sum(1 for node in nodes.values() if node.get('failure_count', 0) > 0)
        if recent_failures > 0:
            print(f"Nodes with Failures: {recent_failures}")
        
        # Check for disabled nodes
        disabled_nodes = sum(1 for node in nodes.values() if node.get('status') == 'disabled')
        if disabled_nodes > 0:
            print(f"Disabled Nodes: {disabled_nodes}")
        
        print()
        print("Press Ctrl+C to exit | Refreshing every 10 seconds...")
    
    def run(self):
        """Run the monitoring dashboard"""
        try:
            while True:
                self.display_dashboard()
                time.sleep(10)
        except KeyboardInterrupt:
            self.clear_screen()
            print("Dashboard stopped.")
            sys.exit(0)

if __name__ == "__main__":
    dashboard = PullMonitoringDashboard()
    dashboard.run()
EOF

chmod +x pull-monitoring-dashboard.py
```

Create comprehensive test:

```yaml
# Create test-distributed-pull.yml
cat > test-distributed-pull.yml << 'EOF'
---
- name: Test distributed pull architecture and monitoring
  hosts: pull_masters
  gather_facts: yes
  
  tasks:
    - name: Create monitoring directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /var/log/ansible-pull/secure
        - /opt/secure-app/config
        - /opt/secure-app/certs
    
    - name: Start distributed pull coordinator (background)
      shell: |
        cd ~/ansible-labs/module-07/lab-7.2
        nohup python3 distributed-pull-coordinator.py > /tmp/coordinator.log 2>&1 &
        echo $! > /tmp/coordinator.pid
      register: coordinator_start
      changed_when: true
    
    - name: Wait for coordinator to initialize
      pause:
        seconds: 15
    
    - name: Check coordinator status
      stat:
        path: /var/log/ansible-pull/coordinator-status.json
      register: coordinator_status_file
    
    - name: Display coordinator status
      block:
        - name: Read coordinator status
          slurp:
            src: /var/log/ansible-pull/coordinator-status.json
          register: coordinator_status_content
          when: coordinator_status_file.stat.exists
        
        - name: Show coordinator status
          debug:
            msg: "{{ coordinator_status_content.content | b64decode | from_json }}"
          when: coordinator_status_file.stat.exists
      when: coordinator_status_file.stat.exists
    
    - name: Test secure pull execution
      shell: |
        cd ~/ansible-labs/module-07/lab-7.2
        ./secure-pull.sh
      register: secure_pull_result
      failed_when: false
      changed_when: false
    
    - name: Check secure pull results
      stat:
        path: "{{ item }}"
      register: secure_pull_files
      loop:
        - /var/log/ansible-pull/secure/status.json
        - /opt/secure-app/config/app.conf
        - /opt/secure-app/config/external-services.yml
    
    - name: Display secure pull results
      debug:
        msg: |
          Secure Pull Results:
          - Execution Status: {{ 'SUCCESS' if secure_pull_result.rc == 0 else 'FAILED' }}
          - Status File: {{ 'EXISTS' if secure_pull_files.results[0].stat.exists else 'MISSING' }}
          - App Config: {{ 'EXISTS' if secure_pull_files.results[1].stat.exists else 'MISSING' }}
          - External Services Config: {{ 'EXISTS' if secure_pull_files.results[2].stat.exists else 'MISSING' }}
    
    - name: Run monitoring dashboard (brief demo)
      shell: |
        cd ~/ansible-labs/module-07/lab-7.2
        timeout 5 python3 pull-monitoring-dashboard.py || true
      register: dashboard_demo
      changed_when: false
    
    - name: Stop coordinator
      shell: |
        if [[ -f /tmp/coordinator.pid ]]; then
          kill $(cat /tmp/coordinator.pid) 2>/dev/null || true
          rm -f /tmp/coordinator.pid
        fi
      changed_when: false
    
    - name: Display distributed pull test summary
      debug:
        msg: |
          Distributed Pull Architecture Test Summary:
          
          Coordinator:
          - Started: {{ 'SUCCESS' if coordinator_start.rc == 0 else 'FAILED' }}
          - Status File: {{ 'CREATED' if coordinator_status_file.stat.exists else 'NOT CREATED' }}
          
          Secure Pull:
          - Execution: {{ 'SUCCESS' if secure_pull_result.rc == 0 else 'FAILED' }}
          - Configuration Files: {{ secure_pull_files.results | selectattr('stat.exists') | list | length }}/{{ secure_pull_files.results | length }} created
          
          Monitoring:
          - Dashboard: Functional
          - Logging: Active
          - Status Reporting: Enabled
          
          Features Demonstrated:
          - Distributed node management
          - Centralized monitoring and reporting
          - Secure pull with encrypted variables
          - Failure detection and recovery
          - Real-time status dashboard
          - Automated scheduling and coordination
EOF

# Run distributed pull test
ansible-playbook -i inventory.ini test-distributed-pull.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Check distributed pull results
echo "=== Distributed Pull Results ==="

# Check coordinator logs
if [[ -f "/var/log/ansible-pull/coordinator.log" ]]; then
    echo "Coordinator log (last 10 lines):"
    tail -10 /var/log/ansible-pull/coordinator.log
fi

# Check secure pull status
if [[ -f "/var/log/ansible-pull/secure/status.json" ]]; then
    echo "Secure pull status:"
    cat /var/log/ansible-pull/secure/status.json | python3 -m json.tool
fi

# Check notifications
if [[ -f "/var/log/ansible-pull/notifications.log" ]]; then
    echo "Recent notifications:"
    tail -5 /var/log/ansible-pull/notifications.log
fi

# Check created configurations
echo "Created secure configurations:"
ls -la /opt/secure-app/config/ 2>/dev/null || echo "No configurations found"
```

### 2. Discussion Points
- How do you implement secure authentication for distributed pull operations?
- What monitoring and alerting strategies work best for pull-based systems?
- How do you handle network partitions and connectivity issues in pull architectures?
- What are the trade-offs between centralized coordination and fully autonomous nodes?

### 3. Clean Up
```bash
# Clean up test artifacts
sudo rm -rf /var/log/ansible-pull /opt/secure-app 2>/dev/null || true
rm -f /tmp/coordinator.pid /tmp/coordinator.log 2>/dev/null || true
```

## Key Takeaways
- Secure pull operations require proper credential management and encryption
- Distributed architectures benefit from centralized monitoring and coordination
- Failure recovery mechanisms are essential for reliable pull-based automation
- Real-time monitoring enables proactive issue resolution
- Proper logging and status tracking facilitate troubleshooting
- Scalable pull architectures require careful design of scheduling and coordination

## Module 7 Summary

### What We Covered
1. **Pull Fundamentals** - Git-based configuration management, basic pull operations, and automated scheduling
2. **Advanced Pull Strategies** - Secure configurations, distributed architectures, monitoring, and failure recovery

### Key Skills Developed
- Implementing pull-based configuration management with Git repositories
- Configuring secure ansible-pull operations with encrypted variables
- Building distributed pull architectures with centralized coordination
- Setting up comprehensive monitoring and alerting for pull operations
- Designing failure recovery and self-healing mechanisms

### Next Steps
You now have comprehensive knowledge of ansible-pull methodologies for decentralized configuration management. These skills enable you to implement scalable, secure, and reliable pull-based automation suitable for large, distributed enterprise environments.

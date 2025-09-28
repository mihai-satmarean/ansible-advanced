# Lab 7.1: Ansible Pull Fundamentals

## Objective
Master the fundamentals of Ansible Pull methodology by setting up Git-based configuration management, implementing basic pull operations, and establishing automated scheduling for decentralized configuration management.

## Duration
25 minutes

## Prerequisites
- Basic Git knowledge and repository access
- Understanding of SSH key authentication
- Familiarity with cron scheduling
- Basic Ansible playbook concepts

## Lab Setup

```bash
cd ~/ansible-labs
mkdir -p module-07
cd module-07

# Create inventory for pull testing
cat > inventory.ini << 'EOF'
[pull_nodes]
node1 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Git Repository Setup for Pull Configuration (8 minutes)

### Task: Create and configure a Git repository for ansible-pull operations

Set up local Git repository:

```bash
# Create pull configuration repository
mkdir -p ansible-pull-repo
cd ansible-pull-repo

# Initialize Git repository
git init
git config user.name "Ansible Pull Lab"
git config user.email "lab@example.com"

# Create basic directory structure
mkdir -p {playbooks,inventory,group_vars,host_vars,roles}
```

Create pull-ready playbook:

```yaml
# Create playbooks/site.yml
cat > playbooks/site.yml << 'EOF'
---
- name: Ansible Pull Configuration Management
  hosts: all
  become: yes
  gather_facts: yes
  
  vars:
    pull_timestamp: "{{ ansible_date_time.iso8601 }}"
    pull_user: "{{ ansible_user_id }}"
    config_version: "1.0.0"
    
  tasks:
    - name: Create ansible-pull log directory
      file:
        path: /var/log/ansible-pull
        state: directory
        mode: '0755'
        owner: root
        group: root
    
    - name: Log pull execution
      lineinfile:
        path: /var/log/ansible-pull/execution.log
        line: "{{ pull_timestamp }} - Pull executed by {{ pull_user }} - Config version {{ config_version }}"
        create: yes
        mode: '0644'
    
    - name: Ensure required packages are installed
      package:
        name:
          - curl
          - wget
          - git
        state: present
    
    - name: Create application directory
      file:
        path: /opt/myapp
        state: directory
        mode: '0755'
        owner: root
        group: root
    
    - name: Deploy application configuration
      template:
        src: app-config.j2
        dest: /opt/myapp/config.yml
        mode: '0644'
        owner: root
        group: root
      notify: restart application service
    
    - name: Ensure application service is running
      systemd:
        name: myapp
        state: started
        enabled: yes
      failed_when: false  # Service might not exist in lab environment
    
    - name: Update system information file
      copy:
        content: |
          System Information (Updated via Ansible Pull)
          ============================================
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Architecture: {{ ansible_architecture }}
          Last Update: {{ pull_timestamp }}
          Updated By: {{ pull_user }}
          Configuration Version: {{ config_version }}
          
          Installed Packages:
          - curl: {{ ansible_facts.packages.curl[0].version if ansible_facts.packages.curl is defined else 'Not found' }}
          - wget: {{ ansible_facts.packages.wget[0].version if ansible_facts.packages.wget is defined else 'Not found' }}
          - git: {{ ansible_facts.packages.git[0].version if ansible_facts.packages.git is defined else 'Not found' }}
          
          Network Information:
          - Default IPv4: {{ ansible_default_ipv4.address | default('N/A') }}
          - DNS Servers: {{ ansible_dns.nameservers | join(', ') if ansible_dns.nameservers is defined else 'N/A' }}
          
          Disk Usage:
          {% for mount in ansible_mounts %}
          - {{ mount.mount }}: {{ ((mount.size_total - mount.size_available) / mount.size_total * 100) | round(1) }}% used
          {% endfor %}
        dest: /opt/myapp/system-info.txt
        mode: '0644'
    
    - name: Create pull status file
      copy:
        content: |
          {
            "last_pull": "{{ pull_timestamp }}",
            "pull_user": "{{ pull_user }}",
            "config_version": "{{ config_version }}",
            "hostname": "{{ ansible_hostname }}",
            "status": "success",
            "playbook": "site.yml",
            "tasks_completed": {{ ansible_play_batch | length }},
            "facts_gathered": true
          }
        dest: /var/log/ansible-pull/status.json
        mode: '0644'
  
  handlers:
    - name: restart application service
      systemd:
        name: myapp
        state: restarted
      failed_when: false
EOF
```

Create template for application configuration:

```yaml
# Create templates directory and template
mkdir -p templates
cat > templates/app-config.j2 << 'EOF'
# Application Configuration
# Generated by Ansible Pull on {{ ansible_date_time.iso8601 }}
# Managed by: {{ ansible_user_id }}

application:
  name: "MyApp"
  version: "{{ config_version }}"
  environment: "{{ app_environment | default('production') }}"
  
server:
  host: "{{ ansible_default_ipv4.address | default('localhost') }}"
  port: {{ app_port | default(8080) }}
  workers: {{ ansible_processor_vcpus | default(2) }}
  
database:
  host: "{{ db_host | default('localhost') }}"
  port: {{ db_port | default(5432) }}
  name: "{{ db_name | default('myapp') }}"
  user: "{{ db_user | default('myapp') }}"
  
logging:
  level: "{{ log_level | default('INFO') }}"
  file: "/var/log/myapp/application.log"
  max_size: "{{ log_max_size | default('100MB') }}"
  
monitoring:
  enabled: {{ monitoring_enabled | default(true) | lower }}
  endpoint: "{{ monitoring_endpoint | default('/metrics') }}"
  interval: {{ monitoring_interval | default(30) }}
  
security:
  ssl_enabled: {{ ssl_enabled | default(false) | lower }}
  ssl_cert: "{{ ssl_cert_path | default('/etc/ssl/certs/myapp.crt') }}"
  ssl_key: "{{ ssl_key_path | default('/etc/ssl/private/myapp.key') }}"
  
features:
  feature_flags:
    new_ui: {{ feature_new_ui | default(false) | lower }}
    advanced_analytics: {{ feature_analytics | default(true) | lower }}
    api_v2: {{ feature_api_v2 | default(false) | lower }}
  
system_info:
  hostname: "{{ ansible_hostname }}"
  os: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
  architecture: "{{ ansible_architecture }}"
  memory_gb: {{ (ansible_memtotal_mb / 1024) | round(1) }}
  cpu_cores: {{ ansible_processor_vcpus }}
  last_updated: "{{ ansible_date_time.iso8601 }}"
EOF
```

Create inventory and variables:

```yaml
# Create inventory/hosts
cat > inventory/hosts << 'EOF'
[web_servers]
web1 ansible_host=localhost ansible_connection=local
web2 ansible_host=localhost ansible_connection=local

[database_servers]
db1 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
config_version=1.0.0
app_environment=production
EOF

# Create group variables
cat > group_vars/web_servers.yml << 'EOF'
---
app_port: 8080
log_level: "INFO"
monitoring_enabled: true
ssl_enabled: false
feature_new_ui: true
feature_api_v2: false
EOF

cat > group_vars/database_servers.yml << 'EOF'
---
db_host: "localhost"
db_port: 5432
db_name: "myapp_prod"
db_user: "myapp_user"
monitoring_enabled: true
log_level: "DEBUG"
EOF

# Create host-specific variables
mkdir -p host_vars
cat > host_vars/web1.yml << 'EOF'
---
app_port: 8081
feature_analytics: true
monitoring_interval: 15
EOF

cat > host_vars/db1.yml << 'EOF'
---
db_name: "myapp_primary"
log_max_size: "500MB"
monitoring_interval: 60
EOF
```

Commit to Git repository:

```bash
# Add and commit files
git add .
git commit -m "Initial ansible-pull configuration

- Added site.yml playbook for basic system configuration
- Created application configuration template
- Set up inventory and group/host variables
- Configured logging and monitoring setup"

# Create a simple remote simulation (for lab purposes)
cd ..
git clone --bare ansible-pull-repo ansible-pull-repo.git
cd ansible-pull-repo
git remote add origin ../ansible-pull-repo.git
git push origin main

echo "Git repository setup completed!"
```

## Exercise 2: Basic Ansible Pull Configuration (10 minutes)

### Task: Configure and execute basic ansible-pull operations

Test ansible-pull command:

```bash
# Test basic ansible-pull execution
cd ~/ansible-labs/module-07

# Create test directory for pull operations
mkdir -p pull-test
cd pull-test

# Execute ansible-pull with local repository
ansible-pull \
  --url ../ansible-pull-repo.git \
  --checkout main \
  --directory /tmp/ansible-pull-test \
  --inventory inventory/hosts \
  --limit localhost \
  playbooks/site.yml

echo "Basic pull execution completed!"
```

Create ansible-pull configuration file:

```bash
# Create ansible-pull configuration
cat > ansible-pull.conf << 'EOF'
# Ansible Pull Configuration
# This file contains default settings for ansible-pull operations

[defaults]
# Repository settings
repository_url = ../ansible-pull-repo.git
checkout_branch = main
playbook_path = playbooks/site.yml
inventory_path = inventory/hosts

# Pull settings
pull_directory = /tmp/ansible-pull
clean_repository = yes
force_pull = no

# Execution settings
limit_hosts = localhost
verbosity = 1
check_mode = no

# Logging settings
log_path = /var/log/ansible-pull/ansible-pull.log
syslog_facility = LOG_LOCAL0

# Security settings
vault_password_file = 
private_key_file = 
become = yes
become_method = sudo
EOF
```

Create pull execution script:

```bash
# Create pull execution script
cat > run-ansible-pull.sh << 'EOF'
#!/bin/bash
# Ansible Pull Execution Script
# Executes ansible-pull with proper error handling and logging

set -euo pipefail

# Configuration
REPO_URL="../ansible-pull-repo.git"
BRANCH="main"
PLAYBOOK="playbooks/site.yml"
INVENTORY="inventory/hosts"
PULL_DIR="/tmp/ansible-pull-$(date +%Y%m%d-%H%M%S)"
LOG_FILE="/var/log/ansible-pull/execution-$(date +%Y%m%d).log"
LOCK_FILE="/var/run/ansible-pull.lock"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Logging function
log() {
    local level=$1
    shift
    local message="$*"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    echo -e "${timestamp} [${level}] ${message}" | tee -a "${LOG_FILE}"
}

# Error handling
error_exit() {
    log "ERROR" "$1"
    cleanup
    exit 1
}

# Cleanup function
cleanup() {
    if [[ -f "${LOCK_FILE}" ]]; then
        rm -f "${LOCK_FILE}"
    fi
    
    if [[ -d "${PULL_DIR}" ]]; then
        rm -rf "${PULL_DIR}"
    fi
}

# Signal handlers
trap cleanup EXIT
trap 'error_exit "Script interrupted"' INT TERM

# Main execution
main() {
    log "INFO" "Starting ansible-pull execution"
    
    # Check for lock file
    if [[ -f "${LOCK_FILE}" ]]; then
        error_exit "Another ansible-pull process is already running (lock file exists)"
    fi
    
    # Create lock file
    echo $$ > "${LOCK_FILE}"
    
    # Create log directory
    mkdir -p "$(dirname "${LOG_FILE}")"
    
    # Pre-execution checks
    log "INFO" "Performing pre-execution checks"
    
    if ! command -v ansible-pull >/dev/null 2>&1; then
        error_exit "ansible-pull command not found"
    fi
    
    if ! command -v git >/dev/null 2>&1; then
        error_exit "git command not found"
    fi
    
    # Execute ansible-pull
    log "INFO" "Executing ansible-pull"
    log "INFO" "Repository: ${REPO_URL}"
    log "INFO" "Branch: ${BRANCH}"
    log "INFO" "Playbook: ${PLAYBOOK}"
    log "INFO" "Pull Directory: ${PULL_DIR}"
    
    if ansible-pull \
        --url "${REPO_URL}" \
        --checkout "${BRANCH}" \
        --directory "${PULL_DIR}" \
        --inventory "${INVENTORY}" \
        --limit localhost \
        --verbose \
        "${PLAYBOOK}" 2>&1 | tee -a "${LOG_FILE}"; then
        
        log "INFO" "ansible-pull execution completed successfully"
        
        # Post-execution validation
        if [[ -f "/var/log/ansible-pull/status.json" ]]; then
            log "INFO" "Pull status file created successfully"
            cat "/var/log/ansible-pull/status.json" | tee -a "${LOG_FILE}"
        fi
        
        if [[ -f "/opt/myapp/system-info.txt" ]]; then
            log "INFO" "System information file updated"
        fi
        
    else
        error_exit "ansible-pull execution failed"
    fi
    
    log "INFO" "ansible-pull execution completed"
}

# Execute main function
main "$@"
EOF

chmod +x run-ansible-pull.sh

# Test the pull script
echo "Testing ansible-pull script..."
./run-ansible-pull.sh

echo "Pull script execution completed!"
```

Verify pull results:

```bash
# Check pull execution results
echo "=== Pull Execution Results ==="

# Check log files
if [[ -f "/var/log/ansible-pull/execution.log" ]]; then
    echo "Execution log:"
    tail -5 /var/log/ansible-pull/execution.log
fi

# Check status file
if [[ -f "/var/log/ansible-pull/status.json" ]]; then
    echo "Status file:"
    cat /var/log/ansible-pull/status.json | python3 -m json.tool
fi

# Check system info
if [[ -f "/opt/myapp/system-info.txt" ]]; then
    echo "System information (first 10 lines):"
    head -10 /opt/myapp/system-info.txt
fi

# Check application config
if [[ -f "/opt/myapp/config.yml" ]]; then
    echo "Application configuration (first 15 lines):"
    head -15 /opt/myapp/config.yml
fi
```

## Exercise 3: Automated Scheduling with Cron (7 minutes)

### Task: Set up automated ansible-pull execution using cron scheduling

Create cron-compatible pull script:

```bash
# Create cron-friendly pull script
cat > cron-ansible-pull.sh << 'EOF'
#!/bin/bash
# Cron-compatible Ansible Pull Script
# Designed for automated execution via cron

# Set PATH for cron environment
export PATH="/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/sbin"

# Configuration
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_URL="${SCRIPT_DIR}/../ansible-pull-repo.git"
BRANCH="main"
PLAYBOOK="playbooks/site.yml"
INVENTORY="inventory/hosts"
PULL_DIR="/tmp/ansible-pull-cron-$(date +%Y%m%d-%H%M%S)"
LOG_FILE="/var/log/ansible-pull/cron-$(date +%Y%m%d).log"
LOCK_FILE="/var/run/ansible-pull-cron.lock"

# Ensure log directory exists
mkdir -p "$(dirname "${LOG_FILE}")"

# Logging function
log() {
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "${timestamp} [CRON] $*" >> "${LOG_FILE}"
}

# Check for lock file
if [[ -f "${LOCK_FILE}" ]]; then
    log "Another ansible-pull process is running, exiting"
    exit 0
fi

# Create lock file
echo $$ > "${LOCK_FILE}"

# Cleanup function
cleanup() {
    rm -f "${LOCK_FILE}"
    if [[ -d "${PULL_DIR}" ]]; then
        rm -rf "${PULL_DIR}"
    fi
}

trap cleanup EXIT

# Execute ansible-pull
log "Starting scheduled ansible-pull execution"

if ansible-pull \
    --url "${REPO_URL}" \
    --checkout "${BRANCH}" \
    --directory "${PULL_DIR}" \
    --inventory "${INVENTORY}" \
    --limit localhost \
    --quiet \
    "${PLAYBOOK}" >> "${LOG_FILE}" 2>&1; then
    
    log "Scheduled ansible-pull execution completed successfully"
    
    # Update last successful run timestamp
    echo "$(date '+%Y-%m-%d %H:%M:%S')" > /var/log/ansible-pull/last-success
    
else
    log "Scheduled ansible-pull execution failed with exit code $?"
    
    # Update last failure timestamp
    echo "$(date '+%Y-%m-%d %H:%M:%S')" > /var/log/ansible-pull/last-failure
fi

log "Scheduled execution finished"
EOF

chmod +x cron-ansible-pull.sh
```

Create cron configuration:

```bash
# Create cron configuration
cat > ansible-pull-cron << 'EOF'
# Ansible Pull Cron Configuration
# Executes ansible-pull every 15 minutes

# Set environment variables for cron
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/sbin
MAILTO=admin@example.com

# Execute ansible-pull every 15 minutes
*/15 * * * * /home/$(whoami)/ansible-labs/module-07/pull-test/cron-ansible-pull.sh

# Execute ansible-pull every hour (alternative schedule)
# 0 * * * * /home/$(whoami)/ansible-labs/module-07/pull-test/cron-ansible-pull.sh

# Execute ansible-pull twice daily (alternative schedule)
# 0 6,18 * * * /home/$(whoami)/ansible-labs/module-07/pull-test/cron-ansible-pull.sh

# Execute ansible-pull daily at 2 AM (alternative schedule)
# 0 2 * * * /home/$(whoami)/ansible-labs/module-07/pull-test/cron-ansible-pull.sh
EOF

# Install cron job (commented out for lab safety)
# crontab ansible-pull-cron

echo "Cron configuration created (not installed for lab safety)"
```

Create systemd timer alternative:

```bash
# Create systemd service file
cat > ansible-pull.service << 'EOF'
[Unit]
Description=Ansible Pull Configuration Management
After=network.target

[Service]
Type=oneshot
User=root
Group=root
WorkingDirectory=/home/$(whoami)/ansible-labs/module-07/pull-test
ExecStart=/home/$(whoami)/ansible-labs/module-07/pull-test/cron-ansible-pull.sh
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Create systemd timer file
cat > ansible-pull.timer << 'EOF'
[Unit]
Description=Run Ansible Pull every 15 minutes
Requires=ansible-pull.service

[Timer]
OnCalendar=*:0/15
Persistent=true

[Install]
WantedBy=timers.target
EOF

echo "Systemd timer configuration created"
```

Create monitoring script:

```bash
# Create pull monitoring script
cat > monitor-ansible-pull.sh << 'EOF'
#!/bin/bash
# Ansible Pull Monitoring Script
# Monitors the health and status of ansible-pull operations

# Configuration
LOG_DIR="/var/log/ansible-pull"
STATUS_FILE="${LOG_DIR}/status.json"
LAST_SUCCESS_FILE="${LOG_DIR}/last-success"
LAST_FAILURE_FILE="${LOG_DIR}/last-failure"
EXECUTION_LOG="${LOG_DIR}/execution.log"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo "=== Ansible Pull Status Monitor ==="
echo "Timestamp: $(date)"
echo

# Check if pull is currently running
if [[ -f "/var/run/ansible-pull.lock" ]] || [[ -f "/var/run/ansible-pull-cron.lock" ]]; then
    echo -e "${YELLOW}Status: RUNNING${NC}"
    echo "Lock file detected - ansible-pull is currently executing"
else
    echo -e "${GREEN}Status: IDLE${NC}"
fi

echo

# Check last successful execution
if [[ -f "${LAST_SUCCESS_FILE}" ]]; then
    last_success=$(cat "${LAST_SUCCESS_FILE}")
    echo -e "Last Successful Run: ${GREEN}${last_success}${NC}"
    
    # Calculate time since last success
    if command -v date >/dev/null 2>&1; then
        last_success_epoch=$(date -d "${last_success}" +%s 2>/dev/null || echo "0")
        current_epoch=$(date +%s)
        time_diff=$((current_epoch - last_success_epoch))
        
        if [[ ${time_diff} -gt 3600 ]]; then  # More than 1 hour
            echo -e "${RED}Warning: Last successful run was more than 1 hour ago${NC}"
        elif [[ ${time_diff} -gt 1800 ]]; then  # More than 30 minutes
            echo -e "${YELLOW}Notice: Last successful run was more than 30 minutes ago${NC}"
        fi
    fi
else
    echo -e "Last Successful Run: ${RED}NEVER${NC}"
fi

# Check last failure
if [[ -f "${LAST_FAILURE_FILE}" ]]; then
    last_failure=$(cat "${LAST_FAILURE_FILE}")
    echo -e "Last Failed Run: ${RED}${last_failure}${NC}"
else
    echo "Last Failed Run: NONE"
fi

echo

# Check status file
if [[ -f "${STATUS_FILE}" ]]; then
    echo "=== Last Pull Status ==="
    if command -v python3 >/dev/null 2>&1; then
        python3 -m json.tool "${STATUS_FILE}" 2>/dev/null || cat "${STATUS_FILE}"
    else
        cat "${STATUS_FILE}"
    fi
else
    echo -e "${RED}No status file found${NC}"
fi

echo

# Check recent log entries
if [[ -f "${EXECUTION_LOG}" ]]; then
    echo "=== Recent Log Entries (Last 5) ==="
    tail -5 "${EXECUTION_LOG}"
else
    echo -e "${RED}No execution log found${NC}"
fi

echo

# Check disk space in pull directories
echo "=== Disk Usage ==="
if [[ -d "/tmp" ]]; then
    echo "Temporary pull directories:"
    du -sh /tmp/ansible-pull* 2>/dev/null | head -5 || echo "No pull directories found"
fi

echo

# Check cron configuration
echo "=== Scheduling Status ==="
if crontab -l 2>/dev/null | grep -q ansible-pull; then
    echo -e "${GREEN}Cron job configured${NC}"
    echo "Cron entries:"
    crontab -l 2>/dev/null | grep ansible-pull
else
    echo -e "${YELLOW}No cron job configured${NC}"
fi

# Check systemd timer
if systemctl is-enabled ansible-pull.timer >/dev/null 2>&1; then
    echo -e "${GREEN}Systemd timer enabled${NC}"
    systemctl status ansible-pull.timer --no-pager -l
else
    echo -e "${YELLOW}Systemd timer not enabled${NC}"
fi

echo
echo "=== Monitor Complete ==="
EOF

chmod +x monitor-ansible-pull.sh

# Test monitoring script
echo "Testing monitoring script..."
./monitor-ansible-pull.sh
```

## Verification and Discussion

### 1. Check Results
```bash
# Verify pull setup
echo "=== Ansible Pull Setup Verification ==="

# Check repository structure
echo "Repository structure:"
ls -la ../ansible-pull-repo/

# Check pull execution logs
echo "Pull execution status:"
ls -la /var/log/ansible-pull/ 2>/dev/null || echo "No logs found"

# Check created files
echo "Created application files:"
ls -la /opt/myapp/ 2>/dev/null || echo "No application files found"

# Test pull script
echo "Testing pull script execution:"
./run-ansible-pull.sh

# Run monitoring
echo "Current pull status:"
./monitor-ansible-pull.sh
```

### 2. Discussion Points
- When would you choose pull vs push model for configuration management?
- How do you handle authentication and security in pull-based systems?
- What are the advantages and disadvantages of scheduled vs event-driven pulls?
- How do you monitor and troubleshoot distributed pull operations?

### 3. Clean Up
```bash
# Clean up test files (optional)
# sudo rm -rf /var/log/ansible-pull /opt/myapp /tmp/ansible-pull*
# crontab -r  # Remove cron jobs if installed
```

## Key Takeaways
- Ansible Pull enables decentralized configuration management
- Git repositories serve as the source of truth for configurations
- Proper scheduling and monitoring are essential for reliable pull operations
- Security considerations include repository access and credential management
- Pull model scales well for large, distributed infrastructures
- Logging and status tracking enable effective troubleshooting

This lab provides hands-on experience with ansible-pull fundamentals, preparing you for implementing pull-based configuration management in enterprise environments.

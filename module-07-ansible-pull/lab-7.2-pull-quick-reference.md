# Lab 7.2: Advanced Pull Strategies (Quick Reference)

## Pattern 1: Secure Pull with Vault

### Basic Secure Setup
```bash
# Create secure repository
mkdir secure-pull-repo && cd secure-pull-repo
git init

# Create vault password file
echo "secure-vault-pass-123" > .vault_pass
chmod 600 .vault_pass

# Create encrypted variables
ansible-vault create group_vars/all.yml --vault-password-file .vault_pass << 'EOF'
---
database_password: "secure_db_pass_456"
api_key: "sk-1234567890abcdef"
service_accounts:
  monitoring:
    username: "monitor_user"
    password: "monitor_pass_789"
EOF
```

### Secure Pull Playbook
```yaml
# playbooks/secure-site.yml
---
- name: Secure Ansible Pull
  hosts: all
  become: yes
  vars_files:
    - ../group_vars/all.yml
  
  tasks:
    - name: Create secure config
      copy:
        content: |
          [database]
          password={{ database_password }}
          
          [api]
          key={{ api_key }}
          
          [monitoring]
          user={{ service_accounts.monitoring.username }}
          pass={{ service_accounts.monitoring.password }}
        dest: /etc/myapp/secure.conf
        mode: '0600'
    
    - name: Log secure pull
      lineinfile:
        path: /var/log/ansible-pull.log
        line: "{{ ansible_date_time.iso8601 }} - Secure pull completed"
        create: yes
```

### Execute Secure Pull
```bash
# Commit and test
git add . && git commit -m "Secure pull setup"

# Run secure ansible-pull
ansible-pull \
  --url /path/to/secure-pull-repo \
  --vault-password-file .vault_pass \
  --directory /tmp/secure-pull \
  playbooks/secure-site.yml
```

## Pattern 2: Automated Pull with Scheduling

### Systemd Service Setup
```bash
# Create systemd service
sudo tee /etc/systemd/system/ansible-pull.service << 'EOF'
[Unit]
Description=Ansible Pull Configuration Management
After=network.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/bin/ansible-pull -U https://github.com/company/config.git site.yml
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Create systemd timer
sudo tee /etc/systemd/system/ansible-pull.timer << 'EOF'
[Unit]
Description=Run Ansible Pull every 30 minutes
Requires=ansible-pull.service

[Timer]
OnCalendar=*:0/30
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Enable and start
sudo systemctl enable ansible-pull.timer
sudo systemctl start ansible-pull.timer
```

### Cron Alternative
```bash
# Add to crontab
crontab -e

# Run every 30 minutes
*/30 * * * * /usr/local/bin/ansible-pull -U https://github.com/company/config.git site.yml >> /var/log/ansible-pull.log 2>&1

# Run daily at 2 AM with cleanup
0 2 * * * /usr/local/bin/ansible-pull -U https://github.com/company/config.git --clean site.yml
```

## Pattern 3: Pull with Monitoring

### Simple Monitoring Script
```bash
# Create monitoring script
cat > monitor-pull.sh << 'EOF'
#!/bin/bash
# Simple ansible-pull monitoring

LOG_FILE="/var/log/ansible-pull.log"
STATUS_FILE="/var/log/ansible-pull-status.json"

# Check last pull
if [[ -f "$LOG_FILE" ]]; then
    LAST_PULL=$(tail -1 "$LOG_FILE" | cut -d' ' -f1-2)
    echo "Last pull: $LAST_PULL"
else
    echo "No pull log found"
fi

# Create status
cat > "$STATUS_FILE" << EOF
{
  "last_check": "$(date -Iseconds)",
  "log_exists": $([ -f "$LOG_FILE" ] && echo "true" || echo "false"),
  "last_pull": "$LAST_PULL"
}
EOF

echo "Status written to $STATUS_FILE"
EOF

chmod +x monitor-pull.sh
```

### Pull with Health Check
```yaml
# Enhanced playbook with health reporting
---
- name: Ansible Pull with Health Check
  hosts: all
  
  tasks:
    - name: Run main configuration
      include_tasks: main-config.yml
      
    - name: Health check
      uri:
        url: "http://localhost:8080/health"
        method: GET
      register: health_check
      failed_when: false
      
    - name: Report status
      copy:
        content: |
          {
            "timestamp": "{{ ansible_date_time.iso8601 }}",
            "hostname": "{{ ansible_hostname }}",
            "pull_status": "success",
            "health_check": "{{ 'passed' if health_check.status == 200 else 'failed' }}",
            "config_applied": true
          }
        dest: /var/log/ansible-pull-status.json
```

## Pattern 4: Multi-Environment Pull

### Environment-Specific Branches
```bash
# Repository structure
# main branch - production config
# staging branch - staging config
# dev branch - development config

# Pull production config
ansible-pull -U https://github.com/company/config.git -C main site.yml

# Pull staging config
ansible-pull -U https://github.com/company/config.git -C staging site.yml

# Pull with environment-specific inventory
ansible-pull -U https://github.com/company/config.git -C production -i inventory/prod site.yml
```

### Environment Detection
```yaml
# Playbook with environment detection
---
- name: Multi-Environment Pull
  hosts: all
  vars:
    environment: "{{ ansible_hostname.split('-')[1] | default('prod') }}"
  
  tasks:
    - name: Load environment-specific variables
      include_vars: "environments/{{ environment }}.yml"
      
    - name: Deploy environment-specific config
      template:
        src: app-config.j2
        dest: /etc/myapp/config-{{ environment }}.yml
        
    - name: Set environment marker
      copy:
        content: "{{ environment }}"
        dest: /etc/myapp/environment
```

## Best Practices Summary

### Security:
- Use Ansible Vault for sensitive data
- Secure Git repository access (SSH keys/tokens)
- Limit repository access permissions
- Regular credential rotation

### Reliability:
- Implement proper error handling
- Use specific Git branches/tags
- Test playbooks before deployment
- Monitor pull execution

### Performance:
- Use shallow Git clones (`--depth 1`)
- Schedule pulls during off-peak hours
- Clean up old pull directories
- Cache Git repositories locally

### Monitoring:
- Log all pull executions
- Create status/health files
- Set up alerting for failures
- Track pull frequency and success rates

## Common ansible-pull Options

```bash
# Basic options
ansible-pull -U <repo-url> [playbook]

# Useful flags
--checkout BRANCH          # Specific branch/tag
--directory DIR            # Clone directory
--inventory FILE           # Inventory file
--limit HOSTS             # Limit to specific hosts
--vault-password-file     # Vault password file
--clean                   # Clean repository before pull
--full                    # Full clone (not shallow)
--sleep SECONDS           # Random sleep before execution
--tags TAGS               # Run specific tags only
--skip-tags TAGS          # Skip specific tags
-v, --verbose             # Verbose output
```

## Troubleshooting

### Common Issues:
1. **Git authentication failures** - Check SSH keys/credentials
2. **Vault password errors** - Verify vault password file permissions
3. **Playbook syntax errors** - Test playbooks before committing
4. **Network connectivity** - Ensure Git repository access
5. **Permission issues** - Check file/directory permissions

### Debug Commands:
```bash
# Test Git access
git clone <repo-url> /tmp/test-clone

# Test ansible-pull with verbose output
ansible-pull -U <repo-url> -v playbook.yml

# Check system logs
journalctl -u ansible-pull.service
tail -f /var/log/ansible-pull.log
```

## Next Steps

### Real-World Implementation:
- Set up Git repository with proper branching strategy
- Implement CI/CD pipeline for playbook testing
- Configure monitoring and alerting
- Establish change management processes

### Advanced Topics:
- Dynamic inventory with pull model
- Integration with configuration management databases
- Automated rollback mechanisms
- Performance optimization for large-scale deployments

---
# Lab 7.1: Ansible Pull Demo (Interactive)

## Objective
Understand Ansible Pull concepts through interactive demonstration and practical examples suitable for all skill levels.

## Duration
45 minutes (interactive demo)

## Format
**Interactive demonstration** - Instructor-led with participant Q&A

## Prerequisites
- Basic understanding of Ansible playbooks
- Familiarity with Git repositories
- Understanding of push vs pull concepts

## Demo Setup

```bash
# Simple setup for demonstration
mkdir -p ~/ansible-pull-demo
cd ~/ansible-pull-demo

# Create basic inventory
cat > inventory.ini << 'EOF'
[demo_hosts]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Demo 1: Push vs Pull Model Concepts (15 minutes)

### Traditional Ansible (Push Model)
**How it works:**
```
Ansible Control Node → SSH → Target Hosts
     (playbook)              (execution)
```

**Characteristics:**
- Control node initiates all actions
- Requires SSH access to all targets
- Centralized execution and scheduling
- Real-time execution and feedback

### Ansible Pull Model
**How it works:**
```
Target Hosts → Git Repository → ansible-pull → Local Execution
   (pulls)      (playbook)        (command)      (on target)
```

**Characteristics:**
- Target nodes initiate their own updates
- No SSH required from control node
- Decentralized execution
- Scheduled via cron/systemd on targets

### When to Use Each Model

**Push Model (Default) - Best for:**
- Interactive deployments
- Real-time orchestration
- Small to medium environments
- When you need immediate feedback
- Complex multi-host coordination

**Pull Model - Best for:**
- Large-scale environments (1000+ hosts)
- Edge devices with limited connectivity
- Self-healing infrastructure
- Environments with strict network security
- Autonomous system updates

### Key Differences Discussion:
- **Control**: Push = centralized, Pull = distributed
- **Connectivity**: Push = SSH required, Pull = Git access required
- **Scaling**: Push = limited by control node, Pull = scales horizontally
- **Security**: Push = SSH keys, Pull = Git authentication
- **Feedback**: Push = immediate, Pull = delayed/logged

## Demo 2: Basic Ansible Pull Demonstration (25 minutes)

### Step 1: Create a Simple Git Repository
```bash
# Create a local Git repository for demo
mkdir -p ansible-pull-repo
cd ansible-pull-repo
git init

# Create a simple playbook
cat > site.yml << 'EOF'
---
- name: Ansible Pull Demo Playbook
  hosts: all
  gather_facts: yes
  
  tasks:
    - name: Create demo directory
      file:
        path: /tmp/ansible-pull-demo
        state: directory
        mode: '0755'
    
    - name: Create system info file
      copy:
        content: |
          Ansible Pull Demo Results
          ========================
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Architecture: {{ ansible_architecture }}
          Memory: {{ ansible_memtotal_mb }}MB
          CPU Cores: {{ ansible_processor_vcpus }}
          
          Pull Information:
          Executed at: {{ ansible_date_time.iso8601 }}
          Executed by: ansible-pull
          Working directory: {{ ansible_env.PWD }}
          
          Network Info:
          Default IP: {{ ansible_default_ipv4.address | default('N/A') }}
        dest: /tmp/ansible-pull-demo/system-info.txt
        mode: '0644'
    
    - name: Log pull execution
      lineinfile:
        path: /tmp/ansible-pull-demo/pull-history.log
        line: "{{ ansible_date_time.iso8601 }} - Pull executed successfully"
        create: yes
        mode: '0644'
    
    - name: Display completion message
      debug:
        msg: |
          Ansible Pull completed successfully!
          Check /tmp/ansible-pull-demo/ for results
          System: {{ ansible_hostname }}
          Time: {{ ansible_date_time.iso8601 }}
EOF

# Create inventory for the repository
cat > inventory << 'EOF'
[all]
localhost ansible_connection=local
EOF

# Commit to Git
git add .
git commit -m "Initial ansible-pull demo playbook"

echo "Git repository created at: $(pwd)"
```

### Step 2: Demonstrate ansible-pull Command
```bash
# Basic ansible-pull syntax
echo "=== Basic ansible-pull Command ==="
echo "ansible-pull -U <repository-url> [playbook]"
echo

# Execute ansible-pull with local repository
echo "=== Executing ansible-pull ==="
cd ~/ansible-pull-demo

ansible-pull \
  --url ~/ansible-pull-demo/ansible-pull-repo \
  --checkout main \
  --directory /tmp/ansible-pull-work \
  --inventory inventory \
  site.yml

echo "=== Pull completed! ==="
```

### Step 3: Examine Results
```bash
# Show what ansible-pull created
echo "=== Ansible Pull Results ==="

# Check the demo directory
ls -la /tmp/ansible-pull-demo/

# Show system info file
echo "=== System Information File ==="
cat /tmp/ansible-pull-demo/system-info.txt

# Show pull history
echo "=== Pull History Log ==="
cat /tmp/ansible-pull-demo/pull-history.log

# Show working directory (where ansible-pull cloned the repo)
echo "=== Working Directory ==="
ls -la /tmp/ansible-pull-work/
```

### Step 4: Advanced ansible-pull Options
```bash
# Show common ansible-pull options
echo "=== Common ansible-pull Options ==="

cat << 'EOF'
ansible-pull [options]

Key Options:
  -U, --url URL           Git repository URL
  -C, --checkout BRANCH   Git branch/tag to checkout (default: HEAD)
  -d, --directory DIR     Directory to clone repository to
  -i, --inventory FILE    Inventory file to use
  -l, --limit SUBSET      Limit to specific hosts
  -t, --tags TAGS         Only run plays and tasks tagged with these values
  --vault-password-file   Vault password file
  -v, --verbose           Verbose output
  --clean                 Clean out the local repository before sync
  --full                  Do a full clone instead of shallow

Examples:
  # Basic pull from GitHub
  ansible-pull -U https://github.com/user/repo.git site.yml
  
  # Pull specific branch with inventory
  ansible-pull -U https://github.com/user/repo.git -C production -i inventory/prod site.yml
  
  # Pull with vault password
  ansible-pull -U https://github.com/user/repo.git --vault-password-file ~/.vault_pass site.yml
  
  # Clean pull with verbose output
  ansible-pull -U https://github.com/user/repo.git --clean -v site.yml
EOF
```

## Demo 3: Scheduling and Automation (5 minutes)

### Cron Integration Example
```bash
# Show how to schedule ansible-pull with cron
echo "=== Scheduling ansible-pull with Cron ==="

cat << 'EOF'
# Example crontab entries for ansible-pull

# Run every 30 minutes
*/30 * * * * /usr/local/bin/ansible-pull -U https://github.com/company/infrastructure.git site.yml >> /var/log/ansible-pull.log 2>&1

# Run every hour with specific inventory
0 * * * * /usr/local/bin/ansible-pull -U https://github.com/company/infrastructure.git -i inventory/production site.yml

# Run daily at 2 AM with cleanup
0 2 * * * /usr/local/bin/ansible-pull -U https://github.com/company/infrastructure.git --clean site.yml

# Environment variables for cron
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=admin@company.com

# Multiple environments
*/15 * * * * /usr/local/bin/ansible-pull -U https://github.com/company/infrastructure.git -C staging -i inventory/staging site.yml
0 */4 * * * /usr/local/bin/ansible-pull -U https://github.com/company/infrastructure.git -C production -i inventory/production site.yml
EOF
```

### Systemd Timer Alternative
```bash
echo "=== Systemd Timer Alternative ==="

cat << 'EOF'
# /etc/systemd/system/ansible-pull.service
[Unit]
Description=Ansible Pull
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/ansible-pull -U https://github.com/company/infrastructure.git site.yml
User=ansible
Group=ansible

# /etc/systemd/system/ansible-pull.timer
[Unit]
Description=Run Ansible Pull every 30 minutes
Requires=ansible-pull.service

[Timer]
OnCalendar=*:0/30
Persistent=true

[Install]
WantedBy=timers.target

# Enable and start
sudo systemctl enable ansible-pull.timer
sudo systemctl start ansible-pull.timer
EOF
```

## Q&A and Best Practices Discussion (5 minutes)

### Discussion Points for Participants:

**For Alexandra & Gabriela (Beginners):**
1. "When might you prefer pull over push in a simple web application setup?"
2. "What are the main advantages of each approach?"
3. "How does this relate to your current deployment processes?"

**For Victor & Vlad (Intermediate):**
1. "How could ansible-pull fit into your CI/CD pipelines?"
2. "What security considerations are important for pull-based automation?"
3. "How would you handle secrets and sensitive data with ansible-pull?"

### Best Practices Summary:

**Security:**
- Use SSH keys or tokens for Git authentication
- Store sensitive data in Ansible Vault
- Limit repository access appropriately
- Monitor pull execution logs

**Reliability:**
- Implement proper error handling and logging
- Use specific Git branches/tags for stability
- Test playbooks thoroughly before deployment
- Have rollback procedures ready

**Performance:**
- Use shallow clones for faster pulls
- Schedule pulls appropriately (avoid peak times)
- Clean repositories periodically
- Monitor system resources

**Enterprise Considerations:**
- Centralized logging and monitoring
- Standardized repository structure
- Change management processes
- Disaster recovery procedures

## Key Takeaways

### What is Ansible Pull?
- Decentralized approach where targets pull configurations
- Uses Git repositories as the source of truth
- Targets execute ansible-pull command locally

### When to Use Ansible Pull:
1. **Large-scale environments** (1000+ hosts)
2. **Edge devices** with limited connectivity
3. **Self-healing infrastructure** requirements
4. **Strict network security** environments
5. **Autonomous updates** scenarios

### Basic Workflow:
1. Create playbooks in Git repository
2. Configure targets with ansible-pull command
3. Schedule execution (cron/systemd)
4. Monitor and log results

### Next Steps for Advanced Users:
- Explore Lab 7.2 for comprehensive automation examples
- Practice with real Git repositories
- Integrate with monitoring systems
- Implement in CI/CD workflows

## Demo Cleanup
```bash
# Clean up demo files
rm -rf /tmp/ansible-pull-demo /tmp/ansible-pull-work
rm -rf ~/ansible-pull-demo
```

---

**Note for Instructor**: This demo emphasizes concepts and practical understanding over complex implementation. Adjust depth based on participant questions and experience levels. For enterprise scenarios, point advanced participants to Lab 7.2 materials.

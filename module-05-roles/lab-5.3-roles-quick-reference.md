# Lab 5.3: Advanced Roles Patterns (Quick Reference)

## Objective
Advanced role patterns and enterprise practices for self-study

## Target Audience
**Victor & Vlad** - Intermediate participants for post-course exploration

## Duration
30-45 minutes (self-paced)

## Pattern 1: Role Dependencies and Composition

### Meta Dependencies
```yaml
# roles/webserver/meta/main.yml
---
galaxy_info:
  role_name: webserver
  author: DevOps Team
  description: Enterprise web server with security baseline
  min_ansible_version: 2.9

dependencies:
  - role: security_baseline
    vars:
      security_level: "high"
      
  - role: monitoring_agent
    vars:
      monitoring_enabled: true
      
  - role: log_aggregation
    when: environment == "production"
```

### Conditional Dependencies
```yaml
# Complex dependency scenarios
dependencies:
  # Always required
  - role: common_baseline
  
  # Environment-specific
  - role: ssl_certificates
    when: ssl_enabled | default(false)
    
  # Platform-specific  
  - role: selinux_config
    when: ansible_os_family == "RedHat"
    
  # Feature-specific
  - role: backup_agent
    when: backup_enabled | default(true)
```

## Pattern 2: Advanced Variable Management

### Multi-Environment Variables
```yaml
# roles/webapp/vars/main.yml
---
# Base configuration
app_name: "WebApplication"
app_version: "{{ app_version_override | default('latest') }}"

# Environment-specific configurations
environment_configs:
  development:
    debug_mode: true
    log_level: "DEBUG"
    database_pool_size: 2
    cache_enabled: false
    
  staging:
    debug_mode: true
    log_level: "INFO"
    database_pool_size: 5
    cache_enabled: true
    
  production:
    debug_mode: false
    log_level: "WARN"
    database_pool_size: 20
    cache_enabled: true
    ssl_required: true

# Merge environment-specific config
app_config: "{{ environment_configs[environment | default('development')] }}"
```

### Variable Validation
```yaml
# roles/webapp/tasks/validate.yml
---
- name: Validate required variables
  assert:
    that:
      - app_name is defined
      - app_name | length > 0
      - environment in ['development', 'staging', 'production']
      - database_host is defined
    fail_msg: "Required variables are missing or invalid"
    
- name: Validate environment-specific requirements
  assert:
    that:
      - ssl_certificate_path is defined
      - ssl_private_key_path is defined
    fail_msg: "SSL certificates required for production environment"
  when: environment == "production"
```

## Pattern 3: Role Composition and Modularity

### Task Organization
```yaml
# roles/webapp/tasks/main.yml
---
- name: Validate configuration
  include_tasks: validate.yml
  tags: ['validate']

- name: Install application dependencies
  include_tasks: install.yml
  tags: ['install']

- name: Configure application
  include_tasks: configure.yml
  tags: ['configure']

- name: Deploy application code
  include_tasks: deploy.yml
  tags: ['deploy']
  when: deploy_code | default(true)

- name: Setup monitoring
  include_tasks: monitoring.yml
  tags: ['monitoring']
  when: monitoring_enabled | default(false)
```

### Conditional Task Inclusion
```yaml
# roles/webapp/tasks/main.yml
---
- name: Include OS-specific tasks
  include_tasks: "{{ ansible_os_family | lower }}.yml"
  
- name: Include environment-specific tasks
  include_tasks: "{{ environment }}.yml"
  when: environment is defined

- name: Include feature-specific tasks
  include_tasks: "features/{{ item }}.yml"
  loop: "{{ enabled_features | default([]) }}"
```

## Pattern 4: Advanced Templates and Files

### Dynamic Template Selection
```yaml
# roles/webapp/tasks/configure.yml
---
- name: Generate application configuration
  template:
    src: "{{ item }}"
    dest: "/etc/webapp/{{ item | basename | regex_replace('\\.j2$', '') }}"
  with_first_found:
    - "config-{{ environment }}-{{ ansible_os_family }}.j2"
    - "config-{{ environment }}.j2"
    - "config-{{ ansible_os_family }}.j2"
    - "config-default.j2"
  notify: restart webapp
```

### Template Inheritance Pattern
```jinja2
{# roles/webapp/templates/base-config.j2 #}
# {{ app_name }} Configuration
# Environment: {{ environment }}
# Generated: {{ ansible_date_time.iso8601 }}

[application]
name={{ app_name }}
version={{ app_version }}
environment={{ environment }}

{% block database_config %}
[database]
host={{ database_host }}
port={{ database_port | default(5432) }}
{% endblock %}

{% block cache_config %}
{% if app_config.cache_enabled %}
[cache]
enabled=true
{% endif %}
{% endblock %}

{% block custom_config %}
# Custom configuration
{% endblock %}
```

## Pattern 5: Role Testing and Validation

### Molecule Testing Structure
```yaml
# roles/webapp/molecule/default/molecule.yml
---
dependency:
  name: galaxy
  
driver:
  name: docker
  
platforms:
  - name: webapp-ubuntu
    image: ubuntu:20.04
    pre_build_image: true
    
  - name: webapp-centos
    image: centos:8
    pre_build_image: true
    
provisioner:
  name: ansible
  inventory:
    host_vars:
      webapp-ubuntu:
        environment: staging
      webapp-centos:
        environment: production
        
verifier:
  name: ansible
```

### Integration Tests
```yaml
# roles/webapp/molecule/default/verify.yml
---
- name: Verify webapp deployment
  hosts: all
  gather_facts: false
  
  tasks:
    - name: Check if application is running
      uri:
        url: "http://localhost:{{ app_port | default(8080) }}/health"
        method: GET
      register: health_check
      
    - name: Verify health check response
      assert:
        that:
          - health_check.status == 200
          - "'healthy' in health_check.json.status"
          
    - name: Check log files exist
      stat:
        path: "/var/log/webapp/app.log"
      register: log_file
      
    - name: Verify logging is working
      assert:
        that:
          - log_file.stat.exists
          - log_file.stat.size > 0
```

## Pattern 6: Enterprise Role Distribution

### Role Metadata for Private Galaxy
```yaml
# roles/webapp/meta/main.yml
---
galaxy_info:
  role_name: webapp
  namespace: company
  author: DevOps Team
  description: Enterprise web application deployment role
  company: Enterprise Corp
  license: Proprietary
  min_ansible_version: 2.9
  
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: EL
      versions:
        - 8
        - 9
        
  galaxy_tags:
    - web
    - application
    - enterprise
    - deployment

dependencies:
  - role: company.security_baseline
    version: ">=1.2.0"
  - role: company.monitoring
    version: "~=2.1.0"
```

### Role Versioning Strategy
```yaml
# .ansible-lint configuration
---
exclude_paths:
  - .cache/
  - .github/
  - molecule/
  
rules:
  # Enforce role standards
  role-name: enable
  yaml-indentation: enable
  line-length:
    max: 120
    
# Version tagging strategy:
# - Major.Minor.Patch (1.2.3)
# - Major: Breaking changes
# - Minor: New features, backward compatible
# - Patch: Bug fixes, security updates
```

## Pattern 7: Role Performance Optimization

### Efficient Task Execution
```yaml
# roles/webapp/tasks/optimized.yml
---
- name: Install multiple packages efficiently
  package:
    name: "{{ webapp_packages }}"
    state: present
  # Better than multiple package tasks

- name: Use block for related tasks
  block:
    - name: Create application directories
      file:
        path: "{{ item }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
      loop: "{{ app_directories }}"
      
    - name: Set directory permissions
      file:
        path: "{{ app_base_dir }}"
        mode: '0755'
        recurse: yes
  when: app_directories is defined
  become: yes
  tags: ['filesystem']

- name: Use changed_when to avoid unnecessary notifications
  command: "systemctl reload {{ app_service }}"
  changed_when: false
  when: config_changed | default(false)
```

### Conditional Execution Optimization
```yaml
# roles/webapp/tasks/conditional.yml
---
- name: Set facts for complex conditions
  set_fact:
    is_production: "{{ environment == 'production' }}"
    needs_ssl: "{{ ssl_enabled | default(false) or environment == 'production' }}"
    backup_required: "{{ backup_enabled | default(is_production) }}"

- name: Configure SSL (only when needed)
  include_tasks: ssl.yml
  when: needs_ssl

- name: Setup backup (production or explicitly enabled)
  include_tasks: backup.yml
  when: backup_required
```

## Best Practices Summary

### Role Design Principles:
1. **Single Responsibility** - One role, one purpose
2. **Idempotency** - Safe to run multiple times
3. **Flexibility** - Configurable through variables
4. **Testability** - Include tests and validation
5. **Documentation** - Clear README and examples

### Variable Management:
- Use `defaults/` for configurable options
- Use `vars/` for internal role logic
- Validate critical variables
- Provide environment-specific configurations
- Use meaningful variable names

### Performance Optimization:
- Minimize task execution with conditions
- Use blocks for related tasks
- Batch operations when possible
- Use `changed_when` appropriately
- Tag tasks for selective execution

### Enterprise Considerations:
- Version control and tagging
- Automated testing (Molecule)
- Security scanning and compliance
- Documentation and change management
- Private Galaxy server integration

## Common Anti-Patterns to Avoid

### Bad Practices:
```yaml
# DON'T: Hardcode values
- name: Install nginx
  package:
    name: nginx  # Should be variable
    
# DON'T: Ignore idempotency
- name: Add user to group
  command: usermod -aG webapp nginx  # Use user module instead
  
# DON'T: Complex logic in templates
{% if complex_condition_with_multiple_variables %}
  # Move logic to tasks/vars instead
{% endif %}

# DON'T: Overly complex roles
# Split large roles into smaller, focused ones
```

### Good Practices:
```yaml
# DO: Use variables
- name: Install web server
  package:
    name: "{{ webserver_package }}"
    
# DO: Use appropriate modules
- name: Add user to group
  user:
    name: nginx
    groups: webapp
    append: yes
    
# DO: Keep templates simple
server_name {{ server_name | default(ansible_hostname) }};

# DO: Compose roles
dependencies:
  - role: security_baseline
  - role: monitoring_agent
```

---

**Note**: This reference covers advanced patterns for experienced users. Start with simple roles and gradually incorporate these patterns as your requirements grow in complexity.

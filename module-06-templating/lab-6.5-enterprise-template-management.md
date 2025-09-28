# Lab 6.5: Enterprise Template Management

## Objective
Establish enterprise-grade template management practices including versioning, governance, testing, and deployment strategies for large-scale automation environments.

## Duration
25 minutes

## Prerequisites
- Completed Labs 6.1-6.4
- Understanding of enterprise governance and change management
- Knowledge of version control and CI/CD concepts

## Lab Setup

```bash
cd ~/ansible-labs/module-06
mkdir -p lab-6.5
cd lab-6.5

# Create inventory for testing
cat > inventory.ini << 'EOF'
[template_managers]
manager1 ansible_host=localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Exercise 1: Template Versioning and Lifecycle Management (10 minutes)

### Task: Implement comprehensive template versioning and lifecycle management

Create template management framework:

```bash
# Create template management directory structure
mkdir -p templates/{versions,approved,deprecated,testing}
mkdir -p template-registry/{metadata,schemas,tests}
mkdir -p governance/{policies,reviews,approvals}
```

Create template metadata schema:

```yaml
# Create template-registry/schemas/template-metadata.yml
cat > template-registry/schemas/template-metadata.yml << 'EOF'
---
# Template Metadata Schema
schema_version: "1.0"
description: "Enterprise template metadata schema for governance and lifecycle management"

required_fields:
  - template_id
  - name
  - version
  - author
  - created_date
  - description
  - category
  - status

optional_fields:
  - tags
  - dependencies
  - supported_platforms
  - minimum_ansible_version
  - testing_requirements
  - approval_status
  - deprecation_date
  - replacement_template

field_definitions:
  template_id:
    type: string
    pattern: '^[a-z0-9-]+$'
    description: "Unique identifier for the template"
    
  name:
    type: string
    min_length: 3
    max_length: 100
    description: "Human-readable template name"
    
  version:
    type: string
    pattern: '^\d+\.\d+\.\d+$'
    description: "Semantic version number"
    
  author:
    type: string
    description: "Template author or team"
    
  created_date:
    type: string
    format: "date-time"
    description: "Template creation timestamp"
    
  description:
    type: string
    min_length: 10
    max_length: 500
    description: "Template description and purpose"
    
  category:
    type: string
    enum: ["infrastructure", "application", "security", "monitoring", "networking"]
    description: "Template category"
    
  status:
    type: string
    enum: ["draft", "testing", "approved", "deprecated", "retired"]
    description: "Template lifecycle status"
    
  tags:
    type: array
    items:
      type: string
    description: "Template tags for categorization"
    
  dependencies:
    type: array
    items:
      type: object
      properties:
        template_id:
          type: string
        version:
          type: string
        required:
          type: boolean
    description: "Template dependencies"
    
  supported_platforms:
    type: array
    items:
      type: string
    description: "Supported operating systems and platforms"
    
  minimum_ansible_version:
    type: string
    pattern: '^\d+\.\d+$'
    description: "Minimum required Ansible version"
    
  testing_requirements:
    type: object
    properties:
      unit_tests:
        type: boolean
      integration_tests:
        type: boolean
      security_tests:
        type: boolean
      performance_tests:
        type: boolean
    description: "Testing requirements for the template"
    
  approval_status:
    type: object
    properties:
      approved_by:
        type: string
      approval_date:
        type: string
        format: "date-time"
      approval_notes:
        type: string
    description: "Template approval information"
    
  deprecation_date:
    type: string
    format: "date-time"
    description: "Template deprecation date"
    
  replacement_template:
    type: string
    description: "ID of replacement template for deprecated templates"

validation_rules:
  - name: "version_increment"
    description: "New version must be higher than previous version"
    
  - name: "approval_required_for_production"
    description: "Templates must be approved before production use"
    
  - name: "testing_required"
    description: "All templates must pass required tests"
    
  - name: "deprecation_notice"
    description: "Deprecated templates must specify replacement"
EOF
```

Create template registry:

```yaml
# Create template-registry/registry.yml
cat > template-registry/registry.yml << 'EOF'
---
# Enterprise Template Registry
registry_version: "1.0"
last_updated: "2024-01-15T10:00:00Z"
total_templates: 0

templates:
  - template_id: "nginx-web-server"
    name: "Nginx Web Server Configuration"
    version: "2.1.0"
    author: "Infrastructure Team"
    created_date: "2024-01-10T09:00:00Z"
    last_modified: "2024-01-15T10:00:00Z"
    description: "Enterprise-grade Nginx web server configuration with SSL, monitoring, and security hardening"
    category: "infrastructure"
    status: "approved"
    tags: ["web", "nginx", "ssl", "security"]
    
    dependencies:
      - template_id: "ssl-certificates"
        version: "1.0.0"
        required: true
      - template_id: "monitoring-agent"
        version: "1.2.0"
        required: false
    
    supported_platforms: ["Ubuntu 20.04", "Ubuntu 22.04", "CentOS 8", "RHEL 8"]
    minimum_ansible_version: "2.9"
    
    testing_requirements:
      unit_tests: true
      integration_tests: true
      security_tests: true
      performance_tests: false
    
    approval_status:
      approved_by: "John Smith (Infrastructure Lead)"
      approval_date: "2024-01-12T14:30:00Z"
      approval_notes: "Approved for production use after security review"
    
    file_locations:
      template: "templates/approved/nginx-web-server-v2.1.0.j2"
      metadata: "template-registry/metadata/nginx-web-server.yml"
      tests: "template-registry/tests/nginx-web-server/"
      documentation: "docs/templates/nginx-web-server.md"
    
    usage_statistics:
      deployments: 45
      last_used: "2024-01-15T08:30:00Z"
      success_rate: 98.5
    
    version_history:
      - version: "2.0.0"
        date: "2024-01-05T10:00:00Z"
        changes: "Added SSL support and security hardening"
        status: "deprecated"
      - version: "1.5.0"
        date: "2023-12-15T10:00:00Z"
        changes: "Initial production version"
        status: "retired"

  - template_id: "postgresql-database"
    name: "PostgreSQL Database Configuration"
    version: "1.8.0"
    author: "Database Team"
    created_date: "2024-01-08T11:00:00Z"
    last_modified: "2024-01-14T16:00:00Z"
    description: "Production-ready PostgreSQL database configuration with backup, replication, and monitoring"
    category: "infrastructure"
    status: "approved"
    tags: ["database", "postgresql", "backup", "replication"]
    
    dependencies:
      - template_id: "backup-agent"
        version: "2.0.0"
        required: true
      - template_id: "monitoring-agent"
        version: "1.2.0"
        required: true
    
    supported_platforms: ["Ubuntu 20.04", "Ubuntu 22.04", "CentOS 8"]
    minimum_ansible_version: "2.9"
    
    testing_requirements:
      unit_tests: true
      integration_tests: true
      security_tests: true
      performance_tests: true
    
    approval_status:
      approved_by: "Sarah Johnson (Database Lead)"
      approval_date: "2024-01-13T11:15:00Z"
      approval_notes: "Approved after performance testing validation"
    
    file_locations:
      template: "templates/approved/postgresql-database-v1.8.0.j2"
      metadata: "template-registry/metadata/postgresql-database.yml"
      tests: "template-registry/tests/postgresql-database/"
      documentation: "docs/templates/postgresql-database.md"
    
    usage_statistics:
      deployments: 12
      last_used: "2024-01-14T20:00:00Z"
      success_rate: 100.0
    
    version_history:
      - version: "1.7.0"
        date: "2024-01-01T10:00:00Z"
        changes: "Added replication support"
        status: "deprecated"

  - template_id: "application-config"
    name: "Generic Application Configuration"
    version: "3.0.0"
    author: "Application Team"
    created_date: "2024-01-12T14:00:00Z"
    last_modified: "2024-01-15T09:00:00Z"
    description: "Flexible application configuration template supporting multiple environments and deployment patterns"
    category: "application"
    status: "testing"
    tags: ["application", "configuration", "multi-environment"]
    
    dependencies: []
    
    supported_platforms: ["Ubuntu 20.04", "Ubuntu 22.04", "CentOS 8", "RHEL 8"]
    minimum_ansible_version: "2.10"
    
    testing_requirements:
      unit_tests: true
      integration_tests: true
      security_tests: false
      performance_tests: false
    
    file_locations:
      template: "templates/testing/application-config-v3.0.0.j2"
      metadata: "template-registry/metadata/application-config.yml"
      tests: "template-registry/tests/application-config/"
      documentation: "docs/templates/application-config.md"
    
    usage_statistics:
      deployments: 3
      last_used: "2024-01-15T07:00:00Z"
      success_rate: 66.7
    
    version_history:
      - version: "2.5.0"
        date: "2023-12-20T10:00:00Z"
        changes: "Added environment-specific overrides"
        status: "approved"

deprecated_templates:
  - template_id: "legacy-apache"
    name: "Legacy Apache Configuration"
    version: "1.0.0"
    deprecation_date: "2024-01-01T00:00:00Z"
    replacement_template: "nginx-web-server"
    reason: "Migrating to Nginx for better performance and security"
    end_of_life_date: "2024-06-01T00:00:00Z"

statistics:
  total_templates: 3
  approved_templates: 2
  testing_templates: 1
  deprecated_templates: 1
  average_success_rate: 88.4
  total_deployments: 60
EOF
```

Create template lifecycle management script:

```python
# Create template-registry/lifecycle-manager.py
cat > template-registry/lifecycle-manager.py << 'EOF'
#!/usr/bin/env python3
"""
Template Lifecycle Management System
Manages template versions, approvals, and lifecycle transitions
"""

import yaml
import json
import sys
import os
from datetime import datetime, timedelta
from pathlib import Path

class TemplateLifecycleManager:
    def __init__(self, registry_file="registry.yml", schema_file="schemas/template-metadata.yml"):
        self.registry_file = registry_file
        self.schema_file = schema_file
        self.registry = self.load_registry()
        self.schema = self.load_schema()
    
    def load_registry(self):
        """Load template registry"""
        try:
            with open(self.registry_file, 'r') as f:
                return yaml.safe_load(f)
        except FileNotFoundError:
            return {"registry_version": "1.0", "templates": [], "deprecated_templates": []}
    
    def load_schema(self):
        """Load template metadata schema"""
        try:
            with open(self.schema_file, 'r') as f:
                return yaml.safe_load(f)
        except FileNotFoundError:
            return {}
    
    def save_registry(self):
        """Save template registry"""
        self.registry["last_updated"] = datetime.now().isoformat() + "Z"
        with open(self.registry_file, 'w') as f:
            yaml.dump(self.registry, f, default_flow_style=False, sort_keys=False)
    
    def validate_template_metadata(self, metadata):
        """Validate template metadata against schema"""
        errors = []
        
        # Check required fields
        for field in self.schema.get("required_fields", []):
            if field not in metadata:
                errors.append(f"Missing required field: {field}")
        
        # Validate field types and constraints
        field_defs = self.schema.get("field_definitions", {})
        for field, value in metadata.items():
            if field in field_defs:
                field_def = field_defs[field]
                
                # Type validation
                if field_def.get("type") == "string" and not isinstance(value, str):
                    errors.append(f"Field {field} must be a string")
                elif field_def.get("type") == "array" and not isinstance(value, list):
                    errors.append(f"Field {field} must be an array")
                
                # Pattern validation
                if field_def.get("pattern") and isinstance(value, str):
                    import re
                    if not re.match(field_def["pattern"], value):
                        errors.append(f"Field {field} does not match required pattern")
                
                # Enum validation
                if field_def.get("enum") and value not in field_def["enum"]:
                    errors.append(f"Field {field} must be one of: {field_def['enum']}")
        
        return errors
    
    def register_template(self, metadata):
        """Register a new template or update existing one"""
        errors = self.validate_template_metadata(metadata)
        if errors:
            return {"success": False, "errors": errors}
        
        template_id = metadata["template_id"]
        
        # Check if template exists
        existing_template = None
        for i, template in enumerate(self.registry["templates"]):
            if template["template_id"] == template_id:
                existing_template = i
                break
        
        # Add timestamp
        metadata["last_modified"] = datetime.now().isoformat() + "Z"
        
        if existing_template is not None:
            # Update existing template
            old_version = self.registry["templates"][existing_template]["version"]
            
            # Add to version history
            if "version_history" not in self.registry["templates"][existing_template]:
                self.registry["templates"][existing_template]["version_history"] = []
            
            self.registry["templates"][existing_template]["version_history"].append({
                "version": old_version,
                "date": self.registry["templates"][existing_template]["last_modified"],
                "changes": metadata.get("change_notes", "Version update"),
                "status": self.registry["templates"][existing_template]["status"]
            })
            
            # Update template
            self.registry["templates"][existing_template].update(metadata)
        else:
            # Add new template
            metadata["created_date"] = datetime.now().isoformat() + "Z"
            metadata["usage_statistics"] = {
                "deployments": 0,
                "last_used": None,
                "success_rate": 0.0
            }
            metadata["version_history"] = []
            
            self.registry["templates"].append(metadata)
        
        self.save_registry()
        return {"success": True, "message": f"Template {template_id} registered successfully"}
    
    def approve_template(self, template_id, approver, notes=""):
        """Approve a template for production use"""
        for template in self.registry["templates"]:
            if template["template_id"] == template_id:
                if template["status"] != "testing":
                    return {"success": False, "error": "Only testing templates can be approved"}
                
                template["status"] = "approved"
                template["approval_status"] = {
                    "approved_by": approver,
                    "approval_date": datetime.now().isoformat() + "Z",
                    "approval_notes": notes
                }
                
                self.save_registry()
                return {"success": True, "message": f"Template {template_id} approved"}
        
        return {"success": False, "error": f"Template {template_id} not found"}
    
    def deprecate_template(self, template_id, replacement_template=None, reason=""):
        """Deprecate a template"""
        for template in self.registry["templates"]:
            if template["template_id"] == template_id:
                template["status"] = "deprecated"
                template["deprecation_date"] = datetime.now().isoformat() + "Z"
                
                if replacement_template:
                    template["replacement_template"] = replacement_template
                
                if reason:
                    template["deprecation_reason"] = reason
                
                # Move to deprecated templates list
                deprecated_template = template.copy()
                self.registry["deprecated_templates"].append(deprecated_template)
                
                self.save_registry()
                return {"success": True, "message": f"Template {template_id} deprecated"}
        
        return {"success": False, "error": f"Template {template_id} not found"}
    
    def get_template_status(self, template_id):
        """Get template status and information"""
        for template in self.registry["templates"]:
            if template["template_id"] == template_id:
                return {
                    "template_id": template_id,
                    "name": template["name"],
                    "version": template["version"],
                    "status": template["status"],
                    "author": template["author"],
                    "last_modified": template["last_modified"],
                    "usage_statistics": template.get("usage_statistics", {}),
                    "dependencies": template.get("dependencies", [])
                }
        
        return {"error": f"Template {template_id} not found"}
    
    def list_templates(self, status=None, category=None):
        """List templates with optional filtering"""
        templates = []
        
        for template in self.registry["templates"]:
            if status and template["status"] != status:
                continue
            if category and template["category"] != category:
                continue
            
            templates.append({
                "template_id": template["template_id"],
                "name": template["name"],
                "version": template["version"],
                "status": template["status"],
                "category": template["category"],
                "author": template["author"],
                "last_modified": template["last_modified"]
            })
        
        return templates
    
    def check_deprecated_templates(self):
        """Check for templates that should be retired"""
        retirement_candidates = []
        
        for template in self.registry["deprecated_templates"]:
            if "end_of_life_date" in template:
                eol_date = datetime.fromisoformat(template["end_of_life_date"].replace("Z", "+00:00"))
                if datetime.now(eol_date.tzinfo) > eol_date:
                    retirement_candidates.append(template["template_id"])
        
        return retirement_candidates
    
    def generate_report(self):
        """Generate template registry report"""
        total_templates = len(self.registry["templates"])
        approved_templates = len([t for t in self.registry["templates"] if t["status"] == "approved"])
        testing_templates = len([t for t in self.registry["templates"] if t["status"] == "testing"])
        deprecated_templates = len(self.registry["deprecated_templates"])
        
        total_deployments = sum(t.get("usage_statistics", {}).get("deployments", 0) for t in self.registry["templates"])
        
        success_rates = [t.get("usage_statistics", {}).get("success_rate", 0) for t in self.registry["templates"] if t.get("usage_statistics", {}).get("deployments", 0) > 0]
        avg_success_rate = sum(success_rates) / len(success_rates) if success_rates else 0
        
        return {
            "report_date": datetime.now().isoformat() + "Z",
            "statistics": {
                "total_templates": total_templates,
                "approved_templates": approved_templates,
                "testing_templates": testing_templates,
                "deprecated_templates": deprecated_templates,
                "total_deployments": total_deployments,
                "average_success_rate": round(avg_success_rate, 1)
            },
            "templates_by_category": self._group_by_category(),
            "recent_activity": self._get_recent_activity(),
            "retirement_candidates": self.check_deprecated_templates()
        }
    
    def _group_by_category(self):
        """Group templates by category"""
        categories = {}
        for template in self.registry["templates"]:
            category = template["category"]
            if category not in categories:
                categories[category] = 0
            categories[category] += 1
        return categories
    
    def _get_recent_activity(self):
        """Get recent template activity"""
        recent_templates = sorted(
            self.registry["templates"],
            key=lambda x: x["last_modified"],
            reverse=True
        )[:5]
        
        return [
            {
                "template_id": t["template_id"],
                "name": t["name"],
                "version": t["version"],
                "status": t["status"],
                "last_modified": t["last_modified"]
            }
            for t in recent_templates
        ]

def main():
    if len(sys.argv) < 2:
        print("Usage: lifecycle-manager.py <command> [args...]")
        print("Commands: register, approve, deprecate, status, list, report")
        sys.exit(1)
    
    manager = TemplateLifecycleManager()
    command = sys.argv[1]
    
    if command == "register":
        if len(sys.argv) < 3:
            print("Usage: lifecycle-manager.py register <metadata_file>")
            sys.exit(1)
        
        with open(sys.argv[2], 'r') as f:
            metadata = yaml.safe_load(f)
        
        result = manager.register_template(metadata)
        print(json.dumps(result, indent=2))
    
    elif command == "approve":
        if len(sys.argv) < 4:
            print("Usage: lifecycle-manager.py approve <template_id> <approver> [notes]")
            sys.exit(1)
        
        template_id = sys.argv[2]
        approver = sys.argv[3]
        notes = sys.argv[4] if len(sys.argv) > 4 else ""
        
        result = manager.approve_template(template_id, approver, notes)
        print(json.dumps(result, indent=2))
    
    elif command == "deprecate":
        if len(sys.argv) < 3:
            print("Usage: lifecycle-manager.py deprecate <template_id> [replacement] [reason]")
            sys.exit(1)
        
        template_id = sys.argv[2]
        replacement = sys.argv[3] if len(sys.argv) > 3 else None
        reason = sys.argv[4] if len(sys.argv) > 4 else ""
        
        result = manager.deprecate_template(template_id, replacement, reason)
        print(json.dumps(result, indent=2))
    
    elif command == "status":
        if len(sys.argv) < 3:
            print("Usage: lifecycle-manager.py status <template_id>")
            sys.exit(1)
        
        template_id = sys.argv[2]
        result = manager.get_template_status(template_id)
        print(json.dumps(result, indent=2))
    
    elif command == "list":
        status_filter = sys.argv[2] if len(sys.argv) > 2 else None
        category_filter = sys.argv[3] if len(sys.argv) > 3 else None
        
        result = manager.list_templates(status_filter, category_filter)
        print(json.dumps(result, indent=2))
    
    elif command == "report":
        result = manager.generate_report()
        print(json.dumps(result, indent=2))
    
    else:
        print(f"Unknown command: {command}")
        sys.exit(1)

if __name__ == "__main__":
    main()
EOF

chmod +x template-registry/lifecycle-manager.py
```

Create template metadata examples:

```yaml
# Create template-registry/metadata/nginx-web-server.yml
mkdir -p template-registry/metadata
cat > template-registry/metadata/nginx-web-server.yml << 'EOF'
---
template_id: "nginx-web-server"
name: "Nginx Web Server Configuration"
version: "2.2.0"
author: "Infrastructure Team"
description: "Enterprise-grade Nginx web server configuration with SSL, monitoring, security hardening, and performance optimization"
category: "infrastructure"
status: "testing"
change_notes: "Added performance optimization and caching support"

tags:
  - "web"
  - "nginx"
  - "ssl"
  - "security"
  - "performance"
  - "caching"

dependencies:
  - template_id: "ssl-certificates"
    version: "1.1.0"
    required: true
  - template_id: "monitoring-agent"
    version: "1.3.0"
    required: false
  - template_id: "log-rotation"
    version: "1.0.0"
    required: false

supported_platforms:
  - "Ubuntu 20.04"
  - "Ubuntu 22.04"
  - "CentOS 8"
  - "RHEL 8"
  - "RHEL 9"

minimum_ansible_version: "2.10"

testing_requirements:
  unit_tests: true
  integration_tests: true
  security_tests: true
  performance_tests: true

file_locations:
  template: "templates/testing/nginx-web-server-v2.2.0.j2"
  metadata: "template-registry/metadata/nginx-web-server.yml"
  tests: "template-registry/tests/nginx-web-server/"
  documentation: "docs/templates/nginx-web-server.md"

configuration_parameters:
  required:
    - server_name
    - document_root
  optional:
    - ssl_enabled
    - worker_processes
    - worker_connections
    - gzip_enabled
    - cache_enabled
    - monitoring_enabled

security_considerations:
  - "SSL/TLS configuration follows industry best practices"
  - "Security headers are automatically configured"
  - "Rate limiting can be enabled for DDoS protection"
  - "Access logging is enabled by default"

performance_characteristics:
  - "Optimized worker process configuration"
  - "Gzip compression enabled by default"
  - "Static file caching configured"
  - "Keep-alive connections optimized"

maintenance_notes:
  - "SSL certificates should be renewed every 90 days"
  - "Log rotation is configured automatically"
  - "Performance metrics are collected if monitoring is enabled"
  - "Configuration validation is performed before deployment"
EOF
```

Create test for lifecycle management:

```yaml
# Create test-lifecycle-management.yml
cat > test-lifecycle-management.yml << 'EOF'
---
- name: Test template versioning and lifecycle management
  hosts: template_managers
  gather_facts: yes
  
  tasks:
    - name: Test template registration
      shell: |
        cd template-registry
        python3 lifecycle-manager.py register metadata/nginx-web-server.yml
      register: registration_result
      changed_when: false
    
    - name: Display registration result
      debug:
        msg: "{{ registration_result.stdout | from_json }}"
    
    - name: Test template approval
      shell: |
        cd template-registry
        python3 lifecycle-manager.py approve nginx-web-server "John Smith (Infrastructure Lead)" "Approved after security and performance testing"
      register: approval_result
      changed_when: false
    
    - name: Display approval result
      debug:
        msg: "{{ approval_result.stdout | from_json }}"
    
    - name: Test template status check
      shell: |
        cd template-registry
        python3 lifecycle-manager.py status nginx-web-server
      register: status_result
      changed_when: false
    
    - name: Display template status
      debug:
        msg: "{{ status_result.stdout | from_json }}"
    
    - name: Test template listing
      shell: |
        cd template-registry
        python3 lifecycle-manager.py list approved
      register: list_result
      changed_when: false
    
    - name: Display approved templates
      debug:
        msg: "{{ list_result.stdout | from_json }}"
    
    - name: Generate registry report
      shell: |
        cd template-registry
        python3 lifecycle-manager.py report
      register: report_result
      changed_when: false
    
    - name: Display registry report
      debug:
        msg: "{{ report_result.stdout | from_json }}"
    
    - name: Display lifecycle management summary
      debug:
        msg: |
          Template Lifecycle Management Summary:
          
          Template Registration: {{ (registration_result.stdout | from_json).success | default(false) }}
          Template Approval: {{ (approval_result.stdout | from_json).success | default(false) }}
          
          Registry Statistics:
          - Total Templates: {{ (report_result.stdout | from_json).statistics.total_templates }}
          - Approved Templates: {{ (report_result.stdout | from_json).statistics.approved_templates }}
          - Testing Templates: {{ (report_result.stdout | from_json).statistics.testing_templates }}
          - Deprecated Templates: {{ (report_result.stdout | from_json).statistics.deprecated_templates }}
          
          Template Status: {{ (status_result.stdout | from_json).status | default('unknown') }}
          Template Version: {{ (status_result.stdout | from_json).version | default('unknown') }}
EOF

# Run lifecycle management test
ansible-playbook -i inventory.ini test-lifecycle-management.yml
```

## Exercise 2: Template Testing and Quality Assurance (8 minutes)

### Task: Implement comprehensive template testing and quality assurance processes

Create template testing framework:

```bash
# Create testing framework structure
mkdir -p template-registry/tests/{unit,integration,security,performance}
mkdir -p template-registry/tests/nginx-web-server/{unit,integration,security}
```

Create template unit tests:

```python
# Create template-registry/tests/nginx-web-server/unit/test_template_syntax.py
cat > template-registry/tests/nginx-web-server/unit/test_template_syntax.py << 'EOF'
#!/usr/bin/env python3
"""
Unit tests for Nginx web server template
Tests template syntax, variable handling, and basic functionality
"""

import unittest
import tempfile
import os
from jinja2 import Environment, FileSystemLoader, TemplateSyntaxError
import yaml

class TestNginxTemplateUnit(unittest.TestCase):
    
    def setUp(self):
        """Set up test environment"""
        self.template_dir = os.path.join(os.path.dirname(__file__), '../../../../templates/testing')
        self.template_name = 'nginx-web-server-v2.2.0.j2'
        
        # Test variables
        self.test_vars = {
            'server_name': 'test.example.com',
            'document_root': '/var/www/test',
            'ssl_enabled': True,
            'worker_processes': 4,
            'worker_connections': 2048,
            'gzip_enabled': True,
            'cache_enabled': False,
            'monitoring_enabled': True,
            'environment': 'testing',
            'ansible_managed': 'Ansible Test',
            'ansible_hostname': 'test-server',
            'ansible_date_time': {'iso8601': '2024-01-15T10:00:00Z'}
        }
    
    def test_template_syntax(self):
        """Test that template has valid Jinja2 syntax"""
        try:
            env = Environment(loader=FileSystemLoader(self.template_dir))
            template = env.get_template(self.template_name)
            self.assertIsNotNone(template)
        except TemplateSyntaxError as e:
            self.fail(f"Template syntax error: {e}")
    
    def test_template_rendering_basic(self):
        """Test basic template rendering with minimal variables"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        minimal_vars = {
            'server_name': 'test.example.com',
            'document_root': '/var/www/test',
            'ansible_managed': 'Ansible Test',
            'ansible_hostname': 'test-server',
            'ansible_date_time': {'iso8601': '2024-01-15T10:00:00Z'}
        }
        
        try:
            result = template.render(**minimal_vars)
            self.assertIsNotNone(result)
            self.assertIn('test.example.com', result)
            self.assertIn('/var/www/test', result)
        except Exception as e:
            self.fail(f"Template rendering failed: {e}")
    
    def test_template_rendering_full(self):
        """Test template rendering with all variables"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        try:
            result = template.render(**self.test_vars)
            self.assertIsNotNone(result)
            self.assertIn('test.example.com', result)
            self.assertIn('worker_processes 4', result)
            self.assertIn('worker_connections 2048', result)
        except Exception as e:
            self.fail(f"Template rendering failed: {e}")
    
    def test_ssl_configuration(self):
        """Test SSL configuration rendering"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        # Test with SSL enabled
        ssl_vars = self.test_vars.copy()
        ssl_vars['ssl_enabled'] = True
        
        result = template.render(**ssl_vars)
        self.assertIn('ssl_certificate', result)
        self.assertIn('ssl_certificate_key', result)
        
        # Test with SSL disabled
        ssl_vars['ssl_enabled'] = False
        result = template.render(**ssl_vars)
        self.assertNotIn('ssl_certificate', result)
    
    def test_conditional_blocks(self):
        """Test conditional block rendering"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        # Test gzip enabled
        gzip_vars = self.test_vars.copy()
        gzip_vars['gzip_enabled'] = True
        
        result = template.render(**gzip_vars)
        self.assertIn('gzip on', result)
        
        # Test gzip disabled
        gzip_vars['gzip_enabled'] = False
        result = template.render(**gzip_vars)
        self.assertIn('gzip off', result)
    
    def test_variable_defaults(self):
        """Test that template handles missing variables with defaults"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        minimal_vars = {
            'server_name': 'test.example.com',
            'document_root': '/var/www/test',
            'ansible_managed': 'Ansible Test',
            'ansible_hostname': 'test-server',
            'ansible_date_time': {'iso8601': '2024-01-15T10:00:00Z'}
        }
        
        try:
            result = template.render(**minimal_vars)
            # Should render without errors even with missing optional variables
            self.assertIsNotNone(result)
        except Exception as e:
            self.fail(f"Template should handle missing optional variables: {e}")
    
    def test_output_format(self):
        """Test that output is valid Nginx configuration format"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        result = template.render(**self.test_vars)
        
        # Check for basic Nginx configuration structure
        self.assertIn('server {', result)
        self.assertIn('listen', result)
        self.assertIn('server_name', result)
        self.assertIn('root', result)
        
        # Check for proper block closure
        self.assertEqual(result.count('{'), result.count('}'))

if __name__ == '__main__':
    unittest.main()
EOF

chmod +x template-registry/tests/nginx-web-server/unit/test_template_syntax.py
```

Create integration tests:

```yaml
# Create template-registry/tests/nginx-web-server/integration/test_integration.yml
cat > template-registry/tests/nginx-web-server/integration/test_integration.yml << 'EOF'
---
# Integration tests for Nginx web server template
- name: Nginx Template Integration Tests
  hosts: localhost
  connection: local
  gather_facts: yes
  
  vars:
    test_server_name: "integration-test.example.com"
    test_document_root: "/tmp/nginx-test"
    test_ssl_enabled: true
    test_worker_processes: 2
    test_worker_connections: 1024
    
  tasks:
    - name: Create test directory
      file:
        path: "{{ test_document_root }}"
        state: directory
        mode: '0755'
    
    - name: Generate Nginx configuration from template
      template:
        src: "../../../../templates/testing/nginx-web-server-v2.2.0.j2"
        dest: "/tmp/nginx-integration-test.conf"
        mode: '0644'
      vars:
        server_name: "{{ test_server_name }}"
        document_root: "{{ test_document_root }}"
        ssl_enabled: "{{ test_ssl_enabled }}"
        worker_processes: "{{ test_worker_processes }}"
        worker_connections: "{{ test_worker_connections }}"
        gzip_enabled: true
        cache_enabled: false
        monitoring_enabled: true
    
    - name: Validate Nginx configuration syntax
      shell: nginx -t -c /tmp/nginx-integration-test.conf
      register: nginx_syntax_check
      failed_when: false
      changed_when: false
    
    - name: Check configuration file exists
      stat:
        path: "/tmp/nginx-integration-test.conf"
      register: config_file_stat
    
    - name: Verify configuration content
      shell: grep -q "{{ test_server_name }}" /tmp/nginx-integration-test.conf
      register: server_name_check
      failed_when: false
      changed_when: false
    
    - name: Verify SSL configuration
      shell: grep -q "ssl_certificate" /tmp/nginx-integration-test.conf
      register: ssl_check
      failed_when: false
      changed_when: false
      when: test_ssl_enabled
    
    - name: Verify worker configuration
      shell: grep -q "worker_processes {{ test_worker_processes }}" /tmp/nginx-integration-test.conf
      register: worker_check
      failed_when: false
      changed_when: false
    
    - name: Display integration test results
      debug:
        msg: |
          Integration Test Results:
          - Configuration Generated: {{ 'PASS' if config_file_stat.stat.exists else 'FAIL' }}
          - Nginx Syntax Check: {{ 'PASS' if nginx_syntax_check.rc == 0 else 'FAIL' }}
          - Server Name Present: {{ 'PASS' if server_name_check.rc == 0 else 'FAIL' }}
          - SSL Configuration: {{ 'PASS' if ssl_check.rc == 0 else 'FAIL' if test_ssl_enabled else 'SKIP' }}
          - Worker Configuration: {{ 'PASS' if worker_check.rc == 0 else 'FAIL' }}
    
    - name: Assert all tests passed
      assert:
        that:
          - config_file_stat.stat.exists
          - nginx_syntax_check.rc == 0
          - server_name_check.rc == 0
          - worker_check.rc == 0
          - not test_ssl_enabled or ssl_check.rc == 0
        fail_msg: "One or more integration tests failed"
        success_msg: "All integration tests passed"
    
    - name: Clean up test files
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - "/tmp/nginx-integration-test.conf"
        - "{{ test_document_root }}"
EOF
```

Create security tests:

```python
# Create template-registry/tests/nginx-web-server/security/test_security.py
cat > template-registry/tests/nginx-web-server/security/test_security.py << 'EOF'
#!/usr/bin/env python3
"""
Security tests for Nginx web server template
Tests security configurations and best practices
"""

import unittest
import tempfile
import os
import re
from jinja2 import Environment, FileSystemLoader

class TestNginxTemplateSecurity(unittest.TestCase):
    
    def setUp(self):
        """Set up test environment"""
        self.template_dir = os.path.join(os.path.dirname(__file__), '../../../../templates/testing')
        self.template_name = 'nginx-web-server-v2.2.0.j2'
        
        self.test_vars = {
            'server_name': 'secure.example.com',
            'document_root': '/var/www/secure',
            'ssl_enabled': True,
            'worker_processes': 4,
            'worker_connections': 2048,
            'gzip_enabled': True,
            'monitoring_enabled': True,
            'environment': 'production',
            'ansible_managed': 'Ansible Security Test',
            'ansible_hostname': 'secure-server',
            'ansible_date_time': {'iso8601': '2024-01-15T10:00:00Z'}
        }
    
    def test_ssl_configuration_security(self):
        """Test SSL/TLS security configuration"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        result = template.render(**self.test_vars)
        
        # Check for SSL protocols
        self.assertIn('ssl_protocols', result)
        self.assertNotIn('SSLv2', result)
        self.assertNotIn('SSLv3', result)
        self.assertNotIn('TLSv1.0', result)
        self.assertNotIn('TLSv1.1', result)
        
        # Should include modern TLS versions
        self.assertTrue(
            'TLSv1.2' in result or 'TLSv1.3' in result,
            "Should include modern TLS versions"
        )
    
    def test_security_headers(self):
        """Test security headers configuration"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        result = template.render(**self.test_vars)
        
        # Check for security headers
        security_headers = [
            'X-Frame-Options',
            'X-Content-Type-Options',
            'X-XSS-Protection',
            'Strict-Transport-Security'
        ]
        
        for header in security_headers:
            self.assertIn(header, result, f"Missing security header: {header}")
    
    def test_server_tokens_disabled(self):
        """Test that server tokens are disabled"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        result = template.render(**self.test_vars)
        
        # Check for server_tokens off
        self.assertIn('server_tokens off', result)
    
    def test_access_log_enabled(self):
        """Test that access logging is enabled"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        result = template.render(**self.test_vars)
        
        # Check for access log configuration
        self.assertIn('access_log', result)
        self.assertNotIn('access_log off', result)
    
    def test_no_sensitive_information_exposure(self):
        """Test that no sensitive information is exposed"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        result = template.render(**self.test_vars)
        
        # Check for potential sensitive information exposure
        sensitive_patterns = [
            r'password\s*=\s*["\']?[^"\'\s]+',
            r'secret\s*=\s*["\']?[^"\'\s]+',
            r'key\s*=\s*["\']?[^"\'\s]+[^/]',  # Exclude file paths
            r'token\s*=\s*["\']?[^"\'\s]+'
        ]
        
        for pattern in sensitive_patterns:
            matches = re.findall(pattern, result, re.IGNORECASE)
            self.assertEqual(len(matches), 0, f"Potential sensitive information exposure: {matches}")
    
    def test_default_server_security(self):
        """Test default server security configuration"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        result = template.render(**self.test_vars)
        
        # Check for default server configuration that drops unknown hosts
        self.assertIn('default_server', result)
    
    def test_file_upload_security(self):
        """Test file upload security settings"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        result = template.render(**self.test_vars)
        
        # Check for client_max_body_size to prevent large uploads
        self.assertIn('client_max_body_size', result)
    
    def test_directory_traversal_protection(self):
        """Test protection against directory traversal"""
        env = Environment(loader=FileSystemLoader(self.template_dir))
        template = env.get_template(self.template_name)
        
        result = template.render(**self.test_vars)
        
        # Check for location blocks that deny access to sensitive files
        self.assertIn('location ~ /\\.', result)  # Hidden files protection
        self.assertIn('deny all', result)

if __name__ == '__main__':
    unittest.main()
EOF

chmod +x template-registry/tests/nginx-web-server/security/test_security.py
```

Create test runner script:

```bash
# Create template-registry/tests/run-tests.sh
cat > template-registry/tests/run-tests.sh << 'EOF'
#!/bin/bash
# Template Testing Runner
# Runs all tests for template quality assurance

set -e

TEMPLATE_ID=${1:-"nginx-web-server"}
TEST_TYPE=${2:-"all"}

echo "=== Template Testing Runner ==="
echo "Template: $TEMPLATE_ID"
echo "Test Type: $TEST_TYPE"
echo "Timestamp: $(date -Iseconds)"
echo

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Test results
UNIT_TESTS_PASSED=0
INTEGRATION_TESTS_PASSED=0
SECURITY_TESTS_PASSED=0
PERFORMANCE_TESTS_PASSED=0

# Function to run unit tests
run_unit_tests() {
    echo -e "${YELLOW}Running Unit Tests...${NC}"
    
    if [ -d "$TEMPLATE_ID/unit" ]; then
        cd "$TEMPLATE_ID/unit"
        
        for test_file in test_*.py; do
            if [ -f "$test_file" ]; then
                echo "Running $test_file..."
                if python3 "$test_file"; then
                    echo -e "${GREEN}✓ $test_file PASSED${NC}"
                    UNIT_TESTS_PASSED=$((UNIT_TESTS_PASSED + 1))
                else
                    echo -e "${RED}✗ $test_file FAILED${NC}"
                fi
            fi
        done
        
        cd ../..
    else
        echo "No unit tests found for $TEMPLATE_ID"
    fi
    
    echo
}

# Function to run integration tests
run_integration_tests() {
    echo -e "${YELLOW}Running Integration Tests...${NC}"
    
    if [ -d "$TEMPLATE_ID/integration" ]; then
        cd "$TEMPLATE_ID/integration"
        
        for test_file in test_*.yml; do
            if [ -f "$test_file" ]; then
                echo "Running $test_file..."
                if ansible-playbook "$test_file" -v; then
                    echo -e "${GREEN}✓ $test_file PASSED${NC}"
                    INTEGRATION_TESTS_PASSED=$((INTEGRATION_TESTS_PASSED + 1))
                else
                    echo -e "${RED}✗ $test_file FAILED${NC}"
                fi
            fi
        done
        
        cd ../..
    else
        echo "No integration tests found for $TEMPLATE_ID"
    fi
    
    echo
}

# Function to run security tests
run_security_tests() {
    echo -e "${YELLOW}Running Security Tests...${NC}"
    
    if [ -d "$TEMPLATE_ID/security" ]; then
        cd "$TEMPLATE_ID/security"
        
        for test_file in test_*.py; do
            if [ -f "$test_file" ]; then
                echo "Running $test_file..."
                if python3 "$test_file"; then
                    echo -e "${GREEN}✓ $test_file PASSED${NC}"
                    SECURITY_TESTS_PASSED=$((SECURITY_TESTS_PASSED + 1))
                else
                    echo -e "${RED}✗ $test_file FAILED${NC}"
                fi
            fi
        done
        
        cd ../..
    else
        echo "No security tests found for $TEMPLATE_ID"
    fi
    
    echo
}

# Function to run performance tests
run_performance_tests() {
    echo -e "${YELLOW}Running Performance Tests...${NC}"
    
    if [ -d "$TEMPLATE_ID/performance" ]; then
        cd "$TEMPLATE_ID/performance"
        
        for test_file in test_*.py test_*.yml; do
            if [ -f "$test_file" ]; then
                echo "Running $test_file..."
                if [[ "$test_file" == *.py ]]; then
                    if python3 "$test_file"; then
                        echo -e "${GREEN}✓ $test_file PASSED${NC}"
                        PERFORMANCE_TESTS_PASSED=$((PERFORMANCE_TESTS_PASSED + 1))
                    else
                        echo -e "${RED}✗ $test_file FAILED${NC}"
                    fi
                elif [[ "$test_file" == *.yml ]]; then
                    if ansible-playbook "$test_file" -v; then
                        echo -e "${GREEN}✓ $test_file PASSED${NC}"
                        PERFORMANCE_TESTS_PASSED=$((PERFORMANCE_TESTS_PASSED + 1))
                    else
                        echo -e "${RED}✗ $test_file FAILED${NC}"
                    fi
                fi
            fi
        done
        
        cd ../..
    else
        echo "No performance tests found for $TEMPLATE_ID"
    fi
    
    echo
}

# Main test execution
case $TEST_TYPE in
    "unit")
        run_unit_tests
        ;;
    "integration")
        run_integration_tests
        ;;
    "security")
        run_security_tests
        ;;
    "performance")
        run_performance_tests
        ;;
    "all")
        run_unit_tests
        run_integration_tests
        run_security_tests
        run_performance_tests
        ;;
    *)
        echo "Unknown test type: $TEST_TYPE"
        echo "Valid types: unit, integration, security, performance, all"
        exit 1
        ;;
esac

# Test summary
echo "=== Test Summary ==="
echo "Template: $TEMPLATE_ID"
echo "Unit Tests Passed: $UNIT_TESTS_PASSED"
echo "Integration Tests Passed: $INTEGRATION_TESTS_PASSED"
echo "Security Tests Passed: $SECURITY_TESTS_PASSED"
echo "Performance Tests Passed: $PERFORMANCE_TESTS_PASSED"

TOTAL_TESTS=$((UNIT_TESTS_PASSED + INTEGRATION_TESTS_PASSED + SECURITY_TESTS_PASSED + PERFORMANCE_TESTS_PASSED))
echo "Total Tests Passed: $TOTAL_TESTS"

if [ $TOTAL_TESTS -gt 0 ]; then
    echo -e "${GREEN}Testing completed successfully!${NC}"
    exit 0
else
    echo -e "${RED}No tests passed or no tests found!${NC}"
    exit 1
fi
EOF

chmod +x template-registry/tests/run-tests.sh
```

Create test for template testing:

```yaml
# Create test-template-testing.yml
cat > test-template-testing.yml << 'EOF'
---
- name: Test template testing and quality assurance
  hosts: template_managers
  gather_facts: yes
  
  tasks:
    - name: Create test template for testing
      copy:
        content: |
          # {{ ansible_managed }}
          # Nginx Configuration Template for Testing
          
          user www-data;
          worker_processes {{ worker_processes | default('auto') }};
          
          events {
              worker_connections {{ worker_connections | default(1024) }};
          }
          
          http {
              include /etc/nginx/mime.types;
              default_type application/octet-stream;
              
              # Security headers
              add_header X-Frame-Options DENY;
              add_header X-Content-Type-Options nosniff;
              add_header X-XSS-Protection "1; mode=block";
              {% if ssl_enabled | default(false) %}
              add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
              {% endif %}
              
              # Hide server tokens
              server_tokens off;
              
              # Gzip configuration
              {% if gzip_enabled | default(true) %}
              gzip on;
              gzip_types text/plain text/css application/json application/javascript;
              {% else %}
              gzip off;
              {% endif %}
              
              # Access logging
              access_log /var/log/nginx/access.log;
              error_log /var/log/nginx/error.log;
              
              # Client settings
              client_max_body_size {{ client_max_body_size | default('64m') }};
              
              server {
                  listen 80{% if ssl_enabled | default(false) %} default_server{% endif %};
                  {% if ssl_enabled | default(false) %}
                  listen 443 ssl default_server;
                  ssl_protocols TLSv1.2 TLSv1.3;
                  ssl_certificate /etc/ssl/certs/{{ server_name }}.crt;
                  ssl_certificate_key /etc/ssl/private/{{ server_name }}.key;
                  {% endif %}
                  
                  server_name {{ server_name }};
                  root {{ document_root }};
                  index index.html index.htm;
                  
                  # Security - deny access to hidden files
                  location ~ /\. {
                      deny all;
                  }
                  
                  {% if monitoring_enabled | default(false) %}
                  # Monitoring endpoint
                  location /nginx_status {
                      stub_status on;
                      access_log off;
                      allow 127.0.0.1;
                      deny all;
                  }
                  {% endif %}
                  
                  location / {
                      try_files $uri $uri/ =404;
                  }
              }
          }
        dest: templates/testing/nginx-web-server-v2.2.0.j2
        mode: '0644'
    
    - name: Run unit tests
      shell: |
        cd template-registry/tests
        python3 nginx-web-server/unit/test_template_syntax.py
      register: unit_test_result
      failed_when: false
      changed_when: false
    
    - name: Run integration tests
      shell: |
        cd template-registry/tests
        ansible-playbook nginx-web-server/integration/test_integration.yml -v
      register: integration_test_result
      failed_when: false
      changed_when: false
    
    - name: Run security tests
      shell: |
        cd template-registry/tests
        python3 nginx-web-server/security/test_security.py
      register: security_test_result
      failed_when: false
      changed_when: false
    
    - name: Run all tests using test runner
      shell: |
        cd template-registry/tests
        ./run-tests.sh nginx-web-server all
      register: all_tests_result
      failed_when: false
      changed_when: false
    
    - name: Display testing results
      debug:
        msg: |
          Template Testing Results:
          
          Unit Tests: {{ 'PASSED' if unit_test_result.rc == 0 else 'FAILED' }}
          Integration Tests: {{ 'PASSED' if integration_test_result.rc == 0 else 'FAILED' }}
          Security Tests: {{ 'PASSED' if security_test_result.rc == 0 else 'FAILED' }}
          
          All Tests Summary:
          {{ all_tests_result.stdout }}
          
          Test Runner Exit Code: {{ all_tests_result.rc }}
          Overall Status: {{ 'PASSED' if all_tests_result.rc == 0 else 'FAILED' }}
EOF

# Run template testing test
ansible-playbook -i inventory.ini test-template-testing.yml
```

## Exercise 3: Template Governance and Deployment (7 minutes)

### Task: Implement template governance policies and automated deployment processes

Create governance policies:

```yaml
# Create governance/policies/template-governance-policy.yml
cat > governance/policies/template-governance-policy.yml << 'EOF'
---
# Template Governance Policy
policy_version: "1.0"
effective_date: "2024-01-15"
review_date: "2024-07-15"

governance_framework:
  name: "Enterprise Template Governance"
  description: "Comprehensive governance framework for Ansible template management"
  scope: "All Ansible templates used in production environments"
  
  principles:
    - "Security by Design"
    - "Quality Assurance"
    - "Change Management"
    - "Compliance and Audit"
    - "Continuous Improvement"

roles_and_responsibilities:
  template_authors:
    - "Develop templates following enterprise standards"
    - "Write comprehensive documentation"
    - "Implement required tests"
    - "Respond to review feedback"
    
  template_reviewers:
    - "Review template code for quality and security"
    - "Validate testing coverage"
    - "Ensure compliance with policies"
    - "Provide constructive feedback"
    
  template_approvers:
    - "Make final approval decisions"
    - "Ensure business alignment"
    - "Authorize production deployment"
    - "Manage template lifecycle"
    
  template_administrators:
    - "Maintain template registry"
    - "Manage template infrastructure"
    - "Monitor template usage"
    - "Enforce governance policies"

approval_workflow:
  development:
    - step: 1
      action: "Template Development"
      responsible: "Template Author"
      deliverables: ["Template Code", "Documentation", "Tests"]
      
    - step: 2
      action: "Self-Review"
      responsible: "Template Author"
      deliverables: ["Review Checklist", "Test Results"]
      
    - step: 3
      action: "Peer Review"
      responsible: "Template Reviewer"
      deliverables: ["Review Comments", "Quality Assessment"]
      
    - step: 4
      action: "Security Review"
      responsible: "Security Team"
      deliverables: ["Security Assessment", "Vulnerability Report"]
      required_for: ["production", "sensitive_data"]
      
    - step: 5
      action: "Architecture Review"
      responsible: "Architecture Team"
      deliverables: ["Architecture Assessment", "Standards Compliance"]
      required_for: ["infrastructure", "complex_templates"]
      
    - step: 6
      action: "Final Approval"
      responsible: "Template Approver"
      deliverables: ["Approval Decision", "Deployment Authorization"]

quality_gates:
  mandatory_checks:
    - name: "Syntax Validation"
      description: "Template must have valid Jinja2 syntax"
      automated: true
      blocking: true
      
    - name: "Security Scan"
      description: "Template must pass security vulnerability scan"
      automated: true
      blocking: true
      
    - name: "Unit Tests"
      description: "All unit tests must pass"
      automated: true
      blocking: true
      
    - name: "Integration Tests"
      description: "Integration tests must pass"
      automated: true
      blocking: true
      
    - name: "Documentation Review"
      description: "Template must have complete documentation"
      automated: false
      blocking: true
      
    - name: "Compliance Check"
      description: "Template must comply with enterprise standards"
      automated: false
      blocking: true

  optional_checks:
    - name: "Performance Tests"
      description: "Performance benchmarks should be met"
      automated: true
      blocking: false
      
    - name: "Accessibility Review"
      description: "Templates should follow accessibility guidelines"
      automated: false
      blocking: false

change_management:
  version_control:
    - "All templates must be version controlled"
    - "Semantic versioning must be used"
    - "Change history must be maintained"
    - "Rollback procedures must be documented"
    
  deployment_process:
    - "Templates must be deployed through automated pipeline"
    - "Deployment must be approved by authorized personnel"
    - "Rollback plan must be available"
    - "Deployment must be monitored and validated"
    
  emergency_procedures:
    - "Emergency template changes require expedited approval"
    - "Emergency changes must be documented and reviewed post-deployment"
    - "Emergency rollback procedures must be available"

compliance_requirements:
  audit_trail:
    - "All template changes must be logged"
    - "Approval decisions must be recorded"
    - "Usage statistics must be maintained"
    - "Access logs must be preserved"
    
  data_protection:
    - "Templates must not contain sensitive data"
    - "Data classification must be respected"
    - "Privacy requirements must be met"
    - "Data retention policies must be followed"
    
  regulatory_compliance:
    - "Industry-specific regulations must be considered"
    - "Compliance documentation must be maintained"
    - "Regular compliance audits must be conducted"

monitoring_and_metrics:
  template_metrics:
    - "Template usage statistics"
    - "Deployment success rates"
    - "Error rates and failure analysis"
    - "Performance metrics"
    
  governance_metrics:
    - "Approval cycle times"
    - "Review completion rates"
    - "Policy compliance scores"
    - "Training completion rates"
    
  reporting:
    - "Monthly governance reports"
    - "Quarterly compliance assessments"
    - "Annual policy reviews"
    - "Incident reports and lessons learned"

training_and_awareness:
  required_training:
    - "Template Development Best Practices"
    - "Security Awareness for Template Authors"
    - "Governance Policy Overview"
    - "Tool Usage and Procedures"
    
  ongoing_education:
    - "Regular lunch-and-learn sessions"
    - "Best practice sharing"
    - "Industry trend updates"
    - "Tool and technology updates"

policy_enforcement:
  automated_enforcement:
    - "Pipeline gates for quality checks"
    - "Automated policy compliance validation"
    - "Access control enforcement"
    - "Usage monitoring and alerting"
    
  manual_enforcement:
    - "Regular policy compliance audits"
    - "Peer review requirements"
    - "Management oversight"
    - "Corrective action procedures"

continuous_improvement:
  feedback_mechanisms:
    - "Template user feedback collection"
    - "Developer experience surveys"
    - "Process improvement suggestions"
    - "Incident analysis and lessons learned"
    
  policy_updates:
    - "Regular policy review and updates"
    - "Industry best practice adoption"
    - "Tool and technology evolution"
    - "Organizational change adaptation"
EOF
```

Create deployment automation:

```yaml
# Create governance/deployment/template-deployment.yml
cat > governance/deployment/template-deployment.yml << 'EOF'
---
# Template Deployment Automation
- name: Enterprise Template Deployment Pipeline
  hosts: localhost
  connection: local
  gather_facts: yes
  
  vars:
    template_id: "{{ template_id | mandatory }}"
    deployment_environment: "{{ deployment_environment | default('staging') }}"
    approval_required: "{{ approval_required | default(true) }}"
    rollback_enabled: "{{ rollback_enabled | default(true) }}"
    
    deployment_config:
      staging:
        target_path: "/opt/ansible/templates/staging"
        validation_required: true
        approval_bypass: false
        monitoring_enabled: true
        
      production:
        target_path: "/opt/ansible/templates/production"
        validation_required: true
        approval_bypass: false
        monitoring_enabled: true
        backup_required: true
  
  tasks:
    - name: Validate deployment parameters
      assert:
        that:
          - template_id is defined
          - template_id != ""
          - deployment_environment in ['staging', 'production']
        fail_msg: "Invalid deployment parameters"
        success_msg: "Deployment parameters validated"
    
    - name: Load template registry
      include_vars:
        file: "../../template-registry/registry.yml"
        name: template_registry
    
    - name: Find template in registry
      set_fact:
        template_info: "{{ item }}"
      loop: "{{ template_registry.templates }}"
      when: item.template_id == template_id
    
    - name: Validate template exists and is approved
      assert:
        that:
          - template_info is defined
          - template_info.status == 'approved' or deployment_environment == 'staging'
        fail_msg: "Template not found or not approved for {{ deployment_environment }}"
        success_msg: "Template validation passed"
    
    - name: Check approval requirements
      block:
        - name: Verify deployment approval
          assert:
            that:
              - not approval_required or template_info.approval_status is defined
            fail_msg: "Deployment approval required but not found"
            success_msg: "Deployment approval verified"
      when: deployment_environment == 'production'
    
    - name: Create backup of existing template
      block:
        - name: Check if template exists
          stat:
            path: "{{ deployment_config[deployment_environment].target_path }}/{{ template_id }}.j2"
          register: existing_template
        
        - name: Create backup
          copy:
            src: "{{ deployment_config[deployment_environment].target_path }}/{{ template_id }}.j2"
            dest: "{{ deployment_config[deployment_environment].target_path }}/backups/{{ template_id }}-{{ ansible_date_time.epoch }}.j2"
            mode: '0644'
          when: existing_template.stat.exists
      when: deployment_config[deployment_environment].backup_required | default(false)
    
    - name: Run pre-deployment validation
      block:
        - name: Run template tests
          shell: |
            cd ../../template-registry/tests
            ./run-tests.sh {{ template_id }} all
          register: test_results
          when: deployment_config[deployment_environment].validation_required
        
        - name: Validate test results
          assert:
            that:
              - test_results.rc == 0
            fail_msg: "Template tests failed - deployment aborted"
            success_msg: "Template tests passed"
          when: deployment_config[deployment_environment].validation_required
    
    - name: Deploy template
      block:
        - name: Copy template to target location
          copy:
            src: "{{ template_info.file_locations.template }}"
            dest: "{{ deployment_config[deployment_environment].target_path }}/{{ template_id }}.j2"
            mode: '0644'
            backup: yes
        
        - name: Update template metadata
          copy:
            content: |
              template_id: {{ template_id }}
              version: {{ template_info.version }}
              deployed_by: {{ ansible_user_id }}
              deployment_date: {{ ansible_date_time.iso8601 }}
              deployment_environment: {{ deployment_environment }}
              source_file: {{ template_info.file_locations.template }}
            dest: "{{ deployment_config[deployment_environment].target_path }}/{{ template_id }}.meta"
            mode: '0644'
    
    - name: Run post-deployment validation
      block:
        - name: Validate deployed template syntax
          shell: |
            python3 -c "
            from jinja2 import Environment, FileSystemLoader
            env = Environment(loader=FileSystemLoader('{{ deployment_config[deployment_environment].target_path }}'))
            template = env.get_template('{{ template_id }}.j2')
            print('Template syntax validation passed')
            "
          register: syntax_validation
        
        - name: Test template rendering
          shell: |
            python3 -c "
            from jinja2 import Environment, FileSystemLoader
            env = Environment(loader=FileSystemLoader('{{ deployment_config[deployment_environment].target_path }}'))
            template = env.get_template('{{ template_id }}.j2')
            result = template.render(
                server_name='test.example.com',
                document_root='/var/www/test',
                ansible_managed='Test',
                ansible_hostname='test-host',
                ansible_date_time={'iso8601': '2024-01-15T10:00:00Z'}
            )
            print('Template rendering test passed')
            "
          register: rendering_test
    
    - name: Update deployment tracking
      block:
        - name: Record deployment in registry
          shell: |
            cd ../../template-registry
            python3 -c "
            import yaml
            import sys
            from datetime import datetime
            
            # Load registry
            with open('registry.yml', 'r') as f:
                registry = yaml.safe_load(f)
            
            # Update deployment statistics
            for template in registry['templates']:
                if template['template_id'] == '{{ template_id }}':
                    if 'deployment_history' not in template:
                        template['deployment_history'] = []
                    
                    template['deployment_history'].append({
                        'environment': '{{ deployment_environment }}',
                        'version': template['version'],
                        'deployed_by': '{{ ansible_user_id }}',
                        'deployment_date': '{{ ansible_date_time.iso8601 }}',
                        'status': 'success'
                    })
                    
                    # Update usage statistics
                    if 'usage_statistics' not in template:
                        template['usage_statistics'] = {'deployments': 0}
                    template['usage_statistics']['deployments'] += 1
                    template['usage_statistics']['last_used'] = '{{ ansible_date_time.iso8601 }}'
                    break
            
            # Save registry
            with open('registry.yml', 'w') as f:
                yaml.dump(registry, f, default_flow_style=False, sort_keys=False)
            
            print('Deployment tracking updated')
            "
    
    - name: Setup monitoring
      block:
        - name: Create monitoring configuration
          copy:
            content: |
              template_id: {{ template_id }}
              version: {{ template_info.version }}
              environment: {{ deployment_environment }}
              monitoring_enabled: true
              health_check_interval: 300
              alert_on_failure: true
              deployment_date: {{ ansible_date_time.iso8601 }}
            dest: "{{ deployment_config[deployment_environment].target_path }}/monitoring/{{ template_id }}.yml"
            mode: '0644'
      when: deployment_config[deployment_environment].monitoring_enabled | default(false)
    
    - name: Display deployment summary
      debug:
        msg: |
          Template Deployment Summary:
          
          Template: {{ template_id }}
          Version: {{ template_info.version }}
          Environment: {{ deployment_environment }}
          Deployed By: {{ ansible_user_id }}
          Deployment Date: {{ ansible_date_time.iso8601 }}
          
          Deployment Status: SUCCESS
          
          Validation Results:
          - Syntax Validation: {{ 'PASSED' if syntax_validation.rc == 0 else 'FAILED' }}
          - Rendering Test: {{ 'PASSED' if rendering_test.rc == 0 else 'FAILED' }}
          
          Target Location: {{ deployment_config[deployment_environment].target_path }}/{{ template_id }}.j2
          Backup Created: {{ 'Yes' if deployment_config[deployment_environment].backup_required | default(false) else 'No' }}
          Monitoring Enabled: {{ 'Yes' if deployment_config[deployment_environment].monitoring_enabled | default(false) else 'No' }}
EOF
```

Create comprehensive test for governance:

```yaml
# Create test-template-governance.yml
cat > test-template-governance.yml << 'EOF'
---
- name: Test template governance and deployment
  hosts: template_managers
  gather_facts: yes
  
  tasks:
    - name: Create deployment directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - "/opt/ansible/templates/staging"
        - "/opt/ansible/templates/production"
        - "/opt/ansible/templates/staging/backups"
        - "/opt/ansible/templates/production/backups"
        - "/opt/ansible/templates/staging/monitoring"
        - "/opt/ansible/templates/production/monitoring"
    
    - name: Test template deployment to staging
      include: governance/deployment/template-deployment.yml
      vars:
        template_id: "nginx-web-server"
        deployment_environment: "staging"
        approval_required: false
    
    - name: Test template deployment to production
      include: governance/deployment/template-deployment.yml
      vars:
        template_id: "nginx-web-server"
        deployment_environment: "production"
        approval_required: true
    
    - name: Verify deployment artifacts
      stat:
        path: "{{ item }}"
      register: deployment_artifacts
      loop:
        - "/opt/ansible/templates/staging/nginx-web-server.j2"
        - "/opt/ansible/templates/staging/nginx-web-server.meta"
        - "/opt/ansible/templates/production/nginx-web-server.j2"
        - "/opt/ansible/templates/production/nginx-web-server.meta"
    
    - name: Generate governance compliance report
      shell: |
        cd template-registry
        python3 lifecycle-manager.py report
      register: compliance_report
      changed_when: false
    
    - name: Display governance test results
      debug:
        msg: |
          Template Governance Test Results:
          
          Deployment Artifacts:
          {% for artifact in deployment_artifacts.results %}
          - {{ artifact.item }}: {{ 'EXISTS' if artifact.stat.exists else 'MISSING' }}
          {% endfor %}
          
          Compliance Report:
          {{ compliance_report.stdout | from_json | to_nice_yaml }}
          
          Governance Framework Status:
          - Policy Version: 1.0
          - Approval Workflow: Implemented
          - Quality Gates: Configured
          - Change Management: Active
          - Compliance Monitoring: Enabled
          
          Template Lifecycle Management:
          - Registration: Automated
          - Approval: Workflow-based
          - Deployment: Pipeline-driven
          - Monitoring: Integrated
          - Retirement: Policy-driven
EOF

# Run governance test
ansible-playbook -i inventory.ini test-template-governance.yml
```

## Verification and Discussion

### 1. Check Results
```bash
# Check template registry
echo "=== Template Registry Status ==="
cd template-registry
python3 lifecycle-manager.py list 2>/dev/null || echo "Registry not available"

# Check test results
echo "=== Template Test Results ==="
ls -la tests/*/unit/ tests/*/integration/ tests/*/security/ 2>/dev/null || echo "No test results"

# Check deployment artifacts
echo "=== Deployment Artifacts ==="
ls -la /opt/ansible/templates/*/nginx-web-server.* 2>/dev/null || echo "No deployment artifacts"

# Check governance policies
echo "=== Governance Policies ==="
ls -la governance/policies/ governance/deployment/ 2>/dev/null || echo "No governance files"
```

### 2. Discussion Points
- How do you implement template governance in your current environment?
- What approval processes work best for your organization?
- How do you balance automation with manual oversight?
- What metrics do you use to measure template quality and adoption?

### 3. Clean Up
```bash
# Clean up test artifacts
sudo rm -rf /opt/ansible/templates/ 2>/dev/null || true
# Keep registry and governance files for reference
```

## Key Takeaways
- Template versioning enables proper lifecycle management
- Comprehensive testing ensures template quality and reliability
- Governance policies provide structure and consistency
- Automated deployment pipelines reduce errors and improve efficiency
- Quality gates prevent problematic templates from reaching production
- Monitoring and metrics enable continuous improvement

## Module 6 Summary

### What We Covered
1. **Advanced Jinja2 Syntax and Expressions** - Complex expressions, filters, and control structures
2. **Dynamic Configuration Generation** - Multi-format configurations and environment-specific patterns
3. **Template Inheritance and Macros** - Hierarchical templates and reusable components
4. **Data Transformation and Processing** - Complex data manipulation and enterprise integration
5. **Enterprise Template Management** - Versioning, testing, governance, and deployment

### Key Skills Developed
- Mastering advanced Jinja2 templating techniques
- Implementing dynamic configuration generation patterns
- Creating reusable template components and inheritance hierarchies
- Processing and transforming complex enterprise data
- Establishing enterprise-grade template management practices

### Next Steps
You now have comprehensive knowledge of advanced Jinja2 templating for enterprise automation. These skills enable you to create sophisticated, maintainable, and scalable configuration management solutions that meet enterprise requirements for quality, security, and governance.

# Weather App with Environment-Specific Templates

## Overview
This lab demonstrates environment-specific template deployment using Ansible. We'll create a weather application that shows mock data in development (Busteni) and real data in production (Bucharest).

## Duration: 45 minutes

## Learning Objectives
- Create environment-specific inventories
- Use conditional templating based on environment
- Deploy different configurations for dev/prod
- Integrate with external APIs conditionally

## Lab Setup

### Create Project Directory
```bash
mkdir weather-app-lab
cd weather-app-lab
mkdir templates
```

### Create Environment-Specific Inventory

Create `hosts.yml`:

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
    host1:
      ansible_host: 13.51.178.222
      ansible_user: ubuntu
    host2:
      ansible_host: 16.16.202.25
      ansible_user: ubuntu
  vars:
    ansible_user: ubuntu
    ansible_ssh_pass: LetMe1n!
    ansible_ssh_private_key_file: ~/.ssh/ansible_key
    ansible_python_interpreter: /usr/bin/python3
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
  children:
    # Development Environment
    development:
      hosts:
        host1:
          city_name: "Busteni"
          weather_data:
            temperature: 18
            condition: "Mountain Fresh"
            humidity: 75
            wind_speed: 2.5
            description: "Cool mountain air with light breeze"
            pressure: 1020
            visibility: 15
      vars:
        environment: development
        server_role: "web"
        environment_type: "dev"
        app_color: "#e8f4f8"
    
    # Production Environment  
    production:
      hosts:
        host2:
          city_name: "Bucharest"
          production_weather:
            temperature: 25
            condition: "Sunny"
            humidity: 60
            wind_speed: 3.2
            description: "Clear skies with gentle breeze"
            pressure: 1013
            visibility: 20
      vars:
        environment: production
        server_role: "web"
        environment_type: "prod"
        app_color: "#ffffff"
    
    # Group all web servers
    webservers:
      children:
        development:
        production:
```

### Create Weather App Template

Create `templates/weather-app.html.j2`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Weather Dashboard - {{ city_name }}</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            margin: 0;
            padding: 20px;
            background: linear-gradient(135deg, {{ app_color }}, #f0f8ff);
            min-height: 100vh;
        }
        .container {
            background-color: white;
            padding: 30px;
            border-radius: 12px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            max-width: 800px;
            margin: 0 auto;
        }
        .header {
            color: #2c3e50;
            text-align: center;
            margin-bottom: 30px;
            border-bottom: 3px solid #3498db;
            padding-bottom: 15px;
        }
        .weather-card {
            background: linear-gradient(135deg, #74b9ff, #0984e3);
            color: white;
            padding: 25px;
            border-radius: 12px;
            text-align: center;
            margin: 20px 0;
        }
        .weather-details {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
            gap: 15px;
            margin-top: 20px;
        }
        .detail-item {
            background: rgba(255,255,255,0.2);
            padding: 15px;
            border-radius: 8px;
            text-align: center;
        }
        .server-info {
            background: #f8f9fa;
            padding: 20px;
            border-radius: 8px;
            margin-top: 20px;
            border-left: 4px solid #3498db;
        }
        .env-badge {
            display: inline-block;
            padding: 5px 12px;
            border-radius: 20px;
            font-size: 0.8em;
            font-weight: bold;
            {% if environment == 'development' %}
            background-color: #f39c12;
            color: white;
            {% else %}
            background-color: #27ae60;
            color: white;
            {% endif %}
        }
        .footer {
            text-align: center;
            margin-top: 30px;
            padding-top: 20px;
            border-top: 1px solid #ecf0f1;
            color: #7f8c8d;
            font-size: 0.9em;
        }
        {% if environment == 'development' %}
        .dev-notice {
            background: #fff3cd;
            border: 1px solid #ffeaa7;
            padding: 15px;
            border-radius: 8px;
            margin: 20px 0;
            color: #856404;
        }
        {% endif %}
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🌤️ Weather Dashboard</h1>
            <p>
                <span class="env-badge">{{ environment | upper }}</span>
                Server: {{ ansible_hostname }}
            </p>
        </div>
        
        <div class="weather-card">
            <h2>{{ city_name }}</h2>
            <div id="weather-info">
                {% if environment == 'development' %}
                <!-- Mock Weather Data for Development -->
                <div class="weather-main">
                    <h3>{{ weather_data.temperature }}°C</h3>
                    <p>{{ weather_data.condition }}</p>
                    <p><em>{{ weather_data.description }}</em></p>
                </div>
                <div class="weather-details">
                    <div class="detail-item">
                        <strong>Humidity</strong><br>
                        {{ weather_data.humidity }}%
                    </div>
                    <div class="detail-item">
                        <strong>Wind</strong><br>
                        {{ weather_data.wind_speed }} m/s
                    </div>
                    <div class="detail-item">
                        <strong>Pressure</strong><br>
                        {{ weather_data.pressure }} hPa
                    </div>
                    <div class="detail-item">
                        <strong>Visibility</strong><br>
                        {{ weather_data.visibility }} km
                    </div>
                </div>
                {% elif environment == 'production' %}
                <!-- Static Weather Data for Production -->
                <div class="weather-main">
                    <h3>{{ production_weather.temperature }}°C</h3>
                    <p>{{ production_weather.condition }}</p>
                    <p><em>{{ production_weather.description }}</em></p>
                </div>
                <div class="weather-details">
                    <div class="detail-item">
                        <strong>Humidity</strong><br>
                        {{ production_weather.humidity }}%
                    </div>
                    <div class="detail-item">
                        <strong>Wind</strong><br>
                        {{ production_weather.wind_speed }} m/s
                    </div>
                    <div class="detail-item">
                        <strong>Pressure</strong><br>
                        {{ production_weather.pressure }} hPa
                    </div>
                    <div class="detail-item">
                        <strong>Visibility</strong><br>
                        {{ production_weather.visibility }} km
                    </div>
                </div>
                {% endif %}
            </div>
        </div>
        
        {% if environment == 'development' %}
        <div class="dev-notice">
            <h4>Development Mode</h4>
            <p>This is mock weather data for {{ city_name }}. In production, real weather data from OpenWeatherMap API will be displayed for {{ city_name }}.</p>
        </div>
        {% endif %}
        
        <div class="server-info">
            <h3>Server Information</h3>
            <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 15px;">
                <div>
                    <strong>Hostname:</strong> {{ ansible_hostname }}<br>
                    <strong>IP Address:</strong> {{ ansible_default_ipv4.address }}<br>
                    <strong>Environment:</strong> {{ environment }}
                </div>
                <div>
                    <strong>OS:</strong> {{ ansible_distribution }} {{ ansible_distribution_version }}<br>
                    <strong>Architecture:</strong> {{ ansible_architecture }}<br>
                    <strong>City:</strong> {{ city_name }}
                </div>
            </div>
        </div>
        
        <div class="footer">
            <p>Deployed by Ansible | Environment: {{ environment }} | {{ ansible_date_time.date }}</p>
            <p>Weather data {% if environment == 'production' %}simulated for {{ city_name }}{% else %}mocked for {{ city_name }}{% endif %}</p>
            <p><small>Refresh page to see updated deployment time</small></p>
        </div>
    </div>
</body>
</html>
```

### Create Deployment Playbook

Create `deploy-weather-app.yml`:

```yaml
---
- name: Deploy Weather App with Environment-Specific Configuration
  hosts: webservers
  become: yes
  vars:
    app_name: "Weather Dashboard"
    app_version: "1.0.0"
    
  tasks:
    - name: Install NGINX web server
      package:
        name: nginx
        state: present
        
    - name: Start and enable NGINX
      service:
        name: nginx
        state: started
        enabled: yes
        
    - name: Deploy environment-specific weather app
      template:
        src: weather-app.html.j2
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
        backup: yes
      register: app_deploy
      
    - name: Display deployment summary
      debug:
        msg: |
          =================================
          Weather App Deployment Summary
          =================================
          Environment: {{ environment }}
          City: {{ city_name }}
          Server: {{ ansible_hostname }}
          IP: {{ ansible_default_ipv4.address }}
          App URL: http://{{ ansible_host }}
          Weather Data: {% if environment == 'production' %}Static ({{ production_weather.condition }}){% elif environment == 'development' %}Mock ({{ weather_data.condition }}){% else %}Unknown{% endif %}
          Template Changed: {{ app_deploy.changed }}
          =================================
```

## Lab Exercises

### Exercise 1: Deploy to Development Environment

```bash
# Deploy to development (Busteni with mock data)
ansible-playbook -i hosts.yml deploy-weather-app.yml --limit development

# Check the result
curl http://13.51.178.222/
```

**Expected Result:**
- Mock weather data for Busteni
- Development environment badge (orange)
- Mountain weather conditions (18°C, Mountain Fresh)
- Light blue background

### Exercise 2: Deploy to Production Environment

```bash
# Deploy to production (Bucharest with static data)
ansible-playbook -i hosts.yml deploy-weather-app.yml --limit production

# Check the result
curl http://16.16.202.25/
```

**Expected Result:**
- Static weather data for Bucharest
- Production environment badge (green)
- Sunny conditions (25°C, Clear skies)
- White background

### Exercise 3: Deploy to Both Environments

```bash
# Deploy to all environments
ansible-playbook -i hosts.yml deploy-weather-app.yml

# Compare the results
echo "=== Development (Busteni) ==="
curl -s http://13.51.178.222/ | grep -A 5 "weather-main"

echo "=== Production (Bucharest) ==="
curl -s http://16.16.202.25/ | grep -A 5 "weather-main"
```

### Exercise 4: Verify Environment Differences

```bash
# Check development server
echo "Development Environment:"
echo "- City: Busteni (mock data)"
echo "- Color scheme: Light blue"
echo "- Weather: 18°C, Mountain Fresh"

# Check production server  
echo "Production Environment:"
echo "- City: Bucharest (static data)"
echo "- Color scheme: White"
echo "- Weather: 25°C, Sunny"
```

## Environment Comparison

| Aspect | Development (host1: 13.51.178.222) | Production (host2: 16.16.202.25) |
|--------|-------------------------------------|-----------------------------------|
| City | Busteni | Bucharest |
| Weather Data | Mock (18°C, Mountain Fresh) | Static (25°C, Sunny) |
| Background Color | Light Blue (#e8f4f8) | White (#ffffff) |
| Badge Color | Orange (DEV) | Green (PROD) |
| Data Update | Manual refresh only | Manual refresh only |
| Complexity | Simple static template | Simple static template |

## Key Learning Points

1. **Environment-Specific Variables**: Different inventory groups can have different variables
2. **Conditional Templating**: Use `{% if %}` statements to render different content
3. **Host-Specific Data**: Each host can have unique configuration data
4. **Conditional Task Execution**: Tasks can run only on specific environments
5. **Template Flexibility**: Single template can generate different outputs based on variables


## Conclusion

This lab demonstrates how Ansible templates can be used to create environment-specific deployments. The same template generates different outputs based on inventory variables, enabling consistent deployment processes across different environments while maintaining environment-specific configurations.

# 🚀 Ansible Guide for Senior DevOps Engineers

## Table of Contents
1. [Ansible Foundations](#ansible-foundations)
2. [Playbooks - Core Concepts](#playbooks---core-concepts)
3. [Control Flow and Execution](#control-flow-and-execution)
4. [Security and Organization](#security-and-organization)
5. [Integrations](#integrations)
6. [Extending Ansible](#extending-ansible)
7. [Advanced DevOps Patterns](#advanced-devops-patterns)

---

## 1. Ansible Foundations

### Introduction to Ansible Configuration

```yaml
# ansible.cfg - Configuration hierarchy (highest to lowest priority)
# 1. ANSIBLE_CONFIG environment variable
# 2. ./ansible.cfg (current directory)
# 3. ~/.ansible.cfg (home directory)
# 4. /etc/ansible/ansible.cfg (system-wide)

[defaults]
inventory = ./inventory
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 86400
callback_whitelist = timer, profile_tasks

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
```

### Understanding Inventories

#### Static Inventory
```ini
# inventory/hosts
[webservers]
web1.example.com ansible_host=192.168.1.10
web2.example.com ansible_host=192.168.1.11

[databases]
db1.example.com ansible_host=192.168.1.20
db2.example.com ansible_host=192.168.1.21

[production:children]
webservers
databases

[production:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/production.pem
```

#### Dynamic Inventory (AWS)
```python
#!/usr/bin/env python3
# inventory/aws_ec2.py
import boto3
import json

def get_inventory():
    ec2 = boto3.client('ec2')
    instances = ec2.describe_instances()
    
    inventory = {
        '_meta': {'hostvars': {}},
        'webservers': {'hosts': []},
        'databases': {'hosts': []}
    }
    
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            if instance['State']['Name'] == 'running':
                name = instance.get('PublicDnsName', instance['InstanceId'])
                tags = {tag['Key']: tag['Value'] for tag in instance.get('Tags', [])}
                
                if tags.get('Role') == 'web':
                    inventory['webservers']['hosts'].append(name)
                elif tags.get('Role') == 'db':
                    inventory['databases']['hosts'].append(name)
                
                inventory['_meta']['hostvars'][name] = {
                    'ansible_host': instance.get('PublicIpAddress'),
                    'instance_type': instance['InstanceType'],
                    'environment': tags.get('Environment', 'unknown')
                }
    
    return inventory

if __name__ == '__main__':
    print(json.dumps(get_inventory(), indent=2))
```

### Essential Ansible Modules

```yaml
# Core modules every DevOps engineer should master
- name: File operations
  file:
    path: /etc/myapp
    state: directory
    owner: myapp
    group: myapp
    mode: '0755'

- name: Template configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    backup: yes
  notify: restart nginx

- name: Package management
  package:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - postgresql
    - redis

- name: Service management
  systemd:
    name: nginx
    state: started
    enabled: yes
    daemon_reload: yes

- name: Command execution
  command: /usr/bin/make install
  args:
    chdir: /tmp/myapp
    creates: /usr/local/bin/myapp

- name: Shell with pipes
  shell: |
    ps aux | grep nginx | grep -v grep | wc -l
  register: nginx_processes

- name: Copy files
  copy:
    src: files/app.conf
    dest: /etc/myapp/app.conf
    owner: root
    group: root
    mode: '0644'
```

---

## 2. Playbooks - Core Concepts

### Anatomy of an Ansible Playbook

```yaml
---
# playbook.yml - Complete structure breakdown
- name: Deploy web application                    # Play name
  hosts: webservers                              # Target hosts
  become: yes                                    # Privilege escalation
  gather_facts: yes                              # Collect system info
  vars:                                          # Play-level variables
    app_name: mywebapp
    app_version: "1.2.3"
  vars_files:                                    # External variable files
    - vars/common.yml
    - "vars/{{ environment }}.yml"
  pre_tasks:                                     # Tasks run before roles
    - name: Update package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
  roles:                                         # Roles to execute
    - common
    - nginx
    - application
  tasks:                                         # Additional tasks
    - name: Ensure application is running
      uri:
        url: "http://{{ inventory_hostname }}/health"
        method: GET
        status_code: 200
  post_tasks:                                    # Tasks run after roles
    - name: Send deployment notification
      mail:
        to: devops@company.com
        subject: "Deployment completed on {{ inventory_hostname }}"
  handlers:                                      # Event-driven tasks
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```

### Variables in Playbooks

```yaml
# Variable precedence (highest to lowest):
# 1. Extra vars (-e)
# 2. Task vars
# 3. Block vars
# 4. Role and include vars
# 5. Play vars
# 6. Host facts
# 7. Playbook vars_files
# 8. Playbook vars_prompt
# 9. Playbook vars
# 10. Host vars
# 11. Group vars
# 12. Default vars

# vars/common.yml
app_config:
  database:
    host: "{{ db_host | default('localhost') }}"
    port: "{{ db_port | default(5432) }}"
    name: "{{ app_name }}_{{ environment }}"
  redis:
    host: "{{ redis_host | default('localhost') }}"
    port: 6379

# Using variables in tasks
- name: Configure application
  template:
    src: app.conf.j2
    dest: "/etc/{{ app_name }}/app.conf"
  vars:
    config_template_vars:
      db_connection: "postgresql://{{ app_config.database.host }}:{{ app_config.database.port }}/{{ app_config.database.name }}"
```

### Facts and System Information

```yaml
# Gathering and using facts
- name: Display system information
  debug:
    msg: |
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      Architecture: {{ ansible_architecture }}
      Memory: {{ ansible_memtotal_mb }}MB
      CPU Cores: {{ ansible_processor_vcpus }}
      Hostname: {{ ansible_hostname }}
      IP Address: {{ ansible_default_ipv4.address }}

# Custom facts
- name: Create custom fact directory
  file:
    path: /etc/ansible/facts.d
    state: directory

- name: Deploy custom fact
  copy:
    content: |
      #!/bin/bash
      echo '{"deployment_date":"'$(date -Iseconds)'","deployed_by":"'$USER'"}'
    dest: /etc/ansible/facts.d/deployment.fact
    mode: '0755'

# Using custom facts
- name: Show deployment info
  debug:
    msg: "Last deployed: {{ ansible_local.deployment.deployment_date }}"
```

### Templating with Jinja2

```jinja2
{# templates/nginx.conf.j2 #}
user {{ nginx_user | default('www-data') }};
worker_processes {{ ansible_processor_vcpus }};

upstream {{ app_name }}_backend {
{% for host in groups['webservers'] %}
    server {{ hostvars[host]['ansible_default_ipv4']['address'] }}:8000;
{% endfor %}
}

server {
    listen 80;
    server_name {{ server_name | default(ansible_fqdn) }};
    
    location / {
        proxy_pass http://{{ app_name }}_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
{% if ssl_enabled | default(false) %}
    listen 443 ssl;
    ssl_certificate {{ ssl_cert_path }};
    ssl_certificate_key {{ ssl_key_path }};
{% endif %}
}

{# Conditional blocks #}
{% if environment == 'production' %}
# Production-specific configuration
client_max_body_size 100M;
{% else %}
# Development configuration
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log debug;
{% endif %}

{# Loops with filters #}
{% for upstream in app_upstreams | selectattr('enabled') %}
upstream {{ upstream.name }} {
{% for server in upstream.servers %}
    server {{ server.host }}:{{ server.port }}{% if server.weight is defined %} weight={{ server.weight }}{% endif %};
{% endfor %}
}
{% endfor %}
```

---

## 3. Control Flow and Execution

### Dynamic Inventories Advanced

```yaml
# Using dynamic groups
- name: Create dynamic groups based on facts
  group_by:
    key: "os_{{ ansible_distribution | lower }}"

- name: Ubuntu-specific tasks
  apt:
    name: ubuntu-specific-package
    state: present
  when: inventory_hostname in groups['os_ubuntu']

# Add hosts dynamically
- name: Add discovered database servers
  add_host:
    name: "{{ item.hostname }}"
    groups: discovered_databases
    ansible_host: "{{ item.ip }}"
  loop: "{{ discovered_db_servers }}"
```

### Conditional Tasks and Loops

```yaml
# Complex conditionals
- name: Install package based on multiple conditions
  package:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - postgresql-client
  when:
    - ansible_distribution in ['Ubuntu', 'Debian']
    - ansible_distribution_version is version('18.04', '>=')
    - inventory_hostname in groups['webservers']

# Register and conditional execution
- name: Check if application is installed
  stat:
    path: /usr/local/bin/myapp
  register: app_binary

- name: Download application
  get_url:
    url: "https://releases.example.com/myapp-{{ app_version }}.tar.gz"
    dest: /tmp/myapp.tar.gz
  when: not app_binary.stat.exists

# Loop with conditionals
- name: Configure services
  template:
    src: "{{ item.name }}.conf.j2"
    dest: "/etc/{{ item.name }}/{{ item.name }}.conf"
  loop:
    - { name: nginx, enabled: true }
    - { name: apache, enabled: false }
    - { name: haproxy, enabled: true }
  when: item.enabled
  notify: "restart {{ item.name }}"

# Loop with complex data
- name: Create users with SSH keys
  user:
    name: "{{ item.username }}"
    groups: "{{ item.groups | join(',') }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
  loop: "{{ users }}"
  loop_control:
    label: "{{ item.username }}"  # Only show username in output

- name: Add SSH keys for users
  authorized_key:
    user: "{{ item.0.username }}"
    key: "{{ item.1 }}"
  loop: "{{ users | subelements('ssh_keys', skip_missing=True) }}"
```

### Asynchronous and Parallel Execution

```yaml
# Asynchronous tasks
- name: Long running deployment
  command: /usr/local/bin/deploy-app.sh
  async: 300  # Maximum time to wait
  poll: 0     # Don't wait, fire and forget
  register: deploy_job

- name: Check deployment status
  async_status:
    jid: "{{ deploy_job.ansible_job_id }}"
  register: deploy_result
  until: deploy_result.finished
  retries: 30
  delay: 10

# Serial execution control
- name: Rolling deployment
  hosts: webservers
  serial: 2  # Deploy to 2 servers at a time
  tasks:
    - name: Remove from load balancer
      uri:
        url: "http://{{ load_balancer }}/remove/{{ inventory_hostname }}"
        method: POST
    
    - name: Deploy application
      include_tasks: deploy.yml
    
    - name: Add back to load balancer
      uri:
        url: "http://{{ load_balancer }}/add/{{ inventory_hostname }}"
        method: POST

# Percentage-based serial execution
- name: Gradual rollout
  hosts: webservers
  serial: "25%"  # Deploy to 25% of servers at a time
```

### Task Delegation

```yaml
# Delegate tasks to different hosts
- name: Update DNS record
  uri:
    url: "https://api.cloudflare.com/client/v4/zones/{{ zone_id }}/dns_records/{{ record_id }}"
    method: PUT
    headers:
      Authorization: "Bearer {{ cloudflare_token }}"
    body_format: json
    body:
      content: "{{ ansible_default_ipv4.address }}"
  delegate_to: localhost
  run_once: true

# Delegate to specific host
- name: Backup database
  postgresql_db:
    name: "{{ app_database }}"
    state: dump
    target: "/backups/{{ app_database }}-{{ ansible_date_time.date }}.sql"
  delegate_to: "{{ groups['databases'][0] }}"

# Local actions
- name: Generate SSL certificate locally
  command: openssl req -new -x509 -days 365 -nodes -out {{ cert_path }} -keyout {{ key_path }}
  delegate_to: localhost
  become: no
```

### Magic Variables

```yaml
# Essential magic variables
- name: Display magic variables
  debug:
    msg: |
      Current host: {{ inventory_hostname }}
      All hosts: {{ groups['all'] }}
      Web servers: {{ groups['webservers'] }}
      Current group: {{ group_names }}
      Hostvars for web1: {{ hostvars['web1.example.com'] }}
      Play hosts: {{ play_hosts }}
      Ansible version: {{ ansible_version.full }}

# Using hostvars for cross-host communication
- name: Configure load balancer with all web servers
  template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
  vars:
    backend_servers: |
      {% for host in groups['webservers'] %}
      server {{ host }} {{ hostvars[host]['ansible_default_ipv4']['address'] }}:80 check
      {% endfor %}
```

### Blocks and Error Handling

```yaml
# Block with error handling
- name: Application deployment block
  block:
    - name: Stop application
      systemd:
        name: myapp
        state: stopped
    
    - name: Deploy new version
      unarchive:
        src: "myapp-{{ app_version }}.tar.gz"
        dest: /opt/myapp
        remote_src: yes
    
    - name: Update configuration
      template:
        src: app.conf.j2
        dest: /etc/myapp/app.conf
    
    - name: Start application
      systemd:
        name: myapp
        state: started
  
  rescue:
    - name: Rollback to previous version
      command: /usr/local/bin/rollback-app.sh
    
    - name: Start application with old version
      systemd:
        name: myapp
        state: started
    
    - name: Send failure notification
      mail:
        to: devops@company.com
        subject: "Deployment failed on {{ inventory_hostname }}"
        body: "Deployment of {{ app_version }} failed. Rolled back to previous version."
  
  always:
    - name: Cleanup temporary files
      file:
        path: /tmp/myapp-{{ app_version }}.tar.gz
        state: absent
  
  when: inventory_hostname in groups['webservers']
```

---

## 4. Security and Organization

### Ansible Vault

```bash
# Create encrypted file
ansible-vault create secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Encrypt existing file
ansible-vault encrypt vars/production.yml

# Decrypt file
ansible-vault decrypt vars/production.yml

# View encrypted file
ansible-vault view secrets.yml

# Change vault password
ansible-vault rekey secrets.yml
```

```yaml
# secrets.yml (encrypted)
$ANSIBLE_VAULT;1.1;AES256
66386439653...

# Using vault in playbooks
- name: Deploy with secrets
  hosts: production
  vars_files:
    - secrets.yml
  tasks:
    - name: Configure database connection
      template:
        src: database.conf.j2
        dest: /etc/myapp/database.conf
      vars:
        db_password: "{{ vault_db_password }}"

# Inline vault variables
- name: Set secret variable
  set_fact:
    api_key: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      66386439653837613...
```

### Using Includes and Imports

```yaml
# main.yml - Orchestration playbook
---
- name: Infrastructure setup
  import_playbook: infrastructure.yml

- name: Application deployment
  import_playbook: application.yml
  when: deploy_app | default(true)

# tasks/main.yml - Task organization
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: Include installation tasks
  include_tasks: install.yml

- name: Include configuration tasks
  include_tasks: configure.yml
  when: configure_app | default(true)

# Dynamic includes
- name: Include environment-specific tasks
  include_tasks: "{{ environment }}.yml"
  vars:
    task_environment: "{{ environment }}"

# Import vs Include differences:
# import_*: Static, processed at parse time, cannot use variables in filename
# include_*: Dynamic, processed at runtime, can use variables in filename
```

### Tags for Selective Execution

```yaml
# Using tags strategically
- name: Install packages
  package:
    name: "{{ item }}"
    state: present
  loop: "{{ required_packages }}"
  tags:
    - packages
    - install
    - never  # Only run when explicitly called

- name: Configure application
  template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
  tags:
    - config
    - app-config
  notify: restart application

- name: Deploy application code
  git:
    repo: "{{ app_repo }}"
    dest: /opt/myapp
    version: "{{ app_version }}"
  tags:
    - deploy
    - code

# Run specific tags
# ansible-playbook site.yml --tags "config"
# ansible-playbook site.yml --tags "install,config"
# ansible-playbook site.yml --skip-tags "deploy"
```

### Roles for Modular Automation

```yaml
# roles/nginx/meta/main.yml
---
dependencies:
  - role: common
    vars:
      common_packages:
        - curl
        - wget

galaxy_info:
  author: DevOps Team
  description: Nginx web server role
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions:
        - bionic
        - focal

# roles/nginx/defaults/main.yml
---
nginx_user: www-data
nginx_worker_processes: auto
nginx_sites:
  - name: default
    template: default.conf.j2
    enabled: true

# roles/nginx/vars/main.yml
---
nginx_package_name: nginx
nginx_service_name: nginx
nginx_config_path: /etc/nginx

# roles/nginx/handlers/main.yml
---
- name: restart nginx
  systemd:
    name: "{{ nginx_service_name }}"
    state: restarted

- name: reload nginx
  systemd:
    name: "{{ nginx_service_name }}"
    state: reloaded

# roles/nginx/tasks/main.yml
---
- name: Install nginx
  package:
    name: "{{ nginx_package_name }}"
    state: present

- name: Configure nginx sites
  template:
    src: "{{ item.template }}"
    dest: "{{ nginx_config_path }}/sites-available/{{ item.name }}"
  loop: "{{ nginx_sites }}"
  notify: reload nginx

- name: Enable nginx sites
  file:
    src: "{{ nginx_config_path }}/sites-available/{{ item.name }}"
    dest: "{{ nginx_config_path }}/sites-enabled/{{ item.name }}"
    state: link
  loop: "{{ nginx_sites }}"
  when: item.enabled | default(true)
  notify: reload nginx
```

---

## 5. Integrations

### Automating AWS with Ansible

```yaml
# AWS infrastructure provisioning
- name: Create VPC and infrastructure
  hosts: localhost
  gather_facts: no
  vars:
    aws_region: us-west-2
    vpc_cidr: 10.0.0.0/16
  tasks:
    - name: Create VPC
      amazon.aws.ec2_vpc_net:
        name: "{{ project_name }}-vpc"
        cidr_block: "{{ vpc_cidr }}"
        region: "{{ aws_region }}"
        tags:
          Environment: "{{ environment }}"
        state: present
      register: vpc

    - name: Create public subnet
      amazon.aws.ec2_vpc_subnet:
        vpc_id: "{{ vpc.vpc.id }}"
        cidr: 10.0.1.0/24
        region: "{{ aws_region }}"
        az: "{{ aws_region }}a"
        tags:
          Name: "{{ project_name }}-public-subnet"
          Type: public
        state: present
      register: public_subnet

    - name: Create security group
      amazon.aws.ec2_security_group:
        name: "{{ project_name }}-web-sg"
        description: Web server security group
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports:
              - 80
              - 443
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            ports:
              - 22
            cidr_ip: "{{ admin_cidr }}"
        tags:
          Environment: "{{ environment }}"
      register: web_sg

    - name: Launch EC2 instances
      amazon.aws.ec2_instance:
        name: "{{ project_name }}-web-{{ item }}"
        image_id: "{{ ami_id }}"
        instance_type: "{{ instance_type }}"
        key_name: "{{ key_pair_name }}"
        vpc_subnet_id: "{{ public_subnet.subnet.id }}"
        security_groups:
          - "{{ web_sg.group_id }}"
        tags:
          Environment: "{{ environment }}"
          Role: webserver
        state: present
      loop: "{{ range(1, web_server_count + 1) | list }}"
      register: ec2_instances

    - name: Add instances to inventory
      add_host:
        name: "{{ item.instances[0].public_dns_name }}"
        groups: webservers
        ansible_host: "{{ item.instances[0].public_ip_address }}"
        ansible_user: ubuntu
        ansible_ssh_private_key_file: "{{ ssh_key_path }}"
      loop: "{{ ec2_instances.results }}"
```

### Managing Docker with Ansible

```yaml
# Docker container management
- name: Deploy containerized application
  hosts: docker_hosts
  become: yes
  vars:
    app_name: mywebapp
    app_version: "1.2.3"
    container_port: 8080
    host_port: 80
  tasks:
    - name: Install Docker
      package:
        name: docker.io
        state: present

    - name: Start Docker service
      systemd:
        name: docker
        state: started
        enabled: yes

    - name: Pull application image
      community.docker.docker_image:
        name: "myregistry/{{ app_name }}"
        tag: "{{ app_version }}"
        source: pull

    - name: Create application network
      community.docker.docker_network:
        name: "{{ app_name }}_network"

    - name: Deploy database container
      community.docker.docker_container:
        name: "{{ app_name }}_db"
        image: postgres:13
        env:
          POSTGRES_DB: "{{ app_name }}"
          POSTGRES_USER: "{{ db_user }}"
          POSTGRES_PASSWORD: "{{ db_password }}"
        volumes:
          - "{{ app_name }}_db_data:/var/lib/postgresql/data"
        networks:
          - name: "{{ app_name }}_network"
        restart_policy: unless-stopped

    - name: Deploy application container
      community.docker.docker_container:
        name: "{{ app_name }}_app"
        image: "myregistry/{{ app_name }}:{{ app_version }}"
        ports:
          - "{{ host_port }}:{{ container_port }}"
        env:
          DATABASE_URL: "postgresql://{{ db_user }}:{{ db_password }}@{{ app_name }}_db/{{ app_name }}"
        networks:
          - name: "{{ app_name }}_network"
        restart_policy: unless-stopped
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:{{ container_port }}/health"]
          interval: 30s
          timeout: 10s
          retries: 3

# Docker Compose management
- name: Deploy with Docker Compose
  hosts: docker_hosts
  tasks:
    - name: Create compose directory
      file:
        path: "/opt/{{ app_name }}"
        state: directory

    - name: Deploy docker-compose.yml
      template:
        src: docker-compose.yml.j2
        dest: "/opt/{{ app_name }}/docker-compose.yml"

    - name: Deploy environment file
      template:
        src: .env.j2
        dest: "/opt/{{ app_name }}/.env"
        mode: '0600'

    - name: Start services with docker-compose
      community.docker.docker_compose:
        project_src: "/opt/{{ app_name }}"
        state: present
        restarted: yes
```

---

## 6. Extending Ansible

### Creating Custom Modules

```python
#!/usr/bin/python
# library/custom_service_check.py

from ansible.module_utils.basic import AnsibleModule
import requests
import time

def check_service_health(url, timeout, retries):
    """Check if service is healthy"""
    for attempt in range(retries):
        try:
            response = requests.get(url, timeout=timeout)
            if response.status_code == 200:
                return True, f"Service healthy (attempt {attempt + 1})"
        except requests.RequestException as e:
            if attempt == retries - 1:
                return False, f"Service unhealthy after {retries} attempts: {str(e)}"
            time.sleep(2)
    return False, "Service check failed"

def main():
    module = AnsibleModule(
        argument_spec=dict(
            url=dict(type='str', required=True),
            timeout=dict(type='int', default=10),
            retries=dict(type='int', default=3),
            expected_status=dict(type='int', default=200)
        ),
        supports_check_mode=True
    )
    
    url = module.params['url']
    timeout = module.params['timeout']
    retries = module.params['retries']
    
    if module.check_mode:
        module.exit_json(changed=False, msg="Check mode - would check service health")
    
    is_healthy, message = check_service_health(url, timeout, retries)
    
    if is_healthy:
        module.exit_json(changed=False, msg=message, healthy=True)
    else:
        module.fail_json(msg=message, healthy=False)

if __name__ == '__main__':
    main()
```

```yaml
# Using custom module
- name: Check application health
  custom_service_check:
    url: "http://{{ inventory_hostname }}/health"
    timeout: 5
    retries: 3
  register: health_check

- name: Fail if service is unhealthy
  fail:
    msg: "Service is not healthy: {{ health_check.msg }}"
  when: not health_check.healthy
```

### Creating Custom Plugins

```python
# filter_plugins/custom_filters.py

def extract_version(package_string):
    """Extract version from package string like 'nginx-1.18.0-1ubuntu1'"""
    import re
    match = re.search(r'-(\d+\.\d+\.\d+)', package_string)
    return match.group(1) if match else None

def generate_password(length=12, include_symbols=True):
    """Generate random password"""
    import random
    import string
    
    chars = string.ascii_letters + string.digits
    if include_symbols:
        chars += "!@#$%^&*"
    
    return ''.join(random.choice(chars) for _ in range(length))

class FilterModule(object):
    def filters(self):
        return {
            'extract_version': extract_version,
            'generate_password': generate_password
        }
```

```yaml
# Using custom filters
- name: Extract package version
  debug:
    msg: "Nginx version: {{ ansible_facts.packages.nginx[0] | extract_version }}"

- name: Generate secure password
  set_fact:
    db_password: "{{ '' | generate_password(16, true) }}"
```

---

## 7. Advanced DevOps Patterns

### Blue-Green Deployment

```yaml
- name: Blue-Green Deployment
  hosts: webservers
  vars:
    current_color: "{{ 'blue' if active_deployment == 'green' else 'green' }}"
    inactive_color: "{{ 'green' if active_deployment == 'blue' else 'blue' }}"
  tasks:
    - name: Deploy to inactive environment
      include_tasks: deploy_app.yml
      vars:
        deployment_color: "{{ inactive_color }}"
        app_port: "{{ 8080 if inactive_color == 'blue' else 8081 }}"

    - name: Health check inactive environment
      uri:
        url: "http://{{ inventory_hostname }}:{{ app_port }}/health"
        status_code: 200
      retries: 10
      delay: 5

    - name: Switch load balancer to new version
      template:
        src: nginx_upstream.conf.j2
        dest: /etc/nginx/conf.d/upstream.conf
      vars:
        active_port: "{{ app_port }}"
      notify: reload nginx

    - name: Update active deployment marker
      set_fact:
        active_deployment: "{{ inactive_color }}"
```

### Canary Deployment

```yaml
- name: Canary Deployment
  hosts: webservers
  serial: 1
  vars:
    canary_percentage: 10
    total_servers: "{{ groups['webservers'] | length }}"
    canary_servers: "{{ (total_servers * canary_percentage / 100) | round | int }}"
  tasks:
    - name: Deploy canary version
      include_tasks: deploy_app.yml
      when: ansible_play_hosts_all.index(inventory_hostname) < canary_servers

    - name: Monitor canary metrics
      uri:
        url: "http://monitoring.example.com/api/metrics"
        method: GET
      register: metrics
      delegate_to: localhost
      run_once: true

    - name: Validate canary success
      assert:
        that:
          - metrics.json.error_rate < 0.01
          - metrics.json.response_time < 500
        fail_msg: "Canary deployment failed validation"

    - name: Continue full deployment
      include_tasks: deploy_app.yml
      when: ansible_play_hosts_all.index(inventory_hostname) >= canary_servers
```

### Infrastructure Testing

```yaml
- name: Infrastructure Testing
  hosts: all
  tasks:
    - name: Test connectivity
      wait_for_connection:
        timeout: 30

    - name: Verify required ports are open
      wait_for:
        port: "{{ item }}"
        host: "{{ inventory_hostname }}"
        timeout: 10
      loop: "{{ required_ports }}"
      delegate_to: localhost

    - name: Check disk space
      assert:
        that:
          - ansible_mounts | selectattr('mount', 'equalto', '/') | map(attribute='size_available') | first > 1000000000
        fail_msg: "Insufficient disk space on {{ inventory_hostname }}"

    - name: Validate service endpoints
      uri:
        url: "http://{{ inventory_hostname }}:{{ item.port }}{{ item.path }}"
        status_code: "{{ item.expected_status | default(200) }}"
      loop: "{{ service_endpoints }}"
      when: inventory_hostname in groups[item.group]
```

### Monitoring Integration

```yaml
- name: Setup monitoring
  hosts: all
  tasks:
    - name: Install monitoring agent
      package:
        name: "{{ monitoring_agent_package }}"
        state: present

    - name: Configure monitoring
      template:
        src: "{{ monitoring_agent }}.conf.j2"
        dest: "/etc/{{ monitoring_agent }}/{{ monitoring_agent }}.conf"
      notify: restart monitoring agent

    - name: Register host with monitoring system
      uri:
        url: "{{ monitoring_api_url }}/hosts"
        method: POST
        body_format: json
        body:
          hostname: "{{ inventory_hostname }}"
          ip: "{{ ansible_default_ipv4.address }}"
          environment: "{{ environment }}"
          role: "{{ group_names | first }}"
        headers:
          Authorization: "Bearer {{ monitoring_api_token }}"
      delegate_to: localhost
```

This comprehensive guide covers advanced Ansible concepts essential for senior DevOps engineers, including practical examples and real-world patterns for infrastructure automation, deployment strategies, and operational excellence.
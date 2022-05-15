### jenkins initial script

```
 pipeline {
    agent any

  stages {
    stage('Initial cleanup') {
      steps {
        dir("${WORKSPACE}") {
          deleteDir()
        }
      }
    }
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }

    stage('Test') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }
    stage('Package') {
      steps {
        script {
          sh 'echo "Packaging App" '
        }
      }
    }
    stage('Deploy') {
      steps {
        script {
          sh 'echo "Deploying to Dev" '
        }
      }
    }
    stage('Clean Up') {
      steps {
        cleanWs()
      }
    }
    }
}
```


```
[db:vars]
ansible_user=ec2-user
ansible_python_interpreter=/usr/bin/python

[tooling]
<Tooling-Web-Server-Private-IP-Address>

[todo]
<Todo-Web-Server-Private-IP-Address>
```

```
---
# Variable setup.
- name: Include OS-specific variables.
  include_vars: "{{ ansible_os_family }}.yml"

- name: Define nginx_user.
  set_fact:
    nginx_user: "{{ __nginx_user }}"
  when: nginx_user is not defined

# Setup/install tasks.
- include_tasks: setup-RedHat.yml
  when: ansible_os_family == 'RedHat'

- include_tasks: setup-Ubuntu.yml
  when: ansible_distribution == 'Ubuntu'

- include_tasks: setup-Debian.yml
  when: ansible_os_family == 'Debian'

- include_tasks: setup-FreeBSD.yml
  when: ansible_os_family == 'FreeBSD'

- include_tasks: setup-OpenBSD.yml
  when: ansible_os_family == 'OpenBSD'

- include_tasks: setup-Archlinux.yml
  when: ansible_os_family == 'Archlinux'

# Vhost configuration.
- import_tasks: vhosts.yml

- name: set webservers host name in /etc/hosts
  become: yes
  blockinfile:
    path: /etc/hosts
    block: |
      {{ item.ip }} {{ item.name }}
  loop: 
      - { name: web1, ip: 172.31.10.177 }
      - { name: web2, ip: 172.31.15.126 }

# Nginx setup.
- name: Copy nginx configuration in place.
  become: true
  template:
    src: "{{ nginx_conf_template }}"
    dest: "{{ nginx_conf_file_path }}"
    owner: root
    group: "{{ root_group }}"
    mode: 0644
  notify:
    - reload nginx

- name: Ensure nginx service is running as configured.
  become: true
  service:
    name: nginx
    state: "{{ nginx_service_state }}"
    enabled: "{{ nginx_service_enabled }}"
```

```
---
# Used only for Debian/Ubuntu installation, as the -t option for apt.
nginx_default_release: ""

# Used only for Redhat installation, enables source Nginx repo.
nginx_yum_repo_enabled: true

# Use the official Nginx PPA for Ubuntu, and the version to use if so.
nginx_ppa_use: false
nginx_ppa_version: stable

# The name of the nginx package to install.
nginx_package_name: "nginx"

nginx_service_state: started
nginx_service_enabled: true

nginx_conf_template: "nginx.conf.j2"
nginx_vhost_template: "vhost.j2"

nginx_worker_processes: >-
  "{{ ansible_processor_vcpus | default(ansible_processor_count) }}"
nginx_worker_connections: "1024"
nginx_multi_accept: "off"

nginx_error_log: "/var/log/nginx/error.log warn"
nginx_access_log: "/var/log/nginx/access.log main buffer=16k flush=2m"

nginx_sendfile: "on"
nginx_tcp_nopush: "on"
nginx_tcp_nodelay: "on"

nginx_keepalive_timeout: "65"
nginx_keepalive_requests: "100"

nginx_server_tokens: "on"

nginx_client_max_body_size: "64m"

nginx_server_names_hash_bucket_size: "64"

nginx_proxy_cache_path: ""

nginx_extra_conf_options: ""
Example extra main options, used within the main nginx's context:
  nginx_extra_conf_options: |
    env VARIABLE;
    include /etc/nginx/main.d/*.conf;

nginx_extra_http_options: ""
Example extra http options, printed inside the main server http config:
   nginx_extra_http_options: |
     proxy_buffering    off;
     proxy_set_header   X-Real-IP $remote_addr;
     proxy_set_header   X-Scheme $scheme;
     proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header   Host $http_host;

nginx_remove_default_vhost: false

# Listen on IPv6 (default: true)
nginx_listen_ipv6: true

nginx_vhosts: []
# Example vhost below, showing all available options:
# - listen: "80" # default: "80"
#   server_name: "example.com" # default: N/A
#   root: "/var/www/example.com" # default: N/A
#   index: "index.html index.htm" # default: "index.html index.htm"
#   filename: "example.com.conf" # Can be used to set the vhost filename.
#
#   # Properties that are only added if defined:
#   server_name_redirect: "www.example.com" # default: N/A
#   error_page: ""
#   access_log: ""
#   error_log: ""
#   extra_parameters: "" # Can be used to add extra config blocks (multiline).
#   template: "" # Can be used to override the `nginx_vhost_template` per host.
#   state: "absent" # To remove the vhost configuration.

nginx_upstreams: 
- name: myapp1
  strategy: "ip_hash" # "least_conn", etc.
  keepalive: 16 # optional
  servers: {
      "web1 weight=3",
      "web2 weight=3",
      "proxy_pass http://myapp1"
  }



nginx_log_format: |-
  '$remote_addr - $remote_user [$time_local] "$request" '
  '$status $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"'

enable_nginx_lb: false
load_balancer_is_required: false

# Webservers 
loadbalancer_name: "myapp1"
web1: "172.31.10.177"
web2: "172.31.15.126"
```
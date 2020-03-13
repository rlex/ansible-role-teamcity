

TeamCity
=========

Hardfork of [this amazing role](https://github.com/matisku/ansible-teamcity-server), which sadly was outdated. Compared to original this role have:

* Full idempotency
* Fixes to allow installation of recent teamcity versions
* Versioning of installations (older version are still kept and you can switch back to them with one symlink change, provided you hasn't performed DB migrations)
* Agent management in same role
* Some other stuff i made along the way

This role will install and configure teamcity server - CI tool from JetBrains.  
It will create fully working, provisioned teamcity server with local (hsqldb) or mysql database.

It can also install teamcity agents, however you will still need to approve them from server web ui.

You will need to specify teamcity_server: true and/or teamcity_agent: true in host/group vars or in playbook to install corresponding service

## Compatibility
This role is compatible with any modern systemd-based distro.  
Requires installation of openjdk on target system.

## Role Variables
| Variable name                           | Default value                                                      | Description                      |
|-----------------------------------------|--------------------------------------------------------------------|----------------------------------|
| teamcity_system_user                    | `teamcity`                                                         | system user for teamcity         |
| teamcity_system_group                   | `teamcity`                                                         | system group for teamcity        |
| teamcity_system_home                    | `/var/lib/teamcity`                                                | home directory for system user   |
| teamcity_server                         | `false`                                                            | Variable for server install      |
| teamcity_server_version                 | `2019.2.2`                                                         | TeamCity version to install      |
| teamcity_server_sha256                  | `108e08b1f9f59d533bb1f7e3a498900bb746bb31051c968ba425887b4c5503e9` | sha256 for TeamCity package      |
| teamcity_admin_user                     | `teamcity`                                                         | provisioned admin user           |
| teamcity_admin_password                 | `teamcity`                                                         | provisioned admin password       |
| teamcity_server_install_dir             | `/opt/teamcity-server`                                             | TeamCity unpack dir              |
| teamcity_server_data_dir                | `{{ teamcity_system_home }}/teamcity-server`                       | TeamCity DATA_DIR                |
| teamcity_server_license_keys            | `[]`                                                               | List of TeamCity licenses        |
| teamcity_server_plugins                 | `[]`                                                               | List of TeamCity plugins         |
| teamcity_server_bind_port               | `8111`                                                             | TeamCity server bind port        |
| teamcity_server_memory                  | `2048`                                                             | TeamCity server memory in MB     |
| teamcity_server_mysql_connector_version | `8.0.19`                                                           | MySQL connector version          |
| teamcity_server_mysql_connector_dir     | `/opt/mysql-connector`                                             | MySQL connector install dir      |
| teamcity_server_mysql_db_user           | `teamcity`                                                         | MySQL username                   |
| teamcity_server_mysql_db_password       | `teamcity`                                                         | MySQL password                   |
| teamcity_server_mysql_db_host           | `localhost`                                                        | MySQL database host              |
| teamcity_server_mysql_db_port           | `3306`                                                             | MySQL database port              |
| teamcity_server_mysql_db_name           | `teamcity`                                                         | MySQL database                   |
| teamcity_server_db_type                 | `local`                                                            | Database: local / mysql          |
| teamcity_agent                          | `false`                                                            | Variable for agent install       |
| teamcity_agent_dir                      | `/opt/teamcity-agent`                                              | Directory for agent install      |
| teamcity_agent_name                     | `{{ ansible_hostname }}`                                           | TeamCity agent internal name     |
| teamcity_server_url                     | `http://localhost:8111`                                            | TeamCity URL for agent           |

There is also some internal knobs in vars/main.yml which you probably shouldn't change unless you really have the need.

## Nginx reverse proxy
There is no nginx management in this role. If you want it, you can use [this nginx role](https://github.com/jdauphant/ansible-role-nginx) with following config as base:
```
nginx_user: www-data
nginx_group: www-data
nginx_keep_only_specified: true
nginx_events_params:
  - worker_connections 512
  - use epoll
  - multi_accept on
nginx_configs:
  websockets:
    - map $http_upgrade $connection_upgrade {
        default upgrade;
        '' '';
      }
nginx_sites:
  tc.example.com:
    - listen {{ ansible_default_ipv4["address"] }}:80
    - listen {{ ansible_default_ipv4["address"] }}:443 ssl http2
    - server_name tc.example.com
    - ssl_certificate /etc/nginx/ssl/example.com/example.com.crt
    - ssl_certificate_key /etc/nginx/ssl/example.com/example.com.key
    - ssl_dhparam /etc/nginx/ssl/dhparam.pem
    - access_log /var/log/nginx/tc_example_com_access.log
    - error_log /var/log/nginx/tc_example_com_error.log
    - client_header_timeout 10m
    - client_body_timeout 10m
    - client_max_body_size 0
    - send_timeout 10m
    - if ($scheme = http) { return 301 https://$server_name$request_uri; }
    - location / {
        proxy_pass http://127.0.0.1:8111;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 900;
        proxy_max_temp_file_size 10240m;
      }

Ansible-Prometheus-Exporters
=========

This role installs Prometheus exporters on the nodes specified in your Ansible inventory file

## Components

* `Node Exporter`: basic server metrics (exporter installation and optional Consul SD config)

* `Blackbox Exporter`: HTTP and ICMP checks (Consul SD config only)

* `MySQL Exporter`: MySQL performance (exporter installation and optional Consul SD config)

* `PostgreSQL Exporter`: PostgreSQL performance (exporter installation and optional Consul SD config)

* `Clickhouse Exporter`: Clickhouse performance (exporter installation and optional Consul SD config)

* `Redis Exporter`: Redis performance (exporter installation and optional Consul SD config)

## Intro

> {{ playbook_dir }} is a default Ansible variable. This is where you keep your playbooks, variables, config files and templates

> Ansible roles you install with Ansible-Galaxy can be stored elsewhere. I used this variable to give you a better understanding of where all these files should be stored

## Config example (variables that are commented out are optional)

* `FILE: {{ playbook_dir }}/vars/prometheus-exporters.yml`

In this file, you can set global variables that will be used on all hosts where you install your Prometheus exporters. See the example below:

> Consul Service Discovery Variables:

```
### consul_url is required by exporters that will be added to Consul Service Discovery. Set this variable if you use Consul SD
consul_url: https://consul.example.com
```

```
### If you do not use Consul SD, you may omit the 'consul_url' variable but make sure you set the following ones to 'true':

# node_exporter_do_not_add_service_to_consul: true
# blackbox_do_not_add_server_to_consul: true
# mysql_exporter_do_not_add_service_to_consul: true
```

> Node Exporter Variables:

```
### This variable is optional in this file. You can add it, if you want to have the Node Exporter installed on all hosts by default. Alternatively, you can add it for each host in the 'host_vars' directory
install_node_exporter: true

### Specify a download URL to get the preferred version of the Node Exporter. You can set this variable in this file in order to keep the version of the Node Exporter in one place 
node_exporter_tar_url: "https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz"

### You can override the default Consul JSON template for the Node Exporter. For example, if you need to add custom tags

# consul_node_exporter_template: "{{ playbook_dir }}/consul_templates/node_exporter.json.j2"
```

> Blackbox Exporter Variables (ICMP Ping Check):

```
### This variable is optional in this file. You can add it, if you want to enable ICMP Ping checks on all hosts by default. Alternatively, you can add it for each host in the 'host_vars' directory
blackbox_exporter_icmp: true

### You can override the default Consul JSON template for the Blackbox Exporter. For example, if you need to add custom tags

# consul_blackbox_exporter_template_server: "{{ playbook_dir }}/consul_templates/blackbox_exporter_icmp.json.j2"
```
* `FILE: {{ playbook_dir }}/host_vars/some-server-1; {{ playbook_dir }}/host_vars/some-server-2, etc.`
`some-server-1`, `some-server-2`, etc. correspond the `inventory_hostname`s you have in your Ansible inventory file; in these files, you can add variables for specific hosts. See the examples below:

> Basic setup (Node Exporter + ICMP Ping)

`host_vars/elasticsearch-data-prod-1`
```
---
install_node_exporter: true
blackbox_exporter_icmp: true
blackbox_exporter_http: false    # this one is optional, it's disabled by default
```

This config tells Ansible to install the Node Exporter and add the server to Consul for further ICMP checks performed by the Blackbox Exporter

> Node Exporter Textfile Collector

`host_vars/elasticsearch-data-prod-1`

```
### You can add cron jobs for the Textfile Collector in the following way:

textfile_collector_cron_jobs: [
  {
    cron_name: "Calculate the number of stopped containers"
    cron_command: "docker ps -f 'status=exited' | grep -Ei 'Exited.*([4-9]|[1-2][0-9]).*hours ago|Exited.*days ago' | wc -l | sed 's/^/node_ecs_docker_stopped_containers_count{} /' > /var/lib/node_exporter/textfile_collector/node_ecs_docker_stopped_containers_count.prom"
    cron_minute: "*/5"
    cron_hour: "*"
    day_of_month: "*"
    month: "*"
    day_of_week: "*"
  },
  {
    cron_name: "Another cron"
    cron_command: "bash /usr/local/scripts/node_exporter_textfile_collector.sh"
    cron_minute: "*/30"
    cron_hour: "*"
    day_of_month: "*"
    month: "*"
    day_of_week: "*"
  }
]
```
The Textfile Collector will look for data in `{{ node_textfile_collector_directory }}` which is set to `/var/lib/node_exporter/textfile_collector` by default.

> MySQL Exporter (Host)

If you would like to install the MySQL Exporter on the host, without Docker (you will want to do so if your MySQL server is not running in Docker, and it's not an RDS or a CloudSQL instance, etc.), then you just need to set several variables:

`host_vars/mysql-prod-server`
```
install_mysql_exporter: true
mysql_exporter_installation_type: "host"
```

And don't forget to add credentials to your file with secrets, say `vars/prometheus-secrets-production.yml`

```
mysql_exporter_user: mysql_exporter
mysql_exporter_password: qwerty12345
```

The user would require the following permissions

```
mysql > CREATE USER 'mysql_exporter'@'%' IDENTIFIED BY 'qwerty12345';
mysql > GRANT PROCESS,REPLICATION CLIENT,SELECT,SHOW DATABASES ON *.* TO 'mysql_exporter'@'%';
mysql > FLUSH PRIVILEGES;
mysql > UPDATE mysql.user SET max_user_connections=10 WHERE user='mysql_exporter';
mysql > FLUSH PRIVILEGES;
```

> MySQL Exporter (Docker)

Works  good for AWS RDS, GCP CloudSQL, etc. when you do not have root access to the MySQL server, thus you are unable to install an exporter there, so you need to install it elsewhere. Also, the Docker option allows you to have several MySQL Exporter containers running on the same host, for example, you have Develop, Staging and Production MySQL instances running on AWS RDS or GCP CloudSQL and you would like to keep all MySQL exporters on the Prometheus server. See the example below:
`host_vars/prometheus-main`
```
---

install_node_exporter: true
blackbox_exporter_icmp: true
blackbox_exporter_http: false
install_mysql_exporter: true
mysql_exporter_installation_type: "docker"

#################################
###     CLOUDSQL EXPORTER     ###
#################################

# If you don't need CloudSQL Proxy, skip this step
mysql_exporter_docker_networks: [
  {
    name: "mysql-production"
  },
  { 
    name: "mysql-staging"
  },
  { 
    name: "mysql-test"
  }
]

# If you server does not have direct access to CloudSQL via an internal IP,
# you can deploy CloudSQL proxy in the following way:

cloudsql_proxy_docker_opts:
- container_name: "cloudsql-proxy-production"
  image_name: "gcr.io/cloudsql-docker/gce-proxy"
  image_tag: "1.16"
  command: "/cloud_sql_proxy -instances={{ gcp_project_name }}:us-east4-a:bo-db-2=tcp:0.0.0.0:3306"
  purge_networks: true
  networks: [ { name: "mysql-production" } ]
- container_name: "cloudsql-proxy-staging"
  image_name: "gcr.io/cloudsql-docker/gce-proxy"
  image_tag: "1.16"
  command: "/cloud_sql_proxy -instances={{ gcp_project_name }}:us-east4-a:bo-db-staging=tcp:0.0.0.0:3306"
  purge_networks: true
  networks: [ { name: "mysql-staging" } ]
- container_name: "cloudsql-proxy-test"
  image_name: "gcr.io/cloudsql-docker/gce-proxy"
  image_tag: "1.16"
  command: "/cloud_sql_proxy -instances={{ gcp_project_name }}:us-east4-a:bo-db-staging=tcp:0.0.0.0:3306"
  purge_networks: true
  networks: [ { name: "mysql-test" } ]

mysql_exporter_docker_opts:
- container_name: "mysql-exporter-production"
  mysql_host: "back-office-production"
  image_name: "prom/mysqld-exporter"
  image_tag: "v0.12.1"
  scrape_port: "9104"
  ports: [ "0.0.0.0:9104:9104" ]
  env_vars:
    DATA_SOURCE_NAME: "{{ mysql_exporter_user }}:{{ mysql_exporter_password }}@(cloudsql-proxy-production:3306)/"
  purge_networks: true
  networks: [ { name: "mysql-production" } ]
- container_name: "mysql-exporter-staging"
  mysql_host: "back-office-staging"
  image_name: "prom/mysqld-exporter"
  image_tag: "v0.12.1"
  scrape_port: "10104"
  ports: [ "0.0.0.0:10104:9104" ]
  env_vars:
    DATA_SOURCE_NAME: "{{ mysql_exporter_user }}:{{ mysql_exporter_password }}@(cloudsql-proxy-staging:3306)/"
  purge_networks: true
  networks: [ { name: "mysql-staging" } ]
- container_name: "mysql-exporter-test"
  mysql_host: "back-office-test"
  image_name: "prom/mysqld-exporter"
  image_tag: "v0.12.1"
  scrape_port: "11104"
  ports: [ "0.0.0.0:11104:9104" ]
  env_vars:
    DATA_SOURCE_NAME: "{{ mysql_exporter_user }}:{{ mysql_exporter_password }}@(cloudsql-proxy-staging:3306)/"
  purge_networks: true
  networks: [ { name: "mysql-test" } ]
```

* `FILE: {{ playbook_dir }}/vars/prometheus-blackbox-domains.yml`

> Blackbox Exporter Variables (HTTP Check):

```
### Enable HTTP checks (the rest of the exporters are not needed thus will not be installed)
blackbox_exporter_http: true

### Specify sites to monitor
blackbox_sites:
  main_site:
    domain: example.com
    url: https://example.com/
    module: http_2xx_check
    address: example.com    # if we use a 3rd party proxy (such as CloudFlare) with a dynamic IP, then we can just speficy our domain name
    parent: gke-production-cluster
  main_api:
    domain: api.example.com
    url: https://api.example.com/health
    module: http_2xx_check
    address: api-example-com-svc-alb-1234567890.eu-west-1.elb.amazonaws.com    # this ALB has several IPs so let's use its DNS name
    parent: aws-ecs-production
  dev_api:
    domain: api-dev.example.com
    url: https://api-dev.example.com/health
    module: http_2xx_check
    address: 1.2.3.4
    parent: dev-server

### Specify sites to delete from monitoring
blackbox_sites_delete:
- old.example.com
- api-old.example.com
- demo.example.com
```

* `FILE: {{ playbook_dir }}/prometheus-exporters-secrets.yml`

```
### In case you have Basic Auth on Consul
consul_basic_auth_user:
consul_basic_auth_password:

mysql_exporter_user: mysql_exporter
mysql_exporter_password: qwerty12345
```

* `FILE: {{ playbook_dir }}/requirements.yml`

```
# Install Python modules if you deploy Prometheus exporters in Docker
- src: https://github.com/arsenii-stefanov/ansible-python.git
  path: roles
  scm: git
# Set up Prometheus Exporters
- src: https://github.com/arsenii-stefanov/ansible-prometheus-exporters.git
  path: roles
  scm: git
```


#### You can override any variable defined in the 'defaults/main.yml'

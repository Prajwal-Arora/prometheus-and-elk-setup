# "Prometheus + Grafana" and "ELK stack" quickstart guide!
## Part 1

>In this part we will setup Prometheus, NodeExporter & Grafana.

### 1.1 Prometheus and NodeExporter Setup
##### 1.1.1 Overview:
- Install Ansible
- Install pip and tox
- Git clone Ansible-prometheus
- Setup role
- Create inventory file
- Create main.yml playbook
- Run playbook
- Install node exporter and configure
- Check web interface
##### 1.1.2 Commands:
- `$ sudo apt install ansible python-pip`
- `$ pip install tox`
- `$ git clone https://github.com/cloudalchemy/ansible-prometheus`
- `$ cd ansible-prometheus/`
- `$ mkdir -p roles/cloudalchemy.prometheus`
- `$ mv defaults/ handlers/ meta/ molecule/ tasks/ templates/ vars/ roles/cloudalchemy.prometheus/`
- `$ vi main.yml`

Copy the following code to **main.yml**
```
---
- hosts: all
  roles:
  - cloudalchemy.prometheus
  vars:
    prometheus_targets:
      node:
      - targets:
        - localhost:9100
        labels:
          env: demosite
```
- `$ vi inventory`

Copy the following code to **inventory**
```
localhost ansible_connection=local
```
- `$ ansible-playbook -i inventory main.yml`
- `$ cd ..`
- `$ curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz`
- `$ tar xzf node_exporter-1.0.1.linux-amd64.tar.gz`
- `$ cd node_exporter-1.0.1.linux-amd64/`
- `$ ./node_exporter &`

> **&** used in the above command allows the script to run in the background, so you can press `ctrl c` and continue with your work

Once the Node Exporter is installed and running, you can verify that metrics are being exported by cURLing the /metrics endpoint

- `$ curl http://localhost:9100/metrics`

You should see something like this -
```
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 3.8996e-05
go_gc_duration_seconds{quantile="0.25"} 4.5926e-05
go_gc_duration_seconds{quantile="0.5"} 5.846e-05
# etc.
```
- `$ sudo vi /etc/prometheus/prometheus.yml`

Your file should look like this -
```
#
# Ansible managed
#
# http://prometheus.io/docs/operating/configuration/

global:
  evaluation_interval: 15s
  scrape_interval: 15s
  scrape_timeout: 10s

  external_labels:
    environment: ip-172-31-14-250.ap-south-1.compute.internal




rule_files:
  - /etc/prometheus/rules/*.rules


scrape_configs:
  - job_name: prometheus
    metrics_path: /metrics
    static_configs:
    - targets:
      - ip-172-31-14-250.ap-south-1.compute.internal:9090
  - file_sd_configs:
    - files:
      - /etc/prometheus/file_sd/node.yml
    job_name: node
  - job_name: 'node_exporter_metrics'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']
```

> You just need to add last 4 lines of the above code to your file **prometheus.yml**

- `$ sudo systemctl restart prometheus`
- `$ ^restart^status`

You should see something like this -
```
● prometheus.service - Prometheus
   Loaded: loaded (/etc/systemd/system/prometheus.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2020-07-01 09:20:46 UTC; 17s ago
 Main PID: 2268 (prometheus)
    Tasks: 8 (limit: 1121)
   CGroup: /system.slice/prometheus.service
           └─2268 /usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus --storage.tsdb.retention.time=30d --storage.tsdb.retention.size=0 --web.console.libraries=/etc/prometheus/con
```
Now that we have prometheus and node_exporter setup, we should be able to view them in the browser
- Prometheus -> http://<public_ip>:9090/graph
- NodeExporter -> http://<public_ip>:9100/metrics

>If you have been running commands locally you should use `localhost` instead of `<public_ip>` in your browser

### 1.2 Grafana Setup
##### 1.2.1 Commands:
- `$ curl -LO https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_5.1.4_amd64.deb`
- `$ sudo apt-get install -y adduser libfontconfig`
- `$ sudo dpkg -i grafana_5.1.4_amd64.deb`
- `$ sudo systemctl start grafana-server`
- `$ sudo systemctl enable grafana-server`

Grafana should be up and running at port 3000, view at -> `http://<public_ip>:3000/login`

> Credentials for Grafana -> username & password = admin

### 1.3 Screenshots

> To sync Grafana with Prometheus refer the screenshots below!



## Part 2

>In this part we will setup the ELK stack -> ElasticSearch, Logstash & Kibana

### 2.1 ElasticSearch Setup
##### 2.1.1 Overview:
- Clone this Github repo: https://github.com/elastic/ansible-elasticsearch
- Create inventory file
- Create main.yaml playbook
- Run the playbook
##### 2.1.2 Commands:
- `$ git clone https://github.com/elastic/ansible-elasticsearch`
- `$ cd ansible-elasticsearch`
- `$ vi main.yml`

Copy the following code to **main.yml**
```
- name: Simple Example
  hosts: localhost
  roles:
    - role: elastic.elasticsearch
  vars:
    es_version: 7.8.0
```
- `$ mkdir -p roles/elastic.elasticsearch`
- `$ mv defaults/ docs/ files/ filter_plugins/ handlers/ helpers/ meta/ tasks/ templates/ test/ vars/ roles/elastic.elasticsearch/`
- `$ vi inventory`

Copy the following code to **inventory**
```
localhost ansible_connection=local
```
- `$ ansible-playbook -i inventory main.yml`

### 2.2 Kibana Setup
##### 2.2.1 Commands:
- `$ sudo apt-get install apache2-utils`
- `$ sudo apt install kibana`
- `$ sudo apt install nginx`
- `$ sudo vi /etc/nginx/sites-enabled/default`

Copy the following code -
```
server {
        listen 80;

        server_name http://13.127.74.93;

        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/htpasswd.users;

        location / {
            proxy_pass http://localhost:5601;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
}
```
- ```$ echo "kibanaadmin: `openssl passwd -apr1`" | sudo tee -a /etc/nginx/htpasswd.users```

> You should get a password prompt after executing the command above
set a password of your choice

- `$ vi /etc/kibana/kibana.yml`

You need to look for ElasticSearch username and password, something like below!
```
# If your Elasticsearch is protected with basic authentication, these settings provide
# the username and password that the Kibana server uses to perform maintenance on the Kibana
# index at startup. Your Kibana users still need to authenticate with Elasticsearch, which
# is proxied through the Kibana server.
#elasticsearch.username: "kibana_system"
#elasticsearch.password: "pass"
```
- `$ sudo systemctl start elasticsearch`
- `$ sudo systemctl start kibana`

Kibana should be up and running, view at -> `http://<public_ip>`

> Credentials for Kibana ->
username - **kibanaadmin**
password - <password_set_during_prompt>

### 2.3 Logstash Setup
##### 2.3.1 Commands:
- `$ curl -LO https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.8.0-amd64.deb`
- `$ sudo dpkg -i filebeat-7.8.0-amd64.deb`
- `$ sudo vi /etc/filebeat/filebeat.yml`

Append the following code -
```
output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "kibana_system"
  password: "pass"
```

> You need to put your credentials that you find in **kibana.yml**,
Sample data is available in the dashboard do add it to see how things look

### 2.4 Screenshots



### Pro Tips on Using the ELK Stack
- The ELK stack requires decent amount of memory, the standard config recommends 2GB (i.e. T2.medium).
- It may also be necessary to stop anything else that you have set up (i.e. Jenkins, Prometheus, and Grafana).
- If the Ansible installation script for Elasticsearch takes too long (more than 5 minutes for a single item) you may have run out of memory.
- Do not be alarmed, just restart the EC2 instance.

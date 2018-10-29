# Services autoscaling for Docker Swarm cluster
Scale your web-services depending on current capacity usage.

## General description
The main idea of this document is to provide a general description of presented approach to Docker Swarm autoscaling. As it is widely known, Docker Swarm does not support any form of autoscaling, nor horizontal nor vertical. However, it is important to be capable of serving high capacity usage peaks. Let's assume that each Swarm cluster must have a reverse proxy to limit access to the services and some monitoing system. An example autoscaling would be based on a Docker Swarm cluster, Nginx reverse proxy and Prometheus monitoring. Each scaling activity would be trigerred by a previously defined Prometheus alert. Scaling would happen by Ansible Playbooks.

## Prerequisities and used tools
There are two basic services the autoscaling will be based on: Docker Swarm cluster and Prometheus container. We will extend the abilities of Prometheus and Docker Swarm by the tools listed below:

### Basic
* Dcoker Swarm cluster
* Nginx Reverse proxy
* Prometheus monitoring

### Must have
* [cAdvisor](https://github.com/google/cadvisor) - container monitoing
* [Prometheus metric library for Nginx](https://github.com/knyar/nginx-lua-prometheus) - reverse proxy monitoing
* [Prometheus-AM-Executor](https://github.com/imgix/prometheus-am-executor) - for triggering scaling jobs
* [Ansible](https://github.com/ansible/ansible) - we all know

### Optional
* [Alertmanager](https://github.com/prometheus/alertmanager) - possible alert routing for Slack or similar

## Configure Nginx Reverse Proxy to collect statistics for Prometheus
Add code shown below to your `nginx.conf` in `http` section.
```
lua_shared_dict prometheus_metrics 10M;
lua_package_path "/path/to/nginx-lua-prometheus/?.lua";
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
  metric_requests = prometheus:counter(
    "nginx_http_requests_total", "Number of HTTP requests", {"host", "status"})
  metric_latency = prometheus:histogram(
    "nginx_http_request_duration_seconds", "HTTP request latency", {"host"})
  metric_connections = prometheus:gauge(
    "nginx_http_connections", "Number of HTTP connections", {"state"})
';
log_by_lua '
  metric_requests:inc(1, {ngx.var.server_name, ngx.var.status})
  metric_latency:observe(tonumber(ngx.var.request_time), {ngx.var.server_name})
';
```
Then, create a server to enable Prometheus statistics scraping.
```
server {
  listen 9145;
  allow 192.168.0.0/16;
  deny all;
  location /metrics {
    content_by_lua '
      metric_connections:set(ngx.var.connections_reading, {"reading"})
      metric_connections:set(ngx.var.connections_waiting, {"waiting"})
      metric_connections:set(ngx.var.connections_writing, {"writing"})
      prometheus:collect()
    ';
  }
}
```
Restart Nginx service.

## Configuring Prometheus to collect data from cAdvisor and Nginx
Below you can find a very basic configuration of the Prometheus Server. It will collect all data provided by cAdvisor and Nginx.
```
global:
  scrape_interval:     30s # By default, scrape targets every 15 seconds.
  evaluation_interval: 30s # By default, scrape targets every 15 seconds.

# Load and evaluate rules in this file every 'evaluation_interval' seconds.
rule_files:
  - 'PATH_TO_YOUR_ALERTING_RULES_FILE'

# alert
alerting:
  alertmanagers:
    - scheme: http
      static_configs:
        - targets: ['HOSTNAME_OF_PROMETHEUS_AM_EXEXUTOR:9092']

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 15s
    static_configs:
      - targets: ['HOSTNAME_OF_PROMETHEUS:9090']
  - job_name: 'nginx'
    scrape_interval: 10s
    static_configs:
      - targets: ['HOSTNAME_OF_NGINX:9145']
  - job_name: 'cadvisor'
    scrape_interval: 30s
    static_configs:
      - targets: ['HOSTNAME_OF_CADVISOR:4194']
```

## Create custom alerts for measuring capacity usage
Prometheus needs a definition of an alert that later would trigger Prometheus-AM-Executor, to scale our services immediately. In our case we'll be measuring the overall service usage by summing all requests time and dividing it by all the workers and containers counted. A sample query scaled to percents is shown below:

```
100 * (sum by (YOUR_LABEL) (delta(nginx_http_request_duration_seconds_sum{job="nginx",method!="OPTIONS",service=~"YOUR_SERVICE_NAME"}[1m])) / (8 * scalar(count(container_last_seen{container_label_com_docker_swarm_service_name=~"YOUR_SERVICE_NAME"})) * 60))
```

However, you'll need to write your own calculating query. According to the Prometheus configuration shown in previous section, the alert would be routed straight to the Prometheus-AM-Executor service.

## Build and configure Prometheus-AM-Executor container
First of all we need to create the main configuration file for the service. As the am-executor is based on Alertmanager, the configuration would be very similar. The only difference is that we route all alerts to the am-executor itself. Let's assume we'll use Prometheus-AM-Executor on 9092 port.

```
global:
  resolve_timeout: 1m
route:
  group_by: ['severity','alertname','instance','name','backend','table','service']
  receiver: 'default'
  routes:

receivers:
- name: 'default'
  webhook_configs:
  - url: http://localhost:9092
```
Now, we need to create a script that will dump all alert variables to a yaml file that can be understood by Ansible. The script will be trigerred every time Prometheus-AM-Export receives an alert from Prometheus so it can also run the ansible-playbook. Here it is:

* Bash script `scale-up.sh`

```
#!/bin/bash
#Dump all necessary variables from incoming alert
echo "status: \"$AMX_STATUS\"" > /tmp/alert_data
echo "service: \"$AMX_ALERT_1_LABEL_service\"" >> /tmp/alert_data
echo "severity: \"$AMX_ALERT_1_LABEL_severity\"" >> /tmp/alert_data
ansible-playbook -e 'host_key_checking=False' -i /home/ansible/inventory /home/ansible/scaling-playbook.yml
```
* Ansible playbook `scaling-playbook.yml`

```yaml
---
- name: "Docker service scaling playobok"
  hosts: ONE_OF_SWARM_MASTERS
  gather_facts: true
  become: true
  tasks:
  - name: "Include alert variables"
    include_vars:
      file: /tmp/alert_data
      name: alert

  - name: "Get current {{ alert.service }} instance #"
    shell: "docker service inspect --format {% raw %}'{{ .Spec.Mode.Replicated.Replicas }}'{% endraw %} {{ alert.service }}"
    register: service_count
    when: alert.status == 'firing'

  - name: "Upscale {{ alert.service }} + 1 "
    shell: "docker service scale {{ alert.service }}={{ service_count.stdout|int + 1 }}"
    when: alert.severity == 'critical' and alert.status == 'firing' and service_count.stdout|int > 2 and service_count.stdout|int < 8

  - name: "Downscale {{ alert.service }} + 1 "
    shell: "docker service scale {{ alert.service }}={{ service_count.stdout|int - 1 }}"
    when: alert.severity == 'warning' and alert.status == 'firing' and service_count.stdout|int > 2 and service_count.stdout|int < 8
```

Next thing to do is to build a docker container that could be placed on the Swarm cluster.
You can use a Dockerfile shown below. It will build the go application and place it in a container with ansible, ssh and all necessary dependencies. As we'll be scaling the services using Ansible Playbooks, we need to add an user with proper private key to allow ssh access. For obvious reasons we're not likely to save a private ssh key to any file. However, it's possible to build docker container with `build-args`. While executing `docker build` command please add `--build-args SSH_PRIVATE_KEY=YOUR_PRIVATE_KEY_CONTENT` arugument.
```Dockerfile
#Building application
FROM golang:alpine as builder
RUN apk --no-cache add git
RUN go get -u github.com/imgix/prometheus-am-executor/...
WORKDIR /go/src/github.com/imgix/prometheus-am-executor
COPY alertmanager.conf /go/src/github.com/imgix/prometheus-am-executor/examples/alertmanager.conf
RUN go get -u github.com/prometheus/alertmanager/... \
    && go get -u github.com/prometheus/client_golang/... \
    && CGO_ENABLED=0 GOARCH=amd64 GOOS=linux \
    go build -a -installsuffix cgo -ldflags '-s -w -extld ld -extldflags -static'

#Building container
FROM alpine
WORKDIR /
COPY --from=builder /go/src/github.com/imgix/prometheus-am-executor/prometheus-am-executor .
RUN adduser -D -h /home/ansible ansible
USER ansible
ARG SSH_PRIVATE_KEY
RUN mkdir /home/ansible/.ssh/
RUN echo "${SSH_PRIVATE_KEY}" > /home/ansible/.ssh/id_rsa
RUN chmod 700 /home/ansible/.ssh
RUN chmod 600 /home/ansible/.ssh/id_rsa
COPY inventory /home/ansible/inventory
COPY scaling-playbook.yml /home/ansible/scaling-playbook.yml
USER root
COPY scale_up.sh /bin/scale_up
RUN chmod -v 755 /bin/scale_up
RUN apk --no-cache add ansible bash openssh-client
USER ansible
ENTRYPOINT ["/prometheus-am-executor"]
```
## Setup the environment
To setup the environment you need to ensure that following components are running:
* Prometheus-AM-Executor
* Prometheus
* cAdvisor
* Nginx Reverse Proxy

To execute a sample alert you can curl the Prometheus-AM-Executor with a command:

```curl -H "Content-Type: application/json" -d '{"receiver":"default","status":"firing","alerts":[{"status":"firing","labels":{"severity":"critical","alertname":"service_capacity_backend","instance":"localhost:5678","job":"broken","monitor":"codelab-monitor","service":"synonyms"},"annotations":{},"startsAt":"2016-04-07T18:08:52.804+02:00","endsAt":"0001-01-01T00:00:00Z","generatorURL":""}],"groupLabels":{"alertname":"InstanceDown"},"commonLabels":{"alertname":"InstanceDown","job":"broken","monitor":"codelab-monitor"},"commonAnnotations":{},"externalURL":"http://oldpad:9093","version":"3","groupKey":9777663806026784477}' PROMETHEUS_AM_EXECUTOR:9092/api/v1/alerts```

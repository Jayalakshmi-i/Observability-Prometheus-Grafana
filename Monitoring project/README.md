# Ultimate DevOps Monitoring and Alerting Project: Ensuring Uptime and Reliability 

YouTube Video URL of the project ->  [https://www.youtube.com/watch?v=JGQI5pkK82w](https://www.youtube.com/watch?v=0Ec_VeO60m8)

<img width="900" alt="Screenshot 2024-09-29 at 9 38 09 PM" src="https://github.com/user-attachments/assets/aea0bede-3372-48b7-bc6b-17799f1e8277">

Here are the step-by-step details to set up an Ultimate DevOps Monitoring project using Prometheus and Alert Manager:

### Components:
   - **Prometheus:** It is an open-source tool for monitoring and alerting applications. It uses the concept of scrapping when target systems metric points are contacted to fetch data at regular intervals.
   - **Node exporter:** It is a monitoring agent that is installed on all target machines so that Prometheus can fetch the data from all the metrics endpoints.
   - **Blackbox exporter:** It is used to get information from the website like traffic is coming from the website or not
   - **Alert manager:** The Alert manager handles alerts sent by client applications such as the Prometheus server. Used to set alert based on conditions so we will be notified ex if the website is down for continuous 1- 5 minutes, service unavailability


#### Pre-requisites to start:
Created a security group with the following ports open: 
   - 22 for SSH
   - 80 for HTTP
   - 443 for HTTPS
   - 25 for SMTP
   - 465 for SMTPS
   - 587 for SMTP
   - 9090 for Prometheus
   - 9093 for Alert manager
   - 9115 for Blackbox Exporter
   - 9100 for Node Exporter
     
### Steps for Downloading, Extracting, and Starting Prometheus, Node Exporter, Blackbox Exporter, and Alertmanager

## VM-1 (Node Exporter)
```bash
sudo apt update
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.8.1.linux-amd64.tar.gz
cd node_exporter-1.8.1.linux-amd64
./node_exporter &
```

## In Virtual machine-2 install Prometheus, Alert Manager, Blackbox Exporter

##### Prometheus
   ```bash
   sudo apt update
   wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
   tar xvfz prometheus-2.52.0.linux-amd64.tar.gz
   cd prometheus-2.52.0.linux-amd64
   ./prometheus --config.file=prometheus.yml &
   ```

##### Alertmanager

   ```bash
   wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
   tar xvfz alertmanager-0.27.0.linux-amd64.tar.gz
   cd alertmanager-0.27.0.linux-amd64
   ./alertmanager --config.file=alertmanager.yml &
   ```

##### Blackbox Exporter
   ```bash
   wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
   tar xvfz blackbox_exporter-0.25.0.linux-amd64.tar.gz
   cd blackbox_exporter-0.25.0.linux-amd64
   ./blackbox_exporter &
   ```
---
Once the VM-1 node exporter is up and running we can see the below webpage

![image](https://github.com/user-attachments/assets/31e9ab3e-f6f5-4875-8914-a4ef66d2b431)
 

Now let’s run a simple game application to monitor

To run the boardgame application on the website page we need to have Java and Maven to build, so we will install them using the below commands

```
cd Boardgame
sudo apt install openjdk-11-jre-headless
sudo apt install maven -y
mvn package 					// to build the project
```
We can execute the jar file to run the application on the browser

```
cd target
ls 		// can see .jar file
java -jar database_service_project-0.0.4.jar
```
![image](https://github.com/user-attachments/assets/8843549d-378e-4e92-85b6-da889b1ad1ee)

Now, we will access the game application at: http://3.135.20.106:8080/

![image](https://github.com/user-attachments/assets/f29bcf19-de7e-4176-8822-29ec58fbb3c4)


Next, go to VM-2 to configure the Prometheus server by defining alert-rules for the different scenarios. and based on these rules we will get the alerts

```
cd Prometheus
./Prometheus &
```

Can access the Prometheus server at: http://3.145.128.69:9090/graph  

![image](https://github.com/user-attachments/assets/085904e1-0f2e-4721-96a2-b097ec51f639)

For now, we can’t see any alert rules so let’s create a new alert_rules.yaml file to configure alert rules in the Prometheus server

## Alert Rules Configuration (`alert_rules.yml`)

### Alert Rules Group
```yaml
groups:
- name: alert_rules                   # Name of the alert rules group
  rules:
    - alert: InstanceDown
      expr: up == 0                   # Expression to detect instance down
      for: 1m
      labels:
        severity: "critical"
      annotations:
        summary: "Endpoint {{ $labels.instance }} down"
        description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."

    - alert: WebsiteDown
      expr: probe_success == 0        # Expression to detect website down
      for: 1m
      labels:
        severity: critical
      annotations:
        description: The website at {{ $labels.instance }} is down.
        summary: Website down

    - alert: HostOutOfMemory
      expr: node_memory_MemAvailable / node_memory_MemTotal * 100 < 25  # Expression to detect low memory
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Host out of memory (instance {{ $labels.instance }})"
        description: "Node memory is filling up (< 25% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

    - alert: HostOutOfDiskSpace
      expr: (node_filesystem_avail{mountpoint="/"} * 100) / node_filesystem_size{mountpoint="/"} < 50  # Expression to detect low disk space
      for: 1s
      labels:
        severity: warning
      annotations:
        summary: "Host out of disk space (instance {{ $labels.instance }})"
        description: "Disk is almost full (< 50% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

    - alert: HostHighCpuLoad
      expr: (sum by (instance) (irate(node_cpu{job="node_exporter_metrics",mode="idle"}[5m]))) > 80  # Expression to detect high CPU load
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Host high CPU load (instance {{ $labels.instance }})"
        description: "CPU load is > 80%\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

    - alert: ServiceUnavailable
      expr: up{job="node_exporter"} == 0  # Expression to detect service unavailability
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Service Unavailable (instance {{ $labels.instance }})"
        description: "The service {{ $labels.job }} is not available\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

    - alert: HighMemoryUsage
      expr: (node_memory_Active / node_memory_MemTotal) * 100 > 90  # Expression to detect high memory usage
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "High Memory Usage (instance {{ $labels.instance }})"
        description: "Memory usage is > 90%\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

    - alert: FileSystemFull
      expr: (node_filesystem_avail / node_filesystem_size) * 100 < 10  # Expression to detect file system almost full
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "File System Almost Full (instance {{ $labels.instance }})"
        description: "File system has < 10% free space\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"
```

Now we need to input the above rules file to the Prometheus server by updating the prometheus.yml file

## Prometheus Configuration (`prometheus.yml`)

### Global Configuration
```yaml
global:
  scrape_interval: 15s                # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s            # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
```

### Rule Files
```yaml
rule_files:
   - "alert_rules.yml"                # Path to alert rules file
  # - "second_rules.yml"              # Additional rule files can be added here
```
Now to view these alert rules on our Prometheus website page, you need to `restart` the Prometheus server
```
pgrep prometheus 			//to get the process id
kill id
./prometheus &
```
![image](https://github.com/user-attachments/assets/229a1dcb-1c8a-438b-a921-6c35e0a9c3c0)

Now we need to connect both the `Alert manager` and `VM-1 node exporter` to the Prometheus server by updating the **prometheus.yml** file

## Node Exporter
```yaml
  - job_name: "node_exporter"         # Job name for node exporter

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["3.135.20.106:9100"]  # Target node exporter endpoint
```
 ![image](https://github.com/user-attachments/assets/06bc4faf-8353-4df4-8fbd-cd93d3132ee0)

After **restarting** the Prometheus server, we should be able to see the node exporter on the Prometheus target section

![image](https://github.com/user-attachments/assets/c7c9b77c-951c-4aa9-9c7c-91b3009afc12)

Next, need to configure the `Blackbox exporter` to scrape the data from the website application, so let’s update the scrapping configs on the prometheus.yml file

## Blackbox Exporter

#### Prometheus Configuration (`prometheus.yml`)

#### Blackbox Exporter
Adding scrape configs data of game application to the prometheus.yml file

```yaml
  - job_name: 'blackbox'              # Job name for blackbox exporter
    metrics_path: /probe              # Path for blackbox probe
    params:
      module: [http_2xx]              # Module to look for HTTP 200 response
    static_configs:
      - targets:
        - http://prometheus.io        # HTTP target
        - https://prometheus.io       # HTTPS target
        - http://3.135.20.106:8080/  # HTTP target with port 8080
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 3.145.128.69:9115  # Blackbox exporter address
```
`Restart` the Prometheus server to reflect the changes

![image](https://github.com/user-attachments/assets/a40bfce9-459b-4bd9-af22-8cd196fc85c8)

Need to start the Blackbox exporter

![image](https://github.com/user-attachments/assets/ba52c740-50e5-4f8b-b0ab-8ea03e4b3a8b)

When we start **Alert Manager**, we won’t be able to see any alerts as of now since we haven’t configured Alert Manager 

![image](https://github.com/user-attachments/assets/00770d5d-13b0-4fc0-a5fa-11e4b9bb3134)

So, let’s configure it!!

Now we need to configure email notifications to get emails when the defined conditions are met

To receive email notifications, we need to enable 2 step verifications on the Gmail account

To configure the email notifications, go to https://myaccount.google.com/apppasswords and then **Enter name** and get **an app password** which can be used for routing configuration

```
cd alertmanager
vi alertmanager.yml
```
### Alertmanager Configuration (`alertmanager.yml`)

### Routing Configuration
```yaml
route:
  group_by: ['alertname']             # Group by alert name
  group_wait: 30s                     # Wait time before sending the first notification
  group_interval: 5m                  # Interval between notifications
  repeat_interval: 1h                 # Interval to resend notifications
  receiver: 'email-notifications'     # Default receiver

receivers:
- name: 'email-notifications'         # Receiver name
  email_configs:
  - to: jayasample1234@gmail.com       # Email recipient
    from: monitor@example.com            # Email sender
    smarthost: smtp.gmail.com:587     # SMTP server
    auth_username: your_email         # SMTP auth username
    auth_identity: your_email         # SMTP auth identity
    auth_password: "bdmq omqh vvkk zoqx"  # SMTP auth password
    send_resolved: true               # Send notifications for resolved alerts
```

### Inhibition Rules
```yaml
inhibit_rules:
  - source_match:
      severity: 'critical'            # Source alert severity
    target_match:
      severity: 'warning'             # Target alert severity
    equal: ['alertname', 'dev', 'instance']  # Fields to match
```

Now, ``Restart`` the alert manager and check

**Hurray, the monitoring setup is complete!!!!**

![image](https://github.com/user-attachments/assets/527baa2d-8956-4014-a6a8-d6a998448108)

To check the functionality let's try shutting down the game application.

![image](https://github.com/user-attachments/assets/0c73648e-5309-4ea1-9f11-93061131b08b)

![image](https://github.com/user-attachments/assets/8709fc3d-3a79-42f6-bab9-2a62fdeb20c5)

Currently, the status is in a pending state

After `1 minute` the status will change to firing state and soon will receive an email notification

![image](https://github.com/user-attachments/assets/931cd4b7-3311-46a0-9616-658c1964d268)

At this point, we can see the notification on the Alert manager

![image](https://github.com/user-attachments/assets/c82f7b84-ebfd-4d43-947d-18f58098d4bb)

### Notes:
- The `&` at the end of each command ensures the process runs in the background.
- Need to ensure that you have configured the `prometheus.yml` and `alertmanager.yml` configuration files correctly before starting the services.
- Adjust the firewall and security settings to allow the necessary ports (typically 9090 for Prometheus, 9093 for Alertmanager, 9115 for Blackbox Exporter, and 9100 for Node Exporter) to be accessible.

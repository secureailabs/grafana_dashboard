# Display CPU and Memory Usage on Grafana Dashboard

## Install Prometheus

Create a system user: 

    sudo useradd \
        --system \
        --no-create-home \
        --shell /bin/false prometheus

Download Prometheus: 
    
    wget https://github.com/prometheus/prometheus/releases/download/v2.32.1/prometheus-2.32.1.linux-amd64.tar.gz

Extract all Prometheus files from the archive:

    tar -xvf prometheus-2.32.1.linux-amd64.tar.gz

Create a new directory for Prometheus configuration files and move files: 

    sudo mkdir -p /data /etc/prometheus
    cd prometheus-2.32.1.linux-amd64
    sudo mv prometheus promtool /usr/local/bin/
    sudo mv consoles/ console_libraries/ /etc/prometheus/
    sudo mv prometheus.yml /etc/prometheus/prometheus.yml

Set correct ownership: 

    sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

Create a systemd unit configuration file: 

    sudo vim /etc/systemd/system/prometheus.service

    [Unit]
    Description=Prometheus
    Wants=network-online.target
    After=network-online.target
    
    StartLimitIntervalSec=500
    StartLimitBurst=5
    
    [Service]
    User=prometheus
    Group=prometheus
    Type=simple
    Restart=on-failure
    RestartSec=5s
    ExecStart=/usr/local/bin/prometheus \
      --config.file=/etc/prometheus/prometheus.yml \
      --storage.tsdb.path=/data \
      --web.console.templates=/etc/prometheus/consoles \
      --web.console.libraries=/etc/prometheus/console_libraries \
      --web.listen-address=0.0.0.0:9090 \
      --web.enable-lifecycle

    [Install]
    WantedBy=multi-user.target


Start and check the status of Prometheus:

    sudo systemctl enable prometheus
    sudo systemctl start prometheus
    sudo systemctl status prometheus

## Install Node Exporter 

Create a system user for Node Exporter:

    sudo useradd \
        --system \
        --no-create-home \
        --shell /bin/false node_exporter

Download & Extract: 

    wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
    tar -xvf node_exporter-1.3.1.linux-amd64.tar.gz

Move binary to the /usr/local/bin

    sudo mv \
      node_exporter-1.3.1.linux-amd64/node_exporter \
      /usr/local/bin/

Create a systemd unit file: 

    sudo vim /etc/systemd/system/node_exporter.service

    [Unit]
    Description=Node Exporter
    Wants=network-online.target
    After=network-online.target
    
    StartLimitIntervalSec=500
    StartLimitBurst=5
    
    [Service]
    User=node_exporter
    Group=node_exporter
    Type=simple
    Restart=on-failure
    RestartSec=5s
    ExecStart=/usr/local/bin/node_exporter \
        --collector.logind
    
    [Install]
    WantedBy=multi-user.target


Start and check the status of Node Exporter

    sudo systemctl enable node_exporter
    sudo systemctl start node_exporter
    sudo systemctl status node_exporter 
    
Add job_name with static_configs 

    sudo vim /etc/prometheus/prometheus.yml

    ...
      - job_name: node_export
        static_configs:
          - targets: ["localhost:9100"]

Check if the config is valid: 

    promtool check config /etc/prometheus/prometheus.yml

Use a post request to reload the config: 

    curl -X POST http://localhost:9090/-/reload

## Install Grafana

Make sure that all dependencies are installed:

    sudo apt-get install -y apt-transport-https software-properties-common

Add GPG key:

    wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

Add this repository for stable releases:

    echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

Update and install Grafana:

    sudo apt-get update
    sudo apt-get -y install grafana

Start and check status of Grafana

    sudo systemctl enable grafana-server
    sudo systemctl start grafana-server
    sudo systemctl status grafana-server

Go to http://localhost:3000 and log into the Grafana using default credentials. Username: admin / Password: admin
To visualize metrics, add data source and select Prometheus. For the URL, enter http://localhost:9090. Click save and test. 

Create a new datasources.yaml file: 

    sudo vim /etc/grafana/provisioning/datasources/datasources.yaml

    apiVersion: 1

    datasources:
      - name: Prometheus
        type: prometheus
        url: http://localhost:9090
        isDefault: true

Restart Grafana to reload the config: 

    sudo systemctl restart grafana-server

## Display CPU and memory metrics on Grafana: 

On Grafana, click "Dashboard" and then "+New"

Select "Prometheus" as the data source in the "Metrics" tab. 

In the PromQL query field, enter a query: 

    100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[2m])) * 100

Click "Apply" to save the panel.

Add another panel for memory usage: 

In the PromQL query field, enter a query: 

    node_memory_MemTotal_bytes - node_memory_MemFree_bytes - node_memory_Buffers_bytes - node_memory_Cached_bytes

Click "Apply" to save the panel. 



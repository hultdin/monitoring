# 00. Docker + Prometheus + Grafana == True

This is an attempt to document the process of installing Docker, Prometheus, and Grafana on Ubuntu 16.04LTS<br>

# 01. Install Docker (CE) and dependencies
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -sudo apt-get update
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli
```

Verify that you now have the key with the fingerprint 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88, by searching for the last 8 characters of the fingerprint.

# 02. Install Docker Compose

Update path according to latest release of Docker Compose, see https://github.com/docker/compose/releases
```
sudo mkdir -p /opt/bin
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /opt/bin/docker-compose
sudo chmod +x /opt/bin/docker-compose
```

# 03. Install Portainer (optional)
Create a volume for Portainer (volumes are the preferred way to persist data in Docker containers and services)
```
sudo docker volume create portainer
```
Create docker-compose.yaml for the 'portainer' stack, see https://github.com/hultdin/monitoring/blob/master/portainer-docker-compose.yml
```
sudo mkdir -p /opt/docker/compose/portainer
wget -q -O - https://raw.githubusercontent.com/hultdin/monitoring/master/portainer-docker-compose.yml | sudo tee /opt/docker/compose/portainer/docker-compose.yaml
```

Build and start the Portainer container
```
cd /opt/docker/compose/portainer
sudo /opt/bin/docker-compose up --build -d
```
This will create a stack called 'portainer'

Open port 9000/tcp in the firewall unless the service is running behind a reverse proxy (recommended)

http://localhost:9000

# 04. Install 'monitoring' stack
Create a minimal Prometheus configuration in /opt/docker/volume/prometheus/config/prometheus.yml
```
# Global configuration
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: [
        'localhost:9090'
      ]
```
Create a directory for the Prometheus TSDB
```
sudo mkdir -p /opt/docker/volume/prometheus/data
sudo chown 65534:65534 /opt/docker/volume/prometheus/data
```
Create a Grafana configuration in /opt/docker/volume/grafana/config/grafana.ini by fetching the default configuration from master branch at https://github.com/grafana/grafana/blob/master/conf/defaults.ini or a custom tag.
```
sudo mkdir -p /opt/docker/volume/grafana/config
wget -q -O - https://raw.githubusercontent.com/grafana/grafana/master/conf/defaults.ini | sudo tee /opt/docker/volume/grafana/config/grafana.ini
```
Create a directory for plugins and a general Grafana volume
```
sudo mkdir -p /opt/docker/volume/grafana/plugins
sudo docker volume create grafana
```
Create the 'monitoring' network (bridge mode)
```
sudo docker network create -d bridge monitoring
```
Create docker-compose.yaml for the 'monitoring' stack, see https://github.com/hultdin/monitoring/blob/master/monitoring-docker-compose.yml
```
sudo mkdir -p /opt/docker/compose/monitoring
wget -q -O - https://raw.githubusercontent.com/hultdin/monitoring/master/monitoring-docker-compose.yml | sudo tee /opt/docker/compose/monitoring/docker-compose.yaml
```
Build and start the 'monitoring' stack (prometheus and grafana)
```
cd /opt/docker/compose/monitoring
sudo /opt/bin/docker-compose up --build -d
```
This will create a stack called 'monitoring'

Open port 3000/tcp and 9090/tcp in the firewall unless the service is running behind a reverse proxy (recommended)

# XX. Install and configure Apache
Apache is used as reverse proxy running on the "native" host (i.e. outside of Docker)
```
sudo apt-get update
sudo apt-get install apache2
sudo ufw allow 80,443/tcp
sudo a2dissite 000-default
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/redirect.conf
```
Update /etc/apache2/sites-available/redirect.conf to rewrite http to https, see https://github.com/hultdin/monitoring/blob/master/redirect.conf
```
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [NE,R,L]
```
Enable Apache mod_ssl and mod_rewrite to get the redirect working
```
sudo a2enmod ssl rewrite
sudo a2ensite redirect
```

# 05. Install blackbox_exporter (optional)

# 06. Install node_exporter (optional)
Download and install node_exporter
```
wget https://github.com/prometheus/node_exporter/releases/download/v0.18.0/node_exporter-0.18.0.linux-amd64.tar.gz
tar xzvf node_exporter-0.18.0.linux-amd64.tar.gz
sudo mkdir -p /opt/node_exporter/bin
sudo cp -r node_exporter-0.18.0.linux-amd64 /opt/node_exporter
sudo ln -s "../node_exporter-0.18.0.linux-amd64/node_exporter" /opt/node_exporter/bin/node_exporter
```

Create low privileged system account 'prometheus'
```
sudo useradd -r -M -N -g 65534 -s /bin/false prometheus
```

Create, enable, and start the node_exporter service, see https://github.com/hultdin/monitoring/blob/master/node_exporter.service
```
wget -q -O - https://raw.githubusercontent.com/hultdin/monitoring/master/node_exporter.service | sudo tee /lib/systemd/system/node_exporter.service
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

Open port 9100/tcp in the firewall unless the service is running behind a reverse proxy

http://localhost:9100/metrics

# XX. Configure Apache as reverse proxy
Redirect http to https using mod_proxy
```
sudo a2enmod proxy proxy_http
sudo cp /etc/apache2/sites-available/000-default-ssl.conf /etc/apache2/sites-available/redirect.conf
```
# 05. References
https://www.portainer.io/installation/
https://prometheus.io/docs/prometheus/latest/installation/#using-docker
https://devopscube.com/monitor-linux-servers-prometheus-node-exporter/
https://github.com/prometheus/node_exporter/releases

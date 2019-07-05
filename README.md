# 00. Docker + Prometheus + Grafana + MySQL == True

This is an attempt to document the process of installing Docker, Prometheus, Grafana, and MySQL on Ubuntu 16.04LTS<br>

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
Volumes are the preferred wa yto persist data in Docker containers and services 
```
sudo docker volume create portainer
```

Create the following docker-compose.yaml file in /opt/docker/compose/portainer
```
version: '3.7'
services:
  portainer:
    image: portainer/portainer:1.21.0
    container_name: 'portainer'
    network_mode: 'bridge'
    command: -H unix:///var/run/docker.sock
    ports:
      - '127.0.0.1:9000:9000'
    volumes:
      - '/var/run/docker.sock:/var/run/docker.sock'
      - 'portainer:/data'
    restart: unless-stopped
volumes:
    portainer:    
       external: true
```
Build and start the Portainer container
```
cd /opt/docker/compose/portainer
sudo /opt/bin/docker-compose up --build -d
```

# 04. Install and configure Apache
Apache is used as reverse proxy running on the "native" host (i.e. outside of Docker)
```
sudo apt-get update
sudo apt-get install apache2
sudo ufw allow 80, 443/tcp
sudo a2dissite 000-default
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/redirect.conf
```
Update /etc/apache2/sites-available/redirect.conf to rewrite http to https
```
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [NE,R,L]
```
Enable mod_ssl and mod_rewrite to get the redirect working
```
sudo a2enmod ssl rewrite
sudo a2ensite redirect
```

# 05. Configure Apache as reverse proxy
Redirect http to https using mod_proxy
```
sudo a2enmod proxy proxy_http
```
# XX. References
https://www.digitalocean.com/community/tutorials/how-to-use-apache-http-server-as-reverse-proxy-using-mod_proxy-extension

🧱 Architecture
                  ┌──────────────┐
                  │  DEV SERVER  │  (Monitoring Server)
                  │              │
                  │ Prometheus   │
                  │ Grafana      │
                  │ Loki         │
                  │ Alertmanager │
                  └─────┬────────┘
                        │
          ┌─────────────┼─────────────┐
          │             │             │
     ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
     │  DEV    │   │   UAT   │   │  PROD   │
     │ NodeExp │   │ NodeExp │   │ NodeExp │
     │Promtail │   │Promtail │   │Promtail │
     └─────────┘   └─────────┘   └─────────┘





🧱 PHASE 1: SAFE ARCHITECTURE 
DEV (Monitoring Server)

Containerized:

Prometheus
Grafana
Loki
Alertmanager


UAT + PROD (Agents only)
Node Exporter
Promtail

👉 This ensures:

Monitoring isolated
Minimal footprint

🟡 PHASE 2: INSTALL DOCKER (DEV ONLY)
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker

Add user:

sudo usermod -aG docker $USER
newgrp docker


🟡 PHASE 3: CREATE MONITORING STACK
sudo mkdir -p /opt/monitoring
cd /opt/monitoring
Create Docker Compose
nano docker-compose.yml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - "--storage.tsdb.retention.time=7d"
    ports:
      - "127.0.0.1:9090:9090"
    mem_limit: 400m
    restart: always

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    mem_limit: 200m
    restart: always

  loki:
    image: grafana/loki
    container_name: loki
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "127.0.0.1:3100:3100"
    mem_limit: 200m
    restart: always

  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    ports:
      - "127.0.0.1:9093:9093"
    mem_limit: 100m
    restart: always

volumes:
  prometheus_data:
  grafana_data:
  
  
🟡 PHASE 4: PROMETHEUS CONFIG
nano prometheus.yml
global:
  scrape_interval: 30s

scrape_configs:
  - job_name: "servers"
    static_configs:
      - targets:
          - "DEV_IP:9100"
          - "UAT_IP:9100"
          - "PROD_IP:9100"

👉 Replace with real IPs

🟡 PHASE 5: START MONITORING STACK
docker-compose up -d

Check:
docker ps

🟡 PHASE 6: INSTALL NODE EXPORTER (ALL SERVERS)

Run on DEV, UAT, PROD:

wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-*.tar.gz
tar -xvf node_exporter-*.tar.gz
sudo mv node_exporter-* /opt/node_exporter

Create service:

sudo nano /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter

[Service]
ExecStart=/opt/node_exporter/node_exporter
Restart=always
Nice=10

[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reexec
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

🟡 PHASE 7: INSTALL PROMTAIL (ALL SERVERS)
wget https://github.com/grafana/loki/releases/latest/download/promtail-linux-amd64
chmod +x promtail-linux-amd64
sudo mv promtail-linux-amd64 /usr/local/bin/promtail

Config:

sudo nano /etc/promtail-config.yml
server:
  http_listen_port: 9080

clients:
  - url: http://DEV_IP:3100/loki/api/v1/push

positions:
  filename: /tmp/positions.yaml

scrape_configs:
  - job_name: syslogs
    static_configs:
      - targets:
          - localhost
        labels:
          job: syslog
          __path__: /var/log/*.log

Service:

sudo nano /etc/systemd/system/promtail.service
[Unit]
Description=Promtail

[Service]
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail-config.yml
Restart=always
Nice=10

[Install]
WantedBy=multi-user.target


sudo systemctl enable promtail
sudo systemctl start promtail

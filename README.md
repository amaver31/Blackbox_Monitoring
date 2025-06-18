# Step-by-Step Monitoring Setup of Private URL using Blackbox Exporter
Step 1: Create Your Project Directory
Open your terminal and run:			

mkdir monitoring-setup
cd monitoring-setup 
# Step 2: Create docker-compose.yml
In the 'monitoring-setup' directory, create a file named 'docker-compose.yml' with the following content:

version: '3.8'
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "8080:80"
    networks:
      - monitor-net

  blackbox:
    image: prom/blackbox-exporter:latest
    container_name: blackbox
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox.yml:/config/blackbox.yml:ro
    command:
      - "--config.file=/config/blackbox.yml"
    networks:
      - monitor-net

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    networks:
      - monitor-net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - monitor-net

networks:
  monitor-net:
    driver: bridge
# Step 3: Configure Blackbox Exporter
Create a file named 'blackbox.yml' in the same directory with this content:

modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2"]
      valid_status_codes: [200]
# Step 4: Configure Prometheus
Create a file named 'prometheus.yml' with the following content:

global:
  scrape_interval: 15s

scrape_configs:
- job_name: 'prometheus'
  static_configs:
    - targets: ['prometheus:9090']

- job_name: 'blackbox'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets: ['http://nginx:80']                  #or http://abc.example.com:8080   or http://localhost:8080
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox:9115
# Step 5: Launch Services
Run the following command in the project directory:

docker compose up -d

This starts:
- nginx on port 8080
- Blackbox Exporter on 9115
- Prometheus on 9090
- Grafana on 3000
# Step 6: Verify Setup
- Open Prometheus UI: http://localhost:9090/targets
  → Check if 'prometheus' and 'blackbox' jobs are UP

- Open Grafana: http://localhost:3000
  → Login: admin/admin, then change password
  → Add Prometheus as a data source (URL: http://prometheus:9090)
  → Import Dashboard (e.g., ID: 7587) to view Blackbox metrics
# Step 7: Monitor Custom URLs
To monitor a private URL like http://abc.example.com:8080, which is a local host url, add this to 'prometheus.yml':

- job_name: 'blackbox-private'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets: ['http://abc.example.com:8080’]
  relabel_configs:
    # same as in 'blackbox' job

Then reload Prometheus:

docker exec prometheus kill -HUP 1

→ Check Prometheus Targets again to confirm status
# Full Summary
Component	Port	Purpose
nginx	8080	HTTP endpoint to monitor
blackbox-exporter	9115	Probes endpoint
prometheus	9090	Scrapes metrics
grafana	3000	Displays metrics/dashboards


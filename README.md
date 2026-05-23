Prometheus + Grafana Monitoring Stack

Observability stack deployed on AWS EC2. Scrapes metrics from node, nginx, and MongoDB, visualized in Grafana.
Built as a roadmap.sh monitoring project: https://roadmap.sh/projects/monitoring


STACK

Prometheus         prom/prometheus:latest                  metrics scraping and storage
Grafana            grafana/grafana:latest                  dashboards and visualization
node_exporter      prom/node-exporter:latest               host metrics (CPU, RAM, disk, network)
nginx-exporter     nginx/nginx-prometheus-exporter:latest  nginx connection and request metrics
mongodb_exporter   percona/mongodb_exporter:0.40           MongoDB metrics


ARCHITECTURE

All containers run on a single EC2 instance alongside an existing blue-green deployment stack.
Exporters that need to reach blue-green containers join blue-green-deployment_default as an external network.
No changes to the blue-green compose file are needed.

EC2 (13.219.84.108)
  blue-green-deployment stack (existing)
    nginx  -- stub_status enabled
    mongo
  monitoring stack (this repo)
    prometheus     -- scrapes all exporters
    grafana        -- port 3001
    node-exporter  -- port 9100
    nginx-exporter -- port 9113, joins blue-green-deployment_default
    mongodb-exporter -- port 9216, joins blue-green-deployment_default


PORTS

3001   Grafana UI (host-mapped)
9090   Prometheus (internal only)
9100   node_exporter (internal)
9113   nginx-exporter (internal)
9216   mongodb_exporter (internal)


DEPLOY

Prerequisites: Docker + Compose v2, EC2 security group with port 3001 open.

  git clone https://github.com/adrifok/prometheus-grafana-metrics-setup
  cd prometheus-grafana-metrics-setup
  echo "GRAFANA_PASSWORD=your-password" > .env
  docker compose up -d

Verify targets:

  docker exec prometheus wget -qO- http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E health|job


GRAFANA ACCESS

URL:      http://13.219.84.108:3001
User:     admin
Password: value of GRAFANA_PASSWORD in .env


DASHBOARDS

Node Exporter Full   working    Grafana ID 1860
NGINX Exporter       working    Grafana ID 12708
MongoDB              no data    Grafana ID 2583

MongoDB dashboard note:
Dashboard 2583 was built for an older version of mongodb_exporter. The exporter v0.40 prefixes
metrics with mongod_ (e.g. mongodb_mongod_metrics_document_total) while the dashboard queries
the old names (e.g. mongodb_metrics_document_total). Prometheus is collecting MongoDB metrics --
the gap is at the display layer. Fix: build a custom dashboard using the actual metric names,
or downgrade the exporter to a version that matches the dashboard.


NGINX STUB_STATUS

Required for nginx-exporter. Added to the blue-green nginx config:

  location /stub_status {
      stub_status;
      allow 172.0.0.0/8;
      deny all;
  }

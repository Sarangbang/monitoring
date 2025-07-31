# EC2 Prometheus + Grafana Monitoring Stack

이 저장소는 Prometheus와 Grafana, Node Exporter를 이용하여 간단하게 서버 모니터링 스택을 구성하기 위한 예제 설정을 제공합니다. `docker-compose`로 손쉽게 기동할 수 있으며, 주요 구성 파일들은 다음과 같은 구조로 배치되어 있습니다.

```text
/opt/monitoring
├── .env.sample
├── docker-compose.yml
├── data
│   ├── grafana
│   └── prometheus
├── prometheus
│   └── prometheus.yml
└── grafana
    └── provisioning
        ├── datasources
        │   └── prometheus.yaml
        └── dashboards
            └── dashboards.yaml
```

## .env.sample

```dotenv
#################################
#  Monitoring Stack Environment #
#################################

# ── Grafana ─────────────────────────────────────────
GRAFANA_ADMIN_PASSWORD=ChangeMe123!
GRAFANA_DOMAIN=

# ── 이미지 태그(선택) ─────────────────────────────────
PROMETHEUS_IMAGE_TAG=v2.48.0
GRAFANA_IMAGE_TAG=11.0.0
NODE_EXPORTER_IMAGE_TAG=v1.8.1

# ── TSDB 보존 주기 (일) ─────────────────────────────
PROM_RETENTION_DAYS=30
```

환경 변수 파일을 복사하여 실제 값을 입력한 뒤 사용합니다.

## docker-compose.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:${PROMETHEUS_IMAGE_TAG:-v2.48.0}
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./data/prometheus:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=${PROM_RETENTION_DAYS:-30}d'
    ports:
      - "9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana-oss:${GRAFANA_IMAGE_TAG:-11.0.0}
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
      - GF_SERVER_DOMAIN=${GRAFANA_DOMAIN}
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
    volumes:
      - ./data/grafana:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    depends_on:
      - prometheus
    ports:
      - "3000:3000"
    restart: unless-stopped

  node_exporter:
    image: prom/node-exporter:${NODE_EXPORTER_IMAGE_TAG:-v1.8.1}
    container_name: node_exporter
    pid: host
    network_mode: host
    command:
      - '--path.rootfs=/host'
    volumes:
      - /:/host:ro,rslave
    restart: unless-stopped
```

## prometheus/prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['10.0.0.10:9100']

  - job_name: 'spring'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['10.0.0.20:8080']
```

## Grafana Provisioning

`grafana/provisioning` 디렉터리에는 데이터소스와 대시보드를 자동으로 등록하기 위한 설정이 포함됩니다. 필요에 따라 추가 대시보드 JSON 파일을 이 위치에 두면 Grafana에 자동으로 로드됩니다.

### datasources/prometheus.yaml
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### dashboards/dashboards.yaml
```yaml
apiVersion: 1
providers:
  - name: default
    folder: 'Imported'
    type: file
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

## 배포 방법

```bash
# 디렉터리 생성
sudo mkdir -p /opt/monitoring/data/{grafana,prometheus}
sudo mkdir -p /opt/monitoring/prometheus
sudo mkdir -p /opt/monitoring/grafana/provisioning/{datasources,dashboards}
cd /opt/monitoring

# 샘플 환경변수 파일 복사
cp .env.sample .env
# 필요한 값 수정
nano .env

# 스택 기동
docker compose pull
docker compose up -d
```

# Standalone ELK (Elasticsearch, Logstash, Kibana)

A minimal, production-friendly *single-node* ELK stack using Docker Compose.

## What you get
- **Elasticsearch** (single-node) on port **9200**
- **Kibana** on port **5601**
- **Logstash** with:
  - Beats input on **5044**
  - TCP/UDP JSON input on **5050**
  - Monitoring API on **9600**

> Defaults to **security disabled** for quickest local start. See the "Enable security" section below to turn it on.

---

## Prerequisites
- Docker & Docker Compose v2
- Linux/macOS/WSL2: increase memory map counts (only needed once per host):

```bash
sudo sysctl -w vm.max_map_count=262144
# Persist across reboots:
# echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-elasticsearch.conf
# sudo sysctl --system
```

---

## Usage

### 1) Start the stack
```bash
docker compose up -d
```

### 2) Verify services

- Elasticsearch:
  ```bash
  curl -s http://localhost:9200 | jq .
  ```
- Kibana UI: open http://localhost:5601
- Logstash health:
  ```bash
  curl -s http://localhost:9600/_node/pipelines | jq .
  ```

### 3) Send test logs

- **JSON via TCP**:
  ```bash
  echo '{ "message": "hello elk", "app": "demo", "env": "dev", "timestamp": "2025-01-01T00:00:00Z" }' | nc -q0 localhost 5000
  ```

- **Beats (Filebeat)**
  Example `filebeat.yml` on a client:
  ```yaml
  filebeat.inputs:
    - type: filestream
      id: demo
      paths: ["/var/log/*.log"]

  output.logstash:
    hosts: ["YOUR_ELK_HOST_IP:5044"]
  ```

Open Kibana → **Analytics > Discover** to see indices such as `logs-YYYY.MM.DD` or your Beat-named indices.

---

## Tuning

- **Memory**: Edit `ES_JAVA_OPTS` and `LS_JAVA_OPTS` in `docker-compose.yml`.
- **Storage**: Elasticsearch data lives in the `esdata` volume. Use a dedicated disk in production.
- **Pipelines**: Add or modify files under `logstash/pipeline/*.conf`. Logstash reloads pipelines automatically on container restart.

---

## Enable security (simple dev-friendly mode)

> This enables auth but **keeps TLS off** for simplicity—fine for local/dev; harden for production.

1. Edit `docker-compose.yml`:

```yaml
services:
  elasticsearch:
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - xpack.security.transport.ssl.enabled=false
      - ELASTIC_PASSWORD=changeme

  kibana:
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=changeme

  logstash:
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m
```

2. Update `logstash/pipeline/logstash.conf` Elasticsearch output:

```ruby
output {
  elasticsearch {
    hosts => [ "http://elasticsearch:9200" ]
    user  => "elastic"
    password => "changeme"
    index => "%{[@metadata][beat]:logs}-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

3. Restart:
```bash
docker compose up -d --force-recreate
```

---

## Notes

- For real production, enable TLS for HTTP/transport, create least-privileged Logstash and Kibana users/roles, and set persistent JVM tuning and OS sysctls.
- Common ports:
  - 9200 (Elasticsearch REST)
  - 5601 (Kibana)
  - 5044 (Beats to Logstash)
  - 5000 (Syslog/JSON to Logstash)
  - 9600 (Logstash monitoring)

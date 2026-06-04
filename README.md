# 🚀 POC OpenTelemetry Local (Opção 2 — Containers Independentes + ClickHouse)

Este guia descreve uma Prova de Conceito (POC) do OpenTelemetry usando **containers Docker totalmente independentes**, simulando o cenário real de múltiplas máquinas enviando dados para um Gateway central com persistência em um banco **ClickHouse**.

---

# 🎯 Objetivo

Validar uma arquitetura realista:

* Um **OTel Gateway** rodando como "servidor central"
* Um **OTel Agent** rodando como "máquina independente" (coletando métricas do host)
* Comunicação via OTLP (gRPC) e persistência via Native TCP
* Execução isolada em containers, simulando cenários reais de rede

---

# 🏗️ Arquitetura

```text
┌──────────────────────────────┐
│        OTel Agent            │
│ (simula VPS / host local)    │
└──────────────┬───────────────┘
               │ OTLP (gRPC - 4317)
               ▼
┌──────────────────────────────┐
│       OTel Gateway           │
│    (Processa e Envia)        │
└──────────────┬───────────────┘
               │ Native TCP (9000)
               ▼
┌──────────────────────────────┐
│        ClickHouse            │
│    (Métricas Salvas)         │
└──────────────────────────────┘
```

---

# 📁 Estrutura do Projeto

Crie três diretórios independentes na sua máquina:

```text
clickhouse/
└── docker-compose.yml

otel-gateway/
├── docker-compose.yml
└── config.yaml

otel-agent/
├── docker-compose.yml
└── config.yaml
```

---

# 🗄️ 1. ClickHouse (Banco de Dados)

O banco de dados precisa ser o primeiro a subir para que o Gateway consiga estabelecer o canal de conexão.

## 🐳 `clickhouse/docker-compose.yml`

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse-server

    ports:
      - "8123:8123" # Interface HTTP
      - "9000:9000" # Native TCP Client (Usado pelo OTel Gateway)

    environment:
      - CLICKHOUSE_DB=otel
      - CLICKHOUSE_USER=default
      - CLICKHOUSE_PASSWORD=password123

    ulimits:
      nofile:
        soft: 262144
        hard: 262144

    restart: always
```

## 🚀 Subir o ClickHouse

Dentro da pasta `clickhouse`:

```bash
docker compose up -d
```

Verifique:

```bash
docker ps
```

---

# 🧠 2. OTel Gateway (Servidor Central)

O Gateway precisa escutar o mundo externo em todas as interfaces (`0.0.0.0`) e utilizar o exportador oficial do ClickHouse.

## 📄 `otel-gateway/config.yaml`

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"

processors:
  batch:

exporters:
  debug:
    verbosity: detailed

  clickhouse:
    endpoint: "tcp://host.docker.internal:9000?username=default&password=password123&database=otel"
    ttl: 72h
    timeout: 5s

    retry_on_failure:
      enabled: true

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, clickhouse]
```

## 🐳 `otel-gateway/docker-compose.yml`

```yaml
services:
  gateway:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-gateway

    command:
      - "--config=/etc/otel/config.yaml"

    volumes:
      - ./config.yaml:/etc/otel/config.yaml

    ports:
      - "4317:4317"
      - "4318:4318"

    extra_hosts:
      - "host.docker.internal:host-gateway"

    restart: always
```

## 🚀 Subir o Gateway

Dentro da pasta `otel-gateway`:

```bash
docker compose up -d
```

Validar logs:

```bash
docker logs -f otel-gateway
```

Você deve observar algo semelhante a:

```text
Starting GRPC server
Starting HTTP server
Everything is ready. Begin running and processing data.
```

---

# 📡 3. OTel Agent (Simulando VPS)

O Agent coleta métricas locais e envia explicitamente utilizando gRPC.

## 📄 `otel-agent/config.yaml`

```yaml
receivers:
  hostmetrics:
    collection_interval: 10s

    scrapers:
      cpu:
      memory:
      load:
      filesystem:
      network:

processors:
  batch:

exporters:
  otlp_grpc:
    endpoint: "host.docker.internal:4317"

    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [hostmetrics]
      processors: [batch]
      exporters: [otlp_grpc]
```

## 🐳 `otel-agent/docker-compose.yml`

```yaml
services:
  agent:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-agent

    command:
      - "--config=/etc/otel/config.yaml"

    volumes:
      - ./config.yaml:/etc/otel/config.yaml

    extra_hosts:
      - "host.docker.internal:host-gateway"

    restart: always
```

## 🚀 Subir o Agent

Dentro da pasta `otel-agent`:

```bash
docker compose up -d
```

---

# 🔌 Comunicação entre os Containers

Fluxo de comunicação:

```text
Agent
  ↓ OTLP gRPC (4317)
Gateway
  ↓ Native TCP (9000)
ClickHouse
```

Estratégia utilizada:

* Agent → `host.docker.internal:4317`
* Gateway escuta em `0.0.0.0:4317`
* Gateway → `host.docker.internal:9000`

---

# 🧪 4. Validação da POC

## 📊 Verificar logs do Gateway

```bash
docker logs -f otel-gateway
```

Se tudo estiver correto, o exporter `debug` exibirá métricas recebidas.

Exemplo:

```text
ResourceMetrics
Metric Name: system.cpu.time
Metric Name: system.memory.usage
Metric Name: system.network.io
```

---

## 📊 Verificar logs do Agent

```bash
docker logs -f otel-agent
```

O log deve permanecer limpo.

Não devem aparecer mensagens como:

```text
connection refused
```

ou

```text
Exporting failed
```

---

# 💻 5. Consultar Dados no ClickHouse

Acesse o cliente SQL do ClickHouse:

```bash
docker exec -it clickhouse-server clickhouse-client \
  --user default \
  --password password123 \
  --database otel
```

---

## Query A — Verificar tabelas criadas automaticamente

```sql
SHOW TABLES;
```

Resultado esperado:

```text
otel_metrics_gauge
otel_metrics_sum
otel_logs
otel_traces
```

(Dependendo da versão do exporter, algumas tabelas podem variar.)

---

## Query B — Validar recebimento de métricas

Execute a consulta abaixo duas ou três vezes com alguns segundos de intervalo.

Os totais devem aumentar continuamente.

```sql
SELECT 'Gauge' AS tipo, MetricName, COUNT(*) AS total FROM otel_metrics_gauge GROUP BY MetricName
UNION ALL
SELECT 'Sum' AS tipo, MetricName, COUNT(*) AS total FROM otel_metrics_sum GROUP BY MetricName;
```

---

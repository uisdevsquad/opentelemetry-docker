# POC OpenTelemetry Local (Containers Independentes + ClickHouse)

Prova de conceito do OpenTelemetry usando containers Docker independentes,
simulando múltiplas máquinas enviando dados para um Gateway central com
persistência em ClickHouse.

## Arquitetura

```
┌──────────────────────────────┐
│        OTel Agent            │
│ (simula VPS / host local)    │
│   CPU, RAM, Disk, Network    │
│   GPU (NVIDIA, opcional)     │
└──────────────┬───────────────┘
               │ OTLP gRPC (:4317)
               ▼
┌──────────────────────────────┐
│       OTel Gateway           │
│    (Processa e Envia)        │
└──────────────┬───────────────┘
               │ Native TCP (:9000)
               ▼
┌──────────────────────────────┐
│        ClickHouse            │
│    (Métricas Salvas)         │
└──────────────────────────────┘
```

Todos os containers compartilham a rede `otel-network` e se comunicam pelo
nome do serviço (DNS interno do Docker).

## Pré-requisitos

- Docker + Docker Compose
- (Opcional) NVIDIA GPU + NVIDIA Container Toolkit para métricas de GPU

## Subir tudo

```bash
# 1. Criar rede compartilhada (uma vez apenas)
docker network create otel-network

# 2. Ordem obrigatória: ClickHouse -> Gateway -> Agent
cd clickhouse    && docker compose up -d
cd otel-gateway  && docker compose up -d
cd otel-agent    && docker compose up -d
```

## O que cada serviço coleta

### Agent (CPU + sistema)

`hostmetrics` coleta do host:

| Métrica | Descrição |
|---|---|
| system.cpu.time | Tempo de CPU por core e estado |
| system.cpu.load_average.1m/5m/15m | Load average |
| system.memory.usage | Uso de memória RAM |
| system.memory.utilization | Percentual de memória usada |
| system.filesystem.usage | Uso de disco |
| system.filesystem.utilization | Percentual de disco usado |
| system.network.io | Tráfego de rede (bytes) |
| system.network.packets | Pacotes de rede |
| system.network.errors | Erros de rede |
| system.network.dropped | Pacotes descartados |
| system.disk.io | Operações de I/O de disco |
| system.disk.operations | Operações de leitura/escrita |
| system.process.count | Número de processos |
| system.processes.created | Processos criados |
| system.paging.usage | Uso de swap/paging |

### GPU NVIDIA (via DCGM Exporter)

Scrape Prometheus no DCGM Exporter, já incluso no docker-compose.

| Métrica | Significado | Unidade |
|---|---|---|
| DCGM_FI_DEV_GPU_UTIL | Utilização da GPU | % |
| DCGM_FI_DEV_MEM_COPY_UTIL | Utilização de cópia de memória | % |
| DCGM_FI_DEV_FB_USED | Memória de framebuffer usada | MiB |
| DCGM_FI_DEV_FB_FREE | Memória de framebuffer livre | MiB |
| DCGM_FI_DEV_GPU_TEMP | Temperatura da GPU | °C |
| DCGM_FI_DEV_POWER_USAGE | Consumo de energia | W |
| DCGM_FI_DEV_SM_CLOCK | Clock do Streaming Multiprocessor | MHz |
| DCGM_FI_DEV_MEM_CLOCK | Clock da memória | MHz |
| DCGM_FI_DEV_ENC_UTIL | Utilização do codificador de vídeo | % |
| DCGM_FI_DEV_DEC_UTIL | Utilização do decodificador de vídeo | % |
| DCGM_FI_DEV_XID_ERRORS | Erros XID (críticos) | erros |
| DCGM_FI_DEV_VGPU_LICENSE_STATUS | Status de licença vGPU | status |

### Gateway

Recebe via OTLP, faz batch, envia para ClickHouse e loga no console (debug).

## Validar funcionamento

### Logs do Gateway (ver métricas chegando)

```bash
docker logs -f otel-gateway
```

Saída esperada:

```
Metric Name: system.cpu.time
Metric Name: system.memory.usage
Metric Name: system.network.io
...
```

### Logs do Agent (sem erros)

```bash
docker logs otel-agent
```

Não deve conter `connection refused` ou `Exporting failed`.

## Consultar métricas no ClickHouse

```bash
docker exec -it clickhouse-server clickhouse-client \
  --user default --password password123 --database otel
```

### Ver tabelas

```sql
SHOW TABLES;
```

### Todas as métricas disponíveis

```sql
SELECT DISTINCT MetricName FROM otel_metrics_gauge ORDER BY MetricName;
```

### Últimas métricas de CPU

```sql
SELECT TimeUnix, MetricName, Value
FROM otel_metrics_gauge
WHERE MetricName LIKE '%cpu%'
ORDER BY TimeUnix DESC LIMIT 20;
```

### Últimas métricas de GPU (se habilitado)

```sql
SELECT TimeUnix, MetricName, Value, Attributes
FROM otel_metrics_gauge
WHERE MetricName LIKE '%DCGM%' OR MetricName LIKE '%gpu%'
ORDER BY TimeUnix DESC LIMIT 20;
```

### Totais por métrica (aumentam com o tempo)

```sql
SELECT MetricName, COUNT(*) AS total
FROM otel_metrics_gauge
GROUP BY MetricName
ORDER BY total DESC;
```

## GPU NVIDIA (opcional)

Usa o [DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter) da NVIDIA para
expor métricas de GPU, e o Agent faz o scrapping via receiver `prometheus`.

### Pré-requisito

Instale o [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
e configure o `runtime: nvidia` como padrão do Docker.

### Ativar

O DCGM Exporter já está definido no `docker-compose.yaml` do Agent com
`runtime: nvidia`. Basta recriar o Agent:

```bash
cd otel-agent && docker compose down && docker compose up -d
```

## Parar tudo

```bash
cd otel-agent    && docker compose down
cd otel-gateway  && docker compose down
cd clickhouse    && docker compose down
```

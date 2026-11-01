# 7. Monitoramento: Prometheus, node-exporter e Grafana

Objetivo: enxergar CPU, RAM, disco, rede e temperatura do servidor antes de subir os projetos, para saber quanto cada um consome.

## Peças

| Serviço | Função | Exposição |
|---|---|---|
| node-exporter | Coleta as métricas do servidor | Rede do host, porta 9100 (só para o Docker) |
| Prometheus | Guarda as métricas (coleta a cada 15 s, 30 dias de histórico) | Rede interna `monitoring` |
| Grafana | Dashboards | `grafana.home.lan` via Traefik |

Só o Grafana entra na rede `proxy`. O Prometheus e o node-exporter não ficam acessíveis de fora.

## Arquivos

Pasta `/opt/homelab/monitoring/`:

- `docker-compose.yml`
- `prometheus.yml`
- `.env`: senha do Grafana (**fora do Git**)

Senha gerada com:

```bash
echo "GRAFANA_PASSWORD=$(openssl rand -base64 18)" > .env
```

### prometheus.yml

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node
    static_configs:
      - targets: ['host.docker.internal:9100']
```

### docker-compose.yml (versão final)

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    extra_hosts:
      - "host.docker.internal:host-gateway"
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=30d
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    network_mode: host
    pid: host
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/rootfs
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - monitoring
      - proxy
    labels:
      - traefik.enable=true
      - traefik.http.routers.grafana.rule=Host(`grafana.home.lan`)
      - traefik.http.services.grafana.loadbalancer.server.port=3000

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
  proxy:
    external: true
```

```bash
cd /opt/homelab/monitoring
docker compose up -d
```

## Configuração do Grafana

1. Entrar em `http://grafana.home.lan` com o usuário `admin` e a senha do `.env`.
2. **Connections → Data sources → Prometheus**, URL `http://prometheus:9090`, depois **Save & test**.
3. **Dashboards → New → Import**, ID **1860** (Node Exporter Full).

## Problema: node-exporter isolado no container

Na primeira versão, o node-exporter rodava na rede `monitoring`, como os outros serviços. O dashboard abriu, mas com dois erros:

- **Nodename** aparecia como `3361a9190f44`, o ID do container, e não `homelab`.
- O gráfico de rede mostrava só `eth0`, a interface virtual do container. O Wi-Fi real (`wlo1`) não era medido.

**Causa:** dentro do Docker, o container tem a própria rede e o próprio nome de host. O node-exporter media o container, não o servidor.

**Solução:**

1. node-exporter com `network_mode: host` e `pid: host`, para enxergar a rede e os processos reais.
2. Prometheus alcançando o host por `host.docker.internal` (via `extra_hosts: host-gateway`).
3. Target trocado em `prometheus.yml`:

   ```bash
   sed -i 's/node-exporter:9100/host.docker.internal:9100/' prometheus.yml
   ```

4. Como agora a porta 9100 está no host, o UFW passa a valer para ela. Liberei a porta só para as redes internas do Docker:

   ```bash
   sudo ufw allow from 172.16.0.0/12 to any port 9100 proto tcp
   ```

### Verificação

```bash
docker exec prometheus wget -qO- http://localhost:9090/api/v1/targets | grep -o '"health":"[a-z]*"'
# "health":"up"
# "health":"up"
```

No Grafana, selecionar a instância `host.docker.internal:9100`. A antiga (`node-exporter:9100`) continua na lista por causa do histórico, mas não recebe mais dados.

## Retrato do servidor em repouso

Com Traefik, Portainer, Prometheus, node-exporter e Grafana rodando:

| Métrica | Valor |
|---|---|
| CPU | ~3% |
| Carga | ~3,5% |
| RAM | ~4% de 31 GiB |
| Disco | ~6% de 913 GiB |
| Swap | 0% de 8 GiB |

Interfaces de rede visíveis: `wlo1` (Wi-Fi), `tailscale0` (acesso remoto), `docker0`, `br-*` e `veth*` (redes virtuais do Docker).

## Aprendizados

- Container que mede o host precisa enxergar o host: `network_mode: host` e `pid: host`.
- Porta no host passa pelo UFW; porta publicada com `ports:` pelo Docker, não.
- Liberar só o necessário: a 9100 fica aberta para o Docker, não para a rede.
# 🏠 Homelab

Servidor pessoal montado em um notebook antigo com **Ubuntu Server 24.04**, usado para hospedar serviços self-hosted, estudar DevOps e rodar projetos pessoais.

Este repositório guarda a configuração dos serviços (Docker Compose) e o passo a passo de toda a montagem, incluindo os problemas encontrados e como foram resolvidos.

## Hardware

| Item | Detalhe |
|---|---|
| Máquina | Notebook Samsung Book (NP550XCJ) |
| CPU | Intel Core i3 10ª geração (4 threads) |
| RAM | ~32 GB |
| Disco | ~1 TB |
| Rede | Wi-Fi (sem cabo) |
| Sistema | Ubuntu Server 24.04 LTS, kernel HWE |

## Arquitetura

```
             Internet
                │
        ┌───────┴────────┐
        │   Tailscale    │  ← acesso remoto (VPN mesh, sem abrir portas)
        └───────┬────────┘
                │
   ┌────────────┴─────────────┐
   │  Servidor (Ubuntu Server)│
   │                          │
   │  UFW + fail2ban          │
   │        │                 │
   │   Traefik (:80)          │  ← única porta exposta pelo Docker
   │     ├── portainer.home.lan
   │     ├── traefik.home.lan
   │     └── grafana.home.lan  (planejado)
   └──────────────────────────┘
```

## Serviços

| Serviço | Endereço | Status |
|---|---|---|
| Traefik (reverse proxy) | `traefik.home.lan` | ✅ rodando |
| Portainer (gestão do Docker) | `portainer.home.lan` | ✅ rodando |
| Prometheus + Grafana + node-exporter | `grafana.home.lan` | 🔜 próximo |
| Gitea | — | 📋 planejado |
| n8n | — | 📋 planejado |
| Ollama (IA local) | — | 📋 planejado |
| AdGuard Home (DNS local) | — | 📋 planejado |
| Backups (restic) | — | 📋 planejado |

## Estrutura do repositório

```
/opt/homelab
├── README.md
├── docs/                  # passo a passo da montagem
├── traefik/
│   └── docker-compose.yml
├── portainer/
│   └── docker-compose.yml
└── .gitignore             # .env e segredos ficam fora do Git
```

Cada serviço tem sua própria pasta com um `docker-compose.yml`. Todos compartilham a rede Docker externa `proxy`, por onde o Traefik descobre os serviços através de labels.

## Jornada

1. [Instalação e rede Wi-Fi](docs/01-instalacao-e-rede.md)
2. [Acesso e segurança](docs/02-acesso-e-seguranca.md)
3. [Energia: tampa, tela e bateria](docs/03-energia.md)
4. [Docker, Traefik e Portainer](docs/04-docker-traefik-portainer.md)
5. [Acesso remoto com Tailscale](docs/05-tailscale.md)

## Como subir tudo do zero

```bash
docker network create proxy
cd /opt/homelab/traefik   && docker compose up -d
cd /opt/homelab/portainer && docker compose up -d
```

## Como atualizar um serviço

```bash
cd /opt/homelab/<servico>
docker compose pull
docker compose up -d
```

Os dados ficam em volumes Docker, então atualizar a imagem não apaga configurações.

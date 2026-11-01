# 4. Docker, Traefik e Portainer

## Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker jovas
exit   # sair e entrar de novo para a permissão valer

docker run hello-world
docker compose version
```

## Estrutura

```bash
docker network create proxy
sudo mkdir -p /opt/homelab/{traefik,portainer}
sudo chown -R jovas:jovas /opt/homelab
cd /opt/homelab && git init
echo ".env" > .gitignore
```

## Traefik

`traefik/docker-compose.yml`: reverse proxy na porta 80, que descobre os serviços pelas labels do Docker. Só serviços com `traefik.enable=true` são publicados.

```bash
cd /opt/homelab/traefik && docker compose up -d
```

## Portainer

`portainer/docker-compose.yml`: interface web para gerenciar o Docker, publicada pelo Traefik em `portainer.home.lan` (porta interna 9000).

```bash
cd /opt/homelab/portainer && docker compose up -d
```

### Problemas na primeira configuração

1. **Setup token**: as versões novas pedem um token, que aparece no log:

   ```bash
   docker logs portainer 2>&1 | grep -i token
   ```

2. **Timeout de 5 minutos**: se o admin não for criado a tempo, o Portainer trava por segurança. Solução:

   ```bash
   docker restart portainer && sleep 5 && docker logs portainer 2>&1 | grep -i token
   ```

3. **Edge Compute**: pulado (Skip). Só é útil para gerenciar servidores remotos.

## Nomes `*.home.lan` no PC (Linux Mint)

```bash
echo "<IP> traefik.home.lan portainer.home.lan grafana.home.lan" | sudo tee -a /etc/hosts
```

Hoje o IP usado é o do Tailscale (ver [05-tailscale.md](05-tailscale.md)).

## Atualizar

```bash
cd /opt/homelab/portainer
docker compose pull
docker compose up -d
```

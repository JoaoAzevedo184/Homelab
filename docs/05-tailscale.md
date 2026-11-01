# 5. Acesso remoto com Tailscale

Objetivo: acessar o homelab de fora de casa sem abrir portas no roteador, com tráfego criptografado (importante em Wi-Fi público).

## Servidor (Ubuntu Server)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up            # autorizar pelo link no navegador
tailscale ip -4
sudo ufw allow in on tailscale0
```

No painel (`login.tailscale.com/admin/machines`): **Disable key expiry** na máquina `homelab`. Sem isso, o servidor perde o acesso em 180 dias.

## PC (Linux Mint)

O PC já estava logado em outra conta do Tailscale. Opções consideradas:

- **A)** `tailscale logout` e entrar na conta do homelab ✅ (escolhida)
- **B)** manter as duas contas e alternar com `tailscale switch`
- **C)** compartilhar a máquina `homelab` com a outra conta (Share)

```bash
sudo tailscale logout
sudo tailscale up
tailscale status
```

## Nomes

O `/etc/hosts` do PC aponta os domínios `*.home.lan` para o IP do Tailscale do servidor. Assim, funciona tanto em casa quanto fora:

```
<IP_TAILSCALE> traefik.home.lan portainer.home.lan grafana.home.lan
```

SSH pelo nome do Tailscale:

```bash
ssh jovas@homelab
```

Em casa, o Tailscale conecta direto pela rede local (cerca de 6 ms de ping), sem passar por relay.

## Pendente

- [ ] Testar de fora de casa (roteador do celular / Shopping Patteo)
- [ ] DNS próprio (AdGuard Home) para os nomes `*.home.lan` funcionarem também no celular

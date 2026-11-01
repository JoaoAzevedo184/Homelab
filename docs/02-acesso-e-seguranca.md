# 2. Acesso e segurança

## Convenção de nomes

| Máquina | Prompt |
|---|---|
| PC (Linux Mint) | `joao@Caverna-Em-Construcao:~$` |
| Servidor (Ubuntu Server) | `jovas@homelab:~$` |

Olhar o prompt antes de rodar qualquer comando evita executar no lugar errado.

## SSH

O OpenSSH não veio instalado (`Unit ssh.service could not be found`).

**No Servidor (Ubuntu Server):**

```bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

**No PC (Linux Mint):** login por chave.

```bash
ssh-keygen -t ed25519
ssh-copy-id jovas@192.168.1.200
ssh jovas@192.168.1.200   # deve entrar sem senha
```

**No Servidor (Ubuntu Server):** só depois de confirmar a chave, desativar senha e root em `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PermitRootLogin no
```

```bash
sudo systemctl restart ssh
```

Testei em um segundo terminal antes de fechar o primeiro, para não ficar trancado do lado de fora.

## Atualizações e pacotes base

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git curl htop ufw fail2ban unattended-upgrades
```

Depois do upgrade apareceu `Pending kernel upgrade!`: bastou um `sudo reboot`.

## Firewall (UFW)

A ordem importa: liberar o SSH **antes** de ativar.

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

> ⚠️ O Docker ignora o UFW para portas publicadas com `ports:`. Por isso, só o Traefik publica portas; os demais serviços ficam atrás dele.

## fail2ban

Aviso inicial: `allowipv6 not defined`. Corrigido com `/etc/fail2ban/fail2ban.local`:

```ini
[DEFAULT]
allowipv6 = auto
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

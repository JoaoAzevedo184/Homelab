# 1. Instalação e rede Wi-Fi

## Instalação

- Ubuntu Server 24.04 instalado a partir de um pendrive com **Ventoy**.
- Durante a instalação, a internet veio do **roteador do celular** (Wi-Fi).

## Problema: sem internet depois da instalação

Depois de instalado, o servidor não tinha rede:

```bash
ip route          # não mostrava nada
ping -c 3 1.1.1.1 # Network is unreachable
```

**Causa:** o instalador salvou no netplan a rede Wi-Fi do **celular**. Com o roteador do celular desligado, o servidor tentava se conectar a uma rede que não existia mais.

**Solução:** editar o arquivo do netplan e trocar o nome da rede e a senha pelos do Wi-Fi de casa.

Outras descobertas nesse processo:

- `/etc/cloud/cloud.cfg.d` não existia porque o cloud-init não estava instalado, então não havia nada sobrescrevendo a rede.
- O `rfkill` não vem instalado. Dá para checar bloqueios do rádio direto pelo sysfs:

```bash
grep . /sys/class/rfkill/*/soft /sys/class/rfkill/*/hard
```

- O pacote `wpasupplicant`, necessário para redes com senha, já estava instalado.

## IP fixo no Wi-Fi

Arquivo `/etc/netplan/50-cloud-init.yaml` (sem credenciais reais):

```yaml
network:
  version: 2
  wifis:
    wlo1:
      dhcp4: false
      addresses: [192.168.1.200/24]
      routes:
        - to: default
          via: 192.168.1.254
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
      access-points:
        "NOME_DA_REDE":
          password: "SENHA_DO_WIFI"
```

```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan try   # reverte sozinho em 120 s se perder a conexão
```

Antes de fixar o IP, conferi se ele estava livre com `ping 192.168.1.200`.

## Desligar a economia de energia do Wi-Fi

Com o power save ligado, servidores Wi-Fi costumam "sumir" da rede depois de um tempo ociosos.

```bash
sudo apt install -y iw
iw dev wlo1 get power_save
```

Serviço `/etc/systemd/system/wifi-powersave-off.service`:

```ini
[Unit]
Description=Desliga power save do Wi-Fi
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iw dev wlo1 set power_save off

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now wifi-powersave-off.service
```

## Verificação após reboot

```bash
ip a show wlo1               # inet 192.168.1.200/24
iw dev wlo1 get power_save   # off
ping -c 3 google.com
```

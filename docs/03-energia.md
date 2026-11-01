# 3. Energia: tampa, tela e bateria

## Não suspender ao fechar a tampa

`/etc/systemd/logind.conf`:

```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

```bash
sudo systemctl restart systemd-logind
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

## Apagar a tela do console

Em `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="consoleblank=60"
```

```bash
sudo update-grub
```

## Limitar a carga da bateria em 80%

Ficar 24h em 100% desgasta a bateria rapidamente.

### Investigação

```bash
ls /sys/class/power_supply/            # ADP1  BAT1  ...
sudo dmidecode -s system-product-name  # 550XCJ/550XCR
ls /sys/bus/acpi/devices/ | grep -i SAM   # SAM0427:00
```

- O kernel 6.8 não expunha `charge_control_end_threshold`.
- O dispositivo ACPI `SAM0427` é o que o driver `samsung-galaxybook` controla. Esse driver entrou no kernel 6.15.
- ⚠️ Evitei o módulo antigo `samsung-laptop` com `force=1`: em notebooks UEFI ele já causou travamentos permanentes.

### Solução: kernel HWE

```bash
sudo apt install -y --install-recommends linux-generic-hwe-24.04
sudo reboot   # com a tampa aberta, por segurança
```

Verificação (kernel 7.0):

```bash
uname -r
lsmod | grep galaxybook
ls /sys/class/power_supply/BAT1/ | grep charge_control
```

Plano de volta, caso o Wi-Fi não subisse: escolher o kernel antigo em **Advanced options** no GRUB.

### Aplicar o limite

`/etc/systemd/system/battery-limit.service`:

```ini
[Unit]
Description=Limita carga da bateria em 80%
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo 80 > /sys/class/power_supply/BAT1/charge_control_end_threshold'

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now battery-limit.service
```

O driver não registrou perfis de energia (`platform_profile`) neste modelo, então o notebook segue o modo padrão da BIOS.

## Verificação após reboot

```bash
cat /sys/class/power_supply/BAT1/charge_control_end_threshold   # 80
grep -o consoleblank=60 /proc/cmdline                            # consoleblank=60
iw dev wlo1 get power_save                                       # off
```

# Домашнее задание:  DHCP, PXE: настройка PXE-сервера для автоматической установки

Автоматическая установка **Ubuntu Server 24.04 (noble)** по сети: PXE-клиент грузится по DHCP/TFTP, забирает kernel/initrd, скачивает ISO по HTTP и устанавливается полностью без интерактива (Subiquity autoinstall / cloud-init `ds=nocloud-net`).

## Стенд

Фактически выполнено на виртуалках Proxmox (узел `Hecate`):

| Роль | VMID | Сеть | IP |
|---|---|---|---|
| `pxeserver` | 510 | vmbr0 + `vmbr-net14` (PXE) | 192.168.1.155 + 10.0.0.20 |
| `pxeclient` | 511 | только `vmbr-net14`, boot = network | по DHCP из 10.0.0.100–120 |

## Что делает Ansible (`ansible/provision.yml`)

1. `dnsmasq` — DHCP (`10.0.0.100–120`) + TFTP (`/srv/tftp/amd64`) на `enp6s19`;
   по DHCP-опции `client-arch` разделяет загрузку: BIOS → `pxelinux.0`, UEFI (arch=7) → `bootx64.efi`.
2. `apache2` — HTTP-репозиторий: ISO (`/srv/images/noble-live-server-amd64.iso`) и seed-файлы (`/srv/ks/`).
3. Netboot-файлы Ubuntu (kernel `linux`, `initrd`, `pxelinux.0`, GRUB для UEFI) из `noble-netboot-amd64.tar.gz`.
4. Шаблоны (`templates/`):
   - `dnsmasq-pxe.conf.j2` — DHCP/TFTP, тегирование UEFI-клиентов;
   - `pxelinux-default.j2` — BIOS-меню: `ip=dhcp iso-url=… autoinstall ds=nocloud-net;s=http://10.0.0.20/srv/ks/`;
   - `grub.cfg.j2` — то же для UEFI-загрузки GRUB'ом по сети;
   - `user-data.j2` — autoinstall: пользователь `otus`, пароль, SSH-сервер, раскладка, разметка `direct` (largest disk), **`shutdown: poweroff`** (клиент выключается после установки), late-commands с установкой GRUB под фактический firmware (UEFI → ESP с fallback `BOOTX64.EFI`, BIOS → MBR);
   - `meta-data`, `vendor-data` — пустые (требование nocloud).
5. Перезапуск `dnsmasq`, `apache2`.

## Запуск

```bash
# Вариант Vagrant (VirtualBox, intnet 'pxenet'):
vagrant up

# Вариант Proxmox (как на стенде):
#   VM 510/511 уже описаны; плейбук выполняется на pxeserver локально:
cd ansible
ansible-playbook provision.yml -c local -i "pxeserver," -e @vars.yml
```

## Как проверить автоустановку

1. Клиент (VM 511) без ОС на диске, boot order = network.
2. **BIOS:** клиент получает адрес по DHCP, грузит `pxelinux.0` → kernel/initrd → автоустановка → poweroff.
3. **UEFI:** у клиента `bios=ovmf` (+ efidisk); dnsmasq отдаёт `bootx64.efi` → GRUB по TFTP → та же автоустановка → poweroff.
4. После poweroff переключить boot на диск — загружается установленная система.
5. Вход по SSH: `ssh otus@<клиентский IP>` (пароль — `otus`).

## Результаты проверки (факт, 2026-09-30)

- Плейбук: `ok=17, failed=0`, идемпотентен (повторные прогоны без ошибок).
- **BIOS (seabios):** PXE-загрузка → autoinstall до конца → poweroff → загрузка с диска:
  `hostname=linux`, `Ubuntu 24.04.3 LTS`, kernel `6.8.0-71-generic`, `ens18 UP 10.0.0.108/24`, `/boot` полный, `ssh active`, вход `otus/otus` подтверждён.
- **UEFI (OVMF + efidisk):** DHCP отдаёт `bootx64.efi` → GRUB по TFTP → тот же user-data → autoinstall → poweroff → загрузка с диска в UEFI-режиме (`/sys/firmware/efi` активен, `/boot/efi` на месте), SSH-вход подтверждён.
- Установленный клиент работает в PXE-сети (DHCP от dnsmasq), интерактива при установке нет.

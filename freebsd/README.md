# Роль freebsd — scripted install (bsdinstall)

Unattended-установка **FreeBSD 13/14/15 с установочного ISO** (disc1).

## Механизм установки

Инсталлятор FreeBSD — **bsdinstall**. Его режим автоматизации — **скрипт установки** (`installerconfig`):

- **preamble** — переменные окружения установщика (`PARTITIONS`, `DISTRIBUTIONS`, `ROOTPASS_ENC`);
- **setup script** — shell-скрипт после `#!`, который выполняется в `chroot` только что установленной системы.

Ключевая особенность: если на установочном носителе есть **`/etc/installerconfig`**, bsdinstall при загрузке сам запускает его в scripted-режиме и после установки перезагружает систему (см. `bsdinstall(8)`, раздел `SCRIPTING`). Поэтому:

1. Роль генерирует `installerconfig` (hostname, сеть DHCP/static, SSH-ключ, пользователь с хешем пароля, sudo NOPASSWD, timezone).
2. `xorriso` ремастит **копию** установочного ISO, добавляя в неё `/etc/installerconfig`; загрузочные записи сохраняются (`-boot_image any replay`). Оригинальный ISO не изменяется.
3. Передаёт факты в общую часть [`install_vm.yml`](../install_vm.yml) с `boot_mode: cdrom`: VM грузится с ремастер-копии как с CD-ROM.
4. `--boot hd,cdrom` задаёт финальный порядок загрузки: во время установки virt-install грузится с CD, а после reboot создаёт финальный домен с загрузкой **с диска** (и без установочного ISO в cdrom) — цикла переустановки не возникает.
5. `--wait` ждёт завершения установки и финального reboot в уже установленную систему.

## Почему `--cdrom`, а не `--location`

`virt-install --location` рассчитан на Linux-инсталляторы: он извлекает из ISO ядро и initrd и передаёт их QEMU. FreeBSD загружается через свой `loader` + `mfsroot`, поэтому такой способ не работает. Загрузка с ISO как с CD-ROM (`--cdrom`) — штатный путь для FreeBSD.

## Сеть

- **DHCP** (`vm_ip_mode: dhcp`) — `ifconfig_<if>=DHCP`; интерфейс определяется автоматически (`vtnet0` для virtio-net, также поддерживаются `em`, `igb`, `re`, `bge`).
- **Статический IP** (`vm_ip_mode: static`) — `ifconfig_<if>="inet <ip> netmask <mask>"` + `defaultrouter`; DNS пишется в `/etc/resolv.conf`.

## sudo во FreeBSD

В базовой системе FreeBSD **нет sudo**. Скрипт установки пытается поставить его из `pkg` (нужна сеть):

```sh
env ASSUME_ALWAYS_YES=yes pkg install -y sudo
```

Если сети/репозитория нет, установка всё равно завершается успешно, а sudo можно поставить позже:

```sh
pkg install sudo
```

Файл `sudoers.d` создаётся в `/usr/local/etc/sudoers.d/` (правильный путь для FreeBSD, а не `/etc/sudoers.d/`).

## Требования к образу

- **disc1 ISO** — содержит `base.txz` и `kernel.txz`;
- **bootonly ISO не подходит** — в нём нет наборов для установки;
- на хосте нужен **xorriso** (как и для Debian).

`os_variant: generic` подходит; при наличии в `osinfo-query os` можно указать `freebsd14.0`, `freebsd15.0` и т.п.

## Переменные

Используются общие переменные из [`../vars.yml`](../vars.yml): `vm_name`, `vm_memory_mb`, `vm_vcpus`, `vm_disk_size`, `iso_path`, `ssh_pub_key`, `vm_password_hash`, `vm_user`, `vm_network`, `vm_network_mode`, `vm_ip_mode`, `vm_ip`, `vm_gateway`, `vm_netmask`, `vm_prefix`, `os_variant`, `timezone`.

## Cloud-режим

FreeBSD также поддерживается в cloud-режиме (`install_mode: cloud` или `os_family: cloud`) с официальным образом `BASIC-CLOUDINIT`. Подробности — в [`../cloud/README.md`](../cloud/README.md).

## Что получается в итоге

Сервер без GUI, root залочен, пользователь в группе `wheel`, вход по SSH-ключу и паролю, sudo NOPASSWD (если удалось поставить пакет), ZFS/UFS-разметка по умолчанию bsdinstall.

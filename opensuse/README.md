# Роль opensuse — AutoYaST

Unattended-установка **openSUSE Leap 15.x / Tumbleweed с DVD ISO**.

## Механизм установки

Инсталлятор openSUSE — **YaST**, режим автоматизации — **AutoYaST**: XML-профиль `autoinst.xml` со всеми ответами (язык, разметка, сеть, пользователь, пакеты, скрипты).

1. Роль генерирует `autoinst.xml`:
   - guided-разметка **LVM на весь диск** (`<storage><proposal><lvm>true</lvm>...`);
   - пользователь с хешем пароля, root с хешем пароля (затем запирается);
   - пакеты `openssh`, `sudo`, паттерны `base` и `enhanced_base`;
   - `sshd` включён через `services-manager`;
   - `chroot-scripts` (выполняются в установленной системе до первой загрузки) раскладывают SSH-ключ, `sudoers.d` и запирают root.
2. Профиль упаковывается в мини-ISO с меткой **OEMDRV** (`genisoimage`). linuxrc/YaST автоматически ищут `autoinst.xml` в корне устройства с этой меткой; дополнительно путь задаётся явно: `autoyast=label://OEMDRV/autoinst.xml`.
3. Передаёт факты в общую часть [`install_vm.yml`](../install_vm.yml): `--location <DVD ISO>,kernel=boot/x86_64/loader/linux,initrd=boot/x86_64/loader/initrd`, OEMDRV-ISO вторым cdrom, `--wait` — до финального reboot.
4. OEMDRV-ISO извлекается из cdrom и удаляется.

## Сеть

Сеть на время установки настраивается параметром **linuxrc `ifcfg`**, а `keep_install_network` переносит её в установленную систему:

- **DHCP** (`vm_ip_mode: dhcp`) — `ifcfg=eth*=dhcp`;
- **Статический IP** (`vm_ip_mode: static`) — `ifcfg=eth*=<ip>/<prefix>,<gateway>,<dns>`.

`eth*` — специальный wildcard linuxrc: первый доступный интерфейс (в том числе `ens3`/`enp0s3`).

## Требования к образу

- **полный DVD ISO** openSUSE Leap / Tumbleweed;
- **Online-ISO не подходит** — в нём нет пакетов, установка требует сеть.

`os_variant: generic` подходит; при наличии в `osinfo-query os` можно указать `opensuse-leap15.6` и т.п.

## Переменные

Используются общие переменные из [`../vars.yml`](../vars.yml): `vm_name`, `vm_memory_mb`, `vm_vcpus`, `vm_disk_size`, `iso_path`, `ssh_pub_key`, `vm_password_hash`, `vm_user`, `vm_network`, `vm_network_mode`, `vm_ip_mode`, `vm_ip`, `vm_gateway`, `vm_netmask`, `vm_prefix`, `os_variant`, `timezone`.

## Cloud-режим

openSUSE также поддерживается в cloud-режиме (`install_mode: cloud` или `os_family: cloud`) с официальным образом **NoCloud**. Подробности — в [`../cloud/README.md`](../cloud/README.md).

## Что получается в итоге

Сервер без GUI, root залочен, пользователь с sudo NOPASSWD, вход по SSH-ключу и паролю, LVM на весь диск, openssh-server.

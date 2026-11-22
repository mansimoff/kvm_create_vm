# Роль cloud — готовые cloud-образы (без установки)

Запуск **официальных cloud-образов** — самый быстрый способ: секунды вместо 10+ минут установки с ISO.

Поддерживаются:

- **Linux**: Rocky / Alma / CentOS Stream / RHEL, Ubuntu, Debian, openSUSE Leap / Tumbleweed;
- **FreeBSD** (`BASIC-CLOUDINIT`, UFS или ZFS).

## Механизм работы

Cloud-образ — это **qcow2 с уже установленной, настроенной минимальной системой**. Инсталлятора в нём нет вообще: VM просто загружается в готовую ОС. Кто задаёт пользователя/ключ, если ОС уже установлена? — **cloud-init**, стандартный агент первого запуска, встроенный во все cloud-образы.

Порядок:

1. **Источник образа** — `cloud_image`: либо URL (скачивается в `images_dir` при первом запуске и переиспользуется между запусками), либо локальный путь к qcow2. Значение задаётся в `vars.yml`, при запуске можно оставить (Enter) или ввести другое.
2. **Overlay-диск**: `qemu-img create -b <базовый образ>` — VM пишет только в overlay, базовый образ остаётся неизменным и переиспользуется другими VM. `vm_disk_size` опционален (иначе наследует размер образа).
3. **Seed-ISO**: `user-data`/`meta-data` (NoCloud-датасорс cloud-init) на мини-ISO с меткой **cidata** — пользователь, sudo NOPASSWD, хеш пароля, SSH-ключ, hostname, timezone.
4. **`virt-install --import`**: режим «образ уже готов» — без `--location` (не нужно извлекать kernel/initrd инсталлятора) и без `--extra-args`. `--wait` не нужен: virt-install стартует VM и сразу выходит.
5. При первой загрузке cloud-init читает cidata-диск и применяет `user-data`. Seed извлекается **после** первой загрузки — это делает `create_vm.yml` в пост-обработке (после получения IP), т.к. NoCloud-датасорс читается один раз.

## Как включить cloud-режим

```yaml
# vars.yml
os_family: opensuse        # или freebsd / debian / ubuntu / rhel
install_mode: cloud
cloud_image: https://download.opensuse.org/repositories/Cloud:/Images:/Leap_15.6/images/openSUSE-Leap-15.6.x86_64-NoCloud.qcow2
```

Либо устаревший способ — `os_family: cloud` (ОС определяется по имени `cloud_image`):

```yaml
os_family: cloud
cloud_image: https://download.freebsd.org/releases/VM-IMAGES/15.1-RELEASE/amd64/Latest/FreeBSD-15.1-RELEASE-amd64-BASIC-CLOUDINIT-zfs.qcow2.xz
```

Если имя файла не содержит название ОС, задайте `cloud_os` явно: `cloud_os: freebsd`.

## Где брать образы

| ОС | URL |
|---|---|
| Rocky | `https://dl.rockylinux.org/pub/rocky/10/images/x86_64/Rocky-10-GenericCloud-Base.x86_64.qcow2` |
| AlmaLinux | `https://repo.almalinux.org/almalinux/10/cloud/x86_64/AlmaLinux-10-GenericCloud-latest.x86_64.qcow2` |
| Ubuntu | `https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img` |
| Debian | `https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2` |
| openSUSE Leap 15.6 | `https://download.opensuse.org/repositories/Cloud:/Images:/Leap_15.6/images/openSUSE-Leap-15.6.x86_64-NoCloud.qcow2` |
| FreeBSD 15.1 | `https://download.freebsd.org/releases/VM-IMAGES/15.1-RELEASE/amd64/Latest/FreeBSD-15.1-RELEASE-amd64-BASIC-CLOUDINIT-zfs.qcow2.xz` |

Образы, сжатые `.xz` (FreeBSD), распаковываются автоматически при первом запуске (нужен пакет `xz` на хосте); сжатый оригинал сохраняется и переиспользуется между запусками VM.

## Разный user-data для разных ОС

Вариант `user-data` выбирается по `cloud_os_effective` (в `cloud/defaults/main.yml`):

### Linux (rhel / ubuntu / debian / opensuse)

- `shell: /bin/bash`, `sudo: ALL=(ALL) NOPASSWD:ALL`;
- статическая сеть — в формате netplan (`network: version: 2`) для Ubuntu/Debian/RHEL;
- для openSUSE статическая сеть настраивается best-effort через `ifcfg`/`wicked` (в `runcmd`), а также ставится `sudo` через `zypper`, если его нет в образе.

### FreeBSD

- `shell: /bin/sh` — bash в базовой системе отсутствует;
- директива `sudo:` не используется: sudo ставится best-effort через `pkg install -y sudo`, а `sudoers` пишется в `/usr/local/etc/sudoers.d/` (правильный путь для FreeBSD);
- статическая сеть (`vm_ip_mode: static`) настраивается через `sysrc`/`rc.conf` и `service netif restart`;
- по умолчанию используется DHCP.

## Переменные

Общие из [`../vars.yml`](../vars.yml): `vm_name`, `vm_memory_mb`, `vm_vcpus`, `ssh_pub_key`, `vm_password_hash`, `vm_user`, `vm_network`, `vm_network_mode`, `vm_ip_mode`, `vm_ip`, `vm_gateway`, `vm_netmask`, `vm_prefix`, `os_variant`, `timezone`.

Специфичные для роли:

| Переменная | Описание |
|---|---|
| `cloud_image` | Путь или URL cloud-образа qcow2; URL качается в `images_dir` и переиспользуется |
| `cloud_os` | ОС внутри образа (`rhel` / `ubuntu` / `debian` / `opensuse` / `freebsd`); по умолчанию определяется по имени `cloud_image` |
| `vm_disk_size` | Необязателен: без него overlay наследует размер базового образа |

## Что получается в итоге

Сервер без GUI, пользователь с sudo NOPASSWD (для FreeBSD — если удалось поставить пакет), вход по SSH-ключу и паролю. VM готова через ~минуту после старта.

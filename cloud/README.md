# Роль cloud — готовые cloud-образы (без установки)

Запуск **официальных cloud-образов** любого из трёх дистрибутивов (rhel / ubuntu / debian) — самый быстрый способ: секунды вместо 10+ минут установки с ISO.

## Механизм работы

Cloud-образ — это **qcow2 с уже установленной, настроенной минимальной системой**. Инсталлятора в нём нет вообще: VM просто загружается в готовую ОС. Кто задаёт пользователя/ключ, если ОС уже установлена? — **cloud-init**, стандартный агент первого запуска, встроенный во все cloud-образы.

Порядок:

1. **Источник образа** — `cloud_image`: либо URL (скачивается в `images_dir` при первом запуске и переиспользуется между запусками), либо локальный путь к qcow2. Значение задаётся в `vars.yml`, при запуске можно оставить (Enter) или ввести другое.
2. **Overlay-диск**: `qemu-img create -b <базовый образ>` — VM пишет только в overlay, базовый образ остаётся неизменным и переиспользуется другими VM. `vm_disk_size` опционален (иначе наследует размер образа).
3. **Seed-ISO**: `user-data`/`meta-data` (NoCloud-датасорс cloud-init) на мини-ISO с меткой **cidata** — пользователь, sudo NOPASSWD, хеш пароля, SSH-ключ, hostname, timezone.
4. **`virt-install --import`**: режим «образ уже готов» — без `--location` (не нужно извлекать kernel/initrd инсталлятора) и без `--extra-args`. `--wait` не нужен: virt-install стартует VM и сразу выходит.
5. При первой загрузке cloud-init читает cidata-диск и применяет `user-data`. Seed извлекается **после** первой загрузки — это делает `create_vm.yml` в пост-обработке (после получения IP), т.к. NoCloud-датасорс читается один раз.

## Где брать образы

| Дистрибутив | URL |
|---|---|
| Rocky | `https://dl.rockylinux.org/pub/rocky/10/images/x86_64/Rocky-10-GenericCloud-Base.x86_64.qcow2` |
| AlmaLinux | `https://repo.almalinux.org/almalinux/10/cloud/x86_64/AlmaLinux-10-GenericCloud-latest.x86_64.qcow2` |
| Ubuntu | `https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img` |
| Debian | `https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2` |
| FreeBSD | `https://download.freebsd.org/releases/VM-IMAGES/14.3-RELEASE/amd64/Latest/freebsd-14.3-RELEASE-amd64-BASIC-CLOUDINIT-zfs.qcow2.xz` |

Образы, сжатые `.xz` (FreeBSD), распаковываются автоматически при первом
запуске (нужен пакет `xz` на хосте); сжатый оригинал сохраняется и
переиспользуется между запусками VM.

## FreeBSD: отличия и ограничения

В образах `BASIC-CLOUDINIT` есть cloud-init с поддержкой NoCloud-датасорса
(наш seed-ISO с меткой `cidata` он читает), поэтому hostname, пользователь и
SSH-ключ применяются штатно. Но user-data по умолчанию писался под Linux,
для FreeBSD стоит учитывать:

- `shell: /bin/bash` не сработает (bash в базовой системе нет) — замените на
  `/bin/sh` или уберите вовсе;
- `sudo` в базовой FreeBSD не установлен — строка `sudo:` в user-data
  безвредна (файл просто будет создан), но sudo-права появятся только после
  `pkg install sudo`; вход по SSH-ключу работает и без этого.

`os_variant: generic` подходит; для лучшей совместимости можно попробовать
`osinfo-query os | grep freebsd` и указать конкретный вариант.

## Переменные

Общие из [`../vars.yml`](../vars.yml): `vm_name`, `vm_memory_mb`, `vm_vcpus`, `ssh_pub_key`, `vm_password_hash`, `vm_user`, `vm_network`, `os_variant`, `timezone`.

Специфичные для роли:

| Переменная | Описание |
|---|---|
| `cloud_image` | Путь или URL cloud-образа qcow2; URL качается в `images_dir` и переиспользуется |
| `vm_disk_size` | Необязателен: без него overlay наследует размер базового образа |

## Что получается в итоге

Сервер без GUI, пользователь с sudo NOPASSWD, вход по SSH-ключу и паролю. VM готова через ~минуту после старта.

# Роль rhel — Kickstart (Anaconda)

Unattended-установка **RHEL-семейства с установочного ISO**: Rocky Linux, AlmaLinux, CentOS Stream, RHEL 8–10.

## Механизм установки

Инсталлятор RHEL-семейства — **Anaconda**, его «язык автоматизации» — **Kickstart** (`ks.cfg`): текстовые ответы на все вопросы установщика (локаль, разметка, пользователь, пакеты, post-скрипты).

Ключевая особенность: Anaconda при старте **сама** ищет подключённый диск с меткой тома **OEMDRV** и, если находит, читает с него `ks.cfg`. Поэтому:

1. Роль генерирует `ks.cfg` (пользователь с хешем пароля, SSH-ключ, DHCP, LVM на весь диск, `reboot`, minimal-environment + openssh-server, sudo NOPASSWD в `%post`).
2. Упаковывает его в мини-ISO с volid `OEMDRV` (`genisoimage`).
3. Передаёт факты (`boot_iso`, `kernel_args`, `seed_iso`, `seed_disk`) в общую часть [`install_vm.yml`](../install_vm.yml): `virt-install --location <установочный ISO>` с OEMDRV-ISO вторым cdrom, `--wait -1` — до финального reboot в готовую систему.
4. OEMDRV-ISO извлекается из cdrom и удаляется.

## Требования к образу

**Полный DVD-ISO, не boot.iso/netinstall** — репозиторий пакетов в `ks.cfg` не задаётся, пакеты берутся с самого ISO.

## Переменные

Используются общие переменные из [`../vars.yml`](../vars.yml): `vm_name`, `vm_memory_mb`, `vm_vcpus`, `vm_disk_size`, `iso_path`, `ssh_pub_key`, `vm_password_hash`, `vm_user`, `vm_network`, `os_variant`, `timezone`.

## Что получается в итоге

Сервер без GUI, root залочен, пользователь (по умолчанию `admin`) в `wheel` с sudo NOPASSWD, вход по SSH-ключу и паролю, LVM на весь диск, openssh-server.

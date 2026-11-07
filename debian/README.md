# Роль debian — preseed (debian-installer)

Unattended-установка **Debian 11/12 (netinst / DVD)**.

## Механизм установки

Инсталлятор Debian — **debian-installer (d-i)**, его режим автоматизации — **preseed**: файл с ответами на вопросы d-i в формате `d-i вопрос тип значение`.

Особенность Debian: d-i умеет читать preseed прямо с установочного диска (`file=/cdrom/preseed.cfg`), но добавить файл в оригинальный ISO нельзя — поэтому **копия установочного образа ремастится через `xorriso`**:

1. Роль генерирует `preseed.cfg` (локаль, сеть, LVM на весь диск, пользователь с хешем пароля, пакеты только с ISO — зеркало отключено) и `postinst.sh`.
2. `xorriso -indev <ISO> -outdev <копия> -map preseed.cfg /preseed.cfg -map postinst.sh /postinst.sh -boot_image any replay` — в копию ISO дописываются оба файла, загрузочные записи сохраняются (`replay`). Оригинальный ISO не трогается.
3. `postinst.sh` выполняется d-i в конце установки (`preseed/late_command`): раскладывает SSH-ключ и sudo NOPASSWD в `/target` (только что установленная система).
4. Передаёт факты в общую часть [`install_vm.yml`](../install_vm.yml): грузимся с **ремастер-копии**, вторая болванка с конфигом не нужна (`seed_iso: ''`).

## Почему locale и keymap продублированы в boot-аргументах

Preseed с `file=/cdrom/preseed.cfg` d-i загружает только **после монтирования установочного диска**, а вопросы «Select a language» и раскладки задаются **до** этого. Без дублей в boot-аргументах установка молча ждёт Enter на первом вопросе — в headless-режиме (`--graphics none`, `--noautoconsole`) за консолью некому следить, и кажется, что инсталлятор завис (лечение симптома «открыть virt-manager и что-то нажать»).

Поэтому в `kernel_args` добавлены cmdline-преобразованные ответы (boot-параметр вида `вопрос=значение` — это preseeding, применяемый **до любого вопроса**):

```
debian-installer/locale=en_US.UTF-8 keyboard-configuration/xkb-keymap=us
```

После этого установка стартует и проходит без единого ручного вмешательства.

## Требования к образу

netinst или DVD — пакеты ставятся с самого ISO, сетевое зеркало не используется.

## Переменные

Используются общие переменные из [`../vars.yml`](../vars.yml): `vm_name`, `vm_memory_mb`, `vm_vcpus`, `vm_disk_size`, `iso_path`, `ssh_pub_key`, `vm_password_hash`, `vm_user`, `vm_network`, `os_variant`, `timezone`.

## Что получается в итоге

Сервер без GUI (только standard-набор + openssh-server + sudo), root залочен, пользователь с sudo NOPASSWD, вход по SSH-ключу, LVM на весь диск.

# Роль ubuntu — autoinstall (Subiquity)

Unattended-установка **Ubuntu Server 20.04+ с live-server ISO**.

## Механизм установки

Инсталлятор Ubuntu Server — **Subiquity**, его режим автоматизации — **autoinstall**: YAML-конфиг в формате cloud-init со всеми ответами (локаль, пользователь, разметка, late-commands).

1. Роль генерирует два файла (стандартный NoCloud-датасорс cloud-init):
   - `user-data` — собственно autoinstall: hostname, пользователь с хешем пароля и SSH-ключом, LVM-разметка, `late-commands` (sudo NOPASSWD) и **обязательный `shutdown: reboot`** — без него Subiquity после установки выключит VM, и некому будет обработать финальную перезагрузку;
   - `meta-data` — instance-id/hostname (для cloud-init формальность).
2. Упаковывает их в мини-ISO с volid **cidata** (`genisoimage`).
3. Чтобы Subiquity вообще включил autoinstall-режим, через boot-аргументы передаётся `autoinstall ds=nocloud` — так он ищет конфиг на cidata-диске.
4. Передаёт факты в общую часть [`install_vm.yml`](../install_vm.yml): `virt-install --location <live ISO>` с cidata-ISO вторым cdrom, `--wait -1`.
5. cidata-ISO извлекается из cdrom и удаляется.

## Требования к образу

Обычный **live-server ISO**, всё оффлайн с диска.

**Не поддерживается**: Ubuntu Desktop 20.04–22.04 (старый инсталлятор Ubiquity без autoinstall). Desktop 23.04+ работает — там уже Subiquity.

## Переменные

Используются общие переменные из [`../vars.yml`](../vars.yml): `vm_name`, `vm_memory_mb`, `vm_vcpus`, `vm_disk_size`, `iso_path`, `ssh_pub_key`, `vm_password_hash`, `vm_user`, `vm_network`, `os_variant`, `timezone`.

## Что получается в итоге

Сервер без GUI, root залочен, пользователь с sudo NOPASSWD, вход по SSH-ключу и паролю, LVM на весь диск, openssh-server.

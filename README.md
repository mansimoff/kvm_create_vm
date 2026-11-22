# kvm-create-vm

Ansible-проект для быстрого создания тестовых VM на локальной машине KVM/libvirt: либо unattended-установка с ISO, либо запуск готового cloud-образа — в обоих случаях получаете VM с вашим пользователем и SSH-доступом по ключу, без единого клика в virt-manager.

```
create_vm.yml     — плейбук: проверки, подтверждение, вызов роли, IP + /etc/hosts
install_vm.yml    — библиотека тасков (диск + virt-install), включается ролями rhel/ubuntu/debian
network.yml       — библиотека тасков: проверка/создание libvirt-сети (NAT или мост)
ansible.cfg       — inventory и roles_path (роли лежат в корне проекта)
inventory.yml     — localhost
vars.yml          — параметры конкретной VM (включая os_family и сеть)
rhel/             — роль: Kickstart (Rocky / Alma / CentOS Stream / RHEL)
ubuntu/           — роль: autoinstall (Ubuntu Server 20.04+)
debian/           — роль: preseed (Debian 11/12)
cloud/            — роль: готовые cloud-образы qcow2 (любой из трёх дистрибутивов)
```

Каждая роль — только `README.md` (подробно про её механизм установки) и `tasks/main.yml`.

## Два режима работы

Режим выбирается переменной **`os_family`** в `vars.yml`:

### 1. Установка с ISO (`rhel` / `ubuntu` / `debian`)

VM ставится с обычного установочного ISO без интерактивных вопросов. Способ передачи конфигурации зависит от инсталлятора:

| os_family | Инсталлятор | Конфиг | Как попадает в инсталлятор |
|---|---|---|---|
| `rhel` | Anaconda | `ks.cfg` (Kickstart) | отдельный ISO с меткой **OEMDRV** — Anaconda находит его сама |
| `ubuntu` | Subiquity | `user-data`/`meta-data` (autoinstall) | отдельный ISO с меткой **cidata** + boot-аргумент `autoinstall ds=nocloud` |
| `debian` | debian-installer | `preseed.cfg` + `postinst.sh` | **remaster копии ISO через xorriso** — preseed вшивается прямо внутрь образа, boot-аргумент `file=/cdrom/preseed.cfg` |

Общий код установки (создание qcow2-диска, `virt-install --location --wait -1`, извлечение seed-ISO) вынесен в [`install_vm.yml`](install_vm.yml) и включается каждой ролью — он ровно один на все три семейства.

### 2. Cloud-образ (`cloud`) — без установки, ~минута

Берётся готовый qcow2 cloud-образ от вендора (у него нет инсталлятора — ОС уже установлена), создаётся overlay-диск поверх него, а пользователь/SSH-ключ/пароль применяет **cloud-init** через seed-ISO с меткой cidata при первой загрузке. VM стартует через `virt-install --import`.

```yaml
# в vars.yml
os_family: cloud
cloud_image: https://dl.rockylinux.org/pub/rocky/10/images/x86_64/Rocky-10-GenericCloud-Base.x86_64.qcow2
# или локальный файл:
# cloud_image: /var/lib/libvirt/images/Rocky-10-GenericCloud-Base.x86_64.qcow2
```

Подробности в [`cloud/README.md`](cloud/README.md).

## Настройка сети

Сеть описывается переменными в `vars.yml`. По умолчанию всё работает как раньше: NAT-сеть libvirt `default`, адрес по DHCP, устройство `virtio`.

### Режим сети (`vm_network_mode`)

| Режим | Что делает |
|---|---|
| `nat` (по умолчанию) | NAT-сеть libvirt. Если сети `vm_network` ещё нет, она создаётся с параметрами `vm_network_gateway`, `vm_netmask` и DHCP-диапазоном `vm_network_dhcp_start..vm_network_dhcp_end`. Если сеть уже существует — используется как есть. |
| `bridge` | Создаётся libvirt-сеть поверх **существующего моста хоста** `vm_bridge` (например, `br0`). Сам мост нужно создать заранее средствами ОС (NetworkManager / systemd-networkd / `brctl`) — плейбук создаёт только libvirt-описание. |

### IP внутри гостя (`vm_ip_mode`)

| Режим | Что делает |
|---|---|
| `dhcp` (по умолчанию) | Гость получает адрес по DHCP: для NAT — от dnsmasq libvirt, для `bridge` — от внешнего DHCP-сервера в мосту. |
| `static` | Адрес задаётся вручную: `vm_ip` (обязателен), `vm_gateway` (по умолчанию `vm_network_gateway`), `vm_netmask` (по умолчанию `/24`). Для `bridge` шлюз нужно указать явно. |

Статический адрес прописывается в unattended-конфиг соответствующей роли: `network --bootproto=static` (Kickstart), `network:` в autoinstall (Ubuntu), `netcfg/get_ipaddress` (preseed), `network:` в cloud-init (cloud).

Примеры:

```yaml
# NAT + DHCP (по умолчанию, как раньше)
vm_network_mode: nat
vm_network: default
vm_ip_mode: dhcp

# NAT + статический IP .50 в сети 192.168.122.0/24
vm_network_mode: nat
vm_network: default
vm_ip_mode: static
vm_ip: 192.168.122.50
vm_netmask: 255.255.255.0
vm_prefix: 24
vm_gateway: 192.168.122.1

# Сеть на базе существующего моста br0 + DHCP из внешней сети
vm_network_mode: bridge
vm_network: br0-net
vm_bridge: br0
vm_ip_mode: dhcp
```

> Параметры NAT-сети применяются только при её **создании**. Если сеть с таким именем уже существует, она не изменяется. Чтобы применить новые значения, удалите её: `virsh net-destroy <имя> && virsh net-undefine <имя>`.
>
> При статическом IP в NAT-сети выбирайте адрес вне DHCP-диапазона (`vm_network_dhcp_start..end`), иначе возможен конфликт. `vm_prefix` должен соответствовать `vm_netmask`.

## Как это работает (порядок выполнения)

1. **Проверки** (до каких-либо действий):
   - обязательные переменные заполнены; параметры сети корректны;
   - **RAM и vCPU VM не больше ресурсов хоста** (`nproc`, `free -m`);
   - **ISO соответствует `os_family`** — дистрибутив определяется по маркерам внутри образа через `isoinfo` (`/.discinfo` → rhel, `/.disk/info` → ubuntu/debian): нельзя запустить «rhel» с образом ubuntu;
   - VM с таким именем ещё не существует.
2. **Путь до образа** — `iso_path` (или `cloud_image` для cloud) задаётся в `vars.yml`, но при запуске плейбук показывает его и позволяет изменить интерактивно (Enter — оставить; удобно, если забыли поправить файл).
3. Печатается конфигурация и запрашивается **подтверждение (yes/no)**.
4. Проверяется/создаётся libvirt-сеть: NAT с параметрами из `vars.yml` либо сеть на базе моста (`network.yml`).
5. Запускается роль по `os_family` — она готовит unattended-конфиги, собирает seed-ISO и выполняет установку/запуск VM.
6. Получение IP (DHCP-аренда, ARP для моста либо заданный вручную адрес) → запись в `/etc/hosts` → итог: `ssh <user>@<vm_name>`.

## Поддерживаемые ОС

| ОС | os_family | ISO | Cloud |
|---|---|---|---|
| Rocky / Alma / CentOS Stream / RHEL 8–10 | `rhel` | ✅ DVD-ISO (не netinstall) | ✅ GenericCloud |
| Ubuntu Server 20.04 / 22.04 / 24.04 | `ubuntu` | ✅ live-server | ✅ cloudimg |
| Ubuntu Desktop 23.04+ | `ubuntu` | ✅ (Subiquity) | — |
| Ubuntu Desktop 20.04–22.04 | — | ❌ (Ubiquity без autoinstall) | — |
| Debian 11 / 12 (netinst, DVD) | `debian` | ✅ | ✅ genericcloud |

Во всех сценариях: серверные системы без GUI, root залочен, доступ через sudo-пользователя по SSH-ключу (и по паролю), LVM на весь диск.

## Требования

На хосте:

```bash
# RHEL-семейство (хост-гипервизор)
sudo dnf install -y qemu-kvm libvirt virt-install genisoimage

# Debian/Ubuntu-хост (гипервизор)
sudo apt install -y qemu-kvm libvirt-daemon-system virtinst genisoimage

# xorriso — только для os_family: debian (remaster ISO с preseed)
sudo dnf install -y xorriso   # или: sudo apt install -y xorriso
```

Пользователь — в группе `libvirt`, либо запуск с `-K` (спросит sudo-пароль).

## Использование

```bash
# отредактировать vars.yml: os_family, имя, cpu, ram, диск, ssh-ключ, хеш пароля
openssl passwd -6 'мойпароль'  # → вставить в vm_password_hash

ansible-playbook create_vm.yml -K
```

`-i inventory.yml` указывать не нужно — он прописан в [`ansible.cfg`](ansible.cfg).
При запуске плейбук показывает путь до образа (`iso_path` / `cloud_image`) из `vars.yml` — Enter оставить его, либо введите другой вручную. Разово задать без правки файла можно и через `-e iso_path=...`.

Разово переопределить что-то можно через `-e`:

```bash
ansible-playbook create_vm.yml -K -e vm_name=other-test -e os_family=debian
```

### Несколько VM параллельно

Запускайте в разных терминалах — единственное условие: разный `vm_name` (файлы дисков/ISO именуются по нему). Держите отдельный vars-файл на сценарий (`vars-web.yml`, `vars-db.yml`) и указывайте через `-e @vars-web.yml`.

### Удаление VM

```bash
sudo virsh destroy test-vm
sudo virsh undefine test-vm --nvram --remove-all-storage
sudo sed -i '/\stest-vm$/d' /etc/hosts
```

## Переменные (vars.yml)

| Переменная | Обязательна | По умолчанию | Описание |
|---|---|---|---|
| `os_family` | да | — | `rhel` / `ubuntu` / `debian` — установка с ISO; `cloud` — готовый cloud-образ |
| `vm_name` | да | — | Имя VM в libvirt и hostname внутри гостя |
| `vm_memory_mb` | да | — | ОЗУ, МБ (не больше RAM хоста — проверяется) |
| `vm_vcpus` | да | — | Число vCPU (не больше CPU хоста — проверяется) |
| `vm_disk_size` | да (кроме cloud) | — | Размер диска (`20G`); для cloud опционально — иначе размер образа |
| `iso_path` | да (кроме cloud) | — | Путь к установочному ISO; при запуске можно оставить (Enter) или ввести другой |
| `ssh_pub_key` | да | — | Публичный SSH-ключ |
| `vm_password_hash` | да | — | Хеш пароля (`openssl passwd -6`) |
| `cloud_image` | для cloud | — | Путь или URL cloud-образа qcow2; при запуске можно оставить (Enter) или ввести другой |
| `vm_user` | нет | `admin` | Пользователь внутри VM, sudo NOPASSWD (+`wheel` для rhel) |
| `vm_network_mode` | нет | `nat` | `nat` — NAT-сеть libvirt; `bridge` — сеть на базе моста хоста |
| `vm_network` | нет | `default` | Имя libvirt-сети |
| `vm_network_gateway` | нет | `192.168.122.1` | Адрес шлюза NAT-сети (при её создании) |
| `vm_netmask` | нет | `255.255.255.0` | Маска сети (`/24`) |
| `vm_prefix` | нет | `24` | Префикс маски для netplan/cloud-init (должен соответствовать `vm_netmask`) |
| `vm_network_dhcp_start` / `vm_network_dhcp_end` | нет | `.100` / `.254` | DHCP-диапазон NAT-сети (при её создании) |
| `vm_bridge` | для `bridge` | `br0` | Имя существующего моста хоста |
| `vm_ip_mode` | нет | `dhcp` | `dhcp` — адрес по DHCP; `static` — задать вручную |
| `vm_ip` | для `static` | — | Статический IP гостя |
| `vm_gateway` | для `static`+`bridge` | `vm_network_gateway` | Шлюз гостя |
| `vm_dns` | нет | `vm_gateway` | DNS-сервер гостя |
| `os_variant` | нет | `generic` | `virt-install --os-variant`, см. `osinfo-query os` |
| `timezone` | нет | `UTC` | Таймзона гостя |
| `images_dir` | нет | `/var/lib/libvirt/images` | Каталог для дисков и ISO |
| `workdir` | нет | `/tmp/kvm-create-vm` | Временная директория для генерируемых конфигов |

## Известные ограничения

- Root внутри VM залочен во всех сценариях — доступ только через sudo-пользователя.
- Для `rhel` `%packages` жёстко использует `@^minimal-environment` (имя группы специфично для comps.xml RHEL-семейства).
- Для `debian` postinst.sh записывает SSH-ключ по uid/gid `1000` — стандартный uid первого пользователя d-i.
- Параметры NAT-сети применяются только при создании новой сети; существующая сеть не изменяется.
- Для `vm_network_mode: bridge` мост должен существовать заранее; при статическом IP шлюз (`vm_gateway`) нужно указать явно.
- Статический IP в NAT-сети должен быть вне DHCP-диапазона libvirt.
- В cloud-режиме статическая сеть задаётся в формате netplan (Linux); для FreeBSD нужна отдельная настройка.
- IPv4-only парсинг адреса из `virsh domifaddr` / `virsh net-dhcp-leases`.

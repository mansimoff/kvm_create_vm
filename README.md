# kvm-create-vm

Ansible-проект для быстрого создания тестовых VM на локальной машине KVM/libvirt: либо unattended-установка с ISO, либо запуск готового cloud-образа — в обоих случаях получаете VM с вашим пользователем и SSH-доступом по ключу, без единого клика в virt-manager.

```
create_vm.yml     — плейбук: проверки, подтверждение, вызов роли, IP + /etc/hosts
install_vm.yml    — библиотека тасков (диск + virt-install), включается ISO-ролями
network.yml       — библиотека тасков: проверка/создание libvirt-сети (NAT или мост)
ansible.cfg       — inventory и roles_path (роли лежат в корне проекта)
inventory.yml     — localhost
vars.yml          — параметры конкретной VM (os_family, install_mode, сеть)
rhel/             — роль: Kickstart (Rocky / Alma / CentOS Stream / RHEL)
ubuntu/           — роль: autoinstall (Ubuntu Server 20.04+)
debian/           — роль: preseed (Debian 11/12)
freebsd/          — роль: bsdinstall script (FreeBSD 13/14/15)
opensuse/         — роль: AutoYaST (openSUSE Leap / Tumbleweed)
cloud/            — роль: готовые cloud-образы qcow2 (Linux и FreeBSD)
```

Каждая ISO-роль — только `README.md` (подробно про её механизм установки) и `tasks/main.yml`; у `cloud` дополнительно есть `defaults/main.yml` с вариантами user-data.

## Быстрый старт 

### Шаг 1. Установите пакеты (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients \
  virtinst genisoimage xorriso xz-utils ansible python3
```

Для RHEL/Alma/Rocky — см. раздел [Настройка хоста](#настройка-хоста-подробно).

### Шаг 2. Разрешите пользователю управлять libvirt

```bash
sudo usermod -aG libvirt,kvm "$USER"
```

Если менять группы не хотите — просто запускайте команду с `-K` (см. шаг 4).

### Шаг 3. Заполните `vars.yml`

ISO нужно скачать заранее с официального сайта дистрибутива (например, в `~/iso/`). Затем откройте [`vars.yml`](vars.yml) в любом редакторе и задайте:

| Параметр | Что это | Пример |
|---|---|---|
| `os_family` | какую ОС ставим | `debian`, `ubuntu`, `rhel`, `freebsd`, `opensuse` |
| `vm_name` | имя VM и hostname | `test-vm` |
| `vm_memory_mb` | ОЗУ в МБ | `2048` |
| `vm_vcpus` | число ядер | `2` |
| `vm_disk_size` | размер диска | `20G` |
| `iso_path` | путь к установочному ISO | `/home/user/iso/debian-12-netinst.iso` |
| `ssh_pub_key` | ваш публичный SSH-ключ | `ssh-ed25519 AAAA... user@host` |
| `vm_password_hash` | хеш пароля (не сам пароль!) | см. ниже |

**Хеш пароля:**

```bash
openssl passwd -6 'мойпароль'
```

Скопируйте полученную строку (она начинается с `$6$`) в `vm_password_hash`.

**SSH-ключ:**

```bash
cat ~/.ssh/id_ed25519.pub
```

Если ключа нет:

```bash
ssh-keygen -t ed25519
```

Скопируйте строку целиком в `ssh_pub_key`.

### Шаг 4. Запустите создание VM

```bash
ansible-playbook create_vm.yml -K
```

Потому что в ansible.cfg есть inventory = inventory.yml, но можно задать через флаг -i


Что будет происходить:

1. `-K` попросит sudo-пароль (нужен для libvirt и `/etc/hosts`).
2. Плейбук покажет путь к ISO из `vars.yml` — нажмите **Enter**, чтобы оставить его.
3. Плейбук покажет конфигурацию и спросит подтверждение — введите **`yes`**.
4. Дальше всё автоматически: установка, получение IP, запись в `/etc/hosts`.

### Шаг 5. Подключитесь к VM

```bash
ssh <vm_user>@<vm_name>
# например: ssh ruslan@test-vm
```

`vm_user` по умолчанию — `admin` (или тот, что вы указали в `vars.yml`).

### Если нужен cloud-образ (без установки, ~1 минута)

В `vars.yml`:

```yaml
os_family: debian          # или freebsd / opensuse / rhel / ubuntu
install_mode: cloud
cloud_image: https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2
```

Запустите ту же команду `ansible-playbook create_vm.yml -K`. ISO не нужен.

### Как удалить VM

```bash
sudo virsh destroy <vm_name>
sudo virsh undefine <vm_name> --nvram --remove-all-storage
sudo sed -i '/\s<vm_name>$/d' /etc/hosts
```

Подробности и тонкости — в разделах ниже.

## Два режима работы

Режим выбирается переменной **`install_mode`** (`iso` по умолчанию / `cloud`), а семейство ОС — **`os_family`** (`rhel` / `ubuntu` / `debian` / `freebsd` / `opensuse`). Для совместимости `os_family: cloud` по-прежнему означает cloud-режим.

### 1. Установка с ISO (`install_mode: iso`)

VM ставится с обычного установочного ISO без интерактивных вопросов. Способ передачи конфигурации зависит от инсталлятора:

| os_family | Инсталлятор | Конфиг | Как попадает в инсталлятор |
|---|---|---|---|
| `rhel` | Anaconda | `ks.cfg` (Kickstart) | отдельный ISO с меткой **OEMDRV** — Anaconda находит его сама |
| `ubuntu` | Subiquity | `user-data`/`meta-data` (autoinstall) | отдельный ISO с меткой **cidata** + boot-аргумент `autoinstall ds=nocloud` |
| `debian` | debian-installer | `preseed.cfg` + `postinst.sh` | **remaster копии ISO через xorriso** — preseed вшивается прямо внутрь образа, boot-аргумент `file=/cdrom/preseed.cfg` |
| `freebsd` | bsdinstall | `installerconfig` (script) | **remaster копии ISO через xorriso** — `/etc/installerconfig` вшивается внутрь, bsdinstall запускает его сам; VM грузится с ISO как с CD-ROM (`--cdrom`) |
| `opensuse` | AutoYaST | `autoinst.xml` | отдельный ISO с меткой **OEMDRV** + boot-аргумент `autoyast=label://OEMDRV/autoinst.xml` |

Общий код установки (создание qcow2-диска, `virt-install` с `--location` или `--cdrom`, извлечение seed-ISO) вынесен в [`install_vm.yml`](install_vm.yml) и включается каждой ролью — он ровно один на все семейства.

### 2. Cloud-образ (`install_mode: cloud`) — без установки, ~минута

Берётся готовый qcow2 cloud-образ от вендора (у него нет инсталлятора — ОС уже установлена), создаётся overlay-диск поверх него, а пользователь/SSH-ключ/пароль применяет **cloud-init** через seed-ISO с меткой cidata при первой загрузке. VM стартует через `virt-install --import`.

```yaml
# в vars.yml
os_family: debian          # или freebsd / opensuse / rhel / ubuntu
install_mode: cloud
cloud_image: https://download.opensuse.org/repositories/Cloud:/Images:/Leap_15.6/images/openSUSE-Leap-15.6.x86_64-NoCloud.qcow2
# или локальный файл:
# cloud_image: /var/lib/libvirt/images/openSUSE-Leap-15.6.x86_64-NoCloud.qcow2
```

ОС внутри cloud-образа определяется автоматически по имени `cloud_image` (freebsd / opensuse / debian / ubuntu / rhel) либо задаётся явно через `cloud_os`. Для FreeBSD используется отдельный user-data (`/bin/sh`, sudo через `pkg`), для Linux — bash + sudo NOPASSWD.

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

Статический адрес прописывается в unattended-конфиг соответствующей роли: `network --bootproto=static` (Kickstart), `network:` в autoinstall (Ubuntu), `netcfg/get_ipaddress` (preseed), `ifcfg=eth*=` (linuxrc/AutoYaST), `sysrc ifconfig_*`/rc.conf (FreeBSD), `network:` в cloud-init (cloud).

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
   - **ISO соответствует `os_family`** — дистрибутив определяется через `isoinfo`: по метке тома (`FreeBSD-...` → freebsd, `openSUSE-...` → opensuse) либо по маркерам внутри образа (`/.discinfo` → rhel, `/.disk/info` → ubuntu/debian): нельзя запустить «rhel» с образом ubuntu;
   - VM с таким именем ещё не существует.
2. **Путь до образа** — `iso_path` (или `cloud_image` для cloud) задаётся в `vars.yml`, но при запуске плейбук показывает его и позволяет изменить интерактивно (Enter — оставить; удобно, если забыли поправить файл).
3. Печатается конфигурация и запрашивается **подтверждение (yes/no)**.
4. Проверяется/создаётся libvirt-сеть: NAT с параметрами из `vars.yml` либо сеть на базе моста (`network.yml`).
5. Запускается роль по `os_family` (в cloud-режиме — общая роль `cloud`) — она готовит unattended-конфиги, собирает seed-ISO и выполняет установку/запуск VM.
6. Получение IP (DHCP-аренда, ARP для моста либо заданный вручную адрес) → запись в `/etc/hosts` → итог: `ssh <user>@<vm_name>`.

## Поддерживаемые ОС

| ОС | os_family | ISO | Cloud |
|---|---|---|---|
| Rocky / Alma / CentOS Stream / RHEL 8–10 | `rhel` | ✅ DVD-ISO (не netinstall) | ✅ GenericCloud |
| Ubuntu Server 20.04 / 22.04 / 24.04 | `ubuntu` | ✅ live-server | ✅ cloudimg |
| Ubuntu Desktop 23.04+ | `ubuntu` | ✅ (Subiquity) | — |
| Ubuntu Desktop 20.04–22.04 | — | ❌ (Ubiquity без autoinstall) | — |
| Debian 11 / 12 (netinst, DVD) | `debian` | ✅ | ✅ genericcloud |
| FreeBSD 13 / 14 / 15 (disc1) | `freebsd` | ✅ disc1 (не bootonly) | ✅ BASIC-CLOUDINIT |
| openSUSE Leap 15.x / Tumbleweed (DVD) | `opensuse` | ✅ DVD (не Online) | ✅ NoCloud |

Во всех сценариях: серверные системы без GUI, root залочен, доступ через sudo-пользователя по SSH-ключу (и по паролю). LVM на весь диск — в rhel / ubuntu / debian / opensuse; FreeBSD использует штатную разметку bsdinstall (UFS/ZFS).

## Настройка хоста (подробно)

### Проверенная конфигурация

> **Проверено автором лично.** Все сценарии проекта — установка с ISO и cloud-образы для **rhel / ubuntu / debian / freebsd / opensuse** — запускались и работали на гипервизоре со следующими характеристиками:

| Компонент | Значение |
|---|---|
| ОС хоста | **Ubuntu 24.04.5 LTS**, x86_64 |
| Ядро | Linux `7.0.0-43-generic` |
| CPU | Intel VT-x (`vmx`), модуль `kvm_intel` загружен |
| Ansible | core **2.21.5** (пакет `ansible 14.5.0`) |
| Python | **3.12.3** |
| libvirt / virsh | **10.0.0** |
| virt-install | **4.1.0** |
| QEMU | **8.2.2** |
| xorriso | **1.5.6** |
| genisoimage | **1.1.11** |
| xz | **5.4.5** |
| libvirt-сеть `default` | active, autostart yes |
| Пользователь | в группах `libvirt`, `kvm`, `sudo` |

> Если у вас похожая конфигурация (Ubuntu 24.04 + libvirt 10 + virt-install 4.1), всё должно работать без изменений. На Debian 12+ и RHEL/Alma/Rocky 9+ тоже должно работать — отличаются только команды установки пакетов.

### 1. Аппаратная виртуализация

Нужен x86_64 CPU с **Intel VT-x** или **AMD-V** и включённая в BIOS/UEFI виртуализация.

Проверка:

```bash
grep -m1 -E 'vmx|svm' /proc/cpuinfo     # должно что-то вывести
ls -l /dev/kvm                          # устройство должно существовать
lsmod | grep -E '^kvm'                  # kvm и kvm_intel/kvm_amd
```

Если `/dev/kvm` нет:

- включите **VT-x/AMD-V** в BIOS/UEFI;
- загрузите модуль: `sudo modprobe kvm_intel` (Intel) или `sudo modprobe kvm_amd` (AMD);
- если хост сам является VM — включите **nested virtualization** у гипервизора.

Без `/dev/kvm` VM будут работать очень медленно (или не запустятся).

### 2. Операционная система хоста

- **Ubuntu 24.04** — проверено;
- Debian 12+;
- RHEL / AlmaLinux / Rocky Linux 9+;
- другой Linux с KVM/libvirt — скорее всего, тоже подойдёт.

Проект рассчитан на **локальный** гипервизор (не remote libvirt).
Можно перенести данный проект на удалённый гипервизор с помощью scp и запустить там.

### 3. Установка пакетов

**Ubuntu / Debian:**

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients \
  virtinst genisoimage xorriso xz-utils ansible python3
```

**RHEL / AlmaLinux / Rocky Linux:**

```bash
sudo dnf install -y qemu-kvm libvirt libvirt-client virt-install \
  genisoimage xorriso xz ansible-core python3
```

Минимально необходимы:

- `qemu-kvm`, `libvirt`, `virt-install` — сам гипервизор и создание VM;
- `genisoimage` — сборка seed-ISO для rhel/ubuntu/opensuse;
- `xorriso` — ремастеринг ISO для **debian** и **freebsd**;
- `xz` — распаковка FreeBSD cloud-образа (`.qcow2.xz`);
- `ansible` / `ansible-core` и `python3` — запуск плейбука.

### 4. Запуск libvirtd

```bash
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
```

### 5. Права пользователя

```bash
sudo usermod -aG libvirt,kvm "$USER"
```

Затем **выйдите из системы и войдите снова** (или выполните `newgrp libvirt`). Проверка:

```bash
id
virsh list --all
```

Если `virsh` работает без sudo — всё настроено. Если не хотите менять группы, запускайте плейбук с `-K`:

```bash
ansible-playbook create_vm.yml -K
```

### 6. Сеть libvirt

По умолчанию используется NAT-сеть `default`. Проверьте:

```bash
virsh net-list --all
```

Если сеть не активна:

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

Если сети `default` вообще нет:

```bash
sudo virsh net-define /usr/share/libvirt/networks/default.xml
sudo virsh net-start default
sudo virsh net-autostart default
```

Путь к `default.xml` может отличаться в зависимости от дистрибутива.

### 7. Каталог для дисков и ISO

По умолчанию — `/var/lib/libvirt/images`. Нужно свободное место:

- ISO установщика (обычно 1–5 ГБ);
- диск VM (`vm_disk_size`, например 20 ГБ);
- для **debian** и **freebsd** — ещё одна копия ISO под ремастеринг (столько же, сколько ISO);
- для cloud — размер cloud-образа + overlay-диск.

Проверка:

```bash
df -h /var/lib/libvirt/images
```

Если каталога нет:

```bash
sudo mkdir -p /var/lib/libvirt/images
sudo chown root:libvirt /var/lib/libvirt/images
sudo chmod 775 /var/lib/libvirt/images
```

Можно изменить `images_dir` в `vars.yml`, но тогда убедитесь, что libvirt имеет доступ к этому каталогу (см. SELinux/AppArmor ниже).

### 8. SELinux / AppArmor

- **Ubuntu** (AppArmor): путь `/var/lib/libvirt/images` разрешён по умолчанию. Для ISO в других каталогах может потребоваться настройка профиля AppArmor.
- **RHEL / Alma / Rocky** (SELinux): если ISO лежит вне `/var/lib/libvirt/images`, задайте ему контекст:

```bash
sudo semanage fcontext -a -t virt_image_t "/path/to/iso(/.*)?"
sudo restorecon -Rv /path/to/iso
```

Если `semanage` не установлен:

```bash
sudo dnf install -y policycoreutils-python-utils
```

Либо просто держите ISO в `/var/lib/libvirt/images` — тогда проблем обычно нет.

### 9. Проверка версий утилит

```bash
ansible --version
virt-install --version
virsh --version
qemu-system-x86_64 --version
xorriso -version
genisoimage --version
xz --version
```

Ориентируйтесь на проверенную конфигурацию выше. Более старые версии могут работать, но не гарантируется.

### 10. Финальная проверка готовности

```bash
ls -l /dev/kvm
virsh net-list --all
ansible-playbook --syntax-check -i inventory.yml create_vm.yml
```

Последняя команда должна вывести:

```
playbook: create_vm.yml
```

без ошибок. После этого можно переходить к [быстрому старту](#быстрый-старт-если-вы-не-знаете-ansible) или к разделу [Использование](#использование-подробно).

## Использование (подробно)

### 1. Выберите ISO или cloud-образ

**Установка с ISO** — скачайте ISO нужного дистрибутива (требования к образам — в таблице «Поддерживаемые ОС»):

- `rhel` — полный DVD (не `boot.iso`/netinstall);
- `ubuntu` — live-server ISO;
- `debian` — netinst или DVD;
- `freebsd` — `disc1` ISO (не bootonly);
- `opensuse` — полный DVD (не Online-ISO).

Путь к ISO укажите в `iso_path`.

**Cloud-образ** — возьмите готовый qcow2 (URL или локальный файл) из [`cloud/README.md`](cloud/README.md) и укажите его в `cloud_image`, а также:

```yaml
install_mode: cloud
```

### 2. Заполните `vars.yml`

Минимально нужно задать:

```yaml
os_family: debian
vm_name: test-vm
vm_memory_mb: 2048
vm_vcpus: 2
vm_disk_size: 20G
iso_path: /home/user/iso/debian-12-netinst.iso
ssh_pub_key: "ssh-ed25519 AAAA... user@host"
vm_password_hash: "$6$..."
```

Плюс при необходимости — сеть (`vm_network_mode`, `vm_ip_mode`, `vm_ip`, …). Полная таблица переменных — ниже.

### 3. Запустите плейбук

```bash
ansible-playbook create_vm.yml -K
```

`-K` запросит sudo-пароль. Если пользователь уже в группе `libvirt` (см. настройку хоста), можно без него:

```bash
ansible-playbook create_vm.yml
```

`-i inventory.yml` указывать **не нужно** — он прописан в [`ansible.cfg`](ansible.cfg).

### 4. Что вводить при запуске

1. **«Путь до установочного ISO»** — нажмите **Enter**, чтобы оставить значение из `vars.yml`, либо введите другой путь.
2. **«Продолжить создание VM?»** — введите **`yes`**.
3. В cloud-режиме вместо ISO будет вопрос про `cloud_image` — также Enter или новый путь/URL.

Дальше плейбук всё делает сам: сеть, установка/импорт, получение IP, запись в `/etc/hosts`. В конце он выведет:

```
VM 'test-vm' установлена и запущена.
IP: 192.168.122.100
Подключение: ssh ruslan@test-vm
```

### 5. Подключитесь к VM

```bash
ssh <vm_user>@<vm_name>
```

### Разовое переопределение параметров

Не обязательно править `vars.yml` — можно передать значения через `-e`:

```bash
ansible-playbook create_vm.yml -K \
  -e vm_name=other-test \
  -e os_family=debian \
  -e iso_path=/home/user/iso/debian-12-netinst.iso
```

### Несколько VM параллельно

Запускайте в разных терминалах — единственное условие: **разный `vm_name`** (файлы дисков/ISO именуются по нему). Удобно держать отдельный vars-файл на сценарий (`vars-web.yml`, `vars-db.yml`) и указывать его так:

```bash
ansible-playbook create_vm.yml -K -e @vars-web.yml
```

### Удаление VM

```bash
sudo virsh destroy <vm_name>
sudo virsh undefine <vm_name> --nvram --remove-all-storage
sudo sed -i '/\s<vm_name>$/d' /etc/hosts
```

## Переменные (vars.yml)

| Переменная | Обязательна | По умолчанию | Описание |
|---|---|---|---|
| `os_family` | да | — | `rhel` / `ubuntu` / `debian` / `freebsd` / `opensuse` — семейство ОС; `cloud` — устаревший способ cloud-режима |
| `install_mode` | нет | `iso` | `iso` — установка с ISO; `cloud` — готовый cloud-образ (для `os_family: cloud` игнорируется) |
| `vm_name` | да | — | Имя VM в libvirt и hostname внутри гостя |
| `vm_memory_mb` | да | — | ОЗУ, МБ (не больше RAM хоста — проверяется) |
| `vm_vcpus` | да | — | Число vCPU (не больше CPU хоста — проверяется) |
| `vm_disk_size` | да (кроме cloud) | — | Размер диска (`20G`); для cloud опционально — иначе размер образа |
| `iso_path` | да (кроме cloud) | — | Путь к установочному ISO; при запуске можно оставить (Enter) или ввести другой |
| `ssh_pub_key` | да | — | Публичный SSH-ключ |
| `vm_password_hash` | да | — | Хеш пароля (`openssl passwd -6`) |
| `cloud_image` | для cloud | — | Путь или URL cloud-образа qcow2; при запуске можно оставить (Enter) или ввести другой |
| `cloud_os` | нет | по имени образа | ОС внутри cloud-образа: `rhel` / `ubuntu` / `debian` / `opensuse` / `freebsd` |
| `vm_user` | нет | `admin` | Пользователь внутри VM, sudo NOPASSWD (+`wheel` для rhel/freebsd) |
| `vm_network_mode` | нет | `nat` | `nat` — NAT-сеть libvirt; `bridge` — сеть на базе моста хоста |
| `vm_network` | нет | `default` | Имя libvirt-сети |
| `vm_network_gateway` | нет | `192.168.122.1` | Адрес шлюза NAT-сети (при её создании) |
| `vm_netmask` | нет | `255.255.255.0` | Маска сети (`/24`) |
| `vm_prefix` | нет | `24` | Префикс маски для netplan/cloud-init и linuxrc `ifcfg` (должен соответствовать `vm_netmask`) |
| `vm_network_dhcp_start` / `vm_network_dhcp_end` | нет | `.100` / `.254` | DHCP-диапазон NAT-сети (при её создании) |
| `vm_bridge` | для `bridge` | `br0` | Имя существующего моста хоста |
| `vm_ip_mode` | нет | `dhcp` | `dhcp` — адрес по DHCP; `static` — задать вручную |
| `vm_ip` | для `static` | — | Статический IP гостя |
| `vm_gateway` | для `static`+`bridge` | `vm_network_gateway` | Шлюз гостя |
| `vm_dns` | нет | `vm_gateway` | DNS-сервер гостя |
| `os_variant` | нет | `generic` | `virt-install --os-variant` (`rocky10`, `opensuse-leap15.6`, `freebsd15.0`, …), см. `osinfo-query os` |
| `timezone` | нет | `UTC` | Таймзона гостя |
| `images_dir` | нет | `/var/lib/libvirt/images` | Каталог для дисков и ISO |
| `workdir` | нет | `/tmp/kvm-create-vm` | Временная директория для генерируемых конфигов |

## Частые проблемы и решения

| Симптом | Причина | Решение |
|---|---|---|
| `Permission denied` при `virsh` / Ansible | пользователь не в группах `libvirt`/`kvm` | `sudo usermod -aG libvirt,kvm "$USER"`, перелогиниться; либо запуск с `-K` |
| `failed to connect to the hypervisor` | libvirtd не запущен | `sudo systemctl enable --now libvirtd` |
| `Network default is not active` | сеть libvirt не запущена | `sudo virsh net-start default && sudo virsh net-autostart default` |
| `KVM is not available` / нет `/dev/kvm` | выключена аппаратная виртуализация | включить VT-x/AMD-V в BIOS/UEFI, `modprobe kvm_intel`/`kvm_amd`; если хост сам VM — nested virtualization |
| `ISO не соответствует os_family` | выбран не тот ISO или `os_family` | проверить Volume ID образа, указать правильный `os_family` или ISO |
| Установка «висит» | недоступны зеркала/репозитории | увеличить `vm_install_timeout_min`, для Ubuntu задать `apt_mirror`, проверить сеть |
| `No space left on device` | мало места в `images_dir` | освободить место; для debian/freebsd нужна ещё одна копия ISO под ремастеринг |
| FreeBSD: `sudo: command not found` | sudo не входит в базовую систему | внутри VM: `pkg install sudo`; роль пытается поставить его автоматически |
| openSUSE: нет SSH | firewall или sshd | профиль отключает firewall и включает sshd; проверить `systemctl status sshd` |
| Статический IP не пингуется | адрес в DHCP-диапазоне, неверная маска или bridge без шлюза | выбрать IP вне DHCP, проверить `vm_netmask`/`vm_prefix`, для bridge задать `vm_gateway` |
| VM есть, но IP не находится | DHCP-аренда ещё не появилась или другое имя сети | подождать, проверить `virsh net-dhcp-leases <vm_network>`, `virsh domifaddr <vm_name> --source arp` |
| Ansible: `unsupported locale setting` | локаль хоста не настроена | `export LC_ALL=C.UTF-8 LANG=C.UTF-8` перед запуском |

## Известные ограничения

- Root внутри VM залочен во всех сценариях — доступ только через sudo-пользователя.
- Для `rhel` `%packages` жёстко использует `@^minimal-environment` (имя группы специфично для comps.xml RHEL-семейства).
- Для `debian` postinst.sh записывает SSH-ключ по uid/gid `1000` — стандартный uid первого пользователя d-i.
- Для `freebsd` sudo в базовой системе отсутствует: роль пытается поставить его через `pkg` (нужна сеть), иначе `pkg install sudo` нужно выполнить после первого входа.
- Для `opensuse` используется guided-разметка AutoYaST с LVM; при необходимости тонкой разметки правьте `autoinst.xml` в роли.
- Параметры NAT-сети применяются только при создании новой сети; существующая сеть не изменяется.
- Для `vm_network_mode: bridge` мост должен существовать заранее; при статическом IP шлюз (`vm_gateway`) нужно указать явно.
- Статический IP в NAT-сети должен быть вне DHCP-диапазона libvirt.
- В cloud-режиме статическая сеть: netplan (Ubuntu/Debian/RHEL), `sysrc`/rc.conf (FreeBSD), ifcfg/wicked (openSUSE, best-effort — при необходимости поправьте вручную).
- IPv4-only парсинг адреса из `virsh domifaddr` / `virsh net-dhcp-leases`.

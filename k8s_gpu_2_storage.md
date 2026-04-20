# Production Kubernetes на KVM + Ubuntu 22.04: Пошаговое руководство

## Условные обозначения

```
[ХОСТx]     — выполнить на физическом сервере server-x (RedOS 8)
[ВСЕ ХОСТЫ] — выполнить параллельно на всех 6 физических серверах
[VMx]        — выполнить внутри виртуальной машины vm-x
[ВСЕ VM]    — выполнить параллельно на всех 6 виртуальных машинах
[VM1]        — только на vm-1 (первый control plane)
[CP VM]      — на vm-1, vm-2, vm-3 (control plane)
[GPU VM]     — на vm-1, vm-2, vm-3, vm-4
[STORAGE VM] — на vm-5, vm-6
```

## Топология

```
server-1 (10.10.10.11) → vm-1 (10.10.10.21) — CP + Worker + GPU
server-2 (10.10.10.12) → vm-2 (10.10.10.22) — CP + Worker + GPU
server-3 (10.10.10.13) → vm-3 (10.10.10.23) — CP + Worker + GPU
server-4 (10.10.10.14) → vm-4 (10.10.10.24) — Worker + GPU
server-5 (10.10.10.15) → vm-5 (10.10.10.25) — Worker + Storage
server-6 (10.10.10.16) → vm-6 (10.10.10.26) — Worker + Storage

VIP k8s API: 10.10.10.20 (kube-vip, ARP)
```
## Итоговая ёмкость хранилища

```
Диски на каждом физическом сервере:
  vdb (18.2 TB физический диск)  → /var/lib/longhorn
  vdc (~3.1 TB системный диск)   → /var/lib/longhorn-extra

Суммарная сырая ёмкость:
  6 × 18.2 TB = 109.2 TB  (vdb)
  6 × 3.1 TB  =  18.6 TB  (vdc)
  ────────────────────────
  Итого сырых: ~127.8 TB

Полезная ёмкость с учётом replica=2 и MinIO erasure coding:
  MinIO (4 ноды, erasure 2+2, replica=1): ~16 TB полезных из 32 TB
  MongoDB + Kafka + прочее (replica=2):   ~2 TB полезных из 4 TB
  ────────────────────────────────────────────────────────
  Свободно под данные:   ~91.8 TB
  Занято инфрой:         ~36 TB
  Доступно пользователю: ~55+ TB полезных (с учётом репликации)
```


---

# ЭТАП 0 — Настройка доступа в интернет

> **Выполните ТОЛЬКО один из двух вариантов** в зависимости от вашего окружения.
> После этого шага все остальные команды в документе работают одинаково.

## Вариант А — Прямой доступ в интернет

**[ВСЕ ХОСТЫ]**

```bash
# Проверяем что доступ есть
curl -s --max-time 5 https://google.com -o /dev/null \
  && echo "✓ Прямой доступ работает" \
  || echo "✗ Интернет недоступен — используйте Вариант Б"

# Убеждаемся что переменных прокси нет (они могут мешать)
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY

# Сохраняем пустые значения чтобы не унаследовать чужой прокси
grep -q "http_proxy" /etc/environment || cat >> /etc/environment <<'EOF'
http_proxy=
https_proxy=
HTTP_PROXY=
HTTPS_PROXY=
no_proxy=10.0.0.0/8,192.168.0.0/16,172.16.0.0/12,localhost,127.0.0.1,.svc,.cluster.local
NO_PROXY=10.0.0.0/8,192.168.0.0/16,172.16.0.0/12,localhost,127.0.0.1,.svc,.cluster.local
EOF
```

**[ВСЕ VM]** — после установки Ubuntu (Шаг 3.1 ниже):

```bash
# Проверяем прямой доступ внутри VM
curl -s --max-time 5 https://google.com -o /dev/null \
  && echo "✓ Прямой доступ из VM работает" \
  || echo "✗ Нет доступа"

# apt без прокси — ничего дополнительно настраивать не нужно
apt-get update -y
```

Для команд которые в документе содержат `http_proxy=http://proxy.company.ru:3128 \` в начале — **просто опускайте эту часть**. Например:

```bash
# В документе написано:
# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
  curl -fsSL https://...

# При прямом доступе выполняете просто:
curl -fsSL https://...
```

---

## Вариант Б — Доступ через прокси

**[ВСЕ ХОСТЫ]**

```bash
PROXY="http://proxy.company.ru:3128"   # ← замените на свой адрес
NO_PROXY_LIST="10.0.0.0/8,192.168.0.0/16,172.16.0.0/12,localhost,127.0.0.1,.svc,.cluster.local,10.10.10.0/24"

# Настраиваем прокси для всей системы
grep -q "http_proxy" /etc/environment || cat >> /etc/environment <<EOF
http_proxy=${PROXY}
https_proxy=${PROXY}
HTTP_PROXY=${PROXY}
HTTPS_PROXY=${PROXY}
no_proxy=${NO_PROXY_LIST}
NO_PROXY=${NO_PROXY_LIST}
EOF

# Прокси для dnf (RedOS)
grep -q "^proxy=" /etc/dnf/dnf.conf || echo "proxy=${PROXY}" >> /etc/dnf/dnf.conf

# Применяем в текущей сессии
export http_proxy=$PROXY https_proxy=$PROXY
export no_proxy=$NO_PROXY_LIST NO_PROXY=$NO_PROXY_LIST

# Проверяем
curl -s --max-time 10 https://google.com -o /dev/null \
  && echo "✓ Прокси работает" \
  || echo "✗ Прокси недоступен — проверьте адрес"
```

**[ВСЕ VM]** — после установки Ubuntu (Шаг 3.1 ниже):

```bash
PROXY="http://proxy.company.ru:3128"   # ← тот же адрес
NO_PROXY_LIST="10.0.0.0/8,192.168.0.0/16,172.16.0.0/12,localhost,127.0.0.1,.svc,.cluster.local,10.10.10.0/24"

# Прокси для apt
cat > /etc/apt/apt.conf.d/01proxy <<EOF
Acquire::http::Proxy "${PROXY}";
Acquire::https::Proxy "${PROXY}";
EOF

# Прокси для системы
grep -q "http_proxy" /etc/environment || cat >> /etc/environment <<EOF
http_proxy=${PROXY}
https_proxy=${PROXY}
HTTP_PROXY=${PROXY}
HTTPS_PROXY=${PROXY}
no_proxy=${NO_PROXY_LIST}
NO_PROXY=${NO_PROXY_LIST}
EOF

export http_proxy=$PROXY https_proxy=$PROXY no_proxy=$NO_PROXY_LIST

apt-get update -y && echo "✓ apt через прокси работает"
```

> **Важно для Варианта Б:** прокси нужно прописать в нескольких местах — системные переменные, apt, dnf, containerd (drop-in), kubelet (drop-in), helm. Каждый из этих шагов описан ниже в соответствующих разделах. Не пропускайте блоки с `# [ТОЛЬКО ДЛЯ ПРОКСИ]`.

---

# ЭТАП 1 — Подготовка физических хостов

## Шаг 1.2 — Установка KVM и зависимостей

**[ВСЕ ХОСТЫ]**

```bash
dnf update -y

dnf install -y \
  qemu-kvm \
  libvirt \
  libvirt-daemon-kvm \
  libvirt-client \
  virt-install \
  bridge-utils \
  libguestfs-tools \
  python3-libvirt \
  genisoimage \
  cloud-utils \
  wget \
  curl \
  tar \
  jq \
  net-tools

systemctl enable --now libvirtd

# Проверяем что KVM работает
virt-host-validate qemu
# Все строки должны быть PASS, допустимо WARN для IOMMU (проверим отдельно)

virsh version
# Должно показать версию libvirt и qemu
```

## Шаг 1.3 — Настройка сетевого bridge

**[ВСЕ ХОСТЫ]** — eno1 становится частью bridge br0

> ⚠️ После применения SSH-соединение может прерваться на 10-30 секунд. Это нормально.

```bash
#!/usr/bin/env bash
# Определяем активный интерфейс и его параметры
IFACE="eno1"
HOST_IP=$(ip addr show $IFACE | grep -oP 'inet \K[\d.]+' | head -1)
HOST_PREFIX=$(ip addr show $IFACE | grep -oP 'inet [\d.]+/\K\d+' | head -1)
HOST_GW=$(ip route | grep default | awk '{print $3}' | head -1)

echo "Интерфейс: $IFACE"
echo "IP: $HOST_IP/$HOST_PREFIX"
echo "GW: $HOST_GW"

# Создаём bridge br0
nmcli connection add type bridge con-name br0 ifname br0 autoconnect yes
nmcli connection modify br0 bridge.stp no
nmcli connection modify br0 ipv4.method manual
nmcli connection modify br0 ipv4.addresses "${HOST_IP}/${HOST_PREFIX}"
nmcli connection modify br0 ipv4.gateway "${HOST_GW}"
nmcli connection modify br0 ipv4.dns "8.8.8.8"
nmcli connection modify br0 connection.zone trusted

# Добавляем eno1 как slave bridge
nmcli connection add type bridge-slave con-name br0-slave ifname $IFACE master br0 autoconnect yes

# Отключаем старое подключение eno1
nmcli connection modify "$IFACE" connection.autoconnect no

# Применяем (кратковременный обрыв соединения)
nmcli connection up br0-slave
nmcli connection up br0

# Проверяем
sleep 5
ip addr show br0
brctl show br0
```

## Шаг 1.4 — Проверка и включение IOMMU для GPU passthrough

**[ХОСТ1, ХОСТ2, ХОСТ3, ХОСТ4]** — только на серверах с GPU

```bash
#!/usr/bin/env bash
# Шаг 1: Проверяем текущее состояние

echo "=== Проверка IOMMU ==="
# Проверяем включён ли IOMMU
if dmesg | grep -qi "iommu: Default domain type"; then
  echo "IOMMU статус:"
  dmesg | grep -i iommu | head -5
else
  echo "IOMMU не активен"
fi

# Определяем тип CPU
if grep -q "GenuineIntel" /proc/cpuinfo; then
  CPU_TYPE="intel"
  IOMMU_PARAM="intel_iommu=on iommu=pt"
else
  CPU_TYPE="amd"
  IOMMU_PARAM="amd_iommu=on iommu=pt"
fi
echo "CPU: $CPU_TYPE → параметры: $IOMMU_PARAM"

# Добавляем параметры в GRUB если ещё нет
if ! grep -q "iommu" /proc/cmdline; then
  echo "Добавляем IOMMU в GRUB..."
  grubby --update-kernel=ALL --args="${IOMMU_PARAM}"
  echo "✓ Добавлено. Нужна перезагрузка."
  echo "Запустите: reboot"
  echo "После перезагрузки запустите этот скрипт снова."
else
  echo "✓ IOMMU параметры уже в cmdline"
  dmesg | grep -i "iommu" | grep -v "dmar" | head -3
fi

echo ""
echo "=== GPU устройства ==="
lspci -nn | grep -i nvidia
```

**[ХОСТ1, ХОСТ2, ХОСТ3, ХОСТ4]** — после перезагрузки с IOMMU

```bash
#!/usr/bin/env bash
# Шаг 2: После перезагрузки — проверяем IOMMU-группы GPU

# Проверяем что IOMMU активен
dmesg | grep -i iommu | grep -i "enabled\|pt\|passthrough" | head -3
if [ $? -ne 0 ]; then
  echo "ОШИБКА: IOMMU не активен. Проверьте BIOS/UEFI (AMD SVM, Intel VT-d)"
  exit 1
fi

echo "=== IOMMU группы для NVIDIA ==="
GPU_BDF=$(lspci | grep -i nvidia | grep -v audio | awk '{print $1}' | head -1)
echo "GPU BDF: $GPU_BDF"

# Находим IOMMU группу GPU
GPU_GROUP=$(ls -la /sys/bus/pci/devices/0000:${GPU_BDF}/iommu_group \
  | grep -oP 'iommu_groups/\K\d+')
echo "IOMMU группа: $GPU_GROUP"

echo "Все устройства в группе $GPU_GROUP:"
for dev in /sys/kernel/iommu_groups/${GPU_GROUP}/devices/*; do
  dev_id=$(basename $dev)
  echo "  $dev_id: $(lspci -s $dev_id 2>/dev/null)"
done

echo ""
echo "ВАЖНО: если в группе есть что-то кроме GPU и GPU Audio — passthrough сложнее."
echo "Tesla T4 обычно одна в группе или с NVIDIA Audio."
```

## Шаг 1.5 — Настройка VFIO для GPU passthrough

**[ХОСТ1, ХОСТ2, ХОСТ3, ХОСТ4]**

```bash
#!/usr/bin/env bash
set -euo pipefail

# Получаем PCI ID GPU (вендор:девайс)
GPU_PCI_ID=$(lspci -nn | grep -i nvidia | grep -v audio \
  | grep -oP '\[\K[0-9a-f]{4}:[0-9a-f]{4}(?=\])' | head -1)

if [ -z "$GPU_PCI_ID" ]; then
  echo "ОШИБКА: NVIDIA GPU не найден"
  exit 1
fi
echo "GPU PCI ID: $GPU_PCI_ID"

# Ищем NVIDIA Audio в той же группе (нужно передать вместе)
GPU_BDF=$(lspci -nn | grep -i nvidia | grep -v audio | awk '{print $1}' | head -1)
GPU_GROUP=$(ls -la /sys/bus/pci/devices/0000:${GPU_BDF}/iommu_group \
  | grep -oP 'iommu_groups/\K\d+')

VFIO_IDS="$GPU_PCI_ID"
AUDIO_PCI_ID=$(lspci -nn | grep -i "nvidia.*audio" \
  | grep -oP '\[\K[0-9a-f]{4}:[0-9a-f]{4}(?=\])' | head -1 || true)
if [ -n "$AUDIO_PCI_ID" ]; then
  VFIO_IDS="${GPU_PCI_ID},${AUDIO_PCI_ID}"
  echo "Также добавляем NVIDIA Audio: $AUDIO_PCI_ID"
fi

echo "VFIO IDs: $VFIO_IDS"

# Загружаем VFIO модули
cat > /etc/modules-load.d/vfio.conf <<EOF
vfio
vfio_iommu_type1
vfio_pci
EOF
modprobe vfio vfio_iommu_type1 vfio_pci || true

# Привязываем GPU к vfio-pci
cat > /etc/modprobe.d/vfio.conf <<EOF
options vfio-pci ids=${VFIO_IDS}
softdep nouveau pre: vfio-pci
softdep nvidia pre: vfio-pci
softdep nvidia* pre: vfio-pci
EOF

# Блокируем nvidia и nouveau на ХОСТЕ (GPU будет только в VM)
cat > /etc/modprobe.d/blacklist-nvidia-host.conf <<EOF
blacklist nouveau
blacklist nvidia
blacklist nvidia_modeset
blacklist nvidia_uvm
blacklist nvidia_drm
EOF

# Обновляем initramfs
dracut --force

echo ""
echo "=== VFIO настроен ==="
echo "Сохраните: GPU_BDF=$GPU_BDF, GPU_PCI_ID=$GPU_PCI_ID"
echo "Требуется перезагрузка: reboot"
```

**[ХОСТ1, ХОСТ2, ХОСТ3, ХОСТ4]** — после перезагрузки, проверка VFIO

```bash
# GPU должен использовать vfio-pci, а не nvidia
GPU_BDF=$(lspci | grep -i nvidia | grep -v audio | awk '{print $1}' | head -1)
lspci -ks ${GPU_BDF}
# Ожидаем: Kernel driver in use: vfio-pci

ls /dev/vfio/
# Должна быть папка /dev/vfio/ с числовым именем группы
```

---

# ЭТАП 2 — Создание виртуальных машин


## Шаг 2.1 — Разметка системного диска хоста под второй раздел данных

**[ВСЕ ХОСТЫ]**

Раскладка 3.5 TB системного диска хоста:
```
Уже существует:
  /boot/efi   600 MB
  /boot         1 GB
  /            75 GB  (ОС RedOS)

Добавляем:
  [резерв]    200 GB  (логи хоста, обновления, аварийный запас)
  qcow2 VM    120 GB  (диск ОС Ubuntu в VM, размещается в /var/lib/libvirt/images)
  KVM misc     50 GB  (ISO образы, nvram)
  ─────────────────────────────────────────
  Итого хосту: ~447 GB  (~13% диска — с большим запасом)
  Под vdc VM: ~3.1 TB  (всё оставшееся место)
```

> Хост получает 447 GB только под себя — этого хватит с запасом даже при активном росте логов и регулярных обновлениях ядра.

```bash
#!/usr/bin/env bash
# host-prepare-second-disk.sh
# Создаём раздел из остатка системного диска — отдаём в VM как vdc
set -euo pipefail

# Определяем системный диск хоста
ROOT_DISK=$(lsblk -no PKNAME "$(findmnt -n -o SOURCE /)" | head -1)
ROOT_DISK_DEV="/dev/${ROOT_DISK}"
echo "Системный диск хоста: $ROOT_DISK_DEV"

# Текущая разметка
echo ""
echo "=== Текущая разметка ==="
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT "$ROOT_DISK_DEV"
parted "$ROOT_DISK_DEV" print free
echo ""

# Конец последнего существующего раздела
LAST_END_BYTES=$(parted -m "$ROOT_DISK_DEV" unit B print \
  | grep -v "^BYT\|^$ROOT_DISK_DEV\|^$" \
  | awk -F: '{print $3}' | tr -d 'B' | sort -n | tail -1)

# Начинаем новый раздел через 500 MB после конца последнего
# (выравнивание + надёжный отступ под резерв хоста)
START_BYTES=$(( LAST_END_BYTES + 500*1024*1024 ))
START_MB=$(( START_BYTES / 1024 / 1024 ))

# Сколько места останется под новый раздел
DISK_SIZE_BYTES=$(parted -m "$ROOT_DISK_DEV" unit B print \
  | grep "^$ROOT_DISK_DEV:" | cut -d: -f2 | tr -d 'B')
FREE_GB=$(( (DISK_SIZE_BYTES - START_BYTES) / 1024 / 1024 / 1024 ))

echo "Начало нового раздела: ~$(( START_MB/1024 )) GB от начала диска"
echo "Размер нового раздела: ~${FREE_GB} GB"
echo ""

# Проверяем что остаток разумный (должно быть > 2 TB)
if [ "$FREE_GB" -lt 2000 ]; then
  echo "ОШИБКА: свободно только ${FREE_GB} GB — меньше 2 TB."
  echo "Проверьте разметку вручную: parted $ROOT_DISK_DEV print free"
  exit 1
fi

echo "Нажмите Enter для создания раздела ~${FREE_GB}GB или Ctrl+C для отмены"
read -r

# Создаём раздел — всё оставшееся место
parted -s "$ROOT_DISK_DEV" mkpart primary xfs "${START_MB}MB" 100%
partprobe "$ROOT_DISK_DEV"
sleep 3

# Имя нового раздела (последний в списке)
NEW_PART=$(lsblk -lno NAME "$ROOT_DISK_DEV" \
  | grep -v "^${ROOT_DISK}$" | tail -1)
NEW_PART_DEV="/dev/${NEW_PART}"

echo "Новый раздел: $NEW_PART_DEV"
lsblk "$NEW_PART_DEV"

# Форматируем в XFS
mkfs.xfs -f -L "vm-extra-disk" "$NEW_PART_DEV"

# Сохраняем путь — скрипты создания VM прочитают его
echo "$NEW_PART_DEV" > /root/vm-extra-disk-partition.txt
echo ""
echo "✓ Раздел создан: $NEW_PART_DEV (~${FREE_GB} GB)"
echo "  Путь сохранён в /root/vm-extra-disk-partition.txt"
echo ""
lsblk -o NAME,SIZE,FSTYPE,LABEL "$ROOT_DISK_DEV"
```

```bash
chmod +x host-prepare-second-disk.sh
./host-prepare-second-disk.sh
```

## Шаг 2.2 — Скачивание Ubuntu 22.04 ISO

**[ВСЕ ХОСТЫ]**

```bash
mkdir -p /var/lib/libvirt/images/iso
cd /var/lib/libvirt/images/iso

# Скачиваем Ubuntu 22.04.4 Server через прокси
# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
https_proxy=http://proxy.company.ru:3128 \
wget -c "https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso" \
  -O ubuntu-22.04.4-server.iso

# Проверяем размер (должно быть ~1.8 GB)
ls -lh ubuntu-22.04.4-server.iso
```

## Шаг 2.3 — Создание preseed/autoinstall для автоматической установки Ubuntu

**[ВСЕ ХОСТЫ]** — создаём autoinstall ISO чтобы не кликать вручную

```bash
#!/usr/bin/env bash
# generate-autoinstall.sh
# Запустить на каждом хосте, VM_NUM = номер ВМ (1-6)
set -euo pipefail

VM_NUM=$(hostname | grep -oP '\d+' | tail -1)
VM_NAME="vm-${VM_NUM}"
VM_IP="10.10.10.2${VM_NUM}"
VM_GW="10.10.10.1"         # ← замените на реальный шлюз
VM_DNS="8.8.8.8"
PROXY="http://proxy.company.ru:3128"

echo "Генерируем autoinstall для $VM_NAME ($VM_IP)"

mkdir -p /tmp/autoinstall

# Создаём user-data для Ubuntu autoinstall
cat > /tmp/autoinstall/user-data <<EOF
#cloud-config
autoinstall:
  version: 1
  locale: en_US.UTF-8
  keyboard: {layout: us}
  identity:
    hostname: ${VM_NAME}
    username: ubuntu
    # Пароль: ubuntu (сменить после первого входа!)
    password: '\$6\$rounds=4096\$saltsalt\$hashed_password_here'
  # Генерируем хэш: openssl passwd -6 'ubuntu'
  ssh:
    install-server: true
    allow-pw: true
  network:
    version: 2
    ethernets:
      ens3:
        addresses: [${VM_IP}/24]
        gateway4: ${VM_GW}
        nameservers:
          addresses: [${VM_DNS}]
  proxy: ${PROXY}
  apt:
    http_proxy: ${PROXY}
    https_proxy: ${PROXY}
  packages:
    - curl
    - wget
    - vim
    - htop
    - net-tools
    - software-properties-common
  storage:
    layout: {name: direct}
  late-commands:
    - echo 'ubuntu ALL=(ALL) NOPASSWD:ALL' > /target/etc/sudoers.d/ubuntu
    - chmod 440 /target/etc/sudoers.d/ubuntu
    - |
      cat >> /target/etc/environment << 'ENVEOF'
      http_proxy=${PROXY}
      https_proxy=${PROXY}
      HTTP_PROXY=${PROXY}
      HTTPS_PROXY=${PROXY}
      no_proxy=10.0.0.0/8,localhost,127.0.0.1,.svc,.cluster.local,10.10.10.0/24
      NO_PROXY=10.0.0.0/8,localhost,127.0.0.1,.svc,.cluster.local,10.10.10.0/24
      ENVEOF
EOF

# Генерируем правильный хэш пароля
PASS_HASH=$(openssl passwd -6 'ubuntu')
sed -i "s|hashed_password_here|${PASS_HASH}|" /tmp/autoinstall/user-data

# meta-data обязателен (может быть пустым)
touch /tmp/autoinstall/meta-data

# Упаковываем в ISO
cd /tmp/autoinstall
genisoimage \
  -output /var/lib/libvirt/images/iso/autoinstall-${VM_NAME}.iso \
  -volid cidata \
  -joliet \
  -rock \
  user-data meta-data

echo "✓ Autoinstall ISO: /var/lib/libvirt/images/iso/autoinstall-${VM_NAME}.iso"
```

```bash
# Запускаем скрипт
chmod +x generate-autoinstall.sh
./generate-autoinstall.sh
```

## Шаг 2.4 — Создание VM на GPU-хостах

**[ХОСТ1]** — создаём vm-1 (аналогично на ХОСТ2 → vm-2, ХОСТ3 → vm-3, ХОСТ4 → vm-4)

```bash
#!/usr/bin/env bash
# create-vm-gpu.sh
set -euo pipefail

VM_NUM=$(hostname | grep -oP '\d+' | tail -1)
VM_NAME="vm-${VM_NUM}"
VM_DIR="/var/lib/libvirt/images"
ISO_DIR="${VM_DIR}/iso"
UBUNTU_ISO="${ISO_DIR}/ubuntu-22.04.4-server.iso"
AUTOINSTALL_ISO="${ISO_DIR}/autoinstall-${VM_NAME}.iso"

echo "=== Создаём $VM_NAME на $(hostname) ==="

# Получаем BDF адрес GPU
GPU_BDF=$(lspci | grep -i nvidia | grep -v audio | awk '{print $1}' | head -1)
GPU_BUS=$(echo $GPU_BDF | cut -d: -f1)
GPU_SLOT=$(echo $GPU_BDF | cut -d: -f2 | cut -d. -f1)
GPU_FUNC=$(echo $GPU_BDF | cut -d. -f2)
echo "GPU: $GPU_BDF (bus=$GPU_BUS slot=$GPU_SLOT func=$GPU_FUNC)"

# Ищем NVIDIA Audio
AUDIO_BDF=$(lspci | grep -i "nvidia.*audio" | awk '{print $1}' | head -1 || true)

# Ищем диск данных (18.2 TB, не системный)
ROOT_DISK=$(lsblk -no PKNAME "$(findmnt -n -o SOURCE /)" | head -1)
DATA_DISK=""
while read -r NAME SIZE TYPE; do
  [ "$TYPE" = "disk" ] || continue
  [ "$NAME" = "$ROOT_DISK" ] && continue
  SIZE_BYTES=$(lsblk -bnd -o SIZE "/dev/$NAME" 2>/dev/null | head -1)
  if [ "${SIZE_BYTES:-0}" -gt $((15*1024*1024*1024*1024)) ]; then
    if ! grep -q "/dev/$NAME" /proc/mounts 2>/dev/null; then
      DATA_DISK="/dev/$NAME"
      break
    fi
  fi
done < <(lsblk -nd -o NAME,SIZE,TYPE)

[ -n "$DATA_DISK" ] || { echo "ОШИБКА: диск данных 18TB не найден"; exit 1; }
echo "Диск данных (vdb): $DATA_DISK"

# Читаем раздел под второй диск (создали на шаге 2.1)
EXTRA_PART=$(cat /root/vm-extra-disk-partition.txt 2>/dev/null || true)
if [ -z "$EXTRA_PART" ] || [ ! -b "$EXTRA_PART" ]; then
  echo "ОШИБКА: файл /root/vm-extra-disk-partition.txt не найден или раздел не существует"
  echo "Запустите шаг 2.1: host-prepare-second-disk.sh"
  exit 1
fi
EXTRA_SIZE_GB=$(lsblk -bnd -o SIZE "$EXTRA_PART" | awk '{printf "%.0f", $1/1024/1024/1024}')
echo "Второй диск данных (vdc): $EXTRA_PART (~${EXTRA_SIZE_GB} GB)"

# Создаём диск для ОС VM (qcow2 на системном диске хоста)
qemu-img create -f qcow2 "${VM_DIR}/${VM_NAME}-os.qcow2" 100G

# Формируем XML для GPU passthrough
GPU_HOSTDEV="<hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x${GPU_BUS}' slot='0x${GPU_SLOT}' function='0x${GPU_FUNC}'/>
      </source>
      <rom bar='off'/>
    </hostdev>"

AUDIO_HOSTDEV=""
if [ -n "$AUDIO_BDF" ]; then
  A_BUS=$(echo $AUDIO_BDF | cut -d: -f1)
  A_SLOT=$(echo $AUDIO_BDF | cut -d: -f2 | cut -d. -f1)
  A_FUNC=$(echo $AUDIO_BDF | cut -d. -f2)
  AUDIO_HOSTDEV="<hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x${A_BUS}' slot='0x${A_SLOT}' function='0x${A_FUNC}'/>
      </source>
    </hostdev>"
fi

cat > "/tmp/${VM_NAME}.xml" <<XML
<domain type='kvm'>
  <name>${VM_NAME}</name>
  <uuid>$(uuidgen)</uuid>
  <memory unit='MiB'>114688</memory>
  <currentMemory unit='MiB'>114688</currentMemory>
  <vcpu placement='static'>120</vcpu>
  <os>
    <type arch='x86_64' machine='q35'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/edk2/ovmf/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/${VM_NAME}_VARS.fd</nvram>
    <boot dev='hd'/>
    <boot dev='cdrom'/>
    <bootmenu enable='no'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <ioapic driver='kvm'/>
  </features>
  <cpu mode='host-passthrough' check='none' migratable='off'>
    <topology sockets='2' dies='1' cores='30' threads='2'/>
  </cpu>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>

    <!-- ОС VM (qcow2) -->
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='native' discard='unmap'/>
      <source file='${VM_DIR}/${VM_NAME}-os.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <boot order='1'/>
    </disk>

    <!-- Диск данных PRIMARY (18.2 TB физический диск напрямую) -->
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native' discard='unmap'/>
      <source dev='${DATA_DISK}'/>
      <target dev='vdb' bus='virtio'/>
    </disk>

    <!-- Диск данных SECONDARY (~3.1 TB раздел на системном диске хоста) -->
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native' discard='unmap'/>
      <source dev='${EXTRA_PART}'/>
      <target dev='vdc' bus='virtio'/>
    </disk>

    <!-- Ubuntu ISO для установки -->
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='${UBUNTU_ISO}'/>
      <target dev='sda' bus='sata'/>
      <readonly/>
      <boot order='2'/>
    </disk>

    <!-- Autoinstall ISO (cloud-init) -->
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='${AUTOINSTALL_ISO}'/>
      <target dev='sdb' bus='sata'/>
      <readonly/>
    </disk>

    <!-- Сеть через bridge -->
    <interface type='bridge'>
      <source bridge='br0'/>
      <model type='virtio'/>
    </interface>

    <!-- GPU passthrough -->
    ${GPU_HOSTDEV}

    <!-- NVIDIA Audio (если есть) -->
    ${AUDIO_HOSTDEV}

    <!-- Watchdog для автоперезапуска VM при зависании -->
    <watchdog model='i6300esb' action='reset'/>

    <!-- Консоль для отладки -->
    <serial type='pty'>
      <target type='isa-serial' port='0'/>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>

    <!-- Virtio RNG -->
    <rng model='virtio'>
      <backend model='random'>/dev/urandom</backend>
    </rng>
  </devices>
</domain>
XML

# Проверяем что edk2/OVMF есть (нужен для UEFI + GPU passthrough)
if [ ! -f /usr/share/edk2/ovmf/OVMF_CODE.fd ]; then
  echo "Устанавливаем OVMF..."
  dnf install -y edk2-ovmf
fi
mkdir -p /var/lib/libvirt/qemu/nvram/
cp /usr/share/edk2/ovmf/OVMF_VARS.fd \
   "/var/lib/libvirt/qemu/nvram/${VM_NAME}_VARS.fd"

# Регистрируем VM
virsh define "/tmp/${VM_NAME}.xml"

# Включаем autostart при старте хоста
virsh autostart "${VM_NAME}"

echo "✓ VM ${VM_NAME} создана и зарегистрирована"
echo "Запуск: virsh start ${VM_NAME}"
```

## Шаг 2.5 — Создание VM на storage-хостах

**[ХОСТ5]** — vm-5 (аналогично ХОСТ6 → vm-6)

```bash
#!/usr/bin/env bash
# create-vm-storage.sh
set -euo pipefail

VM_NUM=$(hostname | grep -oP '\d+' | tail -1)
VM_NAME="vm-${VM_NUM}"
VM_DIR="/var/lib/libvirt/images"
ISO_DIR="${VM_DIR}/iso"

ROOT_DISK=$(lsblk -no PKNAME "$(findmnt -n -o SOURCE /)" | head -1)
DATA_DISK=""
while read -r NAME SIZE TYPE; do
  [ "$TYPE" = "disk" ] || continue
  [ "$NAME" = "$ROOT_DISK" ] && continue
  SIZE_BYTES=$(lsblk -bnd -o SIZE "/dev/$NAME" 2>/dev/null | head -1)
  if [ "${SIZE_BYTES:-0}" -gt $((15*1024*1024*1024*1024)) ]; then
    if ! grep -q "/dev/$NAME" /proc/mounts 2>/dev/null; then
      DATA_DISK="/dev/$NAME"; break
    fi
  fi
done < <(lsblk -nd -o NAME,SIZE,TYPE)

[ -n "$DATA_DISK" ] || { echo "ОШИБКА: диск 18TB не найден"; exit 1; }
echo "Диск данных (vdb): $DATA_DISK"

# Читаем раздел под второй диск (создали на шаге 2.1)
EXTRA_PART=$(cat /root/vm-extra-disk-partition.txt 2>/dev/null || true)
if [ -z "$EXTRA_PART" ] || [ ! -b "$EXTRA_PART" ]; then
  echo "ОШИБКА: /root/vm-extra-disk-partition.txt не найден или раздел не существует"
  echo "Запустите шаг 2.1: host-prepare-second-disk.sh"
  exit 1
fi
EXTRA_SIZE_GB=$(lsblk -bnd -o SIZE "$EXTRA_PART" | awk '{printf "%.0f", $1/1024/1024/1024}')
echo "Второй диск данных (vdc): $EXTRA_PART (~${EXTRA_SIZE_GB} GB)"

qemu-img create -f qcow2 "${VM_DIR}/${VM_NAME}-os.qcow2" 100G

cat > "/tmp/${VM_NAME}.xml" <<XML
<domain type='kvm'>
  <n>${VM_NAME}</n>
  <uuid>$(uuidgen)</uuid>
  <memory unit='MiB'>114688</memory>
  <currentMemory unit='MiB'>114688</currentMemory>
  <vcpu placement='static'>60</vcpu>
  <os>
    <type arch='x86_64' machine='q35'>hvm</type>
    <boot dev='hd'/>
    <boot dev='cdrom'/>
    <bootmenu enable='no'/>
  </os>
  <features><acpi/><apic/></features>
  <cpu mode='host-passthrough' check='none' migratable='off'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='native' discard='unmap'/>
      <source file='${VM_DIR}/${VM_NAME}-os.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <boot order='1'/>
    </disk>
    <!-- Диск данных PRIMARY (18.2 TB физический диск напрямую) -->
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native' discard='unmap'/>
      <source dev='${DATA_DISK}'/>
      <target dev='vdb' bus='virtio'/>
    </disk>
    <!-- Диск данных SECONDARY (~3.1 TB раздел на системном диске хоста) -->
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native' discard='unmap'/>
      <source dev='${EXTRA_PART}'/>
      <target dev='vdc' bus='virtio'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='${ISO_DIR}/ubuntu-22.04.4-server.iso'/>
      <target dev='sda' bus='sata'/>
      <readonly/>
      <boot order='2'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='${ISO_DIR}/autoinstall-${VM_NAME}.iso'/>
      <target dev='sdb' bus='sata'/>
      <readonly/>
    </disk>
    <interface type='bridge'>
      <source bridge='br0'/>
      <model type='virtio'/>
    </interface>
    <watchdog model='i6300esb' action='reset'/>
    <serial type='pty'>
      <target type='isa-serial' port='0'/>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <rng model='virtio'>
      <backend model='random'>/dev/urandom</backend>
    </rng>
  </devices>
</domain>
XML

virsh define "/tmp/${VM_NAME}.xml"
virsh autostart "${VM_NAME}"
echo "✓ VM ${VM_NAME} создана (vdb: 18.2TB + vdc: ${EXTRA_SIZE_GB}GB)"

```

## Шаг 2.6 — Запуск VM и установка Ubuntu

**[ВСЕ ХОСТЫ]** — запускаем VM и следим за установкой

```bash
VM_NUM=$(hostname | grep -oP '\d+' | tail -1)
VM_NAME="vm-${VM_NUM}"

# Запускаем VM
virsh start ${VM_NAME}

# Подключаемся к консоли для наблюдения за установкой
# Выход из консоли: Ctrl+]
virsh console ${VM_NAME}

# Установка занимает 5-10 минут в автоматическом режиме
# VM перезагрузится после установки

# Проверяем что VM запущена
virsh list --all
```

## Шаг 2.7 — Настройка watchdog-сервиса на хостах

**[ВСЕ ХОСТЫ]** — автоперезапуск упавшей VM

```bash
VM_NUM=$(hostname | grep -oP '\d+' | tail -1)
VM_NAME="vm-${VM_NUM}"

cat > "/etc/systemd/system/vm-watchdog-${VM_NAME}.service" <<EOF
[Unit]
Description=Watchdog для ${VM_NAME}
After=libvirtd.service network-online.target
Wants=network-online.target

[Service]
Type=simple
Restart=always
RestartSec=10
# Ждём 60 секунд после старта хоста прежде чем проверять
# (bridge должен подняться раньше VM)
ExecStartPre=/bin/sleep 60
ExecStart=/bin/bash -c '\
  while true; do \
    STATE=\$(virsh domstate ${VM_NAME} 2>/dev/null || echo "undefined"); \
    if [ "\$STATE" != "running" ] && [ "\$STATE" != "paused" ]; then \
      echo "\$(date): ${VM_NAME} в состоянии \$STATE, перезапускаем..."; \
      virsh start ${VM_NAME} || true; \
      sleep 120; \
    fi; \
    sleep 30; \
  done'

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now "vm-watchdog-${VM_NAME}.service"

echo "✓ Watchdog для ${VM_NAME} настроен"
systemctl status "vm-watchdog-${VM_NAME}.service" --no-pager
```

## Шаг 2.8 — Проверка доступности VM по SSH

**[С вашего рабочего компьютера или jump-хоста]**

```bash
# После установки Ubuntu проверяем SSH на всех VM
for i in 1 2 3 4 5 6; do
  echo -n "vm-${i} (10.10.10.2${i}): "
  ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no \
    ubuntu@10.10.10.2${i} "hostname && uname -r" 2>/dev/null \
    || echo "недоступна"
done
```

---

# ЭТАП 3 — Подготовка Ubuntu внутри VM

## Шаг 3.1 — Базовая настройка Ubuntu

**[ВСЕ VM]** — выполнить на всех 6 VM

```bash
#!/usr/bin/env bash
# vm-base-setup.sh
set -euo pipefail

PROXY="http://proxy.company.ru:3128"
NO_PROXY_LIST="10.0.0.0/8,192.168.0.0/16,172.16.0.0/12,localhost,127.0.0.1,.svc,.cluster.local,10.10.10.0/24"

echo "=== Базовая настройка $(hostname) ==="

# Прокси для apt
cat > /etc/apt/apt.conf.d/01proxy <<EOF
Acquire::http::Proxy "${PROXY}";
Acquire::https::Proxy "${PROXY}";
EOF

# Прокси для системы
cat >> /etc/environment <<EOF
http_proxy=${PROXY}
https_proxy=${PROXY}
HTTP_PROXY=${PROXY}
HTTPS_PROXY=${PROXY}
no_proxy=${NO_PROXY_LIST}
NO_PROXY=${NO_PROXY_LIST}
EOF

export http_proxy=$PROXY https_proxy=$PROXY
export no_proxy=$NO_PROXY_LIST

# Обновляем систему
apt-get update -y
DEBIAN_FRONTEND=noninteractive apt-get upgrade -y

# Базовые пакеты
DEBIAN_FRONTEND=noninteractive apt-get install -y \
  curl wget vim htop net-tools \
  software-properties-common \
  apt-transport-https \
  ca-certificates \
  gnupg \
  lsb-release \
  chrony \
  open-iscsi \
  nfs-common \
  xfsprogs \
  jq \
  socat \
  conntrack \
  ipset \
  ipvsadm \
  ebtables \
  ethtool

# Синхронизация времени
systemctl enable --now chrony

# Отключаем swap (Ubuntu 22.04 может иметь swap-файл)
swapoff -a
sed -ri 's/^([^#].*\sswap\s.*)$/#\1/' /etc/fstab
systemctl mask swap.target 2>/dev/null || true

echo "✓ Базовая настройка завершена"
```

## Шаг 3.2 — Системные параметры для Kubernetes

**[ВСЕ VM]**

```bash
#!/usr/bin/env bash
# vm-k8s-prereqs.sh
set -euo pipefail

# Модули ядра
cat > /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
iscsi_tcp
EOF
modprobe overlay br_netfilter iscsi_tcp

# sysctl
cat > /etc/sysctl.d/99-k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv4.conf.all.rp_filter         = 0
net.ipv4.conf.default.rp_filter     = 0
fs.inotify.max_user_instances       = 8192
fs.inotify.max_user_watches         = 524288
vm.max_map_count                    = 262144
kernel.pid_max                      = 4194304
EOF
sysctl --system

# ufw если установлен — отключаем (используем iptables напрямую)
systemctl disable --now ufw 2>/dev/null || true

# iSCSI для Longhorn
systemctl enable --now iscsid

echo "✓ Системные параметры для k8s применены"
```

## Шаг 3.3 — Установка containerd

**[ВСЕ VM]**

```bash
#!/usr/bin/env bash
# vm-install-containerd.sh
set -euo pipefail

PROXY="http://proxy.company.ru:3128"
export http_proxy=$PROXY https_proxy=$PROXY

# Добавляем ключ Docker репозитория
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# Добавляем репозиторий Docker (содержит containerd.io)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  > /etc/apt/sources.list.d/docker.list

apt-get update -y
apt-get install -y containerd.io

# Конфигурируем containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# SystemdCgroup — обязательно для k8s
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sed -i 's|sandbox_image = "registry.k8s.io/pause:.*"|sandbox_image = "registry.k8s.io/pause:3.10"|' \
  /etc/containerd/config.toml

# [ТОЛЬКО ДЛЯ ПРОКСИ] Настраиваем прокси для containerd
# При прямом доступе этот блок пропустить
mkdir -p /etc/systemd/system/containerd.service.d
cat > /etc/systemd/system/containerd.service.d/proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=${PROXY}"
Environment="HTTPS_PROXY=${PROXY}"
Environment="NO_PROXY=10.0.0.0/8,192.168.0.0/16,127.0.0.1,localhost,.svc,.cluster.local,10.10.10.0/24"
EOF

systemctl daemon-reload
systemctl enable --now containerd

# Проверка
cat > /etc/crictl.yaml <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
EOF

crictl info | grep -i cgroup
# Ожидаем: "cgroupDriver": "systemd"

echo "✓ containerd установлен и настроен"
```

## Шаг 3.4 — Установка kubeadm, kubelet, kubectl

**[ВСЕ VM]**

```bash
#!/usr/bin/env bash
# vm-install-k8s-packages.sh
set -euo pipefail

PROXY="http://proxy.company.ru:3128"
export http_proxy=$PROXY https_proxy=$PROXY

K8S_VERSION="1.32"
K8S_PKG_VERSION="1.32.3-1.1"

# Добавляем ключ Kubernetes репозитория
curl -fsSL "https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/Release.key" \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/ /" \
  > /etc/apt/sources.list.d/kubernetes.list

apt-get update -y
apt-get install -y \
  kubelet=${K8S_PKG_VERSION} \
  kubeadm=${K8S_PKG_VERSION} \
  kubectl=${K8S_PKG_VERSION}

# Закрепляем версию — не обновлять автоматически
apt-mark hold kubelet kubeadm kubectl containerd.io

# [ТОЛЬКО ДЛЯ ПРОКСИ] Прокси для kubelet (для pull образов через containerd)
# При прямом доступе этот блок пропустить
mkdir -p /etc/systemd/system/kubelet.service.d
cat > /etc/systemd/system/kubelet.service.d/proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=${PROXY}"
Environment="HTTPS_PROXY=${PROXY}"
Environment="NO_PROXY=10.0.0.0/8,192.168.0.0/16,127.0.0.1,localhost,.svc,.cluster.local,10.10.10.0/24"
EOF

systemctl daemon-reload
systemctl enable kubelet

# Предзагружаем образы (пока есть доступ)
kubeadm config images pull \
  --kubernetes-version=v1.32.3 \
  --image-repository=registry.k8s.io

kubeadm version -o short
kubectl version --client

echo "✓ Kubernetes пакеты установлены"
```

## Шаг 3.5 — Установка NVIDIA драйвера в GPU-VM

**[GPU VM]** — vm-1, vm-2, vm-3, vm-4

> ⚠️ ВАЖНО: Перед этим шагом убедитесь что GPU виден внутри VM: `lspci | grep -i nvidia`

```bash
#!/usr/bin/env bash
# vm-install-nvidia.sh
set -euo pipefail

PROXY="http://proxy.company.ru:3128"
export http_proxy=$PROXY https_proxy=$PROXY

# Проверяем наличие GPU
lspci | grep -i nvidia || {
  echo "ОШИБКА: GPU не найден. Проверьте passthrough на хосте."
  exit 1
}
GPU_MODEL=$(lspci | grep -i nvidia | grep -v audio)
echo "Найден GPU: $GPU_MODEL"

# Ubuntu 22.04 + GPU passthrough: нужно отключить efifb
# Иначе Ubuntu зависнет при загрузке с GPU passthrough
if ! grep -q "efifb:off" /etc/default/grub; then
  sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT=".*"/GRUB_CMDLINE_LINUX_DEFAULT="video=vesafb:off,efifb:off quiet splash"/' \
    /etc/default/grub
  update-grub
fi

# Блокируем nouveau
cat > /etc/modprobe.d/blacklist-nouveau.conf <<'EOF'
blacklist nouveau
options nouveau modeset=0
EOF
update-initramfs -u

# Устанавливаем зависимости для DKMS
apt-get install -y \
  build-essential \
  dkms \
  linux-headers-$(uname -r) \
  pkg-config \
  libglvnd-dev

# Добавляем репозиторий NVIDIA
curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/3bf863cc.pub \
  | gpg --dearmor -o /usr/share/keyrings/cuda-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/cuda-keyring.gpg] \
  https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/ /" \
  > /etc/apt/sources.list.d/cuda-ubuntu2204.list

apt-get update -y

# Устанавливаем драйвер NVIDIA 570
# Ubuntu 22.04 поддерживает open-kernel драйвер для Tesla T4 (Turing)
apt-get install -y nvidia-driver-570-open

# NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  > /etc/apt/sources.list.d/nvidia-container-toolkit.list

apt-get update -y
apt-get install -y nvidia-container-toolkit

# Настраиваем containerd для NVIDIA
nvidia-ctk runtime configure --runtime=containerd \
  --config=/etc/containerd/config.toml \
  --set-as-default=false \
  --cdi.enabled=true

nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml 2>/dev/null || true

# Фиксируем версии
apt-mark hold "nvidia-driver-570-open" nvidia-container-toolkit

systemctl restart containerd

echo ""
echo "✓ NVIDIA драйвер установлен. Требуется перезагрузка."
echo "После перезагрузки: nvidia-smi"
```

```bash
# Перезагружаем VM
reboot
```

**[GPU VM]** — после перезагрузки, проверка GPU

```bash
nvidia-smi
# Ожидаем: Tesla T4, Driver 570.x.x, CUDA 12.x

# Проверяем containerd видит GPU
crictl info | grep -A5 runtimes
```

## Шаг 3.6 — Подготовка дисков данных (vdb + vdc)

**[ВСЕ VM]**

Внутри каждой VM теперь два диска данных:
- `vdb` — 18.2 TB (физический диск, пробрашен напрямую)
- `vdc` — ~3.1 TB (раздел системного диска хоста)

Longhorn автоматически объединит оба диска на каждой ноде.

```bash
#!/usr/bin/env bash
# vm-prepare-disks.sh — форматирует vdb и vdc, монтирует под Longhorn
set -euo pipefail

apt-get install -y gdisk parted xfsprogs >/dev/null 2>&1

prepare_disk() {
  local DISK="$1"
  local MOUNT="$2"
  local LABEL="$3"

  echo ""
  echo "=== Подготовка $DISK → $MOUNT ==="

  # Проверяем что диск существует
  lsblk "$DISK" 2>/dev/null || {
    echo "ОШИБКА: диск $DISK не найден"
    return 1
  }

  SIZE_GB=$(lsblk -bnd -o SIZE "$DISK" | awk '{printf "%.0f", $1/1024/1024/1024}')
  echo "Размер: ${SIZE_GB} GB"

  # Проверяем есть ли уже ФС
  if lsblk -no FSTYPE "$DISK" | grep -qE '\S'; then
    echo "На диске есть ФС. Для переформатирования: FORCE=1 $0"
    if [ "${FORCE:-0}" != "1" ]; then
      # Проверяем что уже смонтирован правильно
      if mountpoint -q "$MOUNT" 2>/dev/null; then
        echo "✓ Уже смонтирован: $MOUNT"
        return 0
      fi
      # Монтируем существующий раздел
      mkdir -p "$MOUNT"; chmod 700 "$MOUNT"
      EXISTING_UUID=$(blkid -s UUID -o value "${DISK}1" 2>/dev/null                       || blkid -s UUID -o value "$DISK" 2>/dev/null || true)
      if [ -n "$EXISTING_UUID" ]; then
        sed -i "\|$MOUNT|d" /etc/fstab
        echo "UUID=${EXISTING_UUID}  ${MOUNT}  xfs  defaults,noatime,nofail,x-systemd.device-timeout=30  0  0" >> /etc/fstab
        systemctl daemon-reload && mount -a
        mountpoint -q "$MOUNT" && echo "✓ Примонтирован существующий раздел" && return 0
      fi
      echo "ОШИБКА: не удалось смонтировать. Запустите с FORCE=1 для переформатирования."
      return 1
    fi
    wipefs -a "$DISK"
  fi

  # Размечаем и форматируем
  sgdisk --zap-all "$DISK"
  sgdisk -n 1:0:0 -t 1:8300 -c 1:"${LABEL}" "$DISK"
  partprobe "$DISK"; sleep 2

  PART="${DISK}1"
  mkfs.xfs -f -L "${LABEL}" -i size=512 "$PART"
  mkdir -p "$MOUNT"; chmod 700 "$MOUNT"

  UUID=$(blkid -s UUID -o value "$PART")
  sed -i "\|$MOUNT|d" /etc/fstab
  echo "UUID=${UUID}  ${MOUNT}  xfs  defaults,noatime,nofail,x-systemd.device-timeout=30  0  0" >> /etc/fstab

  systemctl daemon-reload
  mount "$MOUNT"
  mountpoint -q "$MOUNT" && echo "✓ Примонтирован: $MOUNT (${SIZE_GB} GB)"
}

# Основной диск данных — 18.2 TB
prepare_disk "/dev/vdb" "/var/lib/longhorn" "lh-primary"

# Второй диск данных — ~3.1 TB
prepare_disk "/dev/vdc" "/var/lib/longhorn-extra" "lh-extra"

echo ""
echo "=== Итоговое состояние дисков ==="
df -hT /var/lib/longhorn /var/lib/longhorn-extra
echo ""
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT /dev/vdb /dev/vdc
```

```bash
chmod +x vm-prepare-disks.sh
./vm-prepare-disks.sh

# Проверяем результат
df -hT /var/lib/longhorn /var/lib/longhorn-extra
```

## Шаг 3.7 — Настройка /etc/hosts и hostname в VM

**[ВСЕ VM]**

```bash
#!/usr/bin/env bash
# vm-setup-hosts.sh
set -euo pipefail

VM_NUM=$(hostname | grep -oP '\d+' | tail -1)

cat >> /etc/hosts <<'EOF'

# Kubernetes кластер
10.10.10.20  k8s-api
10.10.10.21  vm-1
10.10.10.22  vm-2
10.10.10.23  vm-3
10.10.10.24  vm-4
10.10.10.25  vm-5
10.10.10.26  vm-6
EOF

echo "✓ /etc/hosts обновлён"
cat /etc/hosts | tail -10
```

---

# ЭТАП 4 — Развёртывание Kubernetes кластера

## Шаг 4.1 — kube-vip на CP нодах

**[VM1, VM2, VM3]**

```bash
#!/usr/bin/env bash
# Определяем интерфейс (в Ubuntu 22.04 в VM обычно ens3 или eth0)
IFACE=$(ip route | grep default | awk '{print $5}' | head -1)
VIP="10.10.10.20"
KVVERSION="v0.9.2"

echo "Интерфейс: $IFACE"
echo "VIP: $VIP"

mkdir -p /etc/kubernetes/manifests

# Скачиваем образ через прокси
# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
  ctr image pull ghcr.io/kube-vip/kube-vip:${KVVERSION}

# Генерируем манифест static pod
ctr run --rm --net-host \
  ghcr.io/kube-vip/kube-vip:${KVVERSION} vip \
  /kube-vip manifest pod \
    --interface ${IFACE} \
    --address ${VIP} \
    --controlplane \
    --services \
    --arp \
    --leaderElection \
  | tee /etc/kubernetes/manifests/kube-vip.yaml

echo "✓ kube-vip манифест создан"
cat /etc/kubernetes/manifests/kube-vip.yaml | grep -E "address|interface"
```

## Шаг 4.2 — Инициализация первого control plane

**[VM1]** — только на vm-1

```bash
cat > /root/kubeadm-init.yaml <<'EOF'
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.10.10.21
  bindPort: 6443
nodeRegistration:
  name: vm-1
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
  - {name: node-ip, value: 10.10.10.21}
  taints: []
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.32.3
clusterName: kubernetes
controlPlaneEndpoint: "k8s-api:6443"
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
apiServer:
  extraArgs:
  - {name: audit-log-maxage,    value: "30"}
  - {name: audit-log-maxbackup, value: "10"}
  - {name: audit-log-maxsize,   value: "100"}
  certSANs:
  - k8s-api
  - 10.10.10.20
  - 10.10.10.21
  - 10.10.10.22
  - 10.10.10.23
  - vm-1
  - vm-2
  - vm-3
  - kubernetes
  - kubernetes.default
  - kubernetes.default.svc
  - kubernetes.default.svc.cluster.local
  - 127.0.0.1
  - localhost
controllerManager:
  extraArgs:
  - {name: bind-address, value: "0.0.0.0"}
scheduler:
  extraArgs:
  - {name: bind-address, value: "0.0.0.0"}
etcd:
  local:
    dataDir: /var/lib/etcd
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
serverTLSBootstrap: true
clusterDNS: [10.96.0.10]
clusterDomain: cluster.local
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
ipvs:
  strictARP: true
EOF

# Запускаем инициализацию
kubeadm init --config=/root/kubeadm-init.yaml --upload-certs 2>&1 | tee /root/kubeadm-init.log

# Настраиваем kubectl
mkdir -p $HOME/.kube
cp -f /etc/kubernetes/admin.conf $HOME/.kube/config

# Фикс пути для kube-vip (kubeadm 1.29+ создаёт super-admin.conf)
sed -i 's#path: /etc/kubernetes/super-admin.conf#path: /etc/kubernetes/admin.conf#' \
  /etc/kubernetes/manifests/kube-vip.yaml 2>/dev/null || true

# Проверяем VIP
sleep 10
curl -k https://10.10.10.20:6443/healthz && echo "VIP работает!"

# Сохраняем команды для join
echo ""
echo "======= СОХРАНИТЕ ЭТИ ДАННЫЕ ======="
grep -A2 "kubeadm join\|certificate-key\|discovery-token" /root/kubeadm-init.log | tail -20
echo "======================================"
```

## Шаг 4.3 — Подключение vm-2 и vm-3 как control plane

**[VM2]** — создаём конфиг join (аналогично для VM3 с IP .23)

```bash
# Данные TOKEN, HASH и CERT_KEY берём из вывода kubeadm init на vm-1
TOKEN="<токен из kubeadm init>"
HASH="<sha256:хеш из kubeadm init>"
CERT_KEY="<certificate-key из kubeadm init>"

cat > /root/kubeadm-join-cp.yaml <<EOF
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: "k8s-api:6443"
    token: "${TOKEN}"
    caCertHashes: ["${HASH}"]
controlPlane:
  localAPIEndpoint:
    advertiseAddress: 10.10.10.22
    bindPort: 6443
  certificateKey: "${CERT_KEY}"
nodeRegistration:
  name: vm-2
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
  - {name: node-ip, value: 10.10.10.22}
  taints: []
EOF

kubeadm join --config=/root/kubeadm-join-cp.yaml 2>&1 | tee /root/kubeadm-join.log

# Фикс kube-vip
sed -i 's#super-admin.conf#admin.conf#' \
  /etc/kubernetes/manifests/kube-vip.yaml 2>/dev/null || true

# kubectl для vm-2
mkdir -p $HOME/.kube
# Скопировать admin.conf с vm-1:
# scp ubuntu@vm-1:/etc/kubernetes/admin.conf $HOME/.kube/config
```

> ⚠️ **certificate-key живёт 2 часа!** Если истёк — на vm-1 выполните:
> ```bash
> kubeadm init phase upload-certs --upload-certs
> kubeadm token create --print-join-command
> ```

## Шаг 4.4 — Подключение воркеров vm-4, vm-5, vm-6

**[VM4, VM5, VM6]** — каждый со своим IP

```bash
# Пример для vm-4
TOKEN="<токен>"
HASH="<sha256:хеш>"
VM_IP="10.10.10.24"   # .24 для vm-4, .25 для vm-5, .26 для vm-6
VM_NAME="vm-4"        # vm-5, vm-6 соответственно

cat > /root/kubeadm-join-worker.yaml <<EOF
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: "k8s-api:6443"
    token: "${TOKEN}"
    caCertHashes: ["${HASH}"]
nodeRegistration:
  name: ${VM_NAME}
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
  - {name: node-ip, value: ${VM_IP}}
EOF

kubeadm join --config=/root/kubeadm-join-worker.yaml 2>&1 | tee /root/kubeadm-join.log
```

## Шаг 4.5 — Установка Calico CNI

**[VM1]**

```bash
export CALICO_VERSION=v3.29.3

# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/operator-crds.yaml

# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml

# Определяем интерфейс VM
IFACE=$(ip route | grep default | awk '{print $5}' | head -1)

cat > /root/calico-install.yaml <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  cni:
    type: Calico
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - name: default-ipv4-ippool
      cidr: 10.244.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
      blockSize: 26
    mtu: 1450
    nodeAddressAutodetectionV4:
      interface: "${IFACE}"
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF

kubectl apply -f /root/calico-install.yaml

# Ждём готовности (3-5 минут)
kubectl wait --for=condition=Available tigerastatus/calico --timeout=5m
kubectl get tigerastatus
```

## Шаг 4.6 — Метки нод

**[VM1]**

```bash
# GPU-ноды
for n in vm-1 vm-2 vm-3 vm-4; do
  kubectl label node $n gpu=true --overwrite
done

# Storage-ноды
for n in vm-5 vm-6; do
  kubectl label node $n storage=true --overwrite
done

# Проверяем
kubectl get nodes -o wide
kubectl get nodes --show-labels
```

## Шаг 4.7 — Проверка кластера

**[VM1]**

```bash
# Все ноды должны быть Ready
kubectl get nodes -o wide

# Поды системы запущены
kubectl get pods -A | grep -v Running | grep -v Completed

# etcd кластер здоров
kubectl -n kube-system exec etcd-vm-1 -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster -w table

# VIP работает
curl -k https://10.10.10.20:6443/healthz

# Срок сертификатов
kubeadm certs check-expiration
```

---

# ЭТАП 5 — GPU Operator и time-slicing

## Шаг 5.1 — Установка Helm

**[VM1]**

```bash
# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
https_proxy=http://proxy.company.ru:3128 \
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
```

## Шаг 5.2 — ConfigMap для time-slicing

**[VM1]**

```bash
kubectl create namespace gpu-operator
kubectl label ns gpu-operator pod-security.kubernetes.io/enforce=privileged --overwrite

# 4 виртуальных слайса на 1 физическую GPU
# Tesla T4: подходит для inference нагрузок
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |-
    version: v1
    sharing:
      timeSlicing:
        renameByDefault: false
        failRequestsGreaterThanOne: false
        resources:
        - name: nvidia.com/gpu
          replicas: 4
EOF
```

## Шаг 5.3 — Установка GPU Operator

**[VM1]**

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

cat > /root/gpu-operator-values.yaml <<'EOF'
operator:
  defaultRuntime: containerd
  runtimeClass: nvidia
  # [ТОЛЬКО ДЛЯ ПРОКСИ] Прокси для GPU Operator
  # При прямом доступе блок env ниже удалить
  env:
  - name: HTTPS_PROXY
    value: http://proxy.company.ru:3128
  - name: HTTP_PROXY
    value: http://proxy.company.ru:3128
  - name: NO_PROXY
    value: 10.0.0.0/8,localhost,127.0.0.1,.svc,.cluster.local

# Драйвер и toolkit установлены вручную
driver:    {enabled: false}
toolkit:   {enabled: false}

devicePlugin:
  enabled: true
  version: v0.17.4
  config:
    name: time-slicing-config
    default: any

gfd: {enabled: true, version: v0.17.4}
nfd: {enabled: true, nodefeaturerules: true}
dcgm: {enabled: false}
dcgmExporter:
  enabled: true
  version: 4.2.3-4.1.1-ubi9

migManager: {enabled: false}
mig:        {strategy: none}
cdi:        {enabled: true, default: false}

daemonsets:
  priorityClassName: system-node-critical
EOF

# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
helm upgrade --install gpu-operator nvidia/gpu-operator \
  -n gpu-operator \
  --version v26.3.1 \
  -f /root/gpu-operator-values.yaml \
  --wait --timeout 15m

# Проверяем слайсы GPU
kubectl get nodes -o json \
  | jq '.items[] | {name: .metadata.name, gpu: .status.capacity."nvidia.com/gpu"}'
# vm-1..vm-4: "4"  (4 слайса × 4 ноды = 16 виртуальных GPU итого)
```

---

# ЭТАП 6 — Longhorn

## Шаг 6.1 — Предустановка зависимостей Longhorn

**[ВСЕ VM]**

```bash
# Multipathd blacklist (критично для Longhorn на Ubuntu)
systemctl is-active multipathd >/dev/null 2>&1 && {
  mkdir -p /etc/multipath/conf.d
  cat > /etc/multipath/conf.d/longhorn.conf <<'EOF'
blacklist {
    devnode "^vd[a-z0-9]+"
    devnode "^sd[a-z0-9]+"
}
EOF
  systemctl restart multipathd
}

# Проверяем что все нужные утилиты есть
for cmd in iscsiadm mkfs.xfs mountpoint lsblk; do
  which $cmd || echo "ОТСУТСТВУЕТ: $cmd"
done

# Проверка совместимости Longhorn
curl -sSfL \
  https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/scripts/environment_check.sh \
  | bash 2>&1 | tail -20
```

## Шаг 6.2 — Установка Longhorn

**[VM1]**

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

# Помечаем ноды — Longhorn создаст диски автоматически
# create-default-disk=config означает что используем кастомный список дисков (не только дефолтный путь)
for n in vm-1 vm-2 vm-3 vm-4 vm-5 vm-6; do
  kubectl label node $n node.longhorn.io/create-default-disk=config --overwrite
done

kubectl create namespace longhorn-system

# Аннотируем ноды — указываем ОБА диска для Longhorn
# vdb → /var/lib/longhorn (основной, 18.2 TB)
# vdc → /var/lib/longhorn-extra (дополнительный, ~3.1 TB)
for n in vm-1 vm-2 vm-3 vm-4 vm-5 vm-6; do
  kubectl annotate node $n     node.longhorn.io/default-disks-config='[
      {
        "path":"/var/lib/longhorn",
        "allowScheduling":true,
        "storageReserved":10737418240,
        "tags":["primary"]
      },
      {
        "path":"/var/lib/longhorn-extra",
        "allowScheduling":true,
        "storageReserved":5368709120,
        "tags":["extra"]
      }
    ]' --overwrite
done

cat > /root/longhorn-values.yaml <<'EOF'
defaultSettings:
  defaultDataPath: /var/lib/longhorn
  createDefaultDiskLabeledNodes: true
  defaultReplicaCount: 2
  defaultDataLocality: best-effort
  replicaSoftAntiAffinity: true
  storageOverProvisioningPercentage: 150
  storageMinimalAvailablePercentage: 10
  upgradeChecker: false
  guaranteedInstanceManagerCPU: 8

persistence:
  defaultClass: false
  reclaimPolicy: Delete

csi:
  attacherReplicaCount: 3
  provisionerReplicaCount: 3
  resizerReplicaCount: 3
  snapshotterReplicaCount: 3

longhornUI:
  replicas: 2

ingress:
  enabled: false
EOF

# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
helm upgrade --install longhorn longhorn/longhorn \
  -n longhorn-system \
  --version 1.7.2 \
  -f /root/longhorn-values.yaml

kubectl -n longhorn-system rollout status ds/longhorn-manager --timeout=10m

# Проверяем что Longhorn видит оба диска на каждой ноде
# Ждём 2-3 минуты после старта — Longhorn обнаруживает диски асинхронно
echo "Ждём обнаружения дисков (60 сек)..."
sleep 60
kubectl -n longhorn-system get nodes.longhorn.io -o custom-columns='NODE:.metadata.name,DISKS:.spec.disks' 2>/dev/null || kubectl -n longhorn-system get nodes.longhorn.io

# Должно показать 2 диска на каждой ноде (primary + extra)
# Если диски не появились — проверяем что пути смонтированы внутри VM:
#   ssh ubuntu@vm-1 "df -hT /var/lib/longhorn /var/lib/longhorn-extra"

# StorageClasses
kubectl apply -f - <<'EOF'
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
  annotations: {storageclass.kubernetes.io/is-default-class: "true"}
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "2880"
  fsType: "ext4"
  dataLocality: "best-effort"
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: longhorn-single}
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "1"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  dataLocality: "strict-local"
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: longhorn-rwx}
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  fsType: "ext4"
  nfsOptions: "vers=4.1,noresvport"
EOF

kubectl get sc
```

---

# ЭТАП 7 — Инфраструктурные сервисы (namespace infra)

## Шаг 7.1 — Kafka (Strimzi KRaft)

**[VM1]**

```bash
kubectl create namespace infra
kubectl label namespace infra pod-security.kubernetes.io/enforce=privileged --overwrite

helm repo add strimzi https://strimzi.io/charts/
helm repo update

# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
helm upgrade --install strimzi-kafka-operator strimzi/strimzi-kafka-operator \
  --version 0.45.0 \
  -n infra \
  --set watchNamespaces='{infra}' \
  --set defaultImageTag=0.45.0

kubectl apply -f - <<'EOF'
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: combined
  namespace: infra
  labels: {strimzi.io/cluster: kafka}
spec:
  replicas: 3
  roles: [controller, broker]
  storage:
    type: jbod
    volumes:
    - id: 0
      type: persistent-claim
      size: 300Gi
      class: longhorn
      deleteClaim: false
  resources:
    requests: {memory: 6Gi, cpu: "2"}
    limits:   {memory: 8Gi, cpu: "4"}
  template:
    pod:
      # Kafka на storage-нодах — не мешает GPU нагрузкам
      # Но у нас только 2 storage-ноды и 3 брокера
      # Третий брокер пойдёт на GPU-ноду (soft anti-affinity)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels: {strimzi.io/cluster: kafka}
              topologyKey: kubernetes.io/hostname
      nodeSelector:
        storage: "true"
---
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka
  namespace: infra
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 3.8.1
    metadataVersion: 3.8-IV0
    listeners:
    - name: plain
      port: 9092
      type: internal
      tls: false
    config:
      offsets.topic.replication.factor: "2"
      transaction.state.log.replication.factor: "2"
      transaction.state.log.min.isr: "1"
      default.replication.factor: "2"
      min.insync.replicas: "1"
      num.partitions: "2"
      auto.create.topics.enable: "false"
      message.max.bytes: "1073741824"
      replica.fetch.max.bytes: "1073741824"
      max.request.size: "1073741824"
      socket.request.max.bytes: "1073741824"
      log.retention.hours: "168"
    jvmOptions:
      "-Xmx": 6g
      "-Xms": 6g
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF

kubectl -n infra get kafka kafka -w   # ждём Ready=True
```

## Шаг 7.2 — MinIO

**[VM1]**

```bash
helm repo add minio-operator https://operator.min.io/
helm repo update

# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
helm upgrade --install minio-operator minio-operator/operator \
  --version 6.0.4 \
  -n infra \
  --set operator.replicaCount=2

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata: {name: minio-env-config, namespace: infra}
type: Opaque
stringData:
  config.env: |
    export MINIO_ROOT_USER="admin"
    export MINIO_ROOT_PASSWORD="ChangeMe-SuperSecure-2026"
---
apiVersion: minio.min.io/v2
kind: Tenant
metadata: {name: minio, namespace: infra}
spec:
  image: quay.io/minio/minio:RELEASE.2024-11-07T00-52-20Z
  configuration: {name: minio-env-config}
  pools:
  - servers: 4
    name: pool-0
    volumesPerServer: 1
    volumeClaimTemplate:
      metadata: {name: data}
      spec:
        storageClassName: longhorn-single
        accessModes: [ReadWriteOnce]
        resources: {requests: {storage: 8Ti}}
    resources:
      requests: {memory: 4Gi, cpu: "2"}
      limits:   {memory: 8Gi, cpu: "4"}
    nodeSelector:
      gpu: "true"
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels: {v1.min.io/tenant: minio}
          topologyKey: kubernetes.io/hostname
  requestAutoCert: false
  mountPath: /export
EOF

kubectl -n infra get tenant minio -w
```

## Шаг 7.3 — MongoDB ReplicaSet

**[VM1]**

```bash
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update

# [ПРОКСИ] http_proxy=http://proxy.company.ru:3128 \ # убрать эту строку при прямом доступе
helm upgrade --install community-operator mongodb/community-operator \
  --version 0.11.0 \
  -n infra \
  --set operator.watchNamespace=infra

kubectl create secret generic mongodb-admin-password \
  -n infra \
  --from-literal=password='ChangeMe-Mongo-2026'

kubectl apply -f - <<'EOF'
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata: {name: mongodb, namespace: infra}
spec:
  members: 3
  type: ReplicaSet
  version: "7.0.14"
  security:
    authentication: {modes: [SCRAM]}
  users:
  - name: admin
    db: admin
    passwordSecretRef: {name: mongodb-admin-password}
    roles:
    - {name: clusterAdmin,          db: admin}
    - {name: userAdminAnyDatabase,  db: admin}
    - {name: readWriteAnyDatabase,  db: admin}
    scramCredentialsSecretName: mongodb-admin-scram
  statefulSet:
    spec:
      template:
        spec:
          affinity:
            podAntiAffinity:
              # Мягкий антиаффинити — допускаем 2 реплики на одной ноде
              # т.к. у нас только 2 storage-ноды для 3 реплик
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchLabels: {app: mongodb-svc}
                  topologyKey: kubernetes.io/hostname
  volumeClaimTemplates:
  - metadata: {name: data-volume}
    spec:
      storageClassName: longhorn
      accessModes: [ReadWriteOnce]
      resources: {requests: {storage: 100Gi}}
  - metadata: {name: logs-volume}
    spec:
      storageClassName: longhorn
      accessModes: [ReadWriteOnce]
      resources: {requests: {storage: 10Gi}}
EOF

kubectl -n infra get mdbc mongodb -w   # ждём Phase=Running
```

## Шаг 7.4 — Redis

**[VM1]**

```bash
kubectl apply -n infra -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata: {name: redis-config, namespace: infra}
data:
  redis.conf: |
    save ""
    appendonly no
    maxmemory 2gb
    maxmemory-policy allkeys-lru
    protected-mode no
    tcp-keepalive 300
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: redis, namespace: infra}
spec:
  replicas: 1
  selector: {matchLabels: {app: redis}}
  template:
    metadata: {labels: {app: redis}}
    spec:
      containers:
      - name: redis
        image: redis:7.4-alpine
        command: [redis-server, /etc/redis/redis.conf]
        ports: [{containerPort: 6379}]
        resources:
          requests: {memory: 256Mi, cpu: 100m}
          limits:   {memory: 3Gi,   cpu: "2"}
        volumeMounts:
        - {mountPath: /etc/redis, name: config}
      volumes:
      - name: config
        configMap: {name: redis-config}
---
apiVersion: v1
kind: Service
metadata: {name: redis, namespace: infra}
spec:
  selector: {app: redis}
  ports: [{port: 6379, targetPort: 6379}]
EOF

kubectl -n infra exec deploy/redis -- redis-cli ping
# PONG
```

---

# ЭТАП 8 — Приложения (namespace production)

## Шаг 8.1 — Registry secret

**[VM1]**

```bash
kubectl create namespace production

kubectl create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.example.com \
  --docker-username='gitlab+deploy-token-1' \
  --docker-password='glpat-xxxxxxxxxxxxxxxxxx' \
  -n production

kubectl patch serviceaccount default -n production \
  -p '{"imagePullSecrets":[{"name":"gitlab-registry"}]}'
```

## Шаг 8.2 — Секреты приложений из .env файлов

**[VM1]** — предварительно скопируйте .env файлы со старых серверов

```bash
# Пример копирования с продового сервера:
# scp -r user@old-server-1:/server/ /tmp/old-server-configs/

kubectl create secret generic tor-env \
  -n production --from-env-file=/tmp/old-server-configs/tor/.env

kubectl create secret generic reader-env \
  -n production --from-env-file=/tmp/old-server-configs/reader/.env

kubectl create secret generic feature-env \
  -n production --from-env-file=/tmp/old-server-configs/feature/.env.1

kubectl create secret generic minio-handler-a-env \
  -n production --from-env-file=/tmp/old-server-configs/minio-handler/.env.v21

kubectl create secret generic minio-handler-b-env \
  -n production --from-env-file=/tmp/old-server-configs/minio-handler/.env.v22

kubectl create secret generic streams-env \
  -n production --from-env-file=/tmp/old-server-configs/streams/.env

kubectl create secret generic cameracolor-env \
  -n production --from-env-file=/tmp/old-server-configs/cameracolor/.env

kubectl create secret generic check-env \
  -n production --from-env-file=/tmp/old-server-configs/check-zaslon/.env

kubectl create secret generic monit-env \
  -n production --from-env-file=/tmp/old-server-configs/monit/.env

kubectl create secret generic car-counter-env \
  -n production --from-env-file=/tmp/old-server-configs/car-counter/.env

kubectl create secret generic controller-env \
  -n production --from-env-file=/tmp/old-server-configs/controller/.env
```

## Шаг 8.3 — Helm чарт

Структура `charts/app-template/` как в предыдущей версии гайда. Деплой:

```bash
CHART=charts/app-template
NS=production

# GPU-приложения (нужен time-slicing, по 1 слайсу)
helm upgrade --install tor    $CHART -f $CHART/envs/values-tor.yaml    -n $NS --wait --atomic
helm upgrade --install reader $CHART -f $CHART/envs/values-reader.yaml -n $NS --wait --atomic
helm upgrade --install feature $CHART -f $CHART/envs/values-feature.yaml -n $NS --wait --atomic
helm upgrade --install minio-handler-a $CHART -f $CHART/envs/values-minio-handler-a.yaml -n $NS --wait --atomic
helm upgrade --install minio-handler-b $CHART -f $CHART/envs/values-minio-handler-b.yaml -n $NS --wait --atomic
helm upgrade --install check  $CHART -f $CHART/envs/values-check.yaml  -n $NS --wait --atomic

# CPU-приложения
helm upgrade --install streams  $CHART -f $CHART/envs/values-streams.yaml  -n $NS --wait --atomic
helm upgrade --install cameracolor $CHART -f $CHART/envs/values-cameracolor.yaml -n $NS --wait --atomic
helm upgrade --install monit    $CHART -f $CHART/envs/values-monit.yaml    -n $NS --wait --atomic
helm upgrade --install car-counter $CHART -f $CHART/envs/values-car-counter.yaml -n $NS --wait --atomic
helm upgrade --install controller $CHART -f $CHART/envs/values-controller.yaml -n $NS --wait --atomic
helm upgrade --install parking-support $CHART -f $CHART/envs/values-parking-support.yaml -n $NS --wait --atomic
```

---

# ЭТАП 9 — GitLab CI/CD

## Шаг 9.1 — RBAC и kubeconfig

**[VM1]**

```bash
kubectl apply -n production -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata: {name: gitlab-deployer, namespace: production}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: gitlab-deployer, namespace: production}
rules:
- apiGroups: ["", "apps", "batch", "networking.k8s.io"]
  resources:
  - pods
  - services
  - configmaps
  - secrets
  - persistentvolumeclaims
  - deployments
  - statefulsets
  - daemonsets
  - jobs
  verbs: [get, list, watch, create, update, patch, delete]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: gitlab-deployer, namespace: production}
subjects: [{kind: ServiceAccount, name: gitlab-deployer, namespace: production}]
roleRef:  {kind: Role, name: gitlab-deployer, apiGroup: rbac.authorization.k8s.io}
---
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-deployer-token
  namespace: production
  annotations: {kubernetes.io/service-account.name: gitlab-deployer}
type: kubernetes.io/service-account-token
EOF

# Генерируем kubeconfig
SA_TOKEN=$(kubectl -n production get secret gitlab-deployer-token \
  -o jsonpath='{.data.token}' | base64 -d)
CA_CERT=$(kubectl -n production get secret gitlab-deployer-token \
  -o jsonpath='{.data.ca\.crt}')

cat > /root/kubeconfig-gitlab.yaml <<EOF
apiVersion: v1
kind: Config
clusters:
- name: prod
  cluster:
    server: https://k8s-api:6443
    certificate-authority-data: ${CA_CERT}
contexts:
- name: gitlab@prod
  context: {cluster: prod, namespace: production, user: gitlab-deployer}
current-context: gitlab@prod
users:
- name: gitlab-deployer
  user: {token: ${SA_TOKEN}}
EOF
chmod 600 /root/kubeconfig-gitlab.yaml
echo "Kubeconfig готов: /root/kubeconfig-gitlab.yaml"
echo "Загрузите его в GitLab CI/CD → Variables → KUBECONFIG (тип File)"
```

---

# ЭТАП 10 — Смена IP при переезде в новый ЦОД

## Что нужно сделать до выключения серверов

**[VM1]** — резервная копия

```bash
TS=$(date +%Y%m%d-%H%M%S)
# Бэкап etcd
kubectl -n kube-system exec etcd-vm-1 -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/snapshot-${TS}.db

# Бэкап конфигов
tar czf /root/kube-backup-${TS}.tgz \
  /etc/kubernetes /var/lib/kubelet /var/lib/etcd

# Манифест всего кластера
kubectl get all,cm,secret,pv,pvc,sa,role,rolebinding -A -o yaml \
  > /root/cluster-backup-${TS}.yaml
```

## После переезда — пошаговая процедура

```bash
# Переменные — заполнить перед запуском
OLD_VIP="10.10.10.20"
NEW_VIP="10.20.30.20"
OLD_NET="10.10.10"
NEW_NET="10.20.30"

# ШАГ 1: На каждой VM обновляем /etc/hosts
for node_ip in 21 22 23 24 25 26; do
  ssh ubuntu@${NEW_NET}.${node_ip} "
    sudo sed -i 's/${OLD_NET}\./${NEW_NET}./g' /etc/hosts
    echo '--- /etc/hosts после замены ---'
    grep -E 'k8s|vm-' /etc/hosts
  "
done

# ШАГ 2: Останавливаем kubelet на всех нодах
for node_ip in 21 22 23 24 25 26; do
  ssh ubuntu@${NEW_NET}.${node_ip} "sudo systemctl stop kubelet"
done

# ШАГ 3: Обновляем kube-vip на CP нодах
for node_ip in 21 22 23; do
  ssh ubuntu@${NEW_NET}.${node_ip} "
    sudo sed -i 's/${OLD_VIP}/${NEW_VIP}/g' \
      /etc/kubernetes/manifests/kube-vip.yaml
  "
done

# ШАГ 4: Перевыпускаем сертификаты на каждой CP-ноде
# Сначала создаём kubeadm-new.yaml (пример для vm-1)
for i in 1 2 3; do
  NODE_IP="${NEW_NET}.$((20+i))"
  ssh ubuntu@$NODE_IP "sudo bash -s" <<ENDSSH
cat > /root/kubeadm-new.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.32.3
controlPlaneEndpoint: "k8s-api:6443"
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
apiServer:
  certSANs:
  - k8s-api
  - ${OLD_NET}.20
  - ${NEW_NET}.20
  - ${OLD_NET}.2${i}
  - ${NEW_NET}.2${i}
  - vm-${i}
  - kubernetes
  - kubernetes.default.svc
  - kubernetes.default.svc.cluster.local
  - 127.0.0.1
EOF

cd /etc/kubernetes/pki
for f in apiserver.crt apiserver.key \
          etcd/server.crt etcd/server.key \
          etcd/peer.crt etcd/peer.key; do
  mv \$f \${f}.old 2>/dev/null || true
done
kubeadm init phase certs apiserver   --config /root/kubeadm-new.yaml
kubeadm init phase certs etcd-server --config /root/kubeadm-new.yaml
kubeadm init phase certs etcd-peer   --config /root/kubeadm-new.yaml
echo "Сертификаты vm-${i} перевыпущены"
openssl x509 -in apiserver.crt -noout -text \
  | grep -A3 "Subject Alternative Name"
ENDSSH
done

# ШАГ 5: Обновляем node-ip в kubelet на всех нодах
for i in 1 2 3 4 5 6; do
  NODE_IP="${NEW_NET}.$((20+i))"
  OLD_IP="${OLD_NET}.$((20+i))"
  ssh ubuntu@$NODE_IP "
    sudo sed -i 's|--node-ip=${OLD_IP}|--node-ip=${NODE_IP}|' \
      /var/lib/kubelet/kubeadm-flags.env
    echo 'kubelet flags:'
    cat /var/lib/kubelet/kubeadm-flags.env
  "
done

# ШАГ 6: Обновляем static pod манифесты на CP нодах
for i in 1 2 3; do
  NODE_IP="${NEW_NET}.$((20+i))"
  ssh ubuntu@$NODE_IP "
    sudo sed -i 's|${OLD_NET}\.|${NEW_NET}.|g' \
      /etc/kubernetes/manifests/etcd.yaml \
      /etc/kubernetes/manifests/kube-apiserver.yaml
  "
done

# ШАГ 7: Стартуем vm-1 первой
ssh ubuntu@${NEW_NET}.21 "sudo systemctl start kubelet"
echo "Ждём 45 секунд..."
sleep 45

# ШАГ 8: Обновляем etcd peer URLs
ssh ubuntu@${NEW_NET}.21 "
  sudo kubectl -n kube-system exec etcd-vm-1 -- etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    member list -w table
"
# Скопировать ID из вывода и обновить peer URLs:
# etcdctl member update <ID1> --peer-urls=https://${NEW_NET}.21:2380
# etcdctl member update <ID2> --peer-urls=https://${NEW_NET}.22:2380
# etcdctl member update <ID3> --peer-urls=https://${NEW_NET}.23:2380

# ШАГ 9: Стартуем остальные CP и воркеры
for node_ip in 22 23 24 25 26; do
  ssh ubuntu@${NEW_NET}.${node_ip} "sudo systemctl start kubelet"
  sleep 15
done

# ШАГ 10: Обновляем ConfigMaps
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl -n kube-system get cm kube-proxy -o yaml \
  | sed "s|${OLD_VIP}|${NEW_VIP}|g" \
  | kubectl apply -f -
kubectl -n kube-system rollout restart ds/kube-proxy
kubectl -n kube-public get cm cluster-info -o yaml \
  | sed "s|${OLD_VIP}|${NEW_VIP}|g" \
  | kubectl apply -f -

# ШАГ 11: Перезапускаем Calico
kubectl -n calico-system rollout restart ds/calico-node
kubectl get tigerastatus

# Финальная проверка
kubectl get nodes -o wide
curl -k https://${NEW_VIP}:6443/healthz
kubeadm certs check-expiration
```

---

# Текущие известные проблемы и решения

| # | Проблема | Решение в этом гайде |
|---|---|---|
| 1 | Ubuntu + GPU passthrough зависает при загрузке | `video=vesafb:off,efifb:off` в GRUB в VM |
| 2 | containerd не видит прокси | systemd drop-in `/etc/systemd/system/containerd.service.d/proxy.conf` |
| 3 | kubeadm images pull не через прокси | `Environment` в kubelet service.d + прокси в `~/.bashrc` |
| 4 | certificate-key для join истекает за 2 часа | `kubeadm init phase upload-certs --upload-certs` для регенерации |
| 5 | etcd на CP-нодах + рабочие нагрузки | Допустимо при нашем масштабе; taints убраны с CP-нод |
| 6 | 3 реплики MongoDB на 2 storage-нодах | `preferredAntiAffinity` — MongoDB переживает 2 реплики на одной ноде |
| 7 | Kafka 3 брокера на 2 storage-нодах | `preferredAntiAffinity` — 3-й брокер может пойти на GPU-ноду |
| 8 | 25 GPU нужно, 4 доступно | Time-slicing: 4 слайса × 4 GPU = 16 виртуальных; feature: 4 реплики вместо 16 |
| 9 | hostNetwork конфликты портов | Убрано, используем ClusterIP + NodePort для внешнего доступа |
| 10 | MinIO занимает RAM/CPU GPU-нод | ResourceRequests + Limits; MinIO на GPU-нодах т.к. там нет 4 других серверов |
| 11 | VM не поднимается если bridge не готов | Задержка 60 сек в watchdog-сервисе до первой проверки |
| 12 | GPU недоступен на хосте после VFIO | Ожидаемо — хост передаёт GPU в VM полностью |
| 13 | IOMMU-группа содержит несколько устройств | Пробрасываем всю группу (GPU + Audio) |
| 14 | no_proxy не настроен для внутренней сети | Прописан во всех местах: environment, apt, containerd, kubelet |
| 15 | Используется только 1 диск данных из 2 доступных | Шаг 2.1: раздел ~3.1 TB из системного диска → vdc в VM → /var/lib/longhorn-extra → Longhorn аннотация с двумя путями |

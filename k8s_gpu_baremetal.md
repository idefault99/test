# Производственное развертывание Kubernetes 1.32 на bare-metal RedOS 8

> **Статус документа:** production-ready, апрель 2026. Все команды проверены для RedOS 8 с ядром 6.12.x.
> 
> **⚠️ Важные замечания по версиям (проверено на апрель 2026):**
> 
> - **Kubernetes 1.32 достиг EOL 28.02.2026.** Последний патч — **v1.32.13** (26.02.2026).  Для long-term production рекомендуется миграция на 1.33.10+ или 1.34.x. Руководство использует 1.32.13 согласно ТЗ.
> - **Longhorn 1.7.x — EOL (04.09.2025).** Использовать **v1.10.2** (Prime stable). Минорные переходы без пропуска версий.
> - **Strimzi 0.45.x — EOL.** Актуальная стабильная — **0.51.0** (поддерживает Kafka 4.1.1, ZooKeeper удалён, KRaft обязателен).
> - **MongoDB Community Operator (`mongodb-kubernetes-operator`) архивирован 12.12.2025.**  Использовать новый унифицированный `mongodb-kubernetes` (v1.x).
> - **kube-vip v0.9.x:** последняя — **v0.9.2** (24.06.2025).  Критичный workaround с `super-admin.conf` обязателен для K8s ≥1.29.
> - **GPU Operator:** v26.3.1. NVIDIA Container Toolkit — **1.19.0** (избегать 1.18.0 — баг с `imports` в containerd config).
> - **Calico v3.29.5** — последний патч ветки 3.29, поддерживает ядро 6.12 в iptables/VXLAN dataplane.
> - На RedOS 8 (ядро 6.12) **прекомпилированные kmod NVIDIA для RHEL 8 (4.18) НЕ работают** — только DKMS.

-----

## Фаза 0. Настройка сети и исходные данные

### Топология кластера

|Сервер  |IP             |Роль             |GPU     |Диски                            |
|--------|---------------|-----------------|--------|---------------------------------|
|server-1|10.10.10.21    |CP + Worker + GPU|Tesla T4|sda (3.5TB OS), sdb (18.2TB data)|
|server-2|10.10.10.22    |CP + Worker + GPU|Tesla T4|sda, sdb                         |
|server-3|10.10.10.23    |CP + Worker + GPU|Tesla T4|sda, sdb                         |
|server-4|10.10.10.24    |Worker + GPU     |Tesla T4|sda, sdb                         |
|server-5|10.10.10.25    |Worker + Storage |—       |sda, sdb                         |
|server-6|10.10.10.26    |Worker + Storage |—       |sda, sdb                         |
|**VIP** |**10.10.10.20**|**kube-vip ARP** |—       |—                                |

- **PodCIDR:** 10.244.0.0/16
- **ServiceCIDR:** 10.96.0.0/12
- **controlPlaneEndpoint:** `k8s-api:6443` (через /etc/hosts, указывает на VIP)
- **GPU капасити:** 4 × Tesla T4 × 4 time-slice = **16 виртуальных GPU**
- **Хранилище:** 6 × 18.2 TB = 109.2 TB raw → ~54 TB полезно (replica=2)

### [ВСЕ СЕРВЕРЫ] Настройка /etc/hosts

```bash
cat <<'EOF' | sudo tee -a /etc/hosts
10.10.10.20  k8s-api
10.10.10.21  server-1
10.10.10.22  server-2
10.10.10.23  server-3
10.10.10.24  server-4
10.10.10.25  server-5
10.10.10.26  server-6
EOF
```

### [ВСЕ СЕРВЕРЫ] Hostname и отключение eno2

```bash
# На каждом сервере своё имя:
sudo hostnamectl set-hostname server-1  # ..server-6 соответственно

# Отключаем неиспользуемый интерфейс
sudo nmcli con down 'eno2' 2>/dev/null || true
sudo nmcli device set eno2 managed no 2>/dev/null || true

# Проверка
ip -br addr show | grep -E 'eno1|eno2'
```

### [ВСЕ СЕРВЕРЫ] Переменные окружения (прокси или прямой доступ)

```bash
# [ПРОКСИ] Вариант с корпоративным прокси
sudo tee /etc/profile.d/proxy.sh <<'EOF'
# [ТОЛЬКО ДЛЯ ПРОКСИ]
export http_proxy="http://proxy.company.ru:3128"
export https_proxy="http://proxy.company.ru:3128"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export no_proxy="localhost,127.0.0.1,::1,10.10.10.0/24,10.244.0.0/16,10.96.0.0/12,.svc,.svc.cluster.local,.cluster.local,k8s-api,server-1,server-2,server-3,server-4,server-5,server-6"
export NO_PROXY="$no_proxy"
EOF
sudo chmod 644 /etc/profile.d/proxy.sh
source /etc/profile.d/proxy.sh

# [ТОЛЬКО ДЛЯ ПРОКСИ] системная среда
sudo tee -a /etc/environment <<'EOF'
http_proxy=http://proxy.company.ru:3128
https_proxy=http://proxy.company.ru:3128
HTTP_PROXY=http://proxy.company.ru:3128
HTTPS_PROXY=http://proxy.company.ru:3128
no_proxy=localhost,127.0.0.1,10.10.10.0/24,10.244.0.0/16,10.96.0.0/12,.svc,.cluster.local
NO_PROXY=localhost,127.0.0.1,10.10.10.0/24,10.244.0.0/16,10.96.0.0/12,.svc,.cluster.local
EOF

# [БЕЗ ПРОКСИ] — файл /etc/profile.d/proxy.sh не создавать, /etc/environment не трогать
```

**Проверка:** `ping -c1 k8s-api` должен разрешиться в `10.10.10.20` (VIP ещё не поднят — это нормально, `ping` завершится ошибкой, но DNS резолвится).

-----

## Фаза 1. Подготовка ОС (RedOS 8, ядро 6.12)

### [ВСЕ СЕРВЕРЫ] Обновление системы и базовые пакеты

```bash
# [ПРОКСИ] — настройка dnf
# [ТОЛЬКО ДЛЯ ПРОКСИ]
sudo sed -i '/^proxy=/d' /etc/dnf/dnf.conf
echo 'proxy=http://proxy.company.ru:3128' | sudo tee -a /etc/dnf/dnf.conf

# Актуализация ядра 6.12 (если ещё не установлено)
sudo -E dnf install -y redos-kernels6-release
sudo -E dnf update -y
sudo -E dnf install -y \
  curl wget git tar rsync jq tree vim-enhanced \
  dnf-plugins-core policycoreutils-python-utils \
  iproute-tc ipvsadm socat conntrack-tools ethtool \
  chrony firewalld NetworkManager-tui \
  kernel-devel-$(uname -r) kernel-headers-$(uname -r) \
  gcc make dkms \
  iscsi-initiator-utils nfs-utils cryptsetup device-mapper \
  container-selinux

# Проверка версии ядра — должна быть 6.12.x.red80
uname -r
```

### [ВСЕ СЕРВЕРЫ] Отключение zram-swap (специфика RedOS 8)

RedOS 8 по умолчанию включает zram-swap — **обязательно** отключить для kubelet:

```bash
sudo systemctl stop dev-zram0.swap 2>/dev/null || true
sudo systemctl mask dev-zram0.swap
sudo swapoff -a
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab

# Проверка — должно быть пусто
swapon --show
free -h | grep -i swap
```

### [ВСЕ СЕРВЕРЫ] systemd-resolved — отключение stub listener

RedOS 8 включает `systemd-resolved`, который конфликтует с CoreDNS: 

```bash
sudo sed -i 's|^#\?DNSStubListener=.*|DNSStubListener=no|' /etc/systemd/resolved.conf
sudo systemctl daemon-reload
sudo systemctl restart systemd-resolved

# Убедиться, что /etc/resolv.conf содержит реальные DNS, а не 127.0.0.53:
cat /etc/resolv.conf
```

### [ВСЕ СЕРВЕРЫ] SELinux → permissive

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
getenforce   # → Permissive
```

**Почему permissive, а не disabled:** kubelet, CSI-драйверы и container-selinux требуют, чтобы подсистема была активна для корректной маркировки томов. В disabled режиме `seLinuxOptions` в podSpec молча игнорируются, а ReadWriteOncePod SELinux mount context (beta в 1.32) не работает.

### [ВСЕ СЕРВЕРЫ] Модули ядра и sysctl

```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
iscsi_tcp
dm_crypt
EOF
sudo modprobe overlay br_netfilter ip_vs ip_vs_rr nf_conntrack iscsi_tcp dm_crypt

cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv4.conf.all.rp_filter         = 0
net.ipv4.neigh.default.gc_thresh1   = 4096
net.ipv4.neigh.default.gc_thresh2   = 8192
net.ipv4.neigh.default.gc_thresh3   = 16384
fs.inotify.max_user_instances       = 8192
fs.inotify.max_user_watches         = 524288
vm.swappiness                       = 0
vm.overcommit_memory                = 1
vm.panic_on_oom                     = 0
kernel.panic                        = 10
kernel.panic_on_oops                = 1
EOF
sudo sysctl --system
```

### [ВСЕ СЕРВЕРЫ] firewalld — правила для Kubernetes

**Прагматичный подход: доверенные сети.** Такой подход обычно используется в однородных bare-metal кластерах:

```bash
sudo systemctl enable --now firewalld

# Добавляем внутренние сети в доверенную зону
sudo firewall-cmd --permanent --zone=trusted --add-source=10.10.10.0/24
sudo firewall-cmd --permanent --zone=trusted --add-source=10.244.0.0/16
sudo firewall-cmd --permanent --zone=trusted --add-source=10.96.0.0/12

# NodePort для внешних клиентов в public зоне
sudo firewall-cmd --permanent --add-port=30000-32767/tcp

sudo firewall-cmd --reload
sudo firewall-cmd --list-all --zone=trusted
```

**ARP-режим kube-vip не требует дополнительных портов** — ARP обрабатывается ядром ниже netfilter.

### [ВСЕ СЕРВЕРЫ] chrony (синхронизация времени)

```bash
sudo systemctl enable --now chronyd
chronyc tracking
chronyc sources
```

### [ВСЕ СЕРВЕРЫ] Установка containerd

RedOS 8 поставляет `containerd` в официальном репозитории:

```bash
sudo dnf install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# SystemdCgroup = true (обязательно для kubeadm 1.32)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Правка пути CNI — специфика RedOS 8 (/usr/libexec/cni вместо /opt/cni/bin)
sudo sed -i 's|bin_dir = "/opt/cni/bin"|bin_dir = "/usr/libexec/cni"|g' /etc/containerd/config.toml
sudo mkdir -p /opt/cni/bin
sudo ln -sf /usr/libexec/cni/* /opt/cni/bin/ 2>/dev/null || true

# Обновление pause-образа для 1.32
sudo sed -i 's#sandbox_image = "registry.k8s.io/pause:3.8"#sandbox_image = "registry.k8s.io/pause:3.10"#' /etc/containerd/config.toml
```

### [ВСЕ СЕРВЕРЫ] Прокси для containerd

```bash
# [ТОЛЬКО ДЛЯ ПРОКСИ]
sudo mkdir -p /etc/systemd/system/containerd.service.d
sudo tee /etc/systemd/system/containerd.service.d/http-proxy.conf <<'EOF'
[Service]
Environment="HTTP_PROXY=http://proxy.company.ru:3128"
Environment="HTTPS_PROXY=http://proxy.company.ru:3128"
Environment="NO_PROXY=localhost,127.0.0.1,10.10.10.0/24,10.244.0.0/16,10.96.0.0/12,.svc,.cluster.local,registry.red-soft.ru"
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl restart containerd

# Проверка
sudo ctr version
sudo systemctl status containerd --no-pager
```

### [ВСЕ СЕРВЕРЫ] Репозиторий Kubernetes и установка kubeadm/kubelet/kubectl

```bash
cat <<'EOF' | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo dnf install -y --disableexcludes=kubernetes kubelet-1.32.13 kubeadm-1.32.13 kubectl-1.32.13
sudo systemctl enable kubelet

# Прокси для kubelet (image-pull через containerd, но kubelet тоже может ходить наружу)
# [ТОЛЬКО ДЛЯ ПРОКСИ]
sudo mkdir -p /etc/systemd/system/kubelet.service.d
sudo tee /etc/systemd/system/kubelet.service.d/http-proxy.conf <<'EOF'
[Service]
Environment="HTTP_PROXY=http://proxy.company.ru:3128"
Environment="HTTPS_PROXY=http://proxy.company.ru:3128"
Environment="NO_PROXY=localhost,127.0.0.1,10.10.10.0/24,10.244.0.0/16,10.96.0.0/12,.svc,.cluster.local"
EOF
sudo systemctl daemon-reload
```

**Проверка фазы 1:**

```bash
uname -r                                # 6.12.x.red80
getenforce                              # Permissive
swapon --show                           # пусто
systemctl is-active containerd kubelet  # active (kubelet может быть activating — это нормально до init)
lsmod | grep -E 'overlay|br_netfilter|iscsi_tcp'
```

-----

## Фаза 2. NVIDIA драйвер 570 DKMS + подготовка диска

### [GPU СЕРВЕРЫ] (server-1..4) Blacklist nouveau

```bash
cat <<'EOF' | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF

sudo grubby --args "rd.driver.blacklist=nouveau modprobe.blacklist=nouveau nouveau.modeset=0" --update-kernel ALL
sudo dracut --force --regenerate-all
sudo reboot
```

После перезагрузки:

```bash
lsmod | grep -i nouveau   # должно быть пусто
```

### [GPU СЕРВЕРЫ] Установка NVIDIA драйвера 570 open-DKMS

На RedOS 8 с ядром 6.12 **прекомпилированные kmod NVIDIA из cuda-rhel8 репозитория НЕ работают** (собраны под 4.18). Только DKMS.

```bash
# Репозиторий CUDA NVIDIA для RHEL 8
sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo

# Импорт ключа вручную (на случай проблем с gpgcheck через прокси)
sudo rpm --import https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/D42D0685.pub

# Сброс модулей и установка open-DKMS потока 570
sudo dnf module reset -y nvidia-driver
sudo dnf module enable -y nvidia-driver:570-open-dkms
sudo dnf module install -y nvidia-driver:570-open-dkms

# Отключаем dnf-плагин NVIDIA, чтобы он не блокировал обновление ядра 6.12
sudo sed -i 's/^enabled=1/enabled=0/' /etc/yum.repos.d/nvidia-plugin*.repo 2>/dev/null || true

# Persistence daemon — критично для Tesla T4 (снижает init-latency)
sudo systemctl enable --now nvidia-persistenced

sudo reboot
```

После перезагрузки — **верификация**:

```bash
nvidia-smi
# Ожидается:
# +---------------------------------------------------------------+
# | NVIDIA-SMI 570.xx   Driver Version: 570.xx   CUDA Version: 12.8 |
# |---------------------------------+-----------------------------|
# |   0  Tesla T4                   On | 00000000:XX:00.0 Off |   0 |
# | N/A   35C    P8    9W /  70W |      0MiB / 15360MiB | 0%      Default |

dkms status
# nvidia-open/570.xxx.xx, 6.12.x.red80, x86_64: installed

# Проверка сборки модуля
modinfo nvidia | grep -E '^(version|filename):'
```

**Известная проблема ядра 6.12:** ранние драйверы 550.x не собирались с 6.12. Версии 550.135+ и вся ветка 570.x собираются штатно. При ошибке `implicit declaration` в `nv-dmabuf.c`  — обновите драйвер до последнего патча ветки 570 (`dnf update --allowerasing`).

### [GPU СЕРВЕРЫ] NVIDIA Container Toolkit 1.19.0

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo \
  | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo

export NCT=1.19.0-1
sudo dnf install -y \
  nvidia-container-toolkit-${NCT} \
  nvidia-container-toolkit-base-${NCT} \
  libnvidia-container-tools-${NCT} \
  libnvidia-container1-${NCT}

# Конфигурация containerd — НЕ делаем nvidia runtime default,
# чтобы не-GPU поды не ломались
sudo nvidia-ctk runtime configure --runtime=containerd --set-as-default=false
sudo systemctl restart containerd

# SELinux-модуль для NVIDIA-контейнеров (полезно даже в permissive)
curl -LO https://raw.githubusercontent.com/NVIDIA/dgx-selinux/master/bin/RHEL7/nvidia-container.pp
sudo semodule -i nvidia-container.pp

# Тест
sudo ctr image pull docker.io/nvidia/cuda:12.8.0-base-ubi8
sudo ctr run --rm --gpus 0 docker.io/nvidia/cuda:12.8.0-base-ubi8 test nvidia-smi
```

**Важно:** избегайте `nvidia-container-toolkit:1.18.0` — он перезаписывает поле `imports` в `/etc/containerd/config.toml`, что ломает GPU Operator. Используйте 1.18.1+ или 1.19.0.

### [ВСЕ СЕРВЕРЫ] Подготовка диска sdb под Longhorn

```bash
# Убедиться что sdb — это тот самый 18.2TB диск, а не OS-диск
lsblk -dno NAME,SIZE,TYPE,MODEL | grep -E 'sd[ab]'

# Форматирование в XFS
sudo mkfs.xfs -f -L longhorn /dev/sdb

UUID=$(sudo blkid -s UUID -o value /dev/sdb)
echo "UUID=$UUID"

sudo mkdir -p /var/lib/longhorn

# Запись в /etc/fstab
echo "UUID=$UUID  /var/lib/longhorn  xfs  defaults,noatime,nodiratime,inode64  0 0" | sudo tee -a /etc/fstab

sudo systemctl daemon-reload
sudo mount -a
df -hT /var/lib/longhorn

# SELinux контекст
sudo semanage fcontext -a -t container_file_t '/var/lib/longhorn(/.*)?'
sudo restorecon -RvF /var/lib/longhorn

# iSCSI initiator (нужен Longhorn)
sudo bash -c 'echo "InitiatorName=$(/sbin/iscsi-iname)" > /etc/iscsi/initiatorname.iscsi'
sudo systemctl enable --now iscsid
```

**Проверка фазы 2:**

```bash
# На GPU-серверах:
nvidia-smi && sudo ctr run --rm --gpus 0 docker.io/nvidia/cuda:12.8.0-base-ubi8 t nvidia-smi
# На всех серверах:
df -hT /var/lib/longhorn && systemctl is-active iscsid
```

-----

## Фаза 3. Развёртывание кластера Kubernetes 1.32

### [СЕРВЕР1] Подготовка kube-vip static pod (до kubeadm init)

```bash
export VIP=10.10.10.20
export INTERFACE=eno1
export KVVERSION=v0.9.2

sudo ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION

sudo mkdir -p /etc/kubernetes/manifests

sudo ctr run --rm --net-host \
  ghcr.io/kube-vip/kube-vip:$KVVERSION kv \
  /kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --arp \
    --leaderElection \
  | sudo tee /etc/kubernetes/manifests/kube-vip.yaml

# КРИТИЧНО: workaround для K8s ≥1.29 — подменяем admin.conf на super-admin.conf
# (до kubeadm init, чтобы kube-vip мог захватить leader lease, пока RBAC ещё не применён)
sudo sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' \
    /etc/kubernetes/manifests/kube-vip.yaml
```

### [СЕРВЕР1] kubeadm ClusterConfiguration

```bash
sudo tee /root/kubeadm-config.yaml <<'EOF'
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.10.10.21
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
    - name: node-ip
      value: 10.10.10.21
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.32.13
clusterName: k8s-prod
controlPlaneEndpoint: "k8s-api:6443"
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
  dnsDomain: cluster.local
apiServer:
  certSANs:
    - k8s-api
    - kubernetes
    - kubernetes.default.svc
    - 10.10.10.20
    - 10.10.10.21
    - 10.10.10.22
    - 10.10.10.23
    - 127.0.0.1
    - localhost
etcd:
  local:
    dataDir: /var/lib/etcd
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
serverTLSBootstrap: true
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: iptables
EOF

# Предварительный pull образов (важно при прокси — увидим ошибку заранее)
sudo kubeadm config images pull --config /root/kubeadm-config.yaml

# Инициализация кластера
sudo kubeadm init --config /root/kubeadm-config.yaml --upload-certs | tee /root/kubeadm-init.log
```

Сохраните из лога **два** join-команды: `kubeadm join ... --control-plane --certificate-key ...` (для CP нод) и обычную `kubeadm join ...` (для воркеров).

### [СЕРВЕР1] После init — откатываем super-admin.conf hack

```bash
# Настраиваем kubectl для root
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# ВАЖНО: вернуть admin.conf — super-admin.conf не реплицируется на другие CP
sudo sed -i 's#path: /etc/kubernetes/super-admin.conf#path: /etc/kubernetes/admin.conf#' \
    /etc/kubernetes/manifests/kube-vip.yaml

# Kubelet автоматически рестартанёт static pod — возможно краткое исчезновение VIP (3-5 сек)
kubectl get pod -n kube-system | grep kube-vip
ping -c3 10.10.10.20
```

### [СЕРВЕР1] Установка Calico v3.29.5 (VXLAN, BGP disabled)

```bash
# Tigera Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.5/manifests/tigera-operator.yaml

# Запрет NetworkManager управлять Calico-интерфейсами
sudo tee /etc/NetworkManager/conf.d/calico.conf <<'EOF'
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:wireguard.cali
EOF
sudo systemctl reload NetworkManager

# Установочный манифест
cat <<'EOF' | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
      - name: default-ipv4-ippool
        blockSize: 26
        cidr: 10.244.0.0/16
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()
    nodeAddressAutodetectionV4:
      interface: eno1
    mtu: 1450
  registry: quay.io/
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF

# Ожидание готовности
kubectl -n calico-system rollout status ds/calico-node --timeout=300s
kubectl wait --for=condition=Available --all apiservice -A --timeout=300s
kubectl get nodes -o wide   # server-1 должен стать Ready
```

### [CP СЕРВЕРЫ 2-3] Присоединение server-2 и server-3 как CP

**На каждом из server-2, server-3 сначала выполнить всё из Фазы 1 и 2.** Затем подготовить kube-vip static pod **точно так же, как на server-1** (включая `super-admin.conf` hack — но на join он **не нужен**, т.к. super-admin.conf не копируется, поэтому оставляем обычный `admin.conf`):

```bash
# На server-2 (аналогично server-3)
export VIP=10.10.10.20
export INTERFACE=eno1
export KVVERSION=v0.9.2
sudo ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
sudo mkdir -p /etc/kubernetes/manifests
sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION kv \
  /kube-vip manifest pod --interface $INTERFACE --address $VIP \
  --controlplane --arp --leaderElection \
  | sudo tee /etc/kubernetes/manifests/kube-vip.yaml

# Используем токен и cert-key из /root/kubeadm-init.log с server-1:
sudo kubeadm join k8s-api:6443 \
  --token <TOKEN_FROM_LOG> \
  --discovery-token-ca-cert-hash sha256:<HASH_FROM_LOG> \
  --control-plane --certificate-key <CERT_KEY_FROM_LOG>

# Настраиваем kubectl
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Если токен просрочен (24 часа), создать новый на server-1:

```bash
sudo kubeadm token create --print-join-command
sudo kubeadm init phase upload-certs --upload-certs   # выдаст новый certificate-key
```

### [WORKER СЕРВЕРЫ 4-6] Присоединение воркеров

```bash
# На server-4, server-5, server-6
sudo kubeadm join k8s-api:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### [СЕРВЕР1] Снятие taint с CP-нод (CP = Worker по ТЗ)

```bash
# CP-ноды по ТЗ также работают как воркеры, снимаем taint
kubectl taint nodes server-1 server-2 server-3 node-role.kubernetes.io/control-plane:NoSchedule- 2>/dev/null || true

kubectl get nodes -o wide
# Все 6 нод должны быть Ready
```

### [СЕРВЕР1] Лейблы нод

```bash
# GPU-ноды
for n in server-1 server-2 server-3 server-4; do
  kubectl label node $n gpu=true --overwrite
  kubectl label node $n node-role.kubernetes.io/gpu=true --overwrite
done

# Storage-ноды
for n in server-5 server-6; do
  kubectl label node $n storage=true --overwrite
  kubectl label node $n node-role.kubernetes.io/storage=true --overwrite
done

# Для Longhorn — все 6 нод хранят данные
for n in server-1 server-2 server-3 server-4 server-5 server-6; do
  kubectl label node $n node.longhorn.io/create-default-disk=true --overwrite
done

kubectl get nodes --show-labels | awk '{print $1, $6}'
```

### Установка Helm 3.20.2

```bash
# [ПРОКСИ]
# [ТОЛЬКО ДЛЯ ПРОКСИ]
export HTTPS_PROXY=http://proxy.company.ru:3128

# Устанавливаем именно 3.20.2 (get-helm-3 с main-веткой сейчас качает Helm 4 по умолчанию)
curl -fsSL https://raw.githubusercontent.com/helm/helm/dev-v3/scripts/get-helm-3 \
  | DESIRED_VERSION=v3.20.2 bash

helm version
```

**Проверка фазы 3:**

```bash
kubectl get nodes -o wide                       # 6 Ready нод
kubectl get pods -A | grep -v Running           # только Completed или пусто
kubectl -n kube-system get pod -l component=kube-vip
ping -c3 10.10.10.20                            # VIP отвечает
curl -k https://k8s-api:6443/healthz            # ok
```

-----

## Фаза 4. GPU Operator + Time-Slicing (4 slice на Tesla T4)

### [СЕРВЕР1] Установка GPU Operator v26.3.1 (driver/toolkit DISABLED)

Драйвер и toolkit установлены вручную на хостах — отключаем их в операторе:

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

kubectl create namespace gpu-operator

# ConfigMap time-slicing ДО установки оператора
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  tesla-t4: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        resources:
          - name: nvidia.com/gpu
            replicas: 4
EOF

helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --version v26.3.1 \
  --set driver.enabled=false \
  --set toolkit.enabled=false \
  --set devicePlugin.config.name=time-slicing-config \
  --set devicePlugin.config.default=tesla-t4 \
  --set cdi.enabled=true \
  --set cdi.default=true \
  --set validator.plugin.env[0].name=WITH_WORKLOAD \
  --set validator.plugin.env[0].value="true"
```

### Лейбл для активации time-slicing на GPU-нодах

```bash
kubectl label node -l nvidia.com/gpu.product=Tesla-T4 \
  nvidia.com/device-plugin.config=tesla-t4 --overwrite

# Если лейбл nvidia.com/gpu.product ещё не появился — подождите готовность gpu-feature-discovery:
kubectl -n gpu-operator rollout status ds/nvidia-device-plugin-daemonset --timeout=300s
```

**Проверка:**

```bash
kubectl get nodes -o json | jq -r '.items[] | select(.metadata.labels["gpu"]=="true") | .metadata.name + ": " + .status.capacity["nvidia.com/gpu"]'
# Ожидаемый вывод:
# server-1: 4
# server-2: 4
# server-3: 4
# server-4: 4
# Итого 16 виртуальных GPU

# Тест-под
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  restartPolicy: Never
  nodeSelector:
    gpu: "true"
  containers:
    - name: cuda
      image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda12.5.0
      resources:
        limits:
          nvidia.com/gpu: 1
EOF

kubectl logs gpu-test  # Test PASSED
kubectl delete pod gpu-test
```

-----

## Фаза 5. Longhorn v1.10.2

### [СЕРВЕР1] Установка Longhorn

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
kubectl create namespace longhorn-system

# Workaround для SELinux + iscsid
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.10.2/deploy/prerequisite/longhorn-iscsi-selinux-workaround.yaml

cat <<'EOF' > /root/longhorn-values.yaml
defaultSettings:
  defaultDataPath: "/var/lib/longhorn"
  defaultReplicaCount: 2
  createDefaultDiskLabeledNodes: true
  storageMinimalAvailablePercentage: 10
  storageOverProvisioningPercentage: 100
  replicaSoftAntiAffinity: false
  replicaAutoBalance: "best-effort"
  defaultDataLocality: "disabled"
  nodeDownPodDeletionPolicy: "delete-both-statefulset-and-deployment-pod"
  orphanResourceAutoDeletion: "replica-data"
  snapshotDataIntegrity: "fast-check"
  v2DataEngine: false
persistence:
  defaultClass: false           # мы создадим StorageClass вручную
  defaultClassReplicaCount: 2
  defaultFsType: xfs
  reclaimPolicy: Delete
csi:
  kubeletRootDir: /var/lib/kubelet
longhornUI:
  replicas: 2
EOF

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --version 1.10.2 \
  -f /root/longhorn-values.yaml

kubectl -n longhorn-system rollout status ds/longhorn-manager --timeout=600s
kubectl -n longhorn-system get pod
```

### [СЕРВЕР1] Две StorageClass

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  fsType: "xfs"
  dataLocality: "disabled"
  replicaAutoBalance: "best-effort"
  unmapMarkSnapChainRemoved: "ignored"
  disableRevisionCounter: "false"
  dataEngine: "v1"
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-single
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
parameters:
  numberOfReplicas: "1"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  fsType: "xfs"
  dataLocality: "strict-local"
  disableRevisionCounter: "true"
  dataEngine: "v1"
EOF

kubectl get storageclass
# longhorn (default)  driver.longhorn.io  Delete  Immediate
# longhorn-single      driver.longhorn.io  Delete  WaitForFirstConsumer
```

**Проверка:**

```bash
# Проверка нод в UI (через port-forward)
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80 &
# Браузер: http://<ssh-tunnel>:8080 → Node → все 6 нод Ready, disk ~18TB

# Smoke-test PVC
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  storageClassName: longhorn
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF
kubectl get pvc test-pvc   # Bound в течение 10-30 сек
kubectl delete pvc test-pvc
```

-----

## Фаза 6. Инфраструктурные сервисы

### 6.1. Kafka через Strimzi 0.51.0 (KRaft, на storage-нодах)

```bash
helm install strimzi-operator oci://quay.io/strimzi-helm/strimzi-kafka-operator \
  --version 0.51.0 \
  --namespace strimzi --create-namespace \
  --set watchAnyNamespace=true

kubectl create namespace kafka

cat <<'EOF' | kubectl apply -f -
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: combined
  namespace: kafka
  labels:
    strimzi.io/cluster: prod-kafka
spec:
  replicas: 3
  roles:
    - controller
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 200Gi
        class: longhorn
        deleteClaim: false
  resources:
    requests: { cpu: 500m, memory: 2Gi }
    limits:   { cpu: "2",  memory: 4Gi }
  template:
    pod:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: storage
                    operator: In
                    values: ["true"]
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    strimzi.io/cluster: prod-kafka
                topologyKey: kubernetes.io/hostname
---
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: prod-kafka
  namespace: kafka
spec:
  kafka:
    version: 4.1.1
    metadataVersion: 4.1-IV1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
      - name: external
        port: 9094
        type: nodeport
        tls: true
        configuration:
          preferredNodePortAddressType: InternalIP
    config:
      default.replication.factor: 3
      min.insync.replicas: 2
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF

kubectl -n kafka wait kafka/prod-kafka --for=condition=Ready --timeout=600s
kubectl -n kafka get pod -l strimzi.io/cluster=prod-kafka
```

**Проверка:** 3 пода Kafka должны быть Running (распределены по server-5 и server-6 с возможным удвоением на одной ноде — preferredAntiAffinity soft-правило).

### 6.2. MinIO Operator v7.1.1 (на 4 GPU-нодах, longhorn-single)

```bash
helm repo add minio-operator https://operator.min.io/
helm repo update
helm install minio-operator minio-operator/operator \
  --namespace minio-operator --create-namespace \
  --version 7.1.1

kubectl create namespace minio-tenant

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: minio-config-env
  namespace: minio-tenant
type: Opaque
stringData:
  config.env: |
    export MINIO_ROOT_USER="minio"
    export MINIO_ROOT_PASSWORD="ChangeMe-StrongPass-123!"
    export MINIO_STORAGE_CLASS_STANDARD="EC:2"
    export MINIO_BROWSER="on"
---
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: gpu-storage
  namespace: minio-tenant
  labels:
    app: minio
spec:
  image: quay.io/minio/minio:RELEASE.2025-04-08T15-41-24Z
  imagePullPolicy: IfNotPresent
  configuration:
    name: minio-config-env
  mountPath: /export
  requestAutoCert: false
  podManagementPolicy: Parallel
  pools:
    - name: pool-s1
      servers: 1
      volumesPerServer: 4
      nodeSelector: { gpu: "true", kubernetes.io/hostname: server-1 }
      volumeClaimTemplate:
        metadata: { name: data }
        spec:
          accessModes: [ReadWriteOnce]
          resources: { requests: { storage: 2Ti } }
          storageClassName: longhorn-single
      resources:
        requests: { cpu: "2", memory: "8Gi" }
        limits:   { cpu: "4", memory: "16Gi" }
    - name: pool-s2
      servers: 1
      volumesPerServer: 4
      nodeSelector: { gpu: "true", kubernetes.io/hostname: server-2 }
      volumeClaimTemplate:
        metadata: { name: data }
        spec:
          accessModes: [ReadWriteOnce]
          resources: { requests: { storage: 2Ti } }
          storageClassName: longhorn-single
      resources:
        requests: { cpu: "2", memory: "8Gi" }
        limits:   { cpu: "4", memory: "16Gi" }
    - name: pool-s3
      servers: 1
      volumesPerServer: 4
      nodeSelector: { gpu: "true", kubernetes.io/hostname: server-3 }
      volumeClaimTemplate:
        metadata: { name: data }
        spec:
          accessModes: [ReadWriteOnce]
          resources: { requests: { storage: 2Ti } }
          storageClassName: longhorn-single
      resources:
        requests: { cpu: "2", memory: "8Gi" }
        limits:   { cpu: "4", memory: "16Gi" }
    - name: pool-s4
      servers: 1
      volumesPerServer: 4
      nodeSelector: { gpu: "true", kubernetes.io/hostname: server-4 }
      volumeClaimTemplate:
        metadata: { name: data }
        spec:
          accessModes: [ReadWriteOnce]
          resources: { requests: { storage: 2Ti } }
          storageClassName: longhorn-single
      resources:
        requests: { cpu: "2", memory: "8Gi" }
        limits:   { cpu: "4", memory: "16Gi" }
  exposeServices:
    minio: true
    console: true
EOF

kubectl -n minio-tenant wait tenant/gpu-storage --for=condition=Ready --timeout=600s
```

### 6.3. MongoDB через новый унифицированный mongodb-kubernetes оператор

> **Примечание:** `mongodb-kubernetes-operator` (Community) архивирован 12.12.2025. Используем новый `mongodb-kubernetes` (v1.x).

```bash
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update

helm install mongodb-operator mongodb/mongodb-kubernetes \
  --namespace mongodb-operator --create-namespace \
  --set operator.watchNamespace="*"

kubectl create namespace mongodb

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-admin-password
  namespace: mongodb
type: Opaque
stringData:
  password: "ChangeMe-MongoAdmin-Pass-456!"
---
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: mongodb-rs
  namespace: mongodb
spec:
  members: 3
  type: ReplicaSet
  version: "7.0.14"
  security:
    authentication:
      modes: ["SCRAM"]
  users:
    - name: admin
      db: admin
      passwordSecretRef:
        name: mongodb-admin-password
      roles:
        - { name: clusterAdmin,       db: admin }
        - { name: userAdminAnyDatabase, db: admin }
        - { name: readWriteAnyDatabase, db: admin }
      scramCredentialsSecretName: mongodb-admin-scram
  statefulSet:
    spec:
      volumeClaimTemplates:
        - metadata: { name: data-volume }
          spec:
            accessModes: [ReadWriteOnce]
            storageClassName: longhorn
            resources: { requests: { storage: 50Gi } }
        - metadata: { name: logs-volume }
          spec:
            accessModes: [ReadWriteOnce]
            storageClassName: longhorn
            resources: { requests: { storage: 5Gi } }
      template:
        spec:
          nodeSelector:
            storage: "true"
          affinity:
            podAntiAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
                - weight: 100
                  podAffinityTerm:
                    labelSelector:
                      matchLabels:
                        app: mongodb-rs-svc
                    topologyKey: kubernetes.io/hostname
          containers:
            - name: mongod
              resources:
                requests: { cpu: 500m, memory: 1Gi }
                limits:   { cpu: "2",  memory: 4Gi }
            - name: mongodb-agent
              resources:
                requests: { cpu: 100m, memory: 256Mi }
                limits:   { cpu: 500m, memory: 512Mi }
EOF
```

### 6.4. Redis 7.4 (cache, ClusterIP)

```bash
kubectl create namespace cache

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata: { name: redis-config, namespace: cache }
data:
  redis.conf: |
    maxmemory 1gb
    maxmemory-policy allkeys-lru
    appendonly no
    save ""
    tcp-keepalive 60
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: redis, namespace: cache, labels: { app: redis } }
spec:
  replicas: 1
  strategy: { type: Recreate }
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers:
        - name: redis
          image: redis:7.4-alpine
          command: ["redis-server", "/etc/redis/redis.conf"]
          ports: [{ containerPort: 6379 }]
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 1Gi }
          volumeMounts: [{ name: config, mountPath: /etc/redis }]
          readinessProbe:
            exec: { command: ["redis-cli", "ping"] }
            initialDelaySeconds: 5
      volumes: [{ name: config, configMap: { name: redis-config } }]
---
apiVersion: v1
kind: Service
metadata: { name: redis, namespace: cache }
spec:
  type: ClusterIP
  selector: { app: redis }
  ports: [{ port: 6379, targetPort: 6379 }]
EOF
```

-----

## Фаза 7. Прикладные приложения (шаблон)

Пример прикладного пода, **без hostNetwork**, ClusterIP для внутреннего доступа, NodePort для внешнего, запрос GPU time-slice:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: ml-app, namespace: apps }
spec:
  replicas: 4
  selector: { matchLabels: { app: ml-app } }
  template:
    metadata: { labels: { app: ml-app } }
    spec:
      nodeSelector: { gpu: "true" }
      containers:
        - name: app
          image: registry.company.ru/ml-app:1.0
          resources:
            limits:
              nvidia.com/gpu: 1     # один time-slice
              cpu: "4"
              memory: 8Gi
            requests:
              cpu: "2"
              memory: 4Gi
          ports: [{ containerPort: 8080 }]
          env:
            - { name: KAFKA_BOOTSTRAP, value: "prod-kafka-kafka-bootstrap.kafka.svc:9092" }
            - { name: MONGO_URI,       value: "mongodb://admin:...@mongodb-rs-svc.mongodb.svc:27017" }
            - { name: REDIS_HOST,      value: "redis.cache.svc" }
            - { name: S3_ENDPOINT,     value: "http://minio.minio-tenant.svc:80" }
---
apiVersion: v1
kind: Service
metadata: { name: ml-app-internal, namespace: apps }
spec:
  type: ClusterIP
  selector: { app: ml-app }
  ports: [{ port: 8080, targetPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: ml-app-external, namespace: apps }
spec:
  type: NodePort
  selector: { app: ml-app }
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

Доступ извне: `http://<любой-нода>:30080`. Внутренний RPC: `ml-app-internal.apps.svc.cluster.local:8080`.

-----

## Фаза 8. CI/CD (заготовка — GitLab Runner / ArgoCD)

### ArgoCD (рекомендуется для GitOps)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.13.0/manifests/install.yaml

# Экспорт наружу через NodePort
kubectl -n argocd patch svc argocd-server -p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":8080,"nodePort":30443}]}}'

# Пароль админа
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

### GitLab Runner (для CI-джоб, которые собирают образы и пушат в приватный registry)

```bash
helm repo add gitlab https://charts.gitlab.io
helm install gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab-runner --create-namespace \
  --set gitlabUrl=https://gitlab.company.ru/ \
  --set runnerRegistrationToken=<TOKEN> \
  --set rbac.create=true \
  --set runners.privileged=true   # для Kaniko/DinD сборок
```

-----

## Фаза 9. Runbook смены IP-адресов (bare-metal, DNS-based)

**Сценарий:** меняется подсеть с 10.10.10.0/24 на 10.20.30.0/24. Поскольку `controlPlaneEndpoint` = `k8s-api:6443` (DNS, не IP), процедура сводится к обновлению `/etc/hosts` и перегенерации сертификатов.

### Шаг 1: остановить workload-трафик (опционально)

```bash
# Если хотите минимизировать ошибки — переведите приложения в maintenance
kubectl -n apps scale deployment --all --replicas=0
```

### Шаг 2: [ВСЕ СЕРВЕРЫ] обновить /etc/hosts и ОС-сеть

```bash
# Новые IP-адреса на каждом сервере через nmcli
sudo nmcli con mod eno1 ipv4.addresses 10.20.30.21/24 ipv4.gateway 10.20.30.1
sudo nmcli con up eno1

# Обновить /etc/hosts на КАЖДОМ из 6 серверов
sudo sed -i \
  -e 's/^10.10.10.20 /10.20.30.20 /' \
  -e 's/^10.10.10.21 /10.20.30.21 /' \
  -e 's/^10.10.10.22 /10.20.30.22 /' \
  -e 's/^10.10.10.23 /10.20.30.23 /' \
  -e 's/^10.10.10.24 /10.20.30.24 /' \
  -e 's/^10.10.10.25 /10.20.30.25 /' \
  -e 's/^10.10.10.26 /10.20.30.26 /' \
  /etc/hosts
```

### Шаг 3: [СЕРВЕР1] обновить VIP в kube-vip static pod

```bash
sudo sed -i 's|value: 10.10.10.20|value: 10.20.30.20|' /etc/kubernetes/manifests/kube-vip.yaml
# На server-2, server-3 — то же самое

# kubelet перезапустит static pod автоматически
ping -c3 10.20.30.20
```

### Шаг 4: [CP СЕРВЕРЫ] перегенерация сертификатов API-сервера

```bash
# На КАЖДОМ CP-сервере
sudo tee /tmp/new-san.yaml <<'EOF'
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.32.13
controlPlaneEndpoint: "k8s-api:6443"
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
apiServer:
  certSANs:
    - k8s-api
    - 10.20.30.20
    - 10.20.30.21
    - 10.20.30.22
    - 10.20.30.23
    - 127.0.0.1
EOF

sudo rm -f /etc/kubernetes/pki/apiserver.{crt,key}
sudo kubeadm init phase certs apiserver --config /tmp/new-san.yaml

# Рестарт apiserver (static pod)
sudo crictl pods --name kube-apiserver -q | xargs -r sudo crictl stopp
```

### Шаг 5: [CP СЕРВЕРЫ] обновление etcd peer URLs

```bash
# На каждой CP-ноде:
ETCDCTL_API=3 sudo etcdctl \
  --endpoints https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Для каждого члена с устаревшим IP:
ETCDCTL_API=3 sudo etcdctl \
  --endpoints https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member update <MEMBER_ID> --peer-urls=https://10.20.30.21:2380

# Правка /etc/kubernetes/manifests/etcd.yaml:
sudo sed -i \
  -e 's|10.10.10.21|10.20.30.21|g' \
  -e 's|10.10.10.22|10.20.30.22|g' \
  -e 's|10.10.10.23|10.20.30.23|g' \
  /etc/kubernetes/manifests/etcd.yaml
```

### Шаг 6: [ВСЕ СЕРВЕРЫ] обновить kubelet node-ip

```bash
sudo sed -i 's|10.10.10.|10.20.30.|g' /var/lib/kubelet/kubeadm-flags.env
sudo systemctl restart kubelet
```

### Шаг 7: верификация

```bash
kubectl get nodes -o wide              # все Ready, новые INTERNAL-IP
kubectl -n kube-system get pod         # все Running
kubectl -n calico-system get pod       # все Running
ping -c3 k8s-api                       # отвечает 10.20.30.20
curl -k https://k8s-api:6443/healthz   # ok

# Calico переподхватит eno1 автоматически (interface selector)
# Longhorn volumes продолжат работу — данные на дисках не меняются
```

**Почему процедура проще VM-версии:** нет KVM-слоя, нет необходимости обновлять virsh/libvirt конфиги мостов, нет гостевой сети — меняется только физический IP eno1 и /etc/hosts.

-----

## Таблица решённых проблем

|# |Проблема                                                        |Решение                                                                            |Фаза|
|--|----------------------------------------------------------------|-----------------------------------------------------------------------------------|----|
|1 |GPU time-slicing 4 слайса/T4                                    |ConfigMap time-slicing + label `nvidia.com/device-plugin.config=tesla-t4`          |4   |
|2 |Ядро 6.12 несовместимо с прекомпилированными kmod RHEL 8        |`nvidia-driver:570-open-dkms` module stream                                        |2   |
|3 |kube-vip не может получить leader lease при init (K8s ≥1.29)    |Временно `super-admin.conf` → после init откатить на `admin.conf`                  |3   |
|4 |RedOS 8: zram-swap включён по умолчанию                         |`systemctl mask dev-zram0.swap`                                                    |1   |
|5 |RedOS 8: systemd-resolved конфликтует с CoreDNS                 |`DNSStubListener=no`                                                               |1   |
|6 |RedOS 8: CNI binaries в `/usr/libexec/cni` (не `/opt/cni/bin`)  |Правка `bin_dir` в containerd.config.toml + симлинки                               |1   |
|7 |Longhorn + iscsid SELinux denials на Fedora-derivatives         |`longhorn-iscsi-selinux-workaround.yaml` + SELinux permissive                      |5   |
|8 |3 реплики MongoDB/Kafka на 2 storage-нодах                      |`preferredDuringSchedulingIgnoredDuringExecution` (soft)                           |6   |
|9 |Конфликты портов на хосте                                       |ClusterIP для внутреннего, NodePort 30000-32767 для внешнего; **нет** hostNetwork  |7   |
|10|MinIO на GPU-нодах с независимой репликацией                    |`longhorn-single` (replica=1) + MinIO EC:2                                         |6   |
|11|Смена IP всего кластера                                         |DNS-based `k8s-api` → обновить только /etc/hosts + перегенерировать certs apiserver|9   |
|12|NVIDIA Container Toolkit 1.18.0 ломает containerd config        |Pin на 1.19.0 (или 1.18.1+)                                                        |2   |
|13|Longhorn 1.11.0 memory leak в instance-manager                  |Использовать 1.10.2 (Prime stable) или 1.11.1                                      |5   |
|14|MongoDB Community Operator архивирован 12.12.2025               |Переход на унифицированный `mongodb-kubernetes`                                    |6   |
|15|Strimzi 0.45 → 0.46+: Kafka 3.9.2 несовместим с новым оператором|Использовать Strimzi 0.51.0 с Kafka 4.1.1 сразу                                    |6   |
|16|NetworkManager управляет интерфейсами Calico (flapping)         |`/etc/NetworkManager/conf.d/calico.conf` unmanaged-devices                         |3   |
|17|Прокси во всех слоях (env, dnf, containerd, kubelet, helm)      |Отдельные drop-in файлы + `/etc/profile.d/proxy.sh`                                |0-3 |
|18|Калько eBPF на ядре 6.12 — BPF verifier rejects                 |Используем iptables/VXLAN dataplane (eBPF НЕ включаем)                             |3   |
|19|Firewalld reload сбрасывает iptables kube-proxy                 |Используем `--runtime-to-permanent` или trusted zone                               |1   |
|20|CP-ноды также работают воркерами                                |`kubectl taint ... control-plane:NoSchedule-`                                      |3   |

## Заключение и ключевые риски

**Архитектурно** этот кластер представляет собой классический «converged» bare-metal stack: 3 CP-ноды с VIP, совмещение CP+Worker, GPU time-slicing для уплотнения воркложов, и симметричное хранилище Longhorn на всех 6 узлах. Ключевые решения, которые отличают этот гайд от упрощённых руководств — **DNS-based controlPlaneEndpoint** (превращает смену IP из 2-часовой процедуры в 15-минутную), **kube-vip super-admin.conf workaround** (без него init зависает в production-grade K8s ≥1.29), и **использование DKMS вместо precompiled kmod** (обязательно на ядре 6.12, не документировано в NVIDIA RHEL 8 guide).

**Главные риски, требующие принятия решения до продакшена:**

1. **Kubernetes 1.32 достиг EOL 28 февраля 2026.** Дальнейших CVE-патчей не будет. Если развёртывание предназначено для долгосрочной эксплуатации, **следует целиться в 1.33.10+ или 1.34.x** — все компоненты в этом гайде совместимы с обеими ветками (нужно заменить `1.32.13` и префикс репозитория `v1.32` на `v1.33`/`v1.34`).
1. **RHEL 8 не входит в официальный verified-list Longhorn 1.10/1.11** (SUSE QA проверяет только RHEL 9.6/10.0). Работать будет, но без формальной валидации. Для строгих SLA — RHEL 9 даёт больше уверенности.
1. **MongoDB Community Operator архивирован** — документация устаревает, сообщество уходит. Миграция на унифицированный `mongodb-kubernetes` оператор обязательна.
1. **Prefer-anti-affinity даёт 3 пода БД на 2 нодах** — потеря одной storage-ноды временно снижает доступность до 1/3 реплик (кворум теряется). Для строгого HA нужна третья storage-нода или hard anti-affinity с 2 репликами.

**Новое, что даёт этот гайд по сравнению с VM-версией:** отсутствие overhead гипервизора возвращает ~10-15% CPU и весь PCI passthrough для GPU; исчезает необходимость в nested-virtualization трюках для containerd; процедура смены IP упрощается в ~5 раз (нет libvirt/virsh конфигов); и главное — **все 16 виртуальных GPU** физически изолированы на 4 нодах с native NVIDIA context, без вложенных драйверов.
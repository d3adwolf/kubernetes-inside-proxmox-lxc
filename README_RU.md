# K8s кластер в LXC Proxmox
**Language:** 🇷🇺 · [🇺🇸](https://github.com/d3adwolf/kubernetes-in-lxc-on-proxmox/blob/main/README.md)<br><br>
Инструкция о том, как развернуть рабочий K8s кластер в LXC контейнере Proxmox. 
### Предисловие
> [!NOTE]\
> Данная инструкция сделана на основе нескольких статей, официальных документов и собственной практики.<br>
> Все ссылки на первоначальные источники указаны в конце **README**.

### Тестировалось на:
`Proxmox:`
> **Kernel Version:** Linux 6.5.11-7-pve (2023-12-05T09:44Z)<br>
> **Manager Version:** pve-manager/8.1.3/b46aac3b42da5d15

`Kubernetes:`
> **kubectl** v1.29.0<br>
> **crictl** v1.29.0<br>
> **cri-dockerd** v0.3.9
## Подготовка Proxmox
### Модули ядра
Подключим в ядро рекомендованные модули для работы контейнеров Docker и в целом для K8s.
Для этого отредактируем `/etc/modules` и добавим:
```bash
overlay
br_netfilter
ip_vs
nf_nat
xt_conntrack
```
Немного распишу зачем нужен каждый модуль:
- `overlay` - модуль для управления переносом сетевого трафика между разными сетевыми интерфейсами или подсетями.
- `br_netfilter` - модуль, который обеспечивает фильтрацию трафика на уровне сетевого моста (bridge).
- `ip_vs` - модуль для управления протоколом IP виртуального сервера (IPVS), который позволяет создавать высокопроизводительные и отказоустойчивые сетевые сервисы.
- `nf_nat` - модуль, который обеспечивает NAT (Network Address Translation) для перенаправления трафика между разными сетевыми интерфейсами или подсетями.
- `xt_conntrack` - модуль для отслеживания состояния TCP/UDP соединений и управления ими на уровне сетевого протокола.

Чтобы не перезагружать ноду, активируем модули через команду:
```bash
modprobe <module>
```
Проверим активные модули:
```bash
lsmod | grep -E 'overlay|br_netfilter|ip_vs|nf_nat|xt_conntrack'
```
Также рекомендуется выполнить команду по обновлению существующего образа initramfs, который используется при загрузке системы:
```bash
update-initramfs -u
```
> [!NOTE]\
> Образ **initramfs** содержит модули ядра, драйверы устройств, скрипты и утилиты, необходимые для корректной работы системы во время загрузки.
### Сетевой трафик
Также убедимся, что `iptables` будет правильно воспринимать сетевой трафик из всех узлов Proxmox, для этого создадим конфиг с разрешением переадресации трафика сети:
```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
Применим параметры командой:
```bash
sysctl --system
```
Проверим изменения в ноде:
```bash
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables
```
### Файл подкачки
Отключаем в настоящий момент swap-раздел:
```bash
swapoff -a
```
> [!NOTE]\
> Для **Kubernetes** принято выключать swap-раздел, чтобы кластер мог правильно оценивать свободную оперативную память и не имел проблемы с производительностью.

И комментируем строки, чтобы при следующей загрузке, он уже не включился:
```bash
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Создание LXC контейнера
В UI Proxmox начнем создание контейнера через "Create CT".
### General
Сразу же включаем галочку на "Advanced" и убираем галочку с "Unprivileged container".
Должно получиться вот так:
- [ ] Unprivileged container
- [X] Nesting
- [X] Advanced
### Template
Здесь выбираем образ на вкус и цвет, мой выбор пал на актуальную версию Ubuntu.
### Memory
Выключаем swap в контейнере:
```
Swap (MiB): 0
```
### Network
В будущем кластере есть **Network Policy** и другие инструменты ограничения трафика, поэтому Firewall, на мой взгляд, можно выключить:
- [ ] Firewall

Обязательно пропишем нашей ноде статический IP-адрес, чтобы он не сменился спустя время:
```
IPv4: Static
IPv4/CIDR: 192.168.0.10/24
Gateway (IPv4): 192.168.0.1
```
Естественно, это просто пример, и вы указываете вашу локальную сеть.
### DNS
Если вы настраивали или имеете дома отдельный DNS-сервер, что хорошо, то лучше указывать его, если ваш маршрутизатор является основным шлюзом и DNS-сервером, то пропускаем эту вкладку.
### Confirm
Не торопимся с запуском контейнера:
- [ ] Start after created

## Настройка LXC контейнера
Теперь нужно подготовить контейнер для правильной работы **K8s** кластера, можете сразу установить удобный редактор текста в Proxmox ноде и в LXC контейнере, я использую `vim`, так как умею из него выходить:
```
apt install -y vim
```
### Действия вне контейнера
В начале выключим контейнер и зайдем на Proxmox под `root` через SSH в каталог `/etc/pve/lxc`, а далее отредактируем через текстовый редактор `<container id>.conf`, где **container id** - индификатор нашего LXC контейнера.

Добавим в файл строки:
```bash
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: "proc:rw sys:rw"
```
Немного расскажу про эти параметры:
- `lxc.apparmor.profile: unconfined` - устанавливает профиль Apparmor для контейнера на "unconfined", что отключает [AppArmor](https://www.apparmor.net/) в LXC.
- `lxc.cgroup.devices.allow:` a - разрешает контейнеру корневой доступ к [cgroup](https://wiki.archlinux.org/title/cgroups).
- `lxc.cap.drop:` - отключает автоматическое отключение некоторых способностей (capabilities) для контейнера, что может быть полезно для некоторых приложений, подробнее в [документации LXC](https://linuxcontainers.org/lxc/manpages/man5/lxc.container.conf.5.html).
- `lxc.mount.auto: "proc:rw sys:rw"` - монтирует корневой раздел /proc и /sys на R/W доступ для контейнера, что обычно необходимо для корректной работы системы.

Теперь нужно перекинуть конфигурация загрузки ядра в контейнер, так как `kubelet` использует конфигурацию для определения настроек среды кластера.

Запускаем контейнер и через `root` в Proxmox выполняем команду:
```bash
pct push <container id> /boot/config-$(uname -r) /boot/config-$(uname -r)
```
Теперь создадим символьную ссылку для `/dev/kmsg`, так как `kubelet` использует это для функции журналирования, в LXC у нас есть для этого `/dev/console`, поэтому будем ссылаться на него, создав bash-скрипт в `/usr/local/bin/conf-kmsg.sh`:
```bash
#!/bin/sh -e
if [ ! -e /dev/kmsg ]; then
    ln -s /dev/console /dev/kmsg
fi

mount --make-rshared /
```
И настроим разовый запуск скрипта при запуске LXC контейнера.

Создадим `/etc/systemd/system/conf-kmsg.service` с таким содержанием:
```bash
[Unit]
Description=Make sure /dev/kmsg exists

[Service]
Type=simple
RemainAfterExit=yes
ExecStart=/usr/local/bin/conf-kmsg.sh
TimeoutStartSec=0

[Install]
WantedBy=default.target
```
Делаем наш скрипт исполняемым и включаем службу:
```bash
chmod +x /usr/local/bin/conf-kmsg.sh
systemctl daemon-reload
systemctl enable --now conf-kmsg
```
### Настройка базового окружения
Обновим установленные пакеты и доставим те, которые пригодяться в дальнейшей настройке:
```bash
apt update && apt upgrade -y
apt install -y wget curl conntrack
```
Удалим дефолтный firewall, потому что в **K8s** используются другие инструменты управления трафика:
```bash
apt remove -y ufw && apt autoremove -y
```
### kubectl
Поставим инструмент для взаимодействия с API-сервером Kubernetes, управления ресурсами и рабочими процессами Kubernetes:
```bash
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
```
Проверим установленную версию:
```bash
kubectl version --client
```
### helm
Поставим инструмент для управления пакетами в Kubernetes, который автоматизирует развертывание приложений:
```bash
apt install -y git
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
rm get_helm.sh
```
Проверим установленную версию:
```bash
helm version
```
> Актуальную версию можно глянуть в [Releases](https://github.com/helm/helm/releases) репозитория, но может установиться более стабильная для вашей ОС.
### crictl
> [!WARNING]\
> Требуется для `minikube`, в остальных случаях ставим по востребованию.

Поставим инструмент для управления контейнерами и их ресурсами в Kubernetes:
```bash
VERSION="v1.29.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```
Проверим установленную версию:
```bash
crictl -v
```
> Актуальную версию можно глянуть в [Releases](https://github.com/kubernetes-sigs/cri-tools/releases) репозитория.
### containernetworking-plugins
> [!WARNING]\
> Требуется для `minikube`, в остальных случаях ставим по востребованию.

Поставим набор плагинов для сетевого взаимодействия контейнеров в Kubernetes:
```bash
CNI_PLUGIN_VERSION="v1.4.0"
CNI_PLUGIN_TAR="cni-plugins-linux-amd64-$CNI_PLUGIN_VERSION.tgz"
CNI_PLUGIN_INSTALL_DIR="/opt/cni/bin"

curl -LO "https://github.com/containernetworking/plugins/releases/download/$CNI_PLUGIN_VERSION/$CNI_PLUGIN_TAR"
mkdir -p "$CNI_PLUGIN_INSTALL_DIR"
tar -xf "$CNI_PLUGIN_TAR" -C "$CNI_PLUGIN_INSTALL_DIR"
rm "$CNI_PLUGIN_TAR"
```
> Актуальную версию можно глянуть в [Releases](https://github.com/containernetworking/plugins/releases) репозитория.
### cri-dockerd
> [!WARNING]\
> Требуется для `minikube` в связке с **Docker**, в остальных случаях ставим при использовании Docker, как Container Runtime в K8s.

Поставим адаптер, который предоставляет совместимость между **Docker Engine** и **CRI** в Kubernetes:
```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.9/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
dpkg -i cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
rm -f cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```
Проверим установленную версию:
```bash
cri-dockerd --version
```
> Актуальную версию можно глянуть в [Releases](https://github.com/Mirantis/cri-dockerd/releases) репозитория.
## Установка Container Runtime
### Docker
> [!WARNING]\
> Требуется для `minikube`, в остальных случаях ставим по востребованию.

Установим зависимости, добавим apt-репозиторий в систему:
```bash
apt update
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
```
Ставим Docker через `apt`:
```bash
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Проверим установленную версию:
```bash
docker version
```
### containerd
Легко и просто установливаем через `apt`:
```bash
apt install -y containerd
```
Проверим установленную версию:
```bash
containerd --version
```
Создадим папку для файла конфигурации:
```bash
mkdir /etc/containerd/
```
Установим настройки по умолчанию для конфигурации контейнера:
```bash
containerd config default > /etc/containerd/config.toml
```
Для разрешения использования **cgroup** переключите флаг параметра `systemdCgroup` в `/etc/containerd/config.toml`:
```bash
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
Также проверим наши изменения:
```bash
cat /etc/containerd/config.toml | grep SystemdCgroup
```
### CRI-O
Создадим переменные с актуальной версией crio:
```bash
export OS=xUbuntu_22.04
export VERSION=1.24
```
> Актуальную версию можно глянуть на [download.opensuse.org](https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/)

Установим зависимости, добавим apt-репозиторий в систему:
```bash
apt install -y gnupg

echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

apt update
```
Ставим crio через `apt`:
```bash
apt install -y cri-o cri-o-runc
```
Проверим установленную версию:
```bash
crio -v
```
Теперь нужно отключить [AppArmor](https://www.apparmor.net/) для crio:
```bash
sed -i 's/# apparmor_profile =\ "crio-default"/apparmor_profile \= "unconfined"/g' /etc/crio/crio.conf
```
Копируем конфиг для работы в `minikube`:
```bash
cp /etc/crio/crio.conf /etc/crio/crio.conf.d/02-crio.conf
```
Проверим изменения:
```bash
cat /etc/crio/crio.conf /etc/crio/crio.conf.d/02-crio.conf | grep apparmor_profile
```
Запускаем crio и добавляем его в автозапуск:
```bash
systemctl enable --now crio
```
## Установка Kubernetes
### minikube
**Container Runtime**

Выбор контейнера для Kubernetes (Container Runtime) зависит от ваших требований и предпочтений, но наиболее распространенными контейнерами для Kubernetes являются Docker, containerd и CRI-O.

1. **Docker** - самый распространенный контейнер, включенный в большинство дистрибутивов Kubernetes.

2. **containerd** - второй самый популярный контейнер, также часто используемый с Kubernetes. 

3. **CRI-O** - контейнер, специально разработанный для соответствия интерфейсу контейнера Kubernetes (CRI).

Чтобы определиться, я создал кластеры **K8s** в одинаковых условиях с разными Container Runtime, и вот что у меня получилось:

| Container Runtime  | Creating time (seconds) |
| ------------------ | ------------------------|
| Docker             | ~25                     |
| containerd         | ~22                     |
| CRI-O              | ~16                     |

**Установка**

Скачиваем пакет и устанавливаем:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
install minikube-linux-amd64 /usr/local/bin/minikube
rm -f minikube-linux-amd64
```
Установим рекомендованные зависимости:
```bash
apt install -y ethtool socat
```
**Docker**

Теперь спокойно запускаем кластер:
```bash
minikube start --vm-driver=none --extra-config=kubeadm.ignore-preflight-errors=SystemVerification --kubernetes-version=v1.29.0 --container-runtime=docker
```

**containerd**

Для запуска `minikube` через **containerd** требуется **docker-cli**.

Как по инструкции выше для **Docker**, делаем:
```bash
apt update
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
```
И просто ставим **docker-cli**:
```bash
apt install -y docker-ce-cli
```
Теперь спокойно запускаем кластер:
```bash
minikube start --vm-driver=none --extra-config=kubeadm.ignore-preflight-errors=SystemVerification --kubernetes-version=v1.29.0 --container-runtime=containerd
```
**crio**

Для запуска `minikube` через **crio** требуется **docker-cli**.

Как по инструкции выше для **Docker**, делаем:
```bash
apt update
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
```
И просто ставим **docker-cli**:
```bash
apt install -y docker-ce-cli
```
Теперь спокойно запускаем кластер:
```bash
minikube start --vm-driver=none --extra-config=kubeadm.ignore-preflight-errors=SystemVerification --kubernetes-version=v1.29.0 --container-runtime=crio
```
**Удаление**

Если захотите удалить кластер достаточно выполнить:
```bash
minikube delete
```
Если нужно удалить все профили, то:
```bash
minikube delete --all
```
Если нужно удалить `minikube`, то из вариантов только:
```bash
rm -f /usr/local/bin/minikube
```

### microk8s
**Установка**

Установим службу, которая управляет snap-пакетами:
```bash
apt install -y snapd
```
Уже через него ставим microk8s:
```bash
snap install microk8s --classic
```
Добавляем пользователя в группу microk8s:
```bash
usermod -a -G microk8s $USER
chown -f -R $USER ~/.kube
```
Желательно, заходим под обычным пользователем:
```bash
su - $USER
```
Смотрим статус кластера:
```bash
microk8s status --wait-ready
```
Создадим алиас, чтобы не вводить microk8s для работы через kubectl:
```bash
alias kubectl='microk8s kubectl'
```
**Удаление**

Чтобы удалить microk8s выполните:
```bash
snap remove microk8s
```
Не забываем про alias:
```bash
unalias kubectl
```
### K3s
**Установка**

Здесь довольно простая установка:
```bash
curl -sfL https://get.k3s.io | sh -
```
Создадим alias, чтобы не вводить k3s для работы через kubectl:
```bash
alias kubectl='k3s kubectl'
```
или перезаписываем конфиги kubectl:
```bash
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && chown $USER ~/.kube/config
chmod 600 ~/.kube/config && export KUBECONFIG=~/.kube/config
```
**Удаление**

Чтобы удалить K3s достаточно выполнить:
```bash
/usr/local/bin/k3s-uninstall.sh
```
Также не забываем про alias:
```bash
unalias kubectl
```
### k0s
**Установка**

Здесь довольно простая установка:
```bash
curl -sSLf https://get.k0s.sh | sh
```
Установим как службу:
```bash
k0s install controller --single
```
Запустим кластер:
```bash
k0s start
```
Проверим статус кластера:
```bash
k0s status
```
Создадим alias, чтобы не вводить k0s для работы через kubectl:
```bash
alias kubectl='k0s kubectl'
```
**Удаление**

Остановим службу:
```bash
k0s stop
```
Удалим службу k0s и все зависимости:
```bash
k0s reset
```
Если нужно удалить `k0s`, то из вариантов только:
```bash
rm -f /usr/local/bin/k0s
```
Также не забываем про alias:
```bash
unalias kubectl
```
## Проверка любого кластера на работоспобность
Выполняем:
```bash
kubectl get nodes && \
echo && \
kubectl get services && \
echo && \
kubectl get pods -A
```
На выходе должны получить текущее состояние кластера, и если он есть, и `STATUS` = **Ready**, то вас можно поздравить.
## Проверка работоспобности сети любого кластера
Создадим деплоймент:
```bash
kubectl create deployment hello-world --image=registry.k8s.io/echoserver:1.10
```
Создадим сервис для деплоймента:
```bash
kubectl expose deployment hello-world --type=NodePort --port=8080
```
Смотрим за запуском пода, а также смотрим его NodePort:
```bash
kubectl get pods -o wide
kubectl get service
```
> Ищем такое: 8080:XXXXX/TCP, где XXXXX - NodePort

Проверим доступность пода вне Proxmox, на своем ноутбуке, выполните `curl` запрос:
```bash
curl <ip adress>:XXXXX
```
Где `<ip adress>` - IP-адрес контейнера LXC, а `XXXXX` - внешний порт нашего пода.

Ответный запрос должен выйти таким:
> Hostname: hello-world-576c8bfdf8-c269c
>
>Pod Information:
> -no pod information available-
>
>Server values:
> server_version=nginx: 1.13.3 - lua: 10008

И так далее. 

После успешных проверок, удалим тестовый деплоймент и сервис.
```bash
kubectl delete services hello-world
kubectl delete deployment hello-world
```
На этом разбор установки **K8s** в LXC можно заканчивать.
## Использованные материалы
Очень рекомендую ознакомиться с ресурсами ниже для более широкого понимания всех процессов:
> Установка [Docker Engine](https://docs.docker.com/engine/)
>
> Установка [Kubernetes](https://kubernetes.io/docs/home/)
>
> Установка [minikube](https://minikube.sigs.k8s.io/docs/)
>
> Установка [microk8s](https://microk8s.io/docs/getting-started)
>
> Установка [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
>
> Установка [K3s](https://docs.k3s.io/quick-start)
>
> Установка [k0s](https://docs.k0sproject.io/head/install/)
>
> Установка [cri-o](https://github.com/cri-o/cri-o)
>
> Настройка [cgroup](https://rootlesscontaine.rs/getting-started/common/cgroup2/)
>
> Статья [блога Гарретта Миллса](https://garrettmills.dev/blog/2022/04/18/Rancher-K3s-Kubernetes-on-Proxmox-Container/)
>
> Инструкция по запуску [Kubernetes в redOS](https://redos.red-soft.ru/base/server-configuring/container/kubernetes/kuber/)
>
> Инструкция по запуску [microk8s в LXD](https://microk8s.io/docs/install-lxd)

## Планы
Я буду стараться дополнять статью, как для себя, так и для всех вас.

Вот мои текущие планы по ней:
- [X] Запустить [k0s](https://docs.k0sproject.io/head/)
- [ ] Запустить [kind](https://kind.sigs.k8s.io/) (в настоящий момент запустить не удается)
- [ ] Поднять кластер через [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [X] Настроить поддержку [cri-o](https://cri-o.io/) для `minikube`


Ваши идеи можете предлагать в [Discussions](https://github.com/d3adwolf/kubernetes-inside-lxc-on-proxmox/discussions).

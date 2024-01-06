# K8s кластер в LXC Proxmox
**Language:** [🇷🇺](https://github.com/d3adwolf/kubernetes-in-lxc-on-proxmox/blob/main/README_RU.md) · [🇺🇸](https://github.com/d3adwolf/kubernetes-in-lxc-on-proxmox/blob/main/README.md)<br><br>
Инструкция о том, как развернуть рабочий K8s кластер в LXC контейнере Proxmox. 
### Предисловие
> [!NOTE]\
> Данная инструкция сделана на основе нескольких статей, официальных документов и собственной практики.
> Все ссылки на первоначальные источники я укажу в конце README.

> [!Warning]\
> Инструкция в разработке с 05.01.2024 на днях будет готовый вариант, выпустил в публичный доступ, чтобы собирать уже обратную связь.

`Proxmox:`
> **Kernel Version:** Linux 6.5.11-7-pve (2023-12-05T09:44Z)<br>
> **Manager Version:** pve-manager/8.1.3/b46aac3b42da5d15

`Kubernetes:`
> **kubeadm** v1.29.0<br>
> **kubelet** v1.29.0<br>
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
Чтобы не перезагружать ноду, активируем модули через команду:
```bash
modprobe <module>
```
Проверим активные модули:
```bash
lsmod | grep -E 'overlay|br_netfilter|ip_vs|nf_nat|xt_conntrack'
```
### Сетевой трафик
Также убедимся, что `iptables` будет правильно воспринимать сетевой трафик из всех узлов Proxmox, для этого создадим конфиг с разрешением переадресации трафика сети:
```bash
cat <<EOF   tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
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
Для Kubernetes принято выключать swap-раздел, чтобы кластер правильно оценивал свободную оперативную память и не имел проблемы с производительностью.
```
Swap (MiB): 0
```
### Network
В будущем кластере есть **Network Policy**, поэтому Firewall, на мой взгляд, можно выключить:
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
Теперь нужно подготовить контейнер для правильной работы K8s кластера, можете сразу установить удобный редактор текста в Proxmox ноде и в LXC контейнере, я использую `vim` и умею из него выходить:
```
apt install -y vim
```
### Действия вне контейнера
В начале выключим контейнер и зайдем на Proxmox под `root` через SSH в каталог `/etc/pve/lxc`, а далее отредактируем через текстовый редактор `<container id>.conf`, где **container id** - индификатор нашего LXC контейнера. Добавим в файл строки:
```bash
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: "proc:rw sys:rw"
```
- `lxc.apparmor.profile: unconfined` - отключает [AppArmor](https://www.apparmor.net/)
- `lxc.cgroup.devices.allow:` a - разрешает контейнеру полный доступ к [cgroup](https://wiki.archlinux.org/title/cgroups)
- `lxc.cap.drop:` - предотвращает "возможности падения", лучше глянуть [документацию LXC](https://linuxcontainers.org/lxc/manpages/man5/lxc.container.conf.5.html)
- `lxc.mount.auto: "proc:rw sys:rw"` - монтирует корневой раздел /proc и /sys на R/W доступ
Теперь нужно перекинуть конфигурация загрузки ядра в контейнер, так как `kubelet` использует конфигурацию для определения настроек среды кластера.
Запускаем контейнер и через `root` в Proxmox выполняем команду:
```bash
pct push <container id> /boot/config-$(uname -r) /boot/config-$(uname -r)
```
Теперь создадим символьную ссылку для `/dev/kmsg`, так как `kubelet` использует это для функции журналирования, в LXC у нас есть для этого `/dev/console`, поэтому будем ссылаться на него, создав Bash скрипт в `/usr/local/bin/conf-kmsg.sh`:
```bash
#!/bin/sh -e
if [ ! -e /dev/kmsg ]; then
    ln -s /dev/console /dev/kmsg
fi

mount --make-rshared /
```
И настроим разовый запуск скрипта при запуске LXC контейнера.
Создаем `/etc/systemd/system/conf-kmsg.service` с таким содержанием:
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
Делаем наш Bash исполняемым и включаем службу:
```bash
chmod +x /usr/local/bin/conf-kmsg.sh
systemctl daemon-reload
systemctl enable --now conf-kmsg
```
### Настройка базового окружения
Обновим установленные пакеты и доставим те, которые пригодяться в дальнейшей настройке:
```bash
apt update && apt upgrade -y
apt install -y wget curl vim conntrack
```
Удалим дефолтный firewall, потому что в K8s используются другие инструменты управления трафиков:
```bash
apt remove -y ufw && apt autoremove -y
```
### kubectl
Поставим инструмент для управления кластером K8s:
```bash
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
```
Проверим установленную версию:
```bash
kubectl version --client
```
### crictl
> [!WARNING]\
> Требуется для `minikube`.

Поставим инструмент для управления Container Runtime Interface:
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
### cri-dockerd
> [!WARNING]\
> Требуется для `minikube` в связке с Docker.

Поставим CRI для Docker'а, чтобы Kubernetes смог им управлять:
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
### containernetworking-plugins
> [!WARNING]\
> Требуется для `minikube`.

Поставим плагин для работы сети в Kubernetes:
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
## Установка Docker
> [!WARNING]\
> Требуется для `minikube`.

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
Проверяем результат и версию Docker:
```bash
docker version
```
## Установка Kubernetes
### kind
> [!WARNING]\
> В настоящий момент не удалось запустить в LXC

### minikube
Скачиваем пакет и устанавливаем:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
install minikube-linux-amd64 /usr/local/bin/minikube
rm -f minikube-linux-amd64
```
Установим зависимости:
```bash
apt install -y ethtool socat
```
Для запуска через CRI Docker воспользуемся этой командой:
```bash
minikube start --vm-driver=none --extra-config=kubeadm.ignore-preflight-errors=SystemVerification --kubernetes-version=v1.29.0 --container-runtime=docker
```
или более правильный выбор в сторону containerd и crio:
```bash
minikube start --vm-driver=none --extra-config=kubeadm.ignore-preflight-errors=SystemVerification --kubernetes-version=v1.29.0 --container-runtime=containerd
```
Если в дальнейшем вы будете работать через обычного пользователя, следует перенести конфиги:
```bash
mv /root/.kube /root/.minikube $HOME
chown -R $USER $HOME/.kube $HOME/.minikube
```
Проверям работоспособность:
```bash
kubectl get nodes -A
kubectl get services
```

### kubeadm
> [!WARNING]\
> В процессе тестирования

### microk8s
Ставим Snap менеджер:
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
Проверям работоспособность:
```bash
microk8s kubectl get nodes -A
microk8s kubectl get services
```
Создадим алиас, чтобы не вводить microk8s для работы через kubectl:
```bash
alias kubectl='microk8s kubectl'
```
### k3s
Здесь довольно простая установка:
```bash
curl -sfL https://get.k3s.io | sh - 
# Check for Ready node, takes ~30 seconds 
k3s kubectl get node 
```
Создадим алиас, чтобы не вводить k3s для работы через kubectl:
```bash
alias kubectl='k3s kubectl'
```
Или перезаписываем конфиги kubectl:
```bash
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER ~/.kube/config
sudo chmod 600 ~/.kube/config && export KUBECONFIG=~/.kube/config
```
Проверям работоспособность:
```bash
kubectl get nodes -A
kubectl get services
```
Чтобы удалить K3s достаточно выполнить:
```bash
/usr/local/bin/k3s-uninstall.sh
```
## Проверка любого кластера на работоспобность
Создаем деплоймент:
```bash
kubectl create deployment hello-world --image=registry.k8s.io/echoserver:1.10
```
Создаем сервис для деплоймента:
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


## Всевозможные ошибки

## Документации по технологиям
> Из документации [Kubernetes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)<br>
> Из статьи [блога Гарретта Миллса](https://garrettmills.dev/blog/2022/04/18/Rancher-K3s-Kubernetes-on-Proxmox-Container/)<br>
> https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md
> https://microk8s.io/docs/getting-started
> https://docs.k3s.io/quick-start
https://microk8s.io/docs/install-lxd
https://habr.com/ru/articles/420913/
https://docs.docker.com/engine/
https://kubernetes.io/docs/home/
https://minikube.sigs.k8s.io/docs/
https://redos.red-soft.ru/base/server-configuring/container/kubernetes/kuber/
https://linuxcontainers.org/lxc/manpages/man5/lxc.container.conf.5.html
https://kind.sigs.k8s.io/docs/user/quick-start/
https://github.com/cri-o/cri-o
https://rootlesscontaine.rs/getting-started/common/cgroup2/
https://stackoverflow.com/questions/65397050/minikube-does-not-start-on-ubuntu-20-04-lts-exiting-due-to-guest-provision

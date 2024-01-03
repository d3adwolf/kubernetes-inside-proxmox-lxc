# K8s кластер в LXC Proxmox
Инструкци о том, как развернуть рабочий K8s кластер в LXC контейнере Proxmox. 
<br>
### Предисловие
> [!NOTE]\
> Данная инструкция сделана на основе нескольких статей, официальных документов и собственной практики.
> Все ссылки на первоначальные источники я укажу в конце README.

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
**Модули ядра**<br>
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
**Сетевой трафик**<br>
Также убедимся, что `iptables` будет правильно воспринимать сетевой трафик из всех узлов Proxmox, для этого создадим конфиг с разрешением переадресации трафика сети:
```bash
cat <<EOF   tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
```
Проверим изменения:
```bash
sysctl --system
```
**Файл подкачки**<br>
```bash
swapoff -a
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
```bash
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: "proc:rw sys:rw"
```

```bash
pct push <container id> /boot/config-$(uname -r) /boot/config-$(uname -r)
```

```bash
#!/bin/sh -e
if [ ! -e /dev/kmsg ]; then
    ln -s /dev/console /dev/kmsg
fi

mount --make-rshared /
```

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

```bash
chmod +x /usr/local/bin/conf-kmsg.sh
systemctl daemon-reload
systemctl enable --now conf-kmsg
```
## Установка базового окружения

## Установка Kubernetes
### Minikube
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
```bash
minikube start --vm-driver none --extra-config kubeadm.ignore-preflight-errors=SystemVerification
```
### MicroK8s
```bash
sudo snap install microk8s --classic
```
```bash
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
```
```bash
su - $USER
```
```bash
microk8s status --wait-ready
```
```bash
microk8s kubectl get nodes
microk8s kubectl get services
```
```bash
alias kubectl='microk8s kubectl'
```
### K3s
```
curl -sfL https://get.k3s.io | sh - 
# Check for Ready node, takes ~30 seconds 
sudo k3s kubectl get node 
```

## Ссылки
> Из документации [Kubernetes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)<br>
> Из статьи [блога Гарретта Миллса](https://garrettmills.dev/blog/2022/04/18/Rancher-K3s-Kubernetes-on-Proxmox-Container/)<br>
> https://github.com/manusa/actions-setup-minikube/issues/7
> https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md
> https://www.mirantis.com/blog/how-to-install-cri-dockerd-and-migrate-nodes-from-dockershim/
> https://kubernetes.io/ru/docs/tasks/tools/install-minikube/
> https://microk8s.io/docs/getting-started
> https://docs.k3s.io/quick-start

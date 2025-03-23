# K8s cluster inside LXC Proxmox
**Language:** [🇷🇺](https://github.com/d3adwolf/kubernetes-in-lxc-on-proxmox/blob/main/README_RU.md) · 🇺🇸<br><br>
Instructions on how to deploy a working K8s cluster in a Proxmox LXC container.
### Preface
> [!NOTE]\
> This instruction is made on the basis of several articles, official documents and my own practice.<br>
> All references to original sources are listed at the end of the **README**.

### Tested on:
`Proxmox:`
> **Kernel Version:** Linux 6.5.11-7-pve (2023-12-05T09:44Z)<br>
> **Manager Version:** pve-manager/8.1.3/b46aac3b42da5d15

`Kubernetes:`
> **kubectl** v1.29.0<br>
> **crictl** v1.29.0<br>
> **cri-dockerd** v0.3.9
## Proxmox preparation
### Kernel modules
Let's plug into the kernel the recommended modules for running Docker containers and for K8s in general.

To do this, edit `/etc/modules` and add:
```bash
overlay
br_netfilter
ip_vs
nf_nat
xt_conntrack
```
I will describe a little bit why each module is needed:
- `overlay` - a module to control the transfer of network traffic between different network interfaces or subnets.
- `br_netfilter` - a module that provides traffic filtering at the bridge level.
- `ip_vs` - module for IP Virtual Server (IPVS) protocol management, which allows creating high-performance and fault-tolerant network services.
- `nf_nat` - a module that provides NAT (Network Address Translation) to redirect traffic between different network interfaces or subnets.
- `xt_conntrack` - a module to monitor the state of TCP/UDP connections and manage them at the network protocol level.

To avoid rebooting the node, we activate the modules via the command:
```bash
modprobe <module>
```
Let's check the active modules:
```bash
lsmod | grep -E 'overlay|br_netfilter|ip_vs|nf_nat|xt_conntrack'
```
It is also recommended to run the command to update the existing initramfs image that is used at system boot:
```bash
update-initramfs -u
```
> [!NOTE]\
> The **initramfs** image contains kernel modules, device drivers, scripts, and utilities needed for the system to work properly at boot time.
### Network traffic
Let's also make sure that `iptables` will correctly accept network traffic from all Proxmox nodes, for this purpose we will create a config with the permission to forward network traffic:
```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
Apply the parameters with the command:
```bash
sysctl --system
```
Check the changes in the node:
```bash
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables
```
### Swap file
Disable the swap partition for the moment:
```bash
swapoff -a
```
> [!NOTE]\
> It is common for **Kubernetes** to turn off the swap partition so that the cluster can properly estimate free RAM and not have performance issues.

And comment out the lines so that on the next boot, it won't be turned on:
```bash
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Create LXC container
In the Proxmox UI, let's start creating the container via "Create CT".
### General
Immediately check "Advanced" and uncheck "Unprivileged container".
It should look like this:
- [ ] Unprivileged container
- [X] Nesting
- [X] Advanced
### Template
Here we choose the image to taste and color, my choice was the current version of Ubuntu.
### Memory
Turn off swap in the container:
```
Swap (MiB): 0
```
### Network
The future cluster has **Network Policy** and other traffic limiting tools, so Firewall can be turned off in my opinion:
- [ ] Firewall

Make sure to give our node a static IP address so that it won't change after a while:
```
IPv4: Static
IPv4/CIDR: 192.168.0.10/24
Gateway (IPv4): 192.168.0.1
```
Naturally, this is just an example and you specify your local network.
### DNS
If you have configured or have a separate DNS server at home, which is fine, it's best to specify it, if your router is the primary gateway and DNS server, then skip this tab.
### Confirm
Let's take our time to start the container:
- [ ] Start after created

### Configure the LXC container
Now we need to prepare the container for proper operation of **K8s** cluster, you can install a convenient text editor in Proxmox node and in LXC container at once, I use `vim` as I know how to exit it:
```
apt install -y vim
```
### Actions outside the container
First, let's shut down the container and log on Proxmox under `root` via SSH into the `/etc/pve/lxc` directory, and then edit `<container id>.conf` via text editor, where **container id** is the identifier of our LXC container.

Add lines to the file:
```bash
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: "proc:rw sys:rw"
```
Let me tell you a little about these parameters:
- `lxc.apparmor.profile: unconfined` - sets the Apparmor profile for the container to "unconfined", which disables [AppArmor](https://www.apparmor.net/) in LXC.
- `lxc.cgroup.devices.allow:` a - allows the container root access to [cgroup](https://wiki.archlinux.org/title/cgroups).
- `lxc.cap.drop:` - disables automatic disabling of some capabilities for the container, which may be useful for some applications, see [LXC documentation](https://linuxcontainers.org/lxc/manpages/man5/lxc.container.conf.5.html) for details.
- `lxc.mount.auto: "proc:rw sys:rw"` - mounts the root partition /proc and /sys to R/W access for the container, which is usually necessary for the system to work correctly.


Now we need to flip the kernel boot configuration to the container, since `kubelet` uses the configuration to define the cluster environment settings.


Start the container and via `root` in Proxmox run the command:
```bash
pct push <container id> /boot/config-$(uname -r) /boot/config-$(uname -r)
```
Now let's create a symbolic reference for `/dev/kmsg`, since `kubelet` uses this for the journaling function, in LXC we have `/dev/console` for this, so we will reference it by creating a bash script in `/usr/local/bin/conf-kmsg.sh`:
```bash
#!/bin/sh -e
if [ ! -e /dev/kmsg ]; then
    ln -s /dev/console /dev/kmsg
fi

mount --make-rshared /
```
And let's configure the script to run once when the LXC container is started.


Create `/etc/systemd/system/conf-kmsg.service` with this content:
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
Make our script executable and enable the service:
```bash
chmod +x /usr/local/bin/conf-kmsg.sh
systemctl daemon-reload
systemctl enable --now conf-kmsg
```
### Customize the base environment
Let's update the installed packages and deliver those that will be useful for further customization:
```bash
apt update && apt upgrade -y
apt install -y wget curl conntrack
```
Let's remove the default firewall, because **K8s** uses other traffic management tools:
```bash
apt remove -y ufw && apt autoremove -y
```
### kubectl
Let's put a tool to interact with the Kubernetes API server, manage Kubernetes resources and workflows:
```bash
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
```
Let's check the installed version:
```bash
kubectl version --client
```
### helm
Let's put a Kubernetes package management tool that automates application deployment:
```bash
apt install -y git
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
rm get_helm.sh
```
Let's check the installed version:
```bash
helm version
```
> The current version can be found in the [Releases](https://github.com/helm/helm/releases) repository, but a more stable version for your OS may be installed.
### crictl
> [!WARNING]\
> Required for `minikube`, in other cases, install on demand.

Let's put a tool for managing containers and their resources in Kubernetes:
```bash
VERSION="v1.29.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```
Let's check the installed version:
```bash
crictl -v
```
> The current version can be found in the [Releases](https://github.com/kubernetes-sigs/cri-tools/releases) repository.
### containernetworking-plugins
> [!WARNING]\
> Required for `minikube`, in other cases put it on demand.


Let's put a set of plugins for container networking in Kubernetes:
```bash
CNI_PLUGIN_VERSION="v1.4.0"
CNI_PLUGIN_TAR="cni-plugins-linux-amd64-$CNI_PLUGIN_VERSION.tgz"
CNI_PLUGIN_INSTALL_DIR="/opt/cni/bin"


curl -LO "https://github.com/containernetworking/plugins/releases/download/$CNI_PLUGIN_VERSION/$CNI_PLUGIN_TAR"
mkdir -p "$CNI_PLUGIN_INSTALL_DIR"
tar -xf "$CNI_PLUGIN_TAR" -C "$CNI_PLUGIN_INSTALL_DIR"
rm "$CNI_PLUGIN_TAR"
```
> You can see the current version in the [Releases](https://github.com/containernetworking/plugins/releases) repository.
### cri-dockerd
> [!WARNING]\
> Required for `minikube` in conjunction with **Docker**, otherwise put it when using Docker as Container Runtime in K8s.

Let's put an adapter that provides compatibility between **Docker Engine** and **CRI** in Kubernetes:
```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.9/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
dpkg -i cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
rm -f cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```
Let's check the installed version:
```bash
cri-dockerd --version
```
> The current version can be found in the [Releases](https://github.com/Mirantis/cri-dockerd/releases) repository.
## Install Container Runtime
### Docker
> [!WARNING]\
> Required for `minikube`, in other cases we install on demand.

Install dependencies, add apt repository to the system:
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
Install Docker via `apt`:
```bash
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Let's check the installed version:
```bash
docker version
```
### containerd
Easy and simple installation via ``apt``:
```bash
apt install -y containerd
```
Let's check the installed version:
```bash
containerd --version
```
Create a folder for the configuration file:
```bash
mkdir /etc/containerd/
```
Set the default settings for the container configuration:
```bash
containerd config default > /etc/containerd/config.toml
```
To enable the use of **cgroup**, toggle the `systemdCgroup` parameter flag in `/etc/containerd/config.toml`:
```bash
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
Let's also verify our changes:
```bash
cat /etc/containerd/config.toml | grep SystemdCgroup
```
### CRI-O
Create variables with the current version of crio:
```bash
export OS=xUbuntu_22.04
export VERSION=1.24
```
> The current version can be found at [download.opensuse.org](https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/)

Install dependencies, add apt repository to the system:
```bash
apt install -y gnupg

echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

apt update
```
Installing crio via `apt`:
```bash
apt install -y cri-o cri-o-runc
```
Check the installed version:
```bash
crio -v
```
Now we need to disable [AppArmor](https://www.apparmor.net/) for crio:
```bash
sed -i 's/# apparmor_profile =\ "crio-default"/apparmor_profile \= "unconfined"/g' /etc/crio/crio.conf
```
Copy the config to work in `minikube`:
```bash
cp /etc/crio/crio.conf /etc/crio/crio.conf.d/02-crio.conf
```
Check the changes:
```bash
cat /etc/crio/crio.conf /etc/crio/crio.conf.d/02-crio.conf | grep apparmor_profile
```
Start crio and add it to autorun:
```bash
systemctl enable --now crio
```
## Install Kubernetes
### minikube
**Container Runtime**

The choice of container for Kubernetes (Container Runtime) depends on your requirements and preferences, but the most common containers for Kubernetes are Docker, containerd and CRI-O.

1. **Docker** is the most common container included in most Kubernetes distributions.

2. **containerd** is the second most popular container, also commonly used with Kubernetes. 

3. **CRI-O** - a container specifically designed to conform to the Kubernetes container interface (CRI).

To make up my mind, I created clusters of **K8s** under the same conditions with different Container Runtime, and this is what I got:

| Container Runtime  | Creating time (seconds) |
| ------------------ | ------------------------|
| Docker             | ~25                     |
| containerd         | ~22                     |
| CRI-O              | ~16                     |

**Installation**

Download the package and install:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
install minikube-linux-amd64 /usr/local/bin/minikube
rm -f minikube-linux-amd64
```
Let's install the recommended dependencies:
```bash
apt install -y ethtool socat
```
**Docker**

Now it's safe to start the cluster:
```bash
minikube start --vm-driver=none --extra-config=kubeadm.ignore-preflight-errors=SystemVerification --kubernetes-version=v1.29.0 --container-runtime=docker
```

**containerd**.

Running `minikube` via **containerd** requires **docker-cli**.
As per the instructions above for **Docker**, do:
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
And just install **docker-cli**:
```bash
apt install -y docker-ce-cli
```
Now it's safe to start the cluster:
```bash
minikube start --vm-driver=none --extra-config=kubeadm.ignore-preflight-errors=SystemVerification --kubernetes-version=v1.29.0 --container-runtime=containerd
```
**crio**

To run `minikube` via **crio** requires **docker-cli**.

As per the instructions above for **Docker**, do:
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
And just put **docker-cli**:
```bash
apt install -y docker-ce-cli
```
Now it's safe to start the cluster:
```bash
minikube start --vm-driver=none --extra-config=kubeadm.ignore-preflight-errors=SystemVerification --kubernetes-version=v1.29.0 --container-runtime=crio
```

**Removal**

If you want to delete the cluster, just execute:
```bash
minikube delete
```
If you want to delete all profiles, then:
```bash
minikube delete --all
```
If you need to delete ``minikube``, the only options are:
```bash
rm -f /usr/local/bin/minikube
```
### microk8s
**Installation**

Let's install the service that manages snap packages:
```bash
apt install -y snapd
```
Install microk8s through it:
```bash
snap install microk8s --classic
```
Add a user to the microk8s group:
```bash
usermod -a -G microk8s $USER
chown -f -R $USER ~/.kube
```
Preferably, log in as a normal user:
```bash
su - $USER
```
See the status of the cluster:
```bash
microk8s status --wait-ready
```
Let's create an alias to avoid typing microk8s to work through kubectl:
```bash
alias kubectl='microk8s kubectl'
```
**Removal**

To delete microk8s execute:
```bash
snap remove microk8s
```
Don't forget the alias:
```bash
unalias kubectl
```
### K3s
**Installation**

Pretty straightforward installation here:
```bash
curl -sfL https://get.k3s.io | sh -
```
Create an alias to avoid typing k3s to work through kubectl:
```bash
alias kubectl='k3s kubectl'
```
Or overwrite the kubectl configs:
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```
**Removal**

To uninstall K3s, simply execute:
```bash
/usr/local/bin/k3s-uninstall.sh
```
Also, don't forget the alias:
```bash
unalias kubectl
```
### k0s
**Installation**

Here's a pretty straightforward installation:
```bash
curl -sSLf https://get.k0s.sh | sh
```
Install as a service:
```bash
k0s install controller --single
```
Start the cluster:
```bash
k0s start
```
Check the status of the cluster:
```bash
k0s status
```
Create an alias to avoid typing k0s to work through kubectl:
```bash
alias kubectl='k0s kubectl'
```
**Removal**

Stop the service:
```bash
k0s stop
```
Remove the k0s service and all dependencies:
```bash
k0s reset
```
If you need to remove ``k0s``, the only options are:
```bash
rm -f /usr/local/bin/k0s
```
Also don't forget about alias:
```bash
unalias kubectl
```
## Test any cluster to see if it's working properly
Execute:
```bash
kubectl get nodes && \
echo && \
kubectl get services && \
echo && \
kubectl get pods -A
```
The output should be the current state of the cluster, and if there is one, and `STATUS` = **Ready**, congratulations.
## Verify the network health of any cluster
Create an deployment:
```bash
kubectl create deployment hello-world --image=registry.k8s.io/echoserver:1.10
```
Create a service for the deployment:
```bash
kubectl expose deployment hello-world --type=NodePort --port=8080
```
Watch the feed start up and also watch its NodePort:
```bash
kubectl get pods -o wide
kubectl get service
```
> Look for this: 8080:XXXXX/TCP, where XXXXX is the NodePort.

Let's check the availability of the pod outside of Proxmox, on your laptop, run a `curl` query:
```bash
curl <ip adress>:XXXXX
```
Where `<ip adress>` is the IP address of the LXC container, and `XXXXX` is the external port of our pod.

The response request should come out like this:
> Hostname: hello-world-576c8bfdf8-c269c
>
>Pod Information:
> -no pod information available-.
>
>Server values:
> server_version=nginx: 1.13.3 - lua: 10008

And so on. 

After successful checks, let's remove the test deployment and the service.
```bash
kubectl delete services hello-world
kubectl delete deployment hello-world
```
This is the end of **K8s** deployment in LXC.
## Materials used
I highly recommend reading the resources below for a broader understanding of all the processes:
> Installing [Docker Engine](https://docs.docker.com/engine/)
>
> Installing [Kubernetes](https://kubernetes.io/docs/home/)
>
> Installing [minikube](https://minikube.sigs.k8s.io/docs/)
>
> Installing [microk8s](https://microk8s.io/docs/getting-started)
>
> Installing [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
>
> Installing [K3s](https://docs.k3s.io/quick-start)
>
> Installing [k0s](https://docs.k0sproject.io/head/install/)
>
> Installing [cri-o](https://github.com/cri-o/cri-o)
>
> Setting up [cgroup](https://rootlesscontaine.rs/getting-started/common/cgroup2/)
>
> Article [Garrett Mills' blog](https://garrettmills.dev/blog/2022/04/18/Rancher-K3s-Kubernetes-on-Proxmox-Container/)
>
> Instructions for running [microk8s in LXD](https://microk8s.io/docs/install-lxd)
>
> Instructions for running [Kubernetes in redOS](https://redos.red-soft.ru/base/server-configuring/container/kubernetes/kuber/)

## Plans
I will try to complete the article, both for myself and for all of you.

Here are my current plans for it:
- [X] Launch [k0s](https://docs.k0sproject.io/head/)
- [ ] Launch [kind](https://kind.sigs.k8s.io/) (currently unable to launch)
- [ ] Raise a cluster via [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [X] Set up [cri-o](https://cri-o.io/) support for `minikube`

Feel free to post your ideas in [Discussions](https://github.com/d3adwolf/kubernetes-inside-lxc-on-proxmox/discussions).

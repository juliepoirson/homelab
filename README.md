Homelab Kubernetes setup, using `kubeadmn` on Rapberry Pis. Used [official](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) docs as a reference and miscellaneous blog posts. For learning and function.

# High Level

- Compute: Raspberry Pi 5s
- CRI: containerd
- CNI: Cilium
- Helm: managing pods
- Argo: [GitOps](https://about.gitlab.com/topics/gitops/) which essentially syncs the actual cluster to config files in GitHub to

# Setting Up New Raspberry Pi 5

## All Nodes

### Assign Static IP

This allows us to more reliably connect the nodes together. All it requires is defining this in the Wifi router (go to gateway IP and set).

### Disable Swap

Keeps processes isolated from each other, can only use allocated amount of ram and if going over won't bleed into system.

Turn off with these commands (from the man page `man swap.conf`)

```bash
$ sudo mkdir -p /etc/rpi/swap.conf.d
$ sudo echo -e "[Main]\nMechanism=none" | sudo tee /etc/rpi/swap.conf.d/disable-swap.conf
$ sudo reboot
```

### Containerd

This is the standard for containerizing k8s. Follow install instructions [here](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-2-from-apt-get-or-dnf). How I did it using the Debian docs:

Setup docker key stuff since it distributes containerd

```bash
# Add Docker's official GPG key:
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl
$ sudo install -m 0755 -d /etc/apt/keyrings
$ sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

$ sudo apt-get update
```

Install containerd and add modules, most docs mention this but I don't see it in any k8s documentation.

```bash
$ sudo apt update && sudo apt install -y containerd.io
$ sudo mkdir -p /etc/containerd
$ containerd config default | sudo tee /etc/containerd/config.toml
$ cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
$ sudo modprobe overlay br_netfilter
$ sudo systemctl restart containerd
```

Next edit `/etc/containerd/config.toml` so that `SystemdCgroup = true`. Then run `sudo systemctl restart containerd`

### Networking

Mostly from [k8s docs](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

To manually enable IPv4 packet forwarding (needed because each pod has an IP that the host will forward to, I think):

```bash
# sysctl params required by setup, params persist across reboots
$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
$ sudo sysctl --system
```

### cgroups

cgroups allow partitioning of resources, obviously very important for k8s.

Easy way to enable cgroup on the pi.

```bash
$ sudo sed -i '$ s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1/' /boot/firmware/cmdline.txt
```

### Kubernetes Tooling

Followed [docs](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

```bash
$ sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
$ sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly

$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
$ sudo systemctl enable --now kubelet
```

## Control Plane Node

### Init

Skip kube-proxy for Cilium

```bash
$ sudo reboot # just in case
$ sudo kubeadm init --skip-phases=addon/kube-proxy

# follow instructions printed, something like
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Save output join command for later, 1Password works. Something like

```bash
$ ubeadm join 192.168.0.52:6443 --token w8m6hp.6t2tcjpjjkgoh9uo \
	--discovery-token-ca-cert-hash sha256...
```

### Helm

Install Helm (needed for cilium setup)

```bash
$ sudo apt-get install curl gpg apt-transport-https --yes
$ curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
$ echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
$ sudo apt-get update
$ sudo apt-get install helm
```

### Cilium

Following [docs](https://docs.cilium.io/en/latest/network/kubernetes/kubeproxy-free/)

```bash
# add helm cilium repo
$ helm repo add cilium https://helm.cilium.io/
$ helm repo update

# install cilium to cluster
$ API_SERVER_IP=<SERVER_IP> API_SERVER_PORT=6443 helm install cilium cilium/cilium \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=${API_SERVER_IP} \
    --set k8sServicePort=${API_SERVER_PORT}
```

### Remove Taint

Allows data plane (workers) to run on the control plane node.

```bash
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Install ArgoCD

One off installation to manage everything!

```bash
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

# Maintaining

### Connect to Control Node From Laptop

For running kubectl commands and maintaining, very useful for port forwarding. Note, to get this to work in Iterm2 on MacOS I had to give Iterm2 local network permissions, and to do that I had to restart Iterm2 and run `telnet <RPI_IP> 6443`. Not the most fun thing to figure out.

```bash
$ ssh $USERNAME@$PI_IP_ADDRESS$ "sudo cat /etc/kubernetes/admin.conf" > ~/.kube/config-pi
$ export KUBECONFIG=~/.kube/config-pi
```

# Local Linux preprations

make sure qemu vms are running with `-cpu host`


```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

```
curl -OL https://github.com/containerd/containerd/releases/download/v1.6.4/containerd-1.6.4-linux-amd64.tar.gz
```

```
sudo tar Cxzvf /usr/local containerd-*-linux-amd64.tar.gz
```

```
curl -OL https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
```

```
sudo mkdir -p /usr/local/lib/systemd/system
```

```
sudo cp containerd.service /usr/local/lib/systemd/system/
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

```
curl -OL https://github.com/opencontainers/runc/releases/download/v1.1.2/runc.amd64
```

```
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

```
curl -OL https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
```

```
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```
sudo apt-get update
```

```
sudo apt-get install -y kubelet=1.22.6-00 kubeadm=1.22.6-00 kubectl=1.22.6-00
```

```
sudo apt-mark hold kubelet kubeadm kubectl
```

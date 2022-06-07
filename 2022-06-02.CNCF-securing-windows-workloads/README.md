# Off-camera

These are the preparation steps that were done off-camera.

> If you are using QEMU, make sure vms are running with `-cpu host` argument.

You can find the recording [here](https://www.linkedin.com/posts/rramezanpour_cncf-on-demand-webinar-securing-windows-activity-6940078386488778752-kTk5?utm_source=linkedin_share&utm_medium=member_desktop_web)

**Since this is my corner of the GitHub I'm going to do some marketing for myself.**
If you like an step by step Azure course checkout my [Certified Calico Operator: Azure Expert](https://academy.tigera.io/course/certified-calico-operator-azure-expert/).



## AKS preparations

To follow AKS guide you need to have an active Azure account. I was using the Azure free account for the recording, if you like to open a free account [click here](https://azure.microsoft.com/free).

Already got an Azure account? [head to AKS tutorial then!](https://github.com/frozenprocess/Tigera-Presentations/blob/master/2022-06-02.CNCF-securing-windows-workloads/01_AKS.md)

## Local Linux preparations

Use the following command to add the required modules to your system. [More information here](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
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

Use the following command to download `containerd`. 
```
curl -OL https://github.com/containerd/containerd/releases/download/v1.6.4/containerd-1.6.4-linux-amd64.tar.gz
```

Un-archive the `containerd` archive file in your local folder.
```
sudo tar Cxzvf /usr/local containerd-*-linux-amd64.tar.gz
```

Use the following command to download the `containerd` service file.
```
curl -OL https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
```

Create a local folder for `systemd` services.
```
sudo mkdir -p /usr/local/lib/systemd/system
```

Copy the service file to your local lib folder that we just created.
```
sudo cp containerd.service /usr/local/lib/systemd/system/
```

Use the following commands to reload `systemd` and add `containerd` to system startup.
```
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

Download the appropriate `runc` binary for your platform.
```
curl -OL https://github.com/opencontainers/runc/releases/download/v1.1.2/runc.amd64
```

Use the following command to install `runc` on your vm.
```
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

Download the latest CNI plugins from containernetworking GitHub repo. 
```
curl -OL https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
```

Use the following command to create and unpack the CNI plugins.
```
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

Add Kubernetes gpg and deb repo address to your apt local sources file.
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Issue a repo update for changes to take place.
```
sudo apt-get update
```

To replicate the AKS cluster I was using the Kubernetes version `1.22.6`.
Use the following command to install kubelet,kubeadm and kubectl binaries on your local **LAB** VM.
```
sudo apt-get install -y kubelet=1.22.6-00 kubeadm=1.22.6-00 kubectl=1.22.6-00
```

Since packages can change over time, it is a good habit to always mark them.
```
sudo apt-mark hold kubelet kubeadm kubectl
```

Everything is set head to [local tutorial](https://github.com/frozenprocess/Tigera-Presentations/blob/master/2022-06-02.CNCF-securing-windows-workloads/02_local.md).

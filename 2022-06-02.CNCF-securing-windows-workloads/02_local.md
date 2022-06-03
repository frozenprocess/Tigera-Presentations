# Control plane node

Use the following command to initiate a Kubernetes cluster. 
```
sudo kubeadm init --pod-network-cidr=172.16.0.0/16
```

Use the following command to install `tigera-operator`.
```
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
```

Configure Calico installation manifest.
```
kubectl apply -f - <<EOF
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/v3.23/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 172.16.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
EOF
```

Turn on the `APIServer`.
```
kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}
EOF
```

Download `calicoctl`.
```
curl -L https://github.com/projectcalico/calico/releases/download/v3.23.1/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
```

Use `calicoctl` to change the IPAM affinity. This is a mandatory step to prevent Linux nodes from borrowing IP from Windows.
```
./calicoctl ipam configure --strictaffinity=true
```

Check the current installation manifest.
```
kubectl get installation default -o yaml
```

You should see a similar result.
```
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    bgp: Enabled
```
> **Note:** This is a mandatory step since Windows doesn't support `IP-IP` encapsulation natively.
Use the following command to change the default `IP-IP` encapsulation to `VXLAN`.
```
kubectl patch installation default --type=merge -p '{"spec": {"calicoNetwork": {"bgp": "Disabled"}}}'
```

# Windows

> **Note:** If you are experiencing huge delays in the downloading steps, use the following command to turnoff the powershell download progress bar which for some reason is tied to your download speed!
```
$ProgressPreference='SilentlyContinue'
```

Download the `containerd` installation script from Project Calico's documentation website.
```
Invoke-WebRequest https://projectcalico.docs.tigera.io/scripts/Install-Containerd.ps1  -OutFile .\Install-Containerd.ps1
```
> **Note:** Keep in mind that if you are using a fresh install you will need to restart your windows and do the following step twice.

Install containerd by issuing the following command.
```
.\Install-Containerd.ps1 -ContainerDVersion 1.6.4 
```

Create a folder called `k` in your windows directory. **Windows directory is usually labeled as C:**
```
mkdir c:\k
```

Transfer the kubeconfig file from your linux node to windows as it is [required](https://github.com/projectcalico/calico/blob/0c5c9f68059e3f2f5b82e72b47dcfbb82948d545/calico/scripts/install-calico-windows.ps1#L206) by the Calico installer to automate the token generation and other procedures during the installtion.
```
scp ubuntu@192.168.122.178:/home/ubuntu/.kube/config c:/k/config
```

Download the Calico installer script for Windows from Project Calico's documentation website.
```
Invoke-WebRequest https://projectcalico.docs.tigera.io/scripts/install-calico-windows.ps1 -OutFile c:\install-calico-windows.ps1
```

> **Note:** You need to `logout` before these variables are available in your environment.
Make sure you are 

Calico installer requires to know where are the [cni binary](https://github.com/projectcalico/calico/blob/0c5c9f68059e3f2f5b82e72b47dcfbb82948d545/calico/scripts/Install-Containerd.ps1#L27) files and what folder should be used for [their configurations](https://github.com/projectcalico/calico/blob/0c5c9f68059e3f2f5b82e72b47dcfbb82948d545/calico/scripts/Install-Containerd.ps1#L25).

You can use the following commands to add the expected values to your system.
```
[System.Environment]::SetEnvironmentVariable('CNI_BIN_DIR','C:\opt\cni\bin',[System.EnvironmentVariableTarget]::Machine)
[System.Environment]::SetEnvironmentVariable('CNI_CONF_DIR','C:\etc\cni\net.d',[System.EnvironmentVariableTarget]::Machine)
```

> **Note:** Keep in mind that you might loose your connection momentarly if you are using a remote connection to control your Windows node.

Everything is set let's run the installer.
```
c:\install-calico-windows.ps1 -KubeVersion 1.22.6 -CalicoBackend VXLAN
```

Use the following command to execute the `install-kube-services.ps1` script.
```
C:\CalicoWindows\kubernetes\install-kube-services.ps1
```

Now restart both `kubelet` and `kube-proxy` services.
```
restart-Service -Name kubelet
restart-Service -Name kube-proxy
```
Perfect!
You should be able to see the windows node in your control plane node list.

# Linux

Use the following manifest to install the windows container workload in your new environment.
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: win-web-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-container
  namespace: win-web-demo
  labels:
    app: win-web-container
spec:
  replicas: 1
  selector:
    matchLabels:
      app: win-web-container
  template:
    metadata:
      labels:
        app: win-web-container
    spec:
      containers:
      - name: web-container
        image: rezareza/wincontainer:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
      nodeSelector:
        kubernetes.io/os: windows
---
apiVersion: v1
kind: Service
metadata:
  name: win-container-service
  namespace: win-web-demo
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: win-web-container
  type: NodePort
EOF
```

Use the following command to get the Service IP address.
```
kubectl get svc -n win-web-demo
```

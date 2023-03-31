Underlying infrastructure
===

> **Note:** It is possible to run the cluster with only one node if the system does not have enough memory to support two nodes.

Use the following command to setup the lab environment:
```bash
multipass launch -n control -c 2 -m 2048M 22.04 --cloud-init https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/01.application-modernization/release/control-init.yaml
multipass launch -n node1 -c 2 22.04 --cloud-init https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/01.application-modernization/release/node-init.yaml
multipass launch -n node2 -c 2 22.04 --cloud-init https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/01.application-modernization/release/node-init.yaml
```

Use the following command to list the multipass instances:
```
multipass list
```

You should see a result similar to the following:
```
control                 Running           172.18.174.216   Ubuntu 22.04 LTS
node1                   Running           172.18.164.254   Ubuntu 22.04 LTS
node2                   Running           172.18.174.122   Ubuntu 22.04 LTS
```

Transferring the cluster key
===

Use the following command to transfer the Kubernetes authentication file to your PC:
```bash
multipass transfer control:/etc/rancher/k3s/k3s.yaml .
```

The first part of the process is to set the KUBECONFIG environment variable to the absolute path of the `k3s.yaml` file. This is done using the command "export KUBECONFIG=$(pwd)/k3s.yaml".

Once this is done, we need to extract the control-plane IP address from Multipass. This can be done using the "multipass list --format csv" command, which will display information about the "control" virtual machine, including its IP address. We can then substitute the IP address that is currently in the `k3s.yaml` file with the control-plane IP address that we just extracted.

After these steps are completed, kubectl will be able to use the updated `k3s.yaml` file to connect to the Kubernetes cluster running on the control-plane virtual machine, allowing us to run commands against the cluster.

Use the following command to modify the `k3s.yaml` file:
```bash
export KUBECONFIG=$(pwd)/k3s.yaml
CONTROL=$(multipass list --format csv | egrep control | cut -d, -f3)
sed -si "s/127.0.0.1/$CONTROL/" k3s.yaml
```


If you are following the tutorial on Windows use the following command in PowerShell: 
```powershell
$env:KUBECONFIG = "$(Get-Location)/k3s.yaml"
$CONTROL = (multipass list --format csv | Select-String control | ForEach-Object { $_.ToString().Split(",")[2] }).Trim()
(Get-Content k3s.yaml) -replace '127.0.0.1', $CONTROL | Set-Content k3s.yaml
```

Next step
===
In the next tutorial we are going to explore [Container security](../02.container-security/readme.md).

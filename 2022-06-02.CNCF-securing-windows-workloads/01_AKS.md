```
az account list
```

```
az feature register --namespace "Microsoft.ContainerService" --name "EnableAKSWindowsCalico" -o table
```

```
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/EnableAKSWindowsCalico')].{Name:name,State:properties.state}"
```

```
az provider register --namespace Microsoft.ContainerService
```

```
az group create --name calico-win-container --location australiaeast -o table
```

```
az aks create --resource-group calico-win-container --name CalicoAKSCluster --node-count 1 --node-vm-size Standard_B2s --network-plugin azure --network-policy calico --generate-ssh-keys -o table
```

```
az aks get-credentials --resource-group calico-win-container --name CalicoAKSCluster --admin
```

```
kubectl get nodes -L kubernetes.io/os
```

```
az aks nodepool add --resource-group calico-win-container --cluster-name CalicoAKSCluster --os-type Windows --name calico --node-vm-size Standard_B2s --node-count 1 --aks-custom-headers WindowsContainerRuntime=containerd
```

```
kubectl get nodes -L kubernetes.io/os
```

```
kubectl apply -f https://raw.githubusercontent.com/frozenprocess/wincontainer/main/Manifests/00_deployment.yaml
```

```
kubectl get svc win-container-service -n win-web-demo
```

```
watch kubectl get pods -n win-web-demo
```

```
kubectl get tigerastatus
```

```
kubectl apply -f https://raw.githubusercontent.com/frozenprocess/wincontainer/main/Manifests/01_apiserver.yaml
```

```
watch kubectl get tigerastatus
```

```
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-app-policy
spec:
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system", "calico-apiserver"}
  types:
  - Ingress
  - Egress
  egress:
    - action: Allow
      protocol: UDP
      destination:
        selector: k8s-app == "kube-dns"
        ports:
          - 53
EOF
```

```
az group delete -g calico-win-container
```

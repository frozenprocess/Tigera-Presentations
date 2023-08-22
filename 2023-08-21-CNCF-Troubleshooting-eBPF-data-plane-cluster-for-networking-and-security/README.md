```
multipass launch -n control -d 50G -c 2 -m 8G 22.04 --cloud-init control-init.yaml
multipass launch -n node1 -d 50G -c 2 -m 8G 22.04 --cloud-init node-init.yaml
```

```
multipass transfer control:/etc/rancher/k3s/k3s.yaml ./k3s.yaml
```

```
CONTROL=$(multipass list --format csv | egrep control | cut -d, -f3)
sed -si "s/127.0.0.1/$CONTROL/" k3s.yaml

export KUBECONFIG=$(pwd)/k3s.yaml

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```

```
kubectl create -f - <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "$CONTROL"
  KUBERNETES_SERVICE_PORT: "6443"
EOF
```

```
kubectl create -f -<<EOF
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
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

```
kubectl get pods -A
```

```
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF", "hostPorts":null}}}'
```

```
kubectl logs -n calico-system ds/calico-node -c calico-node | egrep -i "BPF enabled"
```

```
kubectl create deployment nginx --image=nginx
kubectl create service loadbalancer nginx --tcp 80:80
```

--iface=
```
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- calico-node -bpf counters dump
```

```
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- calico-node -bpf policy dump cali5cee378c3c9 all
```

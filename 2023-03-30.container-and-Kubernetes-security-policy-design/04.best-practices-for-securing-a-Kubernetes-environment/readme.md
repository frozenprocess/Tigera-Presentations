Policy design
===

Use the following command to explicitly allow every traffic in and out of your cluster:
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/00.allow-everything.yaml
```

```mermaid
flowchart LR
subgraph Cluster
    subgraph Node A
    subgraph calico-system
        A[calico-node]
        B[calico-typha]
    end
    A[calico-node] <-->|ingress\negress| C(127.0.0.1)
    B[calico-typha] <-->|ingress\negress| C(127.0.0.1)
    C[Host OS\n127.0.0.1]
    end
end
```

Use the following command to allow containers communicating with the localhost:
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/01.container-to-localhost.yaml
```

```mermaid
flowchart LR
subgraph Cluster
    subgraph namespace a
        A[Pods]
    end
    subgraph namespace b
        B[Pods]
    end
    subgraph kube-System
        C[Pods\nk8s-app == core-dns ]
    end
    A[Pods] -->|egress\n UDP 53| C[Pods\nk8s-app == core-dns ]
    B[Pods] -->|egress\n UDP 53| C[Pods\nk8s-app == core-dns ]
    A[Pods] x--x|ingress\negress| B[Pods]
end
subgraph External resources
    A[Pods] x--x|ingress\negress| Z[The Internet]
    B[Pods] x--x|ingress\negress| Z[The Internet]
    Z[The Internet]
end
```

Use the following command to restrict namespaced resources from reaching internet:
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/02.deny-app-policy.yaml
```

```mermaid
flowchart LR
subgraph Cluster
    subgraph Control Plane
            subgraph a[calico-system]
                A[Pods\nk8s-app == calico-apiserver]
            end
            A[Pods\nk8s-app == calico-apiserver] -->|egress\nTCP 6443| C[KUBE-APISERVER]
    end
    subgraph Node A...N
            subgraph b[calico-system]
                B[Pods\nk8s-app == calico-apiserver]
            end
            B[Pods\nk8s-app == calico-apiserver] -->|egress\nTCP 6443| C[KUBE-APISERVER]            
    end
end
```

Calico API server talks to the Kubernetes API server since api server is protected by hostendpoint policies global() is required
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/03.calico-api-to-kapi.yaml
```

```mermaid
flowchart LR
subgraph Control Plane
    subgraph a[calico-system]
        A["Pods\nk8s-app == calico-kube-controllers"] -->|egress\nTCP exposed port/s| B["Pods\nk8s-app == calico-apiserver"]
    end
end
```

calico-Kubernetes-controller connects to api-server on port 5443
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/04.calico-kube-controller-to-api.yaml
```

```mermaid
flowchart LR
subgraph Cluster
    subgraph control-plane
        subgraph a[calico-system]
            A["Pods\nk8s-app == calico-apiserver"]
        end
        C[HostEndpoint\nInterfaces] -->|egress\nTCP 5443| A["Pods\nk8s-app == calico-apiserver"]
    end
    subgraph Node A...N
        subgraph b[calico-system]
            B["Pods\nk8s-app == calico-apiserver"]
        end
        D[HostEndpoint\nInterfaces] -->|egress\nTCP 5443| B["Pods\nk8s-app == calico-apiserver"]
    end
end
```

```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/05.calico-components-to-apiserver.yaml
```

```mermaid
flowchart LR
subgraph Cluster
    subgraph Control
        A[HostEndpoint\nkubernetes.io/hostname == control]
    end
    subgraph Node 1...n
        B[HostEndpoint\nkubernetes.io/os]
    end
    B[HostEndpoint\nkubernetes.io/os]--ingress\nTCP 6443-->A[HostEndpoint\nkubernetes.io/hostname == control]
    A[HostEndpoint\nkubernetes.io/hostname == control]--egress\nTCP 6443-->B[HostEndpoint\nkubernetes.io/os]
end
```

Workers communication to Control PLane and Kubernetes API server
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/06.worker-nodes-to-kapi.yaml
```

```mermaid
flowchart LR
subgraph Cluster
    subgraph Control
        A[HostEndpoint\nkubernetes.io/hostname == control]
    end
    subgraph Node 1...n
        B["Pods\n k8s-app in {calico-node,calico-apiserver,calico-typha}"]
    end

    B["Pods\n k8s-app in {calico-node,calico-apiserver,calico-typha}"]--egress\nTCP 6443-->A[HostEndpoint\nkubernetes.io/hostname == control]
end
```

targeting srcnat traffic
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/07.srcnated-to-kapi.yaml
```

```mermaid
flowchart LR
subgraph Cluster
    subgraph Control
        A[HostEndpoint\nkuberenetes.io/os]
    end
    subgraph Node 1...n
        B["HostEndpoint\n has(kubernetes.io/os) && kubernetes.io/hostname == "control""]
    end
    A[HostEndpoint\nkuberenetes.io/os]--egress\nTCP 5473-->B["HostEndpoint\n has(kubernetes.io/os) && kubernetes.io/hostname == "control""]
end
```

Calico typha rules
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/08.worker-nodes-to-typha.yaml
```


Private registry
===

```mermaid
flowchart 
subgraph cluster
    A[control]
    B[node1...n]
end
subgraph External resources
    A[control] -->|egress\nTCP 5000| Z[private-repo\nBy IP]
    B[node1...n] -->|egress\nTCP 5000| Z[private-repo\nBy IP]

    Z[private-repo\nBy IP]
end
```

```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: private-registry-policy
spec:
  order: 1001
  egress:
  - action: Allow
    protocol: TCP
    destination:
      nets:
      - $REGISTRY_IP/32
      ports:
      - 5000
```

Enabling HostEndpoint
===

```bash
kubectl patch kubecontrollersconfiguration default --type=merge --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```

```powershell
kubectl patch kubecontrollersconfiguration default --type=merge --patch='{\"spec\": {\"controllers\": {\"node\": {\"hostEndpoint\": {\"autoCreate\": \"Enabled\"}}}}}'
```

Default deny
===

```
kubectl delete -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/conf42/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/00.allow-everything.yaml
```

Next step
===
In the next tutorial we are going to briefly look at [monitoring](../05.monitoring/readme.md).

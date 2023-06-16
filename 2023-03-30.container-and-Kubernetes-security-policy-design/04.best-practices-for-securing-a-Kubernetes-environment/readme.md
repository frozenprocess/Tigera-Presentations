Policy design
===
Kubernetes provides a framework for defining and applying policies to your cluster, but it is up to the administrator to create and enforce those policies. However, on its own, Kubernetes doesn't enforce these policies, and it delegates this to the container networking interface (CNI) plugin. 

One of the key default behaviors in Kubernetes is that all pods are allowed to communicate with each other by default. This means that if a pod is running a vulnerable application, other pods in the same cluster may also be at risk. To mitigate this risk, it is recommended to create network policies that restrict the traffic between pods based on their labels. However, this behavior changes to a default deny the moment that you add a policy to your cluster. When a network policy is added to a Kubernetes cluster, it overrides the default allow-all behavior and enforces the rules (with the help of your CNI) defined in the policy. This means that if a pod tries to communicate with another pod that is not explicitly allowed by the policy, the communication will be denied.

## Policy structure and predefined attributes

### Policy header & kind
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
```

```
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
```

```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
```
### Policy types

```
spec:
  types:
  - Ingress
  - Egress
```

### Rules
Each policy can contain one or more rule, and these rules can affect incoming `ingress` or outgoing `egress` traffic behavior of an endpoint (more about endpoints later). 

The following example is a rule that affects ingress traffic:
```
spec:
  ingress:
    destination:
      selector: has(some-label)
```

### Selectors
Both Kubernetes NetworkPolicy and Calico policies offer multiple selectors that can be used together to explicitly identify and affect a traffic flow.

```

```

## Policy precedence

In Calico, policy priority is determined based on the order assigned to each policy, with lower order policies taking precedence over higher order policies. This means that if you have two policies in a Calico-enabled Kubernetes cluster, one that denies traffic with a order of 10 and the other that allows traffic with a order of 20, the traffic will be denied since the lower order policy takes precedence over the higher order policy.

The following illustration is the behavior that will take place in such a scenario:
```mermaid
flowchart LR
   A[A resource attempting\n to connect]-. Calico looks for the lowest order policy .-> B[Policy with the order of 10\n denies the traffic.\n]
```

In Calico, you can specify the order by adding the `order` field to the policy's `spec` section. For example:
```
spec:
  order: 1
```

#### Kubernetes policy precedence
When you create a Kubernetes NetworkPolicy and a Calico policy in the same cluster, Calico will automatically assign the KNP a default order number of `1000` since Kubernetes policy resources do not provide an explicit way to set the order of policy evaluation and enforcement.

If you want the KNP to be evaluated and enforced before any Calico policies, you will need to set the order number of your Calico policy to a value higher than `1000`. For example, you can set the order number of your Calico policy to `1001` or higher to ensure that it is evaluated and enforced after the KNP.

> **Note:** It is important to carefully manage policy order in Calico to ensure that your policies are evaluated and enforced correctly in your Kubernetes cluster.


#### Calico unordered policies
You might now be wondering what happens when a Calico policy is created with an undefined order number?
In such a case Calico policies will automatically have an order number equal to infinity, and if you have two or more of these policies the tie breaker rule will apply to determine their priority.

#### The tie breaker rule
In a scenario where you have multiple Calico policies without an order number or with the equal order numbers, Calico policy engine will compare the policy names as a tiebreaker to determine the order of policy evaluation and enforcement.

For example, if you have two unordered policies named `alpha` and `zulu`, the policy engine will first evaluate and enforce the policy named `alpha` and then the policy named `zulu`, or if you have two policies named `aac` and `aab` policy `aab` will be evaluated first.

#### The Log exception
Calico allows you to create iptables log rules to log packets that match certain criteria. These log rules can be written inside Calico Policy resources and These log policies can be assigned an order number, but it is important to note that the rule that comes after a log action will be the final decider and no other rule will be evaluated in that policy. Therefore, you should ensure that the policy after a log rule is configured correctly to avoid unintended consequences. 

In the following example, after logging the flow with `some-label` annotation, this policy would also allow flows that are incoming from a source that is annotated with the `block=one` label but the second allow will not take effect since this policy starts with a `Log` action and that particular flow might be denied by the other policies.
```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: log-everything 
spec:
  order: 100000
  ingress:
  - action: Log
    selector: has(some-label)

  - action: Allow
    selector: has(some-label)
    source: 
      selector: blockone == "one"

  - action: Allow
    selector: has(some-label)
    source:
      selector: blocktwo == "two"
```

## Preventing a lockout
As a best practice whenever you are trying to experiment with policies in any environment it is best to implement a `safe-mode` strategy by first explicitly permitting the traffic and then writing your deny rules. This helps to eliminate the chance of accidentally locking yourself out of a cluster if a misconfigured policy is applied.

> **Note:** If you are following this tutorial on Windows and PowerShell use the YAML files in [this](https://github.com/frozenprocess/Tigera-Presentations/tree/master/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment) directory to create the policies.

Use the following command to explicitly allow every traffic in and out of your cluster:
```
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: final-allow-everything 
spec:
  order: 100001
  ingress:
  - action: Allow
  egress:
  - action: Allow
EOF
```

It is worth mentioning, that Calico also implements a default fail-safe, which is a policy with a list of essential ports and services that are required for your cluster to function. Click [here](https://docs.tigera.io/calico/latest/network-policy/hosts/protect-hosts#failsafe-rules), if you like to know more about Calico's fail safe.


Default egress policy for namespaced resources
===
In Calico, a Workload Endpoint (WEP) is a virtual interface that is automatically assigned to VMs, containers, or other workloads running in a Kubernetes cluster. These interfaces are used to enforce network policies and to route traffic from or to your workloads. This allows the traffic originating from your workloads to be easily manipulated by Kubernetes NetworkPolicies or Calico policy resources. Calico also provides a host endpoint (HEP) resource that can be used to enforce a particular network behavior on host networking cards and processes that are running on the host OS.

> **Note:** The default deny in the tutorial is crafted with the assumption that you will be enabling host endpoints policies, which allow for more fine-grained control over network traffic. If you are not interested in using host endpoints policies, you can use [this](https://docs.tigera.io/calico/latest/network-policy/get-started/kubernetes-default-deny) example. 


The following policy example will restrict all ingress and egress traffic for namespace resources:
```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-app-policy
spec:
  order: 1001
  namespaceSelector: has(projectcalico.org/name)
  types:
  - Ingress
  - Egress
```

## Permitting the intended


The following policy example will permit DNS flows for namespace resources:
```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: permit-dns
spec:
  order: 1001
  namespaceSelector: has(projectcalico.org/name)
  egress:
  - action: Allow
    protocol: UDP
    destination:
      selector: 'k8s-app == "kube-dns"'
      ports:
      - 53
```


## Multiple rules


```
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-app-policy
spec:
  order: 1001
  namespaceSelector: has(projectcalico.org/name)
  types:
  - Ingress
  - Egress
  egress:
  - action: Allow
    protocol: UDP
    destination:
      selector: 'k8s-app == "kube-dns"'
      ports:
      - 53
EOF
```

The following image is an illustration for `deny-app-policy`:
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

Workloads to Host OS
===


Use the following command to allow containers communicating with the localhost:
```
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-containers-access-to-localhost 
spec:
  order: 1000
  selector: has(kubernetes.io/os)
  ingress:
  - action: Allow
    destination:
      nets:
      - 127.0.0.0/8
  egress:
  - action: Allow
    destination:
      nets:
      - 127.0.0.0/8
EOF
```

The following image is an illustration for `allow-containers-access-to-localhost`:
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

Workloads to Kubernetes components
===


Calico API server communicates with the Kubernetes API server to view or modify cluster information. `global()` is required
```
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: calico-apiserver-to-kapi
  namespace: calico-apiserver
spec:
  order: 1000
  selector: 'k8s-app == "calico-apiserver"'
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: has(kubernetes.io/os)
      namespaceSelector: global()
      ports:
      - 6443
EOF
```

The following image is an illustration for `calico-apiserver-to-kapi`:
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

Workloads in the same namespace
===


calico-Kubernetes-controller connects to api-server on port 5443
```
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: calico-kube-controllers-to-apiserver
  namespace: calico-system
spec:
  order: 1000
  selector: 'k8s-app == "calico-kube-controllers"'
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: k8s-app == "calico-apiserver"
EOF
```

```mermaid
flowchart LR
subgraph Cluster
    subgraph Control Plane
        subgraph a[calico-system]
            A["Pods\nk8s-app == calico-kube-controllers"] -->|egress\nTCP exposed port/s| B["Pods\nk8s-app == calico-apiserver"]
        end
    end
    subgraph Node A...N
        subgraph c[calico-system]
            D["Pods\nk8s-app == calico-kube-controllers"] -->|egress\nTCP exposed port/s| E["Pods\nk8s-app == calico-apiserver"]
        end
    end
end
```

calico-Kubernetes-controller
===

```
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: calico-components-to-apiserver
spec:
  order: 1000
  selector: has(kubernetes.io/os)
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: 'k8s-app == "calico-apiserver"'
      ports:
      - 5443
EOF
```

The following image is an illustration for `calico-components-to-apiserver`:
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

Nodes communication to Kubernetes API server
===

```
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: worker-nodes-to-kapi
spec:
  order: 1000
  selector: has(kubernetes.io/os)
  ingress:
  - action: Allow
    protocol: TCP
    destination:
      selector: has(kubernetes.io/os) && kubernetes.io/hostname == "control"
      ports:
      - 6443
  # kube-scheduler or other component talking to the local loopback
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: has(kubernetes.io/os) && kubernetes.io/hostname == "control"
      ports:
      - 6443
EOF
```

The following image is an illustration for `worker-nodes-to-kapi` policy:
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


Calico-components to Kubernetes API server
===

```
apiVersion: projectcalico.org/v3  
kind: GlobalNetworkPolicy
metadata:
  name: srcnated-to-kapi
spec:
  order: 1000
  selector: has(k8s-app) && k8s-app in {"calico-node","calico-apiserver","calico-typha"}
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: has(kubernetes.io/os) && kubernetes.io/hostname == "control"
      ports:
      - 6443
```

The following image is an illustration for `srcnated-to-kapi` policy:
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

Calico Typha communication to and from Kubernetes API server
===

Calico typha rules
```
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: worker-nodes-to-typha
spec:
  order: 1000
  selector: has(kubernetes.io/os)
  ingress:
  - action: Allow
    protocol: TCP
    destination:
      selector: has(kubernetes.io/os) && kubernetes.io/hostname == "control"
      ports:
      - 5473
  # kube-scheduler or other component talking to the local loopback
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: has(kubernetes.io/os) && kubernetes.io/hostname == "control"
      ports:
      - 5473
EOF
```

The following image is an illustration for `worker-nodes-to-typha` policy:
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

External IPset and IP CIDR
===

```
kubectl create -f -<<EOF
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
EOF
```

The following image is an illustration for `private-registry-policy` policy:
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

Enabling HostEndpoint
===
has(kubernetes.io/os) && kubernetes.io/hostname == "control"

Use the following command to enable host endpoint policies.
```bash
kubectl patch kubecontrollersconfiguration default --type=merge --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```

If you are following the tutorial on Windows use the following command in PowerShell: 
```powershell
kubectl patch kubecontrollersconfiguration default --type=merge --patch='{\"spec\": {\"controllers\": {\"node\": {\"hostEndpoint\": {\"autoCreate\": \"Enabled\"}}}}}'
```

Default deny
===

Use the following command to drop every communication that has not been explicitly allowed:
```
kubectl delete -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment/00.allow-everything.yaml
```

Next step
===
In the next tutorial we are going to briefly look at [monitoring](../05.monitoring/readme.md).

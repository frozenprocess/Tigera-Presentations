# Before we begin
Install Kind

# Cluster preparation

```
kind create cluster --config cluster.yml
```

```
kubectl get node | egrep -i ready
```

```
kubectl get node -o yaml | egrep -i cni
```

```
        message:Network plugin returns error: cni plugin not initialized'
        message:Network plugin returns error: cni plugin not initialized'
```

```
kubectl create -f https://projectcalico.docs.tigera.io/archive/v3.22/manifests/tigera-operator.yaml
```

```
kubectl apply -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: None
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

After a moment both nodes will be `Ready`, you can verify this by executing the following command:
```
kubectl get node -o wide
```

# Install stars application
Stars is a simple application that can demo policies.

Use the following command to install the stars app:
```
kubectl apply -f stars.yaml
```

It takes a bit for stars to be fully deployed on a cluster. Use the following command to verify deployment:
```
kubectl rollout status deployments/frontend -n stars
```

Open the following url in a browser
> **Note**: In some oses ip might differ, you can verify your node IP by executing `kubectl get node -o wide`.
```
http://172.18.0.2:30002
```
You should see a `ui` with three pods that can freely talk to each other.

# Kubernetes network policy

Establish isolation
```
kubectl apply -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
 name: default-deny
 namespace: stars
spec:
 podSelector:
   matchLabels: {}
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
 name: default-deny
 namespace: client
spec:
 podSelector:
   matchLabels: {}
EOF
```
After some moment if you refresh the `ui` page it should go dark because `default-deny` policies are preventing `management-ui` from establishing connection.

## Permitting ui
Add `management-ui` to the list of permitted traffics.
```
kubectl apply -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
 namespace: stars
 name: allow-ui
spec:
 podSelector:
   matchLabels: {}
 ingress:
   - from:
       - namespaceSelector:
           matchLabels:
             role: management-ui
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
 namespace: client
 name: allow-ui
spec:
 podSelector:
   matchLabels: {}
 ingress:
   - from:
       - namespaceSelector:
           matchLabels:
             role: management-ui
EOF
```
Refresh the `ui` you should see three pods in isolation.
Great everything work, but not so fast!

Use the following command to run a ping from `client` Pod.
```
kubectl exec -it deployments/client -n client -- ping -c 4 8.8.8.8
```
Client pod can connect to a remote server.


Remove the default-deny policies.
```
kubectl delete networkpolicy -n client default-deny
kubectl delete networkpolicy -n stars default-deny
```

> **Bonus**: If you check the `ui` you will see nothing changed. It is important to note Calico default behaviour is set to `Allow` but when you add a policy this will change to an implicit deny at the end of rules.

# Calico global network policy

> **Note**: I've added `management-ui` as an exemption to this rule. If you like to know why there are some exemption here I would highly recommend reading [this document](https://projectcalico.docs.tigera.io/security/kubernetes-default-deny).

Use the following command to establish Isolation in the cluster:
```
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-app-policy
spec:
  order: 100
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system", "calico-apiserver", "management-ui"}
  types:
  - Ingress
  - Egress
  egress:
  # allow all namespaces to communicate to DNS pods
  - action: Allow
    protocol: UDP
    destination:
      selector: 'k8s-app == "kube-dns"'
      ports:
      - 53
EOF
```

Run the ping test again.
```
kubectl exec -it deployments/client -n client -- ping -c 4 8.8.8.8
```

You should see a result similar to:
```
PING 8.8.8.8 (8.8.8.8): 56 data bytes
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss
command terminated with exit code 1
```

Great we have successfully isolated the Pods.


# Shift-left

## Create security namespace
In Kubernetes, namespaces provide a scope for resource names. Namespaces can also be used to divide resources between users in a cluster.

Use the following command to create a namespace for security team.
```
kubectl create namespace security
```
## Create service account
Now that we have a security namespace let’s create a user called `security-member` and assign him to our `security`.
```
kubectl create serviceaccount security-member --namespace security
```
## Export user kubeconfig settings
In order to authenticate a token based user we have to present a token which is stored as a secret object in Kubernetes when that user is created.

The following command will export our user secret name and store it in the `SECRET` environment variable.
```
SECRET=`kubectl get serviceaccount security-member --namespace security --output jsonpath="{.secrets[].name}"`
```
Now that we know our user token name we can export the token using the following command and store it in a `TOKEN` environment variable.
> **Note**: use `-D` if you are using MacOs.
```
TOKEN=`kubectl get secret $SECRET --namespace security --output jsonpath="{.data.token}" | base64 -d`
```
We will also need our API server IP address and port in order to create a context for our kubeconfig file. Following command will extract the IP and PORT number of our Kubernetes API server and store it in the `API_SERVER` environment variable.
```
API_SERVER=`kubectl config view --output jsonpath="{.clusters[].cluster.server}"`
```

Since kind API Server uses SSL we have to use ca certificate information to communicate with API server.
> **Note**: use `-D` if you are using MacOs.
```
kubectl get secret $SECRET --namespace security --output jsonpath="{.data.ca\.crt}" | base64 -d > ca.crt
```

Now that we have all the required information to generate a kubeconfig file we can create a cluster using the following command:
```
kubectl config set-cluster van-meetup \
 --embed-certs=true \
 --server=$API_SERVER \
 --certificate-authority=./ca.crt
```

Create user and assign the token to it.
```
kubectl config set-credentials security-member --token=$TOKEN
```

Create a context and associate our newly generated cluster and user with that context.
```
kubectl config set-context security-context \
 --cluster=van-meetup \
 --user=security-member \
 --namespace=default
```

Since our new user has no assigned roles, it can not access any resources in our cluster. We can easily verify this by using some simple `get` commands.
```
kubectl --context security-context get networkpolicies --all-namespaces
kubectl --context security-context get caliconetworkpolicy -A
```

In K8s, RBAC Groups are used to associate similar users to certain permissions. In this example, our group will include every service account who are members of “security” namespace.
```
kubectl create -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: security-calico
rules:
- apiGroups: ["projectcalico.org"]
  resources:
    - networkpolicies
  verbs:
    - get
    - list
    - create
    - update
    - delete
    - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: security-k8s
rules:
- apiGroups: ["networking.k8s.io"]
  resources: 
    - networkpolicies
  verbs:
    - get
    - list
    - create
    - update
    - delete
    - patch
EOF
```

Bind the permissions to security group by executing the following command:
```
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sec-team-k8s-role-binder
subjects:
- kind: Group
  name: system:serviceaccounts:security
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: security-k8s
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sec-team-calico-role-binder
subjects:
- kind: Group
  name: system:serviceaccounts:security
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: security-calico
  apiGroup: rbac.authorization.k8s.io
EOF
```


```
kubectl --context=security-context apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  namespace: stars
  name: b2f-allow
spec:
  selector: role == 'backend'
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: role == 'frontend'
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  namespace: stars
  name: f2b-allow
spec:
  selector: role == 'frontend'
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: role == 'backend'
      ports:
      - 6379
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  namespace: client
  name: c2f-allow
spec:
  selector: role == 'client'
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: role == 'frontend'
      namespaceSelector: role == 'stars'
      ports:
      - 80
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  namespace: stars
  name: f2c-allow
spec:
  selector: role == 'frontend'
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: role == 'client'
      namespaceSelector: role == 'client'
EOF
```

```
kubectl --context security-context get cnp -A
```

## Stars team

Stars team permissions should be slightly different, they should only be able to create Kubernetes network policy inside their own namespace.

First add a `stars-member` service account to the namespace. 
```
kubectl create serviceaccount stars-member --namespace stars
```

Export the `stars-member` credentials.
```
SECRET=`kubectl get serviceaccount stars-member --namespace stars --output jsonpath="{.secrets[].name}"`
```

```
API_SERVER=`kubectl config view --output jsonpath="{.clusters[].cluster.server}"`
```

```
kubectl get secret $SECRET --namespace stars --output jsonpath="{.data.ca\.crt}" | base64 -d > ca.crt
```

```
kubectl config set-cluster van-meetup \
 --embed-certs=true \
 --server=$API_SERVER \
 --certificate-authority=./ca.crt
```

```
TOKEN=`kubectl get secret $SECRET --namespace stars --output jsonpath="{.data.token}" | base64 -d`
```

```
kubectl config set-credentials stars-member --token=$TOKEN
```

```
kubectl config set-context stars-member \
 --cluster=van-meetup \
 --user=stars-member \
 --namespace=stars
```

Create and bind the permissions to the user.
```
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: stars-k8s
  namespace: stars
rules:
- apiGroups: ["networking.k8s.io"]
  resources: 
    - networkpolicies
  verbs:
    - get
    - list
    - create
    - update
    - delete
    - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: stars-role-binder
  namespace: stars
subjects:
- kind: ServiceAccount
  name: stars-member
  namespace: stars
roleRef:
  kind: Role
  name: stars-k8s
  apiGroup: rbac.authorization.k8s.io
EOF
```

Let's try creating a rule in `client` namespace:
```
kubectl --context=stars-member apply -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: client
  name: deny-all
spec: 
  podSelector: {}
EOF
```

You should see a result similar to:
```
from server for: "STDIN": networkpolicies.networking.k8s.io "deny-all" is forbidden: User "system:serviceaccount:stars:stars-member" cannot get resource "networkpolicies" in API group "networking.k8s.io" in the namespace "client"
```
Seems like stars team members are not permitted to change client namespace policies.

Create Kubernetes policies to allow flow of traffic.
```
kubectl --context=stars-member apply -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      role: backend 
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: client
      ports:
        - protocol: TCP
          port: 6379
EOF
```

# Clean up

```
kind delete clusters van-meetup
```
I've used the instructions from [Certified Calico Azure Expert](https://academy.tigera.io/course/certified-calico-operator-azure-expert/) to create a self-managed cluster, but following instructions should work on any Kubernetes cluster equipped with Calico.

Use the following command to install the Tigera-operator.
```
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
```

Use the following command to install Calico. 
```
kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
 name: default
spec:
 calicoNetwork:
   bgp: Disabled
   nodeAddressAutodetectionV4:
     canReach: 1.1.1.1
   ipPools:
   - blockSize: 26
     cidr: 192.168.0.0/16
     encapsulation: VXLAN
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

Use the following command to monitor your Calico installation process.
```
kubectl get tigerastatus
```

Deploy Google's microservices demo.
```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/release/v0.3.6/release/kubernetes-manifests.yaml
```

Use the following command to check the deployment process.
```
kubectl get deployments
```

```
kubectl patch FelixConfiguration default --type=merge --patch '{"spec": {"policySyncPathPrefix": "/var/run/nodeagent"}}'
```

Use the following command to download istio.
```
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.10.2 sh -
```

browse into the istio folder.
```
cd istio-1.10.2
```

Remove the taints on the master so that you can schedule pods on it.
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Issue the following command to extend your Kubernetes cluster with Istio.
```
./bin/istioctl install
```

Confirm the prompt by pressing down the "Y".
```
This will install the Istio 1.10.2  profile with ["Istio core" "Istiod" "Ingress gateways"] components into the cluster. Proceed? (y/N)
```

Use the following command to apply Istio.
```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/release/v0.3.6/release/istio-manifests.yaml
```

Next, we are going to create a PeerAuthentication resource to define how traffic should be tunneled.
```
kubectl create -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default-strict-mode
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF
```


```
curl https://projectcalico.docs.tigera.io/manifests/alp/istio-inject-configmap-1.10.yaml -o istio-inject-configmap.yaml
```

```
kubectl patch configmap -n istio-system istio-sidecar-injector --patch "$(cat istio-inject-configmap.yaml)"
```

```
kubectl apply -f https://projectcalico.docs.tigera.io/manifests/alp/istio-app-layer-policy-envoy-v3.yaml
```

```
kubectl get service -n=istio-system istio-ingressgateway
```
Open the website.

```
kubectl get globalnetworkpolicy -o yaml
```

```
kubectl apply -f 01_frontend_allow.yaml
kubectl apply -f 02_frontend_deny.yaml
```

```
kubectl get pods -A
```

```
kubectl label namespace default istio-injection=enabled
```

Restart the `frontend` deployment.
```
kubectl rollout restart deployment/frontend
```
If you check the Pod deployment by issuing `kubetl get pods` frontend pods are stuck at initialization.
This is due to the deprecation of `FlexVolume` in recent releases of Kubernetes.

Use the following command to install the CSI driver.
```
kubectl apply -f https://projectcalico.docs.tigera.io/manifests/csi-driver.yaml
```

Restart the deployment again.
```
kubectl rollout restart deployment/frontend
```

Now try opening the website again, home-page should be visible but `/cart' will throw an error.

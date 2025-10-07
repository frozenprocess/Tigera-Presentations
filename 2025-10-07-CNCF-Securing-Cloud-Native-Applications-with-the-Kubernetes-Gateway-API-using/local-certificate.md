A working example of Calico Ingress Gateway in a local environment
====

# Requirements

- Docker
- k3d
- kubectl
- curl

> **Note**: There seems to be an issue when you are using macos, in such an environment the `Verify the installation` step doesn't work. A workaround will be posted shortly.

# Spin up the cluster

```bash
k3d cluster create \
  my-calico-cluster \
  -s 1 -a 2 \
  --k3s-arg '--flannel-backend=none@server:*' \
  --k3s-arg '--disable-network-policy@server:*' \
  --k3s-arg '--disable=traefik@server:*' \
  --k3s-arg '--cluster-cidr=192.168.0.0/16@server:*'
```

# Install Calico

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml
```

```bash
kubectl rollout status -n tigera-operator deployment/tigera-operator
```

You should see a result similar to the following:
```bash
deployment "tigera-operator" successfully rolled out
```
## Configure Calico Installation resource

```bash
kubectl create -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  registry: quay.io/
  # Configures Calico networking.
  kubeletVolumePluginPath: None
  calicoNetwork:
    containerIPForwarding: Enabled
    bgp: Disabled
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

Run the following command to verify the installation process:
```bash
kubectl get tigerastatus
```

You should see a result similar to the following:
```bash
NAME        AVAILABLE   PROGRESSING   DEGRADED   SINCE
apiserver   True        False         False      1s
calico      True        False         False      11s
ippools     True        False         False      101s
```

## Deploy the Microservices Demo

```bash
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/refs/heads/release/v0.10.2/release/kubernetes-manifests.yaml
```

```bash
kubectl  rollout status deployment/frontend
```

```bash
kubectl get svc | egrep frontend
```

Note: IPs may differ in your environment.
You should see a result similar to the following:
```bash
frontend                ClusterIP      10.43.56.88     <none>                          80/TCP         5m12s
frontend-external       LoadBalancer   10.43.78.95     10.89.1.3,10.89.1.4,10.89.1.5   80:30432/TCP   5m12s
```

We need to delete this service, since two ports can not work in the same place.
```bash
kubectl delete svc frontend-external
```

## Enabling Calico Ingress Controller

```bash
kubectl create -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: GatewayAPI
metadata:
 name: default
EOF
```

Use the following command to verify:
```bash
kubectl wait --for=condition=Available tigerastatus gatewayapi
```

You should see a result similar to the following:
```bash
tigerastatus.operator.tigera.io/gatewayapi condition met
```

### Install cert-manager

```bash
kubectl create -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
```

verify
```bash
kubectl wait --timeout=5m --for=condition=Available -n cert-manager deployment/cert-manager
```

```bash
kubectl patch deployment -n cert-manager cert-manager --type='json' --patch '
[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--enable-gateway-api"
  }
]'
```

```bash
kubectl create -f - --<<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: my-selfsigned-ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    secretName: root-secret
EOF
```



Use the following command to create a gateway:
```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-issuer
  name: calico-demo-gw
spec:
  gatewayClassName: tigera-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
    hostname: secure-demo.local
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        group: ""
        name: example-https
EOF
```

Use the following command to create `HTTPRoute`:
```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-redirect
  namespace: default
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: calico-demo-gw
    namespace: default
    port: 80
    sectionName: http
  rules:
    - matches:
      - path:
          type: PathPrefix
          value: /.well-known/acme-challenge/
      backendRefs:
      - group: ""
        kind: Service
        name: frontend
        namespace: default
        port: 80
        weight: 1
    - matches:
      - path:
          type: PathPrefix
          value: /
      filters:
      - type: RequestRedirect
        requestRedirect:
          scheme: https
          statusCode: 301
          port: 443
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-access
  namespace: default
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: calico-demo-gw
    namespace: default
    port: 443
    sectionName: https
  rules:
    - matches:
      - path:
          type: PathPrefix
          value: /
      backendRefs:
      - group: ""
        kind: Service
        name: frontend
        namespace: default
        port: 80
        weight: 1
EOF
```

## Verify the installation

```bash
export GATEWAY_HOST=$(kubectl get gateway/calico-demo-gw -o jsonpath='{.status.addresses[0].value}')
```

```bash
curl -I http://secure-demo.local/ --connect-to "secure-demo.local:80:${GATEWAY_HOST}"
```

You should see a result similar to the following:
```bash
HTTP/1.1 301 Moved Permanently
location: https://secure-demo.local/
date: Tue, 27 May 2025 20:48:08 GMT
transfer-encoding: chunked
```

```bash
curl -I https://secure-demo.local/ --connect-to "secure-demo.local:443:${GATEWAY_HOST}" -k
```

You should see a result similar to the following:
```bash
HTTP/2 200 
set-cookie: shop_session-id=e0c86657-f04a-40cc-a57e-62244a86eb14; Max-Age=172800
date: Tue, 27 May 2025 20:48:52 GMT
content-type: text/html; charset=utf-8
```

## Credits
Shoutout to all the amazing people who write the [Envoy tasks](https://gateway.envoyproxy.io/docs/tasks/quickstart/).

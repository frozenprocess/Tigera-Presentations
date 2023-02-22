# How to install Calico in AWS

> This tutorial is based on [this](https://docs.tigera.io/calico/3.24/getting-started/kubernetes/self-managed-public-cloud/aws) Calico documentation.

## Prerequsits 
[eksctl](https://eksctl.io/)
An AWS account [free tier](https://aws.amazon.com/free/)
k8s-bnech-suite [multi-stream branch](https://github.com/frozenprocess/k8s-bench-suite)

## Creating an EKS control plane

```
eksctl create cluster --name cncf-calico-demo-aws --without-nodegroup
```

## Removing the default CNI 

```
kubectl delete daemonset -n kube-system aws-node
```

## Installing Calico 

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml
```

### Configuring Calico

```
kubectl create -f - <<EOF                                                                        
kind: Installation
apiVersion: operator.tigera.io/v1
metadata:
  name: default
spec:
  kubernetesProvider: EKS
  cni:
    type: Calico
  calicoNetwork:
    bgp: Disabled
EOF
```

## Adding node groups

```
eksctl create nodegroup --cluster cncf-calico-demo-aws --name ng-amd64 --nodes 1  --nodes-min 1  --nodes-max 3 --node-type m5n.xlarge --max-pods-per-node 100
```

```
eksctl create nodegroup --cluster cncf-calico-demo-aws --name ng-arm64 --nodes 1  --nodes-min 1  --nodes-max 3 --node-type m6g.xlarge --max-pods-per-node 100
```

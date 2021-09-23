# Introduction

These are the commands that I used to spin up my EKS cluster for Akraino presentation.
By following these insturcitons you should be able to reproduce everything that I presented in my session. *Fingers crossed*

> I used `eksctl` version `0.63.0`

# Before we begin

Make sure you have `eksctl` on your systsem.

How to install [eksctl](https://eksctl.io/introduction/#installation)

# Cluster creation

Let's use `eksctl` to create an EKS control plane without any nodes. (takes ~16 Minutes)

```
eksctl create cluster --name calico-multi --without-nodegroup --version 1.21
```

Attach 1 `m5.large` node to the EKS control plane with the following command: (takes ~4 Minutes)
```
eksctl create nodegroup  --cluster calico-multi  --name ng-amd64   --nodes 1  --nodes-min 1  --nodes-max 3  --node-type m5.large
```

At this point we have a cluster that uses `amd64` manifests if we try to add an arm instance to it it will spit out an error.
Since I'm big fan of errors let's check the errors for clues.
```
eksctl create nodegroup  --cluster calico-multi  --name ng-arm64   --nodes 1  --nodes-min 1  --nodes-max 3  --node-type m6g.large
```

Allright seems like we have to update the componentes before we are allowed to add an arm64 instance

use the following command to update `coredns`, `aws-node` pods.
```
eksctl utils update-coredns --cluster calico-multi --approve
eksctl utils update-aws-node --cluster calico-multi --approve
eksctl utils update-kube-proxy --cluster calico-multi --approve
```

At the time of this writing using a regular update for `kube-proxy` results in an error. As a workaround we can delete the `kube-proxy` daemonset
```
kubectl delete daemonset -n kube-system kube-proxy
```

Install a new version of `kube-proxy
```
eksctl create addon --name kube-proxy --cluster calico-multi --force
```

Let's try again and add an `arm64` instance to our cluster. (takes ~6 Minutes)
> **Note:** eksctl might still compalin about components being outdated since we updated everything manually it is safe to append `--skip-outdated-addons-check=true` to our command and ignore this error.
```
eksctl create nodegroup  --cluster calico-multi  --name ng-arm64   --nodes 1  --nodes-min 1  --nodes-max 3  --node-type m6g.large --skip-outdated-addons-check=true
```

Apply redis manifest
```
kubectl apply -f redis.yaml
```

# Benchmarks

Benchmarks stats for x86 instance
```
kubectl exec -it redis-x86 -- redis-benchmark -q
```

Benchmarks stats for arm64 instance
```
kubectl exec -it redis-arm -- redis-benchmark -q
```

# Cleanup

Use the following command to teardown the ekscluster.
```
eksctl delete cluster calico-multi
```

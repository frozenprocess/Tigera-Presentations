# Introduction

These are the commands that I used to spin up my EKS cluster for my UTAH meetup presentation.
By following these instructions you should be able to reproduce everything that I presented in my presentation session. *Fingers crossed*

# Before we begin

This tutorial uses Amazon AWS platform to create the required resources. Make sure you have a valid AWS account and `eksctl` configured on your system.

> This tutorial requries `eksctl` version `0.70.0` or later.

How to install [eksctl](https://eksctl.io/introduction/#installation)

**Note:** You can verify your version of `eksctl` using `eksctl version` command.

# Cluster creation

First, let's use `eksctl` to create an EKS control plane with two `amd64` nodes. (takes ~20 Minutes)

```
eksctl create cluster --config-file - <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: calico-utah-multi
  region: us-west-2
  version: '1.21'
nodeGroups:
  - name: ng-amd64
    instanceType: m5.large
    minSize: 1
    maxSize: 3
    desiredCapacity: 2
    amiFamily: Bottlerocket
    ami: auto-ssm
    iam:
      attachPolicyARNs:
      - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
      - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
      - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
      - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
EOF
```

At this point we have a cluster that uses `amd64` manifests, if we try to add an arm instance to the mix it will result in an error message.

Use the following command to trigger the error.
```
eksctl create nodegroup  --cluster calico-utah-multi --name ng-arm64   --nodes 1  --nodes-min 1  --nodes-max 3  --node-type m6g.large
```

Let's search the error for clues.
```
2021-10-18 13:46:10 [✖]  to create an ARM nodegroup kube-proxy, coredns and aws-node addons should be up to date. Please use `eksctl utils update-coredns`, `eksctl utils update-kube-proxy` and `eksctl utils update-aws-node` before proceeding.
```

Seems like we have to update some components before we are allowed to add an arm64 instance.

use the following command to update `coredns`, `aws-node` manifests.
```
eksctl utils update-coredns --cluster calico-utah-multi --approve
eksctl utils update-aws-node --cluster calico-utah-multi --approve
eksctl utils update-kube-proxy --cluster calico-utah-multi --approve
```

Previous update instructions adds `arm64` to the deployment manifest which can be verified using the following command:
```
kubectl get daemonset -n kube-system aws-node -o json | jq '.spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms'
```

At the time of this writing using a regular update for `kube-proxy` may result in errors. As a workaround we can do the update procedure manually for this component.
```
kubectl delete daemonset -n kube-system kube-proxy
```

Install and up to date `kube-proxy` by using the following command:
```
eksctl create addon --name kube-proxy --cluster calico-utah-multi --force
```

Let's try again and add an `arm64` instance to our cluster. (takes ~6 Minutes)

> **Note:** Eksctl might still complain about components being outdated, now that we have updated everything it is safe to ignore this error by appending the `--skip-outdated-addons-check=true` argument to our command.

```
eksctl create nodegroup  --cluster calico-utah-multi --name ng-arm64   --nodes 1  --nodes-min 1  --nodes-max 3  --node-type m6g.large --skip-outdated-addons-check=true
```

Let's verify our multi architectural cluster node architecture by issuing the following command:
```
kubectl get nodes -L kubernetes.io/arch
```

Labeling nodes so apps can bedistributed into right spot.
```
kubectl label node $(kubectl get node -L kubernetes.io/arch | egrep arm | awk '{print $1}') role=worker
kubectl label node $(kubectl get node -L kubernetes.io/arch | egrep amd | awk 'FNR==1{print $1}') role=benchmark
kubectl label node $(kubectl get node -L kubernetes.io/arch | egrep amd | awk 'FNR==2{print $1}') role=worker
```

Use the `demo-app.yaml` manifest to create the servers and services required for the benchmark part of this tutori
```
kubectl apply -f demo-app.yaml
```

There are many ways to benchmark a webserver, I'm going to use `wrk` since good people at NGiNX already have a great [blogpost](https://www.nginx.com/blog/nginx-plus-sizing-guide-how-we-tested/) on how to use it.

**Note:** In these benchmarks `wrk` messures how fast NGiNX can transfer a 1Kb file (similar to CSS or JS file) to a client.
```
kubectl run --rm -it --image docker.io/rezareza/wrk-benchmarker --overrides='{"spec": {"nodeSelector": {"role":"benchmark"},"imagePullPolicy":"IfNotPresent"}}'  benchmarker -- -t 2 -c 50 -d 180s http://my-x86-service.default/1kb.bin
```

You should see a similar result for your `x86` test:
```
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    12.14ms    6.23ms 112.07ms   84.76%
    Req/Sec     2.76k   159.43     2.90k    93.36%
  495510 requests in 3.00m, 461.60GB read
Requests/sec:   2751.53
Transfer/sec:      2.56GB
```

Use the following command to checkout the arm64 performance:
```
kubectl run --rm -it --image docker.io/rezareza/wrk-benchmarker --overrides='{"spec": {"nodeSelector": {"role":"benchmark"},"imagePullPolicy":"IfNotPresent"}}'  benchmarker -- -t 2 -c 50 -d 180s http://my-arm64-service.default/1kb.bin
```

You should see a similar result for your `arm64` test:
```
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.59ms  485.56us  20.74ms   90.62%
    Req/Sec    31.46k     5.40k   38.42k    46.89%
  5635134 requests in 3.00m, 4.46GB read
Requests/sec:  31305.48
Transfer/sec:     25.38MB
```

# Calico eBPF

**Note**: Calico arm64 eBPF support is a community driven effort and it can only be enabled in Calico 3.21 or above.

Install the latest `tigera-opreator` in your cluster 
using the following command:
```
curl -s https://docs.projectcalico.org/master/manifests/tigera-operator.yaml | sed -s 's#kubernetes.io/os: linux#beta.kubernetes.io/arch: amd64#' | kubectl apply -f -
```

Create installation resources required by operator to start the process.
```
kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  cni:
    type: AmazonVPC
  flexVolumePath: /var/lib/kubelet/plugins
  # Enables provider-specific settings required for compatibility.
  kubernetesProvider: EKS
EOF
```

In eBPF mode `kube-proxy` pods can be safely removed from the cluster since Calico will take over all their responsibility. This requires to tell Calico how to directly communicate with the Kubernetes API-server.

Use the following command to get the API server address:
```
kubectl get cm -n kube-system kube-proxy -o yaml | grep server
```

Now that we have the Kubernetes API-Server FQDN address, let's create a configmap to tell Calico about it by using the following command:
```
kubectl apply -f - <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "a0637e880a8850d727fe8637f14572ca.gr7.us-west-2.eks.amazonaws.com"
  KUBERNETES_SERVICE_PORT: "443"
EOF
```

Remove `kube-proxy` pods from the cluster : 
```
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
```

Change Calico dataplane to eBPF by issuing the following command:
```
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF", "hostPorts":null}}}'
```

One of the ways to verify Calico `eBPF` data plane is turned on in your cluster is by checking the `calico-node` pod logs.   
```
kubectl logs -n calico-system daemonset/calico-node  | egrep -i 'bpf enabled'
```
You should see a similar output:
```
2021-10-19 18:52:09.275 [INFO][53] felix/int_dataplane.go 359: BPF enabled, configuring iptables layer to clean up kube-proxy's rules.
2021-10-19 18:52:09.285 [INFO][53] felix/int_dataplane.go 556: BPF enabled, starting BPF endpoint manager and map manager.
2021-10-19 18:52:09.439 [INFO][53] felix/int_dataplane.go 1697: BPF enabled, disabling unprivileged BPF usage.
```
That is it, you have a multi architectrual EKS cluster equipped with Calico and eBPF data plane!


# eBPF benchmarks
Same benchmarks using Calico eBPF dataplane

## eBPF x86
```
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     3.31ms    1.23ms 215.16ms   94.67%
    Req/Sec    15.26k     1.18k   18.58k    85.11%
  2733258 requests in 3.00m, 2.16GB read
Requests/sec:  15181.73
Transfer/sec:     12.31MB
```

## eBPF arm64
```
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.76ms    1.20ms 254.43ms   98.30%
    Req/Sec    28.74k     4.73k   35.25k    46.61%
  5147665 requests in 3.00m, 4.07GB read
Requests/sec:  28592.70
Transfer/sec:     23.18MB
```

# Clean up
Use the following command to remove the cluster that you created by following this tutorial.
```
eksctl delete cluster calico-utah-multi 
```
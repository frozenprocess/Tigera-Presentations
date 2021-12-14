# Introduction

These are the commands that I used to spin up my EKS cluster for my UTAH meetup presentation; By following these instructions you should be able to reproduce everything that I presented in my presentation session. *Fingers crossed*

Click on [NGiNX](https://github.com/frozenprocess/Tigera-Presentations/tree/master/2021-10-15.Utah.Meetup.Multiarch.clusters.networking) or [Redis](https://github.com/frozenprocess/Tigera-Presentations/tree/master/2021-09-24.Akraino.Multiarch.clusters) if you are looking for the benchmark procedure.

# Before we begin

This tutorial uses the Amazon AWS platform to create the required resources to run a Kubernetes cluster so make sure you have a valid AWS account and `eksctl` configured on your system.

> This tutorial requires `eksctl` version `0.70.0` or later.

How to install [eksctl](https://eksctl.io/introduction/#installation)

**Note:** You can verify your version of `eksctl` using `eksctl version` command.

> If you like to have a smooth installation experience make sure you are using `eksctl` version `0.72.0` or above.

# Cluster creation

First, let's use `eksctl` to create a single EKS control plane and one `amd64` node.
```
eksctl create cluster --config-file - <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: calico-cncf-multi
  region: us-west-2
  version: '1.21'
nodeGroups:
  - name: ng-amd64
    instanceType: m5.large
    minSize: 1
    maxSize: 2
    desiredCapacity: 1
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

Add an `arm64` node to the mix.
```
eksctl create nodegroup  --cluster calico-cncf-multi --name ng-arm64   --nodes 1  --nodes-min 1  --nodes-max 2  --node-type m6g.large
```

## Components update
**Note:** Please follow this section if you are using `eksctl` version  `0.72.0` or older. 

In older releases of eksctl you might run into an error when trying to add an `arm64` node group. 

The error output will be similar to this:
```
2021-10-18 13:46:10 [✖]  to create an ARM nodegroup kube-proxy, coredns and aws-node addons should be up to date. Please use `eksctl utils update-coredns`, `eksctl utils update-kube-proxy` and `eksctl utils update-aws-node` before proceeding.
```

In that case, you need to update some of the components before you can create your node group.

use the following command to update `coredns`, `aws-node` manifests.
```
eksctl utils update-coredns --cluster calico-cncf-multi --approve
eksctl utils update-aws-node --cluster calico-cncf-multi --approve
eksctl utils update-kube-proxy --cluster calico-cncf-multi --approve
```

Previous update instructions add's `arm64` to the deployment manifest and it can be verified using the following command:
```
kubectl get daemonset -n kube-system kube-proxy -o json | jq '.spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms'
```

## Kube-proxy update problem

At the time of this writing using a regular update for `kube-proxy` may result in errors. As a workaround, we can do the update procedure manually for this component.
```
kubectl delete daemonset -n kube-system kube-proxy
```

Install and up to date `kube-proxy` by using the following command:
```
eksctl create addon --name kube-proxy --cluster calico-cncf-multi --force
```

Let's try again and add an `arm64` instance to our cluster. (takes ~6 Minutes)

> **Note:** Eksctl might still complain about components being outdated but since we have updated everything manually it is safe to ignore this error by appending the `--skip-outdated-addons-check=true` argument to the previous `nodegroup` creation command.

```
eksctl create nodegroup  --cluster calico-cncf-multi --name ng-arm64   --nodes 1 --nodes-min 1  --nodes-max 3  --node-type m6g.large --skip-outdated-addons-check=true
```

# Verify node architecture

**Note:** Bottlerocket OS `ARM64` kernel might be lower than `AMD64`. 

Let's verify our Kubernetes environment by issuing the following command:
```
kubectl get nodes -L kubernetes.io/arch
```

# Install Calico

Install the latest `tigera-opreator` in your cluster 
using the following command:
```
kubectl apply -f https://docs.projectcalico.org/archive/v3.21/manifests/tigera-operator.yaml
```

Create installation resources required by the tigera-operator to start the process.
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

You can verify the installation process using the following command:
```
kubectl rollout status daemonset calico-node -n calico-system
```

# Install microservices demo on your cluster

I've prepared a multiarch friendly manifest for the Google microservices demo.
Apply the following manifest to install microservices on your cluster:
```
kubectl apply -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2021-12-06.CNCF-Multiarch.migration/microservices-multiarch.yaml
```


# Cleanup

Use the following command to remove the cluster that you created by following this tutorial.
```
eksctl delete cluster calico-cncf-multi
```

# Preparing a container for ARM

**Note**: This section is based on docker.

Download a local clone of `google-microservices-demo` project and navigate into the `src` folder inside of `microservices-demo`.
```
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo/src
```

Each folder contains the files to build it's container image.
Let's head into `adservice` and skim the file names inside of it.
```
cd adservice
```

There is a `Dockerfile` in this folder which indicates that we should be able to build this container using docker and `docker build` instruction.

Now in order to create a multiplatform manifest using docker we can use the `buildx` feature.

Use the following command to create a `buildx` builder.
```
docker buildx create --use
```

Use the following command to build the container
```
docker buildx build --platform linux/amd64,linux/arm64 -t myawesomedockerusername/adservice:v0.3.2 .
```
**Note**: You can change the `myawesomedockerusername` to your docker username and append `--push` to the previous command in order to send the image to your own docker hub repository!


## Not so fast!

Sometimes you might run into a problem where docker build complains and outputs an error simillar to :
```
standard_init_linux.go:228: exec user process caused: exec format error
```
This is because while builder can pull architecture spesific layers from docker hub it can not run `arm` binaries if your `dockerfile` requires it.

One of the ways to solve this issue is to use a Linux kernel feature called [binfmt_misc](https://www.kernel.org/doc/html/latest/admin-guide/binfmt-misc.html).
By registering a mask, flag and an interpreter you will be able to run any type of binary files inside your Linux!

Don't worry you don't have to remember any values, projects such as [multiarch](https://github.com/multiarch/qemu-user-static) can take care of everything!

use the following command to enable `arm` binary support inside the container
```
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

# Final thoughts
ARM architecture seems to be a great alternative to x86 systems; and because of its efficiency,  companies are investing in its future for cloud computing. Transforming applications and clusters might enable cost savings without compromising on performance.

Building container images
===
Image scanning is the process of analyzing container images for potential security vulnerabilities or misconfigurations. It involves inspecting the image's contents and metadata to identify known vulnerabilities in the underlying software components, libraries, and dependencies. Image scanning tools can also check for compliance with security policies and best practices.

> **Note:** For this section make sure that you are inside the 02.container-security folder.

Use the following command in the root directory of this repo to create the loose application:
```
docker build -t container-security/test-image:loose -f container-security/dockerfile.loose container-security/.
```

Use the following command in the root directory of this repo to create the minimal application:
```
docker build -t container-security/test-image:minimal -f container-security/dockerfile.minimal container-security/.
```

Spot the vulnerability with tigera-scanner
===
Tigera-scanner is a closed source image scanner. 

Click [here](https://docs.tigera.io/calico-cloud/image-assurance/scan-image-registries) and follow the instructions to install `tigera-scanner`. 

Use the following command to list the images in your docker local storage:
```
docker image ls
```

You should see a result similar to the following:
```
REPOSITORY                               TAG            IMAGE ID       CREATED        SIZE
container-security/test-image            loose          61f1ceaa35c8   7 hours ago    8.9MB
container-security/test-image            minimal        22b7cea2e90d   7 hours ago    1.85MB
```

> **Note**: If you are following this tutorial in Windows copy the `tigera-scanner` inside your `wsl` instance, and make sure to prepend `wsl` to the following command.

Use the following command to check the image vulnerabilities:
```
tigera-scanner scan container-security/test-image:loose
```

Use the following command to check the image vulnerabilities:
```
tigera-scanner scan container-security/test-image:minimal
```

In this section we've used the Tigera-scanner to issue a simple scan on two images. It is worth noting that Tigera-scanner has a lot more capabilities and its full potential can be unlocked by using [Calico Cloud](https://www.calicocloud.io/home).

<img src="https://www.calicocloud.io/static/media/screen-prod-info-ia.97827030045a13f2de75.png" alt="Calico Cloud" height="500rm">

> Click [here](https://www.calicocloud.io/home) to learn more about Calico Cloud.

Kubernetes Node status
===

Let's get back to our cluster and evaluate its status.
```
kubectl get nodes
```

You should see a result similar to the following:
```
NAME      STATUS     ROLES                  AGE     VERSION
node1     NotReady   <none>                 5m6s    v1.23.16+k3s1
node2     NotReady   <none>                 3m53s   v1.23.16+k3s1
control   NotReady   control-plane,master   5m39s   v1.23.16+k3s1
```
The `NotReady` status indicates that our cluster requires a CNI.

> **Note:** If you're interested in delving deeper into this issue, clicking on this [link](../../2022-07-13.Calico-installation/readme.md) will take you to a comprehensive guide that offers a detailed explanation of the situation.

Let's install Calico to fix this situation.

Use the following command to install the `tigera-operator`:
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
```

Public image registry
===
In containerization, a container image registry is a platform used to store and manage container images. Public container image registries, such as Docker Hub, are widely used and provide a convenient way to share and distribute images.

Use the following command to explore the `tigera-operator` manifest that we just installed on the cluster:
```bash
curl -L https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml | egrep 'image:'
```

If you are following the tutorial on Windows use the following command in PowerShell: 
```powershell
(Invoke-WebRequest https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml).Content.ToString() -split '\n'  | Select-String -Pattern 'image:'
```

You should see a result similar to:
```
image: quay.io/tigera/operator:v1.29.0
```

In this case, the "operator" image is hosted on the Quay.io registry, which is a public registry for container images. The version of the image is v1.29.0 and it is designed, and managed by [Tigera](https://www.tigera.io/tigera-products/calico/) the company behind the open source project Calico.


Private image registry
===
While public container image registries are widely used and provide a convenient way to share and distribute images, there are certain situations where using a private registry is necessary. One example of such a scenario is in industries that are subject to strict regulatory requirements, such as healthcare, finance, and government. These industries may have specific data protection and compliance regulations that require sensitive data to be stored and managed in a secure manner.

Use the following command to run a new VM to host the `private-repository` for this tutorial:
```
multipass launch -n private-repo -d 60G 22.04 --cloud-init https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/01.application-modernization/release/registry-init.yaml
```

In Linux use the following commands to extract the `private-repo` IP address:
```bash
CALICO_VERSION="v3.25.0"
REGISTRY_IP=$(multipass list --format csv | egrep private-repo | cut -d, -f3)
REGISTRY=$REGISTRY_IP:5000
```

If you are following the tutorial on Windows use the following commands in PowerShell: 
```powershell
$CALICO_VERSION = "v3.25.0"
$REGISTRY_IP = (multipass list --format csv | Select-String private-repo | ForEach-Object { $_.ToString().Split(",")[2] }).Trim()
$REGISTRY=$REGISTRY_IP+":5000"
```

> **Note:** You will only need to add the private-registry IP address to your workstation docker configurations, all multipass instances are already configured with the expected certificate information.

When you use a container runtime environment such as Docker, it is important to ensure that your private registry is secured using SSL/TLS. However, if you are using a self-signed or insecure registry, you may encounter issues with pulling images or pushing images to the registry. In order to use an insecure registry with Docker, you must add the IP address of the registry to the Docker daemon's list of insecure registries. This can be done by adding the registry's IP address to the insecure-registries section of the Docker daemon configuration file (/etc/docker/daemon.json on Linux systems).

It is important to note that using an insecure registry can be a security risk, as it allows unencrypted communication between the Docker client and the registry. Therefore, it is recommended to use a self-signed registry only for testing or development purposes, and to secure any production registries using SSL/TLS. If you like to know more about self-signed configurations click [here](https://docs.docker.com/registry/insecure/)


Use the following commands to push the required images to the lab private registry:
```
docker pull calico/typha:$CALICO_VERSION
docker tag calico/typha:$CALICO_VERSION $REGISTRY/calico/typha:$CALICO_VERSION
docker push $REGISTRY/calico/typha:$CALICO_VERSION

docker pull calico/kube-controllers:$CALICO_VERSION
docker tag calico/kube-controllers:$CALICO_VERSION $REGISTRY/calico/kube-controllers:$CALICO_VERSION
docker push $REGISTRY/calico/kube-controllers:$CALICO_VERSION

docker pull calico/cni:$CALICO_VERSION 
docker tag calico/cni:$CALICO_VERSION $REGISTRY/calico/cni:$CALICO_VERSION
docker push $REGISTRY/calico/cni:$CALICO_VERSION

docker pull calico/node-driver-registrar:$CALICO_VERSION 
docker tag calico/node-driver-registrar:$CALICO_VERSION $REGISTRY/calico/node-driver-registrar:$CALICO_VERSION
docker push $REGISTRY/calico/node-driver-registrar:$CALICO_VERSION

docker pull calico/csi:$CALICO_VERSION 
docker tag calico/csi:$CALICO_VERSION $REGISTRY/calico/csi:$CALICO_VERSION
docker push $REGISTRY/calico/csi:$CALICO_VERSION

docker pull calico/node:$CALICO_VERSION 
docker tag calico/node:$CALICO_VERSION $REGISTRY/calico/node:$CALICO_VERSION
docker push $REGISTRY/calico/node:$CALICO_VERSION

docker pull calico/pod2daemon-flexvol:$CALICO_VERSION 
docker tag calico/pod2daemon-flexvol:$CALICO_VERSION $REGISTRY/calico/pod2daemon-flexvol:$CALICO_VERSION
docker push $REGISTRY/calico/pod2daemon-flexvol:$CALICO_VERSION

docker pull calico/apiserver:$CALICO_VERSION 
docker tag calico/apiserver:$CALICO_VERSION $REGISTRY/calico/apiserver:$CALICO_VERSION
docker push $REGISTRY/calico/apiserver:$CALICO_VERSION
```

Installation resource
===
After the Tigera operator is installed, it periodically scans for the installation resource to determine the desired state of Calico. This means that the operator will continuously monitor the installation resource to ensure that the current state of Calico matches the desired state specified in the installation resource.

If any changes are made to the installation resource, such as updating the desired version of Calico or modifying the network policies, the Tigera operator will detect these changes and update the Calico configuration accordingly.

In addition to these core configurations, there are several other configurations that can be customized to fit Calico for your environment. For example, the registry key in the spec tree sets the prefix of all Calico components to a customized value of choice. This can be useful in scenarios where you want to use a private registry to host your Calico images or have a naming convention that differs from the default value.
```
  registry: private-repo:5000/
```
Other customizations include configuring the logging level and format, modifying the IP pools used by Calico, and enabling various features and plugins. By customizing these configurations, you can tailor Calico to meet the specific requirements of your environment.

> **Note:** More details about registry and installation settings can be found [here](https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation).

Use the following command to create an installation resource with the `private-registry` settings:
```
kubectl apply -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  registry: private-repo:5000/
  calicoNetwork:
    bgp: Disabled
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

Once you have created the installation resource, use the following command to query the Calico installation process:
```
kubectl wait --timeout=5m --for=condition=Available tigerastatus --all
```

You can use the following command to verify the image repository that was used for Calico:
```bash
kubectl describe -n calico-system ds/calico-node  | egrep "image:"
```

If you are following the tutorial on Windows use the following commands in PowerShell: 
```powershell
kubectl describe -n calico-system ds/calico-node  | Select-String "image:"
```

Next step
===
In the next tutorial we are going to briefly look at [Network segmentation](../03.network-segmentation/readme.md).

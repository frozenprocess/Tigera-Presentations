Namespace isolation
===
Namespace isolation is a feature in Kubernetes that allows you to create logical partitions within a cluster. These partitions, or namespaces, provide a way to isolate resources and workloads from each other, allowing for better organization, resource management, and security.

Each namespace has its own set of resources, including pods, services, and storage, which are only visible and accessible within that namespace. This means that different namespaces can have overlapping names for their resources, without causing conflicts or confusion.

Use the following command to create a namespace for monitoring which we will deploy, later in this tutorial:
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/03.network-segmentation/01.monitoring-ns.yaml
```

Role Based Access Control (RBAC)
===
Role-based access control (RBAC) is a security mechanism in Kubernetes that provides fine-grained control over access to resources and actions within a cluster. RBAC allows you to define roles and permissions for different users or groups, which can be used to restrict access to sensitive resources and prevent unauthorized actions. RBAC is based on the principle of least privilege, which means that users or groups should only have access to the resources and actions that are necessary for their roles and responsibilities. 

Use the following command to create a cluster role that will be used for monitoring, later in this tutorial:
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/03.network-segmentation/02.cluster-role.yaml
```

Service account
===
In Kubernetes, a service account is an identity that is used by a pod or a set of pods to interact with the Kubernetes API server and other Kubernetes resources. Service accounts are used to authenticate and authorize access to the Kubernetes API, and to provide a secure way for pods to access other resources, such as secrets and ConfigMaps.

Use the following command to create the a service account that will be used for monitoring, later in this tutorial:
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/03.network-segmentation/03.sa.yaml
```

Cluster binding
===
Use the following command to create the required cluster-binding that will be used for monitoring, later in this tutorial:
```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/03.network-segmentation/04.cluster-binding.yaml
```

> **Note:** If you like to learn more about segmentation click <a href="https://www.tigera.io/blog/how-to-integrate-kubernetes-rbac-and-calico-to-achieve-shift-left-security/" target="_blank">here</a> to learn about Calico shift-left capabilities.

Next step
===
In the next tutorial we are going to briefly look at [Best practices](../04.best-practices-for-securing-a-Kubernetes-environment/readme.md).

# AKS dataplanes
Use the aks_test file to create an AKS cluster equipped with Azure-CNI.

Use aks_byocni file to create an AKS cluster equipped with Calico.

Keep in mind that when `Azure-CNI` is installed in your cluster it will be used for networking and can not be changed unless you destroy and recreate your cluster from scratch with a different CNI option.
 
## Quick note!
Heya!
If you are interested in more benchmarking between dataplanes I would highly suggest that you read [this blog]
(https://kinvolk.io/blog/2020/12/egress-filtering-benchmark-part-2-calico-and-cilium/).

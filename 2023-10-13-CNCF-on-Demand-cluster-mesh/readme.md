# Requirements
An AWS account and internet
Terraform
jq

# Building the infrastructure
Grab a copy of demo-cluster project
```
git clone git@github.com:frozenprocess/demo-cluster.git
cd demo-cluster
```

Copy the multicluster terraform file to your current directory by using the folllwing command:
```
cp examples/main.tf-awsmulticluster main.tf
```

Change the default Calico encapsulation to `IPIP` 
```
sed -i  "s/VXLAN/IPIPCrossSubnet/" files/calico-install.sh
```

Use the following command to build the infrastructure:
```
terraform init
terraform apply -auto-approve
```
After the deployment is finished we need to transfer the config file from each cluster to our local computer.

Use the following command to copy the `cluster-a` config file:
```
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i cluster-a.pem ubuntu@$(terraform output -json cluster-a | jq -r ".instance_1_public_ip"):~/.kube/config ca-config
```

Our cluster config file requires a bit of modifications, we need to change the IP address to the public one that is associated with the control plane and change the name of our cluster to `cluster-a`.
```
sed -i "s/127.0.0.1/$(terraform output -json cluster-a | jq -r ".instance_1_public_ip")/" ca-config
sed -i "s/default/cluster-a/" ca-config
```

Use the following commands to store the IP address for each participating node in our cluster in an environment variable:
```
export CLUSTER_A_CONTROL_IP=$(terraform output -json cluster-a | jq -r ".instance_1_private_ip")
export CLUSTER_A_WORKER_IP=$(terraform output -json cluster-a | jq -r ".workers_ip.private_ip[0]")
```

Let's repeat the same commands but this time for `cluster-b`.
```
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i cluster-b.pem ubuntu@$(terraform output -json cluster-b | jq -r ".instance_1_public_ip"):~/.kube/config cb-config
sed -i "s/127.0.0.1/$(terraform output -json cluster-b | jq -r ".instance_1_public_ip")/" cb-config
sed -i "s/default/cluster-b/" cb-config
export CLUSTER_B_CONTROL_IP=$(terraform output -json cluster-b | jq -r ".instance_1_private_ip")
export CLUSTER_B_WORKER_IP=$(terraform output -json cluster-b | jq -r ".workers_ip.private_ip[0]")
```
Finally use the following command to export the config file for kubectl:
```
export KUBECONFIG=$PWD/ca-config:$PWD/cb-config
```

At any time you can use the following commands to get more infomration about your deployment and cluster configurations:
```
terraform output
kubectl config view
```

# Configuring BGP
Now that we have a working cluster let's configure our BGP instances.

> **Note:** We are going to use `---context` to switch between our clusters.


Use the following command to assign an `asNumber` to `cluster-a`:
```
kubectl --context cluster-a create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  asNumber: 65001
  listenPort: 179
  nodeToNodeMeshEnabled: false
  serviceClusterIPs:
    - cidr: 10.43.0.0/16
EOF
```

Use the following command to assign an `asNumber` to `cluster-b`:
```
kubectl --context cluster-b create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  asNumber: 65002
  listenPort: 179
  nodeToNodeMeshEnabled: false
  serviceClusterIPs:
    - cidr: 10.53.0.0/16
EOF
```
Now that the BGP is configured we have to create peers.
## BGP Peers

Use the following command to configure `cluster-b`` nodes as peer for `cluster-a`` nodes.
```
kubectl create --context cluster-a -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2clusterb-control
spec:
  peerIP: $CLUSTER_B_CONTROL_IP
  asNumber: 65002
---
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2clusterb-worker
spec:
  peerIP: $CLUSTER_B_WORKER_IP
  asNumber: 65002
EOF
```

Do the same for `cluster-b`.
```
kubectl create --context cluster-b -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2clustera-control
spec:
  peerIP: $CLUSTER_A_CONTROL_IP
  asNumber: 65001
---
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2clustera-worker
spec:
  peerIP: $CLUSTER_A_WORKER_IP
  asNumber: 65001
EOF
```

Use the following command to check the BGP status:
```
kubectl --context cluster-a exec -n calico-system ds/calico-node -c calico-node -- birdcl show protocols
```

You should see a result similar to the follwoing:
```
BIRD v0.3.3+birdv1.6.8 ready.
name     proto    table    state  since       info
static1  Static   master   up     16:37:48    
kernel1  Kernel   master   up     16:37:48    
device1  Device   master   up     16:37:48    
direct1  Direct   master   up     16:37:48    
Node_172_16_2_116_port_179 BGP      master   up     16:42:53    Established   
Node_34_172_155_248 BGP      master   start  17:22:23    Connect 
```

# Its AWS time!

Use the following commands to get the VPC_ID for each deployment:
```
CLUSTER_A_VPC=$(aws ec2 describe-vpcs --region us-west-2 --filters Name=tag:Environment,Values="Calico Demo" --query "Vpcs[*].VpcId" --output text)
CLUSTER_B_VPC=$(aws ec2 describe-vpcs --region us-east-2 --filters Name=tag:Environment,Values="Calico Demo" --query "Vpcs[*].VpcId" --output text)
```

## VPC peering

> **Note**: Creating a VPC peering connection doesn't mean that resources automagically can talk to eachother.

Now that we have the IDs for each VPC let's make a VPC peering request.
```
aws ec2 create-vpc-peering-connection --vpc-id $CLUSTER_A_VPC --peer-vpc-id $CLUSTER_B_VPC --peer-region us-east-2 2>&1 > /dev/null
```

Use the following commands to store the route table ID for each cluster and store them in environment variables:
```
ROUTE_ID_CA=$(aws ec2 describe-route-tables --filters Name=tag:Environment,Values="Calico Demo" --query "RouteTables[*].RouteTableId" --output text)
ROUTE_ID_CB=$(aws ec2 describe-route-tables --region us-east-2 --filters Name=tag:Environment,Values="Calico Demo" --query "RouteTables[*].RouteTableId" --output text)
```

Use the following command to get the VPC peering ID:
```
PEER_ID=$(aws ec2 describe-vpc-peering-connections --region us-west-2 --query "VpcPeeringConnections[0].VpcPeeringConnectionId" --output text)
```

Now let's use the peer ID to approve our peering request:
```
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEER_ID  --region us-east-2 2>&1
```

## Routing tables

Everything is set for our peering, now let's go ahead and add our routes for each vpc:
```
aws ec2 create-route --region us-west-2 --route-table-id $ROUTE_ID_CA --destination-cidr-block "172.17.0.0/16" --vpc-peering-connection-id $PEER_ID
aws ec2 create-route --region us-east-2 --route-table-id $ROUTE_ID_CB --destination-cidr-block "172.16.0.0/16" --vpc-peering-connection-id $PEER_ID
```

## Security groups

If you recall in our BGP configuration section we set Calico to use port `179` for BGP, now we need to open this port in both VPCs so clusters can actually establish a connection on it.

Use the following commands to get the Security group ID associated with our deployments:
```
SG_CA=$(aws ec2 describe-security-groups --region us-west-2 --filters Name=tag:Environment,Values="Calico Demo" --query "SecurityGroups[*].{ID:GroupId}[0].ID" --output text)
SG_CB=$(aws ec2 describe-security-groups --region us-east-2 --filters Name=tag:Environment,Values="Calico Demo" --query "SecurityGroups[*].{ID:GroupId}[0].ID" --output text)
```

Use the following commands to permit BGP in our deployment security groups:
```
aws ec2 authorize-security-group-ingress --region us-west-2 --group-id $SG_CA --protocol tcp --port 179 --cidr 172.17.0.0/16

aws ec2 authorize-security-group-ingress --region us-east-2 --group-id $SG_CB --protocol tcp --port 179 --cidr 172.16.0.0/16
```

## BGP check

Use the following command and check the BGP status:
```
kubectl --context cluster-a exec -n calico-system ds/calico-node -c calico-node -- birdcl show protocols
```

You should see a result similar to the following:
```
BIRD v0.3.3+birdv1.6.8 ready.
name     proto    table    state  since       info
static1  Static   master   up     04:40:01    
kernel1  Kernel   master   up     04:40:01    
device1  Device   master   up     04:40:01    
direct1  Direct   master   up     04:40:01    
Global_172_17_1_237 BGP      master   up     06:10:31    Established   
Global_172_17_2_153 BGP      master   up     06:10:32    Established 
```

## Cluster mesh

Now that we have BGP configured let's try resolve a remote destination.

Use the following command to create a workload and service on `cluster-a`:
```
kubectl create --context cluster-a deployment nginx --image=nginx
kubectl create --context cluster-a service nodeport nginx --tcp 80:80
```

Use the following command to get the IP address of our workload:
```
kubectl --context cluster-a get pods -o wide
```
You should see a result similar to the following
```
NAME                     READY   STATUS    RESTARTS     AGE   IP             NODE              NOMINATED NODE   READINESS GATES
nginx-77b4fdf86c-lghrk   1/1     Running   0          11h   10.42.186.67   ip-172-16-2-107   <none>           <none>
```

Now use the `netshoot` image to run a pod in cluster-b and test the connectivity to our workload:
```
kubectl --context cluster-b run tmp-shell --rm -i --tty --image nicolaka/netshoot -- ping -c4 10.42.186.67
```

> **Note**: We are using `ipipMode` to encapsualte traffic between clusters. This allows us to establish communication between resources in each cluster even if the underlying networking infrastrcure doesn't know about the remote destination.

Use the following command to inform `cluster-a` about the IPpools that are available in `cluster-b`:
```
kubectl --context cluster-a create -f -<<EOF
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: cluster-b-svc-cidr
spec:
  cidr: 10.53.0.0/16
  ipipMode: CrossSubnet
  natOutgoing: false
  disabled: true
---
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: cluster-b-pod-cidr
spec:
  cidr: 10.52.0.0/16
  ipipMode: CrossSubnet
  natOutgoing: false
  disabled: true
EOF
```

Use the following command to inform `cluster-b` about the IPpools that are available in `cluster-a`:
```
kubectl --context cluster-b create -f -<<EOF
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: cluster-a-svc-cidr
spec:
  cidr: 10.43.0.0/16
  ipipMode: CrossSubnet
  natOutgoing: false
  disabled: true
---
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: cluster-a-pod-cidr
spec:
  cidr: 10.42.0.0/16
  ipipMode: CrossSubnet
  natOutgoing: false
  disabled: true
EOF
```

## AWS time again!

use the following commands to allow IPIP traffic between VPCs:
```
aws ec2 authorize-security-group-ingress --region us-west-2 --group-id $SG_CA --protocol 4 --port 0 --cidr 172.17.0.0/16
aws ec2 authorize-security-group-ingress --region us-east-2 --group-id $SG_CB --protocol 4 --port 0 --cidr 172.16.0.0/16
```

Use the following command to inform Felix (the brain of Calico) about the remote nodes:
```
kubectl --context cluster-a patch felixconfiguration default --type='merge' -p '{"spec":{"externalNodesList":["172.17.0.0/16"]}}'
kubectl --context cluster-b patch felixconfiguration default --type='merge' -p '{"spec":{"externalNodesList":["172.16.0.0/16"]}}'
```

> **Note**: This time you should be able to curl the IP address of our workload.

Now use the `netshoot` image to run a pod in cluster-b and test the connectivity to our workload:
```
kubectl --context cluster-b run tmp-shell --rm -i --tty --image nicolaka/netshoot -- ping -c4 10.42.186.67
```

At this point you can view the Pod but you can't access its service:
```
kubectl patch service nginx -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```
Now use the netshoot pod to curl the service IP address:

## BGP Debugging commands

```
kubectl --context cluster-b exec -n calico-system ds/calico-node -c calico-node -- birdcl show protocols
kubectl --context cluster-b exec -n calico-system ds/calico-node -c calico-node -- birdcl show route
```



# Clean up

```
PEER_ID=$(aws ec2 describe-vpc-peering-connections --region us-west-2 --query "VpcPeeringConnections[*].VpcPeeringConnectionId" --output text)
aws ec2 delete-vpc-peering-connection --region us-west-2 --vpc-peering-connection-id $PEER_ID
aws ec2 delete-vpc-peering-connection --region us-east-2 --vpc-peering-connection-id $PEER_ID
```

```
terraform destroy -auto-approve
```


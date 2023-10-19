# Building the infrastructure

```
git clone git@github.com:frozenprocess/demo-cluster.git
cd demo-cluster
cp examples/main.tf-awsmulticluster main.tf
```

```
sed -i  "s/VXLAN/IPIPCrossSubnet/" files/calico-install.sh
```

```
terraform init
terraform apply -auto-approve
```


```
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i cluster-a.pem ubuntu@$(terraform output -json cluster-a | jq -r ".instance_1_public_ip"):~/.kube/config ca-config
```

```
sed -i "s/127.0.0.1/$(terraform output -json cluster-a | jq -r ".instance_1_public_ip")/" ca-config
sed -i "s/default/cluster-a/" ca-config
```

```
export CLUSTER_A_CONTROL_IP=$(terraform output -json cluster-a | jq -r ".instance_1_private_ip")
export CLUSTER_A_WORKER_IP=$(terraform output -json cluster-a | jq -r ".workers_ip.private_ip[0]")
```

```
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i cluster-b.pem ubuntu@$(terraform output -json cluster-b | jq -r ".instance_1_public_ip"):~/.kube/config cb-config
sed -i "s/127.0.0.1/$(terraform output -json cluster-b | jq -r ".instance_1_public_ip")/" cb-config
sed -i "s/default/cluster-b/" cb-config
export KUBECONFIG=$PWD/ca-config:$PWD/cb-config
export CLUSTER_B_CONTROL_IP=$(terraform output -json cluster-b | jq -r ".instance_1_private_ip")
export CLUSTER_B_WORKER_IP=$(terraform output -json cluster-b | jq -r ".workers_ip.private_ip[0]")
```
terraform output

kubectl config view

## Configuring BGP

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

```
AWS_CP_NAME=$(kubectl --context cluster-a get nodes --selector='node-role.kubernetes.io/master' -o jsonpath='{.items[0].metadata.name}')
kubectl --context cluster-a annotate node $AWS_CP_NAME projectcalico.org/RouteReflectorClusterID=244.0.0.2
kubectl --context cluster-a label node $AWS_CP_NAME route-reflector=true
```

```
AWS_CP_NAME=$(kubectl get --context cluster-b  nodes --selector='node-role.kubernetes.io/master' -o jsonpath='{.items[0].metadata.name}')
kubectl --context cluster-b annotate node $AWS_CP_NAME projectcalico.org/RouteReflectorClusterID=244.0.0.2
kubectl --context cluster-b label node $AWS_CP_NAME route-reflector=true
```

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

```
kubectl create --context cluster-a -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2clusterb-reflector
spec:
  nodeSelector: all()
  peerSelector: route-reflector == 'true'
EOF
```

```
kubectl create --context cluster-b -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2clustera-reflector
spec:
  nodeSelector: all()
  peerSelector: route-reflector == 'true'
EOF
```

```
kubectl --context cluster-a exec -n calico-system ds/calico-node -c calico-node -- birdcl show protocols
```

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


```
CLUSTER_A_VPC=$(aws ec2 describe-vpcs --region us-west-2 --filters Name=tag:Environment,Values="Calico Demo" --query "Vpcs[*].VpcId" --output text)
CLUSTER_B_VPC=$(aws ec2 describe-vpcs --region us-east-2 --filters Name=tag:Environment,Values="Calico Demo" --query "Vpcs[*].VpcId" --output text)
aws ec2 create-vpc-peering-connection --vpc-id $CLUSTER_A_VPC --peer-vpc-id $CLUSTER_B_VPC --peer-region us-east-2 2>&1 > /dev/null
``````


```
ROUTE_ID_CA=$(aws ec2 describe-route-tables --filters Name=tag:Environment,Values="Calico Demo" --query "RouteTables[*].RouteTableId" --output text)
ROUTE_ID_CB=$(aws ec2 describe-route-tables --region us-east-2 --filters Name=tag:Environment,Values="Calico Demo" --query "RouteTables[*].RouteTableId" --output text)


PEER_ID=$(aws ec2 describe-vpc-peering-connections --region us-west-2 --query "VpcPeeringConnections[0].VpcPeeringConnectionId" --output text)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEER_ID  --region us-east-2 2>&1 > /dev/null

aws ec2 create-route --region us-west-2 --route-table-id $ROUTE_ID_CA --destination-cidr-block "172.17.0.0/16" --vpc-peering-connection-id $PEER_ID
aws ec2 create-route --region us-east-2 --route-table-id $ROUTE_ID_CB --destination-cidr-block "172.16.0.0/16" --vpc-peering-connection-id $PEER_ID
```

```
kubectl --context cluster-a exec -n calico-system ds/calico-node -c calico-node -- birdcl show protocols
```


```
SG_CA=$(aws ec2 describe-security-groups --region us-west-2 --filters Name=tag:Environment,Values="Calico Demo" --query "SecurityGroups[*].{ID:GroupId}[0].ID" --output text)
aws ec2 authorize-security-group-ingress --region us-west-2 --group-id $SG_CA --protocol tcp --port 179 --cidr 172.17.0.0/16

SG_CB=$(aws ec2 describe-security-groups --region us-east-2 --filters Name=tag:Environment,Values="Calico Demo" --query "SecurityGroups[*].{ID:GroupId}[0].ID" --output text)
aws ec2 authorize-security-group-ingress --region us-east-2 --group-id $SG_CB --protocol tcp --port 179 --cidr 172.16.0.0/16
```


```
kubectl --context cluster-a exec -n calico-system ds/calico-node -c calico-node -- birdcl show protocols
```



```
kubectl create --context cluster-a deployment nginx --image=nginx
kubectl create --context cluster-a service nodeport nginx --tcp 80:80
```

```
kubectl --context cluster-a get pods -o wide
kubectl --context cluster-b run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
```

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

```
kubectl create --context cluster-a deployment nginx --image=nginx
kubectl create --context cluster-a service nodeport nginx --tcp 80:80
kubectl patch service nginx -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

```
kubectl get pods -o wide
kubectl --context cluster-b run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
```

```
aws ec2 authorize-security-group-ingress --region us-west-2 --group-id $SG_CA --protocol 4 --port 0 --cidr 172.17.0.0/16
aws ec2 authorize-security-group-ingress --region us-east-2 --group-id $SG_CB --protocol 4 --port 0 --cidr 172.16.0.0/16
```

```
kubectl --context cluster-a patch felixconfiguration default --type='merge' -p '{"spec":{"externalNodesList":["172.17.0.0/16"]}}'
kubectl --context cluster-b patch felixconfiguration default --type='merge' -p '{"spec":{"externalNodesList":["172.16.0.0/16"]}}'
```

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


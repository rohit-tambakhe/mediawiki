# Lab setup for Mediawiki using JUJU:

1.	Create the VPC to host the Kubernetes Cluster:

```sh
$ aws ec2 create-vpc --cidr-block 10.250.0.0/16
{
    "Vpc": {
        "CidrBlock": "10.250.0.0/16",
        "DhcpOptionsId": "dopt-0c324b41063602b80",
        "State": "pending",
        "VpcId": "vpc-068f4988d0c46cedf",
        "OwnerId": "808641108845",
        "InstanceTenancy": "default",
        "Ipv6CidrBlockAssociationSet": [],
        "CidrBlockAssociationSet": [
            {
                "AssociationId": "vpc-cidr-assoc-0cf9e2898dc0f6d4f",
                "CidrBlock": "10.250.0.0/16",
                "CidrBlockState": {
                    "State": "associated"
                }
            }
        ],
        "IsDefault": false
    }
}
```

2.	Create an .sh file to store the important environment variables:

```sh
VPC_ID=vpc-068f4988d0c46cedf
echo "VPC_ID=$VPC_ID" >> aws_vars.sh
source aws_vars.sh
```

3.	Create the required subnets for High Availability and save the subnet IDs in aws_vars.sh file we created before:

```sh
$ echo "SUBNET1_ID=$SUBNET1_ID" >> aws_vars.sh
$ echo "SUBNET2_ID=$SUBNET2_ID" >> aws_vars.sh
$ echo "SUBNET3_ID=$SUBNET3_ID" >> aws_vars.sh
$ echo "SUBNET4_ID=$SUBNET4_ID" >> aws_vars.sh
```

If you cat aws_vars.sh file this should be the output:
```sh
$ cat aws_vars.sh
VPC_ID=vpc-068f4988d0c46cedf
SUBNET1_ID=subnet-0f62a10fbeab970b3
SUBNET2_ID=subnet-025b969f518ca0512
SUBNET3_ID=subnet-022861daf8d1e8e29
SUBNET4_ID=subnet-0455b31248ef4f635
```

### NOTE: Before performing operations using kubectl it is recommended to run `source aws_vars.sh` to source the correct environment variables.

4.	Create and Internet Gateway and attach to VPC to make it publicly available.

```sh
$ aws ec2 create-internet-gateway
{
    "InternetGateway": {
        "Attachments": [],
        "InternetGatewayId": "igw-08f44854f67b15e39",
        "OwnerId": "808641108845",
        "Tags": []
    }
}
```

```sh
$ aws ec2 describe-internet-gateways
{
    "InternetGateways": [
        {
            "Attachments": [
                {
                    "State": "available",
                    "VpcId": "vpc-068f4988d0c46cedf"
                }
            ],
            "InternetGatewayId": "igw-08f44854f67b15e39",
            "OwnerId": "808641108845",
            "Tags": []
        }
    ]
}

```

## Attach the internet gateway to VPC:
$ aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW
5.	Get the route table details:

```sh
$ aws ec2 describe-route-tables --filter "Name=vpc-id,Values=$VPC_ID"

{
    "RouteTables": [
        {
            "Associations": [
                {
                    "Main": false,
                    "RouteTableAssociationId": "rtbassoc-0cf74a32459898793",
                    "RouteTableId": "rtb-0157f08f342bdcbff",
                    "SubnetId": "subnet-025b969f518ca0512",
                    "AssociationState": {
                        "State": "associated"
                    }
                },
                {
                    "Main": true,
                    "RouteTableAssociationId": "rtbassoc-0dcd22423a9a453e5",
                    "RouteTableId": "rtb-0157f08f342bdcbff",
                    "AssociationState": {
                        "State": "associated"
                    }
                },
                {
                    "Main": false,
                    "RouteTableAssociationId": "rtbassoc-0ec777e00bbf16ad7",
                    "RouteTableId": "rtb-0157f08f342bdcbff",
                    "SubnetId": "subnet-0f62a10fbeab970b3",
                    "AssociationState": {
                        "State": "associated"
                    }
                },
                {
                    "Main": false,
                    "RouteTableAssociationId": "rtbassoc-0dce727ef7abba3c2",
                    "RouteTableId": "rtb-0157f08f342bdcbff",
                    "SubnetId": "subnet-0455b31248ef4f635",
                    "AssociationState": {
                        "State": "associated"
                    }
                },
                {
                    "Main": false,
                    "RouteTableAssociationId": "rtbassoc-002d0befb8788b137",
                    "RouteTableId": "rtb-0157f08f342bdcbff",
                    "SubnetId": "subnet-022861daf8d1e8e29",
                    "AssociationState": {
                        "State": "associated"
                    }
                }
            ],

}
```

6.	Create the default route to IGW:

```sh
$ aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
{
    "Return": true
}
```

And associate the route tables with all the subnets to make them public:

```sh
$ aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
{
    "Return": true
}

$ aws ec2 associate-route-table --subnet-id $SUBNET1_ID --route-table-id $RT
{
    "AssociationId": "rtbassoc-0ec777e00bbf16ad7",
    "AssociationState": {
        "State": "associated"
    }
}

$ aws ec2 associate-route-table --subnet-id $SUBNET2_ID --route-table-id $RT
{
    "AssociationId": "rtbassoc-0cf74a32459898793",
    "AssociationState": {
        "State": "associated"
    }
}

$ aws ec2 associate-route-table --subnet-id $SUBNET3_ID --route-table-id $RT
{
    "AssociationId": "rtbassoc-002d0befb8788b137",
    "AssociationState": {
        "State": "associated"
    }
}

$ aws ec2 associate-route-table --subnet-id $SUBNET4_ID --route-table-id $RT
{
    "AssociationId": "rtbassoc-0dce727ef7abba3c2",
    "AssociationState": {
        "State": "associated"
    }
}
```

7.	Setup VPC subnet attributes to map public IP on launch:

```sh
$ aws ec2 modify-subnet-attribute --subnet-id $SUBNET1_ID --map-public-ip-on-launch
$ aws ec2 modify-subnet-attribute --subnet-id $SUBNET2_ID --map-public-ip-on-launch
$ aws ec2 modify-subnet-attribute --subnet-id $SUBNET3_ID --map-public-ip-on-launch
$ aws ec2 modify-subnet-attribute --subnet-id $SUBNET4_ID --map-public-ip-on-launch
```

8.	Install Juju automation tool for better administration of Kubernetes Cluster in AWS.
```sh
$ snap install --classic juju
```
9.	Bootstrap the Kubernetes Controller to be used with Juju.
I have deployed it in subnet 1.
```sh
$ juju bootstrap aws kube-controller \
> --config vpc-id=$VPC_ID --config vpc-id-force=true --to "subnet=$SUBNET1_ID"

WARNING! The specified vpc-id does not satisfy the minimum Juju requirements,
but will be used anyway because vpc-id-force=true is also specified.

Using VPC "vpc-068f4988d0c46cedf" in region "us-east-1"
Creating Juju controller "my-controller" on aws/us-east-1
Looking for packaged Juju agent version 2.8.1 for amd64
Launching controller instance(s) on aws/us-east-1...
 - i-08f16c893db947c92 (arch=amd64 mem=4G cores=2) us-east-1a"
Installing Juju agent on bootstrap instance
Fetching Juju Dashboard 0.2.0
Waiting for address
Attempting to connect to 10.250.0.76:22
Attempting to connect to 54.234.251.130:22
Connected to 54.234.251.130
Running machine configuration script...
Bootstrap agent now started
Contacting Juju controller at 54.234.251.130 to verify accessibility...

Bootstrap complete, controller "my-controller" is now available
Controller machines are in the "controller" model
Initial model "default" added

```

## Juju works with different models denoting different Deployments in Kubernetes:
Here I am creating a new model for ThoughtWorks Deployment named kubernetes and adding the public space so that you will be able to access it from the management machine:

```sh
$ juju add-model kubernetes aws --config vpc-id=$VPC_ID
Added 'k8s' model on aws/us-east-1 with credential 'default' for user 'admin'

$ juju add-space public 10.250.0.0/24 10.250.1.0/24 10.250.2.0/24 10.250.3.0/24 -m k8s

added space "public" with subnets 10.250.0.0/24, 10.250.1.0/24, 10.250.2.0/24, 10.250.3.0/24
```
## If you do juju models you will see the model deployed for this lab:

```sh

$ juju models
Controller: kube-controller

Model        Cloud/Region   Type  Status     Machines  Cores  Units  Access  Last connection
controller   aws/us-east-1  ec2   available         1      2  -      admin   just now
default      aws/us-east-1  ec2   available         0      -  -      admin   24 minutes ago
kubernetes*  aws/us-east-1  ec2   available         8     16  13     admin   6 minutes ago

```
10.	Now deploy Highly Available Media Wiki with Kubernetes Cluster using Juju
```sh
$ juju deploy cs:tambakherohit/mediawiki

$ juju deploy cs:tambakherohit/Kubernetes-master

$ juju deploy cs:tambakherohit/Kubernetes-worker
```

## If everything works well considering our network deployment and control plane deployment you will see the following output:

```sh
$ watch -c juju status --color

Every 2.0s: juju status --color                                                                                                          ubuntu-s-4vcpu-8gb-fra1-01: Sun Sep  6 15:49:29 2020

Model       Controller       Cloud/Region   Version  SLA          Timestamp
kubernetes  kube-controller  aws/us-east-1  2.8.1    unsupported  15:49:30Z

App                Version  Status   Scale  Charm              Store       Rev  OS      Notes
containerd         1.3.3    active       2  containerd         jujucharms   88  ubuntu
easyrsa            3.0.1    active       1  easyrsa            jujucharms  325  ubuntu
etcd               3.3.19   active       1  etcd               jujucharms  531  ubuntu
flannel            0.11.0   active       2  flannel            jujucharms  501  ubuntu
haproxy                     unknown      1  haproxy            jujucharms   13  ubuntu
kubernetes-master  1.18.8   active       1  kubernetes-master  jujucharms  865  ubuntu  exposed
kubernetes-worker  1.18.8   active       1  kubernetes-worker  jujucharms  692  ubuntu  exposed
mediawiki                   unknown      1  mediawiki          jujucharms    3  ubuntu
memcached                   unknown      1  memcached          jujucharms   11  ubuntu
mysql                       unknown      1  mysql              jujucharms   29  ubuntu
mysql-slave                 unknown      1  mysql              jujucharms   29  ubuntu

Unit                  Workload  Agent  Machine  Public address  Ports           Message
easyrsa/0*            active    idle   5/lxd/0  252.1.198.43                    Certificate Authority connected.
etcd/0*               active    idle   5        3.88.176.225    2379/tcp        Healthy with 1 known peer
haproxy/0*            unknown   idle   0        54.210.131.226  80/tcp
kubernetes-master/0*  active    idle   5        3.88.176.225    6443/tcp        Kubernetes master running.
  containerd/1        active    idle            3.88.176.225                    Container runtime available
  flannel/1           active    idle            3.88.176.225                    Flannel subnet 10.1.63.1/24
kubernetes-worker/0*  active    idle   6        3.88.18.208     80/tcp,443/tcp  Kubernetes worker running.
  containerd/0*       active    idle            3.88.18.208                     Container runtime available
  flannel/0*          active    idle            3.88.18.208                     Flannel subnet 10.1.98.1/24
mediawiki/0*          unknown   idle   1        34.230.4.252
memcached/0*          unknown   idle   2        54.92.181.16    11211/tcp
mysql-slave/0*        unknown   idle   4        54.174.164.178
mysql/0*              unknown   idle   3        3.237.100.138

Machine  State    DNS             Inst id              Series  AZ          Message
0        started  54.210.131.226  i-04c74f613269a953d  trusty  us-east-1a  running
1        started  34.230.4.252    i-00b5008f9f2e168cb  trusty  us-east-1b  running
2        started  54.92.181.16    i-0a13769db89bd78a9  trusty  us-east-1c  running
3        started  3.237.100.138   i-09606999840b52348  trusty  us-east-1d  running
4        started  54.174.164.178  i-08b7a0175e15444a2  trusty  us-east-1a  running
5        started  3.88.176.225    i-0efa7d239c8748f38  bionic  us-east-1b  running
5/lxd/0  started  252.1.198.43    juju-3c1c0e-5-lxd-0  bionic  us-east-1b  Container started
6        started  3.88.18.208     i-043ff5274e436e538  bionic  us-east-1c  running

```

## For scaling out

You can add and remove more mediawiki instances to horizontally scale based on your convenience:
```sh
$ juju add-unit wiki
```
You can use both kubectl and juju to perform operations on this cluster.
Juju makes it rather simple to add relationships between different pods and scale the cluster on fly.


## JUJU commands reference:
https://juju.is/docs/commands
 
## You can run kubectl commands from the control machine to perform various Kuberneted Operations

```sh
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.152.183.1   <none>        443/TCP   21h

$ kubectl get deployments --all-namespaces
NAMESPACE                         NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
ingress-nginx-kubernetes-worker   default-http-backend-kubernetes-worker   1/1     1            1           22h
kube-system                       coredns                                  1/1     1            1           22h
kube-system                       kube-state-metrics                       1/1     1            1           22h
kube-system                       metrics-server-v0.3.6                    1/1     1            1           22h
kubernetes-dashboard              dashboard-metrics-scraper                1/1     1            1           22h
kubernetes-dashboard              kubernetes-dashboard                     1/1     1            1           22h
```

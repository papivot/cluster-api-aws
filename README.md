# Cluster API on AWS

### Working with **release-0.4** providing support for **v1alpha2**

The following CRDs are available as part of this release - 

```console
clusters.cluster.x-k8s.io
machinedeployments.cluster.x-k8s.io
machines.cluster.x-k8s.io
machinesets.cluster.x-k8s.io
```
```console
awsclusters.infrastructure.cluster.x-k8s.io
awsmachines.infrastructure.cluster.x-k8s.io
awsmachinetemplates.infrastructure.cluster.x-k8s.io
```
```console
kubeadmconfigs.bootstrap.cluster.x-k8s.io
kubeadmconfigtemplates.bootstrap.cluster.x-k8s.io
```

Users get access to the following api resources -
```console
NAME                              SHORTNAMES   APIGROUP                          NAMESPACED   KIND
clusters                          cl           cluster.x-k8s.io                  true         Cluster
machinedeployments                md           cluster.x-k8s.io                  true         MachineDeployment
machines                          ma           cluster.x-k8s.io                  true         Machine
machinesets                       ms           cluster.x-k8s.io                  true         MachineSet
...
awsclusters                                    infrastructure.cluster.x-k8s.io   true         AWSCluster
awsmachines                                    infrastructure.cluster.x-k8s.io   true         AWSMachine
awsmachinetemplates                            infrastructure.cluster.x-k8s.io   true         AWSMachineTemplate
...
kubeadmconfigs                                 bootstrap.cluster.x-k8s.io        true         KubeadmConfig
kubeadmconfigtemplates                         bootstrap.cluster.x-k8s.io        true         KubeadmConfigTemplate
...
```

The controllers and apiservices are as follows - 

```console
cluster-api-controller 			- v1alpha2.cluster.x-k8s.io
cluster-api-aws-controller 		- v1alpha2.infrastructure.cluster.x-k8s.io
cluster-api-kubeadm-controller 	- v1alpha2.bootstrap.cluster.x-k8s.io
```

-----
### Setup 
This guide will use an upto date Ubuntu 18.04 server to run a management cluster. The  management cluster can be run on any K8s cluster. For simplicity sake, we will be using a simple K8s cluster deployed using `kind` . Once the management cluster is setup, it can be used to install and manage workload clusters.

#### Preparations

Make sure the following packages are installed on the Ubuntu server and are up to date. 

* JQ
* golang v1.13.4 at the time of writing this doc.
```shell
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt-get update
sudo apt install golang
```
* git
* curl
* kubectl
* kind
```shell
curl -Lo ./kind https://github.com/kubernetes-sigs/kind/releases/download/v0.7.0/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin
```
* aws cli (v1 or v2) installed and configured to access your AWS account.

#### Clone the stable release-0.4 branch that provides v1alpha2 support 
```shell
git clone https://github.com/kubernetes-sigs/cluster-api-provider-aws.git --branch release-0.4
cd cluster-api-provider-aws
#### make test # To validate everything works fine
make binaries
```

Copy the generated **clusterawsadm** binary to /usr/local/bin
```shell
sudo cp ./bin/clusterawsadm /usr/local/bin
```
Use the binary to create the AWS Cloudformation stack in your AWS account to set up the required IAM users/group/profiles.
```shell
clusterawsadm alpha bootstrap create-stack
```

This should take a while but create the similar IAM objects in the AWS account 

```console
Resource |Type |Status

AWS::IAM::Group |bootstrapper.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::User  |bootstrapper.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE

AWS::IAM::InstanceProfile |control-plane.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::InstanceProfile |controllers.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::InstanceProfile |nodes.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE

AWS::IAM::ManagedPolicy |arn:aws:iam::291518665672:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::ManagedPolicy |arn:aws:iam::291518665672:policy/controllers.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::ManagedPolicy |arn:aws:iam::291518665672:policy/nodes.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE

AWS::IAM::Role |control-plane.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::Role |controllers.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::Role |nodes.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
```

This completes the preliminary steps required to setup the environment. These steps are performed only once.

#### Setting up the Management Cluster 

Export the following environment variables. These variables ***can be modified as per the requirements***. 

```console
export AWS_CREDENTIALS=$(aws iam create-access-key --user-name bootstrapper.cluster-api-provider-aws.sigs.k8s.io)
export AWS_ACCESS_KEY_ID=$(echo $AWS_CREDENTIALS | jq .AccessKey.AccessKeyId -r)
export AWS_SECRET_ACCESS_KEY=$(echo $AWS_CREDENTIALS | jq .AccessKey.SecretAccessKey -r)
export SSH_KEY_NAME="awsbastion"
export CLUSTER_NAME="workload-cluster"
export AWS_REGION="us-east-2"
export CONTROL_PLANE_MACHINE_TYPE="t2.medium"
export NODE_MACHINE_TYPE="t2.medium"
```
Generate the sample yaml files that will be required to setup the management cluster (CRDs/controllers) and the files needed to setup the workload clusters. 

```shell
make generate-examples
```
All the yamls are generated in the `./example/_out` folder

Create a Management cluster using `kind`

```shell
kind create cluster --name=clusterapi
``` 
In a few minutes, the management cluster is instantiated in the Ubuntu server!!!

```console
Creating cluster "clusterapi" ...
 ✓ Ensuring node image (kindest/node:v1.17.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-clusterapi"
You can now use your cluster with:

kubectl cluster-info --context kind-clusterapi

Thanks for using kind! 😊
```
Make sure the cluster is accessible using the kubeconfig file 

```shell
kubectl get pods --all-namespaces
```
should return something similar (with all PODS in a `Running` state)

```console
NAMESPACE            NAME                                               READY   STATUS    RESTARTS   AGE
kube-system          coredns-6955765f44-92gc8                           1/1     Running   0          104s
kube-system          coredns-6955765f44-snqhv                           1/1     Running   0          104s
kube-system          etcd-clusterapi-control-plane                      1/1     Running   0          2m1s
kube-system          kindnet-4h8qx                                      1/1     Running   0          104s
kube-system          kube-apiserver-clusterapi-control-plane            1/1     Running   0          2m1s
kube-system          kube-controller-manager-clusterapi-control-plane   1/1     Running   0          2m
kube-system          kube-proxy-5mqvv                                   1/1     Running   0          104s
kube-system          kube-scheduler-clusterapi-control-plane            1/1     Running   0          2m
local-path-storage   local-path-provisioner-7745554f7f-ktlj5            1/1     Running   0          104s
```

Install the Cluster API CRDs and controllers to this newly created management cluster -

```shell
kubectl apply -f ./examples/_out/provider-components.yaml
```
This should create all the necessary CRDs and controllers in the cluster, with an output similar to this -
```console
namespace/cabpk-system created
namespace/capa-system created
namespace/capi-system created
customresourcedefinition.apiextensions.k8s.io/awsclusters.infrastructure.cluster.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/awsmachines.infrastructure.cluster.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/awsmachinetemplates.infrastructure.cluster.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/clusters.cluster.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/kubeadmconfigs.bootstrap.cluster.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/kubeadmconfigtemplates.bootstrap.cluster.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/machinedeployments.cluster.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/machines.cluster.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/machinesets.cluster.x-k8s.io created
...
secret/capa-manager-bootstrap-credentials created
service/cabpk-controller-manager-metrics-service created
service/capa-controller-manager-metrics-service created
deployment.apps/cabpk-controller-manager created
deployment.apps/capa-controller-manager created
deployment.apps/capi-controller-manager created
```
Make sure that the new controllers are in a running state -

```shell
kubectl get pods --all-namespaces
```

should return similar to this. The `capi-system`,`capa-system` and `cabpk-system` are the new namespaces with the controllers running in them - 

```console
NAMESPACE            NAME                                               READY   STATUS    RESTARTS   AGE
cabpk-system         cabpk-controller-manager-6dcc9b8b96-5lxm6          2/2     Running   0          2m36s
capa-system          capa-controller-manager-57ff9959ff-zvt4w           1/1     Running   0          2m36s
capi-system          capi-controller-manager-66d98dc68f-jx5c5           1/1     Running   0          2m36s
...
```

With this, the management cluster is now ready. You can now start deploying workload-clusters by modifying/tweaking the other yaml files in the `./example/_out` folder. 

### Deploying Clusters (workload clusters)

* Step 1 - Creating the Cluster object

```shell
kubectl apply -f ./examples/_out/cluster.yaml
```

```console
cluster.cluster.x-k8s.io/workload-cluster created
awscluster.infrastructure.cluster.x-k8s.io/workload-cluster created
```
Validate within the AWS console that a new VPC with associated subnets, NAT gateway, security groups and a bastion server has been created. This should take approx. 10 mins. 

* Step 2 - Creating the control plane machines. 3 nodes in this example. 

```shell
kubectl apply -f ./examples/_out/controlplane.yaml
```

```console
kubeadmconfig.bootstrap.cluster.x-k8s.io/workload-cluster-controlplane-0 created
kubeadmconfig.bootstrap.cluster.x-k8s.io/workload-cluster-controlplane-1 created
kubeadmconfig.bootstrap.cluster.x-k8s.io/workload-cluster-controlplane-2 created
machine.cluster.x-k8s.io/workload-cluster-controlplane-0 created
machine.cluster.x-k8s.io/workload-cluster-controlplane-1 created
machine.cluster.x-k8s.io/workload-cluster-controlplane-2 created
awsmachine.infrastructure.cluster.x-k8s.io/workload-cluster-controlplane-0 created
awsmachine.infrastructure.cluster.x-k8s.io/workload-cluster-controlplane-1 created
awsmachine.infrastructure.cluster.x-k8s.io/workload-cluster-controlplane-2 created
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTc4OTA2OTUyNSwtMTM0MzA2MTE2NiwxMD
c2NzE5NTksLTE2ODY4NTc0MTNdfQ==
-->
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

Users have access to the following api resources -
```console
clusters 				cl 		cluster.x-k8s.io 	true 	Cluster
machinedeployments 		md 		cluster.x-k8s.io 	true 	MachineDeployment
machines 				ma 		cluster.x-k8s.io 	true 	Machine
machinesets 			ms 		cluster.x-k8s.io 	true 	MachineSet

awsclusters 					infrastructure.cluster.x-k8s.io true AWSCluster
awsmachines 					infrastructure.cluster.x-k8s.io true AWSMachine
awsmachinetemplates 			infrastructure.cluster.x-k8s.io true AWSMachineTemplate

kubeadmconfigs 					bootstrap.cluster.x-k8s.io true KubeadmConfig
kubeadmconfigtemplates 			bootstrap.cluster.x-k8s.io true KubeadmConfigTemplate
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
make generate-examples
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

Generate the sample yaml files that will be required to setup the management cluster (CRDs/controllers) and the 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQwNDYxMzA3LDEwNzY3MTk1OSwtMTY4Nj
g1NzQxM119
-->
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
* golang v1.13.4 at the time of writing  this doc.
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
make test # To validate everything works fine
make binaries
make generate-examples
```

Copy the generated **clusterawsadm** binary to /usr/local/bin
```shell
cp ./bin/clusterawsadm /usr/local/bin
```
clusterawsadm alpha bootstrap create-stack
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTEwOTE2OTMxNiwtMTY4Njg1NzQxM119
-->
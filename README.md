# Cluster API on AWS <!-- omit in toc -->

### Working with **release-0.4** providing support for **v1alpha2** <!-- omit in toc -->

#### Table of Content <!-- omit in toc -->
- [Technical Details [to expand]](#technical-details-to-expand)
- [Setup](#setup)
  - [Preparations](#preparations)
  - [Clone the stable `release-0.4` branch that provides `v1alpha2` support](#clone-the-stable-release-04-branch-that-provides-v1alpha2-support)
  - [Create the Cloudformation stack](#create-the-cloudformation-stack)
- [Deploying the Management Cluster](#deploying-the-management-cluster)
- [Deploying the Workload Cluster(s)](#deploying-the-workload-clusters)
  - [Creating the Cluster object](#creating-the-cluster-object)
  - [Creating the Control Plane machines.](#creating-the-control-plane-machines)
  - [Generating the Kubeconfig file](#generating-the-kubeconfig-file)
  - [Install the CNI addons to the Workload Cluster.](#install-the-cni-addons-to-the-workload-cluster)
  - [Deploy the Worker Nodes leveraging the Cluster API.](#deploy-the-worker-nodes-leveraging-the-cluster-api)
- [Lifecycle management of Workload Cluster](#lifecycle-management-of-workload-cluster)
  - [Manually increasing # of Worker Nodes](#manually-increasing--of-worker-nodes)
  - [Manually decreasing # of Worker Nodes](#manually-decreasing--of-worker-nodes)
  - [~~Resurrection of a worker node~~](#sresurrection-of-a-worker-nodes)
  - [Manually increasing # of Control Plane Nodes](#manually-increasing--of-control-plane-nodes)
  - [Manually decreasing # of Control Plane Nodes](#manually-decreasing--of-control-plane-nodes)
  - [~~Resurrection of a control plane node~~](#sresurrection-of-a-control-plane-nodes)
  - [Accessing EC2 instances](#accessing-ec2-instances)
  - [Upgrading K8s version on Worker Nodes.](#upgrading-k8s-version-on-worker-nodes)
  - [Destroying/Deleting a Workload cluster](#destroyingdeleting-a-workload-cluster)
- [References](#references)

## Technical Details [to expand]

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
cluster-api-controller                  - v1alpha2.cluster.x-k8s.io
cluster-api-aws-controller              - v1alpha2.infrastructure.cluster.x-k8s.io
cluster-api-kubeadm-controller          - v1alpha2.bootstrap.cluster.x-k8s.io
```

-----
## Setup 
This guide will use an upto date Ubuntu 18.04 server to run a management cluster (the  management cluster can be run on any K8s cluster (pivoting)). For simplicity sake, we will be using a simple K8s cluster, deployed using `kind`. Once the management cluster is setup, it can be used to install and manage workload clusters.

### Preparations

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
* aws cli (v1 or v2) [installed](https://docs.aws.amazon.com/cli/latest/userguide/install-linux.html) and [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) to access your AWS account. Click on the links for additional directions. 

* Export the following environment variables. These variables ***should be modified as per user's requirements***. 

```console
export SSH_KEY_NAME="awsbastion"
export CLUSTER_NAME="workload-cluster"
export AWS_REGION="us-east-2"
export CONTROL_PLANE_MACHINE_TYPE="t2.medium"
export NODE_MACHINE_TYPE="t2.medium"
export KUBERNETES_VERSION=1.16.1
```

### Clone the stable `release-0.4` branch that provides `v1alpha2` support 
```shell
git clone https://github.com/kubernetes-sigs/cluster-api-provider-aws.git --branch release-0.4
cd cluster-api-provider-aws
make binaries
```
Copy the generated **clusterawsadm** binary to /usr/local/bin
```shell
sudo cp ./bin/clusterawsadm /usr/local/bin
```

### Create the Cloudformation stack 

Use the **clusterawsadm** binary to create an AWS Cloudformation stack in your AWS account, to set up the required IAM users/group/profiles.

```shell
clusterawsadm alpha bootstrap create-stack
```
This should take a while but creates the required IAM objects, similar to the output below, in the AWS account 

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

## Deploying the Management Cluster 

Export some additonal enviornment variables. These variables are required, along with the previously exported ones, to generate the sample yaml files. - 

```console
export AWS_CREDENTIALS=$(aws iam create-access-key --user-name bootstrapper.cluster-api-provider-aws.sigs.k8s.io)
export AWS_ACCESS_KEY_ID=$(echo $AWS_CREDENTIALS | jq .AccessKey.AccessKeyId -r)
export AWS_SECRET_ACCESS_KEY=$(echo $AWS_CREDENTIALS | jq .AccessKey.SecretAccessKey -r)
```

Generate the sample yaml files. These files are required to setup the management cluster (CRDs/controllers) and are also needed to setup the workload clusters. 

```shell
make generate-examples
```
All the yamls are generated in the `./example/_out` folder

Create a simple management cluster using `kind`

```shell
kind create cluster --name=clusterapi
``` 
Within a few minutes, the management cluster is instantiated in the Ubuntu server!!!

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
Make sure the cluster is accessible using the kubeconfig file.

```shell
kubectl get pods --all-namespaces
```
should return something similar (with all PODS in a `Running` state)

```console
NAMESPACE            NAME                                               READY   STATUS    RESTARTS   AGE
kube-system          coredns-6955765f44-92gc8                           1/1     Running   0          104s
kube-system          coredns-6955765f44-snqhv                           1/1     Running   0          104s
...
kube-system          kube-proxy-5mqvv                                   1/1     Running   0          104s
kube-system          kube-scheduler-clusterapi-control-plane            1/1     Running   0          2m
local-path-storage   local-path-provisioner-7745554f7f-ktlj5            1/1     Running   0          104s
```

Using the `provider-components.yaml`, install the Cluster API CRDs and controllers to this newly created management cluster -

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
...
secret/capa-manager-bootstrap-credentials created
service/cabpk-controller-manager-metrics-service created
service/capa-controller-manager-metrics-service created
deployment.apps/cabpk-controller-manager created
deployment.apps/capa-controller-manager created
deployment.apps/capi-controller-manager created
```
Make sure that all the new controllers are in a running state -

```shell
kubectl get pods --all-namespaces
```

should return similar to below. The `capi-system`,`capa-system` and `cabpk-system` are the new namespaces with the controllers running in them - 

```console
NAMESPACE            NAME                                               READY   STATUS    RESTARTS   AGE
cabpk-system         cabpk-controller-manager-6dcc9b8b96-5lxm6          2/2     Running   0          2m36s
capa-system          capa-controller-manager-57ff9959ff-zvt4w           1/1     Running   0          2m36s
capi-system          capi-controller-manager-66d98dc68f-jx5c5           1/1     Running   0          2m36s
...
```

With this, the management cluster is now ready. You can now start deploying workload-clusters by modifying/tweaking the other yaml files in the `./example/_out` folder. 

----
## Deploying the Workload Cluster(s) 

These steps would need to be performed for each Workload cluster that need to be deployed. Obviously, settings and variables need to be adjusted accordingly. 

### Creating the Cluster object

Note - If you want to use your custom VPC CIDR block, you can modify the `cluster.yaml` accordingly - 

```yaml
...
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
kind: AWSCluster
metadata:
  name: workload-cluster-aws-1
  namespace: default
spec:
# Add this section with custom cidr blocks and AZ info.
#--------------------------------
  networkSpec:
    subnets:
    - availabilityZone: us-east-2a
      cidrBlock: 10.20.0.0/24
      isPublic: False
    - availabilityZone: us-east-2a
      cidrBlock: 10.20.1.0/24
      isPublic: True
    vpc:
      cidrBlock: 10.20.0.0/16
#--------------------------------
  region: us-east-2
  sshKeyName: awsbastion
 ---
 ...
```

Now create the cluster object.

```shell
kubectl apply -f ./examples/_out/cluster.yaml
```
Should display an output similar to this - 
```console
cluster.cluster.x-k8s.io/workload-cluster created
awscluster.infrastructure.cluster.x-k8s.io/workload-cluster created
```
Validate within the AWS console that a new VPC with associated subnets, NAT gateway, security groups, LB, a bastion server and other objects has been created. This should take approx. 10 mins. 

### Creating the Control Plane machines. 
- 3 nodes in this example. 

```shell
kubectl apply -f ./examples/_out/controlplane.yaml
```
Should display an output similar to this -
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

This may take a while - 10-15 mins. 

### Generating the Kubeconfig file

Once this is complete, we need to grab the kubeconfig file that was associated with this workload server. The `kubeconfig` is stored as a secret within the management cluster, in the namespace associated with the workload cluster (default in this example) . This will allow us to connect to the workload cluster. 

```shell
kubectl get secrets -n default workload-cluster-kubeconfig -o json |jq -r .data.value|base64 -d > /tmp/workload-cluster.conf
```
The `workload-cluster.conf` file should be in a similar kubeconfig file format -

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5ekNDQWJPZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01ERXhPREF6TXpBd01Gb1hEVE13TURFeE5UQXpNelV3TUZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTFN5CjhLYy9lTVdCMzN2MXpaWVpkNjdubE8rd1oyUEN1Z0dIeXE0dTU3eDJ3YmJFcy9mRGpMVWZQOUJ1R1M5Ym4wY3QKeWthZVRrcGpyNk96dzRyR0E0dm1ZVUtLbFNkbXp...
    ...
contexts:
- context:
    cluster: workload-cluster
...
```
###  Install the CNI addons to the Workload Cluster.

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf apply -f ./examples/addons.yaml
```

This should deploy the required addon objects in the workload cluster -

```console
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
...
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```

Validate all the pods are successfully running. 

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf get pods --all-namespaces
```
Should display an output similar to this -

```console
NAMESPACE     NAME                                                               READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-564b6667d7-mbw5f                           1/1     Running   0          102s
kube-system   calico-node-8plqr                                                  1/1     Running   0          102s
kube-system   calico-node-gl8vf                                                  1/1     Running   0          102s
kube-system   calico-node-rjdvd                                                  1/1     Running   0          102s
kube-system   coredns-5644d7b6d9-jtdj4                                           1/1     Running   0          10m
kube-system   coredns-5644d7b6d9-r9mlv                                           1/1     Running   0          10m
kube-system   etcd-ip-10-0-0-18.us-east-2.compute.internal                       1/1     Running   0          9m43s
...
```
###  Deploy the Worker Nodes leveraging the Cluster API.

```shell
kubectl apply -f ./examples/_out/machinedeployment.yaml
```
Should display an output similar to this -

```console
kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io/workload-cluster-md-0 created
machinedeployment.cluster.x-k8s.io/workload-cluster-md-0 created
awsmachinetemplate.infrastructure.cluster.x-k8s.io/workload-cluster-md-0 created
```
In a short time, a machine deployment consisting of 2 machines (2 worker nodes) will be added to the workload cluster. 

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf get nodes
```
Should display an output similar to this -

```console
NAME                                       STATUS   ROLES    AGE     VERSION
ip-10-0-0-158.us-east-2.compute.internal   Ready    <none>   5m30s   v1.16.1
ip-10-0-0-19.us-east-2.compute.internal    Ready    <none>   5m17s   v1.16.1
ip-10-0-0-18.us-east-2.compute.internal    Ready    master   18m     v1.16.1
ip-10-0-0-200.us-east-2.compute.internal   Ready    master   19m     v1.16.1
ip-10-0-0-35.us-east-2.compute.internal    Ready    master   18m     v1.16.1
```
## Lifecycle management of Workload Cluster

### Manually increasing # of Worker Nodes
This is as simple as increasing the replica count of the machinedeployments object.  

```shell
kubectl get md -n default
kubectl edit md workload-cluster-md-0
```
Increase the `replica` from 2 to 3 and save the file (in this example)

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: cluster.x-k8s.io/v1alpha2
kind: MachineDeployment
metadata:
...
  name: workload-cluster-md-0
  namespace: default
  ownerReferences:
  - apiVersion: cluster.x-k8s.io/v1alpha2
...
spec:
  minReadySeconds: 0
  progressDeadlineSeconds: 600
  replicas: 3
``` 
With a few minutes, a new ec2 instance will be spawned and added to the workload cluster.  This can be validated in the EC2 console as well as the following command - 

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf get nodes
```

```console
NAME                                       STATUS   ROLES    AGE     VERSION
ip-10-0-0-158.us-east-2.compute.internal   Ready    <none>   15h     v1.16.1
ip-10-0-0-19.us-east-2.compute.internal    Ready    <none>   15h     v1.16.1
ip-10-0-0-244.us-east-2.compute.internal   Ready    <none>   8m36s   v1.16.1
ip-10-0-0-18.us-east-2.compute.internal    Ready    master   15h     v1.16.1
...
```
### Manually decreasing # of Worker Nodes

This is as simple as decreasing the replica count of the machinedeployments object.  

```shell
kubectl get md -n default
kubectl edit md workload-cluster-md-0
```
Increase the `replica` from 3 to 2 and save the file (in this example)

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: cluster.x-k8s.io/v1alpha2
kind: MachineDeployment
metadata:
...
  name: workload-cluster-md-0
  namespace: default
  ownerReferences:
  - apiVersion: cluster.x-k8s.io/v1alpha2
...
spec:
  minReadySeconds: 0
  progressDeadlineSeconds: 600
  replicas: 2
``` 
With a few minutes, a node will be drained, removed from the cluster and its corresponding  ec2 instance will be terminated.  This can be validated in the EC2 console as well as the following command - 

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf get nodes
```
```console
NAME                                       STATUS   ROLES    AGE     VERSION
ip-10-0-0-19.us-east-2.compute.internal    Ready    <none>   15h     v1.16.1
ip-10-0-0-244.us-east-2.compute.internal   Ready    <none>   15m4s   v1.16.1
ip-10-0-0-18.us-east-2.compute.internal    Ready    master   15h     v1.16.1
...
```
### ~~Resurrection of a worker node~~ 
- [this does now work as v1alpha2 does not support resurrection???]
  
For this demo, we will increase the worker node count back to 3 (in not already done). This can per performed by the steps in the section `Manually increasing the # of worker nodes` Once the cluster has reached a steady state, and all the nodes are available in the cluster, terminate one of the **worker nodes** from the AWS ec2 console. 

~~When the controller next runs the reconciliation loop and sees a missing machine from the machineset, a new ec2 instance should be spawned and added to the workload cluster.~~

Checking the machine status shows the following error 

`  errorMessage: 'Failure detected from referenced resource infrastructure.cluster.x-k8s.io/v1alpha2,
    Kind=AWSMachine with name "workload-cluster-md-0-8bfwx": EC2 instance state "terminated"
    is unexpected'`

Notice the state of the machine in the management cluster - 

```shell
kubectl get ma -n default
```

```console
NAME                                     PROVIDERID                    PHASE
...
workload-cluster-md-0-56d9ccc8d8-lsh4n   aws:////i-01a6e7b148de2eb18   running
workload-cluster-md-0-56d9ccc8d8-mw2cm   aws:////i-03fca5522a51dfe3a   failed
workload-cluster-md-0-56d9ccc8d8-txp5q   aws:////i-01596ec8c736d6f87   running
```
Deleting the failed machine forced the cluster-api to kickstart a new machine provisioning and restore the state of the machinedeployment back to the correct number of worker nodes. 

```shell
kubectl delete ma workload-cluster-md-0-56d9ccc8d8-mw2cm
kubectl get ma -n default
```
```console
NAME                                     PROVIDERID                    PHASE
...
workload-cluster-md-0-56d9ccc8d8-dvrrj                                 provisioning
workload-cluster-md-0-56d9ccc8d8-lsh4n   aws:////i-01a6e7b148de2eb18   running
workload-cluster-md-0-56d9ccc8d8-txp5q   aws:////i-01596ec8c736d6f87   running
```

### Manually increasing # of Control Plane Nodes
Since control plane nodes are not managed by a machinedeployment, a new control plane machine object has to be created, similar to the yaml provided below. 

Note: It was observed that reusing the machinenames caused problems. May be a bug that needs to be addressed. 

```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
kind: KubeadmConfig
metadata:
  name: workload-cluster-controlplane-3
  namespace: default
spec:
  joinConfiguration:
    controlPlane: {}
    nodeRegistration:
      kubeletExtraArgs:
        cloud-provider: aws
      name: '{{ ds.meta_data.hostname }}'
---
apiVersion: cluster.x-k8s.io/v1alpha2
kind: Machine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: workload-cluster
    cluster.x-k8s.io/control-plane: "true"
  name: workload-cluster-controlplane-3
  namespace: default
spec:
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
      kind: KubeadmConfig
      name: workload-cluster-controlplane-3
      namespace: default
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
    kind: AWSMachine
    name: workload-cluster-controlplane-3
    namespace: default
  version: v1.16.1
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
kind: AWSMachine
metadata:
  name: workload-cluster-controlplane-3
  namespace: default
spec:
  iamInstanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
  instanceType: t2.medium
  sshKeyName: awsbastion
```
Change the `instanceType`, `sshKeyName`, `K8s version` and other name references accordingly.  Once done, apply the configuration file.

```shell
kubectl apply -f ./examples/_out/addmasternode.yaml
```
where `./examples/_out/addmasternode.yaml` is the name of the newly created file. Once successfully completed, a new EC2 instance is created, kubeadm joins the node to the cluster as a master node. Validate the results using the AWS EC2 console. You should also observe an additional `instance in server` in the LB configuration.  In the current example, observe the 4 master nodes. 

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf get nodes
```
```console
NAME                                       STATUS   ROLES    AGE   VERSION
ip-10-0-0-18.us-east-2.compute.internal    Ready    master   16h   v1.16.1
ip-10-0-0-200.us-east-2.compute.internal   Ready    master   16h   v1.16.1
ip-10-0-0-226.us-east-2.compute.internal   Ready    master   73s   v1.16.1
ip-10-0-0-35.us-east-2.compute.internal    Ready    master   16h   v1.16.1
...
```
Notice the machines in the management cluster - 

```shell
kubectl get ma -n default
```
```console
kubectl get ma -n default
NAME                                     PROVIDERID                    PHASE
workload-cluster-controlplane-0          aws:////i-09558d582b46da80b   running
workload-cluster-controlplane-1          aws:////i-06543eae83b8a5378   running
workload-cluster-controlplane-2          aws:////i-0ea2bf3d7fe0dc66f   running
workload-cluster-controlplane-3          aws:////i-02dbc49e4f668bdfd   running
...
```
### Manually decreasing # of Control Plane Nodes
Since control plane nodes are not managed by a machinedeployment, a control plane machine object and its associated references would need to be deleted. [check if there is a better way]. In this example, we will delete the `workload-cluster-controlplane-2` machine. Doing so deletes its associated AWSMachine `workload-cluster-controlplane-2` and KubeadmConfig `workload-cluster-controlplane-2`

```shell
kubectl delete ma workload-cluster-controlplane-2
```
Run the following commands to validate that the objects have been removed and EC2 instance has been terminated in the AWS console.
```shell
kubectl get ma
kubectl get AWSMachine
kubectl get KubeadmConfig
```
### ~~Resurrection of a control plane node~~ 
- [this does now work as v1alpha2 does not support resurrection???]
  
For this demo, we will increase the worker node count back to 4 (in not already done). This can per performed by the steps in the section `Manually increasing the # of control plane nodes` Once the cluster has reached a steady state, and all the master nodes are available in the cluster, terminate one of the **master nodes** from the AWS ec2 console. 

~~When the controller next runs the reconciliation loop and sees a missing machine, a new ec2 instance should be spawned and added to the workload cluster as a master node.~~

Notice the state of the machine - 

```shell
kubectl get ma
```

```console
NAME                                     PROVIDERID                    PHASE
...
workload-cluster-controlplane-3          aws:////i-02dbc49e4f668bdfd   running
workload-cluster-controlplane-4          aws:////i-02dcb5b3fd5ada5aa   failed
...
```
To cleanup, delete the failed machine. 

```shell
kubectl delete ma workload-cluster-controlplane-4
```
### Accessing EC2 instances

Note the IP/Public DNS of the bastion host associated with the cluster. For e.g. `ec2-[ip-address].us-east-2.compute.amazonaws.com`

Use the ssh key that was references in the previous section to ssh into the bastion host. For e.g.

```shell
ssh -i [path/to]/awsbastion.pem ubuntu@ec2-ip-address.us-east-2.compute.amazonaws.com
```
```console
The authenticity of host 'ec2-18-223-108-221.us-east-2.compute.amazonaws.com (18.223.108.221)' can't be established.
ECDSA key fingerprint is SHA256:p9ffx8FX6LtmhNoAOUko5mGBYXK9F8tZh1TdfEcT7gI.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'ec2-18-223-108-221.us-east-2.compute.amazonaws.com,18.223.108.221' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-1047-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
...
```
Once you are logged into the bastion host, you have to copy [scp or copy&paste] the above ssh key to the bastion host. Save it as a `.pem` file and set the access mode to `600`. For example - 

```shell
vi ~/.ssh/awsbastion.pem
#Copy and paste the content of the key and save the file.
chmod 600 ~/.ssh/awsbastion.pem
```

Now that the private key file has been transferred to the bastion host, you can connect to any of the EC2 instances that constitute the control plane and worker nodes. Use the `Private DNS Name` from the AWS EC2 console for connection. 

```shell
ssh -i ~/.ssh/awsbastion.pem ubuntu@ip-10-0-0-226.us-east-2.compute.internal
```
```console
The authenticity of host 'ip-10-0-0-226.us-east-2.compute.internal (10.0.0.226)' can't be established.
ECDSA key fingerprint is SHA256:EQIykeXKgZiVslTw/QdyNfH2fZ/V+ZjgMkcqqsRcQNE.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'ip-10-0-0-226.us-east-2.compute.internal,10.0.0.226' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-1052-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Jan 18 22:54:33 UTC 2020
...
```

### Upgrading K8s version on Worker Nodes.

Note only minor version upgrades are recommended. For e.g. 1.16.1 -> 1.16.2. (need to verify if this is even a recommended scenario, if the control plane is still at the lower version - e.g. 1.16.1)

Edit the `machinedeployment.yaml` file and modify the version number to the required minor version 

```yaml
....
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
        kind: AWSMachineTemplate
        name: workload-cluster-aws-1-md-0
        namespace: default
      version: 1.16.2
...
```
Once done, apply the yaml.

```shell
kubectl apply -f machinedeployment.yaml
```
Flow - A new EC2 instance is deployed (using an AMI that available for the new version of Kubernetes requested). Kubeadm workflow joins the instance as a new node to the cluster. One of the older nodes, that is part of the machinedeployment being upgrades, is evacuated, evicted from the cluster and the EC2 instance is terminated. These steps are repeated until all the nodes have been refreshed. 

Once all the nodes have been updates, the version will reflect the new version.
#### Before <!-- omit in toc -->
```console
NAME                                        STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
...
ip-10-20-0-10.us-east-2.compute.internal    Ready    <none>   6m38s   v1.16.1   10.20.0.10    <none>        Ubuntu 18.04.3 LTS   4.15.0-1052-aws   containerd://1.3.0
ip-10-20-0-79.us-east-2.compute.internal    Ready    <none>   6m49s   v1.16.1   10.20.0.79    <none>        Ubuntu 18.04.3 LTS   4.15.0-1052-aws   containerd://1.3.0
...
```
#### After <!-- omit in toc -->

```console
NAME                                        STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
...
ip-10-20-0-141.us-east-2.compute.internal   Ready    <none>   10m   v1.16.2   10.20.0.141   <none>        Ubuntu 18.04.3 LTS   4.15.0-1052-aws   containerd://1.3.0
ip-10-20-0-193.us-east-2.compute.internal   Ready    <none>   12m   v1.16.2   10.20.0.193   <none>        Ubuntu 18.04.3 LTS   4.15.0-1052-aws   containerd://1.3.0
...
```

### Destroying/Deleting a Workload cluster

This should be a relatively simple task of deleting the cluster -

```shell
kubectl delete cluster workload-cluster
```
This will take some time as all EC2 instances and AWS VPC and related objects are destroyed. 
```console
cluster.cluster.x-k8s.io "workload-cluster" deleted
```
## References

1. [https://blog.scottlowe.org/2019/08/27/bootstrapping-a-kubernetes-cluster-on-aws-with-clusterapi/](https://blog.scottlowe.org/2019/08/27/bootstrapping-a-kubernetes-cluster-on-aws-with-clusterapi/)
2. [https://cluster-api.sigs.k8s.io/tasks/installation.html](https://cluster-api.sigs.k8s.io/tasks/installation.html)
3. [https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/v0.3.7/docs/getting-started.md](https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/v0.3.7/docs/getting-started.md)
4. [https://blog.chernand.io/2019/03/19/getting-familiar-with-clusterapi/](https://blog.chernand.io/2019/03/19/getting-familiar-with-clusterapi/)
5. [https://medium.com/condenastengineering/clusterapi-a-guide-on-how-to-get-started-ff9a81262945](https://medium.com/condenastengineering/clusterapi-a-guide-on-how-to-get-started-ff9a81262945).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNzk2ODYxNjM5LDUyODk0MzA5N119
-->
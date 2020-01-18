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
cluster-api-controller                  - v1alpha2.cluster.x-k8s.io
cluster-api-aws-controller              - v1alpha2.infrastructure.cluster.x-k8s.io
cluster-api-kubeadm-controller          - v1alpha2.bootstrap.cluster.x-k8s.io
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

----
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

This may take a while - 10-15 mins. 

* Step 3 - Once this is complete, we need to grab the kubeconfig file that was associated with this workload server. The kubeconfig is stored as a secret in the namespace associated with the cluster (default in this example). This will allow us to connect to the cluster. 

```shell
kubectl get secrets -n default workload-cluster-kubeconfig -o json |jq -r .data.value|base64 -d > /tmp/workload-cluster.conf
```
The workload-cluster.conf file should be in a kubeconfig file format -

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5ekNDQWJPZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01ERXhPREF6TXpBd01Gb1hEVE13TURFeE5UQXpNelV3TUZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTFN5CjhLYy9lTVdCMzN2MXpaWVpkNjdubE8rd1oyUEN1Z0dIeXE0dTU3eDJ3YmJFcy9mRGpMVWZQOUJ1R1M5Ym4wY3QKeWthZVRrcGpyNk96dzRyR0E0dm1ZVUtLbFNkbXpUcC9LZElSUnFBa3ZCUHE5VGZPT3BZNmJ4cVR3QUVVYTB5Zwo1R0VHRm5JUWkwMW5zaEFyeitQS3VZUFNNSk5ML01aaThYSGt0cGVrVmtsMGhMUVZCcnlmUklwM3lnTk5LU1g3CjNMMVBnNmJuT1lybG5Ub1ZrcERnLzlxTXh3N0RqRzB6bDFQcFZOOHNlU1gzTEtHbjR0bUpjRjdKMGwrdTFvZ1cKaTFpOTNH
    ...
contexts:
- context:
    cluster: workload-cluster
...
```
* Step 4 - Install the CNI addons to the workload cluster.

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

* Step 5 - Deploy the worker nodes leveraging the cluster API.

```shell
kubectl apply -f ./examples/_out/machinedeployment.yaml
```
```console
kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io/workload-cluster-md-0 created
machinedeployment.cluster.x-k8s.io/workload-cluster-md-0 created
awsmachinetemplate.infrastructure.cluster.x-k8s.io/workload-cluster-md-0 created
```
In a short time, a machine deployment consisting of 2 machines (2 worker nodes) will be added to the cluster. 

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf get nodes
```
should return something similar - 

```console
NAME                                       STATUS   ROLES    AGE     VERSION
ip-10-0-0-158.us-east-2.compute.internal   Ready    <none>   5m30s   v1.16.1
ip-10-0-0-18.us-east-2.compute.internal    Ready    master   18m     v1.16.1
ip-10-0-0-19.us-east-2.compute.internal    Ready    <none>   5m17s   v1.16.1
ip-10-0-0-200.us-east-2.compute.internal   Ready    master   19m     v1.16.1
ip-10-0-0-35.us-east-2.compute.internal    Ready    master   18m     v1.16.1
```


----------

References - 

1. [https://blog.scottlowe.org/2019/08/27/bootstrapping-a-kubernetes-cluster-on-aws-with-clusterapi/](https://blog.scottlowe.org/2019/08/27/bootstrapping-a-kubernetes-cluster-on-aws-with-clusterapi/)
2. [https://cluster-api.sigs.k8s.io/tasks/installation.html](https://cluster-api.sigs.k8s.io/tasks/installation.html)
3. [https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/v0.3.7/docs/getting-started.md](https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/v0.3.7/docs/getting-started.md)
4. [https://blog.chernand.io/2019/03/19/getting-familiar-with-clusterapi/](https://blog.chernand.io/2019/03/19/getting-familiar-with-clusterapi/)
5. [https://medium.com/condenastengineering/clusterapi-a-guide-on-how-to-get-started-ff9a81262945](https://medium.com/condenastengineering/clusterapi-a-guide-on-how-to-get-started-ff9a81262945)
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTM1OTI0MTE3NSwtNzg5MDY5NTI1LC0xMz
QzMDYxMTY2LDEwNzY3MTk1OSwtMTY4Njg1NzQxM119
-->
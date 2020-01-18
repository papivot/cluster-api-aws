# Cluster API on AWS

### Working with **release-0.4** providing support for **v1alpha2**

The following CRDs are available as part of this release - 

```
clusters.cluster.x-k8s.io
machinedeployments.cluster.x-k8s.io
machines.cluster.x-k8s.io
machinesets.cluster.x-k8s.io
```
```
awsclusters.infrastructure.cluster.x-k8s.io
awsmachines.infrastructure.cluster.x-k8s.io
awsmachinetemplates.infrastructure.cluster.x-k8s.io
```
```
kubeadmconfigs.bootstrap.cluster.x-k8s.io
kubeadmconfigtemplates.bootstrap.cluster.x-k8s.io
```

Users have access to the following api resources -
```
clusters 				cl 		cluster.x-k8s.io 	true 	Cluster
machinedeployments 		md 		cluster.x-k8s.io 	true 	MachineDeployment
machines 				ma 		cluster.x-k8s.io 	true 	Machine
machinesets 			ms 		cluster.x-k8s.io 	true 	MachineSet

awsclusters 						infrastructure.cluster.x-k8s.io true AWSCluster
awsmachines 						infrastructure.cluster.x-k8s.io true AWSMachine
awsmachinetemplates 				infrastructure.cluster.x-k8s.io true AWSMachineTemplate

kubeadmconfigs bootstrap.cluster.x-k8s.io true KubeadmConfig
kubeadmconfigtemplates bootstrap.cluster.x-k8s.io true KubeadmConfigTemplate
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE4NjI1NDQwMDFdfQ==
-->
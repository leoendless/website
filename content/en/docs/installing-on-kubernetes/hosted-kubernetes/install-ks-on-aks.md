---
title: "Deploy KubeSphere on AKS"
keywords: "kubesphere, kubernetes, docker, Azure, AKS"
description: "How to deploy KubeSphere on AKS"
---

This guide walks you through the steps of deploying KubeSphere on [Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/).

## Prepare a AKS cluster

Azure can help you implement infrastructure as code by providing resource deployment automation options; the commonly adopted tool is “Arm templates”, one of the other options is Azure CLI. In this instruction we will use Azure CLI to create all the resources that are needed by kubesphere.

### Use Azure Cloud Shell
You don't have to install the Azure CLI on your machine,  Azure provided a web based terminal. Select the Cloud Shell button on the menu bar at the upper right in the Azure portal.

![Cloud Shell](/images/docs/aks/aks-launch-icon.png)

Select the **Bash** Shell

![Bash Shell](/images/docs/aks/aks-choices-bash.png)
### Create a resource group

An Azure resource group is a logical group in which Azure resources are deployed and managed. The following example creates a resource group named KubeSphereRG in the westus location. 

```bash
az group create --name KubeSphereRG --location westus
```

### Create AKS cluster
Use the **az aks create** command to create an AKS cluster. The following example creates a cluster named KuberSphereCluster with three nodes. This will take several minutes to complete.

```bash
az aks create --resource-group KubeSphereRG --name KuberSphereCluster --node-count 3 --enable-addons monitoring --generate-ssh-keys
```
> Notes:
> - You can use --node-vm-size or -s option to change the Size of Kubernetes nodes. Default: Standard_DS2_v2( 2vCPU, 7GB memory).
> - More options can be found at https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az-aks-create

### Connect to the cluster
To configure kubectl to connect to Kubernetes cluster, use the **az aks get-credentials** command. This command downloads credentials and configures the Kubernetes CLI to use them.

```bash
az aks get-credentials --resource-group KubeSphereRG --name KuberSphereCluster
```

```
kebesphere@Azure:~$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-23754246-vmss000000   Ready    agent   38m   v1.16.13
```
### Check Azure Resources in the Portal
After we execute all the commands above. We should see there are 2 Resource Groups created in the Azure Portal.

![Resource groups](/images/docs/aks/aks-create-command.png)

The Azure Kubernetes Services itself will be placed in the KubeSphereRG. 
 
![Azure Kubernetes Services](/images/docs/aks/aks-dashboard.png)

All the other Resources will be placed in MC_KubeSphereRG_KuberSphereCluster_westus, such as VMs, Load Balancer, Virtual Network, etc.

![Azure Kubernetes Services](/images/docs/aks/aks-all-resources.png)

## Deploy KubeSphere
To Start Deploying KubeSphere, using the following command.
```bash
kubectl apply -f https://raw.githubusercontent.com/kubesphere/ks-installer/master/deploy/kubesphere-installer.yaml
```
Download the cluster-configuration.yaml, so you can customize the configuration. You can also enable pluggable components by set the **enabled** property to **true** in this file.
```bash
wget https://raw.githubusercontent.com/kubesphere/ks-installer/master/deploy/cluster-configuration.yaml
```
The metrics-server was already installed on the AKS, you need disable the component in the cluster-configuration.yaml file. 
```bash
kebesphere@Azure:~$ vim ./cluster-configuration.yaml
---
  metrics_server:                    # (CPU: 56 m, Memory: 44.35 MiB) Whether to install metrics-server. IT enables HPA (Horizontal Pod Autoscaler).
    enabled: false
---
```
Installation process will be started after the cluster configuration applied.
```bash
kubectl apply -f ./cluster-configuration.yaml
```

## Access KubeSphere console

To access KubeSphere console from public ip address, you need change service type to LoadBalancer.
```bash
kubectl edit service ks-console -n kubesphere-system
```
Find the following section, and change the type to **LoadBalancer**.
```
spec:
  clusterIP: 10.0.78.113
  externalTrafficPolicy: Cluster
  ports:
  - name: nginx
    nodePort: 30880
    port: 80
    protocol: TCP
    targetPort: 8000
  selector:
    app: ks-console
    tier: frontend
    version: v3.0.0
  sessionAffinity: None
  type: LoadBalancer # Change the NodePort to LoadBalancer
status:
  loadBalancer: {}
```
After saved the ks-console Service, you can use the following command to get the public ip address.
```
kebesphere@Azure:~$ kubectl get svc/ks-console -n kubesphere-system
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
ks-console   LoadBalancer   10.0.181.93   13.86.xxx.xxx   80:30194/TCP   13m       6379/TCP       10m
```
## Enable Pluggable Components (Optional)
Pluggable components are designed to be pluggable which means you can enable them either before or after installation. See [Enable Pluggable Components](https://github.com/kubesphere/ks-installer#enable-pluggable-components) for details.
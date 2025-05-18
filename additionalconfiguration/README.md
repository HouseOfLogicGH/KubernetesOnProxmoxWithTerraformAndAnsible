
# Additional Configuration for Local Kubernetes - Data Inputs and Outputs

If you have followed the previous videos/instructions you should have a working Kubernetes Cluster running locally, and ideally have Lens configured as well.

You may now be wondering what you can do with it and specifically how to connect it up to other things. These instructions are intended to get you started with this.

The initial configuration steps can be found [here](https://github.com/HouseOfLogicGH/KubernetesOnProxmoxWithTerraformAndAnsible/blob/main/README.md).

## NFS for Local Persistent Data Storage 

NFS gives you a simple way of storing data that can be accessed by Kubernetes that is persistent.

To use this, you will need to install NFS on each of the nodes:

For Ubuntu/Debian
```
sudo apt-get install nfs-common
```
For RHEL/CentOS
```
sudo yum install nfs-utils
```

Test the connectivity from the nodes using:

```
sudo mount -t nfs 192.168.5.210:/volume1/k8snfs /mnt
```
Navigate into the share and check it (and optionally create an index.html file), then dismount the NFS share using:
```
sudo umount /mnt
```

To connect on Kubernetes, you next need to create an NFS storage class, persistent volume and persistent volume claim for each service you will be running (update the options, such as IP addresses and paths, as applicable for your environment).

Each .yml file can either be applied using kubectl apply -f filename.yml or by pasting the contents into Lens using the "create resource" option.

Note: The configuration for the persistent volume is intended that many services share the same volume claim in this instance. You may prefer separate volumes, and separate claims. 

## Example Service Application

You can now create a service that will be backed by persistent storage - this is the example-service-deployment.yml file in this example, and again can be deployed using Lens or kubectl.

With the application deployed, you can now create a clusterip service for internal access in the cluster, and a nodeport based service that will allow access to the application via an IP and Port.

Within the cluster (from other pods), the clusterip service can be accessed:

```
# Create a debug pod
kubectl run curl-debug --image=curlimages/curl --rm -it -- sh

# Once inside the pod, try to reach your service
curl http://example-service
```

Externally, the application can be accessed using the IP address of any node and port specified in the nodeport, eg http://192.168.5.231:30080

Neither of these configurations are typically used for production level deployments, which typically use an ingress and a load balancer.

If you are happy accessing your application using an IP address and port combination, you do not need to continue with the next steps.

If you do want to follow the next section (or want to skip ClusterIP and NodePort), you should deploy the example-service-ingress-service.yml file.

## Load Balancer Setup

Cloud-deployed Kubernetes typically interfaces with the cloud provider's software defined networking for load balancer provisioning. This is not possible with self-hosted deployments, but MetalLB offers a solution instead.

Create the metallb namespace and configmap using Kubectl or Lens to create the resources. Make sure the config map references the IP addresses you want to use for services. Layer 2 networking is preferable for home lab use, but metallb does support other options.

Next, apply the metallb manifest:

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

With this configured, you are nearly complete but an additional config step is required as described [on this metallb config page](https://metallb.io/configuration/migration_to_crds/#procedure).

Run the following commands on the kubernetes master:
```
kubectl get configmap -n metallb-system -o yaml config > config.yaml
docker run -d -v $(pwd):/var/input quay.io/metallb/configmaptocrs
```
This creates Custom Resource Definitions for the configuration to be applied to MetalLB.
Without this option, your configuration will not be applied to metallb and the load balancer will not work as intended.

## Ingress Setup

nginx can be used as an ingress controller for your application. It will need to be installed using helm:

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install nginx-ingress ingress-nginx/ingress-nginx
```

When this completes, you should be able to check for the ingress controller's external IP address:
```
kubectl get service --namespace default nginx-ingress-ingress-nginx-controller --output wide
```

You can now deploy the example-service-ingress-ingress.yml using Lens / Kubectl as before.

When this completes, the external IP address of the service will be that of the ingress controller. You will need to either setup local DNS or create a hostfile entry on each system needing to access the application.

With this in place, you should be able to access the application using its address, eg http://example.yourdomain.lan

If all the steps are followed and files are deployed, you should have three instances of example-service with different ways of accessing them, all using the same example-service application deployment.
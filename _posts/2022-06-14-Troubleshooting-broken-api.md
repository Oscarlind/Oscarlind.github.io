## Getting into a cluster with a broken API server
Every now and then I have had to access clusters where it is no longer possible to login to. Usually this has been due to expired certificates or the underlying infrastructure/platform, e.g. VMware is unhealthy and causing issues in the cluster.

Here I write a little about that and how you can regain access to the cluster in order to fix the issue or continue the troubleshooting.
### The Problem - Expired Certificates
The problem can manifest itself in different ways. The perhaps, in my experience, most common one is expired certificates. Trying to talk with the API should give a error message like this:
```yaml
x509: certificate signed by unknown authority
```
This issue is usually quite easily fixed. One needs _just_ to log in and replace the expired certificate to a current one.

In this scenario, you can try to use the __~/.kube/config__ file that was created during the installation. If that file is no longer available, or is not working, we will have to create a new one.

### Problem with the underlying infrastructure
If the underlying infrastructure is causing issues to the cluster, it can also prevent you from accessing it consistently through the API. In these cases it can be worthwhile to gain access to the API in order to do more troubleshooting. This process can very often be applied then as well (as long as the api-server is running).
## Creating a recovery kubeconfig file
As long as the __kube-apiserver__ container is running, it will be possible to use its recovery _ca_ and _token_ to access the cluster with full privileges. The process can be outlined in three steps:

1. SSH into the control plane (any node running the api-server, should be master nodes).
2. Extract the recovery ca.crt and token
3. Reference these extracted items with a custom kubeconfig file

Below I've created a short script automating the process, after you've connected to a master node. It is based on the assumption that the container runtime interface is __CRI-O__. If you are using something else you can substitute the `crictl` commands. 

Run the script like this:
`. ./kubeconf.sh `

Doing so will source the new kubeconfig to your current shell - allowing you to skip exporting the variable afterwards.
```bash
#!/bin/bash
# Get apiserver name
apiserver=$(crictl ps | grep kube-apiserver[[:space:]] | awk '{ print $1 }')

# Create a kubeconfig and fetch the recovery items.
crictl exec $apiserver cat /etc/kubernetes/static-pod-resources/configmaps/kube-apiserver-cert-syncer-kubeconfig/kubeconfig > /tmp/kubeconfig
crictl exec $apiserver cat /etc/kubernetes/static-pod-resources/secrets/localhost-recovery-client-token/ca.crt > /tmp/ca.crt
crictl exec $apiserver cat /etc/kubernetes/static-pod-resources/secrets/localhost-recovery-client-token/token > /tmp/token

# Change the kubeconfig into using the recovery items.
sed -i 's|/etc/kubernetes/static-pod-resources/configmaps/kube-apiserver-server-ca/ca-bundle.crt|/tmp/ca.crt|' /tmp/kubeconfig
sed -i 's|/etc/kubernetes/static-pod-resources/secrets/localhost-recovery-client-token/token|/tmp/token|' /tmp/kubeconfig

# Export the kubeconfig and get access to the cluster
export KUBECONFIG=/tmp/kubeconfig
```

## Conclusions
Troubleshooting cluster issues can be tedious, especially if you get locked out of your own cluster. This is a little trick that can help you gain access to the API even with it having its certificates expired, or with the cluster having some pretty major problems otherwise, for example with authentication.
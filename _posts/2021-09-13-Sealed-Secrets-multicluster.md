## Sealed Secrets in a Multi-Cluster Environment

A problem everyone encounters when they decide to start working with GitOps is secret-management. How should one handle secrets when everything is to be in a git repository? There is today no standardized solution and the alternatives are plenty.

A popular one is called Sealed Secrets. This tool lets us encrypt standard Kubernetes secrets that can then be added to a git repository. A controller will then decrypt the secret when it is applied to the cluster.

![sealed-secrets](https://user-images.githubusercontent.com/60172320/133062274-f16d66ff-adcf-428d-8c2a-c448faedfbf6.png)


The concept is quite straightforward. We have a private key in the cluster and use a public key to encrypt our secrets. These encrypted secrets can then be added to our repository. When we want to use our secret, we apply the encrypted secret and the controller in our cluster will thereafter use the private key to decrypt it and create a regular Kubernetes secret of it.

What we need to do here is to ensure that the private key is well stored. But how does this work when we have multiple clusters? Since we have a private key per controller does this mean we will use different keys for different clusters? The answer to this is, we can but we do not need to.

Below I will describe two possible implementations of Sealed Secrets in multi-cluster environments.

### Scenario 1 - Separate keys

In this scenario we deploy Sealed Secrets in each cluster and let the tool generate its own keys. What is important to think about in this scenario is that a Sealed Secret that is created with a controller for one cluster can’t be used in another. This is since it can’t be decrypted there. Therefore we need to be observant when we encrypt our secret and also when we apply it to our cluster.

![Multi-key-sealed-secret](https://user-images.githubusercontent.com/60172320/133062413-6458c6bc-a198-44b5-8699-5810c464a616.png)

In this scenario we therefore need to encrypt Cluster-1’s secrets with the certificate from Cluster-1’s controller. For this secret, now Sealed Secret, to be able to be used, it must be deployed in Cluster-1.

Why? Because in Cluster-2 and Cluster-3 our controllers have different private keys and can therefore not decrypt the secret.

In this example with three clusters we also have three different private keys to keep track of. We can quickly see that this alternative gets more complicated the more clusters it is used on.

### Scenario 2 - Single key

When we deploy Sealed Secrets the controller creates its own private key and associated certificate. However, we can add to this by giving it our own certificate/private keys. When the controller gets deployed it searches for a secret called sealed-secrets-key as well as secrets with the label sealedsecrets.bitnami.com/sealed-secrets-key=active. This or these secrets must exist in the same namespace as our controller.


This gives us the opportunity to use the same key and certificate for several clusters.

![SingleController-SS](https://user-images.githubusercontent.com/60172320/133062378-2528f3da-6ba6-45d3-9e81-2440a0dde39b.png)

By deploying a controller and then extracting its private key, we can then also use this for controllers in other clusters.

Advantages with this method is that the tool gets a lot simpler to scale. We do not need to think about which Sealed Secret belongs to which cluster. To only manage a single private key is also a lot easier than managing several.

Of course, this also increases the importance of the private key. It should naturally be managed in an appropriate manner - for example through a vault. 
### Deploying with a Single Key Strategy

First thing we need to do is set up Sealed Secrets in our cluster and install the CLI if we do not already have it.

``` 
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.16.0/kubeseal-linux-amd64 -O kubeseal

sudo install -m 755 kubeseal /usr/local/bin/kubeseal

oc apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.16.0/controller.yaml
```

>- **NOTE:** If we want to install Sealed Secrets in a namespace other than kube-system, install via Kustomize.

We can thereafter check if our controller is up and running.

```
oc get pods -n kube-system
NAME                                            READY     STATUS     RESTARTS     AGE    
sealed-secrets-controller-69c88fd9dc-pw55v      1/1       Running    0            13s
```

Now that we have verified that it is installed, and up and running, we can pull down the private key. But first, to try this out, create a Sealed Secret.

We’ll start with a regular K8s secret.

```
oc create secret generic test-secret --dry-run=client --from-literal=passphrase=topsecret -n kube-system -o yaml > test-secret.yaml
```

Thereafter we will use kubeseal to encrypt it.

```yaml
kubeseal -o yaml <test-secret.yaml > sealed-test-secret.yaml

➜ cat sealed-test-secret.yaml 
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: test-secret
  namespace: kube-system
spec:
  encryptedData:
    passphrase: AgCN2encE0Z2duMKVzwr2NF3FEycQUxLa9qcn+QY8AtEZDwSnEhbyg1p3ARIXo1IFR6wryd/NlpK1V7pm4f7vH93FSMvmJF9UkRO6B7xB53DA+PkXKCAHDtT26jwjTL6WGfpFxaYHPOiAzrH7+U5aveGBiT5W3hXKDCUzvWCLcc2nJsGMvd01hqCdmtbMfB1Slyc69X5GgPlOizz5H+74/nczBOnOnYGZZY/PoeVr2Fxnpj7j1f7usmM3g855tcRVWPZA27Y34sODWLmEqv/RYFjM352nkzk768Wv9I6Ftca1XRM/TZfXjoJL1f5rn9AEhXcXGBb7xUA6dLfezgEXLjnePpwYZ17BMnTKWl0w11tvJvWKWK9XlCDopkbZYHc0rdSiDoHjhQpvRjdFb0sCAJ8Y4BppLuSA3n86a+zJ/Sj+qLrDmcA1y+KIZp+JoB9RNERGiX3KiZFoleOx3hAyOuS5oa7OTPk94WH31H6UOuBSWqfunDVezifSpooamkl+06TFKRZ9AGNL+phPcajIhLkAXVAo3f7xhQx/zo8Cz0mwYtAY/rJxjT13o5RY2MyU9GGkjMlV6EJwNSwlUL/3x8bmzt5h2yh42/c9ZjbiDQZnTeO2noXgGR122EYMNJwoGCdRsenDsMLjUXOy7YN6x3MA1gYRCJqeiKVMKLDVMgMSWMDUK8CbBIUWSjhAsokLGqmb3KvXnrMPKM=
  template:
    metadata:
      creationTimestamp: null
      name: test-secret
      namespace: kube-system
```

To get the actual key we run:

```yaml
oc get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml >master.key

➜ cat master.key 
apiVersion: v1
items:
- apiVersion: v1
  data:
    tls.crt: 
    <long string>
    tls.key: 
    <another long string>
  kind: Secret
  metadata:
    creationTimestamp: "2021-08-24T11:54:54Z"
    generateName: sealed-secrets-key
    labels:
      sealedsecrets.bitnami.com/sealed-secrets-key: active
    name: sealed-secrets-keysxxd8
    namespace: kube-system
    resourceVersion: "581287"
    uid: 4895de98-3dd1-4939-87b3-0ec48d9908d7
  type: kubernetes.io/tls
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""

```

What we just pulled out is a secret that contains the self-generated keypair.

As we can see, we now have a key and a certificate belonging to it. These are also, like all k8s secrets, base64 encoded.

![base64-masterkey](https://user-images.githubusercontent.com/60172320/133062545-f96c1f16-adf2-4d5d-a2df-10595f1e0d5d.png)

We can now do a base64 --decode on these values and get the actual certificate and key. Add these values to two files, for example “sealed.crt” and “sealed.key”.


Now that we have these files we can use them for a Sealed Secret controller in a new cluster.

To understand how this works we need to understand what the controller does when it gets deployed. When we set up Sealed Secrets in the cluster, we spin up a controller that looks for a secret in the same namespace called sealed-secret-key. If this secret does not exist it will get generated - and will then get a name such as sealed-secrets-keyXXXXX.

On the other hand, if there already is a secret with this name in the namespace the controller will make use of it. In this way we can control the actual key that is going to be used by default for our cluster.

Next step will then be to create this secret using our sealed.crt and sealed.key in a new cluster.

```sh
oc create secret tls sealed-secrets-key --key=sealed.key --cert=sealed.crt -n kube-system
```

Now when we deploy Sealed Secrets we can verify that it actually uses the key we created. We can do this by checking the logs.

```yaml
oc logs sealed-secrets-controller-69c88fd9dc-kwdqt
controller version: v0.16.0
2021/08/25 09:30:17 Starting sealed-secrets controller version: v0.16.0
2021/08/25 09:30:17 Searching for existing private keys
2021/08/25 09:30:17 ----- sealed-secrets-key
2021/08/25 09:30:17 HTTP server serving on :8080
2021/08/25 09:30:17 Updating kube-system/test-secret
2021/08/25 09:30:17 Event(v1.ObjectReference{kind:"SealedSecret", Namespace: "kube-system",.....})
```

*Searching for existing private keys
----- sealed-secrets-key*

  
This shows that it did not have to generate the secret by itself, but searched after and found the one we just created. 

Since this secret is identical to the earlier one, the same Sealed Secret - the one we created in the first cluster - can be decrypted even in this second cluster.

Note: When a secret gets encrypted, its name and namespace is used during the actual encryption process. This means that

1. It can only be decrypted if the name of the secret is the same as when it was encrypted.
2. It can only be decrypted in the namespace it was encrypted for. In other words, if the secret was created for namespace1 it won't be able to get decrypted in namespace2. 

These are security measures and default settings. We can change these by using the --scope argument when encrypting our secret.

## Conclusions

When working with GitOps, secret management can be a tricky subject. Sealed Secrets are one of many alternative solutions to this problem. Here we have discussed two different approaches of using the tool in a multi-cluster environment. Further, we have seen a practical example of how to deploy it in a pragmatic way when dealing with several clusters.
## References

1. https://github.com/bitnami-labs/sealed-secrets
2. https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/bring-your-own-certificates.md

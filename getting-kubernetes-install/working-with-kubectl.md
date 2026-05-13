---
icon: kubernetes
---

# Working with kubectl

### `kubectl` <a href="#kubectl" id="kubectl"></a>

`kubectl` is the Kubernetes command-line tool, and you’ll use it in all the hands-on examples. You’ll already have it if you’ve followed the instructions to install either of the clusters.

Type `kubectl` in a terminal window to check if you have it. If you don’t have it, search the web for install `kubectl` and follow the instructions for your system.

It’s important that your `kubectl` version is no more than one minor version higher or lower than your cluster. For example, if your cluster is running Kubernetes 1.29.x, your `kubectl` should be no lower than 1.28.x and no higher than 1.30.x.

At a high level, `kubectl` converts user-friendly commands into HTTP REST requests and sends them to the API server. Behind the scenes, it reads a kubeconfig file to know which cluster to send commands to and which credentials to use.

The kubeconfig file is called `config` and lives in your home directory’s hidden `.kube` folder. It contains definitions for:

* Clusters
* Users (credentials)
* Contexts

**Clusters** is a list of Kubernetes clusters that `kubectl` knows about and allows a single `kubectl` installation to manage multiple clusters. Each cluster definition has a name, certificate info, and API server endpoint.

**Users** is a list of user credentials. For example, you might have a dev user and an ops user with different permissions. Each of these exists in the kubeconfig file and has a friendly name and a set of credentials. If you’re using X.509 certificates, the username and group are embedded in the certificates.

**Contexts** are how `kubectl` groups clusters and users under a friendly name. For example, you might have a context called ops-prod that combines the ops user credentials with the prod cluster. Using `kubectl` with this context will send commands to the API server of the prod cluster and authenticate as the ops user.

The following is a simple kubeconfig file with a single cluster called shield, a single user called coulson, and a single context called director. The director context combines the coulson user and the shield cluster. It’s also set as the default context.

```yaml
apiVersion: v1
kind: Config
clusters:                                      <<==== Cluster definitions in this block
- name: shield                                 <<==== Friendly name for a cluster
  cluster:
    server: https://192.168.1.77:8443          <<==== Cluster's AIP endpoint
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJ   <<==== Cluster's certificate
users:                                         <<==== User definitions in this block
- name: coulson                                <<==== Friendly name not used by Kubernetes
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRV...   <<==== User certificate
    client-key-data: LS0tLS1CRUdJTiBFQyB       <<==== User private key
contexts:                                      <<==== Contexts in this block
- context:                                 
  name: director                               <<==== Context called "director"
    cluster: shield                            <<==== Send commands to this cluster
    user: coulson                              <<==== Authenticate as this user
current-context: director                      <<==== kubectl will use this context

// Example of a kubeconfig with a shield cluster
```

You can run a `kubectl config view` command to view your kubeconfig. The command will redact sensitive data.

You can see your current context with the `kubectl config current-context` command. The following example shows a system with `kubectl` configured to use the cluster and user defined in the Docker Desktop context.

```powershell
$ kubectl config current-context
kind-kind

// Get the current context
```

You can change the current context by running a `kubectl config use-context` command. The following command sets the current context to `hpa-test`. It will only work if your kubeconfig file has a valid context called `hpa-test`.

```powershell
$ kubectl config use-context hpa-test
Switched to context "hpa-test".

$ kubectl config current-context
hpa-test

// Switch to a different context
```

> **Note:** Docker Desktop is free for personal and educational use. If you use it for work, and your company has more than 250 employees or over $10M USD in annual revenue, you have to purchase a license.

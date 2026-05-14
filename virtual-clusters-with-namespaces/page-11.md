---
description: Learn how we can create virtual clusters using namespaces.
icon: kubernetes
---

# Namespaces

### Introduction to Namespaces <a href="#introduction-to-namespaces" id="introduction-to-namespaces"></a>

The first thing to know is that Kubernetes Namespaces are not the same as kernel namespaces.

* **Kernel namespaces:** They partition operating systems into virtual operating systems called containers.
* **Kubernetes Namespaces:** They partition Kubernetes clusters into virtual clusters called Namespaces.

> **Note:** We’ll capitalize Namespace when referring to Kubernetes Namespaces. This follows the pattern of capitalizing Kubernetes API objects and clarifies that we’re referring to Kubernetes Namespaces, not kernel namespaces.

It’s also important to know that Namespaces are a form of soft isolation and enable soft multi-tenancy. For example, we can create Namespaces for our dev, test, and qa environments and apply different quotas and policies to each. However, they won’t stop a compromised workload in one Namespace from impacting workloads in other Namespaces.

The following command shows whether objects are namespaced or not. As we can see, most objects are namespaced, meaning we can deploy them to a specific namespace with custom policies and quotas. Objects that aren’t namespaced, such as `Nodes` and `PersistentVolumes`, are cluster-scoped and cannot be isolated to Namespaces.

```powershell
$ kubectl api-resources
NAME                     SHORTNAMES   ...    NAMESPACED   KIND
nodes                    no                  false        Node
persistentvolumeclaims   pvc                 true         PersistentVolumeClaim
persistentvolumes        pv                  false        PersistentVolume
pods                     po                  true         Pod
podtemplates                                 true         PodTemplate
replicationcontrollers   rc                  true         ReplicationController
resourcequotas           quota               true         ResourceQuota
secrets                                      true         Secret
serviceaccounts          sa                  true         ServiceAccount
services                 svc                 true         Service
<Snip>

// Get the details of api-resources
```

Unless we specify otherwise, Kubernetes deploys objects to the `default` Namespace.

***

## Namespace Use Cases <a href="#page-title" id="page-title"></a>

Learn about tenants and go through some use cases of namespaces.

Namespaces are a way for multiple tenants to share the same cluster.

### Tenant <a href="#tenant" id="tenant"></a>

Tenant is a loose term that can refer to individual applications, different teams or departments, and even external customers. How we implement Namespaces and what we consider as tenants is up to us, but it’s most common to use Namespaces to divide clusters for use by tenants within the same organization. For example, we might divide a production cluster into the following three Namespaces to match the organizational structure:

* `finance`
* `hr`
* `corporate-ops`

We’d deploy Finance apps to the `finance` Namespace, HR apps to the `hr` Namespace, and Corporate apps to the `corporate-ops` Namespace. Each Namespace can have its own users, permissions, resource quotas, and policies.

Using Namespaces to divide a cluster among external tenants isn’t as common. This is because they only provide soft isolation and cannot prevent compromised workloads from escaping the Namespace and impacting workloads in other Namespaces. At the time of writing, the only way to strongly isolate tenants is to run them on their own clusters and their own hardware.

The following figure shows a cluster on the left using Namespaces for soft multi-tenancy. All apps on this cluster share the same nodes and control plane, and compromised workloads can impact both Namespaces. The two clusters on the right provide strong isolation by implementing two separate clusters, each on dedicated hardware.

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption><p>Soft and hard isolation</p></figcaption></figure>

Namespaces are lightweight and easy to manage but only provide soft isolation. Running multiple clusters costs more and introduces more management overhead, but it offers strong isolation.

***

## Default Namespaces <a href="#page-title" id="page-title"></a>

### Namespaces in clusters <a href="#namespaces-in-clusters" id="namespaces-in-clusters"></a>

Every Kubernetes cluster has a set of pre-created Namespaces. We can retrieve the names of all these Namespaces by executing the `kubectl get` command.

Run the following command to list all the namespaces available in our cluster.

```powershell
$ kubectl get namespaces
NAME             STATUS    AGE
default           Active   2d
kube-system       Active   2d
kube-public       Active   2d
kube-node-lease   Active   2d

// Get all namespaces
```

The `default` Namespace is where new objects go if we don’t specify a Namespace when creating them. `kube-system` is where control plane components such as the internal DNS service and the metrics server run. `kube-public` is for objects that need to be readable by anyone. And last but not the least, `kube-node-lease` is used for node heartbeat and managing node leases.

Run a `kubectl describe` to inspect one of the Namespaces on our cluster. We can substitute `namespace` with `ns` when working with `kubectl`.

```powershell
$ kubectl describe ns default
Name:         default
Labels:       kubernetes.io/metadata.name=default
Annotations:  <none>
Status:       Active
No resource quota.
No LimitRange resource.

// Get the attributes of a Pod
```

We can also add `-n` or `--namespace` to `kubectl` commands to filter results against a specific Namespace.

Run the following command to list all Service objects in the `kube-system` Namespace. The output can vary.

```powershell
$ kubectl get svc --namespace kube-system
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                 
kube-dns             ClusterIP      10.43.0.10     <none>        53/UDP,53/TCP,9153...   
metrics-server       ClusterIP      10.43.4.203    <none>        443/TCP                 
traefik-prometheus   ClusterIP      10.43.49.213   <none>        9100/TCP                
traefik              LoadBalancer   10.43.222.75   <pending>     80:31716/TCP,443:31...  

// Get all service objects in the kube-system namespace
```

We can also use the `--all-namespaces` flag to return objects from all Namespaces.

---
icon: kubernetes
---

# Workload Isolation

This section will show us some ways we can use to isolate workloads.

We’ll start at the cluster level, switch to the runtime level, and then look outside the cluster at infrastructure such as network firewalls.

### Cluster-level workload isolation <a href="#cluster-level-workload-isolation" id="cluster-level-workload-isolation"></a>

Cutting straight to the chase, Kubernetes does not support secure multi-tenant clusters. The only way to isolate two workloads is to run them on their own clusters with their own hardware.

Let’s look a bit closer.

The only way to divide a Kubernetes cluster is by creating _Namespaces_. However, these are little more than a way of grouping resources and applying things such as:

* Limits
* Quotas
* RBAC rules

Namespaces do not prevent compromised workloads in one Namespace from impacting workloads in other Namespaces. This means we should never run hostile workloads on the same Kubernetes cluster.

Despite this, Kubernetes Namespaces are useful, and we _should_ use them. Just don’t use them as security boundaries.

#### Namespaces and soft multi-tenancy <a href="#namespaces-and-soft-multi-tenancy" id="namespaces-and-soft-multi-tenancy"></a>

For our purposes, soft multi-tenancy is hosting multiple trusted workloads on shared infrastructure. By _trusted_, we mean workloads that don’t require absolute guarantees that one workload cannot impact another.

An example of a trusted workload might be an e-commerce application with a web front-end service and a back-end recommendation service. As they’re part of the same application, they’re not hostile. However, we might want each one to have its own resource limits managed by different teams.

In situations like this, a single cluster with a Namespace for the front-end service and another for the back-end service might be a good solution.

#### Namespaces and hard multi-tenancy <a href="#namespaces-and-hard-multi-tenancy" id="namespaces-and-hard-multi-tenancy"></a>

We’ll define hard multi-tenancy as hosting untrusted and potentially hostile workloads on shared infrastructure. However, as we said before, this isn’t _currently_ possible with Kubernetes.

This means workloads requiring a strong security boundary need to run on separate Kubernetes clusters! Examples include:

* Isolating production and non-production workloads
* Isolating different customers
* Isolating sensitive projects and business functions

Other examples exist, but the take-home point is that workloads requiring strong separation need their own clusters.

> **Note:** The Kubernetes project has a dedicated _Multi-Tenancy Working Group_ that’s actively working on multi-tenancy models. This means that future Kubernetes releases might have better solutions for hard multi-tenancy.

### Node isolation <a href="#node-isolation" id="node-isolation"></a>

There will be times when we have applications that require non-standard privileges, such as running as root or executing non-standard syscalls. Isolating these on their own clusters might be overkill, but we might justify running them on a ring-fenced subset of worker nodes. Doing this will restrict compromised workloads from only impacting other workloads on the same node.

We should also apply defense-in-depth principles by enabling stricter audit logging and tighter runtime defense options on nodes running workloads with non-standard privileges.

Kubernetes offers several technologies, such as labels, affinity and anti-affinity rules, and taints, to help us target workloads to specific nodes.

### Runtime isolation <a href="#runtime-isolation" id="runtime-isolation"></a>

Containers versus virtual machines used to be a polarizing topic. However, when it came to workload isolation, there is only one winner… virtual machines.

Most container platforms implement _namespaced containers_. In this model, every container shares the host’s kernel, and isolation is provided by kernel constructs, such as namespaces and cgroups, that were never designed as _strong_ security boundaries. Docker, containerd, and CRI-O are popular examples of container runtimes and platforms that implement namespaced containers.

This is very different from the hypervisor model, where every virtual machine gets its own dedicated kernel and is strongly isolated from other virtual machines using hardware enforcement.

However, it’s easier than ever to augment containers with security-related technologies that make them more secure and enable stronger workload isolation. These technologies include AppArmor, SELinux, seccomp, capabilities, user namespaces, and most container runtimes and hosted Kubernetes services do a good job of implementing sensible defaults for them all. However, they can still be complex, especially when troubleshooting.

We should also consider different classes of container runtimes. Two examples are _gVisor_ and _Kata Containers_, both of which provide stronger levels of workload isolation and are easy to integrate with Kubernetes thanks to the _Container Runtime Interface (CRI)_ and _Runtime classes_.

There are also projects that enable Kubernetes to orchestrate other workload types, such as virtual machines (VMs), serverless functions, and WebAssembly (Wasm).

While some of this might overwhelm us, we need to consider all of it when determining the isolation levels our workloads require.

To summarize, the following workload isolation options exist:

1. **Virtual machines:** Every workload gets its own dedicated kernel. It provides excellent isolation but is comparatively slow and resource-intensive.
2. **Namespaced containers:** All containers share the host’s kernel. These are fast and lightweight but require extra effort to improve workload isolation.
3. **Run every container in its own virtual machine:** Solutions like these attempt to combine the versatility of containers with the security of VMs by running every container in its own dedicated VM. Despite using specialized lightweight VMs, these solutions lose much of the appeal of containers, and they’re not very popular.
4. **Use different runtime classes:** This allows us to run all workloads as containers, but we target the workloads requiring stronger isolation to an appropriate container runtime.
5. **Wasm containers:** Wasm containers package Wasm apps in OCI containers that can execute on Kubernetes. These apps only use containers for packaging and scheduling, at runtime they execute inside a secure deny-by-default Wasm host. See the WebAssembly chapter for more details.

### Network isolation <a href="#network-isolation" id="network-isolation"></a>

Firewalls are an integral part of any layered information security system. The goal is to allow authorized communications.

In Kubernetes, Pods communicate over an internal network called the **pod network**. However, Kubernetes doesn’t implement the pod network. Instead, it implements a plugin model called the Container Network Interface (CNI) that allows third-party vendors to implement the pod network. Lots of CNI plugins exist, but they fall into two broad categories:

* Overlay
* BGP

Each has a different impact on firewall implementation and network security.

#### Kubernetes and overlay networking <a href="#kubernetes-and-overlay-networking" id="kubernetes-and-overlay-networking"></a>

Most Kubernetes environments implement the pod network as a simple flat overlay network that hides any network complexity between cluster nodes. For example, we might deploy our cluster nodes across 10 different networks connected by routers, but Pods connect to the flat pod network and communicate without needing to know any of the complexity of the host networking. The following figure shows four nodes on two separate networks and the Pods connected to a single overlay pod network.

<figure><img src="../.gitbook/assets/image (13).png" alt="" width="563"><figcaption><p>Pods connected to an overlay pod network</p></figcaption></figure>

Overlay networks use VXLAN technologies to encapsulate traffic for transmission over a simple flat Layer-2 network operating on top of existing Layer-3 infrastructure. If that’s too much network jargon, all we need to know is that overlay networks encapsulate packets sent by containers. This encapsulation hides the original source and target IP addresses, making it harder for firewalls to know what’s going on. See the below figure.

<figure><img src="../.gitbook/assets/image (14).png" alt="" width="563"><figcaption><p>Encapsulation on overlay network</p></figcaption></figure>

#### Kubernetes and BGP <a href="#kubernetes-and-bgp" id="kubernetes-and-bgp"></a>

BGP is the protocol that powers the internet. However, at its core, it’s a simple and scalable protocol that creates peer relationships that are used to share routes and perform routing.

The following analogy might help. Imagine you want to send a birthday card to a friend, but you've lost their contact and don't have their address. However, your child has a friend at school whose parents are still in touch with your old friend. In this situation, you give the card to your child and ask them to give it to their friend at school. This friend gives it to their parents, who deliver it to your friend.

BGP routing is similar and happens through a network of peers that help each other find routes.

From a security perspective, the important thing is that BGP doesn’t encapsulate packets. This makes things much simpler for firewalls. The following figure shows the same setup using BGP. Notice how there’s no encapsulation.

<figure><img src="../.gitbook/assets/image (15).png" alt="" width="563"><figcaption><p>No encapsulation on overlay network</p></figcaption></figure>

#### How this impacts firewalls <a href="#how-this-impacts-firewalls" id="how-this-impacts-firewalls"></a>

We’ve already said that firewalls allow or disallow traffic flow based on source and destination addresses. For example:

* Allow traffic from the `10.0.0.0/24` network.
* Disallow traffic from the `192.168.0.0/24` network.

Suppose our pod network is an overlay network. In this case, all traffic will be encapsulated, and only firewalls that can open packets and inspect their contents will be able to make useful decisions on whether to allow or deny traffic. We may want to consider a BGP pod network if our firewalls can’t do this.

We should also consider whether to deploy physical firewalls, host-based firewalls, or a combination of both.

Physical firewalls are dedicated network hardware devices that are usually managed by a central team. Host-based firewalls are operating system (OS) features and are usually managed by the team that deploys and manages our Operating Systems. Both solutions have pros and cons, and combining the two is often the most secure. However, we should consider whether our organization has a long and complex procedure for implementing changes to physical firewalls. If it does, it might not suit the nature of our Kubernetes environment.

#### Packet capture <a href="#packet-capture" id="packet-capture"></a>

Regarding networking and IP addresses, not only are Pod IP addresses sometimes obscured by encapsulation, but they are also dynamic and can be recycled and reused by different Pods. We call this IP churn, and it reduces how useful IP addresses are at identifying systems and workloads. With this in mind, the ability to associate IP addresses with Kubernetes-specific identifiers such as Pod IDs, Service aliases, and container IDs when performing things like packet capturing can be extremely useful.

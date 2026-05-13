---
icon: kubernetes
---

# How Do We Deploy Pods?

### static Pods vs. controllers <a href="#static-pods-vs-controllers" id="static-pods-vs-controllers"></a>

There are two ways to deploy Pods:

1. Directly via a Pod manifest (rare)
2. Indirectly via a workload resource and controller (most common)

Deploying directly from a Pod manifest creates a static Pod that cannot self-heal, scale, or perform rolling updates. This is because they’re only managed by the `kubelet` on the node they’re running on, and `kubelets` are limited to restarting containers on the same node. If the node fails, the `kubelet` also fails and cannot do anything to help the Pod.

On the flip side, Pods deployed via workload resources get all the benefits of being managed by a highly available controller that can restart them on other nodes, scale them when demand changes, and perform advanced operations such as rolling updates and versioned rollbacks. The local `kubelet` can still attempt to restart failed containers, but if the node fails or gets evicted, the controller can restart it on a different node.

Remember, when we say restart the Pod, we mean to replace it with a new one.

### The Pod network <a href="#the-pod-network" id="the-pod-network"></a>

Every Kubernetes cluster runs a Pod network and automatically connects all Pods to it. It’s usually a flat Layer-2 overlay network that spans every cluster node and allows every Pod to talk directly to every other Pod, even if the remote Pod is on a different cluster node.

The Pod network is implemented by a third-party plugin that interfaces with Kubernetes and configures the network via the Container Network Interface (CNI).

We choose a network plugin at cluster build time, and it configures the Pod network for the entire cluster. Lots of plugins exist, and each one has its pros and cons. However, at the time of writing, [Cilium](https://cilium.io/) is the most popular and implements many advanced features such as security and observability.

Figure 4.4 shows three nodes running five Pods. All five Pods are connected to the Pod network and can communicate with each other. We can also see the Pod network spanning all three nodes. However, the network is only for Pods and not nodes. As shown in the diagram, we can connect nodes to multiple networks, but the Pod network spans them all.

<figure><img src="../.gitbook/assets/image (20).png" alt="" width="563"><figcaption><p>The Pod network</p></figcaption></figure>

A lot of clusters create a very open Pod network with little or no security. This makes the cluster easy to use and avoids frustrations commonly associated with network security. However, we should use Kubernetes Network Policies and other measures to secure it.

## Multi-Container Pods: Init Containers <a href="#page-title" id="page-title"></a>

**Init containers** are a special type of container defined in the Kubernetes API. We run them in the same Pod as application containers, but Kubernetes guarantees they’ll start and complete _before_ the main app container starts. It also guarantees they’ll only run once.

The purpose of init containers is to prepare and initialize the environment so it’s ready for application containers.

Consider a couple of quick examples.

We have an application that should only start when a remote API is accepting connections. Instead of complicating the main application with the logic to check the remote API, we run that logic in an init container in the same Pod. When we deploy the Pod, the init container comes up first and sends requests to the remote API waiting for it to respond. While this is happening, the main app container cannot start. However, as soon as the remote API accepts a request, the init container completes, and the main app container will start.

Assume we have another application that needs a one-time clone of a remote repository before starting. Again, instead of bloating and complicating the main application with the code to clone and prepare the content (knowledge of the remote server address, certificates, auth, file sync protocol, checksum verifications, etc.), we implement that in an init container that is guaranteed to complete the task before the main application container starts.

A drawback of init containers is that they’re limited to running tasks before the main app container starts. For something that runs alongside the main app container, we need a sidecar container.

### Multi-container Pods: Sidecars <a href="#multi-container-pods-sidecars" id="multi-container-pods-sidecars"></a>

**Sidecar containers** are regular containers that run at the same time as application containers for the entire lifecycle of the Pod.

Unlike init containers, sidecars are not a resource in the Kubernetes API — we’re currently using regular containers to hack the sidecar pattern. Work is in progress to formalize the sidecar pattern in the API, but at the time of writing, it’s still an early alpha feature.

A sidecar container add functionality to an app without having to implement it in the actual app. Common examples include sidecars that scrape logs, sync remote content, broker connections, and munge data. They’re also heavily used by services meshes where the sidecar intercepts network traffic and provides traffic encryption and telemetry.

<figure><img src="../.gitbook/assets/image (22).png" alt="" width="435"><figcaption><p>Service mesh sidecar</p></figcaption></figure>

The above figure shows a multi-container Pod with a main app container and a service mesh sidecar. The sidecar intercepts all network traffic, provides encryption and decryption, and sends telemetry data to the service mesh control plane.

---
icon: kubernetes
---

# Kubernetes from 40K Feet

Kubernetes is both of the following:

* A cluster
* An orchestrator

{% stepper %}
{% step %}
### Kubernetes: Cluster

A Kubernetes cluster is one or more _nodes_ providing CPU, memory, and other resources for use by applications.

Kubernetes supports two node types:

* Control plane nodes
* Worker nodes

Both types can be physical servers, virtual machines, or cloud instances, and both can run on ARM and AMD64/x86-64. Control plane nodes must be Linux, but worker nodes can be Linux or Windows.

Control plane nodes implement the Kubernetes intelligence, and every cluster needs at least one. However, we should have three or five for high availability (HA).

Every control plane node runs every control plane service. These include the API server, the scheduler, and the controllers that implement cloud-native features such as self-healing, auto-scaling, and rollouts.

Worker nodes are for running user applications.

<figure><img src="../.gitbook/assets/image (3) (1).png" alt=""><figcaption><p>A cluster with three control plane nodes and three worker nodes</p></figcaption></figure>

It’s common to run user applications on control plane nodes in development and test environments. However, many production environments restrict user applications to worker nodes so that control plane nodes can focus entirely on cluster operations. Doing this allows control plane nodes to focus on managing the cluster.
{% endstep %}

{% step %}
### Kubernetes: Orchestrator

rchestrator is jargon for a system that deploys and manages applications.

Kubernetes is the industry standard orchestrator and can intelligently deploy applications across nodes and failure zones for optimal performance and availability. It can also fix them when they break, scale them when demand changes, and manage zero-downtime rolling updates.

That’s the big picture. Let’s dig a bit deeper.
{% endstep %}
{% endstepper %}

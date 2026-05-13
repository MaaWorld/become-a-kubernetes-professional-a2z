---
icon: kubernetes
---

# Control Plane and Worker Nodes

### The control plane <a href="#the-control-plane" id="the-control-plane"></a>

The control plane is a collection of system services that implement the brains of Kubernetes. It exposes the API, schedules tasks, implements self-healing, manages scaling operations, and more.

The simplest setups run a single control plane node and are best suited for labs and testing. However, as previously mentioned, we should run three or five control plane nodes in production environments and spread them across availability zones for high availability (HA).

<figure><img src="../.gitbook/assets/image (4).png" alt="" width="550"><figcaption><p>High availability control plane</p></figcaption></figure>

{% hint style="info" %}
**Note:** It’s sometimes considered a production best practice to run all user apps on worker nodes, allowing control plane nodes to allocate all resources to cluster-related operations.
{% endhint %}

Most clusters run every control plane service on every control plane node for HA.

Let’s take a look at the services that make up the control plane.

#### The API server <a href="#the-api-server" id="the-api-server"></a>

The API server is the front end of Kubernetes, and all requests to change and query the state of the cluster go through it. Even internal control plane services communicate with each other via the API server.

It exposes a RESTful API over HTTPS; all requests are subject to authentication and authorization. For example, deploying or updating an app follows this process:

1. Describe the requirements in a YAML configuration file.
2. Post the configuration file to the API server.
3. The request will be authenticated and authorized.
4. The updates will be persisted in the cluster store.
5. The updates will be scheduled to the cluster.

#### The cluster store <a href="#the-cluster-store" id="the-cluster-store"></a>

The cluster store holds the desired state of all applications and cluster components and is the only stateful part of the control plane.

It’s based on the etcd distributed database, and most Kubernetes clusters run an etcd replica on every control plane node for HA. However, large clusters that experience a high rate of change may run a separate etcd cluster for better performance.

Be aware that a highly available cluster store is not a substitute for backup and recovery. We still need adequate ways to recover the cluster store when things go wrong.

Regarding availability, etcd prefers an odd number of replicas to help avoid split-brain conditions. This is where replicas experience communication issues and cannot be sure if they have a quorum (majority).

The following figure shows two etcd configurations experiencing a network partition. The cluster on the left has four nodes and is experiencing a split-brain with two nodes on either side and neither having a majority. The cluster on the right only has three nodes but is not experiencing a split-brain as Node A knows it does not have a majority, whereas Node B and Node C know they do.

<figure><img src="../.gitbook/assets/image (5).png" alt="" width="563"><figcaption><p>HA and split-brain conditions</p></figcaption></figure>

If a split-brain scenario occurs, etcd goes into read-only mode preventing updates to the cluster. User applications will continue working, we just won’t be able to make cluster updates, such as adding or modifying apps and services.

As with all distributed databases, consistency of writes is vital. For example, multiple writes to the same value from different sources need to be handled. etcd uses the Raft consensus algorithm for this.

#### Controllers and the controller manager <a href="#controllers-and-the-controller-manager" id="controllers-and-the-controller-manager"></a>

Kubernetes uses controllers to implement a lot of the cluster intelligence. They all run on the control plane, and some of the more common ones include:

* The Deployment controller
* The StatefulSet controller
* The ReplicaSet controller

Others exist, and we’ll cover some of them later in the book. However, they all run as background watch loops, reconciling the observed state with the desired state.

That’s a lot of jargon; we’ll cover it in detail later in the chapter. But for now, it means controllers ensure the cluster runs what we ask it to run. For example, if we ask for three replicas of an app, a controller will ensure three healthy replicas are running and take appropriate actions if they aren’t.

Kubernetes also runs a controller manager that is responsible for spawning and managing the individual controllers.

<figure><img src="../.gitbook/assets/image (6).png" alt="" width="421"><figcaption><p>A high-level overview of the controller manager and controllers.</p></figcaption></figure>

#### The scheduler <a href="#the-scheduler" id="the-scheduler"></a>

The scheduler watches the API server for new work tasks and assigns them to healthy worker nodes.

It implements the following process:

1. Watch the API server for new tasks.
2. Identify capable nodes.
3. Assign tasks to nodes.

Identifying capable nodes involves predicate checks, filtering, and a ranking algorithm. It checks for taints, affinity and anti-affinity rules, network port availability, and available CPU and memory. It ignores nodes incapable of running the tasks and ranks the remaining ones according to factors such as whether they already have the required image, available CPU and memory, and the number of tasks they’re currently running. Each is worth points, and the nodes with the most points are selected to run the tasks.

The scheduler marks tasks as pending if it can’t find a suitable node.

If the cluster is configured for node autoscaling, the pending task kicks off a cluster autoscaling event that adds a new node and schedules the task for that node.

#### The cloud controller manager <a href="#the-cloud-controller-manager" id="the-cloud-controller-manager"></a>

If our cluster is on a public cloud, such as AWS, Azure, GCP, or Civo, it will run a cloud controller manager that integrates the cluster with cloud services, such as instances, load balancers, and storage. For example, if we’re on a cloud and an application requests a load balancer, the cloud controller manager provisions one of the cloud’s load balancers and connects it to our app.

#### Control plane summary <a href="#control-plane-summary" id="control-plane-summary"></a>

The control plane implements the brains of Kubernetes, including the API Server, the scheduler, and the cluster store. It also implements controllers that ensure the cluster runs what we ask it to run.

<figure><img src="../.gitbook/assets/image (7).png" alt="" width="353"><figcaption><p>A high-level view of a Kubernetes control plane node.</p></figcaption></figure>

We should run three or five control plane nodes for high availability, and large busy clusters might run a separate etcd cluster for better cluster store performance.

The API server is the Kubernetes frontend, and all communication passes through it.

***

### Worker Nodes <a href="#worker-nodes" id="worker-nodes"></a>

Worker nodes are for running user applications and look like the following figure:

<figure><img src="../.gitbook/assets/image (8).png" alt="" width="466"><figcaption><p>Worker nodes</p></figcaption></figure>

Let’s look at the major worker node components.

#### Kubelet <a href="#kubelet" id="kubelet"></a>

The kubelet is the main Kubernetes agent and handles all communication with the cluster.

It performs the following key tasks:

* Watches the API server for new tasks
* Instructs the appropriate runtime to execute tasks
* Reports the status of tasks to the API server

If a task won’t run, the kubelet reports the problem to the API server and lets the control plane decide what actions to take.

#### Runtime <a href="#runtime" id="runtime"></a>

Every worker node has one or more runtimes for executing tasks.

Most new Kubernetes clusters preinstall the containerd runtime and use it to execute tasks. These tasks include:

* Pulling container images
* Managing lifecycle operations such as starting and stopping containers

Older clusters shipped with the Docker runtime, but this is no longer supported. Red Hat OpenShift clusters use the CRI-O runtime. Lots of others exist, and each has its pros and cons.

We’ll use different runtimes in a later chapter.

#### Kube-proxy <a href="#kube-proxy" id="kube-proxy"></a>

The last piece of the node puzzle is the kube-proxy. Every worker node runs a kube-proxy service that implements cluster networking and load balances traffic to tasks running on the node. For example, it makes sure each node gets its own unique IP address and implements local iptables or ipvs rules to handle routing and load balancing of traffic on the Pod network.<br>

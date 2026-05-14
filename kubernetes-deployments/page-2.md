---
icon: kubernetes
---

# Kubernetes Deployment

### Introduction to Deployments <a href="#introduction-to-deployments" id="introduction-to-deployments"></a>

Deployments are the most popular way of running stateless apps on Kubernetes. They add self-healing, scaling, rollouts, and rollbacks.

Consider a quick example.

Assume we have a requirement for a web app that needs to be resilient, scale on demand, and be frequently updated. We write the app, containerize it, and define it in a Pod YAML so it can run on Kubernetes. We then wrap the Pod inside a Deployment and post it to Kubernetes, where the Deployment controller deploys the Pod. At this point, our cluster is running a single Deployment managing a single Pod.

If the Pod fails, the Deployment controller replaces it with a new one. If demand increases, the Deployment controller can deploy more identical Pods. When we update the app, the Deployment controller deletes the old Pods and replaces them with new ones.

Assume the app has another stateless microservice, such as a shopping cart. We’d containerize this, wrap it in its own Pod, wrap the Pod in its own Deployment, and deploy it to the cluster.

At this point, we’d have two Deployments managing two different microservices.

The following figure shows this setup with the Deployment controller watching and managing both Deployments. The web Deployment manages four identical web server Pods, and the cart Deployment manages two identical shopping cart Pods.

<figure><img src="../.gitbook/assets/image (24).png" alt="" width="349"><figcaption><p>Deployments</p></figcaption></figure>

Under the hood, Deployments follow standard Kubernetes architecture comprising of:

1. A resource
2. A controller

At the highest level, resources define objects and controllers manage them.

The Deployment resource exists in the `apps/v1` [API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#deployment-v1-apps) and defines all supported attributes and capabilities. The Deployment controller runs on the control plane, watches Deployments, and reconciles the observed state with the desired state.

### Deployments and Pods <a href="#deployments-and-pods" id="deployments-and-pods"></a>

Every Deployment manages one or more identical Pods.

For example, an application comprising a web service and a shopping cart service will need two Deployments — one for managing the web Pods and the other for managing the shopping cart Pods. Figure 6.1 showed the web Deployment managing four identical web Pods and the cart Deployment managing two identical shopping cart Pods.

The following figure shows a Deployment YAML file requesting four replicas of a single Pod. If we increase the replica count to six, it will deploy and manage two additional identical Pods.

<figure><img src="../.gitbook/assets/image (25).png" alt="" width="371"><figcaption><p>Requesting replicas of a Pod using a deployment file</p></figcaption></figure>

Notice how the Pod spec is defined in a template embedded in the Deployment YAML.

***

## ReplicaSets and Scaling <a href="#page-title" id="page-title"></a>

Learn how Deployments offer self-healing and scalability.

### Deployments and ReplicaSets <a href="#deployments-and-replicasets" id="deployments-and-replicasets"></a>

We’ve repeatedly said that Deployments add self-healing, scaling, rollouts, and rollbacks. However, behind the scenes, it’s actually a different resource called a **ReplicaSet** that provides self-healing and scaling.

The following figure shows the overall architecture of containers, Pods, ReplicaSets, and Deployments. It also shows how they map into a Deployment YAML.

<figure><img src="../.gitbook/assets/image (26).png" alt="" width="563"><figcaption><p>Architecture</p></figcaption></figure>

Posting this Deployment YAML to the cluster will create a Deployment, a ReplicaSet, and two identical Pods running identical containers. The Pods are managed by the ReplicaSet, which, in turn, is managed by the Deployment. We should then perform all management via the Deployment and never directly manage the ReplicaSet or Pods.

### A quick word on scaling <a href="#a-quick-word-on-scaling" id="a-quick-word-on-scaling"></a>

It’s possible to scale our apps manually, and we’ll see how to do that shortly. However, Kubernetes has several autoscalers that automatically scale our apps and infrastructure. Some of them include:

* The Horizontal Pod Autoscaler
* The Vertical Pod Autoscaler
* The Cluster Autoscaler

The **Horizontal Pod Autoscaler** (HPA) adds and removes Pods to meet current demand. Most clusters install it by default, and it’s widely used.

The **Cluster Autoscaler (CA)** adds and removes cluster nodes so we always have enough to run all scheduled Pods. This is also installed by default and widely used.

The **Vertical Pod Autoscaler (VPA)** increases and decreases the CPU and memory allocated to running Pods to meet current demand. It isn’t installed by default, has several known limitations, and is less widely used. Current implementations delete the existing Pod and replace it with a new one every time they scale the Pods resources. This is disruptive and can even result in Kubernetes scheduling the new Pod to a different node. However, work is underway to enable in place updates to live Pods.

Community projects like Karmada take things further by allowing us to scale apps across multiple clusters.

Let’s consider a quick example using the HPA and CA.

We deploy an application to our cluster and configure an HPA to autoscale the number of application Pods between two and 10. Demand increases, and the HPA asks the scheduler to increase the number of Pods from two to four. This works, but demand continues to rise, and the HPA asks the scheduler for another two Pods. However, the scheduler can’t find a node with sufficient resources this time and marks the two new Pods as _pending_. The CA notices the pending Pods and dynamically adds a new cluster node. Once the node joins the cluster, the scheduler assigns the pending Pods to it.

The process works the same for scaling down. For example, the HPA reduces the number of Pods when demand decreases. This may trigger the CA to reduce the number of cluster nodes. When removing a cluster node, Kubernetes has to evict all Pods on the node and replace them with new Pods on the surviving nodes.

We’ll sometimes hear people refer to multi-dimensional autoscaling. This is jargon for combining multiple scaling methods — scaling Pods and nodes, or scaling apps horizontally (adding more Pods) and vertically (adding more resources to existing Pods).

### It’s all about the state <a href="#its-all-about-the-state" id="its-all-about-the-state"></a>

Before going any further, it’s vital that you understand the following concepts. If you already know them, skip straight to the next lesson.

* Desired state
* Observed state (sometimes called actual state or current state)
* Reconciliation

Desired state is what you want, observed state is what you have, and the goal is for them to always match. When they don’t match, a controller starts a reconciliation process to bring observed state back into sync with desired state.

The declarative model is how we declare a desired state to Kubernetes without telling Kubernetes how to implement it. We leave the how up to Kubernetes.

#### Declarative vs. Imperative <a href="#declarative-vs-imperative" id="declarative-vs-imperative"></a>

The declarative model describes an end goal — we tell Kubernetes what we want. The imperative model requires long lists of commands that tell Kubernetes how to reach the end goal.

The following analogy will help:

* Declarative: Give me a chocolate cake to feed 10 people.
* Imperative: Drive to store. Buy eggs, milk, flour, cocoa powder… Drive home. Preheat the oven. Mix the ingredients. Place the mixture in a cake tin. In case of a fan-assisted oven, place the cake in the oven for 30 minutes. If using a conventional oven, place the cake in the oven for 40 minutes. Set a timer. Remove the cake from the oven when the timer expires and turn the oven off. Leave to stand until cool. Add frosting.

The declarative model is simple and leaves the how up to Kubernetes. The imperative model is much more complex as we need to provide all the steps and commands that will hopefully achieve an end goal — in this case, making a chocolate cake for 10 people.

Let’s look at a more concrete example.

Assume we have an application with two microservices — a front-end and a back-end. We anticipate needing five front-end replicas and two back-end replicas.

Taking the declarative approach, we write a simple YAML file requesting five front-end Pods listening externally on port `80`, and two back-end Pods listening internally on port `27017`. We then give the file to Kubernetes and sit back while Kubernetes makes it happen. It’s a beautiful thing!

The opposite is the imperative model. This is usually a long list of complex instructions with no concept of desired state. And, making things worse, imperative instructions can have endless potential variations. For example, the commands to pull and start containerd containers are different from the commands to pull and start CRI-O containers. This results in more work and is prone to more errors, and because it’s not declaring a desired state, there’s no self-healing. It’s devastatingly ugly.

Kubernetes supports both models but strongly prefers the declarative model.

> **Note:** containerd and CRI-O are CRI runtimes that run on Kubernetes workers and perform low-level tasks such as starting and stopping containers.

#### Controllers and reconciliation <a href="#controllers-and-reconciliation" id="controllers-and-reconciliation"></a>

Reconciliation is fundamental to desired state.

For example, ReplicaSets are implemented as a background controller running in a reconciliation loop, ensuring the correct number of Pod replicas are always present. If there aren’t enough Pods, it adds more. If there are too many, it terminates some.

Assume a scenario where desired state is 10 replicas, but only eight are present. It makes no difference if this is due to failures or if an autoscaler has requested an increase. Either way, the ReplicaSet controller creates two new replicas to sync the observed state with the desired state. And the best bit is that it does it without needing help from us!

The exact same reconciliation process enables self-healing, scaling, rollouts, and rollbacks.

***

## Rolling Updates with Deployments <a href="#page-title" id="page-title"></a>

Learn how to perform rolling updates with Deployments.

### Rolling updates <a href="#rolling-updates" id="rolling-updates"></a>

Deployments are amazing at zero-downtime rolling updates (rollouts). But they work best if we design our apps to be:

1. Loosely coupled via APIs
2. Backward and forward-compatible

Both are hallmarks of modern cloud-native microservices apps and work as follows.

Our microservices should always be loosely coupled and only communicate via well-defined APIs. Doing this means we can update and patch any microservice without having to worry about impacting others — all connections are via formalized APIs that expose documented interfaces and hide specifics.

Ensuring releases are backward and forward-compatible means we can perform independent updates without caring which versions of clients are consuming the service. A simple non-tech analogy is a car. Cars expose a standard driving API that includes a steering wheel and foot pedals. As long as we don’t change this API, you can re-map the engine, change the exhaust, and get bigger brakes, all without the driver having to learn any new skills.

With these points in mind, zero-downtime rollouts work like this.

Assume we’re running five replicas of a stateless microservice. Clients can connect to any of the five replicas as long as all clients connect via backward and forward-compatible APIs. To perform a rollout, Kubernetes creates a new replica running the new version and terminates one running the old version. At this point, we’ve got four replicas on the old version and one on the new. This process repeats until all five replicas are on the new version. As the app is stateless and multiple replicas are up and running, clients experience no downtime or interruption of service.

There’s a lot more going on behind the scenes, so let’s take a closer look.

Each microservice is built as a container and wrapped in a Pod. We then wrap each Pod in its own Deployment for self-healing, scaling, and rolling updates. Each Deployment describes the following:

* Number of Pod replicas
* Container images to use
* Network ports
* How to perform rolling updates

We post Deployment YAML files to the API Server, and the ReplicaSet controller ensures the correct number of Pods get scheduled. It also watches the cluster, ensuring the observed state matches the desired state. A Deployment sits above the ReplicaSet, governing its configuration and adding mechanisms for rollouts and rollbacks.

All good so far.

Now, assume we’re exposed to a known vulnerability and need to release an update with the fix. To do this, we update the same Deployment YAML file with the new Pod spec and re-post it to the API server. This updates the existing Deployment object with a new desired state requesting the same number of Pods, but all running the newer version containing the fix.

At this point, the observed state no longer matches the desired state — we’ve got five old Pods, but we want five new ones.

To reconcile, the Deployment controller creates a new ReplicaSet defining the same number of Pods but running the newer version. We now have two ReplicaSets — the original one for the Pods with the old version and the new one for the Pods with the new version. The Deployment controller systematically increments the number of Pods in the new ReplicaSet as it decrements the number in the old ReplicaSet. The net result is a smooth incremental rollout with zero-downtime.

The same process happens for future updates — we keep updating the same Deployment manifest, which we should store in a version control system.

The following figure shows a Deployment that’s been updated once. The initial release created the ReplicaSet on the left, and the update created the one on the right. The update has completed, as the ReplicaSet on the left is no longer managing any Pods, whereas the one on the right is managing three live Pods.

<figure><img src="../.gitbook/assets/image (27).png" alt="" width="525"><figcaption><p>An updated deployment</p></figcaption></figure>

### Rollbacks <a href="#rollbacks" id="rollbacks"></a>

As we saw in Figure 6.4, older ReplicaSets are wound down and no longer manage any Pods. However, their configurations still exist and can be used to easily roll back to previous versions.

The rollback process is the opposite of a rollout — wind an old ReplicaSet up while the current one winds down.

The following figure shows the same app rolled back to the previous config and being managed by the previous ReplicaSet.

<figure><img src="../.gitbook/assets/image (28).png" alt="" width="476"><figcaption><p>A rolled back app managed by a ReplicaSet</p></figcaption></figure>

But that’s not the end. Kubernetes gives us fine-grained control over rollouts and rollbacks. For example, we can insert delays, control the pace and cadence of releases, and even probe the health and status of updated replicas

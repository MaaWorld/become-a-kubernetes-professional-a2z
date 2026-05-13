---
icon: kubernetes
---

# Pods and Deployments

## Pods and containers <a href="#pods-and-containers" id="pods-and-containers"></a>

The simplest configurations run a single container per Pod, which is why we sometimes use the terms Pod and container interchangeably. However, there are powerful use cases for multi-container Pods, including:

* Service meshes
* Helper services that initialize app environments
* Apps with tightly coupled helper functions such as log scrapers

The following figure shows a multi-container Pod with a main application container and a service mesh sidecar. Sidecar is jargon for a helper container that runs in the same Pod as the main app container and provides services to it. In this figure, the service mesh sidecar encrypts network traffic coming in and out of the main app container and provides telemetry.

<figure><img src="../.gitbook/assets/image (9).png" alt="" width="448"><figcaption><p>Multi-container service mesh Pod</p></figcaption></figure>

Multi-container Pods also help us implement the single responsibility principle where every container performs a single simple task. In the above figure, the main app container might be serving a message queue or some other core application feature. Instead of adding the encryption and telemetry logic into the main app, we keep the app simple and implement it in the service mesh container running alongside it in the same Pod.

### Pod anatomy <a href="#pod-anatomy" id="pod-anatomy"></a>

Each Pod is a shared execution environment for one or more containers. The execution environment includes a network stack, volumes, shared memory, and more.

Containers in a single container Pod have the execution environment to themselves, whereas containers in a multi-container Pod share it.

For example, the following figure shows a multi-container Pod with both containers sharing the Pods IP address. The main application container is accessible outside the Pod on `10.0.10.15:8080`, and the sidecar is accessible on `10.0.10.15:5005`. If they need to communicate with each other, container-to-container within the Pod, they can use the Pod’s `localhost` interface.

<figure><img src="../.gitbook/assets/image (10).png" alt="" width="322"><figcaption><p>Multi-container Pod sharing Pod IP</p></figcaption></figure>

We should choose a multi-container Pod when our application has tightly coupled components needing to share resources such as memory or storage. In most other cases, we should use single container Pods and loosely couple them over the network.

### Pod scheduling <a href="#pod-scheduling" id="pod-scheduling"></a>

All containers in a Pod are always scheduled to the same node. This is because Pods are a shared execution environment, and we can’t easily share memory, networking, and volumes across nodes.

Starting a Pod is also an atomic operatio&#x6E;_._ This means Kubernetes only ever marks a Pod as running when all its containers are started. For example, if a Pod has two containers and only one is started, the Pod is not ready.

### Pods as the unit of scaling <a href="#pods-as-the-unit-of-scaling" id="pods-as-the-unit-of-scaling"></a>

Pods are the minimum unit of scheduling in Kubernetes. As such, scaling an application up adds more Pods, and scaling it down deletes Pods. We _do not_ scale by adding more containers to existing Pods. The following figure shows how to scale the web-fe microservice using Pods as the unit of scaling.

<figure><img src="../.gitbook/assets/image (11).png" alt="" width="563"><figcaption><p>Scaling with Pods</p></figcaption></figure>

### Pod lifecycle <a href="#pod-lifecycle" id="pod-lifecycle"></a>

Pods are mortal — they’re created, they live, and they die. Anytime one dies, Kubernetes replaces it with a new one. Even though the new one looks, smells, and feels the same as the old one, it’s always a shiny new one with a new ID and new IP.

This forces us to design applications to be loosely coupled and immune to individual Pod failures.

### Pod immutability <a href="#pod-immutability" id="pod-immutability"></a>

Pods are immutable. This means we never change them once they’re running.

For example, if we need to change or update a Pod, we should always replace it with a new one running the updates. We should never log on to a Pod and change it. This means anytime we talk about 'updating Pods', we always mean deleting the old one and replacing it with a new one. This can be a huge mindset change for some of us, but it fits nicely with modern tools and GitOps-style workflows.

***

## Deployments <a href="#page-title" id="page-title"></a>

Kubernetes works with Pods, you’ll almost always deploy them via high-level controllers such as Deployments, StatefulSets, and DaemonSets. These all run on the control plane and operate as background watch loops, reconciling the observed state with the desired state.

Deployments add self-healing, scaling, rolling updates, and versioned rollbacks to stateless apps.

### Service objects and stable networking <a href="#service-objects-and-stable-networking" id="service-objects-and-stable-networking"></a>

Earlier in the chapter, we said that Pods are mortal and can die. However, if they’re managed by a controller, they get replaced by new Pods with new IDs and new IP addresses. The same thing happens with rollouts and scaling operations:

* Rollouts replace old Pods with new ones and new IPs.
* Scaling up adds new Pods with new IPs.
* Scaling down deletes existing Pods.

Events like these generate IP churn and make Pods unreliable. For example, clients cannot make reliable connections to individual Pods as the Pods are not guaranteed to be there.

This is where Kubernetes Services come into play by providing reliable networking for groups of Pods.

The following figure shows internal and external clients connecting to a group of Pods via a Kubernetes Service. The Service (capital 'S' because it’s a Kubernetes API object) provides a reliable name and IP and load balances requests to the Pods behind it.

<figure><img src="../.gitbook/assets/image (12).png" alt="" width="466"><figcaption><p>Internal and external clients connecting to a group of Pods</p></figcaption></figure>

You should think of Services as having a frontend and a backend. The frontend has a DNS name, IP address, and network port. The backend uses labels to load balance traffic across a dynamic set of Pods.

Services keep a list of healthy Pods as scaling events, rollouts, and failures cause Pods to come and go. This means they’ll always load balance traffic across active healthy Pods. The Service also guarantees the name, IP, and port on the frontend will never change.

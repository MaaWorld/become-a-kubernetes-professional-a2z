---
icon: kubernetes
---

# The Operating System of the Cloud

Explore the concept of Kubernetes as the cloud operating system that abstracts complex cloud resources and schedules application microservices. Understand its role in simplifying deployment across hybrid and multicloud environments, and how it enables seamless cloud migrations by managing resources transparently.

### Kubernetes - what’s in the name? <a href="#kuberneteswhats-in-the-name" id="kuberneteswhats-in-the-name"></a>

Most people pronounce Kubernetes as _koo-ber-net-eez_, but the community is very friendly, and people won’t mind if we pronounce it differently.

Kubernetes comes from the Greek word for _helmsman_, or the person who steers a ship. The logo, a ship’s wheel, reflects this.

<figure><img src="../.gitbook/assets/image (1).png" alt="" width="188"><figcaption></figcaption></figure>

&#x20;Some original engineers wanted to call Kubernetes _Seven of Nine_ after the famous Borg drone from the TV series Star Trek Voyager. Copyright laws wouldn’t allow this, so they gave the logo _seven_ spokes as a subtle reference to _Seven of Nine_.

We’ll often see it shortened to _K8s_ and pronounced as _Kates._ 8 replaces the eight characters between the 'K' and the 's.'

### Operating system <a href="#operating-system" id="operating-system"></a>

Kubernetes is the de facto platform for cloud-native applications, and we sometimes call it _the cloud's operating system (OS)._ This is because Kubernetes abstracts the differences between cloud platforms much like operating systems like Linux and Windows abstract the differences between servers:

* Linux and Windows abstract server resources and schedule application processes.
* Kubernetes abstracts cloud resources and schedules application microservices.

As a quick example, we can schedule applications to Kubernetes without caring if it’s running on AWS, Azure, Civo, Google Cloud Platform (GCP), or on-premises data center. This makes Kubernetes a key enabler for:

* Hybrid cloud
* Multicloud
* Cloud migrations

In summary, Kubernetes makes deploying to one cloud today easier and migrating to another cloud tomorrow.

### Application scheduling <a href="#application-scheduling" id="application-scheduling"></a>

One of the main things an OS does is simplify the scheduling of work tasks.

Computers are complex collections of hardware resources such as CPU, memory, storage, and networking. Thankfully, modern operating systems hide most of this and make the world of application development a far friendlier place. For example, how many developers need to care which CPU core, memory DIMM, or flash chip their code uses? Most of the time, we leave it up to the OS.

Kubernetes does a similar thing with clouds and data centers.

At a high level, a cloud or data center is a complex collection of resources and services. Kubernetes can abstract a lot of these and make them easier to consume. Again, how often do we need to care about which compute node, failure zone, or storage volume our app uses? Most of the time, we’re happy to let Kubernetes decide.

### Recap <a href="#recap" id="recap"></a>

Kubernetes was created by Google engineers based on lessons learned running containers at hyper-scale for many years. It was donated to the community as an open-source project and is now the industry standard platform for deploying and managing cloud-native applications.

It runs on any cloud or on-premises data center and abstracts the underlying infrastructure. This allows you to build hybrid clouds, as well as migrate on, off, and between different clouds. It’s open-sourced under the Apache 2.0 license and owned and managed by the Cloud Native Computing Foundation (CNCF).

---
description: >-
  Explore the fundamentals of Kubernetes by understanding its role as an
  orchestrator of containerized, cloud-native microservices applications. Learn
  how Kubernetes deploys, scales, and self-heals apps
icon: kubernetes
---

# Kubernetes Background

{% hint style="info" %}
Kubernetes is an orchestrator of containerized cloud-native microservices apps.
{% endhint %}

### What is an orchestrator?

An **orchestrator** is a system or platform that deploys applications and dynamically responds to changes. For example, Kubernetes can:

* Deploy applications
* Scale them up and down based on demand
* Self-heal them when things break
* Perform zero-downtime rolling updates and rollbacks

The best part is that it does all this without requiring our involvement. We need to configure a few things in the first place, but once we’ve done that, we sit back and let Kubernetes work its magic.

### What is containerization? <a href="#what-is-containerization" id="what-is-containerization"></a>

**Containerization** is packaging an application and dependencies as an image and then running it as a container.

Thinking of containers as the next generation of virtual machines (VMs) can be useful. Both are ways of packaging and running applications, but containers are smaller, faster, and more portable. Despite these advantages, containers haven’t replaced VMs, and running side-by-side in most cloud-native environments is common. However, containers are the top choice for most new applications.

### Cloud-native <a href="#cloud-native" id="cloud-native"></a>

Cloud-native applications possess cloud-like features such as auto-scaling, self-healing, automated updates, rollbacks, and more. Simply running a regular application in the public cloud does not make it cloud-native.

### Microservices <a href="#microservices" id="microservices"></a>

Microservices applications are built from many small, specialized, and independent parts that work together to form a useful application.

Consider an e-commerce app with the following six features:

* Web frontend
* Catalog
* Shopping cart
* Authentication
* Logging
* Store

To make this a microservices app, we design, develop, deploy, and manage each feature as its small application. We call each of these small apps a microservice, meaning this app will have six microservices.

This design offers huge flexibility by allowing all six microservices to have their own small development teams and release cycles. It also lets us scale and update each one independently.

The most common pattern is to deploy each microservice as its container. This means one or more web frontend containers, one or more catalog containers, one or more shopping cart containers, etc. Scaling any part of the app is as simple as adding or removing containers.

Now that we’ve explained a few things let’s rewrite that jargon-filled sentence from the start of the chapter.

The original sentence read, “Kubernetes is an orchestrator of containerized cloud-native microservices apps.” We now know this means that Kubernetes deploys and manages applications packaged as containers and can easily scale, self-heal, and be updated.

That should clarify some of the industry jargon. But don’t worry if some of it still needs clarification; we’ll cover everything again in much more detail throughout the course.

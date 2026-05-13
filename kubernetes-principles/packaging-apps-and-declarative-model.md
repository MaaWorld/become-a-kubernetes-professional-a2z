---
icon: kubernetes
---

# Packaging Apps and Declarative Model

Kubernetes runs containers, VMs, Wasm apps, and more. However, they all have to be wrapped in Pods to run on Kubernetes.

We’ll cover Pods shortly, but for now, think of them as a thin wrapper that abstracts different types of tasks so they can run on Kubernetes. The following courier analogy might help.

### Courier analogy <a href="#courier-analogy" id="courier-analogy"></a>

Couriers allow us to ship books, clothes, food, electrical items, and more, so long as we use their approved packaging and labels. Once we’ve packaged and labeled our goods, we hand them to the courier for delivery. The courier then handles the complex logistics of which planes and trucks to use, secure hand-offs to local delivery hubs, and eventual delivery to the customer. They also provide services for tracking packages, changing delivery details, and attesting successful delivery. All we have to do is package and label the goods.

Running apps on Kubernetes is similar. Kubernetes can run containers, VMs, Wasm apps and more, so long as we wrap them in Pods. Once wrapped in a Pod, we give the app to Kubernetes, and Kubernetes runs it. This includes the complex logistics of choosing appropriate nodes, joining networks, attaching volumes, and more. Kubernetes even lets us query apps and make changes.

### Deployments <a href="#deployments" id="deployments"></a>

Consider a quick example. We write an app in our favorite language, containerize it, push it to a registry, and wrap it in a Pod. At this point, we can give the Pod to Kubernetes, and Kubernetes will run it. However, most of the time, we’ll use a high-level controller to deploy and manage Pods. To do this, we wrap the Pod inside a controller object such as a Deployment.

Don’t worry about the details! We’ll cover everything in a lot more depth and with lots of examples later in the course. Right now, we only need to know two things:

1. Apps need to be wrapped in Pods to run on Kubernetes.
2. Pods are normally wrapped in high-level controllers for advanced features.

Let’s quickly go back to the courier analogy to help explain the role of controllers.

Most couriers offer additional services such as insurance for the goods we’re shipping, signature and photographic proof of delivery, express delivery services, and more. These add value to the service.

Again, Kubernetes is similar. It implements controllers that add value, such as ensuring the health of apps, automatically scaling when demand increases, and more.

The following figure shows a container wrapped in a Pod, which, in turn, is wrapped in a Deployment. Don’t worry about the YAML configuration. It’s just there to seed the idea.

<figure><img src="../.gitbook/assets/image (3).png" alt="" width="563"><figcaption><p>Object nesting</p></figcaption></figure>

The important thing to understand is that each layer of wrapping adds something:

* The container wraps the app and provides dependencies.
* The Pod wraps the container so it can run on Kubernetes.
* The Deployment wraps the Pod and adds self-healing, scaling, and more.

We post the Deployment (YAML file) to the API server as the desired state of the application, and Kubernetes implements it.

### Declarative model <a href="#declarative-model" id="declarative-model"></a>

The declarative model and desired state are at the core of how Kubernetes operates. They operate on three basic principles:

* Observed state
* Desired state
* Reconciliation

Observed state is what we have, desired state is what we want, and reconciliation is the process of keeping observed state in sync with desired state.

> **Note:** We use the terms actual state, current state, and observed state to mean the same thing — the most up-to-date view of the cluster.

Working of the declarative model

In Kubernetes, the declarative model works like this:

1. We describe the desired state of an application in a YAML manifest file.
2. We post the YAML file to the API server.
3. It gets recorded in the cluster store as a record of intent.
4. A controller notices that the observed state of the cluster doesn’t match the new desired state.
5. The controller makes the necessary changes to reconcile the differences.
6. The controller keeps running in the background, ensuring the observed state matches the desired state.

Let’s have a closer look.

We write manifest files in YAML that tell Kubernetes what an application should look like. We call this desired state, and it usually includes things such as which images to use, how many replicas, and which network ports.

Once we’ve created the manifest, we post it to the API server where it’s authenticated and authorized. The most common way of posting YAML files to Kubernetes is with the `kubectl` command-line utility.

Once authenticated and authorized, the configuration is persisted to the cluster store as a record of intent.

At this point, the observed state of the cluster doesn’t match our new desired state. A controller will notice this and begin the reconciliation process. This will involve making all the changes described in the YAML file and will likely include scheduling new Pods, pulling images, starting containers, attaching them to networks, and starting application processes.

Once reconciliation is completed, the observed state will match the desired state, and everything will be okay. However, the controllers keep running in the background, ready to reconcile any future differences.

### Declarative model vs. Imperative model <a href="#declarative-model-vs-imperative-model" id="declarative-model-vs-imperative-model"></a>

It’s important to understand that what we’ve described is very different from the traditional imperative model:

* The imperative model requires complex scripts of platform-specific commands to achieve an end state.
* The declarative model is a simple platform-agnostic way of describing an end state.

Kubernetes supports both but prefers the declarative model. This is because it integrates with version control systems and enables self-healing, autoscaling, and rolling updates.


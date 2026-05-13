---
icon: kubernetes
---

# Glossary

## Glossary <a href="#glossary" id="glossary"></a>

This glossary defines some of the most common Kubernetes-related terms used in the course.

**Term:** **Definition**

* **Admission controller:** It is a code that validates or mutates resources to enforce policies. It runs as part of the API admission chain immediately after authentication and authorization.
* **Annotation:** It is object metadata that can be used to expose alpha or beta capabilities or integrate with third-party systems.
* **API:** It stands for Application Programming Interface. In Kubernetes, all resources are defined in the API, which is RESTful and exposed via the _API server_.
* **API group:** It is a set of related API resources. For example, networking resources are usually located in the `networking.k8s.io` API group.
* **API resource:** All Kubernetes objects, such as Pods, Deployments, and Services, are defined in the API as resources.
* **API server:** It exposes the API on a secure port over HTTPS. It runs on the control plane.
* **Cloud controller manager:** It is a control plane service that integrates with the underlying cloud platform. For example, when creating a `LoadBalancer` Service, the cloud controller manager implements the logic to provision one of the underlying cloud’s internet-facing load balancers.
* **Cloud-native:** It is a loaded term that means different things to different people. Cloud-native is a way of designing, building, and working with modern applications and infrastructure. We personally consider an application _cloud-native_ if it can self-heal, scale on-demand, perform rolling updates, and possibly rollbacks.
* **Cluster:** It is a set of worker and control plane nodes that work together to run user applications.
* **Cluster store:** It is a control plane feature that holds the state of the cluster and apps. Typically, it is based on the `etcd` distributed data store and runs on the control plane. It can be deployed to its own cluster for higher performance and higher availability.
* **ConfigMap:** It is a Kubernetes object used to hold non-sensitive configuration data. It is a great way to add custom configuration data to a generic container at runtime without editing the image.
* **Container:** It is a lightweight environment for running modern apps. Each container is a virtual operating system with its own process tree, filesystem, shared memory, and more. One container runs one application process.
* **Container Network Interface (CNI):** It is a pluggable interface enabling different network topologies and architectures. Third parties provide CNI plugins that enable overlay networks, BGP networks, and various implementations of each.
* **Container runtime:** It is a low-level software running on every cluster Node responsible for pulling container images, starting containers, stopping containers, and other low-level container operations, typically containerd, Docker, or cri-o. Docker was deprecated in Kubernetes 1.20, and support was removed in 1.24.
* **Container Runtime Interface (CRI):** It is a low-level Kubernetes feature that allows container runtimes to be pluggable. With the CRI, you can choose the best container runtime for your requirements (Docker, containerd, cri-o, kata, etc.).
* **Container Storage Interface (CSI):** It is an interface that enables external third-party storage systems to integrate with Kubernetes. Storage vendors write a CSI driver/plugin that runs as a set of Pods on a cluster and exposes the storage system’s enhanced features to the cluster and applications.
* **containerd:** It is an industry-standard container runtime used in most Kubernetes clusters. It has been donated to the CNCF by Docker, Inc. It is pronounced as _container dee_.
* **Controller:** It is a control plane process running as a reconciliation loop monitoring the cluster and making the necessary changes so that the observed state of the cluster matches the desired state.
* **Control plane:** It is known as the brain of every Kubernetes cluster. It comprises the API, API server, scheduler, all controllers, and more. These components run on all _control plane nodes_ of every cluster.
* **Control plane node:** It is a cluster node hosting control plane services. Usually, it doesn’t run user applications. You should deploy three or five for high availability.
* **CRI-O:** It is container runtime. It is commonly used in OpenShift-based Kubernetes clusters.
* **CRUD:** These are four basic (create, read, update, and delete) operations used by many storage systems.
* **Custom Resource Definition (CRD):** It is an API resource used for adding your own resources to the Kubernetes API.
* **Data plane:** These are the worker Nodes of a cluster that host user applications.
* **Deployment:** It is a controller that deploys and manages a set of stateless Pods. It performs rollouts and rollbacks and can self-heal. It uses a ReplicaSet controller to perform scaling and self-healing operations.
* **Desired state:** It means what the cluster and apps should be like. For example, the _desired state_ of an application microservice might be five replicas of `xyz` container listening on port `8080/tcp`. It is vital to reconciliation.
* **Endpoints object:** It is an up-to-date list of healthy Pods matching a Service’s label selector. Basically, it’s the list of Pods a Service will send traffic to. It might eventually be replaced by EndpointSlices.
* **etcd:** It is an open-source distributed database used as the cluster store on most Kubernetes clusters.
* **Ingress:** It is an API resource that exposes multiple internal Services over a single external-facing LoadBalancer Service. It operates at layer 7 and implements path-based and host-based HTTP routing.
* **Ingress class:** It is an API resource that allows you to specify multiple different Ingress controllers on your cluster.
* **Init container:** It is a specialized container that runs and completes before the main app container starts. It is commonly used to check/initialize the environment for the main app container.
* **JSON:** It stands for JavaScript Object Notation. It is the preferred format for sending and storing data used by Kubernetes.
* **K8s:** It is a shorthand way to write Kubernetes. The "8" replaces the eight characters between the "K" and the "s" of Kubernetes, pronounced as "Kates". It is the reason why people say Kubernetes’ girlfriend is called Kate!
* `kubectl`**:** It is Kubernetes command-line tool. It sends commands to the API server and queries state via the API server.
* **Kubelet:** It is the main Kubernetes agent running on every cluster Node. It watches the API Server for new work assignments and maintains a reporting channel b
* \ack.
* **Kube-proxy:** It runs on every cluster node and implements low-level rules that handle traffic routing from Services to Pods. You send traffic to stable Service names, and kube-proxy ensures that the traffic reaches the Pods.
* **Label:** It is the metadata applied to objects for grouping. It works with label selectors to match Pods with high-level controllers. For example, Services send traffic to Pods based on sets of matching labels.
* **Label selector:** It is used to identify Pods to perform actions on. For example, when a Deployment performs a rolling update, it knows which Pods to update based on its label selector – only Pods with labels matching the Deployment’s label selector will be replaced and updated.
* **Manifest file:** It is a YAML file that holds the configuration of one or more Kubernetes objects. For example, a Service manifest file is typically a YAML file that holds the configuration of a Service object. When you post a manifest file to the API Server, its configuration is deployed to the cluster.
* **Microservices:** It is a design pattern for modern applications. Application features are broken into their own small applications (microservices/containers) and communicate via APIs. They work together to form a useful application.
* **Namespace:** It is a way to partition a single Kubernetes cluster into multiple virtual clusters. It is good for applying different quotas and access control policies on a single cluster. It is not suitable for strong workload isolation.
* **Node:** Also known as worker node, the nodes in a cluster run user applications. They run the `kubelet` process, a container runtime, and kube-proxy.
* **Observed state:** Also known as _current state_ or _actual state,_ it is the most up-to-date view of the cluster and running applications. Controllers are always working to make the observed state match the desired state.
* **Orchestrator:** It is a piece of software that deploys and manages apps. Modern apps are made from many small microservices that work together to form a useful application. Kubernetes orchestrates/manages these, keeps them healthy, scales them up and down, and more. Kubernetes is the de facto orchestrator of microservices apps based on containers.
* **PersistentVolume (PV):** It is a Kubernetes object that maps storage volumes on a cluster. External storage resources must be mapped to PVs before they can be used by applications.
* **PersistentVolumeClaim (PVC):** It is like a ticket/voucher that allows an app to use a PersistentVolume (PV). Without a valid PVC, an app cannot use a PV. It is combined with StorageClasses for dynamic volume creation.
* **Pod:** It is the smallest unit of scheduling on Kubernetes. Every container running on Kubernetes must run inside a Pod. The Pod provides a shared execution environment, including IP address, volumes, shared memory, etc.
* **RBAC:** It stands for role-based access control. It is an authorization module that determines whether authenticated users can perform actions against cluster resources.
* **Reconciliation loop:** It is a controller process watching the state of the cluster via the API Server, ensuring the observed state matches the desired state. Most controllers, such as the Deployment controller, run as a reconciliation loop.
* **ReplicaSet:** It runs as a controller and performs self-healing and scaling. It is used by Deployments.
* **REST:** It stands for REpresentational State Transfer. It is the most common architecture for creating web-based APIs. It uses the common HTTP methods (GET, POST, PUT, PATCH, DELETE) to manipulate and store objects.
* **Secret:** It is like a ConfigMap for sensitive configuration data. It is a way to store sensitive data outside of a container image and have it inserted into a container at runtime.
* **Service:** With a Capital "S", it is a Kubernetes object for providing network access to apps running in Pods. By placing a Service in front of a set of Pods, the Pods can fail, scale up and down, and be replaced without changing the network endpoint for accessing them. It can integrate with cloud platforms and provision internet-facing load balancers.
* **Service mesh:** It is an infrastructure software that enables features such as encryption of Pod-to-Pod traffic, enhanced network telemetry, and advanced routing. Common service meshes used with Kubernetes include Consul, Istio, Linkerd, and Open Service Mesh.
* **Sidecar:** It is a special container that runs alongside and augments a main app container. Service meshes are often implemented as sidecar containers that are injected into Pods and add network functionality.
* **StatefulSet:** It is a controller that deploys and manages stateful Pods. It is similar to a Deployment, but for stateful applications.
* **StorageClass:** It is a way to create different storage tiers/classes on a cluster. You may have a StorageClass called "fast" that creates NVMe-based storage, and another StorageClass called "medium-three-site" that creates slower storage replicated across three sites.
* **Volume:** It is a generic term for persistent storage.
* **WebAssembly (Wasm):** It is a secure sandboxed virtual machine format for executing apps.
* **Worker node:** It is a cluster node for running user applications. It is sometimes called a "Node" or "worker".
* **YAML:** It stands for Yet Another Markup Language. It is the configuration language you normally write Kubernetes configuration files in. It’s a superset of JSON.

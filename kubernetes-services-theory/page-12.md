---
description: Get introduced to another fundamental Kubernetes object, i.e., Services.
icon: kubernetes
---

# Services

### Introduction to Services <a href="#introduction-to-services" id="introduction-to-services"></a>

Kubernetes treats Pods as ephemeral objects and deletes them when any of the following events occur:

* Scale-down operations
* Rolling updates
* Rollbacks
* Failures

This means they’re unreliable, and apps can’t rely on them being there to respond to requests. Fortunately, Kubernetes has a solution — Service objects sit in front of one or more identical Pods and expose them via a reliable DNS name, IP address, and port.

The following figure shows a client connecting to an application via a Service called app1. The client connects to the name or IP of the Service, and the Service forwards requests to the application Pods behind it.

<figure><img src="../.gitbook/assets/image (20).png" alt="" width="520"><figcaption><p>Clients accessing Pods via a Service</p></figcaption></figure>

> **Note:** Services are resources in the Kubernetes API, and as such, we capitalize the “S” to avoid confusion with other uses of the word.

Every Service has a front-end and a back-end. The front-end includes a DNS name, IP address, and network port that Kubernetes guarantees will never change. The back-end is a label selector that sends traffic to healthy Pods with matching labels. Looking back to the above figure, the client sends traffic to the Service on either `app1:8080` or `10.99.11.23:8080`, and Kubernetes guarantees it will reach a Pod with the `project=tkb` label.

Services are also intelligent enough to maintain a list of healthy Pods with matching labels. This means we can scale up and down, perform rolling updates and rollbacks, and Pods can even fail, but the Service will always have an up-to-date list of active healthy Pods.

***

## Labels and Loose Coupling <a href="#page-title" id="page-title"></a>

Learn about loose coupling between Services and Pods via labels and label selectors.

### Directing traffic to Pods <a href="#directing-traffic-to-pods" id="directing-traffic-to-pods"></a>

Services use labels and selectors to know which Pods to send traffic to. This is the same technology that loosely couples Deployments to Pods.

The following figure shows a Service selecting on Pods with the `project=tkb` and `zone=prod` labels.

<figure><img src="../.gitbook/assets/image (21).png" alt="" width="404"><figcaption><p>Services and labels</p></figcaption></figure>

In this example, the Service sends traffic to Pod A, Pod B, and Pod D because they have all the labels it’s looking for. It doesn’t matter that Pod D has additional labels. However, it won’t send traffic to Pod C because it doesn’t have both labels. The following YAML defines a Deployment and a Service.

```yaml
// index.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tkb-2024
spec:
  replicas: 10
  <Snip>
  template:
    metadata:
      labels:
        project: tkb        <<====  Create Pods with these labels
        zone: prod          <<====  Create Pods with these labels
    spec:
      containers:
  <Snip>
---
apiVersion: v1
kind: Service
metadata:
  name: tkb
spec:
  ports:
  - port: 8080
  selector:
    project: tkb            <<==== Send to Pods with these labels
    zone: prod              <<==== Send to Pods with these labels
    
// Contents of a YAML file
```

The Deployment will create Pods with the `project=tkb` and `zone=prod` labels, and the Service will send traffic to them.

***

## Services and Endpoints Objects <a href="#page-title" id="page-title"></a>

Learn about Endpoints objects and the types of Services.

Whenever we create a Service, Kubernetes automatically creates an associated EndpointSlice to track healthy Pods with matching labels.

It works like this:

We create a Service, and the EndpointSlice controller automatically creates an associated EndpointSlice object. Kubernetes then watches the cluster, looking for Pods matching the Service’s label selector. Any new Pods matching the selector are added to the EndpointSlice, whereas any deleted Pods get removed. Applications send traffic to the Service name, and the application’s container uses the cluster DNS to resolve the name to an IP address. The container then sends the traffic to the Service’s IP, and the Service forwards it to one of the Pods listed in the EndpointSlice.

Older versions of Kubernetes used an Endpoints object instead of EndpointSlices. They’re functionally identical, but EndpointSlices perform better on large busy clusters.

### Service types <a href="#service-types" id="service-types"></a>

Kubernetes has several types of Services for different use cases and requirements. The major ones are:

* `ClusterIP`
* `NodePort`
* `LoadBalancer`

`ClusterIP` is the most basic and provides a reliable endpoint (name, IP, and port) on the internal Pod network. `NodePort` Services build on top of `ClusterIP` and allow external clients to connect via a port on every cluster node. `LoadBalancers` build on top of both and integrate with cloud load balancers for extremely simple access from the internet.

All three are important, and we’ll look at each of them in detail.

***

## Accessing Apps from Inside the Cluster <a href="#page-title" id="page-title"></a>

Review the idea of accessing Services from inside the cluster.

### `ClusterIP` Services  <a href="#clusterip-services" id="clusterip-services"></a>

`ClusterIP` is the default. It gets a name and IP that is programmed into the internal network fabric and is only accessible from inside the cluster. This means:

* The IP is only routable on the internal network.
* The name is automatically registered with the cluster’s internal DNS.
* All containers are preprogrammed to use the cluster’s DNS to resolve names.

### Example of `ClusterIP` Service <a href="#example-of-clusterip-service" id="example-of-clusterip-service"></a>

Let’s consider an example.

We’re deploying an application called skippy, and we want other applications on the cluster to access it by its name. To satisfy these requirements, we create a new `ClusterIP` Service called skippy. Kubernetes creates the Service, assigns it an internal IP, and creates the DNS records in the cluster’s internal DNS. Kubernetes also configures all containers on the cluster to use the cluster DNS for name resolution. This means every app on the cluster can connect to the new app using the skippy name.

<figure><img src="../.gitbook/assets/image (29).png" alt="" width="368"><figcaption><p>A Kubernetes cluster containing a Service named skippy</p></figcaption></figure>

However, this doesn’t work outside the cluster, as `ClusterIPs` aren’t routable, and they require access to the cluster DNS.

We’ll go into a lot more detail in a later chapter.

***

## Accessing Apps from Outside the Cluster <a href="#page-title" id="page-title"></a>

Learn how we can access an application from outside the cluster using Services.

### `NodePort` Services <a href="#nodeport-services" id="nodeport-services"></a>

`NodePort` Services build on top of `ClusterIP` Services by adding a dedicated port on every cluster node that external clients can use. We call this dedicated port the `NodePort`.

The following YAML shows a `NodePort` Service called `skippy`.

```yaml
// index.yaml

apiVersion: v1
kind: Service
metadata:
  name: skippy           <<==== Registered with the internal cluster DNS (ClusterIP)
spec:
  type: NodePort         <<==== Service type
  ports:
  - port: 8080           <<==== ClusterIP port
    targetPort: 9000     <<==== Application port in container
    nodePort: 30050      <<==== External port on every cluster node (NodePort)
  selector:
    app: hello-world

// Contents of a YAML file consisting of a NodePort Service object
```

Posting this to the cluster will create a `ClusterIP` Service with the usual internally routable IP and DNS name. It will also create port `30050` on every cluster node and map it back to the `ClusterIP`. This means external clients can send traffic to any cluster node on port `30050` and reach the Service.

The following figure shows a `NodePort` Service exposing three Pods on every cluster node on port `30050`. Step 1 shows an external client hitting a node on the `NodePort`. Step 2 shows the node forwarding the request to the `ClusterIP` of the Service inside the cluster. The Service picks a Pod from the EndpointSlice’s always-up-to-date list in Step 3 and forwards it to the chosen Pod in Step 4.

<figure><img src="../.gitbook/assets/image (30).png" alt="" width="563"><figcaption><p>NordPort service</p></figcaption></figure>

The external client could’ve sent the request to any cluster node, and the Service could’ve sent the request to any of the three healthy Pods. In fact, future requests will probably go to other Pods as the Service performs basic round-robin load balancing.

However, `NodePort` services have two significant limitations:

* They use high-numbered ports between `30000-32767`.
* Clients need to know the names or IPs of nodes, as well as whether nodes are healthy.

This is why most people use `LoadBalancer` Services instead.

### `LoadBalancer` Services  <a href="#loadbalancer-services" id="loadbalancer-services"></a>

`LoadBalancer` Services are the easiest way of exposing Services to external clients. They simplify `NodePort` Services by putting a cloud load balancer in front of them.

The following figure shows a `LoadBalancer` Service. As we can see, it’s basically a `NodePort` Service fronted by a highly-available load balancer with a publicly resolvable DNS name and low port number.

<figure><img src="../.gitbook/assets/image (31).png" alt="" width="563"><figcaption><p>LoadBalancer Service</p></figcaption></figure>

The client connects to the load balancer via a reliable, friendly DNS name on a low-numbered port, and the load balancer forwards the request to a `NodePort` on a healthy cluster node. From there, it’s the same as a `NodePort` Service — send to the internal `ClusterIP` Service, select a Pod from the EndpointSlice, and send the request to the Pod.

The following YAML creates a `LoadBalancer` Service listening on port `8080` and maps it all the way through to port `9000` on Pods with the `project=tkb` label. It automatically creates the required `NodePort` and `ClusterIP` constructs in the background.

```yaml
// index.yaml
apiVersion: v1
kind: Service
metadata:
  name: lb               <<==== Registered with cluster DNS
spec:
  type: LoadBalancer
  ports:
  - port: 8080           <<==== Load balancer port
    targetPort: 9000     <<==== Application port inside container
  selector:
    project: tkb
    
// Contents of a YAML file consisting of a LoadBalancer Service object
```

We’ll create and use a `LoadBalancer` Service in a later chapter.

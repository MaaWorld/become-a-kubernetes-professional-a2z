# Pod Theory and Scheduling

Every app on Kubernetes runs inside a Pod.

* When we deploy an app, we deploy it in a Pod.
* When we terminate an app, we terminate its Pod.
* When we scale an app up, we add more Pods.
* When we scale an app down, we remove Pods.
* When we update an app, we deploy new Pods.

This makes Pods important, and that is why this chapter goes into detail.

### Introduction to Pods <a href="#introduction-to-pods" id="introduction-to-pods"></a>

Kubernetes uses Pods for a lot of reasons. They’re an abstraction layer. They enable resource sharing, add features, enhance scheduling, and more.

Let’s take a closer look at some of these.

#### Pods are an abstraction layer <a href="#pods-are-an-abstraction-layer" id="pods-are-an-abstraction-layer"></a>

Pods abstract the details of different workload types. This means we can run containers, VMs, serverless functions, and Wasm apps in them, and Kubernetes doesn’t know the difference.

Using Pods as an abstraction layer benefits Kubernetes as well as the workloads:

* Kubernetes can focus on deploying and managing Pods without having to care about what’s inside them.
* Heterogenous workloads can run side-by-side on the same cluster, leverage the full power of the declarative Kubernetes API, and get all the other benefits of Pods.

Containers and Wasm apps work with standard Pods, standard workload controllers, and standard runtimes. However, serverless functions and VMs need a bit of extra help.

Serverless functions run in standard Pods but require apps like [Knative](https://knative.dev/) to extend the API with custom resources and controllers. VMs are similar, needing apps like [KubeVirt](https://kubevirt.io/) to extend the API.

The following figure shows four different workloads running on the same cluster. Each workload is wrapped in a Pod, managed by a controller, and uses a standard runtime. VM workloads run in a VirtualMachineInstance (VMI) instead of a Pod, but VMIs are very similar to Pods and utilize many Pod features.

<figure><img src="../.gitbook/assets/image (17).png" alt="" width="563"><figcaption><p>Different workloads wrapped in Pods</p></figcaption></figure>

#### Pods augment workloads <a href="#pods-augment-workloads" id="pods-augment-workloads"></a>

Pods augment workloads in many ways, including all of the following:

* Resource sharing
* Advanced scheduling
* Application health probes
* Restart policies
* Security policies
* Termination control
* Volumes

The following command shows a complete list of Pod attributes and returns over 1,000 lines.

```yaml
$ kubectl explain pods --recursive | more
KIND:     Pod
VERSION:  v1
DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.
FIELDS:
   apiVersion	      <string>
   kind	            <string>
   metadata	        <Object>
      annotations	  <map[string]string>
      labels	      <map[string]string>
      name	        <string>
      namespace	    <string>
<Snip>

// Get the attributes of a Pod
```

We can even drill into specific Pod attributes and see their supported values. The following example drills into the Pod `restartPolicy` attribute.

```powershell
$ kubectl explain pod.spec.restartPolicy
KIND:     Pod
VERSION:  v1
FIELD:    restartPolicy <string>
DESCRIPTION:
     Restart policy for all containers within the pod. One of Always, OnFailure, Never. 
     Default to Always. 
     More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/...
     Possible enum values:
     - `"Always"`
     - `"Never"`
     - `"OnFailure"`
     
// Explaining the Pod restartPolicy attribute
```

Despite adding so much, Pods are lightweight and add very little overhead.

#### Pods enable resource sharing <a href="#pods-enable-resource-sharing" id="pods-enable-resource-sharing"></a>

Pods run one or more containers, and all containers in the same Pod share the Pod’s execution environment. This includes:

* Shared filesystem and volumes (mnt namespace)
* Shared network stack (net namespace)
* Shared memory (IPC namespace)
* Shared process tree (pid namespace)
* Shared hostname (uts namespace)

The following figure shows a multi-container Pod with both containers sharing the Pod’s volume and network resources

<figure><img src="../.gitbook/assets/image (18).png" alt="" width="331"><figcaption><p>Multi-container Pod sharing IP and volume</p></figcaption></figure>

## Pods and Scheduling <a href="#page-title" id="page-title"></a>

Kubernetes guarantees to schedule all containers in the same Pod to the same cluster node. Despite this, you should only put containers in the same Pod if they need to share resources such as memory, volumes, and networking. If your only requirement is to schedule two workloads to the same node, you should put them in their own Pods and use one of the following options to schedule them together.

> **Note:** Before going any further, remember that nodes are host servers that can be physical servers, virtual machines, or cloud instances. Pods wrap containers and execute on nodes.

Pods provide a lot of advanced scheduling features, including all of the following:

* `nodeSelectors`
* Affinity and anti-affinity
* Topology spread constraints
* Resource requests and resource limits

`nodeSelectors` are the simplest way of running Pods on specific nodes. You give the `nodeSelector` a list of labels, and the scheduler will only assign the Pod to a node with all the labels.

Affinity and anti-affinity rules are like a more powerful `nodeSelector`.

As the names suggest, they support affinity and anti-affinity rules. But they also support hard and soft rules, and they can select on nodes as well as Pods:

* Affinity rules _attract._
* Anti-affinity rules _repel._
* Hard rules must be _obeyed._
* Soft rules are only _suggestions._

Selecting on nodes is common and works like a `nodeSelector` where you supply a list of labels, and the scheduler will assign the Pod to nodes possessing the labels.

To select on Pods, the scheduler takes a similar list of labels and schedules the Pod to nodes running other Pods possessing the labels.

Consider a couple of examples.

A hard node affinity rule specifying the `project=qsk` label tells the scheduler it can only run the Pod on nodes with the `project=qsk` label. It won’t schedule the Pod if it can’t find a node with that label. If it were a soft rule, the scheduler would try to find a node with the label, but if it can’t find one, it’ll still schedule it. If it were an anti-affinity rule, the scheduler would look for nodes that don’t have the label. The logic works the same for Pod-based rules.

Topology spread constraints are a flexible way of intelligently spreading Pods across your infrastructure for availability, performance, locality, or any other requirements. A typical example is spreading Pods across your cloud or data center’s underlying availability zones for high availability (HA). However, you can create custom domains for almost anything, such as scheduling Pods closer to data sources, closer to clients for improved network latency, and many more reasons.

Resource requests and resource limits are very important, and every Pod should use them. They tell the scheduler how much CPU and memory a Pod needs, and the scheduler uses them to select nodes with enough resources. If you don’t specify them, the scheduler cannot know what resources a Pod requires and may schedule it to a node with insufficient resources.

### Deploying Pods <a href="#deploying-pods" id="deploying-pods"></a>

Deploying a Pod includes the following steps:

1. Define the Pod in a YAML manifest file.
2. Post the manifest to the API server.
3. The request is authenticated and authorized.
4. The Pod spec is validated.
5. The scheduler filters nodes based on `nodeSelectors`, affinity and anti-affinity rules, topology spread constraints, resource requirements and limits, and more.
6. The Pod is assigned to a healthy node meeting all requirements.
7. The `kubelet` on the node watches the API server and notices the Pod assignment.
8. The `kubelet` downloads the Pod spec and asks the local runtime to start it.
9. The `kubelet` monitors the Pod status and reports status changes to the API server.

If the scheduler can’t find a suitable node, it marks it as pending.

Deploying a Pod is an atomic operation. This means a Pod only starts servicing requests when all its containers are up and running.

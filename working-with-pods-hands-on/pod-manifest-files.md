---
icon: kubernetes
---

# Pod Manifest Files

### Introduction to Pod manifest files <a href="#introduction-to-pod-manifest-files" id="introduction-to-pod-manifest-files"></a>

Let’s see our first Pod manifest. This is the `pod.yml` file from the `pods` folder.

```yaml
// pod.yaml

kind: Pod
apiVersion: v1
metadata:
  name: hello-pod
  labels:
    zone: prod
    version: v1
spec:
  containers:
  - name: hello-ctr
    image: nigelpoulton/k8sbook:1.0
    ports:
    - containerPort: 8080
    resources:
      limits:
        memory: 128Mi
        cpu: 0.5
        
// YAML file of a Pod
```

It’s a simple example, but straight away, we can see four top-level fields: `kind`, `apiVersion`, `metadata`, and `spec`. Let's examine these fields one by one:

* The `kind` field tells Kubernetes what type of object we’re defining. This one’s defining a Pod, but if we were defining a Deployment, the `kind` field would say `Deployment`.
* `apiVersion` tells Kubernetes what version of the API to use when creating the object. This manifest describes a Pod and tells Kubernetes to build it using the `v1` version of the API.
* The `metadata` section names the Pod `hello-pod` and gives it two labels. We’ll use the labels in a future chapter to connect the Pod to a Service for networking.
* Most of the action happens in the `spec` section. This example defines a single-container Pod with an application container called `hello-ctr`. The container is based on the `nigelpoulton/k8sbook:1.0` image, listens on port `8080`, and tells the scheduler it needs a maximum of 256MB of memory and half a CPU.

We just add more containers below the `spec.containers` section to make it a multi-container Pod.

***

Kubernetes YAML files are excellent sources of documentation. We can use them to quickly get new team members up to speed and help bridge the gap between developers and operations.

For example, new team members can read our YAML files and quickly learn our application’s basic functions and requirements. Operations teams can also use them to understand application requirements such as network ports, CPU and memory requirements, and more.

We can also store them in source control repositories for easy versioning and running diffs against other versions.

### Deploying Pods from a manifest file <a href="#deploying-pods-from-a-manifest-file" id="deploying-pods-from-a-manifest-file"></a>

Run the following playground to create a Kubernetes cluster and follow along to deploy our first application.

```yml
// pod.yml

apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    zone: prod
    version: v1
spec:
  containers:
  - name: hello-ctr
    image: nigelpoulton/k8sbook:1.0
    ports:
    - containerPort: 8080
    resources:
      limits:
        memory: 256Mi
        cpu: 0.5
```

Run the following `kubectl apply` command to deploy the Pod. The command sends the `pod.yml` file to the API server defined in the current context of our `kubeconfig` file. It also attaches credentials from our `kubeconfig` file.

```powerquery
$ kubectl apply -f pod.yml
pod/hello-pod created
```

Although the output says the Pod is created, it might still be pulling the image and starting the container.

Run a `kubectl get pods` to check the status.

```powershell
$ kubectl get pods
NAME        READY    STATUS             RESTARTS   AGE
hello-pod   0/1      ContainerCreating  0          9s
```

The Pod in the example isn’t fully created yet — the `READY` column shows zero containers ready, and the `STATUS` column shows why.

This is a good time to mention that Kubernetes automatically pulls (downloads) images from Docker Hub. To use another registry, just add the registry’s URL before the image name in the YAML file.

Once the `READY` column shows `1/1` and the `STATUS` column shows `Running`, our Pod will be running on a healthy cluster and node and monitored by the node’s `kubelet`.

We’ll see how to connect to the app and test it in future chapters.

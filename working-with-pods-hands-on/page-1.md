---
icon: kubernetes
---

# Introspecting Pods

Let’s look at some of the main ways we’ll use `kubectl` to monitor and inspect Pods.

### `kubectl get` <a href="#kubectl-get" id="kubectl-get"></a>

We’ve already run a `kubectl get pods` command and seen that it returns a single line of basic info. However, the following flags get us a lot more information:

* `-o wide` gives a few more columns but is still a single line of output.
* `-o yaml` gets us everything Kubernetes knows about the object.

The following example shows the output of a `kubectl get pods` with the `-o yaml` flag. The output is snipped for the course, but notice how it’s divided into two main parts:

* `spec`
* `status`

The `spec` section shows the desired state of the object, and the `status` section shows the observed state.

```yaml
// index.yaml

$ kubectl get pods hello-pod -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      <Snip>
  name: hello-pod
  namespace: default
spec:                           <<==== Desired state is in this block
  containers:
  - image: nigelpoulton/k8sbook:1.0
    imagePullPolicy: IfNotPresent
    name: hello-ctr
    ports:
    <Snip>
status:                         <<==== Observed state is in this block
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2024-01-03T18:21:51Z"
    status: "True"
    type: Initialized
  <Snip>

// Get the status of hello-pod in YAML format
```

The full output contains much more than the 17-line YAML file we used to create the Pod. So, where does Kubernetes get all this extra detail?

Two main sources:

* Pods have many properties, and anything we don’t explicitly define in a YAML file gets populated with default values.
* The `status` section shows us the current state of the Pod

### `kubectl describe` <a href="#kubectl-describe" id="kubectl-describe"></a>

Another great command is `kubectl describe`. This gives us a nicely formatted overview of an object, including lifecycle events.

```powershell
$ kubectl describe pod hello-pod
Name:         hello-pod
Namespace:    default
Labels:       version=v1
              zone=prod
Status:       Running
IP:           10.1.0.103
Containers:
  hello-ctr:
    Container ID:   containerd://ec0c3e...
    Image:          nigelpoulton/k8sbook:1.0
    Port:           8080/TCP
    <Snip>
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  <Snip>
Events:
  Type    Reason     Age        Message
  ----    ------     ----     -------
  Normal  Scheduled  5m30s    Successfully assigned ...
  Normal  Pulling    5m30s    Pulling image "nigelpoulton/k8sbook:1.0"
  Normal  Pulled     5m8s     Successfully pulled image ...
  Normal  Created    5m8s     Created container hello-ctr
  Normal  Started    5m8s     Started container hello-ctr
```

The output is snipped for the course, but it’s a very useful command.

### `kubectl logs` <a href="#kubectl-logs" id="kubectl-logs"></a>

We can use the `kubectl logs` command to pull the logs from any container in a Pod. The basic format of the command is `kubectl logs <pod>`.

If we run the command against a multi-container Pod, we automatically get the logs from the first container in the Pod. However, we can override this by using the `--container` flag and specifying the container name we want the logs from. If we’re unsure of the names of containers or the order they appear in a multi-container Pod, just run a `kubectl describe pod <pod>` command. We can get the same information from the Pod’s YAML file.

The following YAML shows a multi-container Pod with two containers. The first container is called `app`, and the second is called `syncer`. Running a `kubectl logs` against this Pod without specifying the `--container` flag will get us the logs from the app container.

```yaml
// index.yaml
kind: Pod
apiVersion: v1
metadata:
  name: logtest
spec:
  containers:
  - name: app                   <<==== First container (default)
    image: nginx
      ports:
        - containerPort: 8080
  - name: syncer                <<==== Second container
    image: k8s.gcr.io/git-sync:v3.1.6
    volumeMounts:
    - name: html
<Snip>

// A multi-container Pod with two containers
```

We’d run the following `kubectl logs` command if we wanted the logs from the `syncer` container.

```powershell
$ kubectl logs logtest --container syncer

// Get the logs of the container syncer from the Pod logtest
```

***

## kubectl exec: Running Commands in Pods <a href="#page-title" id="page-title"></a>

The `kubectl exec` command is a great way to execute commands inside running containers.

We can use `kubectl exec` in two ways:

1. Remote command execution
2. Exec session

Remote command execution lets us send commands to a container from our local shell. The container executes the command and returns the output to our shell.

An exec session connects our local shell to the container’s shell and is the same as being logged on to the container.

Let’s look at both, starting with remote command execution.

```powershell
$ kubectl exec hello-pod -- ps
PID   USER     TIME  COMMAND
  1   root      0:00 node ./app.js
 17   root      0:00 ps aux
 
// Run the ps command in the first container of the Pod: hello-pod
```

> If you’re getting the following error: `Error from server (BadRequest): pod hello-pod does not have a host assigned`. Please wait for the Pod to run. You can check the pod status using the `kubectl get pods` command.

The container executed the `ps` command and displayed the result in our local terminal.

The format of the command is `kubectl exec <pod> -- <command>`, and we can execute any command installed in the container. By default, commands execute in the first container in a Pod, but we can override this with the `--container` flag.

Try running the following command.

```powershell
$ kubectl exec hello-pod -- curl localhost:8080
OCI runtime exec failed:...... "curl": executable file not found in $PATH

// Run the localhost:8080 command in the first container of the Pod: hello-pod
```

This one failed because the `curl` command isn’t installed in the container.

Let’s use `kubectl` `exec` to get an interactive exec session to the same container. This works by connecting our terminal to the container’s terminal, and it feels like we’re logged on to the container.

Run the following command to create an exec session to the first container in the `hello-pod` Pod. Our shell prompt will change to indicate we’re connected to the container’s shell.

```powershell
$ kubectl exec -it hello-pod -- sh
#

// Create an exec session
```

The `-it` flag tells `kubectl exec` to make the session interactive by connecting our shell’s STDIN and STDOUT streams to the STDIN and STDOUT of the first container in the Pod. The `sh` command starts a new shell process in the session, and our prompt will change to indicate we’re now inside the container.

Run the following commands from within the exec session to install the `curl` binary and then execute a `curl` command.

```powershell
# apk add curl
<Snip>

# curl localhost:8080
<html><head><title>K8s rocks!</title><link rel="stylesheet" href="http://netdna....

//install the curl command then use 
```

Making changes like this to live Pods is an _anti-pattern_ as Pods are designed as immutable objects. However, it’s okay for demonstration purposes like this.

### Pod hostnames <a href="#pod-hostnames" id="pod-hostnames"></a>

Pods get their names from their YAML file’s `metadata.name` field and Kubernetes uses this as the hostname for every container in the Pod.

If you’re following along, you’ll notice that we have a single Pod deployed called `hello-pod`. We deployed it from the following YAML file that sets the Pod name as `hello-pod`.

```yaml
// index.yaml

kind: Pod
apiVersion: v1
metadata:
  name: hello-pod      <<==== Pod hostname. Inherited by all containers.
  labels:
  <Snip>
  
// Pod YAML file
```

Run the following command from inside our existing exec session to check the container’s hostname. The command is case-sensitive.

```powershell
$ env | grep HOSTNAME
HOSTNAME=hello-pod

// Check the container’s hostname
```

As we can see, the container’s hostname matches the name of the Pod. All containers would have the same hostname if it was a multi-container Pod.

Because of this, we should ensure that Pod names are valid DNS names (a-z, 0-9, the minus and period signs).

Type `exit` to quit our exec session and return to our local terminal.

***

## Check Pod Immutability <a href="#page-title" id="page-title"></a>

Learn how to edit a live Pod and how to specify resource requests and resource limits for each container in a Pod.

### Pods as immutable objects <a href="#pods-as-immutable-objects" id="pods-as-immutable-objects"></a>

Pods are designed as immutable objects, meaning we shouldn’t change them after deployment.

Immutability applies at two levels:

* Object immutability (the Pod)
* App immutability (containers)

Kubernetes handles object immutability by preventing changes to a running Pod’s configuration. However, Kubernetes can’t always prevent us from changing the app and filesystem in containers. We’re responsible for ensuring containers and their apps are stateless and immutable.

The following example uses `kubectl edit` to edit a live Pod object. Try and change any of these attributes:

* Pod name
* Container name
* Container port
* Resource requests and limits

Run the following `kubectl edit` command in the above terminal, and it will open the file in our default editor.

> **Note:** For Mac and Linux users, it will typically open the file in `vi`, whereas for Windows, it’s usually `notepad.exe`.

```yaml
// index.yaml

$ kubectl edit pod hello-pod

# Please edit the object below. Lines beginning with a '#' will be ignored...
apiVersion: v1
kind: Pod
metadata:
  <Snip>
  labels:
    version: v1
    zone: prod
  name: hello-pod                    <<==== Try to change this
  namespace: default
  resourceVersion: "432621"
  uid: a131fb37-ceb4-4484-9e23-26c0b9e7b4f4
spec:
  containers:
  - image: nigelpoulton/k8sbook:1.0
    imagePullPolicy: IfNotPresent
    name: hello-ctr                  <<==== Try to change this
    ports:
    - containerPort: 8080            <<==== Try to change this
      protocol: TCP
    resources:
      limits:
        cpu: 500m                    <<==== Try to change this
        memory: 256Mi                <<==== Try to change this
      requests:
        cpu: 500m                    <<==== Try to change this
        memory: 256Mi                <<==== Try to change this

// Changing the pod specifications
```

Edit the file, save the changes, and close the editor. We’ll get a message telling us the changes are forbidden because the attributes are immutable.

> **Note:** If you get stuck inside the `kubectl edit` session, you can probably exit by pressing "esc" button and then typing the following key combination — `:wq`.

### Resource requests and resource limits <a href="#resource-requests-and-resource-limits" id="resource-requests-and-resource-limits"></a>

Kubernetes lets us specify resource requests and resource limits for each container in a Pod.

* Requests are minimum values.
* Limits are maximum values.

Consider the following snippet from a Pod YAML:

```yaml
// index.yaml
resources:
  requests:              <<==== Minimums for scheduling
    cpu: 0.5
    memory: 256Mi
  limits:                <<==== Maximums for kubelet to cap
    cpu: 1.0
    memory: 512Mi
    
// Example snippet from a Pod YAML
```

This container needs a minimum of 256Mi of memory and half a CPU. The scheduler reads this and assigns it to a node with enough resources. If it can’t find a suitable node, it marks the Pod as pending, and the cluster auto-scaler will attempt to provision a new cluster node.

Assuming the scheduler finds a suitable node, it assigns the Pod to the node, and the `kubelet` downloads the Pod spec and asks the local runtime to start it. As part of the process, the `kubelet` reserves the requested CPU and memory, guaranteeing the resources will be there when needed. It also sets a cap on resource usage based on each container’s resource limits. In this example, it sets a cap of one CPU and 512Mi of memory. Most runtimes will also enforce resource limits, but how each runtime implements this can vary.

While a container executes, it is guaranteed its minimum requirements (requests). However, it’s allowed to use more if the node has additional available resources, but it’s never allowed to use more than what we specify in its limits.

For multi-container Pods, the scheduler combines the requests for all containers and finds a node with enough resources to satisfy the full Pod.

If you’ve been following the examples closely, you’ll have noticed that the `pod.yml` we used to deploy the `hello-pod` only specified resource limits — it didn’t specify resource requests. However, some command outputs have shown limits and requests. This is because Kubernetes automatically sets requests to match limits if we only specify limits.

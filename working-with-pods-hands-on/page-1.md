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

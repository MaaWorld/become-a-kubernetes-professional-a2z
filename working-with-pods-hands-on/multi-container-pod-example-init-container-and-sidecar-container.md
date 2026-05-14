---
icon: kubernetes
---

# Multi-Container Pod Example: Init Container & Sidecar Container

{% stepper %}
{% step %}
#### Introduction to an init container

The following YAML defines a multi-container Pod with an init container and main app container.

```yml
// initpod.yml

apiVersion: v1
kind: Pod
metadata:
  name: initpod
  labels:
    app: initializer
spec:
  initContainers:
  - name: init-ctr
    image: busybox:1.28.4
    command: ['sh', '-c', 'until nslookup k8sbook; do echo waiting for k8sbook service;\\
              sleep 1; done; echo Service found!']
  containers:
    - name: web-ctr
      image: nigelpoulton/web-app:1.0
      ports:
        - containerPort: 8080
```

```yml
// initsvc.yml

apiVersion: v1
kind: Service
metadata:
  name: k8sbook
spec:
  selector:
    app: initializer
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

Defining a container under the `spec.initContainers` block makes it an init container that Kubernetes guarantees will run and complete before regular containers.

Regular app containers are defined under the `spec.containers` block and will not start until all init containers are successfully completed.

This example has a single init container called `init-ctr` and a single app container called `web-ctr`. The init container runs a loop looking for a Kubernetes Service called `k8sBook`. As soon as we create the Service, the init container will get a response and exit. This allows the main container to start. We’ll learn about Services in a later chapter.

We deploy the multi-container Pod with the following command and then run a `kubectl get pods` with the `--watch` flag to see if it comes up.

```powershell
$ kubectl apply -f initpod.yml
pod/initpod created

$ kubectl get pods --watch
NAME      READY   STATUS     RESTARTS   AGE
initpod   0/1     Init:0/1   0          6s

// Create a Pod and check its statusIntroduction to a sidecar container#
```

The `Init:0/1` status tells us that the init container is still running, meaning the main container hasn’t started yet.

> Press the "Ctrl" + "C" buttons to terminate the above session.

If we run a `kubectl describe` command, we’ll see the overall Pod status is `Pending`.

```powershell
$ kubectl describe pod initpod
Name:             initpod
Namespace:        default
Priority:         0
Service Account:  default
Node:             docker-desktop/192.168.65.3
Labels:           app=initializer
Annotations:      <none>
Status:           Pending              <<==== Pod status
<Snip>

// Get the attributes of a Pod
```

The Pod will remain in this phase until we create a Service called `k8sbook`.

Run the following commands to create the Service and re-check the Pod status.

```powershell
$ kubectl apply -f initsvc.yml
service/k8sbook created

$ kubectl get pods --watch
NAME      READY   STATUS            RESTARTS   AGE
initpod   0/1     Init:0/1          0          15s
initpod   0/1     PodInitializing   0          3m39s
initpod   1/1     Running           0          3m57s

// Create a Service object and recheck the Pod's status
```

The init container completes as soon as the Service appears, and the main application container starts. Give it a few seconds to fully start.

If we run another `kubectl describe` against the initpod Pod, we’ll see the init container is in the terminated state because it was completed successfully (exit code 0).
{% endstep %}

{% step %}
#### Introduction to a sidecar container

Sidecar containers run alongside the main application container for the entire lifecycle of the Pod. We currently define them as regular containers under the `spec.containers` section of the Pod YAML, and their job is to augment the main application container or provide a secondary support service.

> **Note:** At the time of writing, Kubernetes doesn’t have API support for sidecar containers. However, Kubernetes 1.28 introduced alpha support for a potential solution.

The following YAML file defines a multi-container Pod with both containers mounting the same shared volume. Listing the main app container as the first container and sidecars after it is conventional.

The main app container is called `ctr-web`. It’s based on an NGINX image and serves as a static web page loaded from the shared HTML volume. The second container is called `ctr-sync` and it is the sidecar. It watches a GitHub repository, and syncs changes into the same shared HTML volume. When the contents of the GitHub repository change, the sidecar copies the updates to the shared volume, where the app container notices and serves an updated version of the web page.

We’ll walk through the following steps to see it in action:

1. Fork the GitHub repository.
2. Update the YAML file with the URL of the forked repository.
3. Deploy the app.
4. Connect to the app and see it display 'This is version 1.0.'
5. Make a change to the fork of the GitHub repository.
6. Verify that the changes appear on the web page.

Go to GitHub and fork the following repository. You’ll need a GitHub account to do this.

```
https://github.com/nigelpoulton/ps-sidecar
```

Use the following widget to execute the commands.

```yml
// sidecar-local.yml

apiVersion: v1
kind: Pod
metadata:
  name: git-sync
  labels:
    app: sidecar
spec:
  containers:
  - name: ctr-web
    image: nginx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/
  - name: ctr-sync
    image: k8s.gcr.io/git-sync:v3.1.6
    volumeMounts:
    - name: html
      mountPath: /tmp/git
    env:
    - name: GIT_SYNC_REPO
      value: https://github.com/nigelpoulton/ps-sidecar.git
    - name: GIT_SYNC_BRANCH
      value: master
    - name: GIT_SYNC_DEPTH
      value: "1"
    - name: GIT_SYNC_DEST
      value: "html"
  volumes:
  - name: html
    emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: svc-sidecar
spec:
  selector:
    app: sidecar
  type: NodePort
  ports:
  - port: 80
    nodePort: 30001
```

Edit the `sidecar-local.yml` file in the above widget. Change the `GIT_SYNC_REPO` value to match the URL of your forked repo, and run the widget again.

Run the following command to deploy the application. It will deploy the Pod as well as a Service we’ll use to connect to the app.

```c
// main.c 
$ kubectl apply -f sidecar-local.yml
pod/git-sync created
service/svc-sidecar created

// Deploy the application
```

Check the status of the Pod with the following command.

```powershell
// main.c
 $ kubectl get pods
```

As soon as the Pod enters the running state, execute the following command:

```c
// main.c
$ kubectl get svc

$ kubectl port-forward svc/svc-sidecar 8080:80 --address 0.0.0.0

// Check the status of the Service
```

Click the link in the above terminal to view the web page in a new browser tab. It will display 'This is version 1.0.' Be sure to complete the following steps against your forked repo:<br>

* Go to your forked repo and edit the `index.html` file.
* Change the `<h1>` line to something different and save the changes.
* Refresh the app’s web page to see the updates.

Congratulations! The sidecar container successfully watched a remote Git repo, synced the changes to a shared volume, and the main app container updated the web page. Feel free to run the `kubectl get pods` and `kubectl describe pod` commands to see how multi-container Pods appear in the outputs.

#### Clean up <a href="#clean-up" id="clean-up"></a>

If you’ve created a cluster on a cloud distribution, you’ll have the following objects on your cluster.

| **Pods**    | **Services** |
| ----------- | ------------ |
| `hello-pod` |              |
| `initpod`   | k8sbook      |
| `git-sync`  | svc-sidecar  |

Delete them with the following commands.

```powershell
$ kubectl delete pod hello-pod initpod git-sync
pod "hello-pod" deleted
pod "initpod" deleted
pod "git-sync" deleted

$ kubectl delete svc k8sbook svc-sidecar
service "k8sbook" deleted
service "svc-sidecar" deleted

// Delete the Pod and Service objects
```

The objects can be deleted using their YAML files.

```powershell
$ kubectl delete -f sidecarpod.yml -f initpod.yml -f pod.yml -f initsvc.yml
pod "git-sync" deleted
service "svc-sidecar" deleted
pod "initpod" deleted
pod "hello-pod" deleted
service "k8sbook" deleted

// Delete the objects using their YAML files
```

You may also want to delete your fork of the GitHub repository.
{% endstep %}
{% endstepper %}
